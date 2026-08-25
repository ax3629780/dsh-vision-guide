# dsh-vision-guide

给**纯文本**的 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 模型（如 `deepseek-v4-pro`）加上图片理解能力，**不需要改核心包**。

> English guide: [README.md](./README.md)

> **已验证版本**：`@deepseek-ai/dsh@0.1.0-rc.6`。用到的扩展接缝（`ctx.llm.resolveModelInfo`、`llm/stream`、`inputModalities`）属 pre-1.0，可能随版本调整。

## 要解决的问题

DeepSeek V4 Pro / Flash 等很多强编码模型是**纯文本**的，在聊天里贴图会直接报错：

> Model "...": does not accept image input

## 解决方案

使用社区插件 [`@dsh-plugin/dsh-auxiliary`](https://github.com/dsh-plugins/dsh-auxiliary)，它在**官方 `ctx.llm` 接缝**上叠加了视觉（及可选的压缩 / 子代理 / 标题 / 审批路由）：

1. 通过包装 `resolveModelInfo`，让纯文本模型「宣称支持图片」，官方放行检查自然通过——**无需改核心包**；
2. `llm/stream` 监听器把图片块重写为轻量的 `[image: {...}]` 文本引用；
3. 主模型**按需**调用 `describe_image` 工具，从视觉模型（示例里是 `gpt-5.6-sol`）取回图片内容。

## 快速开始

```sh
# 1. 安装（需要 PATH 里有 pnpm）
dsh plugin --profile web add @dsh-plugin/dsh-auxiliary

# 2. 在 ~/.dsh/settings.yaml 配置视觉路由
```

```yaml
dsh-auxiliary:
  vision:
    provider: new-api          # 已注册的 provider
    model: gpt-5.6-sol         # 该 provider 下支持图片的模型
```

```sh
# 3. 重启
dsh web
```

之后在聊天里贴图即可。UI 里保留原图，主模型通过 `describe_image` 查看内容。

## 为什么选这个方案（优势）

| 优势 | 说明 |
|---|---|
| **复用已有 provider** | `vision.provider` / `vision.model` 直接指向 `llm-pi-ai` 里已注册的模型，不用猜 baseURL |
| **不 hack 核心包** | 走官方接缝（`ctx.llm`、`resolveModelInfo` 包装、`llm/stream` 瀑布），重装不丢 |
| **按需识图（省）** | 图片先变轻量文本引用，只有主模型真正调用 `describe_image` 时才跑视觉模型 |
| **一个插件多条路由** | 视觉 + 压缩 + 子代理 + 标题 + 审批 + 生图 |
| **CI + TypeScript** | 每次发布前在 Linux/macOS/Windows 三平台跑 `typecheck` + `build` |

## 参考的开源项目

本指南是在调研了多个社区视觉插件后写成的。完整对比和选型理由见 [docs/alternatives.md](./docs/alternatives.md)。

## 踩坑记录

崩溃（`prepared LLM call config changed...`）和「无法切换模型」这两个坑的根因与修复见 [docs/troubleshooting.md](./docs/troubleshooting.md)。

## 降本优化

除了视觉，`dsh-auxiliary` 还能把压缩、标题、子代理、审批、生图等工作路由到更便宜的模型，见 [docs/optimizing.md](./docs/optimizing.md)。

## License

[MIT](./LICENSE)
