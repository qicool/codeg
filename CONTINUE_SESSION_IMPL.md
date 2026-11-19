# 委派子会话续聊：continue_with_session / close_session 实现说明

本目录改动在 **codeg 0.46.0（HEAD b457eca0b，2026-11-18）** 上补齐了委派 v2 的两个 MCP 工具，对应上游 open issue #602 / #603 与未合并 PR #382 的需求。

## 研究结论（为什么这样做，而不是移植 PR #382）

1. 当前 main 的 `delegation/mod.rs` 明确写着：
   > "v1 is one-shot ... **v2 will introduce `continue_with_session` / `close_session` tools without protocol breakage.**"
   即协议层（BrokerMessage / transport）本就设计成能无痛加这两个工具。
2. 底层原语已全部就位：`spawn_for_resume` + `send_resume_prompt`（按 external session id 恢复子 ACP 会话）、`has_live_connection_for_conversation`、`PendingInner` 的 completed cache / resuming claim / inflight 竞态框架、`CompletedTask` 已存 child_conversation_id 等。
3. PR #382（2026-07-26 基线、3 commit、56 文件、+9096 行）**未合并**。补全 git 历史后重跑 cherry-pick 仍是 **40 个文件冲突**（PR 基线 `dea521f` 与当前 main 相隔 4 个月、上千 commit，核心文件膨胀 1.3~4 倍），且含 modify/delete 冲突。机械移植不可行——PR 大半改动是前端 UI / i18n / HTTP 命令层。

因此本实现直接在当前代码的 v2 扩展点上做，改动集中在 delegation 后端 6 个文件，前端零改动。

## 设计思路

`resume_delegation` 已具备续聊所需的全部复杂逻辑（状态门 → 按 external_id 恢复子会话 → 发 prompt → 重新注册 running → 复用 `get_delegation_status`/`cancel_delegation` 收尾），唯一限制是只接受 Canceled/中断状态且 prompt 固定。本实现把同一条管线参数化为三种模式，靠请求上的字段分流：

- `message = None, close_session = false`：原 `resume_delegation`，行为完全不变。
- `message = Some(...)`：`continue_with_session`。允许 `Completed` 状态，把 `message` 作为子会话下一条 prompt 原样发送。
- `close_session = true`：`close_session`。取消 running 的 turn、标记会话 released，之后 `continue_with_session` 拒绝该 id。

`close_session` 与 `continue` **共用 `BrokerMessage::ResumeTask` 同一条 wire**（靠 `close_session` 布尔标志分流），不新增消息变体 / round-trip / listener arm，改动最小。`#[serde(default)]` 保证旧 companion 省略新字段时仍兼容。

## 工具契约

```
continue_with_session(task_id, message, continuation_id)
  - task_id: delegate_to_agent 返回的 id（续聊沿用同一个 id，见下方语义）
  - message: 发给同一子会话的增量指令（子会话带完整历史）
  - continuation_id: 调用方生成的幂等键，重试同请求传同值
  - 返回: Running ack；结果用 get_delegation_status 收
  - 拒绝: 仍在运行 / 已 close / 对该父会话未知

close_session(task_id, continuation_id)
  - 取消 running 的 turn、释放 child 连接、标记 released
  - 保留 child conversation 与 transcript（只释放资源，不删数据）
  - 已完成时返回最后已知状态；幂等回放
```

## 「复用同一 task_id」的含义与代价（重要）

`task_id` 是 broker 里 `running`/`completed` 两张 HashMap 的 key，也是前端 delegation 卡片绑定的 id，一个 id 对应一条任务、一个终态。

现有 `resume_delegation` **本来就是**复活同一 task_id（`completed → running → completed`），其 resuming claim / inflight / disposition 竞态框架原生支持。本实现的续聊复用这条成熟管线，因此：

- 状态在该 id 上回流，旧一轮的 `completed` 结果会被新一轮覆盖——**多轮历史不形成独立时间线**。
- 子 Agent 自身的会话仍保留完整多轮上下文（续聊复用了同一个 ACP session），所以「上下文连续性」这个核心诉求是满足的；被覆盖的只是 codeg 侧对该 task 的**报告缓存**。
- 前端卡片、`get_delegation_status` 的 completed cache、cancel、长轮询都按「一个 id 一个终态」设计，状态回流在这些地方语义上偏弱但功能可用。

PR #382 用「每轮独立 turn_id + 稳定 Session」规避了覆盖，但那需要改状态查询层（`classify_locked` / `get_tasks_status`）与前端多轮时间线，属完整后端+前端改造。本次按「MCP 工具层补齐」的范围，**保留同一 task_id**，并在 `continue_with_session` 的工具描述里明确写出覆盖提示。**如果调用方需要保留前几轮结果，应在发起下一轮续聊前自行留存。**

## 改动清单（6 个文件，约 300 行新增）

| 文件 | 改动 |
|---|---|
| `src-tauri/src/acp/delegation/types.rs` | `ResumeDelegationRequest` 增加 `message` / `continuation_id` / `close_session` 字段（serde default） |
| `.../transport.rs` | `BrokerResumeTaskRequest` 增加同名字段；测试构造点补齐 |
| `.../listener.rs` | `process_resume_task` 透传新字段；测试构造点补齐 |
| `.../broker.rs` | ① `PendingInner` 增加 `continuations` 幂等 ledger 与 `released` 集合；② `build_continue_prompt`；③ `resume_delegation` 入口幂等回放 + CLOSE 分流 + CONTINUE 拒绝 released；④ 内存门与 DB 门在 `Completed + message.is_some()` 放行；⑤ 发送时按模式选 prompt；⑥ 成功 ack 写 ledger；⑦ `close_session` 方法（复用 `cancel_task_by_id`）；⑧ 测试 helper 补字段 |
| `.../companion.rs` | `allows_tool` 暴露两个工具；新增 `continue_with_session` / `close_session` 两个 `tools/call` arm（参数校验、mint external_handle、复用 resume round trip） |
| `.../tool_schema.json` | 增加两个工具定义（44 个工具）；`continue_with_session` 描述补充 task_id 复用/覆盖提示 |

## 关键语义与已知边界

1. **运行中的任务**：`Running` 仍被拒绝（提示先等待或取消），与 PR #382 一致。
2. **幂等**：`continuation_id` 进程内去重。并发同 key 由 `resuming` claim 间接挡住；串行重试由 ledger 回放首次结果。进程重启后 ledger 清空（与内存 completed 缓存同生命周期）。
3. **close 后 continue 拒绝**：`released` 集合进程内维护；复用 `not_resumable_report`（error_code `not_resumable`）报拒绝。
4. **原 resume 不受影响**：`message = None` 且 `close_session = false` 的所有分支、错误文案、状态约束保持原样，旧单测应全部通过。
5. **未做（完整后端/前端改造范围外）**：独立 turn_id / turn_version、多轮时间线 UI、`@Session` 分组、keep-alive 保活上限、HTTP/Tauri continue 命令、i18n、close 对已结束父会话的恢复。这些是 PR #382 的完整三层模型，需要编译环境逐项适配。

## 验证状态

- 静态检查：**已完成且全绿**。脚本校验了 schema 合法（44 tools）、两个工具齐全、所有结构体字面量均补齐 `close_session` 字段、companion/listener/broker 关键符号与分流齐备。6 个文件 diff 已逐一人工审查，`not_resumable_report` 签名、`cancel_task_by_id` 参数均核对无误。
- 文本格式：UTF-8 无 BOM + CRLF，`D:\work\.codex\scripts\text-format.ps1` 检查通过。
- 编译 / 单测：**未执行**。本机没有 Rust 工具链（`cargo` 不在 PATH），按项目规范由用户手工验证。

### 建议的验证步骤（在有 Rust 环境时）

```bash
cd src-tauri
cargo check --all-targets --features test-utils
cargo test --features test-utils --lib acp::delegation
```

### 建议补充的测试（未写入，避免交付无法验证的代码）

- `build_continue_prompt` 对 message 做 trim、原样返回。
- 内存门：已完成的、caller 自有 task + `message=Some` → Proceed；`message=None` → not_resumable。
- 幂等：同一 continuation_id 两次调用，第二次命中 ledger、不触发第二次 spawn。
- close：running 的 task 被取消并标记 released；随后 continue 被拒。
- serde：省略新字段时解码为默认值（向后兼容）。

## 部署注意

companion（`codeg-mcp.exe`）与 listener 在同一制品中、协议无版本字段：改了 transport 结构后必须同时重新编译二者，不能让旧 companion 配新 listener。
