# DeepSeek Image Skill（deepseek-vision-qa）

这个 Skill 用于通过 OpenCLI 驱动 DeepSeek 的原生识图模式，完成图片视觉 QA（版式、文案、UI、图表、照片等检查）。

## 功能概述

- 使用 DeepSeek 原生多模态能力，无需第三方识图服务
- 在同一个浏览器会话中完成上传、提问、发送、读取回复
- 面向视觉 QA，支持按检查清单逐项输出问题

## 前置条件

在开始前请确保：

```bash
opencli doctor
opencli deepseek status
```

要求：
- `opencli doctor` 全绿
- `opencli deepseek status` 显示 `Connected` 且已登录

## 强制规则（必须遵守）

在写任何 prompt 之前，必须先问用户这 3 个问题：

1. 这是什么类型的图？（PPT / UI 截图 / 海报 / 图表 / 照片…）
2. 这张图的预期内容是什么？
3. 你希望重点检查什么？

> 不允许跳过该步骤直接发 prompt。

## 快速使用流程

会话名建议固定为 `deepseek-qa`（与 `SKILL.md` 约定一致），这样在连续执行 bind / upload / state / type / click / read 时不会混淆会话上下文。

1. 新建会话并绑定 `deepseek-qa`
2. 刷新页面并切到识图模式
3. 上传图片（建议用 `input[type=file]`）
4. 输入定制化检查 prompt
5. 发送并读取回复

示例（按实际 ref 替换）：

```bash
# 1) 新建 + 绑定
opencli deepseek new
opencli browser deepseek-qa bind

# 2) 刷新并切识图模式
opencli browser deepseek-qa eval "location.reload()"
sleep 3 && opencli browser deepseek-qa bind
opencli browser deepseek-qa eval "document.querySelectorAll('[role=radio]')[2].click()"

# 3) 上传图片（注意：upload 作为刷新后的首个 DOM-marker 命令；否则可能触发 markerAttr 重复声明错误）
opencli browser deepseek-qa upload 'input[type=file]' /path/to/image.png
opencli browser deepseek-qa keys Escape

# 4) 查看 state 拿到 textarea 和发送按钮 ref
opencli browser deepseek-qa state

# 5) 输入并发送
opencli browser deepseek-qa type <textarea-ref> "这是一张[图片类型]。预期内容：[描述]。请检查：1)... 2)..."
opencli browser deepseek-qa click <send-btn-ref>
sleep 20
opencli deepseek read -f plain
```

## Prompt 建议

高质量 prompt 建议包含：
- 图片类型
- 预期内容
- 明确检查项（如溢出、重叠、对齐、对比度、可读性等）
- 输出格式要求（逐项回答，问题标 ❌，正常标 ✅）

## 常见问题

- `markerAttr has already been declared`：刷新页面后用 `eval` 切模式，再先执行 `upload`
- 上传后弹出搜索框：执行 `opencli browser deepseek-qa keys Escape`
- 没识别成图像问答：确认已切换到识图模式再上传

## 参考

- 详细规则与完整操作请查看仓库内文档：[`SKILL.md`](./SKILL.md)
