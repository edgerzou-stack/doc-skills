# Hardware Architecture Dashboard Generation Skill (Native EDA Style)

本文档总结了将传统硬件详细设计文档（Markdown/Word）转化为**“现代化、交互式、极客风 Web Dashboard”**的核心技能（Skill）与最佳实践。在未来的硬件设计文档开发中，应严格遵循本套体系进行构建。

## 1. 核心设计理念 (Core Philosophy)
*   **摒弃静态文档**：不再使用干瘪的 Markdown 或 PDF，而是构建拥有左侧全局导航（Sidebar）和右侧内容区（Content Pane）的单页面 Web 应用（SPA）。
*   **极致的极客美学**：采用类似顶级开发者文档（如 Stripe, Tailwind）的视觉规范。深色顶栏、玻璃拟态（Glassmorphism）吸顶导航、高对比度语法高亮。
*   **原生 EDA 级图表**：不依赖外部低清图片，使用原生 SVG、HTML/CSS 渲染的 Gantt 图、Python + Graphviz 生成的架构图（复杂场景）以及 Mermaid 内联图表（简单场景），确保在 4K 屏幕下依然无限放大不失真。
*   **极致图文并茂 (Visual-First Strategy)**：技术原理的解释绝不能是干瘪的纯文本。**强制约束：每一段核心逻辑或算法思想的文字说明，都必须配有一张精心设计的可视化图表（Diagram），并附带对应的中文算法步骤流程（Algorithm Steps）**，做到“文字释义 + 架构图表 + 中文算法步骤”三位一体，让读者能够瞬间建立直观且深刻的理解。不建议使用纯代码语法的伪代码，应使用自然语言与流程图结合的方式表达清楚。

## 2. UI 框架与排版规范 (UI & Layout Guidelines)
*   **字体栈 (Typography)**：正文优先使用系统级无衬线字体（如 `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto`），代码和图表标识严格使用等宽字体（`'Roboto Mono', monospace` 或 `Consolas`）。
*   **色彩系统 (Color Palette)**：
    *   主色调采用深邃的科技蓝/极客黑（如 `#0f172a`, `#1e293b`）。
    *   文字使用板岩灰（`#334155` 用于标题，`#475569` 用于正文）。
    *   突出警示信息采用高饱和度色彩（如红色警示框 `#fef2f2` 背景 + `#ef4444` 左边框）。
*   **交互细节**：表格增加 Hover 斑马线效果，侧边栏导航点击高亮并支持平滑滚动（`scroll-behavior: smooth`）。

## 3. 高级硬件图表可视化 (Advanced Hardware Visualizations)

这是本套 Skill 的核心灵魂，必须摒弃传统的粗糙表达方式：

### 3.1 物理流水线拓扑图 (Custom SVG Datapath)
*   **布局策略**：对于芯片内部的物理数据流，采用原生 SVG 绘制 **“Z字型” (S-shape)** 或折返式流水线拓扑，模拟真实的物理走线，极大地节省横向空间。
*   **尺寸与防压缩策略 (Critical)**：
    *   画布宽度必须足够大（如 `1400px`），模块文字字号至少 **20px**，连线位宽标识至少 **14px**。
    *   **致命陷阱规避**：浏览器会自动压缩超大 SVG。必须在 CSS 中为 SVG 注入强硬规则：`min-width: 1400px !important; max-width: none !important; height: auto !important;`。
    *   外层包裹 `div` 必须设置 `overflow-x: auto`，并且对齐方式**必须使用 `justify-content: flex-start;`**，绝对不能用 `center`，否则超出的左侧内容会被永久裁切！

### 3.2 周期级时序图 (Native HTML/CSS Gantt)
*   **痛点**：传统 Markdown 表格无法直观表达硬件的 Clock Cycle 时序。
*   **解决方案**：使用 `div` 栅格拼接出来的原生甘特图（Gantt Chart）。
*   **结构**：左侧固定表头（信号名/模块名），右侧为横向可滚动的周期网格。通过精准计算 `width` 和 `left` 偏移量来绘制彩色 Block，完美表达硬件加法器复用、流水线 Bubble、握手等待等精细时序。

### 3.3 图表工具选择决策（Mermaid vs Python + Graphviz vs CSS）

在绘制架构图时，根据图表复杂度选择合适的工具：

| 图表类型 | 推荐工具 | 判断标准 |
|---------|---------|--------|
| 简单线性流程（节点数 ≤ 5、无分支） | **Mermaid** | 直接写在 HTML 中，零构建成本 |
| 简单状态机、序列图 | **Mermaid** | Mermaid 对这类图的支持非常好 |
| 多分支流水线、RDO/IPD 环路 | **Python + Graphviz** | 需要精确控制布局和连线路由 |
| 节点数 > 5 且有复杂连线 | **Python + Graphviz** | Mermaid 布局引擎在此场景下不可控 |
| **强物理空间依赖、块状拆分架构** | **CSS Native Layout** | 直接用 HTML/CSS 绝对定位/Grid/Flexbox 手绘，展现空间物理坐标与极致颜值 |

**核心原则：如果你发现自己在 Mermaid 上需要写大量 CSS hack 或排版补丁来修复布局，说明应交给 Python+Graphviz；如果你需要表达具有强烈长宽、位置、划分比例的物理空间概念，绝对不要用节点图，直接用 CSS 画图！**

#### Mermaid 简单用法（仅限简单图表）
*   直接在 HTML 中使用 `.mermaid` 容器内联书写
*   节点文本中包含特殊字符时，必须用双引号包裹
*   下方必须紧跟 `.diagram-caption` 图注

#### Python + Graphviz 用法（复杂图表）
*   详见本文档末尾的《Python + Graphviz 架构图生成技能指南》

### 3.4 CSS 原生空间排版绘图 (CSS Native Spatial Layouts)

对于具有**强空间属性和相对位置关系**的架构概念（例如：视频编码中的 CU 块划分、CTU/TU 的二维网格映射、相邻宏块的空间依赖等），**坚决抛弃传统的节点连线图**，改用完全原生的 HTML/CSS 手绘排版。

*   **直观空间表达**：将抽象的算法数据流直接“放在它应该在的二维物理位置上”。例如，使用 `position: absolute` 把“上方依赖块”和“左侧依赖块”以真实比例直接摆放在“当前块”的上方和左侧。
*   **极致极客美学 (Aesthetics-First)**：
    *   **色彩与材质**：大量使用现代化的线性渐变 (`linear-gradient`)、发光与内阴影 (`box-shadow: inset...`)、半透明叠加态等高级质感。
    *   **数据流转**：抛弃单调的细线箭头，改用具有立体感的精美胶囊按钮（带粗体字、底色和 SVG 箭头图标）来表示数据抽取流向。
*   **最佳实践场景**：
    *   **空间结构树展示**：在一个代表最大面积的全局方框（如 64x64）中，按比例进行二叉/三叉划分，并对内部独立区域用不同主题色高亮“复用区”与“计算区”，其表达效果吊打任何纯文字或树状图。
    *   **算法流向视图**：左侧是强空间约束的输入块阵列，中间是精致的卡片式“算法处理中枢”，右侧发散成精美的属性结果列表，彻底重构技术讲解体验。

## 4. 硬件代码与高亮 (RTL Code & Syntax Highlighting)
*   **引擎**：集成 `highlight.js`，并必须显式加载 `verilog.min.js` 语言包。
*   **生命周期**：确保 HTML 中的 JS 加载顺序：`核心包 -> verilog语言包 -> hljs.highlightAll()`。HTML 标签必须使用 `<code class="language-verilog">`（注意不是 systemverilog，引擎只认 verilog）。
*   **主题覆写 (Theme Override)**：
    *   采用 `Atom One Dark` 等深色极客主题。
    *   **注释高亮补丁**：默认的暗灰色注释在深色背景下可视性极差。必须在全局 CSS 注入补丁：`.hljs-comment { color: #86efac !important; font-style: italic; }`，将注释改为亮浅绿色斜体，大幅提升阅读体验。

## 5. 自动化构建工具流 (Python Parser Workflow)
*   文档由 Python 脚本自动将散落的资料组合、解析并生成最终的 HTML。
*   **正则表达式与锚点**：脚本需具备自动扫描全文标题、生成侧边栏锚点并注入 HTML 对应 `id` 的能力。
*   **特殊块拦截**：脚本应当能够识别特定的标记（如 `[GANTT_START]`、`[SVG_DATAPATH_START]`），并在这些位置注入高阶可视化组件的代码。

## 6. 侧边栏目录与弹性布局 (Sidebar TOC & Flexbox Layout)
*   **布局重构**：抛弃传统的单栏居中流式布局。在全局 `body` 使用 `display: flex; height: 100vh; overflow: hidden;`，将页面切分为左侧固定的 `.sidebar` 和右侧滚动的 `.main-content`，以匹配专业级 Dashboard 体验。
*   **目录自动生成 (Vanilla JS)**：
    *   绝不手动编写锚点目录。必须在 HTML 底部注入 Vanilla JS 脚本。
    *   脚本逻辑：使用 `document.querySelectorAll('h2, h3')` 提取大纲，为每个 DOM 节点赋予唯一 `id`，然后在 `.sidebar` 中动态生成包裹 `<a>` 标签的 `<li>` 列表。
*   **交互联动**：
    *   **平滑跳转**：为 TOC 链接绑定 `click` 事件并调用 `scrollTo({ behavior: 'smooth' })`。
    *   **滚动监听 (Scroll Tracking)**：监听右侧主内容的 `scroll` 事件，动态计算当前到达的标题位置，实时高亮左侧 TOC 中的对应链接 (`.active` 样式)。

### 6.1 TOC 与 `.algorithm-steps` 盒子的配合

*   **章节 `h3` 必须在盒子外**：若 `1.1 / 1.2 / 1.3` 写在 `.algorithm-steps` 内部，选择器 `.container h3:not(.algorithm-steps h3)` 会**漏掉整章**，目录与正文严重错位。
*   **盒子内用 `h4`**：示例子块（如「整体判决示例」「候选 Level 产生」）用 `<h4>`，避免与节号 `h3` 竞争 TOC。
*   **卡片标题禁止重复节号**：紫色卡片头写「整体判决示例」，不写「1.1 xxx」。

## 9. 算法原理章编排模板（实战：RDOQ Dashboard）

> 新增其他模块（Deblock、SAO…）的同类附录时，遵循 [Doc_Skills_Organization_Skill.md](Doc_Skills_Organization_Skill.md)——写入本 Skill 的 §N 实战，**禁止**再建 `{Module}_Doc_Skill.md`。

适用于「算法原理 + 硬件优化」双章结构。参考成品：[rdoq_hardware_dashboard.html](https://edgerzou-stack.github.io/rdo-architecture/html/rdoq_hardware_dashboard.html)。

### 9.1 第一章三大块（禁止按扫描/CABAC 平铺）

| 节号 | 职责 |
|------|------|
| **1.1** 整体过程与原理 | 扫描、数据冒险、候选 Level、**整体判决示例**、Cost 公式桥 |
| **1.2** Dcost 的计算 | 反量化误差、Parseval、定点式、配图 |
| **1.3** Bincost 的计算 | CABAC 四步链路、ctxInc、Context_Array、Entropy_LUT |

1.1 末尾「承上启下」须同时含概念式 `Cost = Dcost + λ×Bincost` 与定点图 `rdoq_formula.png`。

### 9.2 内容归类

| 内容 | 放哪里 | 禁止 |
|------|--------|------|
| 决策树（D+R 判决） | 1.1 | 1.3 末尾 |
| Cost 定点公式 | 1.1 | 1.3 末尾 |
| Dcost / Parseval 图 | 1.2 | 1.1 / 1.3 |
| CABAC 四步推演 | 1.3 | 1.1 |
| Entropy_LUT 建表公式 | 1.3 图下文字 | 叠在曲线图上 |

判断法：Cost 拔河 → 1.1；系数域 SSE → 1.2；CABAC 上下文 → 1.3。

### 9.3 图表 vs 正文

*   **曲线图**（Entropy LUT）：图只画形态，α/P_LPS/LUT 公式放图下。
*   **决策树**（Graphviz）：保留 D/R 数值节点；粗糙 Level=3 列表删掉。
*   **配图脚本**（本地 `.py`，不入 Git）：`draw_decision_tree.py`、`draw_dcost.py`、`draw_entropy_lut.py`、`draw_formula.py` → 复制 PNG 到 `html/` 再提交。

### 9.4 重构检查清单

```
- [ ] 第一章仅 3 个 h3：1.1 / 1.2 / 1.3
- [ ] 1.1：决策树 + Cost 公式，卡片无重复节号
- [ ] 1.2：rdoq_dcost.png
- [ ] 1.3：四步推演完整，无 Level=3 粗糙示例
- [ ] Entropy_LUT 图下含建表三步
- [ ] div 开闭平衡；README 与章节结构一致
```

## 7. 核心突破机制的视觉强调 (Critical Mechanism Highlighting)
*   **痛点与突破口分离**：对于架构中最核心的“作弊/解耦”机制（例如：为了流水线解耦，强行使用**原始像素**替代**重建像素**来进行模式粗选），这种颠覆常规理论的突破点，必须与普通的说明文字拉开视觉差距。
*   **强制使用预警/强提醒样式 (`.critical-alert`)**：
    *   绝不可使用普通的 `<div class="highlight">`（那只是用于补充说明或知识普及）。
    *   必须使用带醒目边框、刺眼底色和特殊字体的 `.critical-alert` 样式（如红色左边框、淡红底色）。
    *   内部的核心关键字词，还要进一步嵌套 `<span style="background: #fecaca; padding: 2px 6px; border-radius: 4px; color: #991b1b; font-weight: 700;">` 这种极其抢眼的底色高亮。
    *   **目的**：让硬件架构师或算法人员在快速扫视文档时，第一眼就能被这个极其重要的架构魔法设计“刺瞎”，确保知识传达零遗漏。

## 8. 防呆与一致性约束 (DOM & Navigation Consistency)
*   **物理 DOM 顺序强约束**：在进行文档层级重构或章节移动时，**绝对禁止**只修改标题序号（如把 3.1 改为 1.2）而不迁移底层 HTML 代码块的“掩耳盗铃”式操作。所有正文段落、图表 div 必须在 HTML 文件中与它们的父级 `<h>` 标签保持严格的**物理从上到下**顺序，且务必检查 `</div>` 标签的完美闭合，防止整段文章被错误吞噬。
*   **双导航同源提取策略 (Dual-Navigation Sync)**：
    *   **严禁硬编码**：绝对禁止手动写死顶部的胶囊导航或左侧的目录索引。
    *   **同源渲染**：必须使用统一的 Vanilla JS 脚本，在页面加载时一次性扫描全文的 `<h2>` 和 `<h3>` 标签。
    *   **职责分化**：左侧 TOC 渲染为树状细分目录（包含 `<h2>` 和 `<h3>`）；顶部胶囊导航作为页面核心脉络视图，仅渲染 `<h2>` 的标题内容。
    *   **状态同步**：滚动监听器 (`scroll`) 必须同时更新这两套导航的 `.active` 状态，确保用户的横向视觉（顶部）和纵向视觉（左侧）永远保持完美一致。

---
**使用说明**：未来让 AI 协助撰写新的芯片模块详细设计说明书时，可直接将此文档内容喂给 AI，并下达指令：“*请按照《Hardware Architecture Dashboard Generation Skill》的标准，为我解析并生成新模块的设计文档。*”


# 交互式标准 HTML 文档模板规范 (Interactive HTML Dashboard Standard Skill)

本文档定义了目前标准化的单页面交互式 HTML 硬件/架构设计文档规范。它结合了极简侧边栏、顶部胶囊导航、以及深度定制的 Mermaid 原生图表等特性，作为后续生成设计文档的**基准 HTML 框架**。

## 1. 核心架构与布局体系 (Core Layout Architecture)
标准文档抛弃了流式中心化布局，采用 **“固定侧边栏 (Sidebar) + 独立滚动主内容区 (Main Content) + 内容卡片 (Container)”** 的经典 Dashboard 布局。
*   **侧边栏 (Sidebar)**：宽度固定 (300px)，带有轻微的右侧阴影，专门用于展示全量分级目录 (H2 & H3)。
*   **主内容区与卡片 (Main Content & Container)**：主区负责滚动 (`overflow-y: auto`)，内部放置一个最大宽度 `1000px` 的卡片式 `div.container` 以集中视线，避免宽屏下文字过长影响阅读。
*   **顶部导航 (Top Nav)**：在卡片顶部提供横向的胶囊状快速导航（仅提取核心的 H2 章节），用于快速在宏观模块间跳跃。

## 2. 色彩系统与视觉规范 (Color System & Visuals)
使用 CSS 原生变量定义全局主题色，确保高度统一与极客观感：
```css
:root {
    --primary: #4a148c;   /* 核心主色调（深紫/深蓝均可，此处以深紫为例） */
    --secondary: #7b1fa2; /* 次级色调 */
    --accent: #ff6f00;    /* 强调色（橙色等） */
    --bg: #f3e5f5;        /* 页面底色/部分高亮底色 */
    --card-bg: #ffffff;   /* 内容卡片纯白底色 */
}
```

## 3. 定制化警告与信息块 (Custom Callout Blocks)
技术文档中必须通过醒目的区块来区分普通描述与“核心痛点/架构突破点”。要求使用特定的 CSS 类：
*   **`.highlight` (核心知识/痛点提示)**：浅色背景（如 `#e3f2fd`），配合左侧加粗边框，用于解释某个协议或算法的基础知识。
*   **`.success` (我们的解决方案/突破口)**：淡绿色背景（如 `#e8f5e9`），配合深绿边框，用于强势展示 C-Model 或硬件是如何解决上述痛点的。

## 4. 图表与图注 (Diagrams & Captions)
*   **简单图表 (Mermaid)**：采用 ES6 模块导入 Mermaid 库，仅用于简单的线性流程图和状态机：
    ```html
    <script type="module">
      import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
      mermaid.initialize({ startOnLoad: true, theme: 'default' });
    </script>
    ```
*   **复杂图表 (Python + Graphviz)**：使用 `<img src="assets/diagram_X.svg">` 嵌入预生成的 SVG，样式为 `max-width: 100%; height: auto;`
*   **图注规范**：所有图表下方**必须**紧跟 `.diagram-caption` 说明，交代该图表的核心意图。

## 5. 自动目录与双导航联动 (Auto TOC & Dual Navigation)
这是标准模板的核心逻辑灵魂，**绝不手动编写目录或链接**，全依赖 Vanilla JS 自动扫描与状态同步：
1.  **扫描目标**：`document.querySelectorAll('.container h2:not(.toc-title), .container h3')`。
2.  **Sidebar 生成**：将所有 H2 设为 `level-2`，H3 设为 `level-3`（带缩进），插入左侧列表。
3.  **Top Nav 生成**：仅提取 H2 的内容，渲染为带有 `.top-nav-link` 样式的横向胶囊按钮。
4.  **联动监听 (Scroll Tracking)**：监听 `main-content` 的滚动事件，计算当前处于哪个标题管辖范围内，并**同时更新**左侧 Sidebar 和顶部 Top Nav 对应项的 `.active` 高亮状态。

---

## 附录：标准的 HTML Boilerplate 代码

以下是符合该标准的极致纯净版 HTML 框架骨架。在新建任何硬件架构总结时，请直接 Copy 此代码并在 `<div class="container">` 内部补充内容：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>硬件架构设计文档标准模板</title>
    <script type="module">
      import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
      mermaid.initialize({ startOnLoad: true, theme: 'default' });
    </script>
    <style>
        :root {
            --primary: #0288d1;
            --secondary: #0277bd;
            --accent: #ff6f00;
            --bg: #f8fafc;
            --card-bg: #ffffff;
        }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; line-height: 1.6; color: #333; margin: 0; padding: 0; display: flex; height: 100vh; overflow: hidden; background: var(--bg); }
        .sidebar { width: 300px; background: white; border-right: 1px solid #e2e8f0; overflow-y: auto; padding: 30px; flex-shrink: 0; box-shadow: 2px 0 10px rgba(0,0,0,0.02); }
        .sidebar h2.toc-title { font-size: 14px; color: #94a3b8; text-transform: uppercase; margin-bottom: 20px; letter-spacing: 1px; margin-top: 0; }
        .sidebar ul { list-style: none; padding: 0; margin: 0; }
        .sidebar li { margin-bottom: 8px; }
        .sidebar a { color: #475569; text-decoration: none; display: block; padding: 8px 12px; border-radius: 6px; font-size: 14px; transition: all 0.2s; }
        .sidebar a:hover { background: #f1f5f9; color: var(--primary); }
        .sidebar a.active { background: #eff6ff; color: var(--primary); font-weight: 600; border-left: 3px solid var(--primary); }
        .sidebar li.level-2 a { padding-left: 12px; }
        .sidebar li.level-3 a { padding-left: 30px; font-size: 13px; color: #64748b; }
        
        .main-content { flex-grow: 1; overflow-y: auto; padding: 40px; scroll-behavior: smooth; }
        .container { max-width: 1000px; margin: 0 auto; background: var(--card-bg); padding: 40px 60px; border-radius: 12px; box-shadow: 0 10px 30px rgba(0,0,0,0.08); }

        h1 { color: var(--primary); text-align: center; border-bottom: 3px solid var(--secondary); padding-bottom: 15px; margin-bottom: 40px; }
        h2 { color: var(--secondary); margin-top: 40px; border-left: 5px solid var(--secondary); padding-left: 15px; }
        h3 { color: #555; }
        
        .highlight { background-color: #e3f2fd; border-left: 4px solid var(--secondary); padding: 15px; margin: 20px 0; border-radius: 0 4px 4px 0; }
        .success { background-color: #e8f5e9; border-left: 4px solid #2e7d32; padding: 15px; margin: 20px 0; border-radius: 0 4px 4px 0; }
        
        .mermaid { display: flex; justify-content: center; margin: 30px 0; background: #fafafa; padding: 20px; border: 1px solid #eee; border-radius: 8px; }
        .diagram-caption { text-align: center; color: #7f8c8d; font-size: 0.9em; margin-top: -15px; margin-bottom: 30px; font-style: italic; }
        
        .top-nav { display: flex; flex-wrap: wrap; gap: 10px; margin-bottom: 25px; padding-bottom: 15px; border-bottom: 1px solid #e0e0e0; }
        .top-nav-link { background: #f8fafc; color: var(--primary); padding: 8px 16px; border-radius: 20px; font-size: 13px; font-weight: 600; text-decoration: none; border: 1px solid #e2e8f0; transition: all 0.2s; }
        .top-nav-link:hover { background: #e0f2fe; }
        .top-nav-link.active { background: var(--primary); color: white; border-color: var(--primary); }
    </style>
</head>
<body>

<div class="sidebar">
    <h2 class="toc-title">目录导航</h2>
    <ul id="toc-list"></ul>
</div>
<div class="main-content" id="main-content">
<div class="container">
    <div class="top-nav" id="top-nav"></div>
    <h1>模块与架构机制总结<br><span style="font-size:0.6em; color:#7f8c8d;">Architecture & Bottlenecks</span></h1>

    <!-- TODO: Write Content Here -->
    <h2>1. 核心架构痛点解析</h2>
    <p>在此处填入硬件瓶颈或者算法介绍内容...</p>
    
    <div class="highlight">
        <strong>💡 概念提示：</strong><br>
        这是需要引起读者注意的基础概念知识点。
    </div>

    <div class="success">
        <strong>我们的解决方案：</strong><br>
        这是我们应对上述痛点的独创解决思路。
    </div>

</div>
</div>

<script>
    document.addEventListener('DOMContentLoaded', () => {
        const tocList = document.getElementById('toc-list');
        const topNav = document.getElementById('top-nav');
        const headers = document.querySelectorAll('.container h2:not(.toc-title), .container h3');
        const mainContent = document.getElementById('main-content');
        
        headers.forEach((header, index) => {
            const id = 'section-' + index;
            header.id = id;
            
            const li = document.createElement('li');
            li.className = header.tagName === 'H2' ? 'level-2' : 'level-3';
            const a = document.createElement('a');
            a.href = '#' + id;
            a.textContent = header.textContent;
            
            a.addEventListener('click', (e) => {
                e.preventDefault();
                document.getElementById(id).scrollIntoView({ behavior: 'smooth' });
                // Fallback for container scroll alignment
                mainContent.scrollTo({ top: document.getElementById(id).offsetTop - 40, behavior: 'smooth' });
            });
            
            li.appendChild(a);
            tocList.appendChild(li);

            if (header.tagName === 'H2' && topNav) {
                const topA = document.createElement('a');
                topA.href = '#' + id;
                topA.className = 'top-nav-link';
                topA.textContent = header.textContent;
                topA.addEventListener('click', (e) => {
                    e.preventDefault();
                    mainContent.scrollTo({ top: document.getElementById(id).offsetTop - 40, behavior: 'smooth' });
                });
                topNav.appendChild(topA);
            }
        });
        
        mainContent.addEventListener('scroll', () => {
            let currentId = '';
            let currentH2Id = '';
            headers.forEach(header => {
                if (mainContent.scrollTop >= header.offsetTop - 100) {
                    currentId = header.id;
                    if (header.tagName === 'H2') currentH2Id = header.id;
                }
            });
            
            if (currentId) {
                document.querySelectorAll('.sidebar a').forEach(link => {
                    link.classList.remove('active');
                    if (link.getAttribute('href') === '#' + currentId) link.classList.add('active');
                });
            }
            if (currentH2Id) {
                document.querySelectorAll('.top-nav-link').forEach(link => {
                    link.classList.remove('active');
                    if (link.getAttribute('href') === '#' + currentH2Id) link.classList.add('active');
                });
            } else if (currentId) {
                let foundH2 = '';
                for (let i = 0; i < headers.length; i++) {
                    if (headers[i].id === currentId) {
                        for (let j = i; j >= 0; j--) {
                            if (headers[j].tagName === 'H2') { foundH2 = headers[j].id; break; }
                        }
                        break;
                    }
                }
                if (foundH2) {
                    document.querySelectorAll('.top-nav-link').forEach(link => {
                        link.classList.remove('active');
                        if (link.getAttribute('href') === '#' + foundH2) link.classList.add('active');
                    });
                }
            }
        });
        
        if(tocList.firstChild) tocList.firstChild.querySelector('a').classList.add('active');
        if(topNav && topNav.firstChild) topNav.firstChild.classList.add('active');
    });
</script>
</body>
</html>
```


# 硬件架构图表自适应排版技能指南 (Hardware Diagram Responsive Design Skill)

在编写现代化的硬件架构详细设计文档（尤其是基于 Web/HTML 的在线文档）时，硬件模块的流水线图、层级架构图以及巨型矩阵往往非常庞大。为了达到**“尽可能放大图表以保证极佳的可读性，同时绝对不产生横向滚动条（无需拖动）”**的最佳用户体验，总结出以下黄金排版定律与画图技能：

## 1. 原生 SVG 图表：绝对的相对宽度 (The 100% Rule)

**问题场景**：硬件时序图、数据流图或流水线图（如 Z 字形 MTT Flow）通常包含大量细密的文字和逻辑块。如果在 SVG 内部写死 `min-width` 或者在外部容器限制宽度，在小屏幕上就会出现滚动条，在大屏幕上又可能显得太小。

**Skill 解决方案**：
* **彻底解除宽度死锁**：在 SVG 根节点上，**永远不要**写死 `width="1400px"` 或 `min-width: 1400px`。
* **使用 ViewBox + 100% 宽度**：
  ```xml
  <!-- 正确的自适应 SVG 头部 -->
  <svg width="100%" height="auto" viewBox="0 0 1100 460" xmlns="...">
  ```
  通过定义画布的物理长宽比 (`viewBox`)，并将其渲染宽度设置为 `100%`。无论容器被压缩到多小，SVG 都会完美按比例缩放，永远不会越界产生滚动条。

## 2. Python + Graphviz 生成的架构图：自适应嵌入

对于复杂的硬件流水线图和多分支架构图，使用 Python + Graphviz 生成 SVG 后，通过 `<img>` 标签嵌入 HTML：

```html
<div style="text-align: center;">
    <img src="assets/diagram_X.svg" style="max-width: 100%; height: auto;">
</div>
```

**核心要点**：
* 使用 `max-width: 100%` 让 SVG 自适应缩放，绝不产生滚动条
* 通过压缩 Graphviz 节点间距和缩短标签来提升视觉字号，而不是盲目增大 fontsize
* 详细的 Graphviz 排版参数和避坑指南见本文档末尾的《Python + Graphviz 架构图生成技能指南》
**问题场景**：硬件算法（如 VVC）中常有 $32 \times 32$ 乃至 $64 \times 64$ 的巨大参数矩阵（如 DCT 系数矩阵）。如果按常规表格渲染，32 列数据必定会撑爆任何标准网页，出现冗长的横向滚动条。

**Skill 解决方案**：
对于需要完整呈现、不能折叠的超大表格，采用**基于规模的动态尺寸计算**，核心策略是“牺牲边缘留白，死保全局可视”。
* **动态 CSS 配置（以 Python 生成器为例）**：
  ```python
  if N >= 32:
      padding = "2px 2px"  # 极限压缩单元格内边距
      font_size = "10px"   # 将字号缩小至刚好可读的物理极限
      cell_width = "18px"  # 强制固定物理宽度
  elif N == 16:
      padding = "3px 5px"
      font_size = "12px"
      cell_width = "22px"
  else:
      padding = "4px 8px"
      font_size = "13px"
      cell_width = "25px"
  ```
* **强制表格布局锁 (table-layout: fixed)**：
  在包裹 `<table>` 的样式中加入 `table-layout: fixed;`。这会剥夺浏览器自作主张撑宽表格的权力，强制其遵循你设定的紧凑尺寸，从而让 $32 \times 32$ 的巨大矩阵能精准地卡在常规网页的可用宽度内（如 800px），实现“免拖动”全局预览。

## 总结
**“大字号、满屏宽、零拖拽”**的终极图表艺术，核心在于打破一切绝对尺寸的枷锁，全面拥抱**相对比例（ViewBox / 100%）**、**最优的空间排列方向（LR 替代 TD）**，以及**针对数据密度的降级策略（动态表格尺寸）**。以后所有硬件设计文档的图表，都应以这三个原则为绝对基准。


# Python + Graphviz 架构图生成技能指南 (Python + Graphviz Diagram Generation Skill)

本技能指南总结了在硬件架构文档中，使用 **Python + Graphviz** 替代 Mermaid 生成高质量架构图的完整方法论与实战避坑经验。

---

## 1. 核心战略决策：为什么弃用 Mermaid

在经过大量实战迭代后，我们**正式废弃 Mermaid 作为复杂硬件架构图的渲染引擎**。

### Mermaid 的致命缺陷（实战踩坑总结）

| 缺陷 | 具体表现 |
|------|---------|
| **DAG 布局引擎失控** | Mermaid 的 `dagre`/`elk` 在横向 (`LR`) 布局中经常产生交叉连线和节点重叠（"穿模现象"） |
| **响应式缩放对抗** | Mermaid 默认注入 `max-width: 100%`，会将 4000px 宽的图强制压缩至屏幕宽度，导致 30px 字号在屏幕上看起来只有 8px |
| **hack 补丁层层叠加** | 为了对抗缩放，需要 `useMaxWidth: false` + CSS `min-width` + `!important` 等一系列脆弱的 hack，且互相冲突 |
| **Flexbox 容器陷阱** | 将 Mermaid SVG 放在 `display: flex` 容器中时，`flex-shrink: 1` 会无视所有 CSS 覆写，强行将图压缩 |
| **字号与间距不可兼得** | 大字号（≥28px）需要 `nodeSpacing ≥ 40`，但拉大间距又会导致图表物理宽度暴增，触发缩放对抗循环 |

### Python + Graphviz 的绝对优势

- **精确的底层渲染控制**：Graphviz 的 `dot` 引擎直接计算节点坐标，不存在前端 CSS 对抗问题
- **原生 SVG 输出**：生成的 SVG 是纯粹的矢量图，浏览器用 `max-width: 100%` 即可完美自适应
- **字号与布局解耦**：字号在 Graphviz 内部设置，浏览器缩放时等比例保持，不会被 CSS 覆写
- **分支路由清晰**：`splines='spline'` 使用贝塞尔曲线自动分离重叠的连线

> **Mermaid 保留场景**：仅限于极简单的线性序列图或状态机（节点数 ≤ 5、无分支）。所有复杂流水线、多分支环路图，一律使用 Python + Graphviz。

---

## 2. 标准工作流 (Standard Workflow)

```
scripts/generate_diagram_X.py  →  python3 运行  →  assets/diagram_X.svg  →  <img> 嵌入 HTML
```

### 2.1 目录结构规范

```
project/
├── index.html
├── scripts/
│   └── generate_diagram_8.py
└── assets/
    └── diagram_8.svg
```

### 2.2 HTML 嵌入规范

```html
<div style="margin: 30px 0; background: #fafafa; padding: 20px;
            border: 1px solid #eee; border-radius: 8px; text-align: center;">
    <img src="assets/diagram_X.svg" alt="图表描述"
         style="max-width: 100%; height: auto; display: inline-block;">
</div>
<div class="diagram-caption">图 X：图表标题</div>
```

**关键 CSS 规则：**
- ✅ `max-width: 100%` — 让 SVG 自适应缩放，永远不超出屏幕
- ✅ `height: auto` — 保持宽高比
- ❌ 绝对不要用 `display: flex` 包裹 — Flexbox 的 `flex-shrink` 会强制压缩图片
- ❌ 绝对不要用 `min-width: 100%; max-width: none` — 会产生横向滚动条
- ❌ 绝对不要用 `overflow-x: auto` — 禁止横向滚动条出现

---

## 3. Graphviz 排版黄金法则 (Layout Golden Rules)

### 3.1 布局方向选择

| 场景 | 推荐方向 | 理由 |
|------|---------|------|
| 线性流水线（≤8 节点） | `rankdir='LR'` | 横向排布，充分利用宽屏 |
| 超长流水线（>8 节点） | `rankdir='LR'` + 缩短标签 | 压缩宽度比折叠更清晰 |
| 多层级架构树 | `rankdir='TB'` | 纵向层级更直观 |

> **核心原则：优先使用 LR 横向布局。** 当横向过宽时，**缩短节点标签**（用缩写如 `T/Q`、`IQ/IT`、`Recon`）而不是折叠路径。折叠会导致连线交叉、阅读路径混乱。

> **进阶实战：漏斗形剪枝流水线的 LR 布局重构**
> 当表达“16进2”这种淘汰架构时，初学者习惯用 `rankdir='TB'` 自上而下画，导致流水线变成了垂直堆叠的“文字墙”。最佳实践是强行使用 `rankdir='LR'` 并配合特定几何形状（如 Graphviz 的 `shape=cds` 标签形节点作排序器），以此从视觉上强势凸显“横向漏斗过滤”和“精英晋级”的数据流向体感，这比垂直推叠的方框直观十倍。

### 3.2 字号与间距的最佳平衡

在 `max-width: 100%` 自适应缩放的前提下，**字体在屏幕上的视觉大小 = Graphviz 字号 ÷ SVG 总宽度 × 屏幕宽度**。因此：

- **要让字体看起来更大，核心策略是压缩 SVG 总宽度**，而不是无限增大字号
- 节点标签用缩写、减少 margin/nodesep/ranksep，都能有效缩窄 SVG

**推荐参数组合（LR 横向流水线）：**

```python
# 全局布局
dot.attr(rankdir='LR', splines='spline', nodesep='0.4', ranksep='0.5', pad='0.2')

# 子图标题
fontsize='22'

# 节点字号（核心！）
fontsize='20', margin='0.15,0.08'

# 边标签字号
fontsize='16'
```

### 3.3 节点标签缩写原则

| 原始标签 | 缩写标签 | 技巧 |
|---------|---------|------|
| `1. IPD 生成预测块` | `1. IPD\n预测块` | 用换行替代长横排 |
| `2. 变换与量化 (T/Q)` | `2. T/Q` | 用业界通用缩写 |
| `4. 反量化与反变换 (IQ/IT)` | `4. IQ/IT` | 同上 |
| `5. 生成重建块 (Recon)` | `5. Recon\n重建块` | 英文缩写 + 中文注释 |
| `6. 计算失真 D (SAD/SSE)` | `6. 失真D\n(SAD/SSE)` | 去掉"计算"等动词 |

### 3.4 连线样式规范

```python
# 主脊椎：实线 + 高权重（强制对齐）
c.edge('IPD', 'Sub', weight='10')

# 数据输入：虚线（表示外部数据流入）
c.edge('Orig', 'Sub', style='dashed')

# 旁路分支：虚线（表示副产物/反馈）
c.edge('TQ', 'Rate', style='dashed')
```

---

## 4. 架构图结构设计原则 (Architecture Design Principles)

### 4.1 输入数据独立成簇

所有外部输入必须用 `cluster` 子图隔离，与主处理环路在视觉上明确分离：

```python
with c.subgraph(name='cluster_Inputs') as ci:
    ci.attr(label='输入', style='dashed, rounded', color='#999999')
    ci.node('Mode', '候选模式')
    ci.node('Orig', '原始像素')
```

### 4.2 主脊椎必须笔直

使用 `weight='10'` 强制主环路中的所有边保持对齐，形成一条清晰可读的主干线：

```python
c.edge('IPD', 'Sub', weight='10')
c.edge('Sub', 'TQ', weight='10')
# ... 一路到终点
```

### 4.3 分支通路清晰可见

- 每条分支使用 `dashed` / `dotted` 线型与主脊椎区分
- 使用 `splines='spline'`（贝塞尔曲线），引擎会自动将重叠的平行线弹开
- **绝对禁止** 使用 `splines='ortho'`（正交直角线）——多条平行线共用通道时必然重叠

### 4.4 色彩编码系统

```python
# 预测模块 — 冷色（蓝）
fillcolor='#e1f5fe', color='#0288d1'

# 变换/量化模块 — 暖色（橙）
fillcolor='#fff3e0', color='#f57c00'

# 重建模块 — 生命色（绿）
fillcolor='#e8f5e9', color='#2e7d32'

# 决策节点 — 警示色（粉红）
fillcolor='#fce4ec', color='#c2185b'

# 运算节点（相加/相减） — 中性色（紫）
fillcolor='#ede7f6', color='#7e57c2'
```

---

## 5. 实战避坑血泪总结 (Pitfalls & Lessons Learned)

### 陷阱一：Flexbox 容器偷偷压缩 SVG

- **症状**：Python 脚本生成了 3500px 宽的大图，HTML 中也设了 `max-width: none`，但图还是被压扁
- **根因**：外层 `div` 使用了 `display: flex`，Flexbox 子元素默认 `flex-shrink: 1`
- **对策**：绝对不要用 `display: flex` 包裹 `<img>`。用 `text-align: center` 实现居中

### 陷阱二：字号设到 30px 但屏幕上依然很小

- **症状**：Graphviz fontsize 设成了 30，但网页上字体还是很小
- **根因**：fontsize=30 使得 SVG 物理宽度暴增，被 `max-width: 100%` 等比缩放后字体也等比缩小
- **对策**：**缩短节点标签、压缩间距来降低 SVG 总宽度**，而不是盲目加大字号

### 陷阱三：正交连线 (ortho) 导致箭头严重交叠

- **症状**：多条从同一节点出发的连线完全重叠成一条线
- **根因**：`splines='ortho'` 强制走直角，平行线无法分离
- **对策**：改用 `splines='spline'`（贝塞尔曲线），引擎自动用弧度将重叠线分开

### 陷阱四：折叠路径导致阅读混乱

- **症状**：为了压缩宽度把主环路折成多行，结果连线到处飞
- **根因**：Graphviz 的 `rank='same'` 无法精确控制行内排序和跨行连线走向
- **对策**：**不要折叠。** 保持单条直线主脊椎，通过缩短标签来压缩宽度

---

## 6. 标准参考模板 (Reference Template)

```python
import graphviz
import os

script_dir = os.path.dirname(os.path.abspath(__file__))
os.chdir(script_dir)

dot = graphviz.Digraph(comment='图表名称', format='svg', engine='dot')
dot.attr(rankdir='LR', splines='spline',
         nodesep='0.4', ranksep='0.5', pad='0.2',
         fontname='Helvetica, Arial, sans-serif')

with dot.subgraph(name='cluster_Main') as c:
    c.attr(label='主环路标题', style='filled, rounded',
           fillcolor='#ffffe0', color='#fbc02d', penwidth='2',
           fontname='Helvetica, Arial, sans-serif', fontsize='22', margin='12')
    
    c.attr('node', shape='box', style='filled, rounded',
           fontname='Helvetica, Arial, sans-serif', fontsize='20', margin='0.15,0.08')
    c.attr('edge', fontname='Helvetica, Arial, sans-serif', fontsize='16', penwidth='1.8')
    
    # 输入簇独立
    with c.subgraph(name='cluster_Inputs') as ci:
        ci.attr(label='输入', style='dashed, rounded', color='#999999', fontsize='16')
        ci.node('In1', '输入1')
        ci.node('In2', '输入2')
    
    # 主脊椎节点 — 用缩写标签
    c.node('A', '1. 步骤A', fillcolor='#e1f5fe', color='#0288d1')
    c.node('B', '2. 步骤B', fillcolor='#fff3e0', color='#f57c00')
    c.node('C', '3. 步骤C', fillcolor='#e8f5e9', color='#2e7d32')
    
    # 主脊椎边 — weight=10 强制对齐
    c.edge('A', 'B', weight='10')
    c.edge('B', 'C', weight='10')
    
    # 输入边 — 虚线
    c.edge('In1', 'A')
    c.edge('In2', 'B', style='dashed')

output_path = os.path.join(os.path.dirname(script_dir), 'assets', 'diagram_name')
dot.render(output_path, cleanup=True)
```

---

**使用说明**：在为硬件架构文档绘制任何复杂图表时，直接遵循本指南的工作流和参数规范，可在一次迭代内产出"大字号、无滚动条、分支清晰"的工业级架构图。

