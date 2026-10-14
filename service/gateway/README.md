# Proof Gateway

Gateway 位于模型客户端与上游模型 API 之间。它在请求发往模型之前读取 Proof 的 canonical 状态投影、追加必要状态，并在断片期间控制历史可见性。Gateway 不维护第二套生理状态。

## 支持的接口

- OpenAI Chat Completions：`/v1/chat/completions`
- OpenAI Responses：`/v1/responses`
- Anthropic Messages：`/v1/messages`

客户端把模型 `base_url` 指向 Proof Gateway，并通过 `x-proof-gateway-key` 提交对应 Agent 的 Gateway token。需要稳定续接时同时发送 `X-Proof-Conversation-Id`。

## 配置

| 变量 | 用途 |
|---|---|
| `PROOF_GATEWAY_ENABLED=1` | 启用 Gateway 路由；默认关闭 |
| `PROOF_OPENAI_BASE_URL` | OpenAI 兼容上游的 HTTPS `/v1` 地址 |
| `PROOF_OPENAI_API_KEY` | 服务端持有的 OpenAI 兼容上游密钥 |
| `PROOF_ANTHROPIC_BASE_URL` | Anthropic 兼容上游的 HTTPS `/v1` 地址 |
| `PROOF_ANTHROPIC_API_KEY` | 服务端持有的 Anthropic 兼容上游密钥 |
| `PROOF_GATEWAY_ALLOW_SOFT_BLACKOUT=1` | 允许上游托管历史在断片期显式降级为软断片 |

生产上游只允许公网 HTTPS 地址，不跟随重定向，也不接受 URL userinfo、查询串或私网目标。客户端提交的上游鉴权头会被移除，再由 Gateway 注入服务端密钥。错误与日志会遮蔽密钥。

## 身份与状态

Gateway token 只映射到服务端注册的一个 Agent identity。身份不由请求正文或模型自报决定。每个身份拥有独立的 Proof 状态与无正文消息账本。

Gateway 每轮调用 `buildAgentTurnContext(engine, agentId, now)` 获取状态，不自行计算酒精、敏感度、断片、reset 或 revision。消息账本只保留指纹、时间与恢复发射标记，不保存聊天正文。

## 注入位置

动态 `[Proof 状态]` 追加在本轮输入末尾，以尽量保持 system 与既有历史前缀稳定。Chat Completions 有两条落地路径，差别很大：

- **尾部是用户消息**（普通对话）：状态追加在该消息正文之后。用户原文仍在最前，归属本来就清楚。
- **尾部不是用户消息**（主动唤醒收尾于 assistant、或 tool result 收尾）：provider 不接受这两种角色收尾，因此只能另起一条 `user` 消息承载状态。这条消息必须带 `GATEWAY_ENVIRONMENT_ENVELOPE` 自报身份。

第二条路径里的声明不是装饰。它缺席时，模型会把整条读成「这一轮用户说的话」，发现里面只有系统状态，于是推断「没有新消息、这轮是我的回合继续」，开始自己跟自己接话。回归用例见 `tests/gateway-adapters.test.mjs` 的 OA8–OA10：声明必须在状态正文之前；尾部是真实用户消息时不得出现声明；声明不得进账本的 `proofContext`，否则下一轮 replay 会把它贴到用户真实消息的末尾。

Responses 走 `instructions`、Anthropic 走 `system`，两者本来就不占对话轮次，不需要声明。

## 注入与缓存

状态注入会增加输入 token，也可能影响 prompt cache；不同 provider 的缓存边界和计费规则不同，应以上游 `usage` 为准。

状态第一次出现、revision 变化、reset 或需要重新提醒时会产生新的动态输入。去重可以减少重复状态，不能让变化后的状态免费。

## 断片边界

Gateway 只能过滤实际经过它、并能由它重写的模型输入：

- 客户端每轮提交完整 `messages` 或 `input` 时，可以执行硬过滤。
- 使用上游托管历史、只提交 `previous_response_id` 等引用时，Gateway 无法删除上游已经保存的内容；默认拒绍，或在显式开启后降级为软断片。
- 原始历史仍在客户端输入中时，恢复阶段可以按允许的清晰度取回对应片段。
- 原始历史已被宿主压缩或删除时，只能使用 Proof 留下的低分辨率事实。

Gateway 不修改宿主本地聊天记录，也不声称删除模型提供方已经保存的历史。

## 与 MCP 的关系

Gateway 负责自动状态投递与上下文过滤，不给 Agent 增加点酒、领取 link 或 reset 工具。若希望 Agent 在聊天中主动喝酒，应同时接入 MCP，或由宿主实现等价的 authenticated HTTP 调用。

MCP 与 Gateway 使用同一个 Agent identity 时，共享同一份持久状态；它们不是两套身体。

## 网络要求

域名不是协议硬要求。本机、内网或 VPN 可以直接使用可达 IP；公网部署应使用可信 HTTPS。不要在公网明文 HTTP、前端代码、URL、Markdown 或日志中暴露 Gateway token 与上游 API key。
