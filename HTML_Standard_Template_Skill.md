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

## 4. Mermaid 图表与图注 (Mermaid & Captions)
*   **ESM 模块化加载**：统一采用现代化的 ES6 模块导入 Mermaid 库：
    ```html
    <script type="module">
      import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
      mermaid.initialize({ startOnLoad: true, theme: 'default' });
    </script>
    ```
*   **居中排版与图注**：包裹在 `.mermaid` 容器内并居中显示，下方**必须**紧跟一段 `.diagram-caption` 的文字说明，交代该图表的核心意图。

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
