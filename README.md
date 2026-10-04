# 工业过程数字孪生与异常根因推理服务

纯后端 Node.js + TypeScript + Fastify 服务，默认无需 Docker、数据库或消息中间件即可运行。

## 启动

```bash
npm install
npm run dev
```

默认端口：`6301`。健康检查：`GET /health`。

## 业务闭环

- 领域状态模型与事件输入
- 可回放的时间序列/仿真流程
- 约束校验与风险门禁
- 方案版本、审计轨迹与异常处置
- 所有状态暂存于进程内，便于本机演示；可扩展为数据库、事件总线和实时流处理架构

## 增强业务闭环

- 配方版本、工艺窗口风险评估与审批
- What-if 工况推演，输出吞吐量、质量、能耗和风险
- 受控闭环控制命令，受操作窗口和高严重度异常门禁约束
- 维修窗口、备件准备、下游影响与 LOTO 锁定确认
- 综合过程健康评分

扩展接口：`/api/recipes`、`/api/control/commands`、`/api/maintenance/windows`、`/api/process/health-score`。

## 异常-调查-工单联动

调查假设可转换为关联工单，避免异常、调查、工单彼此独立导致的重复派单或未修复先关闭：

- `POST /api/investigations/:id/hypotheses/:hid/convert`：把假设转换为关联工单（携带 anomalyId/investigationId/hypothesisId）；重复转换幂等返回原工单（`deduplicated:true`），不重复派单。
- 工单状态机：`planned → in_progress → completed`。
  - `POST /api/work-orders/:id/start`：进入处理中（重复开工/已完成返回 409）。
  - `POST /api/work-orders/:id/complete`：进入已完成，必须提交处置证据 `evidence`（跳过开工、重复完成返回 409）。
- `POST /api/anomalies/:id/resolve`：异常存在关联工单时，必须所有关联工单均为 `completed` 且已提交处置证据，否则 409 阻止关闭；无关联工单时保留直接关闭的旧行为。
- 保留旧行为：`POST /api/work-orders` 独立建工单（`anomalyId:null`，不阻塞任何异常）、调查证据继续通过 `/api/investigations/:id/evidence` 追加。
- 查询：`GET /api/work-orders`（可按 status/assetId/anomalyId 过滤）、`GET /api/anomalies/:id/work-orders`。
- 异常、调查、假设、工单不存在均返回 404；审计（`GET /api/audit`）记录关联四元组、`work-order.link`、每次状态变迁（from→to）、处置证据以及关闭拦截（`anomaly.resolve.blocked`），可完整还原关联与状态变化。
