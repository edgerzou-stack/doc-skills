# Hardware Architecture Dashboard Generation Skill (Native EDA Style)

本文档总结了将传统硬件详细设计文档（Markdown/Word）转化为**“现代化、交互式、极客风 Web Dashboard”**的核心技能（Skill）与最佳实践。在未来的硬件设计文档开发中，应严格遵循本套体系进行构建。

## 1. 核心设计理念 (Core Philosophy)
*   **摒弃静态文档**：不再使用干瘪的 Markdown 或 PDF，而是构建拥有左侧全局导航（Sidebar）和右侧内容区（Content Pane）的单页面 Web 应用（SPA）。
*   **极致的极客美学**：采用类似顶级开发者文档（如 Stripe, Tailwind）的视觉规范。深色顶栏、玻璃拟态（Glassmorphism）吸顶导航、高对比度语法高亮。
*   **原生 EDA 级图表**：不依赖外部低清图片，全量使用原生 SVG、HTML/CSS 渲染的 Gantt 图以及调优后的 Mermaid 图表，确保在 4K 屏幕下依然无限放大不失真。
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

### 3.3 模块层级与状态机 (Mermaid.js)
*   使用 Mermaid 绘制模块架构树，但默认样式极度紧凑。
*   **强制调优**：
    *   在图表开头必须注入配置项：`%%{init: {'theme': 'base', 'flowchart': {'nodeSpacing': 80, 'rankSpacing': 150, 'curve': 'bumpX'}}}%%`。拉大间距并使用平滑贝塞尔曲线。
    *   同样应用 SVG 防压缩规则，设置 `min-width: 1200px !important` 和外层容器的 `justify-content: flex-start`，确保图表庞大且舒展。

### 3.4 Mermaid 陷阱规避与排版进阶策略 (Mermaid Traps & Advanced Layouts)
在通过 Mermaid 绘制复杂硬件流水线时，极易踩中词法解析器崩溃或布局失控的雷区，必须严格遵守以下排版与语法纪律：
*   **致命陷阱：Syntax error in text (特殊字符引发的渲染崩溃)**：
    *   **痛点**：当节点文本或子图 (subgraph) 标题中出现空格、括号 `()`、冒号 `:`、斜杠 `/`、破折号 `-` 或 HTML 换行符 `<br>` 时，会导致 Mermaid (尤其是 10.x 版本) 解析引擎直接崩溃，弹出炸弹图标。
    *   **强制规避**：**必须**为所有带文字的节点和子图标题加上**双引号 `""`** 进行包裹转义。例如：错误写法 `A[文本(包含特殊)]` / `subgraph S1 [带 空格 标题]`，**正确写法** `A["文本(包含特殊)"]` / `subgraph S1 ["带 空格 标题"]`。
*   **排版进阶 1：消除多子图垂直留白与文字遮挡 (Row-by-Row 水平流)**：
    *   **痛点**：当你想把两个步骤（如 Luma 和 Chroma 流水线）分为上下两行，且每行内部节点**横向**展开时，如果直接用线连接 `子图A的节点 -> 子图B的节点`，Mermaid 会强行将内部节点拉扯成一条垂直长线，无视内部的 `direction LR`。此外，如果跨图连线上带有长文本说明（如 `A -.->|"长文本"| B`），极易造成文字与节点方框相互遮挡、重叠。
    *   **解决方案 (外竖内横隔离法 + 规范化连线)**：
        1. 外层使用 `flowchart TB`，每个 `subgraph` 内部使用 `direction LR`。
        2. **最核心一步**：绝不能让连线跨越子图内部的节点！应将跨图的依赖线连在外部大框上（例如 `Row1 -.-> Row2`），彻底切断跨图干涉，完美实现上下双行、内部横向展开。
        3. **防止遮挡**：跨大框的带字连线，**强制要求使用点阵文字语法** `Row1 -. "说明文字" .-> Row2`，绝对禁止使用 `|文字|` 语法，这样能最大程度促使布局引擎分配合理的水平留白，避免元素遮挡。
*   **排版进阶 2：并排对比面板 (Side-by-Side Columns)**：
    *   **需求**：需要将两种流水线架构（如全串行 vs MD并发）作为两个垂直的长面板，**左右并排**放置以缩小垂直纵深。
    *   **解决方案 (外横内竖法)**：外层使用 `flowchart LR` 将两大赛道左右并排铺开，而在每个 `subgraph` 内部强制使用 `direction TD` 让具体步骤从上往下流转。由此得到两个优雅的并肩长列。

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

## 7. 核心突破机制的视觉强调 (Critical Mechanism Highlighting)
*   **痛点与突破口分离**：对于架构中最核心的“作弊/解耦”机制（例如：为了流水线解耦，强行使用**原始像素**替代**重建像素**来进行模式粗选），这种颠覆常规理论的突破点，必须与普通的说明文字拉开视觉差距。
*   **强制使用预警/强提醒样式 (`.critical-alert`)**：
    *   绝不可使用普通的 `<div class="highlight">`（那只是用于补充说明或知识普及）。
    *   必须使用带醒目边框、刺眼底色和特殊字体的 `.critical-alert` 样式（如红色左边框、淡红底色）。
    *   内部的核心关键字词，还要进一步嵌套 `<span style="background: #fecaca; padding: 2px 6px; border-radius: 4px; color: #991b1b; font-weight: 700;">` 这种极其抢眼的底色高亮。
    *   **目的**：让硬件架构师或算法人员在快速扫视文档时，第一眼就能被这个极其重要的架构魔法设计“刺瞎”，确保知识传达零遗漏。

---
**使用说明**：未来让 AI 协助撰写新的芯片模块详细设计说明书时，可直接将此文档内容喂给 AI，并下达指令：“*请按照《Hardware Architecture Dashboard Generation Skill》的标准，为我解析并生成新模块的设计文档。*”
