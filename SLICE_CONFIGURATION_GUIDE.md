# 5G Network Slicing Configuration Guide

## 问题1：Slice List在代码中的使用位置和作用

### 1.1 按3GPP Rel-17标准的Slice使用流程

根据3GPP TS 23.501（5G系统架构）和TS 38.413（NG-AP协议）：

#### **A. gNB侧的Slice信息传递**

1. **F1AP Setup Request (DU → CU)**
   - **位置**: `openair2/F1AP/lib/f1ap_interface_management.c:488`
   - **函数**: `write_slice_info(c->num_ssi, c->nssai)`
   - **作用**: DU通过F1AP Setup Request向CU报告每个PLMN支持的Slice列表（TAI Slice Support List）
   - **3GPP规范**: TS 38.473 §9.3.1.10 Served Cell Information
   ```c
   // F1AP编码：每个PLMN的slice列表
   servedPLMN_item->iE_Extensions = write_slice_info(c->num_ssi, c->nssai);
   ```

2. **NG Setup Request (gNB → AMF)**
   - **位置**: `openair3/NGAP/ngap_gNB.c:727-741`
   - **函数**: NG Setup消息编码
   - **作用**: gNB向AMF报告每个PLMN支持的Slice列表
   - **3GPP规范**: TS 38.413 §9.2.6.1 NG SETUP REQUEST
   ```c
   for (int si = 0; si < plmn_req->num_nssai; si++) {
     ssi = calloc(1, sizeof(NGAP_SliceSupportItem_t));
     INT8_TO_OCTET_STRING(plmn_req->s_nssai[si].sst, &ssi->s_NSSAI.sST);
     const uint32_t sd = plmn_req->s_nssai[si].sd;
     if (sd != 0xFFFFFF) {  // SD有效
       // 编码SD（3字节）
       ssi->s_NSSAI.sD->buf[0] = (sd & 0xff0000) >> 16;
       ssi->s_NSSAI.sD->buf[1] = (sd & 0x00ff00) >> 8;
       ssi->s_NSSAI.sD->buf[2] = (sd & 0x0000ff);
     }
     asn1cSeqAdd(&plmn->tAISliceSupportList.list, ssi);
   }
   ```

#### **B. UE侧的Slice使用**

1. **UE Configuration (静态配置)**
   - **位置**: `ue.conf`文件
   - **字段**: `nssai_sst`, `nssai_sd`
   - **作用**: UE预配置的Slice信息，用于PDU Session建立
   ```conf
   uicc0 = {
     imsi = "208930000000003";
     nssai_sst = 1;
     nssai_sd = 0x010203;
   }
   ```

2. **Registration Request (UE → AMF)**
   - **位置**: `openair3/NAS/NR_UE/nr_nas_msg.c`
   - **作用**: UE在Registration Request中可以携带Requested NSSAI
   - **3GPP规范**: TS 24.501 §5.5.1.2.2

3. **Registration Accept (AMF → UE)**
   - **位置**: `openair3/NAS/NR_UE/nr_nas_msg.c:1791`
   - **函数**: `get_user_nssai_idx()`
   - **作用**: AMF下发**Allowed NSSAI**，UE匹配自己配置的NSSAI
   - **3GPP规范**: TS 24.501 §5.5.1.2.4
   ```c
   static int get_user_nssai_idx(const nr_nas_msg_snssai_t allowed_nssai[NAS_MAX_NUMBER_SLICES], 
                                  const nr_ue_nas_t *nas)
   {
     for (int i = 0; i < NAS_MAX_NUMBER_SLICES; i++) {
       const nr_nas_msg_snssai_t *nssai = allowed_nssai + i;
       bool sd_match = !nssai->sd || (nas->uicc->nssai_sd == *nssai->sd);
       if ((nas->uicc->nssai_sst == nssai->sst) && sd_match)
         return i;
     }
     return -1;
   }
   ```

4. **PDU Session Establishment Request (UE → SMF)**
   - **位置**: `openair3/NAS/NR_UE/nr_nas_msg.c:1694-1704`
   - **作用**: UE请求PDU Session时携带S-NSSAI，指定使用哪个Slice
   - **3GPP规范**: TS 24.501 §8.3.2
   ```c
   void request_pdusession(nr_ue_nas_t *nas, int pdusession_id)
   {
     NAS_PDU_SESSION_REQ(message_p).sst = nas->uicc->nssai_sst;
     NAS_PDU_SESSION_REQ(message_p).sd = nas->uicc->nssai_sd;
     // 编码到NAS消息
     mm_msg->snssai.value[0] = pdu_req->sst;  // SST
     if (has_nssai_sd)
       INT24_TO_BUFFER(pdu_req->sd, &mm_msg->snssai.value[1]);  // SD
   }
   ```

#### **C. RRC/PDCP层的Slice关联**

1. **DRB与Slice关联**
   - **位置**: gNB RRC处理NGAP PDU Session Setup Request
   - **作用**: 每个DRB关联一个PDU Session，每个PDU Session关联一个S-NSSAI
   - **数据流**: AMF → gNB (NGAP) → CU-CP (F1AP) → CU-UP (E1AP) → DU

### 1.2 OAI代码中当前的nssai[]数组使用

**当前存在的问题（你发现的）：**

```c
typedef struct f1ap_served_cell_info_t {
  plmn_id_t plmn;
  uint64_t nr_cellid;
  
  uint16_t num_ssi;                    // 全局slice计数
  nssai_t nssai[MAX_NUM_SLICES];      // ❌ 所有PLMN共享
  
  uint16_t num_plmn;
  plmn_id_t plmn_list[F1AP_MAX_NB_PLMNS];  // PLMN列表
}
```

**使用位置：**
1. `openair2/GNB_APP/gnb_config.c:1114` - 从配置文件读取slice信息
2. `openair2/F1AP/lib/f1ap_interface_management.c:488` - F1AP Setup编码
3. `openair3/NGAP/ngap_gNB.c:727` - NG Setup编码

**问题：** 无法区分哪些slice属于哪个PLMN！

---

## 问题2：按3GPP Rel-17规范，Slice ID应该如何配置？

### 2.1 5G Slice配置的标准架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    3GPP 5G Slice 配置层次                        │
└─────────────────────────────────────────────────────────────────┘

Layer 1: Subscriber (UE) Configuration
├─ UE UICC/SIM卡配置
│  ├─ Configured NSSAI (预配置的slice列表)
│  │  └─ 用途：UE开机后可请求的slice
│  └─ Default NSSAI (默认slice)
│     └─ 用途：没有allowed NSSAI时使用
│
├─ OAI UE配置文件 (ue.conf)
│  └─ nssai_sst, nssai_sd
│     └─ 用途：模拟UE的slice配置

Layer 2: RAN (gNB) Configuration
├─ gNB配置文件 (gnb.conf)
│  └─ plmn_list[].snssaiList[]
│     ├─ 用途：声明gNB支持的slice（per PLMN）
│     └─ 传递路径：gNB → AMF (NG Setup)
│
├─ F1AP层
│  └─ Served Cell Info → TAI Slice Support List
│     └─ 用途：DU向CU报告支持的slice

Layer 3: Core Network Configuration
├─ AMF配置 (amfcfg.yaml)
│  ├─ supportedNSSAI[]
│  │  └─ 用途：AMF支持的slice列表
│  └─ plmnSupportList[].plmnId + snssaiList[]
│     └─ 用途：每个PLMN的slice配置
│
├─ NSSF配置 (nssf.yaml)
│  └─ nsiInformationList[]
│     └─ 用途：Slice选择策略
│
├─ SMF配置 (smfcfg.yaml)
│  └─ snssaiInfos[]
│     └─ 用途：SMF管理的slice和DNN映射
│
└─ 5GC GUI (Subscriber管理)
   └─ Subscriber → Slice ID
      └─ 用途：订阅关系，AMF鉴权时检查
```

### 2.2 完整的Slice配置示例

#### **场景：Multi-PLMN with Different Slices**

```yaml
# ========================================
# 网络拓扑
# ========================================
# PLMN 1: 208/93 (中国移动模拟)
#   - Slice 1: SST=1, SD=0x010203 (eMBB)
#   - Slice 2: SST=2, SD=0x020304 (URLLC)
#
# PLMN 2: 460/11 (中国联通模拟)
#   - Slice 1: SST=1, SD=0x010204 (eMBB)
#   - Slice 3: SST=3, SD=0x030405 (MIoT)
```

#### **A. gNB配置 (gnb.conf)**

```conf
plmn_list = (
  { 
    mcc = 208; 
    mnc = 93; 
    mnc_length = 2; 
    snssaiList = (
      { sst = 1; sd = 0x010203; },  # eMBB slice
      { sst = 2; sd = 0x020304; }   # URLLC slice
    ); 
  },
  { 
    mcc = 460; 
    mnc = 11; 
    mnc_length = 2; 
    snssaiList = (
      { sst = 1; sd = 0x010204; },  # eMBB slice (不同SD)
      { sst = 3; sd = 0x030405; }   # MIoT slice
    ); 
  }
);
```

#### **B. UE配置 (ue.conf)**

```conf
# UE使用PLMN 208/93的eMBB slice
uicc0 = {
  imsi = "208930000000003";
  key = "8baf473f2f8fd09487cccbd7097c6862";
  opc = "8e27b6af0e692e750f32667a3b14605d";
  dnn = "internet";
  nssai_sst = 1;          # 请求eMBB slice
  nssai_sd = 0x010203;    # PLMN 208/93的eMBB
}
```

#### **C. AMF配置 (amfcfg.yaml)**

```yaml
plmnSupportList:
  - plmnId:
      mcc: "208"
      mnc: "93"
    snssaiList:
      - sst: 1
        sd: "010203"  # eMBB
      - sst: 2
        sd: "020304"  # URLLC
  - plmnId:
      mcc: "460"
      mnc: "11"
    snssaiList:
      - sst: 1
        sd: "010204"  # eMBB (不同SD)
      - sst: 3
        sd: "030405"  # MIoT

supportedNSSAI:
  - sst: 1
    sd: "010203"
  - sst: 2
    sd: "020304"
  - sst: 1
    sd: "010204"
  - sst: 3
    sd: "030405"
```

#### **D. SMF配置 (smfcfg.yaml)**

```yaml
snssaiInfos:
  - sNssai:
      sst: 1
      sd: "010203"
    dnnInfos:
      - dnn: "internet"
  - sNssai:
      sst: 2
      sd: "020304"
    dnnInfos:
      - dnn: "internet"
  - sNssai:
      sst: 1
      sd: "010204"
    dnnInfos:
      - dnn: "internet"
  - sNssai:
      sst: 3
      sd: "030405"
    dnnInfos:
      - dnn: "iot"
```

#### **E. 5GC Subscriber配置 (通过GUI或MongoDB)**

```json
{
  "imsi": "208930000000003",
  "plmnId": "20893",
  "ueId": "imsi-208930000000003",
  "AuthenticationSubscription": {
    "permanentKey": {
      "permanentKeyValue": "8baf473f2f8fd09487cccbd7097c6862"
    },
    "opc": {
      "opcValue": "8e27b6af0e692e750f32667a3b14605d"
    }
  },
  "SessionManagementSubscriptionData": [
    {
      "singleNssai": {
        "sst": 1,
        "sd": "010203"
      },
      "dnnConfigurations": {
        "internet": {
          "pduSessionTypes": {
            "defaultSessionType": "IPV4"
          },
          "sessionAmbr": {
            "uplink": "100 Mbps",
            "downlink": "100 Mbps"
          }
        }
      }
    }
  ]
}
```

### 2.3 Slice建立过程（完整流程）

```
UE                gNB               AMF               SMF               UPF
│                  │                 │                 │                 │
│ 1. Registration Request          │                 │                 │
│  (Requested NSSAI: SST=1, SD=010203)                │                 │
├─────────────────>│                 │                 │                 │
│                  │ 2. Initial UE Message             │                 │
│                  │  (Serving PLMN, Requested NSSAI)  │                 │
│                  ├────────────────>│                 │                 │
│                  │                 │ 3. AMF验证:      │                 │
│                  │                 │  - PLMN是否支持  │                 │
│                  │                 │  - Subscriber权限│                 │
│                  │                 │  - 生成Allowed NSSAI              │
│                  │                 │                 │                 │
│ 4. Registration Accept            │                 │                 │
│  (Allowed NSSAI: SST=1, SD=010203) │                 │                 │
│<─────────────────┼─────────────────┤                 │                 │
│                  │                 │                 │                 │
│ 5. PDU Session Establishment Request                │                 │
│  (S-NSSAI: SST=1, SD=010203, DNN=internet)          │                 │
├─────────────────>│                 │                 │                 │
│                  │ 6. PDU Session Resource Setup Request               │
│                  │  (S-NSSAI, QoS Flows)             │                 │
│                  ├────────────────>│                 │                 │
│                  │                 │ 7. Nsmf_PDUSession_CreateSMContext│
│                  │                 ├────────────────>│                 │
│                  │                 │                 │ 8. SMF检查:    │
│                  │                 │                 │  - S-NSSAI配置  │
│                  │                 │                 │  - DNN映射     │
│                  │                 │                 │  - QoS策略     │
│                  │                 │                 │                 │
│                  │                 │                 │ 9. N4 Session  │
│                  │                 │                 ├───────────────>│
│                  │                 │                 │                 │
│                  │ 10. PDU Session Resource Setup Response             │
│                  │  (DRB Config, UP TNL Info)        │                 │
│                  │<────────────────┼─────────────────┤                 │
│                  │                 │                 │                 │
│ 11. RRCReconfiguration (DRB Setup)│                 │                 │
│<─────────────────┤                 │                 │                 │
│                  │                 │                 │                 │
│ 12. Data传输 (通过DRB, QoS基于S-NSSAI)               │                 │
│<═════════════════╪═════════════════╪═════════════════╪═══════════════>│
```

**关键验证点：**
1. **NG Setup**: gNB告诉AMF支持哪些(PLMN, S-NSSAI)组合
2. **Registration**: AMF根据gNB能力和订阅信息生成Allowed NSSAI
3. **PDU Session**: SMF检查S-NSSAI是否在Allowed NSSAI中
4. **DRB建立**: QoS映射到S-NSSAI对应的slice

---

## 问题3：如果增加served_plmn_list，应该如何修改？

### 3.1 当前结构的问题总结

```c
// ❌ 错误的设计（当前）
typedef struct f1ap_served_cell_info_t {
  plmn_id_t plmn;  // Global Cell ID的PLMN
  uint64_t nr_cellid;
  
  uint16_t num_ssi;                // 所有PLMN共享的slice计数
  nssai_t nssai[MAX_NUM_SLICES];  // 所有PLMN共享的slice数组
  
  uint16_t num_plmn;
  plmn_id_t plmn_list[F1AP_MAX_NB_PLMNS];  // PLMN列表
  
  // ❌ 问题：无法区分nssai[0]属于哪个PLMN！
}
```

### 3.2 正确的结构设计

```c
// ✅ 正确的设计（Robert建议）
typedef struct f1ap_plmn_info_t {
  plmn_id_t plmn;
  uint16_t num_nssai;
  nssai_t nssai[MAX_NUM_SLICES];  // 这个PLMN的slice列表
} f1ap_plmn_info_t;

typedef struct f1ap_served_cell_info_t {
  // NR CGI - 保留（用于Global Cell ID）
  plmn_id_t plmn;
  uint64_t nr_cellid;
  
  uint16_t nr_pci;
  uint32_t *tac;
  
  // PLMN列表（每个PLMN带自己的slice信息）
  uint16_t num_plmn;
  f1ap_plmn_info_t plmn_list[F1AP_MAX_NB_PLMNS];
  
  // ✅ 优势：数据绑定，plmn_list[i]包含PLMN和其slice列表
  
  f1ap_mode_t mode;
  // ...
}
```

### 3.3 需要修改的代码位置

#### **A. 配置文件读取 (gnb_config.c)**

**当前代码**（需要修改）：
```c
// openair2/GNB_APP/gnb_config.c:1114
info->num_ssi = set_snssai_config(info->nssai, MAX_NUM_SLICES, 0, 0);
```

**修改后**：
```c
// 读取每个PLMN的slice配置
for (int p = 0; p < info->num_plmn; p++) {
  f1ap_plmn_info_t *plmn_info = &info->plmn_list[p];
  // 读取PLMN ID
  plmn_info->plmn.mcc = ...;
  plmn_info->plmn.mnc = ...;
  plmn_info->plmn.mnc_digit_length = ...;
  // 读取这个PLMN的slice列表
  plmn_info->num_nssai = set_snssai_config(
    plmn_info->nssai, 
    MAX_NUM_SLICES, 
    0,  // gNB index
    p   // PLMN index
  );
}
```

#### **B. F1AP编码 (f1ap_interface_management.c)**

**当前代码**（需要修改）：
```c
// Line 488
servedPLMN_item->iE_Extensions = write_slice_info(c->num_ssi, c->nssai);
```

**修改后**：
```c
// 为每个PLMN编码其slice列表
for (int p = 0; p < c->num_plmn; p++) {
  asn1cSequenceAdd(scell_info.servedPLMNs.list, 
                   F1AP_ServedPLMNs_Item_t, servedPLMN_item);
  
  const f1ap_plmn_info_t *plmn_info = &c->plmn_list[p];
  
  // 编码PLMN ID
  MCC_MNC_TO_PLMNID(plmn_info->plmn.mcc, 
                    plmn_info->plmn.mnc, 
                    plmn_info->plmn.mnc_digit_length, 
                    &servedPLMN_item->pLMN_Identity);
  
  // 编码这个PLMN的slice列表
  servedPLMN_item->iE_Extensions = write_slice_info(
    plmn_info->num_nssai, 
    plmn_info->nssai
  );
}
```

#### **C. F1AP解码 (f1ap_interface_management.c)**

**当前位置**: `decode_served_cell_info()` 函数

**修改**: 解码Served PLMNs列表，每个PLMN带其slice信息

```c
static bool decode_served_cell_info(const F1AP_Served_Cell_Information_t *in, 
                                    f1ap_served_cell_info_t *info)
{
  // ... existing code for NR CGI, PCI, TAC ...
  
  // 解码Served PLMNs
  info->num_plmn = in->servedPLMNs.list.count;
  for (int p = 0; p < info->num_plmn; p++) {
    const F1AP_ServedPLMNs_Item_t *servedPLMN = in->servedPLMNs.list.array[p];
    f1ap_plmn_info_t *plmn_info = &info->plmn_list[p];
    
    // 解码PLMN ID
    PLMNID_TO_MCC_MNC(&servedPLMN->pLMN_Identity,
                      plmn_info->plmn.mcc,
                      plmn_info->plmn.mnc,
                      plmn_info->plmn.mnc_digit_length);
    
    // 解码slice列表
    if (servedPLMN->iE_Extensions) {
      plmn_info->num_nssai = read_slice_info(
        servedPLMN->iE_Extensions,
        plmn_info->nssai
      );
    } else {
      plmn_info->num_nssai = 0;
    }
  }
  
  return true;
}
```

#### **D. NGAP层 (ngap_gNB.c)**

**当前代码**（需要修改）：
```c
// Line 727
for (int si = 0; si < plmn_req->num_nssai; si++) {
  // 编码slice
}
```

**修改后**：
```c
// 为每个PLMN编码其slice列表
for (int p = 0; p < info->num_plmn; p++) {
  const f1ap_plmn_info_t *plmn_info = &info->plmn_list[p];
  
  // 添加Supported TA Item
  asn1cSequenceAdd(&supportedTAList->list, 
                   NGAP_SupportedTAItem_t, ta_item);
  
  // 添加Broadcast PLMN Item
  asn1cSequenceAdd(&ta_item->broadcastPLMNList.list,
                   NGAP_BroadcastPLMNItem_t, plmn);
  
  // 编码PLMN ID
  MCC_MNC_TO_PLMNID(plmn_info->plmn.mcc,
                    plmn_info->plmn.mnc,
                    plmn_info->plmn.mnc_digit_length,
                    &plmn->pLMN_Identity);
  
  // 编码TAI Slice Support List
  for (int si = 0; si < plmn_info->num_nssai; si++) {
    NGAP_SliceSupportItem_t *ssi = calloc(1, sizeof(*ssi));
    INT8_TO_OCTET_STRING(plmn_info->nssai[si].sst, &ssi->s_NSSAI.sST);
    
    const uint32_t sd = plmn_info->nssai[si].sd;
    if (sd != 0xFFFFFF) {
      ssi->s_NSSAI.sD = calloc(1, sizeof(NGAP_SD_t));
      ssi->s_NSSAI.sD->buf = calloc(3, sizeof(uint8_t));
      ssi->s_NSSAI.sD->size = 3;
      ssi->s_NSSAI.sD->buf[0] = (sd & 0xff0000) >> 16;
      ssi->s_NSSAI.sD->buf[1] = (sd & 0x00ff00) >> 8;
      ssi->s_NSSAI.sD->buf[2] = (sd & 0x0000ff);
    }
    
    asn1cSeqAdd(&plmn->tAISliceSupportList.list, ssi);
  }
}
```

#### **E. 单元测试 (f1ap_lib_test.c)**

**位置**: `openair2/F1AP/tests/f1ap_lib_test.c:232-234`

**当前代码**：
```c
.num_ssi = 1,
.nssai[0].sst = 1,
.nssai[0].sd = 1,
```

**修改后**：
```c
.num_plmn = 2,
.plmn_list = {
  {
    .plmn = {.mcc = 208, .mnc = 93, .mnc_digit_length = 2},
    .num_nssai = 2,
    .nssai = {
      {.sst = 1, .sd = 0x010203},  // eMBB
      {.sst = 2, .sd = 0x020304},  // URLLC
    }
  },
  {
    .plmn = {.mcc = 460, .mnc = 11, .mnc_digit_length = 2},
    .num_nssai = 2,
    .nssai = {
      {.sst = 1, .sd = 0x010204},  // eMBB
      {.sst = 3, .sd = 0x030405},  // MIoT
    }
  }
},
```

### 3.4 需要注意的关键点

1. **保留第一个`plmn`字段**
   - 原因：NR CGI (Global Cell ID) = PLMN + Cell ID
   - 不能删除，必须保留用于小区标识

2. **向后兼容性**
   - 删除`num_ssi`和`nssai[]`数组
   - 所有引用这两个字段的代码都需要修改

3. **F1AP协议约束**
   - 每个Served PLMN可以有自己的slice列表
   - 符合3GPP TS 38.473 §9.3.1.10规范

4. **NGAP协议约束**
   - NG Setup时每个Broadcast PLMN带TAI Slice Support List
   - 符合3GPP TS 38.413 §9.2.6.1规范

5. **编译和测试**
   ```bash
   # 编译F1AP单元测试
   cd ~/openairinterface5g
   mkdir -p build && cd build
   cmake .. -GNinja -DENABLE_TESTS=ON
   ninja f1ap_lib_test
   
   # 运行测试
   ./f1ap_lib_test
   ```

### 3.5 完整的修改清单

```
修改文件列表：
├─ openair2/COMMON/f1ap_messages_types.h
│  └─ 添加f1ap_plmn_info_t, 修改f1ap_served_cell_info_t
│
├─ openair2/GNB_APP/gnb_config.c
│  └─ read_du_cell_info(): 读取每个PLMN的slice配置
│
├─ openair2/F1AP/lib/f1ap_interface_management.c
│  ├─ encode_served_cell_info(): 编码多个PLMN及其slice
│  └─ decode_served_cell_info(): 解码多个PLMN及其slice
│
├─ openair2/F1AP/lib/f1ap_lib_common.c
│  └─ eq_f1ap_cell_info(): 比较函数，修改为比较plmn_list
│
├─ openair2/F1AP/tests/f1ap_lib_test.c
│  └─ 单元测试：添加多PLMN多slice测试用例
│
├─ openair3/NGAP/ngap_gNB.c
│  └─ ngap_gNB_generate_ng_setup_request(): NGAP编码
│
└─ common/utils/telnetsrv/telnetsrv_o1.c
   └─ set_bwconfig(): telnet命令（已修复）
```

---

## 总结

**回答问题1**: Slice list主要用于：
- F1AP Setup: DU告诉CU支持的slice
- NG Setup: gNB告诉AMF支持的slice  
- PDU Session建立: UE请求特定slice，AMF/SMF验证

**回答问题2**: 按3GPP标准配置：
- UE: ue.conf配置nssai_sst/sd
- gNB: gnb.conf每个PLMN配置snssaiList
- AMF: amfcfg.yaml配置plmnSupportList和snssaiList
- SMF: smfcfg.yaml配置snssaiInfos
- Subscriber: 5GC数据库配置允许的slice

**回答问题3**: 修改要点：
- ✅ 创建f1ap_plmn_info_t结构体
- ✅ 修改f1ap_served_cell_info_t使用plmn_list[]
- ⚠️ 保留第一个plmn字段（用于NR CGI）
- 🔄 更新所有F1AP/NGAP编解码代码
- 🧪 更新单元测试验证正确性
