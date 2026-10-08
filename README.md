# A-SWE Runtime Implementation Plan

本仓库保存 **Adaptive Agent Runtime for Software Engineering** 的实施方案与架构审计。

## Documentation Map

- [Master Plan](plan/master-plan.md) — 项目定位、总体流程、工程边界、实施阶段与包装重点。
- [Task Understanding & Planning Spec](specs/01-task-planning.md) — TaskSpec、TaskContract、Constraint Compiler、Planner、Acceptance Compiler。
- [Capability / Provider / Team Spec](specs/02-capability-provider-dag.md) — Capability、Provider、Team、Agent/Skill/MCP contracts。
- [Execution Runtime Spec](specs/03-execution-runtime.md) — Workspace、Backend、TaskDAG、Scheduler、Handoff、Retry/Repair/Reverify。
- [Evidence / Evaluation Spec](specs/04-evidence-evaluation.md) — Trace、EvidenceStore、Evaluation、Review。
- [DeerFlow Source Audit](audits/deerflow-source-audit.md) — Phase 0 源码审计与架构冻结依据。
- [PoC Matrix](tests/poc-matrix.md) — Architecture conformance / Go-No-Go PoC。

## Source-of-Truth Rule

拆分后不再维护根目录单体方案。一个概念的完整 authoritative definition 只保存在对应 Spec；Master Plan 负责路线、阶段与索引。

原单体方案保存在：

- [archive/adaptive_swe_runtime_implementation_plan_v2_monolith.md](archive/adaptive_swe_runtime_implementation_plan_v2_monolith.md)

该文件仅用于历史追溯，不再更新。
