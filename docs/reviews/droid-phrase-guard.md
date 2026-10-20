# Droid 403 与 client-phrase guard — 设计评审

- 状态：草案（待实现）
- 日期：2026-10-20
- 范围：Factory (Droid) 订阅在 **droid CLI 原生客户端**上的 403；只针对 Factory 对
  client-identity 短语的过滤，不涉及内容安全 / 合规审查。
- 相关代码：`packages/factory`（magpie provider 插件，0.1.36）

> 本文中的触发短语都写成 JSON `\uXXXX` 转义形式。**不要把这些转义解码后粘进
> droid 会话**：解码后的原文就是会让 Factory 返回 403 的内容。

## 1. 问题

droid 会话在某个时间点后持续返回 403：

```
Error: 403
{"detail":"Forbidden","status":403,"title":"Forbidden","requestId":"hkg1::…"}
```

会话本身没有结束，但之后每次模型请求都失败，最终以
`agent_turn_outcome: model_authentication_failed` 结束。用户只能放弃或手工修复会话。

这类 403 与模型、账号额度、组织无关，是 **Factory 服务端对请求体里固定
client-identity 短语的过滤**。

## 2. 证据（本机实际会话）

会话：`dfbbf7ea-061d-478f-a60f-d7f3eb4d4c58`
（`~/.factory/sessions/-Users-byronwayne-Projects-formless-story-forge-agent/`）
droid：`0.244.0`，模型：`claude-opus-5-5`（High）。

| 时间 (UTC) | 事件 |
|---|---|
| 2026-10-19T18:15:07.101Z | tool result（`Execute`） |
| 2026-10-19T18:15:07.284Z | tool result（`Script`） |
| 2026-10-19T18:15:07.390Z | 第一个 403，requestId `hkg1::sp6rs-1792433707390-…` |
| 2026-10-19T18:34:01.562Z | 403，requestId `hkg1::4m4cv-1792434841562-…` |
| 2026-10-19T18:34:23.211Z | 403，requestId `hkg1::wjkhb-1792434863211-…` |
| 2026-10-19T18:34:52.264Z | 403，requestId `hkg1::86gm5-1792434892264-…` |
| 2026-10-19T18:34:54.422Z | `agent_turn_outcome: model_authentication_failed` |

两条 tool result 的内容是读取另一个 pi 会话 JSONL 的输出，其中引用了 pi 系统提示词
的开头（以下为转义形式）：

```
\u0059ou are an expert coding assistant operating inside pi, a coding agent harness.
```

tool result 的两条记录（JSONL 第 7444、7446 行）都包含这句话；第一条 403 的
requestId 时间戳与它们相差不到 300ms。此后每轮请求都会带上这段历史，因此持续 403。

**结论**：短语不是由 droid 系统提示词引入的，而是由 **工具输出**（读取/打印了
pi 会话文件）进入历史，并在后续每一轮请求中随历史一起发送。

## 3. 根因

Factory 服务端在请求体（序列化后的 JSON 原文）里做固定短语的子串匹配，命中即
返回通用 `403 Forbidden` + `requestId`。

- **精确、区分大小写**；
- 扫描的是请求体原文，不会先做 JSON 解码；
- 因此把短语首字符写成 `\uXXXX`（线上序列化为 `\\uXXXX`）就能避开匹配，而
  模型仍能读懂/解码该文本。这一点已由 `@magpie-community/opencode-factory-auth`
  0.1.35 的修复验证：tool output 里的同一段文本从 403 变为 200。

droid 原生客户端不会做这种转义，所以一旦短语进入会话历史，会话就中毒了。

短语分两组（来源：插件 `client-phrases.json`）：

| 组 | 短语（转义形式） | 行为 |
|---|---|---|
| anywhere | `\u0059ou are an expert coding assistant operating inside pi, a coding agent harness.` | 任何文本中出现都拒绝；本会话由它触发 |
| anywhere | `\u0054he conversation history before this point was compacted into the following summary:` | 同上；压缩时最危险，因为历史会被压成一段用户文本 |
| toolOnly | Claude Code 身份句、`\u0059ou have been invoked in the following environment:`、`\u0078-anthropic-billing-header…`、`\u0054his session is being continued…`、`\u0028user's private global instructions…)` | 在 tool output 中被拒绝（其他 client 的请求）；本会话里 Claude Code 句从 10:47 躺到 18:15 都没触发，说明 droid 自身对这些短语是放行的 |

因此 guard 应该：

- 对 `anywhere` 组 **硬拦截**；
- 对 `toolOnly` 组只 **警告**，不要误杀。

## 4. droid hook 的能力边界

从 droid 0.244.0 二进制里的 hook runner / schema 确认（事件：
`PreToolUse`、`PostToolUse`、`Notification`、`UserPromptSubmit`、`Stop`、
`SubagentStop`、`PreCompact`、`SessionStart`、`SessionEnd`）。

| 事件 | 能否改内容 | 机制 / 限制 |
|---|---|---|
| `UserPromptSubmit` | ✅ 改 prompt | `hookSpecificOutput.updatedInput.prompt` 会替换本轮 prompt |
| `PreToolUse` | ✅ 改 tool_input / 拦截 | `updatedInput` 合并进 input；`permissionDecision: allow\|deny\|ask`；`continue:false` / exit 3 中止；exit 2 拒绝 |
| `PostToolUse` | ❌ 不能改 tool_response | 只能 `additionalContext`（追加系统消息）、`systemMessage`（用户可见警告）；`continue:false` 抛 `HookStopError`，**结果被丢弃**，因为 `updateToolResult` 在 hook 之后才执行 |
| `SessionStart` | 能改 transcript 文件 | **在历史加载之后**才运行（`loadSession` 先读 `y`，再 `executeSessionStartHooks`），改文件不影响当前这次 resume |
| `SessionEnd` / `Stop` | 能改 transcript 文件 | 适合清理；下次 resume 生效 |
| `PreCompact` | 能阻止压缩 | 不能改内容 |

关键结论：

1. **PostToolUse 是唯一能阻止 tool output 进入历史的 hook 点**，代价是本轮中止、
   该条结果丢弃（读/执行类工具通常没有 `committedToolIds`，可以丢；写文件类
   committed 工具可能已经进历史，拦不住）。
2. **SessionStart 不能修当前一次 resume**；要修已经有毒的会话，必须在 droid 启动
   之前用独立 cleaner 处理，或者依赖上一次 `SessionEnd` 的清理。
3. `UserPromptSubmit` / `PreToolUse` 可以改文本，但**改 tool input 要谨慎**：对写文件
   工具做转义会把 `\u0059` 写进文件内容，所以对写类工具更适合 `deny`。

## 5. 方案对比

### A. 手工 / CLI cleaner（先做）

逐行 `JSON.parse` transcript，递归遍历所有字符串，把 `anywhere` 短语替换成
`\uXXXX` 形式（首字符转义），再 `JSON.stringify` 写回；备份 + 校验 JSON。

- 优点：零运行时风险；能救已有会话；
- 缺点：只能事后处理，不能阻止新一轮中毒；
- 适用：`droid --resume` 之前跑一次。

### B. droid hook guard（推荐主线）

一个脚本 `~/.factory/hooks/phrase-guard.mjs`，按 `hook_event_name` 分发：

| 事件 | 行为 |
|---|---|
| `UserPromptSubmit` | 命中 → 把 prompt 里的短语转义后通过 `updatedInput.prompt` 写回 |
| `PreToolUse` | 命中 → `permissionDecision: deny` + 原因（阻止把短语写进文件/命令） |
| `PostToolUse` | 命中 `anywhere` → `{"continue": false, "stopReason": "..."}`，丢弃结果并提示重跑；命中 `toolOnly` → 只 `systemMessage` 警告 |
| `SessionEnd` / `Stop` | 清理 transcript，供下次 resume |
| `SessionStart` | 可选：再清一次文件（对当前 resume 无效，但可修复外部改动） |

Hook 输出示例：

```json
{"hookSpecificOutput":{"hookEventName":"UserPromptSubmit","updatedInput":{"prompt":"…\\u0059ou are…"}}}
```

```json
{"continue":false,"stopReason":"This tool output contains a phrase Factory refuses; the result was discarded. Re-run the command with the phrase filtered."}
```

```json
{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny","permissionDecisionReason":"This call contains a phrase Factory refuses. Rewrite it before retrying."}}
```

注意 stopReason / reason 里**不要**出现短语原文，避免 hook 自己的输出再次触发。

### C. 让 droid 走 magpie 的 factory provider

droid 原生请求不经过插件；如果让 droid 用自定义模型指向 magpie gateway，则
0.1.35+ 的 `sanitizeBody` 会在请求出口统一转义，hook 都不需要。

- 优点：最彻底，不依赖 droid hook；
- 缺点：自定义模型 / 工具调用 / provider 路由的兼容性要验证，且引入额外网关依赖；
- 结论：作为 B 不够用时的备选，不作为第一步。

### D. 不处理

每次读到 pi/Claude Code 的会话文件就赌运气，不可接受。

## 6. 推荐

**A + B**：

1. 先发布 cleaner（手动 / 可选 alias），救已有会话；
2. 再装 guard hook：`UserPromptSubmit` 转义、`PreToolUse` deny、
   `PostToolUse` 对 `anywhere` 中止并丢弃、`toolOnly` 仅警告；
3. `SessionEnd` 清理 transcript，保证下一次 resume 干净；
4. 如果实现后发现 committed 工具仍会漏，再评估 C。

短语表必须与插件共用同一份 `client-phrases.json`（首字符已转义，读它本身不会
触发 403）。升级插件时同步更新 hook 的副本。

## 7. 测试计划

**单元**

- cleaner：嵌套 JSON、转义引号、多短语、幂等、JSON 合法性、备份；
- matcher：`anywhere` / `toolOnly` 分组，大小写敏感，不应匹配片段；
- hook 契约：用样例 stdin 跑每个事件，断言 stdout JSON 形状；
- 自检：`grep` 确认脚本、README、fixtures 里没有短语原文（只有 `\uXXXX`）。

**端到端（scratch 会话）**

1. 在临时 droid 会话里让工具输出包含 `anywhere` 短语；
2. 期望 `PostToolUse` 中止、结果不落盘、界面显示 stopReason；
3. 下一轮请求应为 200；`Stop` / `SessionEnd` 后 transcript 不含原文；
4. `droid --resume` 同一会话，确认不再 403；
5. 对照：`toolOnly` 短语只警告、会话继续可用；
6. 回归：普通会话（无短语）不受影响。

## 8. 风险与待决问题

- **PostToolUse 中止的 UX**：本轮工具白跑，用户需要重跑；这是阻止中毒的必要代价。
- **committed 工具**：写文件/有 side effect 的工具可能已提交结果，abort 不一定能丢弃；
  需要实测并记录。
- **SessionStart 时序**：当前 resume 仍可能带着旧短语；必须靠 cleaner 或上次
  SessionEnd。文档和 CLI 要明确这一点。
- **短语表漂移**：Factory 可能增删短语；跟随插件升级，必要时加一个 fixture/探针。
- **hook 维护**：脚本放在 `~/.factory/hooks/`，不随插件发布；若要共享，考虑做成
  droid plugin / marketplace。
- **边界**：本方案只处理 client-identity 误报，不用于规避内容安全审查。

## 9. 决策（待确认）

- [ ] 采用 A + B；
- [ ] `toolOnly` 只警告；
- [ ] `PostToolUse` 命中 `anywhere` 时中止并丢弃；
- [ ] cleaner 的入口：`phrase-guard clean <session.jsonl>`；
- [ ] 是否顺带把 hook 打包成 droid plugin（可选）。

## 附录：短语表（转义形式）

```
anywhere:
  \u0059ou are an expert coding assistant operating inside pi, a coding agent harness.
  \u0054he conversation history before this point was compacted into the following summary:

toolOnly:
  \u0059ou are Claude Code, Anthropic's official CLI for Claude.
  \u0059ou are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK.
  \u0059ou are a Claude agent, built on Anthropic's Claude Agent SDK.
  \u0059ou have been invoked in the following environment:
  \u0078-anthropic-billing-header: cc_version=
  \u0054his session is being continued from a previous conversation that ran out of context.
  \u0028user's private global instructions for all projects)
```

注意：上表每一行都只转义了首字符；**解码任意一行都会得到触发 403 的原文**，
不要把这些转义解码后粘贴到 droid 会话。实际使用时以 `client-phrases.json` 为准，
本文仅作说明。
