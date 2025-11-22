# MOCN AMF Selection 详细分析

## `ngap_gNB_nnsf_select_amf_by_plmn_id()` 函数分析

### 1. 函数功能解析

这个函数的作用是：**根据 UE 选择的 PLMN ID 从多个 AMF 中选择最合适的一个**。

#### 核心逻辑：
```c
ngap_gNB_amf_data_t *ngap_gNB_nnsf_select_amf_by_plmn_id(
    ngap_gNB_instance_t *instance_p,                    // gNB 实例（包含所有已连接的 AMF）
    const ngap_rrc_establishment_cause_t cause,         // RRC 建立原因
    const plmn_id_t selected_plmn_identity              // UE 选择的 PLMN
)
```

#### 函数做的事情（逐步分析）：

**Step 1: 遍历所有已连接的 AMF**
```c
RB_FOREACH(amf_data_p, ngap_amf_map, &instance_p->ngap_amf_head) {
```
- `instance_p->ngap_amf_head` 是一个红黑树，存储所有 gNB 已连接的 AMF
- 每个 `amf_data_p` 代表一个 AMF 连接（SCTP 关联）

**Step 2: 检查 AMF 状态**
```c
if (amf_data_p->state != NGAP_GNB_STATE_CONNECTED) {
    // 如果 AMF 未连接或 overload，根据 cause 决定是否跳过
    if (amf_data_p->state == NGAP_GNB_OVERLOAD) {
        // 检查 overload 状态和 RRC cause 的兼容性
        if ((cause == NGAP_RRC_CAUSE_MO_DATA) && 
            (amf_data_p->overload_state == NGAP_OVERLOAD_REJECT_MO_DATA)) {
            continue;  // 跳过这个 AMF
        }
        // 其他 overload 检查...
    } else {
        continue;  // AMF 未连接，跳过
    }
}
```
- 只考虑 **已连接** 或 **overload 但可处理该 cause** 的 AMF

**Step 3: 匹配 UE 选择的 PLMN**
```c
/* Looking for served GUAMI PLMN Identity selected matching the one provided by the UE */
STAILQ_FOREACH(guami_p, &amf_data_p->served_guami, next) {
    STAILQ_FOREACH(served_plmn_p, &guami_p->served_plmns, next) {
        if ((served_plmn_p->mcc == selected_plmn_identity.mcc) && 
            (served_plmn_p->mnc == selected_plmn_identity.mnc)) {
            break;  // 找到匹配的 PLMN
        }
    }
    if (served_plmn_p) break;  // 找到就停止外层循环
}
if (!served_plmn_p) continue;  // 这个 AMF 不支持该 PLMN，跳过
```

关键点：
- 每个 AMF 有 `served_guami` 列表，每个 GUAMI 有 `served_plmns` 列表
- 检查 AMF 是否服务于 UE 选择的 PLMN（MCC + MNC 匹配）
- **这是 MOCN 的核心**：不同的 AMF 可以服务不同的 PLMN

**Step 4: 选择容量最高的 AMF**
```c
if (current_capacity < amf_data_p->relative_amf_capacity) {
    current_capacity = amf_data_p->relative_amf_capacity;
    amf_highest_capacity_p = amf_data_p;  // 记录最高容量的 AMF
}
```
- 在所有支持该 PLMN 的 AMF 中，选择 **relative_amf_capacity** 最高的
- 实现负载均衡

**Step 5: 返回结果**
```c
return amf_highest_capacity_p;  // 可能是 NULL（没有 AMF 支持该 PLMN）
```

---

### 2. 完整调用链分析

```
【用户侧】
UE → 选择 PLMN（通过 SIB1 看到的 PLMN 列表）
   ↓
UE → RRC Setup Complete（携带 selected PLMN ID）
   ↓
═══════════ 空中接口 ═══════════
   ↓
【gNB-DU (PHY/MAC)】
gNB-DU → 接收 RRC Setup Complete
   ↓
gNB-DU → 解码 RRC 消息，提取 selected PLMN
   ↓ F1AP (Initial UL RRC Message Transfer)
【gNB-CU (RRC/NGAP)】
gNB-CU RRC → rrc_gNB_process_RRCSetupComplete()
   ↓
gNB-CU RRC → 提取 NAS PDU 和 selected PLMN
   ↓ ITTI message (NGAP_NAS_FIRST_REQ)
gNB-CU NGAP Task → ngap_gNB_handle_nas_first_req()
   ↓
【AMF 选择逻辑】(ngap_gNB_nas_procedures.c:select_amf())
   ├─ 优先级 1: 如果 UE 提供 GUAMI（已注册的 AMF）
   │    └→ ngap_gNB_nnsf_select_amf_by_guami()
   │
   ├─ 优先级 2: 如果 UE 提供 5G-S-TMSI
   │    └→ ngap_gNB_nnsf_select_amf_by_amf_setid()
   │
   ├─ 优先级 3: 根据 selected PLMN 选择 ← **MOCN 的关键**
   │    └→ ngap_gNB_nnsf_select_amf_by_plmn_id()  ← **你找的这个函数**
   │         ├─ 遍历所有 AMF
   │         ├─ 过滤：只考虑已连接且支持该 PLMN 的 AMF
   │         ├─ 选择：relative_amf_capacity 最高的
   │         └─ 返回：选中的 AMF (或 NULL)
   │
   └─ 优先级 4: 兜底策略
        └→ ngap_gNB_nnsf_select_amf()  (选择 capacity 最高的 AMF)
   ↓
【选中 AMF 后】
ngap_gNB_handle_nas_first_req() → 构造 NGAP Initial UE Message
   ↓ SCTP (N2 interface)
【目标 AMF】
AMF → 接收 Initial UE Message
AMF → 提取 NAS PDU 和 selected PLMN
AMF → 验证 UE subscription（是否属于该 PLMN）
AMF → 开始认证和注册流程
```

---

### 3. 函数在哪里？

**位置：gNB-CU 的 NGAP 层**

```
5G gNB 架构（MOCN 场景）：

┌─────────────────────────────────────────────────────────────┐
│                       gNB-CU                                 │
│  ┌──────────────┐  ┌──────────────────────────────────────┐│
│  │     RRC      │  │         NGAP Task                     ││
│  │              │  │  ┌──────────────────────────────────┐ ││
│  │ - 解码 RRC   │  │  │ ngap_gNB_nnsf_select_amf_by_*() │ ││ ← 函数在这里
│  │ - 提取 PLMN  │→→│  │ - 遍历所有 AMF 连接              │ ││
│  └──────────────┘  │  │ - 匹配 PLMN                      │ ││
│                     │  │ - 选择最优 AMF                   │ ││
│                     │  └──────────────────────────────────┘ ││
│                     │                                        ││
│                     │  ngap_amf_head (红黑树):              ││
│                     │  ├─ AMF_A (208-93, capacity=100)      ││
│                     │  ├─ AMF_B (208-94, capacity=80)       ││
│                     │  └─ AMF_C (208-95, capacity=120)      ││
│                     └────────────────────────────────────────┘│
└────────────────────────────┬────────────────────────────────┘
                             │ N2 (NGAP over SCTP)
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ↓                    ↓                    ↓
   ┌─────────┐         ┌─────────┐         ┌─────────┐
   │ AMF_A   │         │ AMF_B   │         │ AMF_C   │
   │ PLMN:   │         │ PLMN:   │         │ PLMN:   │
   │ 208-93  │         │ 208-94  │         │ 208-95  │
   │ (运营商A)│         │ (运营商B)│         │ (运营商C)│
   └─────────┘         └─────────┘         └─────────┘
       │                    │                    │
       └────────────────────┴────────────────────┘
                    5G Core Networks
```

**特点：**
- 函数在 **gNB-CU 的 NGAP 任务**中执行
- 不在 DU（DU 只负责转发 RRC 消息）
- 不在 RU（RU 只负责射频）

---

### 4. OAI 的人说得对吗？

**✅ 完全正确！这个函数确实是 MOCN 的关键。**

#### 他说对的地方：

1. **"这个函数可以连接多个 AMF"** ✅
   - 函数通过 `RB_FOREACH` 遍历 `instance_p->ngap_amf_head`
   - 这个红黑树可以存储**多个 AMF 连接**
   - 每个 AMF 通过独立的 SCTP 关联连接

2. **"根据 PLMN 选择 AMF"** ✅
   - 代码 137-145 行明确检查 `served_plmn_p->mcc` 和 `mnc`
   - 只选择支持 UE 所选 PLMN 的 AMF

3. **"MOCN 需要这个功能"** ✅
   - MOCN 的定义就是：多个运营商共享 RAN，但有独立的 Core Network
   - 这个函数实现了：**不同 PLMN 路由到不同 AMF（不同 Core Network）**

---

### 5. 需要补充的关键点

#### ✅ **当前代码已经支持的（不需要改）：**

1. **多 AMF 连接能力** ✅
   ```c
   // ngap_gNB_defs.h:222
   typedef struct ngap_gNB_instance_s {
     uint32_t ngap_amf_nb;                  // AMF 数量
     uint32_t ngap_amf_associated_nb;       // 已连接的 AMF 数量
     RB_HEAD(ngap_amf_map, ...) ngap_amf_head;  // AMF 树（可存多个）
   }
   ```

2. **PLMN 匹配逻辑** ✅
   ```c
   // 每个 AMF 的 served_guami 包含 served_plmns 列表
   struct served_guami_s {
     STAILQ_HEAD(served_plmns_s, plmn_identity_s) served_plmns;
   }
   ```

3. **负载均衡** ✅
   ```c
   // 选择 relative_amf_capacity 最高的 AMF
   if (current_capacity < amf_data_p->relative_amf_capacity) {
     amf_highest_capacity_p = amf_data_p;
   }
   ```

#### ❌ **需要验证和补充的（MOCN 部署）：**

##### 1. **配置文件需要支持多个 AMF**

**当前配置格式（单 AMF）：**
```conf
# ci-scripts/conf_files/gnb.sa.band78.106prb.rfsim.conf:162
amf_ip_address = ({ ipv4 = "192.168.71.132"; });
```

**MOCN 需要的格式（多 AMF）：**
```conf
# 需要确认 OAI 是否已支持以下格式
amf_ip_address = (
    { ipv4 = "192.168.1.100"; },  # AMF for PLMN 208-93
    { ipv4 = "192.168.1.101"; },  # AMF for PLMN 208-94
    { ipv4 = "192.168.1.102"; }   # AMF for PLMN 208-95
);
```

**需要验证：**
- 配置文件解析是否支持数组格式？
- 代码位置：`openair2/GNB_APP/gnb_config.c` 或 `gnb_paramdef.h`

##### 2. **NG Setup 过程需要与多个 AMF 建立连接**

```c
// 启动时，gNB 需要：
for (int i = 0; i < config->num_amf; i++) {
    // 对每个 AMF 建立 SCTP 关联
    sctp_connect_to_amf(config->amf_list[i].ip_address, config->amf_list[i].port);
    // 发送 NG Setup Request
    send_ng_setup_request(amf_connection[i]);
}
```

**需要检查：**
- `openair3/NGAP/ngap_gNB_management_procedures.c`
- 是否已支持向多个 AMF 发送 NG Setup？

##### 3. **AMF 配置需要声明支持的 PLMN**

每个 AMF 在 NG Setup Response 中需要告知 gNB 自己支持的 PLMN：

```asn1
NG Setup Response ::= SEQUENCE {
    ServedGUAMIList ::= SEQUENCE (SIZE(1..256)) OF SEQUENCE {
        guami GUAMI,
        backupAMFName AMFName OPTIONAL,
        ...
    }
}

GUAMI ::= SEQUENCE {
    pLMNIdentity PLMNIdentity,  ← 这里声明 AMF 支持的 PLMN
    aMFRegionID AMFRegionID,
    aMFSetID AMFSetID,
    aMFPointer AMFPointer,
    ...
}
```

**gNB 端处理：**
```c
// openair3/NGAP/ngap_gNB_management_procedures.c
// 在 ngap_gNB_handle_ng_setup_response() 中
for (int i = 0; i < served_guami_list->count; i++) {
    // 解析每个 GUAMI 的 PLMN
    plmn_identity_s *plmn = parse_plmn(...);
    // 存储到 amf_data_p->served_guami
    add_served_plmn(amf_data_p, plmn);
}
```

##### 4. **日志和调试**

**需要添加详细日志：**
```c
// 在 ngap_gNB_nnsf_select_amf_by_plmn_id() 中添加
LOG_I(NGAP, "AMF Selection for UE with PLMN MCC=%03d MNC=%0*d:\n",
      selected_plmn_identity.mcc,
      selected_plmn_identity.mnc_digit_length,
      selected_plmn_identity.mnc);

RB_FOREACH(amf_data_p, ngap_amf_map, &instance_p->ngap_amf_head) {
    LOG_D(NGAP, "  Checking AMF '%s' (assoc_id=%d, state=%d, capacity=%d)\n",
          amf_data_p->amf_name,
          amf_data_p->assoc_id,
          amf_data_p->state,
          amf_data_p->relative_amf_capacity);
    
    STAILQ_FOREACH(guami_p, &amf_data_p->served_guami, next) {
        STAILQ_FOREACH(served_plmn_p, &guami_p->served_plmns, next) {
            LOG_D(NGAP, "    Served PLMN: MCC=%03d MNC=%0*d\n",
                  served_plmn_p->mcc,
                  served_plmn_p->mnc_digit_length,
                  served_plmn_p->mnc);
            
            if ((served_plmn_p->mcc == selected_plmn_identity.mcc) && 
                (served_plmn_p->mnc == selected_plmn_identity.mnc)) {
                LOG_I(NGAP, "  ✓ PLMN MATCH! AMF '%s' supports this PLMN\n", 
                      amf_data_p->amf_name);
            }
        }
    }
}

if (amf_highest_capacity_p) {
    LOG_I(NGAP, "✓ Selected AMF '%s' with capacity=%d for PLMN MCC=%03d MNC=%0*d\n",
          amf_highest_capacity_p->amf_name,
          amf_highest_capacity_p->relative_amf_capacity,
          selected_plmn_identity.mcc,
          selected_plmn_identity.mnc_digit_length,
          selected_plmn_identity.mnc);
} else {
    LOG_E(NGAP, "✗ No AMF found for PLMN MCC=%03d MNC=%0*d\n",
          selected_plmn_identity.mcc,
          selected_plmn_identity.mnc_digit_length,
          selected_plmn_identity.mnc);
}
```

##### 5. **错误处理**

**当前代码（第 154 行）：**
```c
return amf_highest_capacity_p;  // 可能返回 NULL
```

**调用方需要检查：**
```c
// ngap_gNB_nas_procedures.c:123
amf = ngap_gNB_nnsf_select_amf_by_plmn_id(instance_p, msg->establishment_cause, msg->plmn);
if (amf) {
    LOG_I(NGAP, "UE %d: Selected AMF...\n", ...);
    return amf;
} else {
    // 当前会 fallback 到 highest capacity AMF
    amf = ngap_gNB_nnsf_select_amf(instance_p, msg->establishment_cause);
}
```

**问题：**
- 如果 UE 选择的 PLMN 没有对应 AMF，是否应该直接拒绝？
- 还是 fallback 到其他 AMF（这样会导致注册失败）？

**建议：**
```c
if (!amf) {
    LOG_E(NGAP, "UE %d: No AMF found for selected PLMN MCC=%03d MNC=%0*d, rejecting UE\n",
          msg->gNB_ue_ngap_id,
          msg->plmn.mcc,
          msg->plmn.mnc_digit_length,
          msg->plmn.mnc);
    // 发送 RRC Reject 给 UE
    send_rrc_reject(msg->gNB_ue_ngap_id, RRC_REJECT_CAUSE_NO_SUITABLE_CELL);
    return NULL;
}
```

---

### 6. MOCN 完整验证清单

#### Phase 1: 配置验证
- [ ] 检查 gNB 配置文件是否支持多 AMF（数组格式）
- [ ] 检查配置解析代码（`openair2/GNB_APP/gnb_config.c`）
- [ ] 测试配置多个 AMF IP 地址

#### Phase 2: 连接验证
- [ ] 启动 gNB，观察日志确认与多个 AMF 建立 SCTP 连接
- [ ] 检查 NG Setup Request/Response 消息
- [ ] 确认每个 AMF 的 `served_guami` 正确解析

#### Phase 3: PLMN 广播验证（前面已分析）
- [ ] 修改 `get_SIB1_NR()` 支持多 PLMN
- [ ] UE 能看到多个 PLMN

#### Phase 4: AMF 选择验证
- [ ] UE 选择 PLMN A，打印日志确认选择了 AMF_A
- [ ] UE 选择 PLMN B，打印日志确认选择了 AMF_B
- [ ] 添加日志到 `ngap_gNB_nnsf_select_amf_by_plmn_id()`

#### Phase 5: 端到端验证
- [ ] UE 选择 PLMN A → 连接到 Core Network A → 注册成功
- [ ] UE 选择 PLMN B → 连接到 Core Network B → 注册成功
- [ ] 抓包验证 Initial UE Message 发送到正确的 AMF

---

### 7. 代码修改建议（增强日志）

**修改 `ngap_gNB_nnsf_select_amf_by_plmn_id()` 添加详细日志：**

```c
ngap_gNB_amf_data_t *ngap_gNB_nnsf_select_amf_by_plmn_id(ngap_gNB_instance_t *instance_p,
                                                         const ngap_rrc_establishment_cause_t cause,
                                                         const plmn_id_t selected_plmn_identity)
{
  struct ngap_gNB_amf_data_s *amf_data_p = NULL;
  struct ngap_gNB_amf_data_s *amf_highest_capacity_p = NULL;
  uint8_t current_capacity = 0;
  int num_amf_checked = 0;
  int num_amf_plmn_matched = 0;

  LOG_I(NGAP, "[MOCN] AMF Selection: UE selected PLMN MCC=%03d MNC=%0*d\n",
        selected_plmn_identity.mcc,
        selected_plmn_identity.mnc_digit_length,
        selected_plmn_identity.mnc);

  RB_FOREACH(amf_data_p, ngap_amf_map, &instance_p->ngap_amf_head) {
    num_amf_checked++;
    
    LOG_D(NGAP, "[MOCN]   Checking AMF '%s' (assoc_id=%d, state=%s, capacity=%d)\n",
          amf_data_p->amf_name ? amf_data_p->amf_name : "UNNAMED",
          amf_data_p->assoc_id,
          amf_data_p->state == NGAP_GNB_STATE_CONNECTED ? "CONNECTED" :
          amf_data_p->state == NGAP_GNB_OVERLOAD ? "OVERLOAD" : "DISCONNECTED",
          amf_data_p->relative_amf_capacity);

    struct served_guami_s *guami_p = NULL;
    struct plmn_identity_s *served_plmn_p = NULL;
    
    if (amf_data_p->state != NGAP_GNB_STATE_CONNECTED) {
      // ... (原有的 overload 检查代码)
      LOG_D(NGAP, "[MOCN]     AMF not available (state=%d), skipping\n", amf_data_p->state);
      continue;
    }

    /* Looking for served GUAMI PLMN Identity selected matching the one provided by the UE */
    STAILQ_FOREACH(guami_p, &amf_data_p->served_guami, next) {
      STAILQ_FOREACH(served_plmn_p, &guami_p->served_plmns, next) {
        LOG_D(NGAP, "[MOCN]     Served PLMN: MCC=%03d MNC=%0*d\n",
              served_plmn_p->mcc,
              served_plmn_p->mnc_digit_length,
              served_plmn_p->mnc);
              
        if ((served_plmn_p->mcc == selected_plmn_identity.mcc) && 
            (served_plmn_p->mnc == selected_plmn_identity.mnc)) {
          LOG_I(NGAP, "[MOCN]     ✓ PLMN MATCH! AMF '%s' supports PLMN %03d-%0*d\n",
                amf_data_p->amf_name ? amf_data_p->amf_name : "UNNAMED",
                served_plmn_p->mcc,
                served_plmn_p->mnc_digit_length,
                served_plmn_p->mnc);
          break;
        }
      }
      if (served_plmn_p) break;
    }
    
    if (!served_plmn_p) {
      LOG_D(NGAP, "[MOCN]     No PLMN match, skipping AMF '%s'\n",
            amf_data_p->amf_name ? amf_data_p->amf_name : "UNNAMED");
      continue;
    }

    num_amf_plmn_matched++;

    if (current_capacity < amf_data_p->relative_amf_capacity) {
      current_capacity = amf_data_p->relative_amf_capacity;
      amf_highest_capacity_p = amf_data_p;
      LOG_D(NGAP, "[MOCN]     New best candidate: AMF '%s' with capacity=%d\n",
            amf_data_p->amf_name ? amf_data_p->amf_name : "UNNAMED",
            current_capacity);
    }
  }

  if (amf_highest_capacity_p) {
    LOG_I(NGAP, "[MOCN] ✓ Selected AMF '%s' (assoc_id=%d, capacity=%d) for PLMN %03d-%0*d "
          "(checked %d AMFs, %d matched PLMN)\n",
          amf_highest_capacity_p->amf_name ? amf_highest_capacity_p->amf_name : "UNNAMED",
          amf_highest_capacity_p->assoc_id,
          amf_highest_capacity_p->relative_amf_capacity,
          selected_plmn_identity.mcc,
          selected_plmn_identity.mnc_digit_length,
          selected_plmn_identity.mnc,
          num_amf_checked,
          num_amf_plmn_matched);
  } else {
    LOG_E(NGAP, "[MOCN] ✗ No AMF found for PLMN %03d-%0*d "
          "(checked %d AMFs, %d matched PLMN)\n",
          selected_plmn_identity.mcc,
          selected_plmn_identity.mnc_digit_length,
          selected_plmn_identity.mnc,
          num_amf_checked,
          num_amf_plmn_matched);
  }

  return amf_highest_capacity_p;
}
```

---

## 总结

### ✅ OAI 的人说的完全正确

这个函数 **`ngap_gNB_nnsf_select_amf_by_plmn_id()`** 确实是 MOCN 的关键组件之一：

1. **已支持多 AMF 连接架构** ✅（数据结构已就位）
2. **已支持基于 PLMN 的 AMF 选择** ✅（逻辑已完整）
3. **已支持负载均衡** ✅（选择最高 capacity）

### ❓ 需要验证的部分

1. **配置文件** - 是否支持配置多个 AMF？
2. **NG Setup** - 启动时是否向所有 AMF 建立连接？
3. **AMF 配置** - 每个 AMF 是否正确声明自己支持的 PLMN？
4. **日志** - 是否有足够的调试日志来验证选择过程？

### 🎯 下一步行动

1. **查找配置解析代码**，确认如何配置多个 AMF
2. **添加详细日志**（如上所示）
3. **端到端测试**：
   - 部署 2-3 个 AMF（不同 PLMN）
   - 在 SIB1 广播多个 PLMN
   - UE 选择不同 PLMN，观察日志确认路由正确

---

## 参考资料

- 3GPP TS 23.501: System architecture (AMF Discovery and Selection, clause 6.3.5)
- 3GPP TS 38.413: NG Application Protocol (NG Setup, clause 8.7.1)
- 3GPP TS 23.251: Network sharing
- OAI Source: `openair3/NGAP/ngap_gNB_nnsf.c`
