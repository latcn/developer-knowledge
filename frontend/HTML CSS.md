# 🎨 HTML/CSS 完整指南：从本质到工程化实践

> **适用读者**：具备后端开发经验，希望系统掌握前端渲染原理与工程化 CSS 架构的开发者。

这份指南汇总了此前所有对话的核心知识，并按照 **"本质原理 → 概念详解 → 设计哲学 → 最佳实践 → 完整源码"** 的脉络重新组织。你可以把它当作一份**可运行的、带详细注释的参考手册**。

---

## 第一部分：HTML/CSS 的本质（后端视角）

### 1.1 渲染管线的硬件协作

| 阶段 | 执行硬件 | 产物 | 类比 |
| :--- | :--- | :--- | :--- |
| **DOM 树构建** | CPU | 内存中的节点树 | 解析 JSON 生成对象模型 |
| **样式计算（Style）** | CPU | 每个节点的最终样式 | 配置合并（覆盖/继承） |
| **布局（Layout/Reflow）** | CPU | 每个节点的几何信息（x, y, w, h） | 内存分配/地址计算 |
| **绘制（Paint）** | CPU → 内存 | 位图（Bitmap） | 序列化为二进制数据 |
| **上传纹理** | CPU → GPU 总线 | 显存中的纹理（Texture） | DMA 拷贝 |
| **合成（Composite）** | GPU | 屏幕像素 | 矩阵乘法 + 光栅化 |

### 1.2 核心性能定律

> **"CPU 负责画图（Paint）和算布局（Layout），GPU 负责贴图（Composite）。"**

- **改变 `left/top/width/height`** → 触发 **Layout → Paint → Composite**（CPU 重活）
- **改变 `transform/opacity`** → 仅触发 **Composite**（GPU 轻活）

### 1.3 为什么会有"回流（Reflow）"和"重绘（Repaint）"？

- **回流（Reflow）** = **Layout 阶段**。当元素的几何尺寸（宽高、边距、位置）变化时，浏览器必须重新计算整个受影响子树的所有节点的位置。这是 **CPU 最昂贵的操作**（相当于数据库中修改了表结构，需要重建索引）。
- **重绘（Repaint）** = **Paint 阶段**。当元素的颜色、背景、阴影等不影响几何的属性变化时，浏览器只需重新绘制该节点的位图，不触发布局。**代价比回流低，但仍消耗 CPU**。
- **合成（Composite）** = 仅 GPU 操作，**完全不碰 CPU**。

---

## 第二部分：CSS 盒模型（Box Model）—— 一切布局的基石

### 2.1 盒模型的"洋葱结构"

从外到内：**Margin（外边距） → Border（边框） → Padding（内边距） → Content（内容区）**

```text
┌─────────────────────────────────────────────┐
│                 Margin（透明）               │
│  ┌─────────────────────────────────────────┐│
│  │              Border                     ││
│  │  ┌─────────────────────────────────┐   ││
│  │  │          Padding                │   ││
│  │  │  ┌─────────────────────────┐   │   ││
│  │  │  │       Content           │   │   ││
│  │  │  │                         │   │   ││
│  │  │  └─────────────────────────┘   │   ││
│  │  └─────────────────────────────────┘   ││
│  └─────────────────────────────────────────┘│
└─────────────────────────────────────────────┘
```

### 2.2 `box-sizing`：两种计算模式的本质

| 模式 | CSS 设置 | `width: 200px; padding: 20px; border: 2px;` 的实际渲染宽度 | 记忆口诀 |
| :--- | :--- | :--- | :--- |
| **标准盒模型（W3C）** | `box-sizing: content-box` | **200 + 20×2 + 2×2 = 244px**<br>（padding 和 border 向外撑大） | "设置即内容" |
| **怪异盒模型（IE）** | `box-sizing: border-box` | **200px**（padding 和 border 向内挤压内容区） | "设置即总宽" |

**为什么 `border-box` 是前端铁律？**
- 当你说 `width: 50%` 时，你希望它**正好占父容器的一半**，而不是"一半 + 内边距 + 边框"。
- 类比后端：这就好比你在配置线程池大小，`coreSize=10` 就应该正好 10 个线程，而不是 10 + 一些额外开销。

**最佳实践（DSH 源码风格）**：
```css
*,
*::before,
*::after {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}
```
所有元素统一为 `border-box`，彻底消灭尺寸计算的心智负担。

---

## 第三部分：Flexbox（一维布局模型）

### 3.1 主轴（Main Axis）与交叉轴（Cross Axis）

Flexbox 的本质是 **"在一条轴上分配剩余空间"**。

```text
         主轴方向（flex-direction: row → 从左到右）
         ════════════════════════════════════════
         ┌───────┐  ┌───────┐  ┌───────┐
         │ Item1 │  │ Item2 │  │ Item3 │  ← 元素沿主轴排列
         └───────┘  └───────┘  └───────┘
                        ↑
                  交叉轴方向（从上到下）
```

| 属性 | 作用于 | 效果 | 后端类比 |
| :--- | :--- | :--- | :--- |
| `justify-content` | **主轴** | 主轴上的对齐和间距分配 | 负载均衡算法（Round-Robin/Weighted） |
| `align-items` | **交叉轴** | 交叉轴上的对齐（顶部/居中/拉伸） | 垂直对齐规格 |
| `flex-grow` | **主轴** | 抢剩余空间的比例 | 弹性线程池抢占空闲 CPU |
| `flex-shrink` | **主轴** | 空间不足时的收缩比例 | 优先级队列中的降级策略 |
| `flex-basis` | **主轴** | 主轴方向的基准尺寸 | 初始分配配额 |

### 3.2 `flex: 1` 的完整展开

```css
flex: 1;  /* 等价于 */
flex-grow: 1;
flex-shrink: 1;
flex-basis: 0%;
```

- **`flex-grow: 1`**：有剩余空间时，全部抢过来。
- **`flex-shrink: 1`**：空间不足时，允许收缩。
- **`flex-basis: 0%`**：从 0 开始计算，完全由 `flex-grow` 决定最终大小。

### 3.3 Flex 子项的 "内容保护锁"（核心陷阱）

**问题**：Flex 子项的 `min-width` 默认值是 `auto`，这意味着**子项不会被压缩到比内容更窄**。

```css
/* 默认行为（隐含的 min-width: auto） */
.flex-child {
    flex: 1;
    /* 隐含：min-width: auto; 内容多宽，子项至少多宽 */
}

/* 解除锁定（DSH 铁律） */
.flex-child {
    flex: 1;
    min-width: 0;    /* 允许缩到 0 */
    overflow: hidden; /* 裁掉溢出的内容 */
}
```

**为什么这样设计？**
浏览器默认 `min-width: auto` 是为了**保护内容的可读性**。如果你把一个包含长单词的按钮压缩到 10px，用户看不清内容。但有时候（比如侧栏折叠），我们确实需要让元素缩得比内容更窄——此时必须显式 `min-width: 0`。

**DSH 源码印证**（文档 2.3）：
```css
.sidebarCol, .centerCol, .detailsCol {
    min-width: 0;
    overflow: hidden;
}
```

---

## 第四部分：Grid（二维布局模型）

### 4.1 显式网格 vs 隐式网格

- **显式网格**：你用 `grid-template-columns` 和 `grid-template-rows` 定义的轨道。
- **隐式网格**：Grid 项目超出显式定义行数时，自动生成的额外行（由 `grid-auto-rows` 控制）。

### 4.2 核心单位：`fr`（Fraction）

`fr` 是 **"剩余空间的分数"**。

```css
grid-template-columns: 200px 1fr 2fr;
/* 
  第一列固定 200px
  剩余空间分为 3 份
  第二列占 1 份，第三列占 2 份
*/
```

### 4.3 `minmax()` 函数（设定轨道尺寸范围）

```css
grid-template-columns: 1fr minmax(640px, 1fr) 1fr;
/* 中间列最小 640px，且可以弹性扩张 */
```

### 4.4 Grid 的过渡动画（折叠侧栏的标准做法）

**铁律**：只有 `grid-template-columns` 支持平滑过渡，`display: none` 做不到。

```css
.app-frame {
    display: grid;
    grid-template-columns: 240px 1fr 320px;
    transition: grid-template-columns 0.25s ease-in-out;
}

/* 折叠侧栏：列宽从 240px 变为 64px，触发平滑动画 */
.app-frame.collapsed {
    grid-template-columns: 64px 1fr 320px;
}
```

### 4.5 DSH 的 `computeColumns` 函数逻辑（JS 计算列宽）

当 CSS 的简单 `fr` 无法满足复杂约束时，DSH 用 JS 计算后内联写入 `grid-template-columns`：

```javascript
export function computeColumns(viewport, sidebar, details) {
    // 侧栏最小 200px，最大 400px
    const s = clampWidth(sidebar, SIDEBAR_MIN, SIDEBAR_MAX);
    // 主区最小 640px（死保对话流可读性）
    const CENTER_MIN = 640;

    // 先尝试三列等比例分配
    if (s + details + CENTER_MIN <= viewport) {
        return { sidebar: s, center: viewport - s - details, details };
    }
    // 空间不足：先压缩详情栏
    const d1 = Math.max(DETAILS_MIN, viewport - s - CENTER_MIN);
    // 再不行：牺牲详情栏（宽度归零）
    return { sidebar: s, center: Math.max(0, viewport - s), details: 0 };
}
```

---

## 第五部分：CSS 变量与主题系统（Token）

### 5.1 什么是 Token？

Token 是**带语义的 CSS 变量名**，相当于前端的"配置中心"。

### 5.2 DSH 的三层 Token 架构

| 层级 | 命名规则 | 示例 | 后端类比 |
| :--- | :--- | :--- | :--- |
| **Static（色板）** | `--dsw-static-<色系>-<亮度>` | `--dsw-static-neutral-bluish-950: rgb(21,21,23);` | 硬件资源（CPU/内存规格） |
| **Alias（语义）** | `--dsw-alias-<用途>-<状态>` | `--dsw-alias-bg-base: var(--dsw-static-neutral-bluish-00);` | 配置项（`cache.redis.host`） |
| **Specific（组件专用）** | `--dsw-specific-<组件>-<属性>` | 组件内覆盖别名 | 局部覆盖（`@Qualifier`） |

### 5.3 一键深色模式的原理

```css
/* 浅色模式（默认） */
body {
    --dsw-alias-bg-base: var(--dsw-static-white);
    --dsw-alias-text-primary: var(--dsw-static-gray-900);
}

/* 深色模式：只需覆盖变量定义 */
body[data-ds-dark-theme] {
    --dsw-alias-bg-base: var(--dsw-static-gray-950);
    --dsw-alias-text-primary: var(--dsw-static-white);
}
```
所有组件都用 `var(--dsw-alias-bg-base)` 引用背景色，深色模式自动生效，**不需要修改任何组件代码**。

---

## 第六部分：滚动条与溢出治理

### 6.1 滚动条挤压内容的解决方案

**问题**：滚动条出现时，视口宽度减少，页面内容重新布局（Reflow），导致布局抖动。

**DSH 铁律（文档 2.5）**：
```css
body {
    scrollbar-gutter: stable; /* 永远预留滚动条位置 */
}

.scroll-container {
    overflow-y: auto;
    overflow-x: hidden;
    scrollbar-gutter: stable;
}
```

### 6.2 自定义滚动条样式

```css
/* WebKit 内核（Chrome/Safari/Edge） */
::-webkit-scrollbar {
    width: 6px;
    height: 6px;
}
::-webkit-scrollbar-thumb {
    background: var(--dsh-scrollbar-thumb);
    border-radius: 4px;
}
::-webkit-scrollbar-track {
    background: transparent;
}

/* Firefox */
@supports not selector(::-webkit-scrollbar) {
    body {
        scrollbar-width: thin;
        scrollbar-color: var(--dsh-scrollbar-thumb) transparent;
    }
}
```

---

## 第七部分：性能优化必知必会

### 7.1 `transform` vs `left/top`（GPU vs CPU）

| 写法 | 触发阶段 | 硬件 |
| :--- | :--- | :--- |
| `left: 100px;` | **Layout → Paint → Composite** | CPU 全重 |
| `transform: translateX(100px);` | **仅 Composite** | GPU 矩阵乘法 |

**为什么 `transform` 快？**
- CPU 只在第一帧"画一次图"（Paint），上传到显存。
- 之后每一帧，GPU 直接对纹理做矩阵变换。
- 类比：CPU 负责把一张照片扫描进电脑，GPU 负责在屏幕上随意拖拽、旋转它。

### 7.2 `will-change` 的正确用法

```css
/* ✅ 正确：只在确实需要动画的元素上开启 */
.animated-menu {
    will-change: transform, opacity;
}

/* ❌ 错误：对大量静态元素开启，爆显存 */
.list-item {
    will-change: transform; /* 100 个列表项 = 100 个独立合成层 */
}
```

### 7.3 rAF（requestAnimationFrame）批处理

**原理**：将一帧内（约 16ms）的多次 DOM 变更合并为 1 次渲染。

```javascript
function rafBatch() {
    let scheduled = false;
    const pending = [];

    return function flush(callback) {
        pending.push(callback);
        if (!scheduled) {
            scheduled = true;
            requestAnimationFrame(() => {
                const snapshot = [...pending];
                pending.length = 0;
                scheduled = false;
                snapshot.forEach(fn => fn());
            });
        }
    };
}
```

---

## 第八部分：完整可运行源码（综合示例）

这份 HTML 文件包含了上述所有核心概念，并以可视化区块展示，你可以直接保存运行，逐个验证。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>HTML/CSS 完整指南 - 综合示例</title>
    <style>
        /* ================================================================
           1. 全局 Reset + box-sizing 铁律
           ================================================================ */
        *,
        *::before,
        *::after {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }

        /* ================================================================
           2. 全局滚动条治理（scrollbar-gutter: stable）
           ================================================================ */
        body {
            font-family: system-ui, -apple-system, 'PingFang SC', 'Microsoft YaHei', sans-serif;
            background: #f0f2f5;
            color: #1a1a1a;
            padding: 24px;
            scrollbar-gutter: stable;
        }

        /* ================================================================
           3. 自定义滚动条（WebKit + Firefox 双路径）
           ================================================================ */
        ::-webkit-scrollbar {
            width: 6px;
            height: 6px;
        }
        ::-webkit-scrollbar-thumb {
            background: #c1c7cd;
            border-radius: 4px;
        }
        ::-webkit-scrollbar-track {
            background: transparent;
        }

        @supports not selector(::-webkit-scrollbar) {
            body {
                scrollbar-width: thin;
                scrollbar-color: #c1c7cd transparent;
            }
        }

        /* ================================================================
           4. 页面容器
           ================================================================ */
        .container {
            max-width: 1200px;
            margin: 0 auto;
            display: flex;
            flex-direction: column;
            gap: 24px;
        }

        /* ================================================================
           5. 卡片组件（通用）
           ================================================================ */
        .card {
            background: #ffffff;
            border-radius: 12px;
            padding: 24px;
            border: 1px solid #e9ecf0;
            box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
        }
        .card-title {
            font-size: 18px;
            font-weight: 600;
            margin-bottom: 16px;
            display: flex;
            align-items: center;
            gap: 10px;
            color: #1a1a1a;
        }
        .card-title .badge {
            font-size: 12px;
            font-weight: 400;
            background: #eef2f6;
            padding: 2px 12px;
            border-radius: 20px;
            color: #4a5a6a;
        }
        .card-desc {
            font-size: 14px;
            color: #5a6a7a;
            margin-bottom: 16px;
            line-height: 1.6;
        }

        /* ================================================================
           6. Flex 布局示例：导航栏
           ================================================================ */
        .nav-demo {
            display: flex;
            align-items: center;
            gap: 12px;
            background: #2c3e50;
            padding: 12px 20px;
            border-radius: 8px;
            color: white;
        }
        .nav-demo .brand {
            font-weight: 700;
            font-size: 18px;
            flex-shrink: 0;
        }
        .nav-demo .spacer {
            flex: 1;
        }
        .nav-demo .nav-item {
            padding: 6px 14px;
            border-radius: 6px;
            background: rgba(255, 255, 255, 0.08);
            cursor: default;
            transition: background 0.15s;
            flex-shrink: 0;
        }
        .nav-demo .nav-item:hover {
            background: rgba(255, 255, 255, 0.18);
        }
        .nav-demo .nav-item.active {
            background: #3498db;
        }

        /* ================================================================
           7. Flex 子项 min-width:0 陷阱演示
           ================================================================ */
        .flex-trap-demo {
            display: flex;
            gap: 12px;
            background: #f8f9fa;
            border-radius: 8px;
            padding: 16px;
            border: 2px dashed #d0d7de;
            width: 100%;
        }
        .flex-trap-demo .item {
            flex: 1;
            background: #3498db;
            color: white;
            padding: 12px;
            border-radius: 6px;
            text-align: center;
            /* 关键：默认 min-width: auto，长单词会撑破 */
            /* 解除锁定：放开下面两行注释 */
            /* min-width: 0; */
            /* overflow: hidden; */
        }
        .flex-trap-demo .item .long-word {
            /* 模拟超长内容 */
            display: inline-block;
            white-space: nowrap;
            background: rgba(255, 255, 255, 0.2);
            padding: 2px 6px;
            border-radius: 4px;
            font-size: 13px;
        }
        .flex-trap-demo .item:nth-child(2) {
            background: #2ecc71;
        }
        .flex-trap-demo .item:nth-child(3) {
            background: #e67e22;
        }

        /* ================================================================
           8. Grid 三栏布局（模拟 DSH AppFrame）
           ================================================================ */
        .grid-frame {
            display: grid;
            grid-template-columns: 200px 1fr 260px;
            grid-template-rows: 300px;
            gap: 12px;
            transition: grid-template-columns 0.25s cubic-bezier(0.4, 0, 0.2, 1);
            border-radius: 8px;
            overflow: hidden;
        }
        /* 【铁律】Grid 所有子项必须 min-width:0 + overflow:hidden */
        .grid-frame>* {
            min-width: 0;
            overflow: hidden;
        }

        .grid-sidebar {
            background: #2c3e50;
            color: #ecf0f1;
            padding: 16px;
            display: flex;
            flex-direction: column;
            gap: 8px;
        }
        .grid-sidebar .title {
            font-weight: 600;
            font-size: 14px;
            border-bottom: 1px solid rgba(255, 255, 255, 0.1);
            padding-bottom: 8px;
        }
        .grid-sidebar .item {
            padding: 6px 12px;
            border-radius: 4px;
            background: rgba(255, 255, 255, 0.05);
            font-size: 13px;
            cursor: default;
        }
        .grid-sidebar .item.active {
            background: #3498db;
        }

        .grid-center {
            background: #ffffff;
            padding: 16px;
            display: flex;
            flex-direction: column;
            border: 1px solid #e9ecf0;
            border-radius: 8px;
        }
        .grid-center .header {
            font-weight: 600;
            border-bottom: 1px solid #e9ecf0;
            padding-bottom: 8px;
            flex-shrink: 0;
        }
        .grid-center .body {
            flex: 1;
            overflow-y: auto;
            padding: 12px 0;
            font-size: 14px;
            color: #3a4a5a;
            scrollbar-gutter: stable;
            min-height: 0;
        }
        .grid-center .body .msg {
            padding: 8px 12px;
            background: #f1f3f5;
            border-radius: 6px;
            margin-bottom: 8px;
        }
        .grid-center .body .msg.self {
            background: #3498db;
            color: white;
            text-align: right;
        }
        .grid-center .footer {
            flex-shrink: 0;
            display: flex;
            gap: 8px;
            padding-top: 8px;
            border-top: 1px solid #e9ecf0;
        }
        .grid-center .footer input {
            flex: 1;
            padding: 6px 12px;
            border: 1px solid #d0d7de;
            border-radius: 6px;
            outline: none;
            min-width: 0;
        }
        .grid-center .footer input:focus {
            border-color: #3498db;
        }
        .grid-center .footer button {
            padding: 6px 18px;
            background: #3498db;
            color: white;
            border: none;
            border-radius: 6px;
            cursor: pointer;
            flex-shrink: 0;
        }

        .grid-details {
            background: #f8f9fa;
            padding: 16px;
            border: 1px solid #e9ecf0;
            border-radius: 8px;
            font-size: 13px;
            color: #3a4a5a;
            overflow-y: auto;
        }
        .grid-details .label {
            font-weight: 600;
            color: #1a1a1a;
        }

        /* Grid 折叠控制 */
        .grid-frame.collapsed-details {
            grid-template-columns: 200px 1fr 0px;
        }
        .grid-frame.collapsed-sidebar {
            grid-template-columns: 56px 1fr 260px;
        }
        .grid-frame.collapsed-sidebar .grid-sidebar .title span,
        .grid-frame.collapsed-sidebar .grid-sidebar .item span {
            display: none;
        }
        .grid-frame.collapsed-sidebar .grid-sidebar .item {
            text-align: center;
            padding: 6px 0;
        }

        /* ================================================================
           9. transform vs left 性能对比演示
           ================================================================ */
        .perf-demo {
            display: flex;
            gap: 40px;
            padding: 16px 0;
            align-items: center;
            flex-wrap: wrap;
        }
        .perf-box {
            width: 120px;
            height: 80px;
            border-radius: 8px;
            display: flex;
            align-items: center;
            justify-content: center;
            color: white;
            font-weight: 600;
            font-size: 13px;
            transition: 1s ease-in-out;
            flex-shrink: 0;
        }
        .perf-box.bad {
            background: #e74c3c;
            position: relative;
            left: 0;
        }
        .perf-box.good {
            background: #2ecc71;
            transform: translateX(0);
        }
        .perf-box.moved-bad {
            left: 200px;
        }
        .perf-box.moved-good {
            transform: translateX(200px);
        }
        .perf-label {
            font-size: 13px;
            color: #5a6a7a;
        }

        /* ================================================================
           10. CSS 变量 Token 演示（主题切换）
           ================================================================ */
        /* 定义静态色板 */
        :root {
            --static-white: #ffffff;
            --static-gray-100: #f0f2f5;
            --static-gray-800: #2c3e50;
            --static-gray-900: #1a1a1a;
            --static-blue-500: #3498db;
        }

        /* 语义 Token（浅色主题） */
        body {
            --alias-bg-card: var(--static-white);
            --alias-bg-page: var(--static-gray-100);
            --alias-text-primary: var(--static-gray-900);
            --alias-text-secondary: #5a6a7a;
            --alias-border: #e9ecf0;
            --alias-brand: var(--static-blue-500);
        }

        /* 深色主题（通过 data 属性切换） */
        body[data-theme="dark"] {
            --alias-bg-card: #1e1e2a;
            --alias-bg-page: #0d1117;
            --alias-text-primary: #e6edf3;
            --alias-text-secondary: #8b949e;
            --alias-border: #30363d;
            --alias-brand: #58a6ff;
        }

        /* 应用 Token */
        body {
            background: var(--alias-bg-page);
            color: var(--alias-text-primary);
            transition: background 0.3s, color 0.3s;
        }
        .card {
            background: var(--alias-bg-card);
            border-color: var(--alias-border);
        }
        .card-title {
            color: var(--alias-text-primary);
        }
        .card-desc {
            color: var(--alias-text-secondary);
        }

        /* ================================================================
           11. 响应式断点（@media）
           ================================================================ */
        @media (max-width: 900px) {
            .grid-frame {
                grid-template-columns: 160px 1fr 0px;
            }
            .grid-details {
                display: none;
            }
        }
        @media (max-width: 600px) {
            .grid-frame {
                grid-template-columns: 0px 1fr 0px;
            }
            .grid-sidebar {
                display: none;
            }
            .perf-demo {
                flex-direction: column;
                gap: 20px;
            }
            body {
                padding: 12px;
            }
        }

        /* ================================================================
           12. 工具类
           ================================================================ */
        .flex-row {
            display: flex;
            gap: 12px;
            flex-wrap: wrap;
            align-items: center;
        }
        .mt-2 {
            margin-top: 12px;
        }
        .text-sm {
            font-size: 13px;
            color: var(--alias-text-secondary);
        }
        .gap-2 {
            gap: 12px;
        }
        .btn {
            padding: 6px 16px;
            border: none;
            border-radius: 6px;
            background: var(--alias-brand);
            color: white;
            cursor: pointer;
            font-weight: 500;
            transition: opacity 0.15s;
        }
        .btn:hover {
            opacity: 0.85;
        }
        .btn.secondary {
            background: var(--alias-border);
            color: var(--alias-text-primary);
        }
    </style>
</head>
<body>
    <div class="container">

        <!-- ============================================================
        标题区
        ============================================================ -->
        <div class="card">
            <div class="card-title">
                🎨 HTML/CSS 完整指南
                <span class="badge">综合示例 · 可运行</span>
            </div>
            <div class="card-desc">
                这份示例包含了盒模型、Flex/Grid 布局、CSS 变量 Token、transform 性能优化、
                滚动治理、响应式断点等全部核心概念。点击下方按钮验证原理。
            </div>
            <div class="flex-row">
                <button class="btn" onclick="toggleTheme()">🌓 切换深色/浅色主题</button>
                <button class="btn secondary" onclick="toggleDetails()">📋 折叠/展开详情栏</button>
                <button class="btn secondary" onclick="toggleSidebar()">📂 折叠/展开侧栏</button>
                <button class="btn secondary" onclick="togglePerf()">🏃 运行性能对比</button>
                <button class="btn secondary" onclick="toggleFlexTrap()">🔓 切换 Flex 陷阱修复</button>
            </div>
        </div>

        <!-- ============================================================
        1. 盒模型可视化
        ============================================================ -->
        <div class="card">
            <div class="card-title">📦 盒模型（Box Model） <span class="badge">border-box 铁律</span></div>
            <div class="card-desc">
                所有元素统一 <code>box-sizing: border-box</code>，宽高 = content + padding + border。
                下方盒子的总宽度固定为 200px，padding 和 border 向内挤压内容区。
            </div>
            <div style="display:flex;gap:24px;flex-wrap:wrap;align-items:center;">
                <div style="background:#3498db;color:white;width:200px;padding:16px;border:4px solid #2c3e50;border-radius:8px;text-align:center;">
                    总宽 200px<br>
                    <span style="font-size:12px;">padding:16px + border:4px</span>
                </div>
                <div style="font-size:13px;color:var(--alias-text-secondary);">
                    <code>width: 200px; padding: 16px; border: 4px;</code><br>
                    内容区宽度 = 200 - 16×2 - 4×2 = <strong>160px</strong>
                </div>
            </div>
        </div>

        <!-- ============================================================
        2. Flex 导航栏 + min-width:0 陷阱
        ============================================================ -->
        <div class="card">
            <div class="card-title">🧭 Flex 布局 <span class="badge">主轴 / 交叉轴 / min-width:0</span></div>
            <div class="card-desc">
                上方是典型的 Flex 导航栏：<code>justify-content</code> 控制主轴分布，
                <code>align-items</code> 控制交叉轴居中。下方演示 <strong>min-width:auto 陷阱</strong>。
            </div>

            <!-- Flex 导航栏 -->
            <div class="nav-demo" style="margin-bottom:16px;">
                <span class="brand">📋 DSH</span>
                <span class="spacer"></span>
                <span class="nav-item active">工作区</span>
                <span class="nav-item">会话</span>
                <span class="nav-item">设置</span>
                <span class="nav-item" style="background:#e74c3c;">退出</span>
            </div>

            <!-- min-width:0 陷阱 -->
            <div style="font-size:13px;color:var(--alias-text-secondary);margin-bottom:8px;">
                <strong>⚠️ Flex 子项默认 <code>min-width: auto</code></strong>：长内容会撑破容器。
                <span id="flexTrapStatus" style="color:#e74c3c;">❌ 当前：溢出（未修复）</span>
            </div>
            <div class="flex-trap-demo" id="flexTrapDemo">
                <div class="item" id="flexItem1">
                    <span class="long-word">supercalifragilisticexpialidocious</span>
                </div>
                <div class="item">正常文本 B</div>
                <div class="item">正常文本 C</div>
            </div>
            <div style="font-size:12px;color:var(--alias-text-secondary);margin-top:8px;">
                💡 点击上方 "切换 Flex 陷阱修复" 按钮，观察 <code>min-width:0 + overflow:hidden</code> 的效果。
            </div>
        </div>

        <!-- ============================================================
        3. Grid 三栏布局（DSH 风格）
        ============================================================ -->
        <div class="card">
            <div class="card-title">📐 Grid 三栏布局 <span class="badge">DSH AppFrame 风格</span></div>
            <div class="card-desc">
                使用 <code>grid-template-columns</code> 控制三栏宽度，<code>transition</code> 实现平滑折叠。
                所有列强制 <code>min-width:0 + overflow:hidden</code> 防止内容撑破。
            </div>

            <div class="grid-frame" id="gridFrame">
                <!-- 侧栏 -->
                <div class="grid-sidebar">
                    <div class="title">📂 <span>会话列表</span></div>
                    <div class="item active"><span>设计系统讨论</span></div>
                    <div class="item"><span>API 网关重构</span></div>
                    <div class="item"><span>性能优化方案</span></div>
                    <div class="item"><span>UI 插槽注册</span></div>
                    <div class="item"><span>滚动契约实现</span></div>
                    <div style="flex:1;min-height:0;"></div>
                    <div class="item" style="background:rgba(255,255,255,0.05);">
                        <span>👤 后端工程师</span>
                    </div>
                </div>

                <!-- 主区 -->
                <div class="grid-center">
                    <div class="header">💬 对话流 · 主工作区</div>
                    <div class="body" id="chatBody">
                        <div class="msg">你好！我是 DSH 助手。</div>
                        <div class="msg self">请解释 CSS Grid 的 fr 单位。</div>
                        <div class="msg"><code>fr</code> 是剩余空间的分数单位。</div>
                        <div class="msg self">明白了，谢谢！</div>
                    </div>
                    <div class="footer">
                        <input type="text" placeholder="输入消息..." id="chatInput" />
                        <button onclick="addChatMessage()">发送</button>
                    </div>
                </div>

                <!-- 详情栏 -->
                <div class="grid-details" id="detailsPanel">
                    <div class="label">📌 当前上下文</div>
                    <div style="margin-top:8px;line-height:1.6;">
                        此处展示选中节点的元数据、<br>
                        工具调用详情或思考链。<br>
                        <span style="color:var(--alias-brand);">slot: sidebar.workspaces</span>
                    </div>
                    <hr style="margin:12px 0;border-color:var(--alias-border);">
                    <div class="label">🔧 插件信息</div>
                    <div style="margin-top:4px;font-size:12px;">
                        ds-depth: v1.2.0<br>
                        dsh-core: v3.1.0
                    </div>
                </div>
            </div>
        </div>

        <!-- ============================================================
        4. transform vs left 性能对比
        ============================================================ -->
        <div class="card">
            <div class="card-title">⚡ transform vs left <span class="badge">GPU vs CPU</span></div>
            <div class="card-desc">
                <strong style="color:#e74c3c;">红色（改 left）</strong> 触发 Layout → Paint → Composite（CPU 重活）。
                <strong style="color:#2ecc71;">绿色（改 transform）</strong> 仅触发 Composite（GPU 轻活）。
                点击下方按钮移动它们，观察流畅度差异。
            </div>
            <div class="perf-demo">
                <div>
                    <div class="perf-box bad" id="badBox">改 left</div>
                    <div class="perf-label">❌ CPU 重</div>
                </div>
                <div>
                    <div class="perf-box good" id="goodBox">改 transform</div>
                    <div class="perf-label">✅ GPU 轻</div>
                </div>
            </div>
            <div class="flex-row">
                <button class="btn" onclick="movePerfBoxes()">🏃 移动（往返）</button>
                <button class="btn secondary" onclick="resetPerfBoxes()">↩️ 复位</button>
                <span class="text-sm" id="perfStatus">就绪</span>
            </div>
            <div style="font-size:12px;color:var(--alias-text-secondary);margin-top:8px;">
                💡 打开 Chrome DevTools → Performance 面板，录制动画过程，
                观察 <code>Layout</code> 和 <code>Paint</code> 的耗时差异。
            </div>
        </div>

        <!-- ============================================================
        5. CSS 变量 Token 展示
        ============================================================ -->
        <div class="card">
            <div class="card-title">🎨 CSS 变量 Token <span class="badge">主题系统</span></div>
            <div class="card-desc">
                所有颜色从语义 Token 中取：<code>--alias-bg-card</code>、<code>--alias-text-primary</code> 等。
                切换主题只需修改 <code>body[data-theme]</code> 上的变量覆盖。
            </div>
            <div style="display:grid;grid-template-columns:repeat(auto-fit,minmax(120px,1fr));gap:12px;">
                <div style="background:var(--alias-bg-card);border:1px solid var(--alias-border);padding:12px;border-radius:6px;">
                    <span style="color:var(--alias-text-primary);">主文字</span>
                </div>
                <div style="background:var(--alias-bg-card);border:1px solid var(--alias-border);padding:12px;border-radius:6px;">
                    <span style="color:var(--alias-text-secondary);">次要文字</span>
                </div>
                <div style="background:var(--alias-brand);padding:12px;border-radius:6px;color:white;text-align:center;">
                    品牌色
                </div>
                <div style="background:var(--alias-bg-page);padding:12px;border-radius:6px;border:1px solid var(--alias-border);">
                    页面背景
                </div>
            </div>
            <div style="margin-top:12px;font-size:12px;color:var(--alias-text-secondary);">
                当前主题：<span id="themeLabel">浅色</span>
            </div>
        </div>

        <!-- ============================================================
        6. 滚动治理 + scrollbar-gutter
        ============================================================ -->
        <div class="card">
            <div class="card-title">📜 滚动治理 <span class="badge">scrollbar-gutter: stable</span></div>
            <div class="card-desc">
                下方容器开启 <code>scrollbar-gutter: stable</code>，滚动条永远预留位置，
                内容宽度不会因滚动条出现/消失而抖动（CLS 优化）。
            </div>
            <div style="display:grid;grid-template-columns:1fr 1fr;gap:16px;">
                <div style="background:var(--alias-bg-card);border:1px solid var(--alias-border);border-radius:8px;padding:12px;height:120px;overflow-y:auto;scrollbar-gutter:stable;">
                    <div style="padding:4px 0;">📌 第 1 行</div>
                    <div style="padding:4px 0;">📌 第 2 行</div>
                    <div style="padding:4px 0;">📌 第 3 行</div>
                    <div style="padding:4px 0;">📌 第 4 行</div>
                    <div style="padding:4px 0;">📌 第 5 行</div>
                    <div style="padding:4px 0;">📌 第 6 行</div>
                    <div style="padding:4px 0;">📌 第 7 行</div>
                    <div style="padding:4px 0;">📌 第 8 行</div>
                </div>
                <div style="background:var(--alias-bg-card);border:1px solid var(--alias-border);border-radius:8px;padding:12px;height:120px;overflow-y:auto;scrollbar-gutter:stable;">
                    <div style="padding:4px 0;">✅ 滚动条出现时</div>
                    <div style="padding:4px 0;">✅ 内容宽度不变</div>
                    <div style="padding:4px 0;">✅ 无布局抖动</div>
                    <div style="padding:4px 0;">📐 这就是 DSH 的</div>
                    <div style="padding:4px 0;">📐 滚动治理铁律</div>
                </div>
            </div>
        </div>

    </div>

    <!-- ============================================================
    JavaScript 交互脚本
    ============================================================ -->
    <script>
        // ================================================================
        // 1. 主题切换（Token 演示）
        // ================================================================
        let isDark = false;

        function toggleTheme() {
            isDark = !isDark;
            document.body.setAttribute('data-theme', isDark ? 'dark' : '');
            document.getElementById('themeLabel').textContent = isDark ? '深色' : '浅色';
        }

        // ================================================================
        // 2. Grid 折叠控制
        // ================================================================
        function toggleDetails() {
            const frame = document.getElementById('gridFrame');
            frame.classList.toggle('collapsed-details');
        }

        function toggleSidebar() {
            const frame = document.getElementById('gridFrame');
            frame.classList.toggle('collapsed-sidebar');
        }

        // ================================================================
        // 3. Flex min-width:0 陷阱切换
        // ================================================================
        let flexTrapFixed = false;

        function toggleFlexTrap() {
            flexTrapFixed = !flexTrapFixed;
            const items = document.querySelectorAll('.flex-trap-demo .item');
            const status = document.getElementById('flexTrapStatus');

            if (flexTrapFixed) {
                items.forEach(el => {
                    el.style.minWidth = '0';
                    el.style.overflow = 'hidden';
                });
                status.textContent = '✅ 已修复：min-width:0 + overflow:hidden';
                status.style.color = '#2ecc71';
            } else {
                items.forEach(el => {
                    el.style.minWidth = '';
                    el.style.overflow = '';
                });
                status.textContent = '❌ 当前：溢出（未修复）';
                status.style.color = '#e74c3c';
            }
        }

        // ================================================================
        // 4. transform vs left 性能对比
        // ================================================================
        let perfMoved = false;

        function movePerfBoxes() {
            const bad = document.getElementById('badBox');
            const good = document.getElementById('goodBox');
            perfMoved = !perfMoved;

            if (perfMoved) {
                // 改 left：触发 Layout → Paint → Composite
                bad.classList.add('moved-bad');
                // 改 transform：仅 Composite
                good.classList.add('moved-good');
                document.getElementById('perfStatus').textContent = '⏳ 动画进行中...';
            } else {
                bad.classList.remove('moved-bad');
                good.classList.remove('moved-good');
                document.getElementById('perfStatus').textContent = '↩️ 已复位';
            }
            // 1 秒后更新状态
            setTimeout(() => {
                if (perfMoved) {
                    document.getElementById('perfStatus').textContent = '✅ 动画完成（对比流畅度）';
                }
            }, 1200);
        }

        function resetPerfBoxes() {
            const bad = document.getElementById('badBox');
            const good = document.getElementById('goodBox');
            bad.classList.remove('moved-bad');
            good.classList.remove('moved-good');
            perfMoved = false;
            document.getElementById('perfStatus').textContent = '↩️ 已复位';
        }

        // ================================================================
        // 5. 聊天消息（Grid 主区交互）
        // ================================================================
        function addChatMessage() {
            const input = document.getElementById('chatInput');
            const body = document.getElementById('chatBody');
            const text = input.value.trim();
            if (!text) return;

            const msg = document.createElement('div');
            msg.className = 'msg self';
            msg.textContent = text;
            body.appendChild(msg);
            input.value = '';

            // 滚动到底部
            body.scrollTop = body.scrollHeight;

            // 模拟回复
            setTimeout(() => {
                const reply = document.createElement('div');
                reply.className = 'msg';
                reply.textContent = '✅ 已收到：' + text.slice(0, 20) + (text.length > 20 ? '...' : '');
                body.appendChild(reply);
                body.scrollTop = body.scrollHeight;
            }, 600);
        }

        // Enter 发送（含 IME 守卫）
        document.getElementById('chatInput').addEventListener('keydown', function(e) {
            if (e.key === 'Enter' && !e.isComposing) {
                e.preventDefault();
                addChatMessage();
            }
        });

        // ================================================================
        // 6. 初始化状态
        // ================================================================
        console.log('✅ HTML/CSS 完整指南已加载！');
        console.log('📌 观测点：1. 盒模型 border-box  2. Flex/Grid 布局  3. transform vs left');
        console.log('📌 4. CSS 变量 Token 主题  5. scrollbar-gutter: stable  6. @media 响应式');
    </script>
</body>
</html>
```

---

## 第九部分：知识点速查表

| 主题 | 核心概念 | DSH 源码印证 |
| :--- | :--- | :--- |
| **盒模型** | `box-sizing: border-box` 铁律，padding 向内挤压 | 全局 Reset |
| **Flexbox** | 主轴/交叉轴，`min-width:0` 解除内容保护锁 | `.sidebarCol { min-width:0 }`（文档 2.3） |
| **Grid** | `grid-template-columns` 过渡动画，`fr` 单位 | `AppFrame.module.css`（文档 2.3） |
| **CSS 变量** | 三层 Token（Static→Alias→Specific），深色模式 | `design-platform.css`（文档 2.2） |
| **滚动治理** | `scrollbar-gutter: stable` 防布局抖动 | `scrollbar.css`（文档 2.5） |
| **性能优化** | `transform` 走 GPU，`will-change` 克制使用 | `ChatView.tsx` useLayoutEffect（文档 1.2） |
| **响应式** | `@media` 断点，容器查询 | `columns.ts` 自动折叠（文档 2.7） |
| **GPU 加速** | 合成层（Layer）管理，显存占用控制 | 虚拟化列表（文档 1.6） |

---

这份指南涵盖了从**盒模型原理**到**GPU 合成机制**的全部知识点。保存为 `.html` 文件运行，你可以亲手验证每一个概念。这就是 DSH 源码背后完整的 CSS 工程化体系。
