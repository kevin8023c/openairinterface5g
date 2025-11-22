# MOCN (Multi-Operator Core Network) 实现分析

## 概述
MOCN允许一个gNB连接到多个运营商的核心网（AMF）。实现的关键是：
1. **下行**：gNB在SIB1中广播多个PLMN，UE可以看到并选择
2. **上行**：UE选择的PLMN传递回gNB，gNB根据PLMN选择正确的AMF

---

## 一、下行调用链：gNB广播多个PLMNs给UE

### 1.1 配置层面（Configuration → MAC）

**文件路径和函数调用链：**

```
1. openair2/GNB_APP/gnb_app.c
   └─ gnb_app_task()
      └─ 读取配置文件，创建 gNB_RrcConfigurationReq 消息
      └─ configuration.num_plmn = [从配置读取]
      └─ configuration.plmn[i] = {mcc, mnc, mnc_digit_length}

2. openair2/RRC/NR/rrc_gNB.c
   └─ rrc_gNB_process_GTPV1U() / rrc_gNB_init()
      └─ 接收 GNB_APP_REGISTER_REQ 消息
      └─ 存储到 rrc->configuration.plmn[]
      └─ rrc->configuration.num_plmn

3. openair2/LAYER2/NR_MAC_gNB/main.c 或 gNB_scheduler.c
   └─ 调用 nr_mac_config_scc()
      └─ 传递 rrc->configuration 到 MAC 层

4. openair2/LAYER2/NR_MAC_gNB/gNB_scheduler.c
   └─ nr_mac_config_scc()
      └─ 调用 get_SIB1_NR(scc, &mac_config)

5. openair2/LAYER2/NR_MAC_gNB/nr_radio_config.c
   └─ get_SIB1_NR(const NR_ServingCellConfigCommon_t *scc, 
                   const nr_mac_config_t *mac_config)
      ⚠️ **当前问题位置：第2714行**
      └─ int num_plmn = 1;  // 硬编码只支持1个PLMN！
      └─ 只循环1次，填充1个PLMN到 SIB1
```

### 1.2 SIB1 广播流程

```
6. openair2/LAYER2/NR_MAC_gNB/nr_radio_config.c
   └─ get_SIB1_NR()
      └─ 创建 NR_BCCH_DL_SCH_Message_t
      └─ sib1->cellAccessRelatedInfo.plmn_IdentityInfoList
         └─ asn1cSequenceAdd(..., NR_PLMN_IdentityInfo, nr_plmn_info)
            └─ for (i = 0; i < num_plmn; i++)  // ⚠️ 当前只循环1次
               └─ 填充 plmn_IdentityList (MCC/MNC)
               └─ 填充 trackingAreaCode
               └─ 填充 cellIdentity

7. openair2/LAYER2/NR_MAC_gNB/gNB_scheduler_bch.c
   └─ schedule_nr_sib1()
      └─ 将 SIB1 编码并通过 PDSCH 发送

8. **PHY层广播 SIB1**
   └─ UE接收到包含多个PLMN的SIB1
```

---

## 二、上行调用链：UE选择PLMN → gNB AMF选择

### 2.1 UE侧：接收SIB1并选择PLMN

```
1. openair2/LAYER2/NR_MAC_UE/main.c
   └─ nr_ue_get_sdu() / MAC调度器
      └─ 接收 BCCH-DL-SCH (SIB1)

2. openair2/LAYER2/NR_MAC_UE/nr_ue_procedures.c
   └─ nr_ue_decode_BCCH_DL_SCH()
      └─ 解码 SIB1
      └─ 发送给 RRC: NR_MAC_RRC_CONFIG_SIB1

3. openair2/RRC/NR_UE/L2_interface_ue.c
   └─ process_msg_rcc_to_mac()
      └─ case NR_MAC_RRC_CONFIG_SIB1:
         └─ nr_rrc_mac_config_req_sib1(instance_id, 0, sib1, ...)

4. openair2/LAYER2/NR_MAC_UE/config_ue.c
   └─ nr_rrc_mac_config_req_sib1()
      └─ 解析 sib1->cellAccessRelatedInfo.plmn_IdentityInfoList
      ⚠️ **UE在这里可以看到所有广播的PLMNs**
      └─ 存储到 MAC/RRC 内部结构

5. openair2/RRC/NR_UE/rrc_UE.c
   └─ nr_rrc_ue_establish_rrc_connection()
      └─ rrc->selected_plmn_identity = 1;  
      ⚠️ **当前硬编码选择第1个PLMN（第1264行）**
      ⚠️ **需要改成：让UE根据NAS层请求选择正确的PLMN**
```

### 2.2 UE发送RRCSetupComplete，携带选择的PLMN

```
6. openair2/RRC/NR/MESSAGES/asn1_msg.c
   └─ do_NR_RRCSetupComplete()
      └─ ies->selectedPLMN_Identity = sel_plmn_id;  
         // sel_plmn_id 是1-based索引（1表示第1个PLMN，2表示第2个，等等）

7. **UE通过DCCH发送 RRCSetupComplete**
   └─ RRCSetupComplete 包含：
      - dedicatedNAS_Message (NAS初始消息)
      - selectedPLMN_Identity (UE选择的PLMN索引，1-based)
      - registeredAMF (如果有5G-S-TMSI的话，包含之前的AMF信息)
```

### 2.3 gNB侧：接收RRCSetupComplete，提取PLMN

```
8. openair2/RRC/NR/rrc_gNB.c
   └─ rrc_gNB_decode_dcch()
      └─ case NR_UL_DCCH_MessageType__c1_PR_rrcSetupComplete:
         └─ rrc_gNB_process_RRCSetupComplete()

9. openair2/RRC/NR/rrc_gNB.c
   └─ rrc_gNB_process_RRCSetupComplete()
      └─ 解析 rrcSetupComplete->selectedPLMN_Identity
      └─ 调用 rrc_gNB_send_NGAP_NAS_FIRST_REQ()

10. openair2/RRC/NR/rrc_gNB_NGAP.c
    └─ rrc_gNB_send_NGAP_NAS_FIRST_REQ()
       └─ int idx = rrcSetupComplete->selectedPLMN_Identity - 1;  
          // 转换为0-based索引
       └─ **第248行：检查索引范围**
          if (idx < 0 || idx >= rrc->configuration.num_plmn) {
              LOG_E(NGAP, "selected PLMN index (%d) out of bounds\n", idx);
              return;
          }
       └─ **第254行：从配置中选择对应的PLMN**
          req->plmn = rrc->configuration.plmn[idx];
       └─ UE->serving_plmn = req->plmn;  // 存储UE当前的serving PLMN
       └─ 创建 NGAP_NAS_FIRST_REQ 消息，发送给NGAP层
```

### 2.4 NGAP层：根据PLMN选择AMF

```
11. openair3/NGAP/ngap_gNB_nas_procedures.c
    └─ ngap_gNB_handle_nas_first_req()
       └─ select_amf(instance_p, UEfirstReq)

12. openair3/NGAP/ngap_gNB_nas_procedures.c
    └─ select_amf()  // 第70行开始
       └─ AMF选择逻辑（按优先级）：
          
          **优先级1：如果UE提供了GUAMI**
          └─ if (msg->ue_identity.presenceMask & NGAP_UE_IDENTITIES_guami)
             └─ ngap_gNB_nnsf_select_amf_by_guami()
          
          **优先级2：如果UE提供了5G-S-TMSI**
          └─ if (msg->ue_identity.presenceMask & NGAP_UE_IDENTITIES_FiveG_s_tmsi)
             └─ ngap_gNB_nnsf_select_amf_by_amf_setid(
                   instance_p, 
                   msg->establishment_cause, 
                   msg->plmn,  // ⚠️ 使用UE选择的PLMN
                   fgs_tmsi->amf_set_id)
          
          **优先级3：仅根据PLMN选择** ⚠️ **MOCN的关键**
          └─ ngap_gNB_nnsf_select_amf_by_plmn_id(
                instance_p, 
                msg->establishment_cause, 
                msg->plmn)  // ⚠️ 使用UE选择的PLMN
          
          **优先级4：选择容量最高的AMF**
          └─ ngap_gNB_nnsf_select_amf(instance_p, msg->establishment_cause)

13. openair3/NGAP/ngap_gNB_nnsf.c
    └─ ngap_gNB_nnsf_select_amf_by_plmn_id()  // 第90行
       ⚠️ **这是MOCN的核心函数**
       └─ 遍历所有已连接的AMF:
          RB_FOREACH(amf_data_p, ngap_amf_map, &instance_p->ngap_amf_head)
          
          └─ 检查AMF状态（连接、过载检查）
          
          └─ **关键匹配逻辑（第128-143行）：**
             STAILQ_FOREACH(guami_p, &amf_data_p->served_guami, next) {
                STAILQ_FOREACH(served_plmn_p, &guami_p->served_plmns, next) {
                   if ((served_plmn_p->mcc == selected_plmn_identity.mcc) && 
                       (served_plmn_p->mnc == selected_plmn_identity.mnc)) {
                       // 找到了匹配UE选择PLMN的AMF！
                       break;
                   }
                }
             }
          
          └─ 在所有匹配PLMN的AMF中，选择容量最高的：
             if (current_capacity < amf_data_p->relative_amf_capacity) {
                current_capacity = amf_data_p->relative_amf_capacity;
                amf_highest_capacity_p = amf_data_p;
             }
       
       └─ return amf_highest_capacity_p;

14. openair3/NGAP/ngap_gNB_nas_procedures.c
    └─ ngap_gNB_handle_nas_first_req()
       └─ 使用选中的AMF，构造NGAP InitialUEMessage
       └─ 发送给对应的AMF（通过SCTP关联）
```

---

## 三、需要修改的代码部分

### 3.1 配置层面（确保支持多PLMN配置）

**文件：** `openair2/GNB_APP/gnb_config.c` 或配置文件解析部分

**修改目标：**
```c
// 当前配置结构已支持多PLMN（PLMN_LIST_MAX_SIZE = 6）
typedef struct gNB_RrcConfigurationReq_s {
  uint32_t tac;
  plmn_id_t plmn[PLMN_LIST_MAX_SIZE];  // ✅ 已支持数组
  uint8_t num_plmn;                    // ✅ 已有计数器
  // ...
} gNB_RrcConfigurationReq;
```

**需要做的：**
- 在配置文件中添加多个PLMN条目
- 确保配置解析代码读取所有PLMN并填充到 `plmn[]` 数组
- 设置正确的 `num_plmn` 值

---

### 3.2 核心修改：get_SIB1_NR() 支持广播多个PLMNs

**文件：** `openair2/LAYER2/NR_MAC_gNB/nr_radio_config.c`

**当前代码（第2714-2738行）：**
```c
// cellAccessRelatedInfo
// TODO : Add support for more than one PLMN
int num_plmn = 1; // int num_plmn = configuration->num_plmn;
asn1cSequenceAdd(sib1->cellAccessRelatedInfo.plmn_IdentityInfoList.list, 
                 struct NR_PLMN_IdentityInfo, nr_plmn_info);
for (int i = 0; i < num_plmn; ++i) {
  asn1cSequenceAdd(nr_plmn_info->plmn_IdentityList.list, 
                   struct NR_PLMN_Identity, nr_plmn);
  // 填充第1个PLMN的 MCC/MNC
  // ...
}
NR_CELL_ID_TO_BIT_STRING(cellID, &nr_plmn_info->cellIdentity);
nr_plmn_info->trackingAreaCode = CALLOC(...);
// 填充TAC，只对应第1个PLMN
```

**问题分析：**
1. **结构错误**：当前代码只创建了1个 `NR_PLMN_IdentityInfo`，然后循环添加多个PLMN Identity
2. **3GPP规范**：`plmn_IdentityInfoList` 应该包含多个 `PLMN_IdentityInfo`，每个info包含：
   - `plmn_IdentityList`（一个或多个PLMN）
   - `trackingAreaCode`（TAC）
   - `cellIdentity`（Cell ID）
   - `cellReservedForOperatorUse`

**修改方案1（推荐）：每个运营商一个PLMN_IdentityInfo**

```c
// cellAccessRelatedInfo
int num_plmn = mac_config->num_plmn;  // ⚠️ 改：从配置读取
AssertFatal(num_plmn > 0 && num_plmn <= PLMN_LIST_MAX_SIZE, 
            "Invalid num_plmn: %d\n", num_plmn);

// 为每个PLMN创建一个 PLMN_IdentityInfo
for (int i = 0; i < num_plmn; ++i) {
  const plmn_id_t *plmn = &mac_config->plmn[i];  // ⚠️ 改：从配置数组获取
  
  // 为第i个PLMN创建 PLMN_IdentityInfo
  asn1cSequenceAdd(sib1->cellAccessRelatedInfo.plmn_IdentityInfoList.list, 
                   struct NR_PLMN_IdentityInfo, nr_plmn_info);
  
  // 每个 PLMN_IdentityInfo 包含1个PLMN Identity
  asn1cSequenceAdd(nr_plmn_info->plmn_IdentityList.list, 
                   struct NR_PLMN_Identity, nr_plmn);
  
  // 填充MCC
  asn1cCalloc(nr_plmn->mcc, mcc);
  int confMcc = plmn->mcc;
  asn1cSequenceAdd(mcc->list, NR_MCC_MNC_Digit_t, mcc0);
  *mcc0 = (confMcc / 100) % 10;
  asn1cSequenceAdd(mcc->list, NR_MCC_MNC_Digit_t, mcc1);
  *mcc1 = (confMcc / 10) % 10;
  asn1cSequenceAdd(mcc->list, NR_MCC_MNC_Digit_t, mcc2);
  *mcc2 = confMcc % 10;
  
  // 填充MNC
  int mnc = plmn->mnc;
  if (plmn->mnc_digit_length == 3) {
    asn1cSequenceAdd(nr_plmn->mnc.list, NR_MCC_MNC_Digit_t, mnc0);
    *mnc0 = (mnc / 100) % 10;
  }
  asn1cSequenceAdd(nr_plmn->mnc.list, NR_MCC_MNC_Digit_t, mnc1);
  *mnc1 = (mnc / 10) % 10;
  asn1cSequenceAdd(nr_plmn->mnc.list, NR_MCC_MNC_Digit_t, mnc2);
  *mnc2 = (mnc) % 10;
  
  // 填充Cell Identity（所有PLMN共享同一个cell）
  NR_CELL_ID_TO_BIT_STRING(cellID, &nr_plmn_info->cellIdentity);
  
  nr_plmn_info->cellReservedForOperatorUse = 
      NR_PLMN_IdentityInfo__cellReservedForOperatorUse_notReserved;
  
  // 填充TAC（所有PLMN共享同一个TAC，MOCN场景）
  nr_plmn_info->trackingAreaCode = CALLOC(1, sizeof(NR_TrackingAreaCode_t));
  AssertFatal(nr_plmn_info->trackingAreaCode != NULL, "out of memory\n");
  uint32_t tmp2 = htobe32(tac);
  nr_plmn_info->trackingAreaCode->buf = CALLOC(1, 3);
  AssertFatal(nr_plmn_info->trackingAreaCode->buf != NULL, "out of memory\n");
  memcpy(nr_plmn_info->trackingAreaCode->buf, ((char *)&tmp2) + 1, 3);
  nr_plmn_info->trackingAreaCode->size = 3;
  nr_plmn_info->trackingAreaCode->bits_unused = 0;
  
  LOG_I(MAC, "Added PLMN[%d] to SIB1: MCC=%03d MNC=%0*d\n", 
        i, plmn->mcc, plmn->mnc_digit_length, plmn->mnc);
}
```

**关键要点：**
- 外层循环 `num_plmn` 次，每次创建一个 `PLMN_IdentityInfo`
- 每个 `PLMN_IdentityInfo` 包含1个PLMN Identity
- Cell Identity和TAC在MOCN场景下通常相同（共享同一个cell）
- 如果不同运营商需要不同TAC，需要扩展配置结构

---

### 3.3 传递配置到 get_SIB1_NR()

**文件：** `openair2/LAYER2/NR_MAC_gNB/nr_radio_config.h`

**当前函数签名：**
```c
NR_BCCH_DL_SCH_Message_t *get_SIB1_NR(const NR_ServingCellConfigCommon_t *scc,
                                      const nr_mac_config_t *mac_config);
```

**问题：** `nr_mac_config_t` 结构体中**没有** `num_plmn` 和 `plmn[]` 字段！

**查看 nr_mac_config_t 定义（openair2/LAYER2/NR_MAC_gNB/nr_mac_gNB.h 第188行）：**
```c
typedef struct nr_mac_config_t {
  int sib1_tda;
  nr_pdsch_AntennaPorts_t pdsch_AntennaPorts;
  // ... 其他字段
  // ⚠️ 缺少 num_plmn 和 plmn[]！
} nr_mac_config_t;
```

**修改方案A：扩展 nr_mac_config_t**

```c
typedef struct nr_mac_config_t {
  // ... 现有字段
  
  // ⚠️ 新增：PLMN配置（MOCN支持）
  int num_plmn;
  plmn_id_t plmn[PLMN_LIST_MAX_SIZE];
  
  // ... 其他字段
} nr_mac_config_t;
```

**然后在调用 get_SIB1_NR() 之前，填充这些字段：**

```c
// 在 nr_mac_config_scc() 或初始化函数中
nr_mac_config_t *mac_config = &nrmac->radio_config;
mac_config->num_plmn = rrc->configuration.num_plmn;
for (int i = 0; i < mac_config->num_plmn; i++) {
  mac_config->plmn[i] = rrc->configuration.plmn[i];
}
```

**修改方案B：直接传递 RRC configuration**

或者，修改 `get_SIB1_NR()` 签名，直接传递包含PLMN信息的结构：

```c
// 在 nr_radio_config.h 中
NR_BCCH_DL_SCH_Message_t *get_SIB1_NR(
    const NR_ServingCellConfigCommon_t *scc,
    const nr_mac_config_t *mac_config,
    int num_plmn,                    // ⚠️ 新增
    const plmn_id_t *plmn_list);     // ⚠️ 新增
```

**推荐方案A**，因为更符合现有架构。

---

### 3.4 UE侧：选择PLMN（可选，当前默认选第1个）

**文件：** `openair2/RRC/NR_UE/rrc_UE.c`

**当前代码（第1264行）：**
```c
rrc->selected_plmn_identity = 1;  // 硬编码选择第1个PLMN
```

**MOCN场景下的修改：**

UE应该根据NAS层的请求选择PLMN，而不是硬编码。完整的PLMN选择应该包括：

```c
// 在 nr_rrc_mac_config_req_sib1() 中存储SIB1中的所有PLMNs
void nr_rrc_mac_config_req_sib1(module_id_t module_id, int cc_idP, 
                                NR_SIB1_t *sib1, bool can_start_ra)
{
  NR_UE_MAC_INST_t *mac = get_mac_inst(module_id);
  // ...
  
  // ⚠️ 新增：存储所有可用的PLMNs
  NR_PLMN_IdentityInfoList_t *plmn_info_list = 
      &sib1->cellAccessRelatedInfo.plmn_IdentityInfoList;
  
  LOG_I(NR_RRC, "SIB1 contains %d PLMN(s):\n", 
        plmn_info_list->list.count);
  
  for (int i = 0; i < plmn_info_list->list.count; i++) {
    NR_PLMN_IdentityInfo_t *plmn_info = plmn_info_list->list.array[i];
    if (plmn_info->plmn_IdentityList.list.count > 0) {
      NR_PLMN_Identity_t *plmn_id = plmn_info->plmn_IdentityList.list.array[0];
      
      // 解析MCC/MNC
      int mcc = 0, mnc = 0, mnc_digit_length = 2;
      if (plmn_id->mcc) {
        mcc = *plmn_id->mcc->list.array[0] * 100 +
              *plmn_id->mcc->list.array[1] * 10 +
              *plmn_id->mcc->list.array[2];
      }
      mnc_digit_length = plmn_id->mnc.list.count;
      if (mnc_digit_length == 2) {
        mnc = *plmn_id->mnc.list.array[0] * 10 +
              *plmn_id->mnc.list.array[1];
      } else {
        mnc = *plmn_id->mnc.list.array[0] * 100 +
              *plmn_id->mnc.list.array[1] * 10 +
              *plmn_id->mnc.list.array[2];
      }
      
      LOG_I(NR_RRC, "  PLMN[%d]: MCC=%03d MNC=%0*d\n", 
            i + 1, mcc, mnc_digit_length, mnc);
      
      // ⚠️ 存储到RRC/MAC结构中，供后续PLMN选择使用
      // mac->available_plmns[i] = {mcc, mnc, mnc_digit_length};
    }
  }
  
  // ...
}
```

```c
// 在 rrc_UE.c 中，建立RRC连接时
// 根据NAS层的请求选择PLMN
int nr_rrc_ue_establish_rrc_connection(...)
{
  // ...
  
  // ⚠️ 修改：从NAS层获取期望的PLMN，而不是硬编码
  // 如果NAS层没有指定，默认选择第1个
  if (nas_requested_plmn_available) {
    // 查找匹配NAS请求的PLMN索引
    for (int i = 0; i < num_available_plmns; i++) {
      if (available_plmns[i].mcc == nas_requested_plmn.mcc &&
          available_plmns[i].mnc == nas_requested_plmn.mnc) {
        rrc->selected_plmn_identity = i + 1;  // 1-based
        LOG_I(NR_RRC, "Selected PLMN[%d] matching NAS request: "
              "MCC=%03d MNC=%0*d\n", 
              rrc->selected_plmn_identity,
              available_plmns[i].mcc,
              available_plmns[i].mnc_digit_length,
              available_plmns[i].mnc);
        break;
      }
    }
  } else {
    // 默认选择第1个PLMN
    rrc->selected_plmn_identity = 1;
  }
  
  // ...
}
```

**注意：** 对于初始测试，可以保持 `selected_plmn_identity = 1`，只要gNB正确广播了多个PLMN即可。UE会默认选择第1个PLMN。

---

### 3.5 验证和日志

在关键函数中添加日志，验证MOCN流程：

**1. gNB SIB1生成（nr_radio_config.c）**
```c
LOG_I(MAC, "Generating SIB1 with %d PLMN(s)\n", num_plmn);
for (int i = 0; i < num_plmn; i++) {
  LOG_I(MAC, "  PLMN[%d]: MCC=%03d MNC=%0*d\n", 
        i, plmn[i].mcc, plmn[i].mnc_digit_length, plmn[i].mnc);
}
```

**2. gNB接收RRCSetupComplete（rrc_gNB_NGAP.c）**
```c
LOG_I(NGAP, "UE selected PLMN index=%d (MCC=%03d MNC=%0*d)\n",
      idx, req->plmn.mcc, req->plmn.mnc_digit_length, req->plmn.mnc);
```

**3. AMF选择（ngap_gNB_nnsf.c）**
```c
// 在 ngap_gNB_nnsf_select_amf_by_plmn_id() 中
LOG_I(NGAP, "Selecting AMF for UE-selected PLMN: MCC=%03d MNC=%0*d\n",
      selected_plmn_identity.mcc, 
      selected_plmn_identity.mnc_digit_length,
      selected_plmn_identity.mnc);

// 匹配成功时
LOG_I(NGAP, "AMF '%s' serves PLMN MCC=%03d MNC=%0*d (capacity=%d)\n",
      amf_data_p->amf_name,
      served_plmn_p->mcc,
      served_plmn_p->mnc_digit_length,
      served_plmn_p->mnc,
      amf_data_p->relative_amf_capacity);

// 最终选择
if (amf_highest_capacity_p) {
  LOG_I(NGAP, "✅ Selected AMF '%s' for PLMN MCC=%03d MNC=%0*d\n",
        amf_highest_capacity_p->amf_name,
        selected_plmn_identity.mcc,
        selected_plmn_identity.mnc_digit_length,
        selected_plmn_identity.mnc);
}
```

---

## 四、测试步骤

### 4.1 配置两个PLMNs

修改gNB配置文件（例如 `gnb.conf`）：

```conf
gNBs = {
  plmn_list = (
    {
      mcc = 208;
      mnc = 95;
      mnc_length = 2;
    },
    {
      mcc = 208;
      mnc = 96;
      mnc_length = 2;
    }
  );
  
  tracking_area_code = 1;
  // ...
}
```

### 4.2 配置两个AMF连接

确保gNB配置中连接到两个AMF，每个AMF服务不同的PLMN：

```conf
NGAP_CONFIGURATION = {
  AMF_LIST = (
    {
      AMF_NAME = "AMF-Operator1";
      AMF_IPV4_ADDRESS = "192.168.1.100";
      AMF_PORT = 38412;
      SERVED_GUAMI = (
        {
          MCC = 208;
          MNC = 95;
          AMF_REGION_ID = 128;
          AMF_SET_ID = 1;
          AMF_POINTER = 1;
        }
      );
    },
    {
      AMF_NAME = "AMF-Operator2";
      AMF_IPV4_ADDRESS = "192.168.1.101";
      AMF_PORT = 38412;
      SERVED_GUAMI = (
        {
          MCC = 208;
          MNC = 96;
          AMF_REGION_ID = 128;
          AMF_SET_ID = 2;
          AMF_POINTER = 1;
        }
      );
    }
  );
}
```

### 4.3 验证流程

1. **启动gNB，检查日志：**
   ```
   [MAC] Generating SIB1 with 2 PLMN(s)
   [MAC]   PLMN[0]: MCC=208 MNC=95
   [MAC]   PLMN[1]: MCC=208 MNC=96
   ```

2. **启动UE，让其接入网络**

3. **在gNB日志中查看UE选择的PLMN：**
   ```
   [NGAP] UE selected PLMN index=0 (MCC=208 MNC=95)
   或
   [NGAP] UE selected PLMN index=1 (MCC=208 MNC=96)
   ```

4. **在AMF选择日志中验证：**
   ```
   [NGAP] Selecting AMF for UE-selected PLMN: MCC=208 MNC=95
   [NGAP] AMF 'AMF-Operator1' serves PLMN MCC=208 MNC=95 (capacity=255)
   [NGAP] ✅ Selected AMF 'AMF-Operator1' for PLMN MCC=208 MNC=95
   ```

5. **抓包验证SIB1内容：**
   使用Wireshark抓取BCCH-DL-SCH消息，解码SIB1，确认 `cellAccessRelatedInfo.plmn-IdentityInfoList` 包含2个条目。

6. **测试不同PLMN的UE：**
   - 配置UE1选择PLMN 208/95，验证连接到AMF-Operator1
   - 配置UE2选择PLMN 208/96，验证连接到AMF-Operator2

---

## 五、总结

### 关键修改点：

| 位置 | 文件 | 函数 | 修改内容 |
|------|------|------|----------|
| 1 | `nr_radio_config.c` | `get_SIB1_NR()` | 第2714行：改 `int num_plmn = 1` 为从配置读取，循环创建多个`PLMN_IdentityInfo` |
| 2 | `nr_mac_gNB.h` | `nr_mac_config_t` | 新增 `int num_plmn` 和 `plmn_id_t plmn[]` 字段 |
| 3 | `rrc_gNB.c` | 初始化函数 | 将RRC配置的PLMN信息传递给MAC层 |
| 4 | `rrc_UE.c` | UE PLMN选择 | （可选）改进UE的PLMN选择逻辑，支持NAS请求 |

### 已经正确工作的部分：

✅ **RRC层**：`rrc_gNB_send_NGAP_NAS_FIRST_REQ()` 已正确提取UE选择的PLMN  
✅ **NGAP层**：`ngap_gNB_nnsf_select_amf_by_plmn_id()` 已实现根据PLMN匹配AMF  
✅ **配置结构**：`gNB_RrcConfigurationReq` 已支持多PLMN数组  

### 最小改动方案（第一步）：

**只需修改1个文件，1个函数，约20行代码：**

📝 `openair2/LAYER2/NR_MAC_gNB/nr_radio_config.c` 中的 `get_SIB1_NR()`：
- 从配置读取 `num_plmn`
- 外层循环创建多个 `PLMN_IdentityInfo`
- 每个info填充对应的PLMN、TAC、Cell ID

这样就能实现SIB1广播多个PLMN，UE可以看到并选择，整个上行流程（PLMN选择→AMF选择）已经ready！

---

## 附录：相关数据结构

### plmn_id_t
```c
typedef struct plmn_identity_s {
  uint16_t mcc;                 // Mobile Country Code
  uint16_t mnc;                 // Mobile Network Code
  uint8_t  mnc_digit_length;    // 2 or 3
} plmn_id_t;
```

### NR_PLMN_IdentityInfo (ASN.1结构)
```
PLMN-IdentityInfo ::= SEQUENCE {
    plmn-IdentityList           PLMN-IdentityList,
    trackingAreaCode            TrackingAreaCode OPTIONAL,
    ranac                       RAN-AreaCode OPTIONAL,
    cellIdentity                CellIdentity,
    cellReservedForOperatorUse  ENUMERATED {reserved, notReserved},
    ...
}

PLMN-IdentityList ::= SEQUENCE (SIZE (1..maxPLMN)) OF PLMN-Identity

maxPLMN INTEGER ::= 12
```

### MOCN场景示例：
```
SIB1.cellAccessRelatedInfo.plmn-IdentityInfoList:
  [0]: PLMN-IdentityInfo
        plmn-IdentityList: [MCC=208, MNC=95]
        trackingAreaCode: 0x0001
        cellIdentity: 0x12345600
        cellReservedForOperatorUse: notReserved
  [1]: PLMN-IdentityInfo
        plmn-IdentityList: [MCC=208, MNC=96]
        trackingAreaCode: 0x0001  // Same TAC (shared cell)
        cellIdentity: 0x12345600  // Same Cell ID (shared cell)
        cellReservedForOperatorUse: notReserved
```
