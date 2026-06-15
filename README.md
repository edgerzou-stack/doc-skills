# Doc Skills — 硬件交互式文档生成与部署

面向硬件架构师 / RTL 工程师：从 Word/Excel 静态文档，迁移到 **零横向滚动、Git 版本化、可交互** 的 HTML Dashboard。

本仓库按 **一条工作流、四类能力 + 仓库治理** 组织，不再为单个模块（如 RDOQ）单独拆 Skill 文件。

---

## 推荐工作流

```
整理 / 合并 Skill 规则（Organization Skill）  ← 新增模块经验时先过一遍
    ↓
本地研发（.py / .docx / C-Model 对照）
    → 生成 HTML + PNG（Interactive Doc Skill）
    → 内容精度与 C-Model 对齐（Strict Alignment Skill）
    → 算法-硬件妥协写入第二章（Co-design Skill）
    → 白名单 Git 推送 GitHub Pages（Deployment Skill）
```

---

## Skill 索引

### 0. [Doc Skills Organization Skill](Doc_Skills_Organization_Skill.md)

**Skill 仓库本身怎么整理、合并、避免按模块越拆越乱。**

按工作流维度分类（非按 RDOQ/Deblock 等 IP）；内容归属决策树；合并 SOP；何时才允许新建 Skill 文件；README 与工作流同步规范。  
**新增任何模块文档经验前，先读本 Skill。**

### 核心 Skill（4 份）

### 1. [Hardware Interactive Doc Generation Skill](Hardware_Interactive_Doc_Generation_Skill.md)

**文档怎么写、怎么排、怎么画。**

| 能力块 | 内容 |
|--------|------|
| Dashboard 框架 | Sidebar + Top Nav + 自动 TOC、色彩与排版 |
| 图表工具链 | Mermaid / Graphviz / CSS 空间图、甘特时序 |
| 章节编排 | TOC 与 `.algorithm-steps` 配合、DOM 顺序约束 |
| **算法原理章模板** | **§9 RDOQ Dashboard 实战**（1.1 整体 / 1.2 Dcost / 1.3 Bincost、内容归类、图/文分工、检查清单） |
| 动态交互 | 扫描动画、MathJax（详见 Dynamic Simulation Skill 交叉引用） |

### 2. [Git Documentation Deployment Skill](Git_Documentation_Deployment_Skill.md)

**怎么推上 GitHub Pages 且仓库保持干净。**

Git 索引剥离、`gitignore` 白名单（仅 HTML + PNG + README）、README 导航站、鉴权陷阱。

### 3. [Hardware-Algorithm Co-design Skill](Hardware_Algorithm_CoDesign_Skill.md)

**算法章写什么妥协、为什么不会 Mismatch。**

编解码一致性心理包袱、Fast/Slow Loop、Luma-Chroma Bypass、尺寸剪枝等非对称架构。

### 4. [C-Model Strict Alignment Skill](cmodel_strict_alignment_skill.md)

**写什么必须和 C-Model 一字不差。**

拒绝夸张修辞、时序依赖、图注强制、ROI 类比、变量物理意义；**§9 Bincost 四步推演**（ctxInc → m_ucState → LUT → bit 数）。

### 附： [Hardware Dynamic State Simulation Skill](Hardware_Dynamic_State_Simulation_Skill.md)

动态扫描动画、Grid/Flexbox 状态机、MathJax 公式渲染——作为 **Interactive Doc Skill §5 的扩展**，不单独构成一类文档。

---

## 参考成品

| 文档 | 链接 |
|------|------|
| 架构总览 | [index.html](https://edgerzou-stack.github.io/rdo-architecture/html/index.html) |
| RDOQ 算法 + 硬件优化 | [rdoq_hardware_dashboard.html](https://edgerzou-stack.github.io/rdo-architecture/html/rdoq_hardware_dashboard.html) |
| 流水线顺序与气泡 | [pipeline_comparison_static.html](https://edgerzou-stack.github.io/rdo-architecture/html/pipeline_comparison_static.html) |

源码仓：[edgerzou-stack/rdo-architecture](https://github.com/edgerzou-stack/rdo-architecture)

---

## 给 AI 的指令示例

> 按 **Organization Skill** 把 `{模块}` 文档经验合并进现有 Skill 的 §N，更新 README，删除独立模块 Skill 文件。

> 按 **Doc Skills 工作流** 重构 `rdoq_hardware_dashboard.html` 第一章：遵循 Interactive Doc §9 三章结构 + Strict Alignment §9 四步 Bincost；修正 TOC；Entropy_LUT 公式放图下；完成后按 Deployment Skill 推送。

---

*Empowering Hardware Engineers with Modern Web UI/UX.*
