# Doc Skills — 硬件交互式文档生成与部署

![Doc Skills](https://img.shields.io/badge/Doc_Skills-Workflow-3b82f6?style=for-the-badge)
![Web Dashboard](https://img.shields.io/badge/Web-Dashboard-ec4899?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Active-10b981?style=for-the-badge)

Welcome to the **Doc Skills** repository. 
面向硬件架构师 / RTL 工程师：从 Word/Excel 静态文档，迁移到 **零横向滚动、Git 版本化、可交互** 的 HTML Dashboard。

本仓库按 **一条工作流、四类能力 + 仓库治理** 组织，不再为单个模块单独拆 Skill 文件。

---

## 🚀 推荐工作流 (Workflow)

```
整理 / 合并 Skill 规则（Organization Skill）  ← 新增模块经验时先过一遍
    ↓
本地研发（.py / .docx / C-Model 对照）
    → 生成 HTML + SVG/PNG（Interactive Doc Skill）
    → 内容精度与 C-Model 对齐（Strict Alignment Skill）
    → 算法-硬件妥协写入第二章（Co-design Skill）
    → 白名单 Git 推送 GitHub Pages（Deployment Skill）
```

---

## 🛠️ 核心 Skill 矩阵 (Core Skills)

### 0. [Doc Skills Organization Skill](Doc_Skills_Organization_Skill.md)
**Skill 仓库本身怎么整理、合并、避免按模块越拆越乱。**
- 按工作流维度分类（非按 IP 模块）；内容归属决策树；合并 SOP。
- **新增任何模块文档经验前，先读本 Skill。**

### 1. [Hardware Interactive Doc Generation Skill](Hardware_Interactive_Doc_Generation_Skill.md)
**文档怎么写、怎么排、怎么画。**
- **Dashboard 框架**：Sidebar + Top Nav + 自动 TOC、色彩与排版。
- **图表工具链**：Python (Graphviz) 静态 SVG / Mermaid / CSS 空间图。
- **动态交互扩展**：扫描动画、MathJax 公式原生渲染（附录10）。
- **算法原理章模板**：RDOQ Dashboard 实战（整体/Dcost/Bincost）、内容归类。

### 2. [Git Documentation Deployment Skill](Git_Documentation_Deployment_Skill.md)
**怎么推上 GitHub Pages 且仓库保持干净。**
- Git 索引剥离、`gitignore` 白名单（仅 HTML + SVG/PNG + README）、鉴权陷阱。

### 3. [Hardware-Algorithm Co-design Skill](Hardware_Algorithm_CoDesign_Skill.md)
**算法章写什么妥协、为什么不会 Mismatch。**
- 编解码一致性心理包袱、Fast/Slow Loop、Luma-Chroma Bypass、尺寸剪枝等。

### 4. [C-Model Strict Alignment Skill](cmodel_strict_alignment_skill.md)
**写什么必须和 C-Model 一字不差。**
- **客观严谨**：拒绝夸张情绪化修辞、严禁无意义奉承，坚守工程底色。
- 时序依赖、图注强制、ROI 类比、变量物理意义溯源。
- **CABAC 四步推演**：ctxInc → m_ucState → LUT → bit 数。

---

## 🌟 参考成品展示 (Reference Dashboards)

| 文档名称 | 访问链接 |
|------|------|
| **架构总览** | [index.html](https://edgerzou-stack.github.io/rdo-architecture/html/index.html) |
| **CABAC Intra/Inter 解析 (静态 SVG)** | [intra_inter_scabac_dashboard.html](https://edgerzou-stack.github.io/rdo-architecture/html/intra_inter_scabac_dashboard.html) |
| **RDOQ 算法 + 硬件优化** | [rdoq_hardware_dashboard.html](https://edgerzou-stack.github.io/rdo-architecture/html/rdoq_hardware_dashboard.html) |
| **流水线顺序与气泡** | [pipeline_comparison_static.html](https://edgerzou-stack.github.io/rdo-architecture/html/pipeline_comparison_static.html) |

🔗 **源码仓：** [edgerzou-stack/rdo-architecture](https://github.com/edgerzou-stack/rdo-architecture)

---

## 🤖 给 AI 的指令示例

> 按 **Organization Skill** 把 `{模块}` 文档经验合并进现有 Skill 的 §N，更新 README，删除独立模块 Skill 文件。

> 按 **Doc Skills 工作流** 重构 `rdoq_hardware_dashboard.html` 第一章：遵循 Interactive Doc §9 三章结构 + Strict Alignment §9 四步 Bincost；图表改用 Python SVG；完成后按 Deployment Skill 推送。

---
*Empowering Hardware Engineers with Modern Web UI/UX.*
