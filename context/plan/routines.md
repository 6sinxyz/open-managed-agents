# OpenMA Routines 功能规划

## Summary

目标产物：`/Users/alexa/workspace/open-managed-agents/context/plan/routines.md`。

本功能为 Open Managed Agents 增加类似 Claude Routines 的一等产品层：用户可以保存一组可重复执行的 agent 配置，包括 agent、environment、prompt、触发器和连接器上下文，并通过定时、API、GitHub 事件或手动运行创建可追踪的 session。

现有 `schedule` / `cancel_schedule` / `list_schedules` 保持不变，它仍是“单个 session 自唤醒”能力；新增 Routines 是“跨 session 的可保存自动化定义 + run history + Console 管理”。

## Key Changes

### 数据模型

新增 tenant-scoped store，优先实现为 `@open-managed-agents/routines-store`，同时支持 Cloudflare D1 和 Node SQL 后端。

核心表：

- `routine_definitions`
  - `id`
  - `tenant_id`
  - `name`
  - `description`
  - `enabled`
  - `agent_id`
  - `environment_id`
  - `prompt_template`
  - `trigger_type`: `schedule | api | github_event`
  - `trigger_config_json`
  - `max_concurrent_runs`
  - `created_by`
  - `created_at`
  - `updated_at`
- `routine_runs`
  - `id`
  - `tenant_id`
  - `routine_id`
  - `status`: `queued | running | completed | failed | cancelled`
  - `trigger_source`
  - `trigger_payload_json`
  - `idempotency_key`
  - `session_id`
  - `error_message`
  - `created_at`
  - `started_at`
  - `completed_at`
- `routine_trigger_tokens`
  - `id`
  - `tenant_id`
  - `routine_id`
  - `token_hash`
  - `created_at`
  - `revoked_at`

默认策略：

- `max_concurrent_runs = 1`
- API trigger token 只存 hash，不回显明文
- webhook/API payload 入库限制 64KB，超限返回 413 或保存摘要引用
- `idempotency_key` 用于 GitHub delivery id、API `Idempotency-Key` 和 schedule tick 防重复

### Public API

新增 `/v1/routines` API，沿用现有 tenant/auth/API key 机制。

- `POST /v1/routines`
  - 创建 routine definition
- `GET /v1/routines`
  - 列表，支持 `enabled`、`trigger_type`、`agent_id` 过滤
- `GET /v1/routines/:id`
  - 获取详情
- `PATCH /v1/routines/:id`
  - 更新 name、description、enabled、prompt、trigger、agent/environment、concurrency
- `DELETE /v1/routines/:id`
  - 软删除或禁用；已有 run history 保留
- `POST /v1/routines/:id/run`
  - 已认证手动运行
- `GET /v1/routines/:id/runs`
  - 查看运行历史
- `GET /v1/routine-runs/:run_id`
  - 查看单次运行、session id、错误信息
- `POST /v1/routines/:id/trigger`
  - 外部 API 触发，使用 `Authorization: Bearer <routine_trigger_token>`

GitHub event trigger 复用现有 integrations/webhook 管线，不新开 webhook 根入口；由 GitHub provider 在匹配 routine trigger config 后调用 RoutineRunService。

### Runtime 行为

Routine 执行流程：

1. trigger 进入 RoutineRunService。
2. 检查 routine 是否 enabled、tenant 是否匹配、并发是否超限、idempotency 是否已处理。
3. 创建 `routine_runs` row，状态为 `queued`。
4. 创建新 session，使用 routine 的 `agent_id` 和 `environment_id`。
5. 渲染 `prompt_template`，把 trigger payload 注入为首条 `user.message` 的 metadata。
6. session 创建成功后写入 `session_id`，状态进入 `running`。
7. session 完成、失败或取消后同步更新 run 状态。
8. Console 和 API 均以 `routine_runs` 为事实源展示历史。

定时触发：

- 新增 scheduler job：`routine-tick`
- 默认 cron：`* * * * *`
- 每次 tick 扫描 due schedule routines
- 支持基础 cron 表达式和 timezone
- 每个 tick 有 per-tenant/per-routine 上限，避免单租户耗尽 runtime budget

GitHub 触发：

- 支持 `pull_request.opened`
- 支持 `pull_request.synchronize`
- 支持 `issues.opened`
- 支持按 repo、branch、label、event type 过滤
- 使用 GitHub delivery id 做 idempotency key

API 触发：

- 支持任意 JSON body
- 支持 `Idempotency-Key`
- payload 可在 prompt template 中引用

### Console UI

新增 Routines 一级或二级入口，完整 UI 覆盖创建、编辑、运行和排障。

页面：

- Routines List
  - 展示 name、enabled、trigger、agent、environment、last run、status
  - 支持 enable/disable、manual run、进入详情
  - 空态说明“创建第一个 routine”
- Routine Create/Edit
  - 基础信息：name、description、enabled
  - Agent/Environment 选择器
  - Trigger wizard：Schedule、API、GitHub Event
  - Prompt template 编辑器
  - Concurrency 设置
  - API trigger token 创建/撤销
- Routine Detail
  - 配置摘要
  - 最近运行列表
  - 手动运行按钮
  - 最近错误提示
- Run Detail
  - trigger payload 摘要
  - rendered prompt
  - session 链接
  - 状态时间线
  - 错误信息

UI 复用现有 Console 组件和 EvalRuns 页面模式；不要引入新的设计系统。

## Test Plan

### Unit Tests

- routine store CRUD、tenant isolation、soft delete/disable
- trigger token hash 校验、撤销后拒绝
- idempotency key 重放不重复创建 run
- max concurrency 超限时排队或拒绝，默认拒绝并记录 skipped/failed 原因
- prompt template 渲染 payload 字段

### Integration Tests

- `POST /v1/routines` 创建后能列表/详情读取
- authenticated manual run 创建 session 并写入首条 user message
- schedule routine 在 `routine-tick` 后创建 run 和 session
- API trigger 使用 bearer token 创建 run
- GitHub webhook 事件匹配 routine 后创建 run
- disabled routine 不触发
- 删除 agent/environment 时阻止或提示 active routine 依赖

### Console / Browser Tests

- 创建 schedule routine 的完整表单流
- 创建 API routine 并展示一次性 token
- GitHub trigger 配置过滤条件
- Run history 展示 running/completed/failed
- 从 Run Detail 跳转到 session
- 空态、加载态、错误态、权限不足态

### Deployment Verification

- Cloudflare deploy 后检查 `/health`
- 检查 scheduler 注册包含 `routine-tick`
- 创建测试 schedule routine，等待一次 tick，确认 run/session 产生
- API trigger smoke test
- Console browser dogfood 截图留档

## Assumptions

- 本次规划目标是 Claude 对标版，不是最小 MVP；但实现可以拆成多 PR。
- 第一阶段仍要求完整 UI 规划，不降级为 API-only。
- Routine 不替代现有 session-level `schedule` 工具；两者共存。
- Routine 使用现有 Agent、Environment、Session、Integration、Scheduler 能力，不新增独立 agent runtime。
- API key、model card、vault、connector 权限复用现有 tenant 权限模型。
- Feature flag 使用 `ROUTINES_ENABLED`，默认关闭；部署验证环境可打开。
- 文档写入目标为远程 OpenMA fork 的 `context/plan/routines.md`，实现阶段如目录不存在则创建。
