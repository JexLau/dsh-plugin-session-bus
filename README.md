# Session Bus：让 DSH 会话之间直接说话

## 它解决什么问题

你同时开多个 DSH 会话做不同的事，经常需要它们互相知道对方的情况：

- 这个会话要问另一个会话：“你那边 PlayerController 的字段布局定了吗？”
- 一个会话做完了，要通知另一个会话可以接着干。
- 两个会话要并行分工，需要把一部分上下文或任务交接过去。
- 你不想切窗口、复制粘贴，也不想让消息中转经过你。

Session Bus 让任意两个会话直接对话：A 发一条消息给 B，B 空闲会被唤醒处理；B 可以直接回 A。没有房间、没有群、没有广播，一次只发给一个会话。

## 怎么用

不需要安装、不需要配置，直接在当前会话说：

```text
看看现在有哪些会话可以直呼。
```

它会列出所有可直呼的会话（名字、描述、状态、所在目录）。

然后让 agent 代发：

```text
问一下 @Decompiler：PlayerController 的字段布局定了吗？
```

```text
把这条转给 @Auth：认证部分我已经改完，可以接着做联调。
```

对方会话会被唤醒并开始一轮，用同样的 `session_bus_send` 回复你。你不用切窗口。

也可以让它主动协作，不必你点名：

```text
这件事你判断需要谁配合就直接找对端会话，不用问我。
```

改自己的称呼 / 停止接收：

```text
把登记名改成 Auth，描述写“负责认证重构”。
不要再接收直接消息了。
```

## 工具

| 工具 | 作用 | 参数 |
|---|---|---|
| `session_bus_peers` | 列出可直呼会话（自动发现所有可见普通会话） | 无 |
| `session_bus_send` | 给一个会话发直接消息；空闲会被唤醒 | `to`（名字/@名/session id，只能一个）、`message`、`mode`（`queue` 默认 / `steer`） |
| `session_bus_history` | 读本会话收发的直接消息 | `limit` 1–100 |
| `session_bus_register` | 显式改名 / 写职责描述 | `name`、`description` |
| `session_bus_unregister` | 停止接收直接消息 | 无 |

## 行为

- 自动发现：`peers` / `send` 会列出 Host 里所有可见普通会话，对端不需要提前登记；名字默认取会话标题。
- 点对点：一次只发给一个会话；要通知两个就发两次，没有广播。
- 投递与唤醒：
  - 目标 Agent 活着：直接投递；空闲时 `followup` 唤醒它开始一轮。
  - 目标 Agent 已释放的普通会话：先 `sessionController.resolveAgent()` 恢复，再投递。
  - 队友/子会话在两次 turn 之间不可达（属于 Agent Teams 生命周期）。
- 未读合流：多条发往同一会话的消息在约 100ms 内合并成一次唤醒，一次最多带 20 条；超出会提示用 `history` 补读。
- 不重复唤醒：目标正在运行、或已经有一条消息在等它启动时，新消息只注入到下一步，不再额外开一轮。
- 回执真实：`session_bus_send` 返回的 `delivered` 是实际投递结果；失败会报错，不写“已送达”。
- 记录：本会话的收发记录落盘 `$DSH_HOME/session-bus/messages.jsonl`；`history` 只看得到本会话参与的消息。
- 防刷屏：每会话每分钟 30 条、同消息 3 秒去重、每小时最多主动发起 12 个新对话（回复不计）。

## 主动协作

system prompt 明确授权 agent 在判断需要时自己发起会话对话，不需要人类先 `@`：

- 触发：需要只有另一个会话才有的上下文或产出、把独立工作交给更合适的会话、通知里程碑或阻塞。
- 做法：先 `peers` 找最相关且在线的一个会话，一条消息写清：你是谁、需要什么、期望输出、是否阻塞你、何时需要回复。
- 不刷屏：不群发、不进展刷屏、不空回执、不重复追问；对方没回就等或问人类。
- 预算：每小时 12 个新对话，超出会被工具拒绝；回复对方最近消息不计入。
- 花钱、删数据、对外发布仍然先问人类。

## 开发文档

实现细节、投递状态机、验证记录与限制见 [docs/implementation.md](docs/implementation.md)。

## 给面板 / 其他插件读取（只读）

其他 Host 插件可以读同一份 peers 快照，不需要安装或触发发送：

```js
const snapshot = await ctx.get('sessionBus')?.snapshot()
// { generatedAt, peers: [{ sessionId, name, description, status, kind, cwd, lastSeen, registeredAt, source }] }
```

- `ctx.get('sessionBus')` 是 Host 服务；`snapshot()` 只读，不发送、不唤醒。
- Web Client 同源请求走 `POST /session-bus/snapshot`（Connection RPC 信封，`method: "snapshot"`），需要浏览器会话认证；返回 `{ok:true, value:{generatedAt, peers}}`。
- `source` 为 `registered`（显式/首次登记）或 `discovered`（自动发现）；`kind` 为 `root`/`child`，面板一般只展示 root。

## 配置

`cordis.patch.yml` 里给 `session-bus` 行传 config：

| 字段 | 默认 | 说明 |
|---|---|---|
| `enabled` | `true` | 关掉插件 |
| `language` | `zh` | 唤醒消息和 system prompt 语言，可选 `zh`、`en` |
| `autoJoin` | `true` | 首次工具调用自动登记本会话；`false` 时必须先 `session_bus_register` |
| `autoDiscover` | `true` | 自动发现 Host 里所有可见普通会话；`false` 时只显示主动登记过的会话 |
| `promptSection` | `true` | 是否注入 system prompt 使用说明 |
| `historyPath` | `$DSH_HOME/session-bus/messages.jsonl` | 直接消息记录路径 |
| `maxMessageChars` | `8000` | 单条消息上限 |
| `maxInitiatedPerHour` | `12` | 每小时主动发起的新对话上限；回复不计；`0` 不限 |
| `coalesceMs` | `100` | 同一目标的消息合并窗口（毫秒） |
| `unreadCap` | `20` | 一次唤醒最多携带的未读消息数 |
| `wakePendingTtlMs` | `60000` | 已安排唤醒但目标迟迟未启动时，多久后允许再次 followup |
| `rateLimitPerMinute` | `30` | 每会话每分钟发送上限 |
| `dedupeWindowMs` | `3000` | 相同消息去重窗口 |

## 边界

- 一次一个 peer。这是刻意的，和“直接对话”一致。
- 只对可见的普通会话；队友/子会话不会出现在 peers 里，也不能在轮次之间被唤醒。
- 给冷会话发消息会恢复它并跑一轮，有 token 成本。
- 归档会话没有投递前预检：消息进 inbox，但归档的 pre-step 会被拒绝，不产生回复。
- 名字是内存态：Harness 重启后清空，会话下次调用工具时按标题重新发现；消息记录保留。
- 没有 UI 面板；可视化面板还在另一会话的设计阶段。
