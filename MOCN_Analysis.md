# MOCN (Multi-Operator Core Network) 实现分析

## 关于 `get_SIB1_NR()` 函数的详细分析

### 1. 它在做什么？

`get_SIB1_NR()` 函数的作用是**生成并编码 5G NR 的 SIB1 (System Information Block Type 1)** 消息。

**SIB1 的核心功能：**
- **系统信息广播**：gNB 通过 SIB1 向所有 UE 广播小区的基本配置信息
- **PLMN 信息**：告诉 UE 这个小区属于哪个运营商（PLMN ID = MCC + MNC）
- **小区选择信息**：包含 cell ID、TAC (Tracking Area Code)、接入限制等
- **无线资源配置**：初始 BWP、RACH、PDCCH/PDSCH/PUSCH/PUCCH 配置等

**函数具体做的事情：**
1. 创建 `NR_BCCH_DL_SCH_Message_t` 结构（SIB1 消息）
2. 填充 **cellSelectionInfo**（小区选择信息，如最小接收功率 q_RxLevMin）
3. **❗关键部分❗** 填充 **cellAccessRelatedInfo**：
   - **PLMN Identity List**（当前只支持 1 个 PLMN）— **这就是需要修改的地方**
   - Cell Identity（36-bit cell ID）
   - Tracking Area Code
4. 填充 **servingCellConfigCommon**（服务小区公共配置）：
   - 下行配置（频率、BWP、PDCCH/PDSCH）
   - 上行配置（频率、BWP、RACH/PUSCH/PUCCH）
   - SSB 配置（同步信号块位置、功率）
   - TDD 配置（如果是 TDD 模式）
5. 填充 **ue_TimersAndConstants**（UE 定时器和常数）

### 2. 它到底在哪里：CU/DU/RU？

**答案：它在 DU (Distributed Unit) 的 MAC 层。**

**架构分析：**

根据 OAI 的 F1 设计文档（`doc/F1AP/F1-design.md`）：

```
5G gNB 架构（CU-DU 分离）：
┌─────────────────────────────────────────┐
│           gNB-CU (Centralized Unit)     │
│  ┌──────────┐  ┌──────────┐  ┌────────┐│
│  │   RRC    │  │   PDCP   │  │  SDAP  ││
│  └──────────┘  └──────────┘  └────────┘│
└────────────────────┬────────────────────┘
                     │ F1 interface (F1-C + F1-U)
                     │ F1AP messages over SCTP
                     │
┌────────────────────┴────────────────────┐
│        gNB-DU (Distributed Unit)        │
│  ┌──────────┐  ┌──────────┐  ┌────────┐│
│  │   RLC    │  │   MAC    │  │  PHY   ││ <--- get_SIB1_NR() 在这里
│  └──────────┘  └──────────┘  └────────┘│
└────────────────────┬────────────────────┘
                     │
                     │ Fronthaul interface
                     │
┌────────────────────┴────────────────────┐
│                RU (Radio Unit)          │
│              RF Hardware                │
└─────────────────────────────────────────┘
```

**函数位置证据：**
1. **文件路径**：`openair2/LAYER2/NR_MAC_gNB/nr_radio_config.c`
   - `LAYER2` → 第二层协议栈
   - `NR_MAC_gNB` → 5G NR gNB 的 MAC 层
   
2. **调用链**：
   ```c
   // 在 openair2/LAYER2/NR_MAC_gNB/config.c:1096
   void nr_mac_configure_sib1(gNB_MAC_INST *nrmac, ...)
   {
     NR_BCCH_DL_SCH_Message_t *sib1 = get_SIB1_NR(scc, plmn, cellID, tac, ...);
     cc->sib1 = sib1;
     cc->sib1_bcch_length = encode_SIB_NR(sib1, ...); // 编码后发给 PHY
   }
   ```

3. **在 F1 架构中**：
   - **Monolithic SA 模式**：DU 和 CU 在同一个进程，RRC 直接调用 MAC
   - **CU-DU Split 模式**：
     - RRC (在 CU) 通过 F1AP 消息告诉 MAC (在 DU) 配置信息
     - `get_SIB1_NR()` 在 **DU 侧**根据配置生成 SIB1
     - DU 通过 PHY 层广播 SIB1

**为什么 SIB1 在 DU 而不是 CU？**
- SIB1 需要周期性广播（通过 BCCH-DL-SCH 信道）
- 广播调度由 **MAC 层**负责（MAC 在 DU）
- RRC（在 CU）只负责配置内容，实际编码和调度在 DU

### 3. OAI 的人说的话对吗？

**结论：基本正确，但有些细节需要澄清。**

#### ✅ **正确的部分：**

1. **"2714 行只读了 1 个 PLMN"** ✅
   ```c
   // 第 2714 行
   int num_plmn = 1; // int num_plmn = configuration->num_plmn;
   ```
   确实硬编码为 1，注释里也说明了原本应该从配置读取。

2. **"要改成多个"** ✅
   需要修改循环逻辑，支持配置并广播多个 PLMN。

3. **"need to extend from outside, from the configuration"** ✅
   需要从外部配置文件（gNB 配置）读取多个 PLMN 信息。

4. **"UE 可以做 operator search 看到 different PLMNs"** ✅
   UE 解码 SIB1 后，能看到 `plmn_IdentityInfoList` 中的所有 PLMN。

5. **"UE select one, transported to CU, picked up in AMF function"** ✅
   UE 选择的 PLMN 会在后续流程中使用（见下文流程 4）。

#### ❌ **需要澄清的部分：**

1. **"RRC part is missing"**
   - **不准确**：RRC 部分已经有 SIB1 编码能力
   - **准确的说法**：需要扩展 RRC 配置解析，支持多 PLMN 输入

2. **"broadcast multiple PLMNs"**
   - **技术上**：是在 **一个 SIB1 消息中广播多个 PLMN**，而不是多个 SIB1
   - SIB1 中有 `plmn_IdentityInfoList`（PLMN 列表），本来就支持多个

### 4. 广播多个 PLMN 的完整流程

```
┌──────────────────────────────────────────────────────────────┐
│ Phase 1: 配置阶段（静态配置或动态从 Core 获取）               │
└──────────────────────────────────────────────────────────────┘

[OAI gNB 配置文件] → 包含多个 PLMN 信息
    ├─ PLMN 1: MCC=208, MNC=93 (运营商 A)
    ├─ PLMN 2: MCC=208, MNC=94 (运营商 B)
    └─ PLMN 3: MCC=208, MNC=95 (运营商 C)

           ↓ (启动时读取配置)

[gNB-CU (RRC)]
    ├─ 解析配置文件
    ├─ 通过 F1AP: F1 Setup Request 发送配置给 DU
    └─ (如果是 Monolithic，直接调用 nr_mac_configure_sib1)

           ↓ F1AP (F1 Setup Response)

[gNB-DU (MAC)]
    ├─ 接收多 PLMN 配置
    ├─ 调用 get_SIB1_NR(..., plmn_list, ...)  ← **需要修改这里**
    └─ 生成 SIB1，包含所有 PLMN

┌──────────────────────────────────────────────────────────────┐
│ Phase 2: 广播阶段（周期性广播 SIB1）                          │
└──────────────────────────────────────────────────────────────┘

[gNB-DU (MAC scheduler)]
    └─→ 每 160ms 周期调度 SIB1 传输（BCCH-DL-SCH）
           ↓
    [PHY 层] → 通过 SSB 和 PDSCH 广播
           ↓
    ═══════════ 空中接口 ═══════════
           ↓
    [UE] 接收并解码 SIB1
        └─→ SIB1.cellAccessRelatedInfo.plmn_IdentityInfoList
             ├─ PLMN 1: MCC=208, MNC=93
             ├─ PLMN 2: MCC=208, MNC=94
             └─ PLMN 3: MCC=208, MNC=95

┌──────────────────────────────────────────────────────────────┐
│ Phase 3: UE 选择 PLMN                                         │
└──────────────────────────────────────────────────────────────┘

[UE NAS 层]
    ├─ Manual PLMN selection：用户手动选择运营商
    │  或
    └─ Automatic PLMN selection：根据优先级（USIM 中的 PLMN selector）

           ↓ (选择 PLMN 2: MCC=208, MNC=94)

[UE] 发起 RRC Connection Setup Request
    └─→ 携带 selected PLMN ID (208-94)

┌──────────────────────────────────────────────────────────────┐
│ Phase 4: 连接建立与 PLMN 路由                                 │
└──────────────────────────────────────────────────────────────┘

    [UE] → RRC Setup Request (with PLMN ID)
           ↓
    [gNB-DU] → F1AP: Initial UL RRC Message Transfer
           ↓
    [gNB-CU] → 解码 RRC message，提取 selected PLMN
           ↓
    [gNB-CU] → NGAP: Initial UE Message → **选择对应的 AMF**
           |
           ├─ 如果选 PLMN 1 → 路由到 AMF_A (Core Network A)
           ├─ 如果选 PLMN 2 → 路由到 AMF_B (Core Network B)
           └─ 如果选 PLMN 3 → 路由到 AMF_C (Core Network C)
           ↓
    [AMF] 接收 Initial UE Message
        ├─ 提取 selected PLMN (MCC=208, MNC=94)
        ├─ 验证 UE 的 USIM 是否属于该 PLMN
        └─ 继续认证和注册流程

┌──────────────────────────────────────────────────────────────┐
│ Phase 5: 验证与 Debug                                          │
└──────────────────────────────────────────────────────────────┘

[gNB-CU] 打印日志：
    LOG_I(RRC, "UE selected PLMN: MCC=%d, MNC=%d", mcc, mnc);

[AMF] 打印日志：
    LOG_I(NAS, "Received Initial UE with PLMN: MCC=%d, MNC=%d", mcc, mnc);

→ 验证：UE 选择的 PLMN 是否正确传递到 AMF
```

### 关键技术点

#### **MOCN 与 PLMN sharing 的区别：**

| 特性 | MOCN | Network Sharing (GWCN) |
|------|------|------------------------|
| Core Network | **多个独立的 5GC** | 共享同一个 5GC |
| PLMN ID | 每个运营商有自己的 PLMN | 多个 PLMN 映射到同一个 5GC |
| SIB1 | 广播多个 PLMN | 广播多个 PLMN |
| AMF 选择 | **根据 UE 选择的 PLMN 路由到不同 AMF** | 所有 PLMN 都路由到同一个 AMF |
| 计费 | 各运营商独立计费 | 共享运营商统一计费 |

#### **为什么需要"从配置扩展"：**

当前代码：
```c
// 硬编码，只能有 1 个 PLMN
int num_plmn = 1;
const plmn_id_t *plmn = ...;  // 单个 PLMN
```

需要改成：
```c
// 从配置读取多个 PLMN
int num_plmn = config->num_plmn;  // 可能是 2, 3, ...
const plmn_id_t *plmn_list = config->plmn_list;  // PLMN 数组
```

### 下一步修改建议

#### 1. **修改配置文件结构**（`gnb.conf`）
```conf
gNBs = (
{
  // ...
  plmn_list = (
    { mcc = 208; mnc = 93; mnc_length = 2; },  // 运营商 A
    { mcc = 208; mnc = 94; mnc_length = 2; },  // 运营商 B
    { mcc = 208; mnc = 95; mnc_length = 2; }   // 运营商 C
  );
  // ...
}
);
```

#### 2. **修改 `get_SIB1_NR()` 函数签名**
```c
// 旧版本
NR_BCCH_DL_SCH_Message_t *get_SIB1_NR(
    const NR_ServingCellConfigCommon_t *scc,
    const plmn_id_t *plmn,  // ← 单个 PLMN
    uint64_t cellID,
    int tac,
    const nr_mac_config_t *mac_config
);

// 新版本（MOCN）
NR_BCCH_DL_SCH_Message_t *get_SIB1_NR(
    const NR_ServingCellConfigCommon_t *scc,
    const plmn_id_t *plmn_list,  // ← PLMN 列表
    int num_plmn,                // ← PLMN 数量
    uint64_t cellID,
    int tac,
    const nr_mac_config_t *mac_config
);
```

#### 3. **修改第 2714-2738 行的循环逻辑**
```c
// 当前代码（只支持 1 个 PLMN）
int num_plmn = 1;
asn1cSequenceAdd(..., nr_plmn_info);
for (int i = 0; i < num_plmn; ++i) {
    asn1cSequenceAdd(nr_plmn_info->plmn_IdentityList.list, ..., nr_plmn);
    // 填充 plmn[0] 的 MCC/MNC
}

// 修改后（支持多个 PLMN）
for (int i = 0; i < num_plmn; ++i) {
    asn1cSequenceAdd(..., nr_plmn_info);
    asn1cSequenceAdd(nr_plmn_info->plmn_IdentityList.list, ..., nr_plmn);
    
    // 填充 plmn_list[i] 的 MCC/MNC
    asn1cCalloc(nr_plmn->mcc, mcc);
    int confMcc = plmn_list[i].mcc;  // ← 使用 plmn_list[i]
    // ... (MCC 的 3 位数字)
    
    int mnc = plmn_list[i].mnc;      // ← 使用 plmn_list[i]
    if (plmn_list[i].mnc_digit_length == 3) {
        // ... (MNC 的 2 或 3 位数字)
    }
    
    // Cell ID 和 TAC（所有 PLMN 共享同一个 cell）
    NR_CELL_ID_TO_BIT_STRING(cellID, &nr_plmn_info->cellIdentity);
    nr_plmn_info->trackingAreaCode = ... // 填充 TAC
}
```

#### 4. **在 gNB-CU (RRC) 侧添加 PLMN 路由逻辑**
```c
// 在 RRC 接收到 UE 的 RRC Setup Request 时
void rrc_gNB_process_RRCSetupRequest(...) {
    // 提取 UE 选择的 PLMN
    plmn_id_t selected_plmn = extract_plmn_from_rrc_message(...);
    
    // 根据 PLMN 选择对应的 AMF
    ngap_amf_t *target_amf = select_amf_by_plmn(selected_plmn);
    
    // 发送 NGAP Initial UE Message 到对应的 AMF
    ngap_send_initial_ue_message(target_amf, ue_context, nas_pdu);
    
    // Debug 打印
    LOG_I(NR_RRC, "UE %x selected PLMN MCC=%d MNC=%d, routing to AMF %s\n",
          ue_id, selected_plmn.mcc, selected_plmn.mnc, target_amf->name);
}
```

#### 5. **配置多个 AMF 连接**
```conf
# gNB 配置文件
AMF_list = (
  {
    AMF_IPAddress   = "192.168.1.100";  # AMF for PLMN 208-93
    AMF_Port        = 38412;
    PLMN            = { mcc = 208; mnc = 93; };
  },
  {
    AMF_IPAddress   = "192.168.1.101";  # AMF for PLMN 208-94
    AMF_Port        = 38412;
    PLMN            = { mcc = 208; mnc = 94; };
  }
);
```

## 总结

1. **`get_SIB1_NR()` 的作用**：在 DU 的 MAC 层生成并编码 SIB1 系统信息，用于向 UE 广播小区配置。

2. **所在位置**：gNB-DU 的 MAC 层（`openair2/LAYER2/NR_MAC_gNB/`），不在 CU 也不在 RU。

3. **OAI 的建议基本正确**：确实需要扩展第 2714 行的代码，支持从配置读取并广播多个 PLMN。

4. **完整流程**：
   - 配置阶段：gNB 从配置文件读取多个 PLMN
   - 广播阶段：DU 在 SIB1 中广播所有 PLMN
   - 选择阶段：UE 选择一个 PLMN
   - 路由阶段：gNB-CU 根据 UE 选择的 PLMN 路由到对应的 AMF
   - 验证阶段：在各层打印日志，确认 PLMN 选择和路由正确

## 参考资料

- 3GPP TS 38.331: NR RRC Protocol specification
- 3GPP TS 38.473: F1 Application Protocol (F1AP)
- 3GPP TS 38.413: NG Application Protocol (NGAP)
- 3GPP TS 23.501: System architecture for the 5G System
- OAI F1 Design: `doc/F1AP/F1-design.md`
