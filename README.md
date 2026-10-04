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

## 异常—调查—工单闭环

异常交接必须先调查、用证据验证假设，再决定派单；工单与异常通过假设建立关联，避免重复派单和未完工关闭异常。

- `POST /api/investigations/:id/hypotheses/:hypothesisId/convert-work-order`：把调查假设转换为与异常关联的工单。同一假设重复转换为幂等操作，直接返回原工单（审计记 `work-order.convert-replay`）。
- 工单状态机：`planned` → `processing` → `completed`。
  - `POST /api/work-orders/:id/start`：开工；重复开工幂等；已完成工单再开工返回 409。
  - `POST /api/work-orders/:id/complete`：完成**必须提交处置证据** `{"evidence":{"observation":"..."}}`，否则 400；重复完成返回 409。
  - `GET /api/work-orders?status=&anomalyId=`：查询工单。
- `POST /api/anomalies/:id/resolve`：存在关联工单时，必须全部 `completed` 且都带有处置证据，否则 409 并返回待处理工单清单（审计记 `anomaly.resolve-blocked`）；**无关联工单时保留旧行为**，提交 resolution evidence 即可关闭；重复关闭返回 409。
- 保留旧行为：独立建工单 `POST /api/work-orders`（`source: 'manual'`，不挂异常）、调查追加证据 `POST /api/investigations/:id/evidence`、无工单凭证据关闭异常。
- 不存在的异常/调查/假设/工单均返回 404。
- 审计可还原完整链路：`work-order.convert-from-hypothesis`（含 anomalyId/investigationId/hypothesisId 关联）、`work-order.status-change`（from/to）、`anomaly.resolve-blocked`、`anomaly.resolve`。
