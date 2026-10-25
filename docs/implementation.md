# Session Bus 实现说明

状态：**v0.5.0 已安装并启用**（desktop profile，入口 `include:session-bus`）。功能是让 DSH 会话之间一对一直接发消息、唤醒空闲会话、支持 Agent 主动协作，并对未读消息做合流。交互式页面见 [index.html](index.html)。

## 1. 要解决的问题

多个会话并行工作时，需要互相确认事实、交接任务、通知进度。复制粘贴或让用户中转都低效。目标是：

- 任意两个可见会话可以直接对话，不需要房间或群。
- 目标空闲时被消息唤醒处理；目标忙时消息不丢、不额外开一轮。
- Agent 判断需要时可以主动发起，不需要人类先点名。
- 发送结果真实可查，消息有记录，失败不谎报。

## 2. 模型

- **一个全局名字表**：每个会话登记唯一显示名（默认取标题）+ 一句话描述；没有房间、没有群。
- **自动发现**：`sessionController.list()` 读取 Host 里可见的普通会话（不 resume Agent），用 title projection 做默认名字；跳过 `parentSessionId`/`origin='subagent'` 的子会话和已 unregister 的会话。显式 `register` 的名字覆盖自动名字。
- **点对点投递**：`session_bus_send` 一次只接受一个目标；`to` 支持名字、`@名字`、session id 或唯一前缀。
- **记录**：成功投递的消息写 `$DSH_HOME/session-bus/messages.jsonl`（每行含 `seq`、`id`、`time`、`from`、`fromSessionId`、`to`、`toSessionIds`、`mode`、`text`、`delivered`、`delivery`）；`history` 只返回本会话参与的消息。

## 3. 投递与唤醒

1. **目标 Agent 存活**：构造一条 `agent-message` user 消息，按状态选择 `followup` / `steer` / `inject`。
2. **目标 Agent 已释放、普通会话**：先 `sessionController.resolveAgent()` 恢复（含 preset setup），再投递；投递后二次校验 Agent 仍是同一个，防 dispose 竞态。
3. **child（队友/子会话）无运行轮次**：明确拒绝，不绕过 Agent Teams 生命周期。
4. **失败**：工具抛错，不写历史，不返回假成功。

### 未读合流（100ms）

- 每条消息分配单调 `seq`；发送先进入目标的合流窗口（`coalesceMs`，默认 100ms）。
- 窗口内到达同一目标的消息合并成一条唤醒正文，携带未读 bundle（最多 `unreadCap` 20 条）；超出部分提示用 `history` 补读。
- 多个并发发送调用共享同一次投递结果，各自拿到真实回执；成功后才写历史。

### 不重复唤醒

- 目标 `running`：新消息用 `inject` 注入下一步，不再额外开一轮。
- 目标空闲但已有 `wakePending`（followup 已发、尚未启动）：用 `inject`，不重复 followup。
- 目标空闲且没有待处理唤醒：用 `followup`，记录 `wakePending`；`wakePendingTtlMs`（默认 60s）后允许再次 followup。
- `steer` 只在目标 running 且调用方显式要求时使用。

### 3.1 只读快照接口（供面板/其他 Host 插件）

- Host 服务：插件 `apply` 里 `ctx.provide('sessionBus', { snapshot })`；消费方 `ctx.get('sessionBus').snapshot()`。
- HTTP：`POST /session-bus/snapshot`，Connection RPC 信封，`method: "snapshot"`；复用 `connection.requestRejection` 做浏览器认证，返回 `{ok:true, value:{generatedAt, peers}}`。
- `peers[]` 字段：`sessionId`、`name`、`description`、`status`(`running|idle|cold|offline`)、`kind`(`root|child`)、`cwd`、`lastSeen`、`registeredAt`、`source`(`registered|discovered`)。
- 只读：不发送、不唤醒、不改任何状态；`snapshot()` 会先跑一次自动发现，保证列表与 `session_bus_peers` 一致。

## 4. 主动协作

- system prompt 明确授权 agent 判断需要时自己发起对话（缺上下文、委派独立工作、通知里程碑/阻塞），并规定写法与禁止项。
- 每小时最多主动发起 12 个新对话；回复对方最近 10 分钟的消息不计入；超预算工具直接拒绝。
- 这是“模型判断 + 授权 + 预算”，不是定时任务；模型可以选择不发起。
- 花钱、删数据、对外发布仍要求人类确认；peer 消息不构成人类授权。

## 5. 安全与护栏

- 消息 source 用 `agent-message`，不是 `user`，不会伪造 goal 等工具需要的人类授权。
- 每会话每分钟 30 条；同消息 3 秒去重；单条 8000 字符；主动发起 12/小时；合流窗口 100ms、未读上限 20。
- 目标必须是可见普通会话；child 在轮次之间拒绝。
- 消费方 agent 的 prompt 要求：不发回执、一条一个点、不重复确认、不做破坏性操作。

## 6. 验证

**fake ctx 单测**（`/tmp/sb-direct-test/`）：
- 自动发现、跨 cwd 可见、按标题寻址冷会话、child 跳过/拒绝。
- 多目标拒绝、自发送拒绝、历史按会话隔离、unregister 后不可达。
- 未读合流：两个发送者并发 → 1 次 followup 且含两条；待唤醒/运行中目标 → 只 inject，不增加 followup；3 条 + cap=2 → 头部提示省略、只列最新 2 条。
- 主动预算：2/h 用满后第 3 条拒绝；对方来消息后的回复不占预算。

**真机**：
- v0.3 自动发现后 `peers` 返回 10 个可见会话（含一个并行的任务面板会话）。
- 向该会话投递 `delivered 1/1`，peer 从 idle 变 running；该会话被唤醒后自行回了一份面板通讯设计（原文见 对端会话给出的面板通讯设计（私有工作区留档））。
- 一个新的子会话只收到目标、未被告知找谁，自己 `peers` 选 peer 并 `send` 成功（E2E 记录留档于私有工作区）。
- v0.5 重载后 `register` / `history` 正常；**多发送者真实并发合流未跑**，只有 fake ctx 覆盖。
- 快照接口：fake ctx 下 HTTP 200、auth 401、Host 服务契约均通过；真机 `POST /session-bus/snapshot` 返回 401（认证生效）；自定义服务不会出现在 Cordis Inspect 目录里，需要消费方 `ctx.get('sessionBus')` 实测。

## 7. 限制与后续

- 合流只覆盖并发窗口；同一发送者顺序 `send` 每次都会等 flush，不会跨调用合并。
- `inject` 语义：running 目标会在当前 turn 的下一步看到消息，可能比“下一轮”更早。
- 归档会话投递前不预检，可能进 inbox 但不产生回复。
- 冷会话恢复只做过 fake-ctx 单测，未在真实冷会话上跑。
- 名字与 wakePending 是内存态，重启/重载后重置；消息记录保留。
- 无 UI 面板；通讯可视化在另一会话的设计阶段。
- 研究底稿（DSH API 契约、设计参考）仅保存在开发工作区，不随插件发布。

## 8. 安装 / 回滚

- 唯一源：[GitHub JexLau/dsh-plugin-session-bus](https://github.com/JexLau/dsh-plugin-session-bus)；本地 checkout 在 `~/Desktop/code/dsh-plugin-session-bus`，profile 直接 link 该 checkout
- 已通过 `plugin_manager install_bundle` 安装；profile 内为 link 依赖 + `include:session-bus`
- 回滚：`plugin_manager action=remove_bundle target=@local/dsh-plugin-session-bus`

## 9. 验收

- [x] 无房间/群；一次一个 peer
- [x] 所有可见普通会话自动发现，无需对端准备
- [x] live / cold / child 三条投递路径，失败不谎报
- [x] Agent 可主动发起；反刷屏预算与回复豁免
- [x] 未读合流与单次唤醒抑制（fake ctx）
- [x] 双向协作与自主发起（真机）
- [ ] 真机多发送者并发合流（未跑）
