# Hardware Dynamic State Simulation & Math Rendering Skill

本文档总结了在极客风硬件/算法 Dashboard 中，如何利用原生前端技术（HTML/JS/CSS）以及数学渲染引擎（MathJax），构建**底层状态机动态推演微件（Interactive State Machine Widget）**的核心技术手段。在开发类似 CABAC Ctx 计算、状态机跳转等深度底层原理的文档时，应严格遵循此方案。

## 1. 核心设计理念

* **摒弃静态“贴代码”**：底层代码（尤其是 C-Model）往往枯燥难懂。绝对不能单纯地贴出一段 C++ 代码了事。必须将代码拆解为“动作”、“状态”、“数据流”，并通过可视化组件映射出来。
* **双模互动推演 (Dual-Mode Simulation)**：
  * **单点手动模式 (Manual)**：允许工程师像 debug 一样，手动点击控制输入变量（如宏块的空间状态），观察算式、寄存器和输出内存地址的联动反应。
  * **自动播放模式 (Auto-play)**：内置一套基于 `setInterval` 的状态遍历逻辑，能够像幻灯片一样自动巡检边界条件，极大降低读者理解门槛，非常适合汇报演示。
* **“动态白话文”解读**：取消大段的静态代码框，取而代之的是一段**动态的自然语言说明**。当状态发生改变时，说明文本中的数值、结论乃至文字颜色都要通过 JS 实时动态更新。
* **数学公式优雅化 (Math Beauty)**：坚决杜绝使用代码字符或模糊截图拼凑数学公式。底层算法推导必须使用标准 LaTeX 并借助 MathJax 进行网页端原生渲染，做到极高清晰度和极强专业感。

## 2. 关键实现套路 (Implementation Techniques)

### 2.1 CSS Grid 与空间状态映射
对于视频编解码中的空间相邻块（Left, Top PU 等），使用 `CSS Grid` 配合绝对定位构建 2D 空间排布。为每个宏块块赋予独立的 ID，绑定 `click` 事件切换内部状态变量。
```html
<!-- 范例：CSS Grid 空间映射 -->
<div style="display: grid; grid-template-columns: 80px 80px; gap: 10px;">
    <div></div> <!-- 占位 -->
    <div id="btnTop" class="state-block">Top PU</div>
    <div id="btnLeft" class="state-block">Left PU</div>
    <div class="current-block">当前 CU</div>
</div>
```

### 2.2 1D 数组内存指针的动态光标
将 `ContextModel3DBuffer` 等底层扁平化内存数组，具象化为横向排布的 HTML Flexbox。通过 JS 实时计算出的索引值（如 `uiCtx`），动态赋予对应 DOM 元素 `box-shadow` 和全透明度，模拟“探照灯”寻址效果。

### 2.3 动态自然语言解读更新
利用 JS 模板字符串（Template Literals），将计算逻辑转换为带高亮样式的自然语言片段。
```javascript
// 每次点击后重新拼装解释说明
textExp.innerHTML = `
    1. 左侧宏块是 <strong style="color: ${leftSkip ? '#10b981' : '#ef4444'};">${leftSkip ? 'True' : 'False'}</strong>，基础增量 +${leftVal}。<br>
    2. 结论：最终索引 <code>uiCtx = ${uiCtx}</code>。
`;
```

### 2.4 MathJax LaTeX 公式注入
硬件和底层算法中经常包含各种对数（$-log_2(P)$）、指数和定点化计算公式。
必须在 HTML 的 `<head>` 中动态或静态注入 MathJax 库：
```html
<script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
<script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
```
随后，在页面中大量使用 `\\[ ... \\]` 来进行 Block 级的硬核公式渲染，使用 `\\( ... \\)` 进行 Inline 级渲染。确保定点精度（如 15-bit $2^{15}=32768$）在公式下方附带严格注释。
