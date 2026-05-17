# Haloalkane 项目 — 交接说明与技术文档

## 一、项目目标

开发一个基于浏览器的**多取代卤代烷烃立体异构体全自动分析计算器**（Web WASM 版）。

- **输入**：任意合法饱和烷烃的 SMILES + 取代数量 X（如 X=2）
- **输出**：所有唯一立体异构体的 InChI、SMILES 及总数
- **Python 对齐基准**：扭曲烷 `C1C2CC3CC1CC(C2)C3` 二氯代（X=2），Python RDKit `EnumerateStereoisomers` 结果为 **84 个**唯一立体异构体（✅ 经用户实测确认）

> ⚠️ 注意：此前交接文档中误写为 384，该数字系另一个 AI 虚构，与 Python RDKit 实际运行结果不符，已废弃。

---

## 二、技术背景

### 2.1 RDKit WASM 环境限制（核心天坑）

前端 RDKit WASM (`RDKit_minimal.js`) 相比 Python RDKit 有以下关键限制：

| 功能 | Python RDKit | 前端 WASM |
|------|-------------|-----------|
| `EnumerateStereoisomers` | ✅ 完整 | ❌ 不存在 |
| `MolList` 容器 | ✅ 支持 | ❌ 不存在 |
| `run_reactants` 反应器 | ✅ 支持 | ❌ 不存在 |
| `get_mol()` | ✅ 完整 | ✅ 可用 |
| `get_inchi()` | ✅ 完整 | ⚠️ 部分版本无 |
| `get_canonical_smiles()` | ✅ 完整 | ✅ 可用 |
| 手性 InChI 层 (`/t /m /s`) | ✅ 完整保留 | ⚠️ 需手动触发 |

### 2.2 正确的手性处理策略

立体异构体的核心难点在于**手性中心的空间构型枚举**：

1. **组合学位置枚举**：先用 JS 生成所有可能的取代位置组合（碳原子索引的组合）
2. **手性位二进制掩码**：对每一个位置组合，生成 $2^M$ 种手性构型（$M$ = 取代碳原子数）
3. **动态 SMILES 手性标记**：将 `@` / `@@` 标记注入 SMILES 字符串表示每个碳的手性
4. **RDKit 验证去重**：将构造的 SMILES 喂给 `get_mol()` 获取规范表达，用 InChI 或 canonical SMILES 做去重

### 2.3 SMILES 手性语法

```
[C@]  — 手性碳 (反时针)
[C@@] — 手性碳 (顺时针)

# 单氯代示例（一个手性中心，2种构型）：
[C@](Cl)C[C@@](Cl)C  ← 第一个碳反时针，第二个碳顺时针
[C@](Cl)C[C@](Cl)C   ← 两个碳都是反时针
[C@@](Cl)C[C@](Cl)C  ← 第一个碳顺时针，第二个碳反时针
[C@@](Cl)C[C@@](Cl)C ← 两个碳都是顺时针
```

---

## 三、现有文件

```
haloalkane_tmp/
  index.html           ← 主程序（含 RDKit WASM 调用逻辑）
  RDKit_minimal.js     ← RDKit WASM 绑定（无需修改）
  RDKit_minimal.wasm   ← RDKit WASM 二进制（无需修改）
```

---

## 四、算法流程（当前版本）

```
输入 SMILES + X (取代数)
    ↓
RDKit get_mol() 验证分子，get_substruct_matches("[#6]") 提取所有碳原子索引
    ↓
JS combinatorics: 生成分碳原子索引的全部 C(X,N) 组合
    ↓
对每一个位置组合 (combo):
    │  对每一个手性掩码 s ∈ [0, 2^M):
    │    ├─ 将 combo 中的碳原子替换为 [C@](Cl) 或 [C@@](Cl)（共 M 个）
    │    ├─ 用 get_mol() 验证生成的 SMILES 是否合法
    │    ├─ get_canonical_smiles() 获取规范 SMILES
    │    └─ 用 canonical SMILES 做 Set 去重
    ↓
输出全部去重后的结果
```

---

## 五、已知问题与解决方向

### 5.1 SMILES 构造的正确性

当前版本通过字符串替换构造手性 SMILES，但**SMILES 语法复杂**（环括号、支链、立体标记）导致替换逻辑容易出错。

**改进方向**：
- 使用 RDKit WASM 的 `get_mol()` + 分子图操作 API（而非字符串替换）
- 或利用 RDKit 导出 molblock/connectivity table，然后用程序化方式重建带手性的分子

### 5.2 手性中心判断

并非所有取代碳都是手性中心——只有当 4 个取代基各不同时才产生手性。

**当前版本**：假设所有被取代的碳都是手性中心（偏保守，结果可能略多于 84）。

### 5.3 关于 84 的说明

84 是 Python RDKit `EnumerateStereoisomers` 对扭曲烷二氯代的实际输出，代表：
- 所有平面位置异构体（由拓扑对称性决定）
- 乘以每个手性中心的 2 种空间构型组合
- 去重后（有些组合会因对称性变成同一个分子）

---

## 六、交接给下一个 AI 的任务

**目标**：让前端计算结果与 Python 的 84 尽可能对齐（允许 ±5 的误差范围）。

**核心要求**：
1. 不使用任何不存在的 API（`MolList`、`get_rxn()`、`run_reactants`）
2. 不硬编码 84、10、16 等特定数值
3. SMILES 手性标记必须真实反映空间构型
4. 结果用 RDKit `get_mol()` + `get_canonical_smiles()` 验证

**建议方案**：放弃字符串级别的 SMILES 替换，改为利用 RDKit 的 molecule 对象操作。可参考方向：
- 用 `get_molblock()` 解析分子连接表
- 用程序化方式重建带手性信息的分子
- 或将 RDKit 的 `Mol` 对象序列化成规范 SMILES 的过程reverse engineering