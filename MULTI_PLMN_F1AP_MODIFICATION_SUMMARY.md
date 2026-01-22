# Multi-PLMN F1AP 修改总结

## 📋 修改概览

本次修改实现了 **Multi-PLMN (Multiple Operator Core Network)** 支持，允许 gNB 同时服务多个 PLMN，每个 PLMN 有独立的网络切片配置。

---

## ✅ 已完成的 4 处代码修改

### **1️⃣ `encode_served_cell_info()` - F1AP 编码函数**

**文件**: `openair2/F1AP/lib/f1ap_interface_management.c`

**修改内容**:
- ✅ 将单 PLMN 编码改为循环编码所有 PLMN
- ✅ 每个 PLMN 使用 `served_plmn_list[i]` 中的独立切片列表
- ✅ 添加 LOG 输出，显示编码的 PLMN 数量和每个切片信息
- ✅ 旧代码保留为注释，方便回滚

**关键代码**:
```c
// [MOCN] NEW CODE: Encode all PLMNs with their independent slice lists
LOG_I(F1AP, "[MOCN] encode_served_cell_info: Encoding %d PLMNs\n", c->num_plmn);
for (int i = 0; i < c->num_plmn; i++) {
  const f1ap_served_plmn_info_t *plmn_info = &c->served_plmn_list[i];
  // 编码 PLMN Identity 和切片列表
  servedPLMN_item->iE_Extensions = write_slice_info(plmn_info->num_nssai, plmn_info->nssai);
}
```

---

### **2️⃣ `decode_served_cell_info()` - F1AP 解码函数**

**文件**: `openair2/F1AP/lib/f1ap_interface_management.c`

**修改内容**:
- ✅ 将单 PLMN 解码改为循环解码所有 PLMN
- ✅ 填充 `served_plmn_list[]` 数组
- ✅ 添加 LOG 输出，显示解码的 PLMN 数量和切片信息
- ✅ 旧代码保留为注释

**关键代码**:
```c
// [MOCN] NEW CODE: Decode all PLMNs with their independent slice lists
LOG_I(F1AP, "[MOCN] decode_served_cell_info: Decoding %d PLMNs\n", in->servedPLMNs.list.count);
info->num_plmn = in->servedPLMNs.list.count;
for (int i = 0; i < in->servedPLMNs.list.count; i++) {
  // 解码 PLMN 和切片到 served_plmn_list[i]
  info->served_plmn_list[i].num_nssai = read_slice_info(plmn_item, ...);
}
```

---

### **3️⃣ `eq_f1ap_cell_info()` - 比较函数**

**文件**: `openair2/F1AP/lib/f1ap_lib_common.c`

**修改内容**:
- ✅ 比较 `served_plmn_list[]` 而不是全局 `nssai[]`
- ✅ 逐个比较每个 PLMN 及其切片列表
- ✅ 旧代码保留为注释

**关键代码**:
```c
// [MOCN] NEW CODE: Compare served_plmn_list (per-PLMN slices)
_F1_EQ_CHECK_INT(a->num_plmn, b->num_plmn);
for (int i = 0; i < a->num_plmn; ++i) {
  // 比较 PLMN 和切片
  _F1_EQ_CHECK_INT(a->served_plmn_list[i].num_nssai, b->served_plmn_list[i].num_nssai);
  for (int s = 0; s < a->served_plmn_list[i].num_nssai; ++s) {
    // 比较每个切片的 SST 和 SD
  }
}
```

---

### **4️⃣ `copy_f1ap_served_cell_info()` - 复制函数**

**文件**: `openair2/F1AP/lib/f1ap_interface_management.c`

**修改内容**:
- ✅ 复制 `served_plmn_list[]` 数组
- ✅ 保留 `num_ssi` 用于兼容性
- ✅ 旧代码保留为注释

**关键代码**:
```c
// [MOCN] NEW CODE: Copy served_plmn_list (per-PLMN slices)
for (int i = 0; i < src->num_plmn; ++i) {
  dst.served_plmn_list[i] = src->served_plmn_list[i];
}
```

---

## 📊 问题回答

### **Q1: DU encode，CU decode 的流程？**

✅ **是的！** 完整流程：

```
gNB-DU:
  f1ap_served_cell_info_t (内部结构)
    ↓ encode_served_cell_info()
  F1AP_Served_Cell_Information_t (ASN.1)
    ↓ F1AP Setup Request (DU → CU)
gNB-CU:
  F1AP_Served_Cell_Information_t (ASN.1)
    ↓ decode_served_cell_info()
  f1ap_served_cell_info_t (内部结构)
```

---

### **Q2: 旧的 nssai[] 会不会还有地方用到？**

✅ **保留策略**:
- `nssai[]` 和 `num_ssi` **保留在结构体中**，不删除
- 所有使用旧字段的代码都注释为 `// [MOCN] OLD CODE`
- 新代码优先使用 `served_plmn_list[]`
- `gnb_config.c` 中的 `set_snssai_config(info->nssai, ...)` **保留**（已在前面的修改中处理）

**使用旧字段的地方**（已全部修改）:
| 位置 | 状态 |
|------|------|
| `encode_served_cell_info()` | ✅ 已改为 `served_plmn_list[]` |
| `decode_served_cell_info()` | ✅ 已改为 `served_plmn_list[]` |
| `eq_f1ap_cell_info()` | ✅ 已改为 `served_plmn_list[]` |
| `copy_f1ap_served_cell_info()` | ✅ 已改为 `served_plmn_list[]` |

---

### **Q3: 哪些逻辑需要改成使用 served_plmn_list？**

✅ **已修改的逻辑**:
1. ✅ F1AP 编码层（`encode_served_cell_info`）
2. ✅ F1AP 解码层（`decode_served_cell_info`）
3. ✅ 比较函数（`eq_f1ap_cell_info`）
4. ✅ 复制函数（`copy_f1ap_served_cell_info`）

✅ **不需要修改的部分**:
- `info->plmn` 字段 **必须保留**，用于 NR CGI（Global Cell ID）
- RRC 层使用 `info->plmn` 是正常的，因为 Cell ID 只有一个主 PLMN

---

### **Q4: 添加 LOG 和保留旧代码为注释**

✅ **LOG 输出位置**:

**Encode 端** (`encode_served_cell_info`):
```
[F1AP][I] [MOCN] encode_served_cell_info: Encoding 2 PLMNs
[F1AP][I] [MOCN]   PLMN[0]: MCC=460, MNC=11, num_slices=1
[F1AP][I] [MOCN]     Slice[0]: SST=1, SD=0x010204
[F1AP][I] [MOCN]   PLMN[1]: MCC=208, MNC=93, num_slices=1
[F1AP][I] [MOCN]     Slice[0]: SST=1, SD=0x010203
```

**Decode 端** (`decode_served_cell_info`):
```
[F1AP][I] [MOCN] decode_served_cell_info: Decoding 2 PLMNs
[F1AP][I] [MOCN]   PLMN[0]: MCC=460, MNC=11, num_slices=1
[F1AP][I] [MOCN]     Slice[0]: SST=1, SD=0x010204
[F1AP][I] [MOCN]   PLMN[1]: MCC=208, MNC=93, num_slices=1
[F1AP][I] [MOCN]     Slice[0]: SST=1, SD=0x010203
```

✅ **所有旧代码都保留为注释**，标记为 `// [MOCN] OLD CODE - kept for reference/rollback`

---

## 🔧 编译和测试

### **编译**:
```bash
cd ~/openairinterface5g
./cyh_build_oai.sh
```

### **运行 gNB**:
```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf --gNBs.[0].min_rxtxtime 6 --rfsim
```

### **预期日志**:
1. ✅ `read_du_cell_info`: 显示 2 个 PLMN 被读取
2. ✅ `encode_served_cell_info`: 显示 2 个 PLMN 被编码
3. ✅ F1AP Setup Request 成功发送到 CU
4. ✅ CU 端 `decode_served_cell_info`: 显示 2 个 PLMN 被解码

---

## 🔄 回滚方案

如果需要回滚，只需：
1. 删除 `// [MOCN] NEW CODE` 部分
2. 取消注释 `// [MOCN] OLD CODE` 部分
3. 重新编译

---

## 📝 下一步工作

1. ✅ 测试编译是否通过
2. ✅ 运行 gNB，检查 LOG 输出
3. ⏳ 验证 F1AP Setup Request 是否包含 2 个 PLMN
4. ⏳ 测试 UE 注册到不同 PLMN
5. ⏳ 更新单元测试（`f1ap_lib_test.c`）

---

## 📖 相关文档

- 3GPP TS 38.473: F1 Application Protocol (F1AP)
- 3GPP TS 23.501: System architecture for 5G
- `SLICE_CONFIGURATION_GUIDE.md`: 网络切片配置指南
