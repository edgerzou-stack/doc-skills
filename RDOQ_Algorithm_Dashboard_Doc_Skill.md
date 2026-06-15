# RDOQ 算法原理 Dashboard 文档技能 (RDOQ Algorithm Dashboard Doc Skill)

本文档提炼自 [`rdo-architecture`](https://github.com/edgerzou-stack/rdo-architecture) 中 `html/rdoq_hardware_dashboard.html` 的迭代经验。用于指导 AI 或工程师编写/重构 **RDOQ 算法 + 硬件优化** 类交互式 HTML 规格书。

> **参考成品**：[RDOQ Hardware Dashboard (GitHub Pages)](https://edgerzou-stack.github.io/rdo-architecture/html/rdoq_hardware_dashboard.html)

---

## 1. 第一章必须采用「三大块」结构

RDOQ 算法原理章（第一章）**禁止**按「扫描 / 候选 / CABAC」平铺三节，必须按读者决策路径组织：

| 节号 | 标题 | 职责 |
|------|------|------|
| **1.1** | 整体过程与原理 | 扫描路径、数据冒险、候选 Level、**整体判决示例**、Cost 公式桥 |
| **1.2** | Dcost（失真代价）的计算 | 反量化误差、Parseval、定点实现式、配图 |
| **1.3** | Bincost（码率代价）的计算 | CABAC 四步链路、ctxInc、Context_Array、Entropy_LUT 建表与查表 |

**承上启下公式块**（放在 1.1 末尾、1.2 之前）必须同时包含：
- 概念式：`Cost = Dcost + λ × Bincost`
- 定点实现式图：`rdoq_formula.png`（`d²·2^scaleBits + ⌊λ·bits/2^shift⌋`）

---

## 2. 内容归类与放置规则（Content Placement Matrix）

| 内容类型 | 应放置位置 | 禁止放置 |
|----------|------------|----------|
| 决策树（D+R 综合判决） | **1.1** 整体判决示例 | 1.3 末尾 |
| Cost 定点公式图 | **1.1** 承上启下 | 1.3 末尾 |
| Dcost 误差 / Parseval 图 | **1.2** | 1.1 / 1.3 |
| CABAC 四步链路、ctxInc 推演 | **1.3** | 1.1 |
| Entropy_LUT 建表三步 | **1.3** 图下方文字 | 叠在曲线图上 |
| 嵌套循环宏观流程图 | **可省略**（文字总结足够） | 与扫描动画重复 |

**章节归属判断**：若图表核心是 **Cost = D + R 拔河**，归 1.1；若核心是 **CABAC 上下文 / 查表**，归 1.3；若核心是 **系数域平方误差**，归 1.2。

---

## 3. 目录（TOC）一致性规则

### 3.1 仅 `h2` / `h3` 进入侧边栏

TOC 脚本标准选择器：

```javascript
document.querySelectorAll('.container h2, .container h3:not(.algorithm-steps h3)')
```

### 3.2 章节标题必须露出 `.algorithm-steps` 外

**错误**：把 `1.1 / 1.2 / 1.3` 写在 `.algorithm-steps` 盒子内的 `<h3>` —— 会被 TOC 排除，导致目录只有第二章子节。

**正确**：`h3` 放在 `.algorithm-steps` **之前**，盒子内用 `<h4>` 作子标题。

### 3.3 禁止重复节号

卡片/示例块的紫色标题用 **「整体判决示例」**，**不要**再写 `1.1 xxx`，避免与目录中的 `1.1 整体过程与原理` 冲突。

---

## 4. 图表 vs 正文分工（Image / Text Separation）

| 图表 | 图内保留 | 图下正文承担 |
|------|----------|--------------|
| `rdoq_entropy_lut.png` | 三条曲线 | α、P_LPS、LUT 建表公式、抽样表 |
| `rdoq_decision_tree.png` | 完整 D/R 节点与数值 | 三栏文字可补充，不与图重复删细节 |
| `rdoq_dcost.png` | 误差 d + D(1)/D(0) 数值 | 三个关键点文字推导 |
| Mermaid 简单流程 | 节点名 | `.diagram-caption` 图注 |

**原则**：Matplotlib / 曲线类图 **禁止** 叠加 LaTeX 公式框；Graphviz 决策类图 **保留** 教学所需的数值节点。

---

## 5. Bincost 四步全链路推演模板（1.3 必备）

对单个语法元素（如 `sig_coeff_flag=1`）必须走完四步，数字自洽：

1. **语法元素 → ctxInc**：BaseOffset + LumaCGOffset + cnt  
2. **Context_Array → m_ucState**：initValue + QP → slope/offset/initState  
3. **建 Entropy_LUT 并定位**：pStateIdx → LUT[2s] / LUT[2s+1]  
4. **查出 bit 数**：`(pStateIdx<<1) | (flag_val ^ valMPS)` → Fractional Bits  

**删除**粗糙的「Level=3 逐项估 bit」列表；有四步推演即可。

---

## 6. 配图生成脚本（本地，不入 Git）

脚本保留在本地工程目录（如 `KHEnc_Perftest/`），**不**提交到 `rdo-architecture`（该仓仅追踪 HTML + PNG）。

| 脚本 | 输出 |
|------|------|
| `draw_decision_tree.py` | `rdoq_decision_tree.png` |
| `draw_dcost.py` | `rdoq_dcost.png` |
| `draw_entropy_lut.py` | `rdoq_entropy_lut.png`（无图内公式） |
| `draw_formula.py` | `rdoq_formula.png` |

生成后复制 PNG 到 `rdo-architecture/html/`，再 `git add` 白名单文件。

---

## 7. 部署检查清单

```
- [ ] 第一章仅 3 个 h3：1.1 / 1.2 / 1.3
- [ ] 1.1 含决策树 + Cost 公式，无重复「1.1」卡片标题
- [ ] 1.2 含 rdoq_dcost.png
- [ ] 1.3 含四步推演，无 Level=3 粗糙示例
- [ ] Entropy_LUT 图下含建表三步，图上无公式
- [ ] div 开闭标签平衡（脚本：`grep -c '<div'` vs `'</div>'`）
- [ ] README Live Link 描述与三章结构一致
- [ ] 同步 KHEnc_Perftest 本地副本（若存在）
```

---

## 8. 与通用 Skill 的关系

| Skill | 关系 |
|-------|------|
| [Hardware_Interactive_Doc_Generation_Skill.md](Hardware_Interactive_Doc_Generation_Skill.md) | 布局、TOC、Mermaid/Graphviz 通用规范 |
| [Git_Documentation_Deployment_Skill.md](Git_Documentation_Deployment_Skill.md) | HTML-only 白名单部署 |
| [cmodel_strict_alignment_skill.md](cmodel_strict_alignment_skill.md) | CABAC/ctxInc 数值必须与 C-Model 对齐 |
| **本文档** | RDOQ Dashboard **章节编排与内容归类** 专用 |

**指令示例**：

> 请按《RDOQ Algorithm Dashboard Doc Skill》重构第一章：1.1 整体过程 + 判决示例，1.2 Dcost，1.3 Bincost；修正 TOC；Entropy_LUT 公式放图下。
