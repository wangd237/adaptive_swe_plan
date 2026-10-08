# A-SWE Runtime Implementation Plan

本仓库保存 **Adaptive Agent Runtime for Software Engineering** 的实施方案与架构审计。

## Documentation Map

- [Master Plan](plan/master-plan.md) — 项目定位、总体流程、工程边界、实施阶段与包装重点。
- [Task Understanding & Planning Spec](specs/01-task-planning.md) — TaskSpec、TaskContract、ValidatedWorkPlan、Acceptance Compiler，以及 shared EvidenceRef / NodeHandoff / EvidenceStore contracts。
- [Capability / Provider / Team Spec](specs/02-capability-provider-dag.md) — WorkspaceAccess、Capability、Provider、Team、BackendInventory / Preparation、Agent/Skill/MCP contracts。
- [Execution Runtime Spec](specs/03-execution-runtime.md) — Workspace、Repository state、Backend execution result、TaskNode/TaskDAG、Scheduler、Retry/Repair/Reverify、TaskResult。
- [Evidence / Evaluation Spec](specs/04-evidence-evaluation.md) — Trace semantics、ContractVerdict、Task Evaluation、Review/Evaluation presentation。
- [DeerFlow Source Audit](audits/deerflow-source-audit.md) — Phase 0 源码审计与架构冻结依据。
- [PoC Matrix](tests/poc-matrix.md) — Architecture conformance / Go-No-Go PoC。

## Schema Ownership Map

| Authoritative Spec | Owns |
|---|---|
| `specs/01-task-planning.md` | RepositoryProfile / TaskSpec / Constraint & Contract / WorkKind / WorkPlanProposal / ValidatedWorkPlan / VerificationCommand / WorkspaceRevision / ExecutionCompleteness / EvidenceRef / TaskEvidenceRef / ReceiptRef / NodeHandoff / ExecutionEvidenceStore |
| `specs/02-capability-provider-dag.md` | WorkspaceAccess / CapabilitySpec / CapabilityBinding / ToolEffect / Provider & Team contracts / BackendInventorySnapshot / NodeExecutionPreparation |
| `specs/03-execution-runtime.md` | WorkspaceSession / RepositoryStateDigest / NodeExecutionResult / NodeAcceptanceResult / TaskNode / TaskDAG / NodeExecutionPolicy / NodeRuntimeState / dispatch tickets / Repair contracts / TaskResult |
| `specs/04-evidence-evaluation.md` | ContractLeafVerdict / ContractVerdict / Evaluation & Trace semantics |

Rules:

1. 一个 schema class 在 active specs 中只能有一个完整定义；
2. 其他文档只能链接/引用，不复制一份“略有不同”的 class；
3. `archive/` 永远不是 implementation authority；
4. 若 ownership 必须迁移，需在一次变更中同步引用、PoC 与 README map；
5. Design Freeze 后新增 core schema 需要说明为什么现有 owner 无法表达该 contract。

## Source-of-Truth Rule

拆分后不再维护根目录单体方案。一个概念的完整 authoritative definition 只保存在对应 Spec；Master Plan 负责路线、阶段与索引。

原单体方案保存在：

- [archive/adaptive_swe_runtime_implementation_plan_v2_monolith.md](archive/adaptive_swe_runtime_implementation_plan_v2_monolith.md)

该文件仅用于历史追溯，不再更新。
