# RolePlay Hub 角色卡 UI 开发指南

本文档旨在帮助开发者和创作者编写适合 RolePlay Hub 渲染的角色卡自定义 UI 界面（HTML卡片/状态栏/HUD）。

## 1. 简介

RolePlay Hub 支持在对话气泡中渲染复杂的 HTML 内容。这允许创作者设计交互式的状态面板、背包系统、角色状态栏（HUD）等。系统会自动将检测到的 HTML 代码块（或包含 HTML 标签的文本）封装在一个安全的 `<iframe>` 中进行渲染。

## 2. 渲染环境与基本原理

当系统检测到 Markdown 内容中包含 HTML 结构（如 `<div>`, `<table>`, `<style>`, `<script>` 等）时，会执行以下操作：

1.  **沙箱隔离**：创建一个 `<iframe>` 元素，并设置 `sandbox="allow-scripts allow-forms allow-popups allow-modals allow-same-origin"` 属性，确保安全。
2.  **样式重置**：自动注入一套 CSS Reset 规则，消除默认边距，确保内容宽度自适应。
3.  **脚本注入**：注入 jQuery (v3.7.1) 和通信辅助脚本（用于高度自适应和与主窗口通信）。
4.  **高度自适应**：内置脚本会监听内容变化（DOM 变动、图片加载、窗口缩放），自动调整 iframe 高度以适应内容，消除滚动条。

## 3. 快速开始 (HTML 模板)

你可以直接在角色卡的输出中（或通过正则脚本替换）使用以下基础结构。系统会自动为你补全 `<html>`, `<head>`, `<body>` 等标签，但显式声明它们也是支持的。

### 最小化示例

```html
<div style="padding: 10px; background: #f0f9ff; border-radius: 8px; border: 1px solid #bae6fd;">
    <h3 style="margin: 0 0 5px 0; color: #0369a1;">状态面板</h3>
    <p style="margin: 0; font-size: 12px; color: #0c4a6e;">
        HP: <span style="color: #ef4444; font-weight: bold;">85/100</span> | 
        MP: <span style="color: #3b82f6; font-weight: bold;">40/50</span>
    </p>
    <!-- 点击按钮发送消息给 AI -->
    <button onclick="triggerSlash('/use_potion')" style="margin-top: 8px; padding: 4px 8px; border: none; background: #3b82f6; color: white; border-radius: 4px; cursor: pointer;">
        使用药水
    </button>
</div>
```

## 4. 样式与布局

### 默认注入的 CSS Reset

系统会自动在 `<head>` 中注入以下样式，请在设计时考虑到这一点：

```css
html, body {
    margin: 0 !important;
    padding: 0 !important;
    width: 100% !important;
    word-wrap: break-word !important;
    box-sizing: border-box !important;
    overflow: hidden !important; /* 隐藏滚动条，由 JS 控制高度 */
}
::-webkit-scrollbar { display: none; }
*, *::before, *::after { box-sizing: inherit !important; }
img, video, canvas, svg { max-width: 100% !important; height: auto !important; }
table { display: block !important; overflow-x: auto !important; max-width: 100% !important; }
```

### 内置 CSS 类 (Sinan HUD)

为了方便快速开发，系统内置了一套类似 SillyTavern 风格的 CSS 类，你可以直接使用：

-   `.sinan-hud`: HUD 容器，具有磨砂玻璃效果和渐变背景。
-   `.char-card`: 角色卡片容器。
-   `.char-name`: 角色名称样式。
-   `.char-mood`: 角色心情文本样式。
-   `.bar-bg` / `.bar-fill`: 进度条背景和填充样式。

**示例：**

```html
<div class="sinan-hud">
    <div class="char-card c-yufan">
        <div class="char-name">
            羽凡 <span class="char-mood">冷静</span>
        </div>
        <div class="bar-bg">
            <div class="bar-fill" style="width: 80%;"></div>
        </div>
    </div>
</div>
```

## 5. 交互与脚本 (JavaScript)

### 与 AI 对话交互 (`triggerSlash`)

要在 UI 中触发用户发送消息（例如点击按钮让角色执行动作），请使用 `window.triggerSlash(text)` 函数。

```javascript
// 发送文本 "查看背包" 给 AI，就像用户在输入框输入并发送一样
triggerSlash('查看背包');

// 也可以触发 slash commands (取决于后端支持)
triggerSlash('/stats');
```

**注意**：`triggerSlash` 会在父窗口中执行，触发一次新的 AI 生成请求。

### 高度自适应机制

内置脚本会通过以下方式尝试保持 iframe 高度完美适应内容：
1.  **ResizeObserver**: 监听 `document.body` 的尺寸变化。
2.  **Image onload**: 监听所有图片的加载完成事件。
3.  **Click Event**: 监听点击事件，并在此后 600ms 内持续检测高度（为了适配折叠面板/手风琴动画）。
4.  **轮询**: 在加载初期进行几次强制检查。

**最佳实践**：如果你的 UI 包含动态展开/收起的内容（如 `<details>` 或自定义下拉菜单），请确保你的动画是平滑的，内置脚本通常能自动处理。如果是通过 JS 动态插入大量内容，建议在插入后手动触发一次窗口 resize 事件或稍等片刻，虽然 `ResizeObserver` 通常能捕获到。

### 引入外部库

环境默认已通过 CDN 引入了 **jQuery 3.7.1** (`defer` 加载)。你可以直接使用 `$` 或 `jQuery`。

```html
<script>
    $(document).ready(function() {
        $('#my-btn').click(function() {
            $(this).text('已点击');
        });
    });
</script>
```

你也可以在你的 HTML 中引入其他 CDN 库（如 FontAwesome, Bootstrap 等），但请注意加载速度可能会影响显示体验。

## 6. 开发建议

1.  **移动端优先**：RolePlay Hub 在移动端使用广泛。请确保你的 UI 在窄屏（手机）下表现良好。使用百分比宽度或 Flexbox/Grid 布局，避免固定的大宽度像素值。
    -   系统默认视口设置：`<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">`
2.  **避免全局滚动条**：iframe 默认被设置为 `overflow: hidden`。请确保内容高度是“真实”的，不要设置 `body { height: 100vh }` 这种样式，否则会导致高度计算错误，产生无限拉伸或截断。让内容自然撑开高度。
3.  **固定定位慎用**：尽量避免使用 `position: fixed`。因为是在 iframe 内部，fixed 是相对于 iframe 视口的，而 iframe 的高度是动态变化的。如果 iframe 变得很长，fixed 元素可能会出现在用户看不到的地方（或者遮挡内容）。通常推荐使用 `position: absolute` 或 Flex 布局。
4.  **背景透明**：iframe 本身背景默认为白色 (`bg-white`)。如果你希望 UI 融入聊天气泡背景，可以在你的根元素设置背景色，覆盖满整个 iframe。
5.  **图片加载**：尽量为 `<img>` 标签指定 `width` 或 `aspect-ratio`，这有助于防止图片加载过程中的布局抖动（Layout Shift），虽然内置脚本会处理加载后的高度修正。

## 7. 示例：带交互的RPG状态栏

将以下代码放入角色卡的 Prompt、World Info 或正则替换中：

```html
<div style="font-family: sans-serif; background: linear-gradient(135deg, #2c3e50, #4ca1af); color: white; padding: 12px; border-radius: 10px; box-shadow: 0 4px 6px rgba(0,0,0,0.1);">
    <div style="display: flex; align-items: center; margin-bottom: 8px;">
        <div style="width: 40px; height: 40px; background: white; border-radius: 50%; overflow: hidden; margin-right: 10px; border: 2px solid rgba(255,255,255,0.5);">
            <!-- 占位头像，实际使用可用 img -->
            <svg viewBox="0 0 24 24" fill="#333" style="width:100%;height:100%;"><path d="M12 12c2.21 0 4-1.79 4-4s-1.79-4-4-4-4 1.79-4 4 1.79 4 4 4zm0 2c-2.67 0-8 1.34-8 4v2h16v-2c0-2.66-5.33-4-8-4z"/></svg>
        </div>
        <div style="flex: 1;">
            <div style="font-weight: bold; font-size: 1.1em;">勇者 {{user}}</div>
            <div style="font-size: 0.8em; opacity: 0.8;">Lv. 15 见习骑士</div>
        </div>
    </div>
    
    <!-- 状态条 -->
    <div style="margin-bottom: 4px; font-size: 0.8em;">HP</div>
    <div style="background: rgba(0,0,0,0.3); height: 8px; border-radius: 4px; overflow: hidden; margin-bottom: 8px;">
        <div style="background: #ff6b6b; width: 75%; height: 100%;"></div>
    </div>
    
    <div style="margin-bottom: 4px; font-size: 0.8em;">MP</div>
    <div style="background: rgba(0,0,0,0.3); height: 8px; border-radius: 4px; overflow: hidden; margin-bottom: 12px;">
        <div style="background: #4ecdc4; width: 45%; height: 100%;"></div>
    </div>

    <!-- 交互区域 -->
    <div style="display: flex; gap: 8px;">
        <button onclick="triggerSlash('检查状态')" style="flex: 1; padding: 6px; border: none; background: rgba(255,255,255,0.2); color: white; border-radius: 6px; cursor: pointer; font-size: 0.9em; transition: background 0.2s;" onmouseover="this.style.background='rgba(255,255,255,0.3)'" onmouseout="this.style.background='rgba(255,255,255,0.2)'">
            🔍 检查
        </button>
        <button onclick="triggerSlash('使用治疗药水')" style="flex: 1; padding: 6px; border: none; background: rgba(16, 185, 129, 0.6); color: white; border-radius: 6px; cursor: pointer; font-size: 0.9em;">
            💊 治疗
        </button>
    </div>
</div>
```

## 8. 高级功能指南

### 8.1 复杂交互组件

除了简单的按钮点击，您还可以利用 JavaScript 实现更复杂的组件，例如标签页（Tabs）、折叠面板（Accordion）甚至小游戏。

#### 折叠面板示例

```html
<div style="border: 1px solid #e5e7eb; border-radius: 8px; overflow: hidden; background: white;">
    <div onclick="this.nextElementSibling.style.display = this.nextElementSibling.style.display === 'none' ? 'block' : 'none'"
         style="padding: 10px 15px; background: #f9fafb; cursor: pointer; font-weight: bold; display: flex; justify-content: space-between; align-items: center;">
        <span>🎒 背包 (点击展开)</span>
        <span style="font-size: 12px; color: #6b7280;">5/20</span>
    </div>
    <div style="display: none; padding: 15px; border-top: 1px solid #e5e7eb;">
        <ul style="margin: 0; padding-left: 20px;">
            <li>长剑 x1</li>
            <li>治疗药水 x3</li>
            <li>魔法卷轴 x1</li>
        </ul>
        <div style="margin-top: 10px; text-align: right;">
            <button onclick="triggerSlash('整理背包')" style="font-size: 12px; padding: 4px 8px; background: #e5e7eb; border: none; border-radius: 4px; cursor: pointer;">整理</button>
        </div>
    </div>
</div>
```

**注意**：RolePlay Hub 的自适应脚本会自动监测点击事件后的高度变化，因此展开/收起动画通常能平滑过渡。

### 8.2 利用 Prompt 指导 AI 生成 UI

您可以编写特定的 System Prompt 或 Character Prompt，指导 AI 在特定情况下输出包含 UI 代码的回复。

**Prompt 示例：**

```text
[System Note: 当用户的 HP 发生变化，或获得新物品时，请在回复的末尾附加一个 HTML 状态块。
格式如下：
```html
<div class="status-update">
  <div>HP: {{当前HP}}/{{最大HP}}</div>
  <div>Inventory: {{新增物品}}</div>
</div>
```
请确保只更新变化的部分。]
```

**配合正则脚本（Regex Script）：**

为了避免 AI 输出的代码直接显示给用户，您可以创建一个正则脚本，将特定的 AI 输出格式转换为美观的 UI。

1.  **Regex**: `\[STATUS: HP=(\d+)/(\d+) MP=(\d+)/(\d+)\]`
2.  **Replacement**:
    ```html
    <div class="sinan-hud">
      <div class="char-card c-tongqiu">
        <div class="char-name">状态更新</div>
        <div class="bar-bg"><div class="bar-fill" style="width: $1%;"></div></div>
        <div style="font-size: 10px; margin-top: 2px;">HP: $1/$2 | MP: $3/$4</div>
      </div>
    </div>
    ```
3.  **Placement**: AI Output Only

### 8.3 状态管理与持久化

由于每个气泡都是独立的 `<iframe>`，它们之间**不共享状态**。变量无法在气泡之间持久保存。

如果您需要“记忆”某些状态（如好感度、金币），必须依赖 **AI 的上下文记忆** 或 **World Info**。

**工作流建议：**

1.  **显示**：AI 在回复中输出当前状态（如 `[Gold: 100]`）。
2.  **渲染**：通过正则将 `[Gold: 100]` 渲染为漂亮的 UI。
3.  **更新**：用户点击 UI 按钮（如“购买”），触发 `triggerSlash('购买长剑')`。
4.  **循环**：AI 接收指令，处理逻辑，并在新的回复中输出更新后的状态（如 `[Gold: 50]`）。

### 8.4 调试技巧

1.  **浏览器控制台**：您可以在浏览器控制台（F12）中查看 iframe 内的报错。
2.  **独立测试**：先将 HTML 代码保存为本地 `.html` 文件进行测试，确认样式和 JS 逻辑无误后再放入角色卡。
3.  **查看源码**：在 RolePlay Hub 中，您可以使用“编辑消息”功能查看 AI 输出的原始 HTML 代码，检查是否有格式错误。