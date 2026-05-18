---
name: deepseek-vision-qa
description: |
  Use OpenCLI to drive DeepSeek's native vision model (识图模式) in Chrome for image-based QA.
  DeepSeek has built-in multimodal support — no third-party vision service needed.
  Triggers: "DeepSeek 识图", "DeepSeek vision", "用 DeepSeek 看图", "DeepSeek 视觉模式",
  "deepseek识别图片", "deepseek vision QA", "让 DeepSeek 看看这张图".
  This is the DEFAULT choice when the user needs image QA — DeepSeek has native vision support, no third-party service required.
  MANDATORY: Before writing any QA prompt, ask the user what the image is, what it should contain, and what to inspect.
allowed-tools: Bash(opencli:*), Read, Write, Bash(python3:*)
---

# deepseek-vision-qa

用 OpenCLI 驱动 DeepSeek 原生识图模式做视觉 QA。DeepSeek 自带多模态，无需借助第三方服务。

**全程在同一个 `opencli browser <session>` 里完成。**

## Prerequisites

```bash
opencli doctor              # 必须全绿
opencli deepseek status     # 必须 Connected + Login: Yes
```

如果没登录：在 Chrome 打开 `https://chat.deepseek.com/` 登录。

## Core Workflow (5 steps)

所有命令使用统一 session 名 `deepseek-qa`。

### Step 1: Confirm the image

确认图片路径。如需生成测试图：

```bash
python3 -c "
from PIL import Image, ImageDraw
img = Image.new('RGB', (800, 600), '#1a1a2e')
draw = ImageDraw.Draw(img)
draw.rectangle([50, 50, 350, 250], fill='#e94560', outline='white', width=3)
draw.text((100, 300), 'Test content', fill='white')
img.save('/tmp/test-image.png')
"
```

### Step 2: Ask the user about the image (MANDATORY)

**在写任何 prompt 之前，必须先问用户这三个问题：**

1. **这是什么类型的图？** — PPT 幻灯片 / UI 截图 / 海报 / 图表 / 照片 / ...
2. **这张图的预期内容是什么？** — 应该有哪些元素、文字、布局
3. **你想重点检查什么？** — 或者让 AI 根据图片类型建议检查项

如果用户说"不知道""你看着办"，则根据图片类型推断检查项：

| 图片类型 | 默认检查项 |
|---------|-----------|
| PPT 幻灯片 | 文字溢出/截断、元素重叠、间距均匀、对比度、对齐、占位符残留 |
| UI 截图 | 布局错位、文字截断、按钮可点击区域、颜色一致性、响应式问题 |
| 海报/宣传图 | 视觉层次、文字可读性、品牌色一致、关键信息是否突出、留白 |
| 数据图表 | 坐标轴标签、图例完整、数据标签位置、颜色区分度、标题准确 |
| 照片/一般图片 | 构图、清晰度、曝光、主体是否突出 |

**禁止跳过这一步直接写 prompt。**

### Step 3: Refresh page and switch to 识图 mode

**必须先刷新页面**（清除 OpenCLI markerAttr 上下文），然后用 `eval` 切换模式（eval 不注入 markerAttr，不会导致后续 upload 失败）。

```bash
# 刷新页面，清除 JS 上下文
opencli browser deepseek-qa eval "location.reload()"

# 等页面加载完，重新绑定
sleep 3 && opencli browser deepseek-qa bind

# 用 eval 点击识图 radio（第3个 radio，index 2）
opencli browser deepseek-qa eval "document.querySelectorAll('[role=radio]')[2].click()"
```

验证模式：
```bash
opencli browser deepseek-qa state 2>&1 | grep '识图模式'
# 应显示：使用识图模式开始对话
```

> **为什么用 eval 而不是 click？** OpenCLI v1.0.15 有 markerAttr bug：`state`/`find`/`click`/`upload` 都会在页面注入 `markerAttr` 变量，但只有第一个能成功声明，后续调用会报 `SyntaxError: Identifier 'markerAttr' has already been declared`。`eval` 不走 DOM marker 逻辑，所以用 eval 切换模式，让 upload 成为首个 DOM-marker 命令。

### Step 4: Upload the image

**upload 必须是刷新后首个 DOM-marker 命令**，用 CSS selector 定位 file input：

```bash
# 直接上传（不要先调 state/find！否则 markerAttr 已存在，upload 会失败）
opencli browser deepseek-qa upload 'input[type=file]' /path/to/image.png
```

接受格式：png, jpg, jpeg, svg, bmp, gif, webp, avif, tiff 等。

上传成功后关闭可能弹出的搜索框：

```bash
opencli browser deepseek-qa keys Escape
```

### Step 5: Type custom prompt, send, and read

```bash
# 关闭搜索框（upload 后可能弹出）
opencli browser deepseek-qa keys Escape

# 拿 textarea 和发送按钮的 ref
opencli browser deepseek-qa state
```

找到：
- **textarea**：`placeholder=给 DeepSeek 发送消息`
- **发送按钮**：textarea 右侧带 svg 图标的 `role=button`（file input 旁边，通常 ref 编号最大）
- **深度思考**：`<span>深度思考</span>` 旁的 button

```bash
# 输入定制 prompt
opencli browser deepseek-qa type <textarea-ref> "<your-custom-prompt>"

# 点击发送
opencli browser deepseek-qa click <send-btn-ref>

# 等待回复（视图片复杂度 10-30 秒）
sleep 20

# 读取回复
opencli deepseek read -f plain
```

> **可选：开启深度思考** — 在发送前点击"深度思考"按钮，让 DeepSeek 用 DeepThink 模式分析。复杂图像推荐开启。

## Prompt Writing Guide

**不要发泛泛的 prompt**。

### Bad
```
解释图片
看看这图
```

### Good
```
这是一张PPT幻灯片。预期内容：大字标题"君子善假于物"，下方三行小字。
请检查：
1. 文字是否有溢出或截断
2. 元素是否相互重叠
3. 间距是否均匀
4. 对比度是否足够
5. 是否有残留的占位符
6. 对齐是否一致

逐一回答，有问题标❌，没问题标✅。
```

### Template

```
这是一张[图片类型]。
预期内容：[简单描述]。
请检查：
1. [检查项1]
2. [检查项2]
...

逐一回答，有问题具体说明位置和原因。
```

## Pro Tips

- **Step 2 不可跳过**：先问用途再写 prompt。
- **全程用同一个 session**：bind → upload → state → type → click → read 全在 `deepseek-qa` session 里。
- **markerAttr 避坑**：刷新后，用 `eval` 切换模式，`upload` 作为首个 DOM 命令。`state`/`find`/`click` 只能在 `upload` 之后调用。
- **upload 用 CSS selector**：`'input[type=file]'` 比 numeric ref 更可靠。
- **upload 后先关搜索框**：`keys Escape`，否则可能干扰后续操作。
- **upload 后必须重新 state**：上传后 DOM ref 编号会变。
- **发送按钮识别**：上传图片并输入文字后，发送按钮会从灰色变亮。它是个带 svg 的 `role=button`，通常在 file input 的右侧。
- **DeepThink 可选**：复杂图像分析建议开启深度思考，简单检查不用。
- **等待时间**：带图片 + 深度思考的请求可能需要 15-30 秒。

## Troubleshooting

| symptom | fix |
|---------|-----|
| `opencli doctor` 红灯 | `opencli daemon restart && opencli doctor` |
| `deepseek status` 未登录 | 打开 `chat.deepseek.com` 登录 |
| **`SyntaxError: Identifier 'markerAttr' has already been declared`** | OpenCLI v1.0.15 bug。刷新页面 → bind → 用 `eval` 做模式切換 → `upload` 作为首个 DOM 命令。绝不能在 upload 前调 `state`/`find`/`click` |
| file input 找不到 | 用 CSS selector `'input[type=file]'` 直接上传，不要用 numeric ref |
| 上传后弹出"搜索对话内容" | `opencli browser deepseek-qa keys Escape` 关掉 |
| 发送按钮灰色点不了 | 需要同时满足：图片已上传 + textarea 有文字 |
| 回复没识别图片（给了"AA AB AC"这种文字提取） | 没切到识图模式！刷新页面，用 `eval` 点击第3个 `[role=radio]`，再上传 |
| 回复太泛 | prompt 太模糊，加图片类型 + 预期内容 + 检查清单 |

## Full Example Script

```bash
# 0. 环境检查
opencli doctor
opencli deepseek status

# 1. 开新对话 + 绑定
opencli deepseek new
opencli browser deepseek-qa bind

# 2. 刷新页面（清除 markerAttr）+ 用 eval 切换识图模式
opencli browser deepseek-qa eval "location.reload()"
sleep 3 && opencli browser deepseek-qa bind
opencli browser deepseek-qa eval "document.querySelectorAll('[role=radio]')[2].click()"

# 3. 上传图片（首个 DOM-marker 命令，不能先调 state）
opencli browser deepseek-qa upload 'input[type=file]' /path/to/image.png

# 4. 关搜索框 + state 拿 ref
opencli browser deepseek-qa keys Escape
opencli browser deepseek-qa state
# → 找到 textarea ref（如 521）和发送按钮 ref（如 528）

# 5. 输入 prompt + 发送
opencli browser deepseek-qa type 521 "这是一张[图片类型]。预期内容：[...]。请检查：1. ... 2. ... 逐一回答。"
opencli browser deepseek-qa click 528

# 6. 等待并读取
sleep 20
opencli deepseek read -f plain
```

