# Adaptive Agent Runtime for Software Engineering 实施方案书

> 版本：MVP 面试项目收敛版（DeerFlow 源码审计后更新）  
> 项目简称：**A-SWE Runtime**  
> 核心目标：在较短实施周期内，完成一个可运行、可演示、可解释的自适应软件工程 Agent Runtime。

---

## 1. 项目概述

### 1.1 项目名称

**Adaptive Agent Runtime for Software Engineering（A-SWE Runtime）**

中文暂定名：

**面向软件工程的自适应智能体运行时系统**

### 1.2 项目定位

A-SWE Runtime 面向 Repository-Level Software Engineering Task，目标不是构建一个固定流程的 Coding Agent，也不是重复实现一套完整 Agent 基础设施，而是构建一个能够根据任务特征动态完成以下决策的软件工程智能体运行时：

- 当前任务需要哪些能力；
- 哪些 Agent / Skill / Tool 能够提供这些能力；
- 是否需要 Multi-Agent；
- 应构建怎样的 Agent Topology；
- 任务应如何拆解为可执行 DAG；
- 执行过程是否可观测、可追踪；
- 最终结果是否满足软件工程任务要求。

传统 SWE Agent 常采用固定执行结构，例如：

```text
Task
 ↓
Planner
 ↓
Coder
 ↓
Tester
 ↓
Reviewer
```

该模式的问题在于：

- 简单任务仍需经过完整 Multi-Agent 链路；
- 不同任务使用同一执行拓扑；
- Agent、Skill、Tool 之间缺少统一能力抽象；
- Runtime 很难解释“为什么选择这个 Agent”；
- Multi-Agent 协作可能引入不必要的 Token、时间与协调成本。

A-SWE Runtime 改为：

```text
Software Engineering Task
            │
            ▼
      Task Understanding
            │
            ▼
     Capability Analysis
            │
            ▼
    Capability Resolution
            │
            ▼
      Team Construction
            │
            ▼
        Task DAG
            │
            ▼
    Adaptive Execution
            │
      ┌─────┴─────┐
      ▼           ▼
   Trace      Evaluation
      │           │
      └─────┬─────┘
            ▼
          Result
```

项目当前默认通过 Adapter 对接 DeerFlow Harness，复用其 Subagent Execution、Skills、MCP、Sandbox、Context Management、Tool Receipt 等基础能力；A-SWE Runtime 自身保持独立，核心新增能力集中在任务理解、Capability Resolution、Workspace Runtime、动态团队构建、任务调度、Execution Trace 和 Evaluation。

### 1.3 核心项目命题

本项目一期需要证明的不是“Agent 能否写代码”，而是：

> **同一个 Runtime 面对不同软件工程任务，能够根据任务所需能力、复杂度和风险，生成不同的 Agent 执行结构。**

一句话定位：

> **A task-adaptive SWE agent runtime that dynamically composes capabilities, agents and execution topology.**

---

## 2. 项目建设目标

### 2.1 总体目标

建设一个面向软件工程任务的自适应 Agent Runtime，使系统能够自动完成：

1. 理解用户提交的软件工程任务；
2. 将自然语言任务转换为结构化 `TaskSpec`；
3. 创建并维护任务级 `WorkspaceSession`，保证同一任务内各执行节点共享稳定的软件工程工作区；
4. 识别任务所需 Capability；
5. 从 Agent、Skill、Tool、MCP Tool 中解析 Capability Provider；
6. 根据任务复杂度、风险和能力覆盖情况构建最小可行 Agent Team；
7. 生成 Task DAG，并显式标注节点对共享 Workspace 的 READ / WRITE 行为；
8. 调度 Agent、Skill 与 Tool 执行；
9. 记录 Runtime Decision、节点执行证据与底层 Harness Trace；
10. 对 Patch、测试结果和执行结果进行自动 Evaluation；
11. 输出结构化任务结果与运行指标。

### 2.2 一期建设重点

一期聚焦七个核心能力：

```text
1. Task Analyzer
2. Workspace Runtime
3. Capability Registry / Resolver
4. Dynamic Team Builder
5. Task Scheduler
6. Execution Trace / Observability
7. Evaluation
```

其中 `Workspace Runtime` 不是额外的 Agent，而是负责维护 Repository-Level Task 的共享执行身份与工作区生命周期。

### 2.3 一期明确不做

为了控制交付周期，一期不将以下能力作为必要目标：

- SWE-bench 等正式 Benchmark；
- 大规模 Experience Memory；
- Self-Improving / RL / Bandit；
- 复杂 `P(success | task, team)` 预测模型；
- Knowledge Graph；
- Agent Swarm；
- 大量 MCP Server；
- 多模型 Agent Ranking；
- 自动 Skill 生成；
- 大规模 Web UI 平台化建设；
- 分布式 Workspace / 多租户远程 Sandbox 平台；
- 复杂文件级冲突预测与自动 Merge。

一期重点是：

> **Runtime 核心链路能够真实运行，并能够通过 3～5 个 Demo Case 清晰展示不同任务产生不同执行拓扑，同时保证多个 DAG 节点在同一任务工作区内安全协作。**

---

## 3. 总体系统架构

A-SWE Runtime 采用“控制面 + Workspace Runtime + 执行适配层 + DeerFlow Execution Plane”的设计。

```text
┌──────────────────────────────────────────────────────────────┐
│                       A-SWE Runtime                           │
│                                                              │
│  Task Analyzer                                               │
│  Capability Registry / Resolver                              │
│  Dynamic Team Builder                                        │
│  Task DAG / Scheduler                                        │
│  Execution Trace / Observability                             │
│  Evaluation                                                  │
│                                                              │
├───────────────────────────┬──────────────────────────────────┤
│      Workspace Runtime    │       Runtime Decision Plane     │
│                           │                                  │
│  WorkspaceSession         │  what / who / when               │
│  thread_id / user_id      │                                  │
│  repo / base_ref          │                                  │
│  shared workspace         │                                  │
├───────────────────────────┴──────────────────────────────────┤
│                DeerFlow Execution Adapter                    │
│                                                              │
│     TaskNode + AgentProvider + WorkspaceSession               │
│                         │                                    │
│                         ▼                                    │
│               DeerFlowExecutionBackend                       │
├──────────────────────────────────────────────────────────────┤
│                   DeerFlow Execution Plane                   │
│                                                              │
│  SubagentConfig → SubagentExecutor → SubagentResult          │
│  Skills / Tools / MCP / Middleware / Sandbox / Receipts      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

核心职责边界：

> **A-SWE 决定 what / who / when；DeerFlow 负责 how to execute safely。**

### 3.1 A-SWE Runtime 层

A-SWE Runtime 是项目主体，负责：

- Task Understanding；
- Capability Modeling；
- Capability Provider Resolution；
- Dynamic Team Formation；
- DAG Planning；
- Agent Assignment；
- Workspace-aware Scheduling；
- Retry / Failure Propagation；
- Runtime Decision Trace；
- Evaluation；
- 运行指标采集。

A-SWE 不让 DeerFlow Lead Agent 自主决定任务拓扑。Team 与 DAG 由 A-SWE 控制面显式生成，底层 Harness 只负责执行已确定的节点。

### 3.2 Workspace Runtime

Repository-Level SWE Task 不是若干互相独立的 Agent 调用，而是在同一份代码状态上持续推进的执行过程，因此一期显式引入 `WorkspaceSession`。

建议数据结构：

```python
class WorkspaceSession(BaseModel):
    task_id: str

    # DeerFlow execution identity
    thread_id: str
    user_id: str

    # runtime workspace
    workspace_root: str

    # optional Git repository bound to this workspace
    repository: RepositoryBinding | None = None

    status: str
```

关键约束：

```text
one A-SWE task
      │
      ▼
one WorkspaceSession
      │
      ├── stable user_id
      └── stable thread_id
              │
              ▼
shared DeerFlow thread workspace / sandbox identity
```

Explorer、Coder、Tester、Reviewer 只要在同一 `WorkspaceSession` 中执行，就必须使用相同的 `user_id + thread_id`。

这使：

```text
Explorer discovers code
        ↓
Coder modifies code
        ↓
Tester observes the modification
        ↓
Reviewer reviews the same working tree
```

成为 Runtime 的显式契约，而不是依赖 Agent 输出在 Prompt 中传递代码状态。

Repository / Workspace Bootstrap 的职责边界经 P0-2 审计后固定为：

> **Repository Bootstrap 属于 A-SWE Control Plane，不属于 Agent Task。**

A-SWE 不让 Coder / Explorer 自行决定 Repository 的 clone、checkout 和 baseline。Runtime 必须在第一个 Agent Node 执行前确定性完成 Repository Bootstrap，并冻结 Repository 基线。

建议增加：

```python
class RepositoryBinding(BaseModel):
    source_type: str

    remote_url: str | None = None
    requested_ref: str | None = None

    # immutable runtime authority
    resolved_base_sha: str

    repository_root: str
    default_branch: str | None = None

    clean_at_bootstrap: bool
```

其中 `requested_ref` 可以是：

```text
main
develop
feature/foo
v1.2.0
<commit sha>
```

但 Runtime 真正使用的基线必须是：

```text
resolved_base_sha
```

而不是可漂移的 branch / tag 名称。

一期推荐一个 Task 对应一个独立 Thread Workspace，并直接将 Repository Root 放在：

```text
/mnt/user-data/workspace
```

而不是：

```text
/mnt/user-data/workspace/<repo-name>
```

因为 DeerFlow 的 bash 工具本身会将默认工作目录定位到 thread workspace，这样 Explorer、Coder、Tester、Reviewer 均天然在 Repository Root 执行。

Repository Bootstrap 标准流程：

```text
WorkspaceSession Allocate
        │
        ▼
Prepare Empty Workspace
        │
        ▼
Clone / Materialize Repository
        │
        ▼
Resolve requested_ref
        │
        ▼
Checkout exact resolved_base_sha
        │
        ▼
Verify Repository Clean
        │
        ▼
Capture Baseline Workspace Snapshot
        │
        ▼
WORKSPACE READY
```

一期推荐以 detached HEAD / immutable baseline 方式运行，使 Agent 只修改 Working Tree，不负责 commit / branch / merge / rebase 等 Repository lifecycle operation。

### 3.2.1 Repository Invariant

A-SWE 必须在关键 WRITE Node 后检查 Repository invariant：

```python
class RepositoryInvariant(BaseModel):
    git_repo_exists: bool
    head_sha: str
    head_matches_baseline: bool
```

最重要的不变量：

```text
.git exists
HEAD == resolved_base_sha
```

如果 Agent 擅自 commit、checkout、rebase 等导致 HEAD 偏离 baseline，一期直接将该 Node 判为失败，不做静默自动恢复。

### 3.2.2 Repository Change Evidence

DeerFlow 已提供 filesystem-level `workspace_changes`：

```text
before snapshot
      ↓
execution
      ↓
after snapshot
      ↓
filesystem change result
```

A-SWE 直接复用：

```text
capture_workspace_snapshot
compare_snapshots
```

但 DeerFlow 的 workspace scanner 明确忽略 `.git`，因此它不是 Git-aware Patch。

一期采用双证据模型：

```text
Repository Result
      │
      ├── Git ChangeSet
      │     └── authoritative repository patch
      │
      └── Workspace ChangeSet
            └── filesystem execution evidence
```

建议：

```python
class RepositoryChangeSet(BaseModel):
    base_sha: str

    head_sha: str
    head_matches_baseline: bool

    tracked_diff: str
    changed_files: list[str]
    untracked_files: list[str]

    dirty: bool
```

注意：`git diff HEAD` 不包含 untracked files，因此最终 Patch 提取不能仅依赖该命令。

正式输出完整 Patch 时，可使用 temporary Git index 生成不污染真实 index 的 patch：

```text
temporary GIT_INDEX_FILE
        ↓
git read-tree <base_sha>
        ↓
git add -A
        ↓
git diff --cached --binary <base_sha>
```

Baseline Workspace Snapshot 必须在 clone / checkout 完成之后采集，否则整个 Repository 会被误判为 task-created files。

Task-level snapshot 用于最终 ChangeSet；WRITE Node 可选用 before / after snapshot 做 Node → file change attribution；纯 READ Node 一期不强制全量 snapshot。

### 3.3 DeerFlow Execution Adapter

Adapter 不再暴露：

```python
create_agent()
run_agent()
load_skill()
execute_tool()
run_in_sandbox()
```

这类过度贴近 Harness 内部组件的接口。

一期收敛为节点级执行接口：

```python
class ExecutionBackend(Protocol):
    async def execute_node(
        self,
        node: TaskNode,
        provider: AgentProvider,
        workspace: WorkspaceSession,
    ) -> NodeExecutionResult:
        ...

    async def cancel_node(
        self,
        execution_id: str,
    ) -> None:
        ...

    async def check_acceptance(
        self,
        result: NodeExecutionResult,
        acceptance_criteria: list[str],
        workspace: WorkspaceSession,
    ) -> NodeAcceptanceResult:
        ...
```

因此 A-SWE 核心模块只理解：

```text
TaskNode
AgentProvider
WorkspaceSession
NodeExecutionResult
NodeAcceptanceResult
```

而不知道 DeerFlow 内部如何加载 Skill、Tool、MCP、Middleware 或 Sandbox。

### 3.4 DeerFlowExecutionBackend

一期推荐将 DeerFlow 的 `SubagentExecutor` 作为 DAG Worker Primitive。

执行链路：

```text
A-SWE Scheduler
      │
      ▼
TaskNode
      │
      ▼
Capability / Agent Provider
      │
      ▼
DeerFlowExecutionBackend
      │
      ├── resolve SubagentConfig
      ├── resolve tools / skills / MCP
      ├── bind WorkspaceSession.thread_id
      ├── bind WorkspaceSession.user_id
      └── use shared execution capacity
      │
      ▼
SubagentExecutor
      │
      ▼
SubagentResult
      │
      ▼
NodeExecutionResult
```

`SubagentExecutor` 已能接收：

- `SubagentConfig`；
- tools；
- app config；
- thread data；
- sandbox state；
- `thread_id`；
- `user_id`；
- trace id；
- shared execution capacity；
- acceptance criteria。

Adapter 可以持有一个进程级 `SubagentRuntime` / shared execution capacity，并将同一容量控制器传入不同节点执行，避免 A-SWE 自己重新实现底层并发 admission control。

### 3.5 NodeExecutionResult

A-SWE 不直接暴露 DeerFlow `SubagentResult`，而映射成自身稳定 Schema。

建议至少保留：

```python
class NodeExecutionResult(BaseModel):
    execution_id: str
    node_id: str
    status: str
    result: str | None
    error: str | None

    stop_reason: str | None

    started_at: datetime | None
    completed_at: datetime | None

    token_usage: list[dict]
    tool_receipts: list[dict] | None
    bash_executions: list[dict] | None

    backend_trace_id: str | None
```

其中 `stop_reason` 必须与 `status` 分离，例如：

```text
status = completed
stop_reason = turn_capped
```

表示任务产生了可用结果，但执行过程触发了 Runtime Guardrail，不应被误认为 clean completion。

### 3.6 DeerFlow 集成基线与升级策略

本轮源码审计冻结基线：

```text
bytedance/deer-flow
commit: c0895d295bba34f6e95188fca380f555dabed891
```

重点验证路径：

```text
deerflow/subagents/config.py
deerflow/subagents/executor.py
deerflow/subagents/runtime.py
deerflow/subagents/capacity.py
deerflow/sandbox/middleware.py
deerflow/sandbox/lease.py
deerflow/subagents/acceptance_checks.py
```

由于 `SubagentExecutor` 属于较低层 Harness API，A-SWE 必须：

1. 将所有 DeerFlow-specific import 限制在 `integrations/deerflow/`；
2. 固定并记录兼容的 DeerFlow commit / version；
3. 为 Adapter 建立独立 integration tests；
4. 升级 DeerFlow 时优先重新执行 P0.5 PoC，而不是直接假设接口兼容。

### 3.7 DeerFlow Harness 复用范围

底层 Harness 主要复用：

- Subagent 执行；
- Tool Calling；
- Skill Discovery / Activation；
- MCP Tool Routing；
- Sandbox；
- Sandbox Lease；
- Context / Thread Data；
- Tool Receipt；
- Token / Turn / Loop Guard；
- Langfuse / LangSmith 等底层 Trace 能力。

实施原则：

> 能通过 Execution Adapter 使用的基础设施能力，不在 A-SWE 中重复实现。

---

## 4. Task Understanding 与 Planning Pipeline

### 4.1 模块目标

A-SWE 不让 Task Analyzer 直接产生最终 Capability / Team / DAG。

一期将任务规划拆成：

```text
Task Request
      ↓
Repository Profile
      ↓
Task Understanding
      ↓
Planning Context Acquisition
      ↓
Semantic Planning
      ↓
Plan Validation / Normalization
      ↓
Capability Resolution
      ↓
Provider / Team Selection
      ↓
DAG Materialization
```

核心原则：

> **The planner proposes work packages; the runtime compiles and governs execution.**

其中：

```text
LLM
→ 理解语义、提出工作拆解

Runtime
→ 验证、修正、补全、绑定 Capability / Provider / Workspace Policy，并生成最终可执行 DAG
```

因此：

> **LLM proposes semantics. Runtime owns execution semantics.**

### 4.2 RepositoryProfile

Planner 不应只根据用户一句自然语言需求凭空生成 Repository-Level Plan。

Repository Bootstrap 完成后，Runtime 先产生廉价、确定性的 `RepositoryProfile`。

建议：

```python
class RepositoryProfile(BaseModel):
    base_sha: str

    tracked_file_count: int
    top_level_tree: list[str]

    languages: dict[str, int]

    manifests: list[str]
    test_configs: list[str]
    build_configs: list[str]
    ci_configs: list[str]

    guidance_files: list[str]

    test_framework_hints: list[str]
    build_system_hints: list[str]

    task_anchor_matches: list[AnchorMatch]

    truncated: bool
```

Basic Profile 每个 Repository-Level Task 都可以确定性采集：

```text
git metadata
tracked file inventory
shallow tree
language distribution
manifest files
test/build configuration
CI configuration
guidance file inventory
```

一期不做：

```text
full repository embedding
all-source-code LLM ingestion
repository-wide semantic index requirement
```

### 4.3 Targeted Repository Profiling

根据用户任务中的显式 anchor，再做廉价定向探测。

例如任务包含：

```text
UserService.login
/database/session
specific error string
config key
API endpoint
```

Runtime 可以使用 deterministic search 获取：

```text
exact path match
symbol/string match
manifest match
test-path match
```

目标是尽量减少 LLM Recon Probe 的必要性。

### 4.4 TaskSpec

Task Analyzer 输入：

```text
TaskRequest
+
Basic RepositoryProfile
+
Targeted Profile Evidence
```

其模型调用不直接依赖 DeerFlow 私有模型工厂，而通过 A-SWE 的 `ReasoningBackend` 完成结构化推理。

输出：

```text
TaskSpec
+
TaskContractDraft
```

其中 `TaskSpec` 表示任务理解结果；`TaskContractDraft` 保存带来源与权威级别的候选约束。

示例：

```python
class TaskSpec(BaseModel):
    task_type: TaskType
    description: str

    domains: list[str]
    repository_level: bool

    complexity: Complexity
    risk: RiskLevel

    testing_required: bool
    review_required: bool

    scope_hints: list[str]
    capability_hints: list[str]

    planning_uncertainties: list[str]
```

其中：

> `capability_hints` 只是 Task-Level semantic hints，不是最终 authoritative `required_capabilities`。

最终 Required Capability 由 Validated Work Items 在 Node-Level 产生。

### 4.4.1 ReasoningBackend

A-SWE Core 不直接调用：

```python
deerflow.models.create_chat_model
```

也不自行维护第二套 provider credential / model configuration。

建议定义：

```python
class ReasoningBackend(Protocol):
    async def generate_structured(
        self,
        *,
        purpose: str,
        system_prompt: str,
        data_context: str,
        response_schema: dict,
        model_role: str | None = None,
    ) -> StructuredReasoningResult:
        ...
```

DeerFlow integration 实现优先复用公开的：

```text
deerflow_extension_api.ModelInvoker
```

其已经提供：

- operator-granted logical model roles；
- timeout / concurrency / input-output bounds；
- JSON Schema validation；
- normalized provider errors；
- usage metadata；
- host-owned provider credentials。

建议 logical roles：

```text
task_analyzer → analyzer
semantic_planner → planner
```

A-SWE Core 只认识 logical role，不绑定具体模型厂商。

### 4.4.2 TaskContract Draft

Task Analyzer 不再把所有信息平铺进 `TaskSpec` 并视为同等权威。

新增：

```python
class ConstraintSource(str, Enum):
    RUNTIME_POLICY = "runtime_policy"
    USER_EXPLICIT = "user_explicit"
    REPOSITORY_GUIDANCE = "repository_guidance"
    ANALYZER_INFERRED = "analyzer_inferred"
    RECON_INFERRED = "recon_inferred"

class ConstraintDraft(BaseModel):
    kind: str
    value: object

    source: ConstraintSource
    hard: bool

    evidence: str | None = None
    evidence_ref: str | None = None

class TaskContractDraft(BaseModel):
    deliverables: list[ConstraintDraft]
    constraints: list[ConstraintDraft]
    forbidden_actions: list[ConstraintDraft]
    verification_requirements: list[ConstraintDraft]
```

注意：

> Draft 只是 Analyzer Proposal，不是最终 authoritative TaskContract。

它必须经过后续 `ConstraintCompiler` 做 provenance validation、conflict resolution 与 policy merge。

### 4.5 Task Analyzer Rule Validation

LLM 输出不能直接作为最终调度依据。

例如：

```text
如果 task_type == bug_fix
→ capability_hints 至少包含 code_modification

如果用户明确要求 regression test
→ testing_required = true

如果 risk == high
→ review_required = true

如果目标文件 / symbol 在 Profile 中没有解析到
→ 增加 planning_uncertainty

如果用户明确限定文件 / 模块
→ 写入 scope_hints
```

Task Analyzer 的 Rule Validation 只修正 TaskSpec，不创建 Agent 和 DAG。

### 4.6 Planning Context Gate

Runtime 根据 TaskSpec 与 RepositoryProfile 判断 Planner 是否已有足够 Repository Context。

```text
Basic Profile
      +
TaskSpec
      +
Targeted Evidence
      ↓
Planning Context Gate
      /          \
 enough        insufficient
   │               │
   │               ▼
   │        Read-only Recon Probe
   │               │
   └───────┬───────┘
           ▼
     PlanningContext
```

一期可以使用确定性 trigger，而不是完全依赖模型自报 confidence。

例如：

```text
repository_level == true
AND
task_type in {bug_fix, refactor, architecture_change}
AND
relevant target / root cause area unresolved
→ Recon Probe

task explicitly names exact file
AND
operation is local and low-risk
→ skip Recon Probe
```

### 4.7 Read-only Reconnaissance Probe

Recon Probe 属于：

> **Planning Infrastructure**

而不是最终 Execution Team 的业务节点。

它使用同一个 WorkspaceSession，但必须严格只读。

一期建议只暴露 DeerFlow read-only tools：

```text
ls
glob
grep
read_file
```

禁止：

```text
bash
write_file
str_replace
task
MCP write tools
```

DeerFlow 当前工具分组已经明确区分：

```text
ls / read_file / glob / grep → file:read
write_file / str_replace    → file:write
bash                         → bash
```

Recon Probe 建议输出：

```python
class ReconFinding(BaseModel):
    claim: str
    path: str | None
    line_start: int | None
    line_end: int | None

    confidence: Literal["observed", "inferred"]

class ReconReport(BaseModel):
    findings: list[ReconFinding]
    likely_areas: list[str]
    unresolved_questions: list[str]
```

其中 path / line range 由 Runtime 做基础可验证性检查。

### 4.8 TaskContract / Constraint Compiler Boundary

TaskSpec 解决：

> **系统如何理解任务。**

TaskContract 解决：

> **系统最终必须满足什么、禁止什么、哪些条件具有何种 authority。**

编译链：

```text
TaskRequest
+
Runtime Policy
+
Repository Guidance
+
TaskContractDraft
      ↓
ConstraintCompiler
      ↓
Compiled TaskContract
      ↓
SemanticPlanner / PlanValidator
```

核心原则：

> **Inference can inform planning, but only compiled authority can constrain execution.**

ConstraintCompiler 的具体规则在 P0-4 审计中继续冻结。

### 4.9 Repository Evidence Trust Boundary

Repository 内容、Recon 输出和 predecessor Agent 报告都是：

> **Untrusted Runtime Data**

不能获得 Framework / System authority。

信任层级建议：

```text
Runtime Safety / Policy
        ↓
Explicit User Requirement
        ↓
Repository Guidance
        ↓
Repository Evidence
        ↓
Agent Self-Report
```

Repository 中的：

```text
README.md
AGENTS.md
CONTRIBUTING.md
source comments
test data
```

可以提供工程上下文，但不能覆盖 Runtime Policy。

DeerFlow 自身已经采用同类 Prompt Trust 原则：

```text
framework authority → system channel
user/model-influenced text → sanitized HumanMessage data channel
```

A-SWE 必须沿用这一边界。

### 4.10 SemanticPlanner

SemanticPlanner 输入：

```text
TaskSpec
Compiled TaskContract
RepositoryProfile
ReconReport（如果存在）
Capability Catalog（只提供语义能力，不提供 Agent roster）
```

其中：

- `TaskSpec` 提供 task classification / risk / scope inference；
- `TaskContract` 提供 authoritative deliverables / constraints / forbidden actions / verification obligations；
- Repository / Recon 信息属于 untrusted evidence，不可提升为 Runtime Policy。

Planner 不应该看到：

```text
Explorer
Coder
Tester
Reviewer
```

避免 Planner 为了已有 Agent 反向制造 workflow。

Planner 只回答：

> 为完成任务，需要哪些 bounded work packages？

### 4.11 WorkPlanProposal

LLM 只输出 proposal，不直接输出可执行 `TaskDAG`。

```python
class WorkItemProposal(BaseModel):
    id: str
    objective: str

    capability_hints: list[str]
    depends_on: list[str]

    # hint only; runtime owns final side effect
    effect_hint: Literal["read", "write"] | None = None

    acceptance_intent: list[str] = []

class WorkPlanProposal(BaseModel):
    items: list[WorkItemProposal]
    rationale: str
```

### 4.12 Node Boundary Policy

DAG Node 是：

> **bounded work package**

而不是 Todo item / reasoning step。

一期拆分原则：

| 情况 | 是否拆成独立 Node |
|---|---:|
| 可以真正并行 | 是 |
| 需要不同 specialist provider | 是 |
| READ → WRITE side-effect boundary | 候选边界，不强制 |
| WRITE → verification boundary | 是 |
| mandatory Review gate | 是 |
| 有独立 acceptance condition | 是 |
| 强依赖且需要同一上下文连续推理 | 否 |
| 同一 Provider 连续完成更便宜 | 否 |
| 拆分会重复 Repository discovery | 否 |

例如：

```text
Locate DB code
+
Understand lifecycle
+
Identify root cause
```

一期更倾向合并为：

```text
Diagnose DB connection leak
```

而不是创建三个 Agent Node。

READ → WRITE 也不是绝对拆分边界。若：

- 语义强依赖；
- 同一 Provider 能覆盖 diagnosis + modification；
- 拆分会导致重复 Repository discovery；
- 没有明显 parallel / specialist benefit；

则可以编译成一个：

```text
Diagnose and Implement
workspace_access = WRITE
```

代价是该 Node 在整个执行期间独占 Workspace。

因此 NodeBoundaryPolicy 的目标不是“尽量多拆”，而是：

> **在 specialization / parallelism / verification benefit 与 handoff / duplicate discovery / coordination cost 之间选择最小合理 work package。**

### 4.13 Two-Stage Plan Compiler

Plan validation 分为两道关：

```text
WorkPlanProposal
      ↓
SemanticPlanValidator
      ↓
ValidatedWorkPlan
      ↓
Capability / Provider Resolution
      ↓
ExecutionPlanValidator
      ↓
DAG Materializer
      ↓
Executable TaskDAG
```

#### SemanticPlanValidator

Provider 选择前即可判断：

- Schema correctness；
- ID uniqueness；
- dependency existence；
- self dependency；
- DAG acyclic；
- capability vocabulary；
- user / task obligation coverage；
- node count / plan budget；
- NodeBoundaryPolicy；
- missing verification / review gates。

#### ExecutionPlanValidator

Provider Assignment 后判断：

- Provider 是否覆盖 required capabilities；
- required tools 是否在 Provider static contract 中；
- required skills 是否可用；
- Sandbox mode 是否支持所需 execution；
- Node tool allowlist 是否满足 least privilege；
- workspace access 是否与 Capability side effect 一致；
- acceptance criteria 是否可编译；
- Runtime budget 是否可满足。

因此：

> **semantic plan valid ≠ executable under the current deployment**

### 4.14 PlanValidator / PlanNormalizer

LLM Proposal 必须经过 deterministic validation。

规则分三类。

#### Hard Reject

不能安全自动修复：

```text
cycle
self dependency
dependency references nonexistent item
unknown / impossible capability
semantic contradiction
node count exceeds hard limit
unsatisfied hard user constraint
```

结果：

```text
PLAN_INVALID
```

进入 bounded replan 或直接失败。

#### Monotonic Safety Repair

可以安全加强：

```text
code_modification present
AND effect_hint == READ
→ upgrade WRITE

testing_required == true
AND no verification work item
→ append verification gate

risk == high
AND no review work item
→ append review gate

unordered workspace-conflicting nodes
→ add deterministic ordering edge
```

规则只能增强 safety / completeness，不能静默删除用户需求，也不能重写核心任务目标。

#### Warning / Optimization

例如：

```text
多个强依赖 READ items 可以合并
重复 Repository discovery
过度细粒度 decomposition
same-provider handoff overhead
```

一期先记录 Trace Warning；只有语义单调、安全的 normalization 才自动应用。

### 4.15 Acceptance Compilation

Planner 的 `acceptance_intent` 是语义意图，不直接成为 DeerFlow canonical acceptance criteria。

```text
Acceptance Intent
      ↓
Acceptance Compiler
      ↓
Canonical Acceptance Criteria
```

例如：

```text
"regression tests should pass"
      ↓
tests_passed:<resolved-test-command>
```

无法确定性编译的 criterion：

```text
→ UNVERIFIED / reviewer-level condition
```

不把 “looks correct” 之类自由文本伪装成 deterministic acceptance。

### 4.16 Capability 从 Task-Level 下沉到 WorkItem-Level

authoritative capability flow：

```text
ValidatedWorkPlan
      ↓
WorkItem Required Capabilities
      ↓
Capability Resolver
```

TaskSpec 中只有 `capability_hints`。

Task-level capability set：

```text
Union(all validated work-item capabilities)
```

### 4.17 Handoff Contract

DAG dependency 不只表示 control order，还表示 data handoff。

一期不强求把 Agent 自由文本完全解析成复杂语义对象；优先使用：

```python
class NodeHandoff(BaseModel):
    source_node_id: str

    report: str

    evidence_paths: list[str]
    changed_paths: list[str]

    acceptance_summary: dict | None
    receipt_ids: list[str]
```

其中：

- `report` 必须 bounded；
- `changed_paths` 优先来自 deterministic workspace/git evidence；
- `receipt_ids` 来自 DeerFlow Tool Receipt；
- Handoff 仍然是 model report，不是事实权威。

后续若确有价值，再扩展结构化 findings / unresolved_questions。

### 4.18 Planning Replan Boundary

一期区分 planning-time replan 与 execution-time replan。

#### Planning-time

尚未执行任何业务 Node，没有 Workspace mutation：

```text
invalid proposal
→ validator diagnostics
→ bounded planner retry
```

#### Execution-time

只允许：

```text
READ Node returns PLAN_INVALIDATED
AND no WRITE has executed
AND replan budget remains
→ bounded replan
```

一旦 Workspace 已经发生业务 WRITE：

```text
arbitrary DAG replan = disabled in MVP
```

因为这时需要定义 rollback、result reuse、dirty-state semantics。

一期推荐：

```text
planning_attempts <= 2
execution_replans <= 1
execution replan only before first WRITE
```

不实现无界动态 DAG spawning。

---

## 5. Capability Registry：能力注册中心

### 5.1 核心设计原则

A-SWE Runtime 将 Capability 作为 Runtime 的语义调度单位。

Capability 表示：

> **一个 bounded work package 为完成目标所需要的能力。**

例如：

```text
repo_exploration
code_search
bug_diagnosis
python_debugging
database_analysis
code_modification
test_generation
regression_testing
code_review
```

Agent、Skill 和 Tool 是 Capability Provider。

### 5.2 Capability 与 Capability Provider

```text
                     Capability
                         │
                         ▼
                Capability Provider
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
        Agent           Skill           Tool
                                         │
                              ┌──────────┴──────────┐
                              ▼                     ▼
                        Built-in Tool            MCP Tool
```

该模型中：

- Capability：WorkItem 需要做什么；
- Agent：谁来执行；
- Skill：执行时加载哪些 SOP / domain knowledge；
- Tool：真正执行外部动作；
- MCP：Tool 的一种外部接入来源，而不是 Capability 本身。

### 5.3 Capability Metadata

Capability 不只保存 provider mapping，还应保存 Plan Compiler 所需要的执行约束。

建议：

```python
class CapabilitySpec(BaseModel):
    id: str
    description: str

    side_effect: Literal["read", "write"]

    eligible_agents: list[str]
    preferred_skills: list[str]
    required_tools: list[str]

    default_acceptance_kind: str | None = None
```

例如：

```yaml
capability:
  id: code_modification
  description: modify repository source code

  side_effect: write

  eligible_agents:
    - coder

  required_tools:
    - read_file
    - write_file
    - str_replace
```

### 5.4 Capability Side-Effect Authority

最终 `workspace_access` 不由 LLM 决定，而从 Capability metadata 编译。

例如：

```text
repo_exploration   → READ
code_search        → READ
bug_diagnosis      → READ
code_modification  → WRITE
test_generation    → WRITE
regression_testing → READ
code_review        → READ
```

若 WorkItem 同时包含多个 Capability：

```text
WorkspaceAccess = max(side_effects)
```

一期定义：

```text
READ < WRITE
```

LLM 的 `effect_hint` 可以更保守，但不能把 Runtime 判定的 WRITE 降级为 READ。

### 5.5 与 Plugin System 的边界

Capability-Centric Runtime 与“Everything is a Plugin”不是同一层概念。

```text
Plugin / Extension System
→ 系统组件如何注册、加载、替换

Capability System
→ 当前 WorkItem 需要什么能力，应选择哪些 Provider
```

A-SWE 不重新实现底层 Plugin Framework。

### 5.6 一期 Capability 范围

一期建议：

```text
repo_exploration
code_search
bug_diagnosis
code_modification
test_generation
regression_testing
code_review
```

必要时增加：

```text
python_debugging
database_analysis
architecture_analysis
```

Repository clone / checkout / base SHA / final patch 不属于普通 Agent Capability，而属于 Workspace Runtime。

---

## 6. Capability Resolver：能力解析器

### 6.1 模块职责

Capability Resolver 的 authoritative input 不再是 `TaskSpec.required_capabilities`，而是：

```text
ValidatedWorkPlan
+
Capability Registry
+
Available Providers
```

对每一个 WorkItem 输出：

```text
ResolvedWorkItemCapabilities
```

### 6.2 Node-Level Resolution

例如：

```text
WorkItem: diagnose
→ repo_exploration
→ code_search
→ bug_diagnosis

WorkItem: implement
→ code_modification

WorkItem: verify
→ regression_testing
```

Resolver 输出候选 Agent / Skill / Tool 组合。

### 6.3 Provider Selection

一期：

```text
Required Capability
        │
        ▼
Candidate Providers
        │
        ▼
Availability Filter
        │
        ▼
Compatibility / Permission Filter
        │
        ▼
Workspace / Tool Requirement Filter
        │
        ▼
Priority / Estimated Cost Rule
        │
        ▼
Provider Assignment
```

不需要一期实现学习型成功率预测。

### 6.4 核心价值

Capability Resolver 将：

```text
Work Package Semantics
```

与：

```text
具体 Agent / Skill / Tool
```

解耦。

因此 SemanticPlanner 不需要知道 Agent roster，Team Builder 也不需要重新理解任务语义。

---

## 7. Dynamic Team Builder：动态团队构建模块

### 7.1 建设目标

Dynamic Team Builder 输入：

```text
ValidatedWorkPlan
+
Provider Assignments
+
Task Risk / Constraints
```

输出：

```text
TeamSpec
```

Team Builder 只回答：

> **哪些执行 Provider 参与本任务？**

不负责：

```text
Task decomposition
dependency topology
execution order
parallelism
```

这些属于 Planning / DAG Materializer。

### 7.2 核心原则

Multi-Agent 不是默认选择。

> **能够由一个 Provider 覆盖全部 WorkItem 的任务，不应为了“Multi-Agent”强行创建多个 Agent。**

### 7.3 TeamSpec 只保存 Roster

建议：

```python
class TeamMember(BaseModel):
    provider_id: str
    capabilities: list[str]

    selected_for_nodes: list[str]
    selection_reason: str

class TeamSpec(BaseModel):
    members: list[TeamMember]
```

例如：

```yaml
team:
  members:
    - provider_id: repo_explorer
      selected_for_nodes:
        - diagnose
      selection_reason: diagnosis_requires_repository_exploration

    - provider_id: coder
      selected_for_nodes:
        - implement
      selection_reason: code_modification

    - provider_id: tester
      selected_for_nodes:
        - verify
      selection_reason: regression_testing
```

TeamSpec 不包含：

```text
Explorer → Coder → Tester
strategy: explore_code_test_review
```

Topology 只存在于最终 TaskDAG。

### 7.4 Planning Recon 不计入 Execution Team

Planning Phase 中的只读 Recon Probe 属于 Runtime planning infrastructure。

例如：

```text
Planning:
  Repo Profile
  Recon Probe

Execution Team:
  Coder
  Tester
  Reviewer
```

Recon Probe 不因为运行过一次就自动成为 TeamSpec 成员。

### 7.5 Explainability

Team Builder 仍需记录：

```text
为什么选
为什么不选
哪些 WorkItem 被谁覆盖
```

这些进入 Decision Trace。

---

## 8. MVP Team Selection Policy：最小可行团队策略

### 8.1 一期选择目标

在：

```text
Capability Coverage
Risk Constraint
Provider Compatibility
Tool / Permission Availability
```

全部满足的前提下：

> **选择最小可行 Provider Set。**

形式化表达：

```text
Minimize:
    Provider Count
    + Estimated Coordination Cost

Subject to:
    WorkItemCapabilityCoverage == 100%
    RiskConstraints == satisfied
    ProviderCompatibility == satisfied
```

### 8.2 一期不构造虚假 Success Probability

一期没有足够历史数据可靠估计：

```text
P(success | task, team)
```

因此不使用形式复杂但无数据基础的模型。

### 8.3 规则示例

```text
低风险单文件修改
且 Coder 覆盖全部所需 Capability
→ Team = {Coder}

Diagnosis WorkItem 需要 repo_exploration + bug_diagnosis
且 Coder profile 不满足
→ 加入 Explorer

testing_required == true
且没有现有 Provider 覆盖 regression_testing
→ 加入 Tester

risk == high
→ Review WorkItem 必须有 Reviewer-compatible Provider
```

注意：

> Team Selection Rule 不负责决定这些 Provider 的先后关系。

---

## 9. DAG Materialization 与 Task Scheduler

### 9.1 DAG Materializer 的职责

DAG Materializer 输入：

```text
ValidatedWorkPlan
+
Provider Assignment
+
Capability Metadata
+
Workspace Policy
+
Acceptance Policy
```

输出：

```text
Executable TaskDAG
```

LLM 不直接生成最终 TaskNode。

### 9.2 TaskNode

```python
class TaskNode(BaseModel):
    id: str
    objective: str

    required_capabilities: list[str]
    provider_id: str

    dependencies: list[str]

    workspace_access: WorkspaceAccess

    affected_paths: list[str] | None = None

    acceptance_criteria: list[str] = []

    retry_policy: RetryPolicy
    repair_policy: RepairPolicy | None = None

    status: NodeStatus
```

### 9.3 Side-Effect Compilation

`workspace_access` 由 Runtime 根据 Capability metadata 编译。

```text
READ + READ
→ READ

READ + WRITE
→ WRITE
```

LLM `effect_hint` 不具有 authority。

### 9.4 Dependency / Workspace Conflict Normalization

Planner dependency 先做：

```text
existence validation
cycle detection
transitive sanity check
side-effect normalization
```

Workspace Mutex 只能保证“不会同时执行”，不能保证“谁先执行”。

因此所有 unordered conflict pair 必须在 Materialization 阶段获得 deterministic order。

#### unordered WRITE / WRITE

```text
WRITE A
  ↓
WRITE B
```

按 validated planner ordinal 等稳定顺序串行。

#### unordered READ / WRITE

若只靠 lock：

```text
READ first  → 看到 mutation 前状态
WRITE first → 看到 mutation 后状态
```

会形成语义不确定性。

一期默认：

```text
READ
 ↓
WRITE
```

即独立 read-only exploration 优先基于当前稳定状态完成。

如果 READ 本来就应该观察 mutation 后状态，Planner 必须显式生成：

```text
WRITE → READ
```

依赖。

原则：

> **Workspace lock 负责运行时互斥；DAG normalization 负责确定性语义顺序。**

### 9.5 READ Parallelism

只有 unordered READ / READ 允许真实并行：

```text
Inspect API ─┐
             ├→ downstream
Inspect Test ─┘
```

并行仍受：

- NodeBoundaryPolicy；
- backend execution capacity；
- global Runtime budget；

限制。

### 9.6 Handoff Data Dependency

DAG Edge 同时表示：

```text
Control Dependency
+
Data Dependency
```

下游请求：

```text
Original Task
+
Current Node Objective
+
Bounded Dependency Handoffs
+
Runtime Constraints
+
Acceptance Criteria
```

Handoff 进入 untrusted data channel。

### 9.7 Scheduler 的职责

Scheduler 不负责 Task decomposition。

一期职责：

- Ready Node calculation；
- dependency enforcement；
- READ-only parallel dispatch；
- Workspace Access arbitration；
- WRITE exclusivity；
- Node status tracking；
- retry classification；
- bounded repair；
- cancellation；
- failure propagation；
- acceptance gate；
- Repository invariant gate；
- handoff routing；
- result aggregation。

### 9.8 Failure Taxonomy

不能把所有失败统一成 “retry once”。

至少区分：

```text
AdmissionFailure
ExecutionTransientFailure
AcceptanceFailure
VerificationFailure
PlanInvalidated
PolicyViolation
RepositoryInvariantFailure
Cancelled
```

处理原则：

| Failure | 行为 |
|---|---|
| admission failure before execution | wait / bounded retry |
| READ transient failure | RetryPolicy |
| WRITE failure 且无 workspace change | bounded retry |
| WRITE failure 且已有 workspace change | fail closed |
| acceptance does-not-hold | repair unmet condition / fail |
| verification test failure | bounded upstream repair，再 verify |
| UNVERIFIED | additional deterministic check 或保留 uncertainty |
| PLAN_INVALIDATED before any WRITE | bounded replan |
| PLAN_INVALIDATED after WRITE | MVP fail / restart-from-baseline |
| policy / repository invariant violation | fail closed |
| cancelled | propagate cancellation |

### 9.9 WRITE Retry Safety

DeerFlow `workspace_changes` snapshot 适合作为 evidence / diff，不是通用 transaction rollback：

- binary / large / sensitive content 可能不可恢复；
- snapshot 有 scan / file / diff limit；
- 它不是 Git transaction log；
- `.git` 本身被 scanner 排除。

因此 WRITE Node 前可以 capture pre-attempt snapshot。

失败后：

```text
no workspace change
→ retry may be allowed

workspace changed
→ DIRTY_WRITE_FAILURE
→ no automatic retry in MVP
```

后续若实现真正的 Git/worktree checkpoint，再开放 dirty-write rollback + retry。

### 9.10 Retry vs Repair

```text
Retry
→ 同一个 Node 因 transient execution fault 再执行

Repair
→ 下游 acceptance / verification 提供了新 evidence，
  让 upstream WRITE Node 修正已有 patch
```

典型闭环：

```text
Implement
   ↓
Verify
   │
   └─ tests fail
        ↓
Repair Implement with failure handoff
        ↓
Verify again
```

MVP 推荐：

```text
max_repairs_per_write = 1
```

Repair 不创建任意新 DAG topology，而是静态 DAG 上的 bounded attempt state machine。

优先只支持：

> **single-writer → deterministic verification failure → repair → reverify**

多 Writer repair、任意 rollback、Reviewer-only feedback 自动修复暂不做。

### 9.11 Execution-Time Replan Boundary

只有：

```text
PLAN_INVALIDATED
AND invalidating Node is READ
AND no WRITE has executed
AND replan budget remains
```

才能 replan。

一旦发生 WRITE，任意 replan 一期关闭。

### 9.12 Runtime Assembly Attestation

Compile-time Provider feasibility 基于 A-SWE Provider Contract。

但 DeerFlow 真正组装 Subagent 时还会执行：

- `SubagentConfig.tools` allowlist；
- `disallowed_tools` denylist；
- runtime authorization filter；
- Skill authorization；
- MCP / deferred tool assembly；
- Sandbox policy。

因此实际 bound tools 可能被进一步收窄。

DeerFlow `SubagentExecutor.assembly_descriptor` 在 assembly 后记录：

- effective model；
- authorization-filtered tools；
- enabled skills；
- effective policies；
- assembly fingerprint。

A-SWE Adapter 在 Node 结果中记录 descriptor / fingerprint，并检查：

```text
required_tools ⊆ actual_bound_tools
required_skills ⊆ actual_enabled_skills  # when hard-required
```

若不满足：

```text
PROVIDER_ASSEMBLY_MISMATCH
```

Node 不得因为模型自报成功而被接受。

DeerFlow 的 `AgentAssemblyObserver` 是 fail-open notification hook，异常会被吞并记录，因此不能作为 execution safety gate。

### 9.13 Node-Scoped Least Privilege

A-SWE 不应仅选择 Agent，然后让其继承一大包工具。

ExecutionPlanValidator 生成：

```python
class NodeExecutionPolicy(BaseModel):
    required_tools: list[str]
    allowed_tools: list[str]
    required_skills: list[str]

    workspace_access: WorkspaceAccess

    timeout_seconds: int
    max_turns: int
```

Adapter 将 Node policy 映射成显式 `SubagentConfig.tools` allowlist。

例如 Recon Probe：

```text
ls
glob
grep
read_file
```

明确没有：

```text
bash
write_file
str_replace
task
```

Runtime authorization 仍可进一步收窄，但不能由 A-SWE 绕过。

### 9.14 Compiled Plan Descriptor / Fingerprint

最终 Executable TaskDAG 需要稳定 identity。

```python
class CompiledPlanDescriptor(BaseModel):
    repository_base_sha: str

    task_contract_hash: str
    validated_plan_hash: str

    capability_registry_hash: str
    provider_registry_hash: str
    policy_hash: str

    dag_hash: str

    repairs: list[PlanRepair]
    warnings: list[PlanWarning]

    fingerprint: str
```

所有：

- runtime-injected verification / review node；
- workspace ordering edge；
- acceptance canonicalization；
- access upgrade；
- normalization；

都必须进入 repair / normalization log。

Trace 最终关联：

```text
A-SWE Plan Fingerprint
        │
        ├── Node A → DeerFlow Assembly Fingerprint
        ├── Node B → DeerFlow Assembly Fingerprint
        └── Node C → DeerFlow Assembly Fingerprint
```

形成端到端 reproducibility evidence。

### 9.15 Node 执行流程

```text
NodeReady
   │
   ▼
Dependency Handoff Assemble
   │
   ▼
WorkspaceAccessCheck
   │
   ▼
WRITE? capture pre-attempt snapshot
   │
   ▼
ExecutionBackend.execute_node()
   │
   ▼
Runtime Assembly Attestation
   │
   ▼
Failure Classification
   │
   ├── safe retry
   ├── dirty-write fail
   ├── bounded repair
   └── continue
   │
   ▼
Acceptance Check
   │
   ▼
Repository Invariant Check（WRITE）
   │
   ▼
Create NodeHandoff
   │
   ▼
Release Workspace Access
```

### 9.16 一期 Runtime Budget

所有预算必须属于 Runtime config，而不是 Prompt 建议。

P1 建议初始值：

```text
max_work_items = 8
max_parallel_read_nodes = min(3, backend_capacity)
planning_attempts = 2
execution_replans = 1        # pre-WRITE only
max_repairs_per_write = 1
```

其中 DeerFlow 当前 pinned baseline 的 native subagent runtime 默认：

```text
max_running = 3
```

A-SWE 不应把自身并行预算设置得高于 backend 实际 capacity。

### 9.17 一期不做

- arbitrary execution-time DAG spawning；
- dirty WRITE transactional rollback；
- unlimited replanning；
- multi-writer automatic repair；
- parallel write merge；
- distributed scheduler；
- cross-machine workspace lock；
- RL scheduling。

---

## 10. Agent Registry：智能体注册中心

### 10.1 设计定位

Capability Registry 定义：

> 系统需要 / 拥有什么能力。

Agent Registry 定义：

> 哪些执行主体能够提供这些能力，以及该执行主体在 DeerFlow Execution Plane 中应被映射成怎样的 `SubagentConfig`。

### 10.2 一期 Agent

一期只保留：

```text
Repo Explorer
Coder
Tester
Reviewer
```

`SWE Lead` 作为 A-SWE Runtime Coordinator 的逻辑角色存在，不实现为一个负责自主 delegation 的长期 DeerFlow Lead Agent。

### 10.3 Agent Metadata

```yaml
agent:
  id: repo_explorer
  role: explorer

capabilities:
  - repo_exploration
  - code_search
  - dependency_analysis

preferred_tools:
  - search_code
  - read_file
  - git

preferred_skills:
  - repository_navigation

workspace_access: read
cost_class: low
parallelizable: true
```

Tester：

```yaml
agent:
  id: tester
  role: tester

capabilities:
  - test_generation
  - regression_testing
  - failure_analysis

preferred_tools:
  - shell

preferred_skills:
  - pytest

workspace_access: read
```

Coder 默认具有：

```text
workspace_access = write
```

Reviewer 默认只读。

### 10.4 AgentProvider

A-SWE 内部使用 `AgentProvider`，而不是直接持有 DeerFlow 对象：

```python
class AgentProvider(BaseModel):
    id: str
    role: str
    capabilities: list[str]

    tools: list[str]
    skills: list[str]

    workspace_access: WorkspaceAccess

    backend: str = "deerflow"
    backend_agent_type: str
```

### 10.5 DeerFlow 映射

执行时：

```text
AgentProvider
     │
     ▼
DeerFlowExecutionBackend
     │
     ▼
SubagentConfig
     │
     ▼
SubagentExecutor
```

A-SWE 的 Agent Registry 不直接实例化 LangGraph Agent，也不直接管理 Sandbox。

### 10.6 Agent Registry 不负责的事情

Agent Registry 不负责：

- DAG Scheduling；
- Workspace Lock；
- Sandbox 生命周期；
- Tool 实际执行；
- Skill 文件加载；
- MCP Session；
- Evaluation。

这些职责分别归 Scheduler、Workspace Runtime、Execution Backend 与 Evaluation 模块。

---

## 11. Skills 系统设计

### 11.1 Skill 定位

Skill 表示：

> 执行某类任务所需要的知识、规范、SOP、操作经验和上下文模板。

Agent 表示执行主体，Skill 表示动态装配的能力增强。

因此不建议设计：

```text
PythonAgent
ReactAgent
DockerAgent
```

而更适合设计：

```text
Coder
+
Python Debugging Skill
```

或者：

```text
Coder
+
Pytest Skill
```

### 11.2 一期 Skills

一期只建设：

```text
repository-navigation
python-debugging
pytest
code-review
git
```

### 11.3 动态装配

例如：

```text
Task: Python Bug

Coder
+
python-debugging
+
pytest
```

```text
Task: Repository Exploration

Explorer
+
repository-navigation
```

### 11.4 Progressive Loading

继续复用 Harness 已有 Progressive Loading 思路：

```text
Skill Metadata
      ↓
Capability Match
      ↓
Load SKILL.md
      ↓
Load Reference / Script on Demand
```

避免一次性加载所有 Skill 内容。

### 11.5 DeerFlow Direct Subagent 的 Skill Boundary

DeerFlow 标准设计中，Lead Agent 通常拥有 thread-level `/mnt/skills` physical projection，Subagent 的 `skills` 配置主要限制：

- Skill discovery；
- Skill activation；
- active skill 的 allowed-tools policy。

A-SWE 一期直接使用 `SubagentExecutor`，不依赖 DeerFlow Lead Agent 进行自主 delegation，因此必须明确：

> **Phase 1 的 Skill allowlist 是 capability / execution policy boundary，不宣称为每个 Subagent 独立的 filesystem security boundary。**

这对面试型 MVP 不构成阻塞，但属于 Known Boundary。

后续若需要多租户或强安全隔离，再单独设计 per-agent filesystem projection / sandbox isolation。

---

## 12. MCP 集成设计

### 12.1 MCP 的正确定位

MCP 在 A-SWE 中不定义为 Capability 本身，而定义为：

> **External Tool / Context Provider Mechanism。**

关系为：

```text
Capability
    │
    ▼
Tool Requirement
    │
    ├── Built-in Tool
    │
    └── MCP Tool
            │
            ▼
       MCP Server
```

### 12.2 一期范围

一期只考虑 GitHub MCP，且不是强制依赖。

用途包括：

```text
repository_context
issue_context
pull_request_context
commit_history
```

如果本地 Repository 已具备足够上下文，一期可以优先使用本地 filesystem / git tool 完成核心 Demo。

### 12.3 Capability Mapping

例如：

```yaml
capability: issue_context

providers:
  tools:
    - source: mcp
      server: github
      tool: get_issue
```

A-SWE Runtime 关心的是：

```text
issue_context
```

而不是让上层 Team Builder 直接依赖：

```text
github.get_issue
```

---

## 13. Execution Trace / Observability

### 13.1 模块定位

Execution Trace 是一期核心模块，但 A-SWE 不重复实现 DeerFlow 已有的低层 LLM / Tool tracing。

一期采用三层 Trace：

```text
┌────────────────────────────────────────────┐
│          Layer 1: A-SWE Decision Trace     │
│                                            │
│ TaskAnalyzed                               │
│ CapabilityResolved                        │
│ TeamSelected                              │
│ DAGCreated                                 │
│ NodeScheduled                             │
│ WorkspaceAccessGranted / Blocked           │
│ NodeRetried                               │
│ EvaluationCompleted                       │
└────────────────────────────────────────────┘
                      │
                    node_id
                      ▼
┌────────────────────────────────────────────┐
│       Layer 2: Node Execution Evidence     │
│                                            │
│ status / stop_reason                       │
│ duration                                   │
│ token usage                                │
│ tool receipts                              │
│ bash execution evidence                    │
│ acceptance verdict                         │
└────────────────────────────────────────────┘
                      │
                 backend_trace_id
                      ▼
┌────────────────────────────────────────────┐
│       Layer 3: DeerFlow / LLM Trace        │
│                                            │
│ model spans                                │
│ detailed tool execution                    │
│ middleware                                 │
│ Langfuse / LangSmith                       │
└────────────────────────────────────────────┘
```

### 13.2 A-SWE Runtime Event

A-SWE 自己记录 Runtime Decision 与 Scheduler 事件：

```python
class RuntimeEvent(BaseModel):
    event_id: str
    task_id: str
    event_type: str
    timestamp: datetime

    node_id: str | None = None
    source: str
    payload: dict
```

### 13.3 一期事件类型

至少记录：

```text
TaskReceived
TaskAnalyzed

WorkspaceCreated
WorkspaceReady

CapabilityRequired
CapabilityResolved

TeamPlanningStarted
AgentSelected
SkillSelected
ToolSelected
TeamCreated

DAGCreated
NodeReady
NodeScheduled

WorkspaceAccessGranted
WorkspaceAccessBlocked
WorkspaceAccessReleased

NodeStarted
NodeCompleted
NodeFailed
NodeRetried
NodeCancelled

AcceptanceChecked

EvaluationStarted
EvaluationCompleted

TaskCompleted
TaskFailed
```

低层 `ToolCalled / ToolReturned / LLMStarted` 不强制重新转写成 A-SWE Event；需要时通过 `backend_trace_id` 下钻到底层 Trace。

### 13.4 决策可解释性

例如：

```json
{
  "event_type": "AgentSelected",
  "task_id": "task_001",
  "payload": {
    "agent": "repo_explorer",
    "required_capability": "repo_exploration",
    "reason": "repository_level_task",
    "alternatives": ["coder"],
    "rejected_reason": {
      "coder": "exploration_scope_too_large"
    }
  }
}
```

Workspace 并发决策也必须可解释：

```json
{
  "event_type": "WorkspaceAccessBlocked",
  "node_id": "implement_fix_2",
  "payload": {
    "requested": "write",
    "reason": "another_write_node_is_running"
  }
}
```

### 13.5 Tool Receipt 作为 Execution Evidence

DeerFlow `SubagentResult.tool_receipts` 可直接作为节点级执行证据。

Receipt 至少包含：

```text
tool_call_id
tool_name
status
args_sha256
output_sha256
output_bytes
created_at
```

A-SWE 不必为了“看起来可观测”再给每一个 Tool 包一层重复 callback。

Tool Receipt 用于回答：

> 这个 Node 实际调用过哪些 Tool，以及这些调用是否成功。

若需要查看完整 Tool 输入、输出和 LLM span，则通过底层 Trace 查看。

### 13.6 Trace Metrics

一期至少记录：

```text
Task Duration
Node Count
Agent Count
LLM / Token Usage（底层可获取时）
Tool Receipt Count
Retry Count
Node Failure Count
Workspace Block Count
Selected Skills
Selected Tools
Final Status
```

### 13.7 Trace UI

一期 UI 不需要复杂平台化，可以实现一个简单任务详情页：

```text
Task: Fix database connection leak
Status: Running

Task Analysis
────────────────────────
Type           bug_fix
Complexity     medium
Risk           high

Workspace
────────────────────────
Thread         aswe-task-001
Status         ready

Capabilities
────────────────────────
✓ repo_exploration
✓ database_analysis
✓ code_modification
✓ regression_testing

Selected Team
────────────────────────
Explorer
   ↓
Coder
   ↓
Tester
   ↓
Reviewer

Execution Timeline
────────────────────────
✓ inspect repository       READ
✓ locate DB layer          READ
✓ analyze lifecycle        READ
→ implement patch          WRITE
○ regression test          READ
○ review                   READ

Metrics
────────────────────────
Agents             4
Retries            1
Workspace Blocks   0
```

对于面试 Demo，Trace UI 的核心不是展示日志数量，而是展示：

> **Runtime 为什么生成当前 Team / DAG，以及 Scheduler 如何在共享 Workspace 上安全执行。**

---

## 14. Evaluation 与 Review 体系

### 14.1 模块定位

Evaluation 用于判断：

> Agent 是否真正完成了软件工程任务。

Evaluation 不等同于正式 Benchmark。

一期不做大规模 Benchmark，但每一次任务执行必须有内部 Evaluation。

### 14.2 两级验证

一期将验证拆成：

```text
Node Acceptance
      │
      ▼
Task Evaluation
```

#### Node Acceptance

回答：

> 当前 DAG Node 声称完成的结果，是否有确定性执行证据支持？

#### Task Evaluation

回答：

> 整个 Patch 是否解决原始问题，并满足 Repository-Level 软件工程要求？

### 14.3 DeerFlow Acceptance Checker 复用

`SubagentExecutor` 会采集 acceptance 所需的 execution evidence，例如：

- tool receipts；
- bash executions；
- thread / workspace context。

但标准 DeerFlow 中最终 deterministic acceptance check 位于 `task_tool` 的父调用路径。

A-SWE 绕开 Lead Agent / `task_tool`，因此 DeerFlow Adapter 必须显式回接：

```text
SubagentExecutor
      │
      ▼
SubagentResult
      │
      ▼
deerflow.subagents.acceptance_checks
      │
      ▼
NodeAcceptanceResult
```

原则：

> **复用 DeerFlow acceptance checker，不重新实现一套平行的 acceptance 语义。**

一期优先支持 DeerFlow 已能确定性判断的 criterion，例如：

```text
file:<path> exists
file:<path> non-empty
file_written:<path>
tests_passed:<command>
```

无法确定性判断的 criterion 标记为：

```text
UNVERIFIED
```

而不是由 LLM 强行判为成功。

### 14.4 Task Evaluation Pipeline

```text
Code Patch
    │
    ▼
Evaluator
    │
    ├── Build / Import Check
    ├── Existing Tests
    ├── Generated Tests
    ├── Regression Tests
    ├── Optional Static Check
    └── Reviewer
    │
    ▼
Evaluation Report
```

### 14.5 EvaluationResult

```yaml
evaluation:
  execution_status: completed

  acceptance:
    all_required_nodes_hold: true
    unverified: []

  build:
    status: pass

  tests:
    existing:
      passed: 241
      failed: 0

    added:
      passed: 4
      failed: 0

  reviewer:
    status: pass
    concerns: []

  final_status: accepted
```

### 14.6 Reviewer 关注点

Reviewer 主要检查：

- Patch 是否解决原始问题；
- 是否存在无关修改；
- 是否违反 Repository Coding Convention；
- 是否可能引入 Regression；
- 是否缺少测试；
- 是否存在明显安全 / 性能问题；
- 是否可以进一步简化。

### 14.7 Evaluation 原则

优先级：

```text
Deterministic Check
        >
Execution Evidence
        >
Reviewer Judgment
        >
LLM Self-Claim
```

例如：

```text
pytest
build
import
lint
file existence
recorded execution evidence
```

能够确定的结果，不应只依赖 Reviewer Agent 判断。

---

## 15. 核心运行流程

A-SWE Runtime 一期完整链路：

```text
TaskRequest
     │
     ▼
Runtime Policy Snapshot
     │
     ▼
Workspace Bootstrap
     │
     ▼
Basic RepositoryProfile
     │
     ▼
Task Analyzer
     │
     ├──────────────→ TaskSpec
     │
     └──────────────→ TaskContractDraft
                         │
                         ▼
                  Constraint Compiler
                         │
                         ▼
                  Compiled TaskContract
                         │
                         ▼
Targeted Repository Profiling
     │
     ▼
Planning Context Gate
   /       \
enough     gap
  │         │
  │         ▼
  │     Recon Probe
  │      READ ONLY
  │         │
  └────┬────┘
       ▼
PlanningContext
       │
       ▼
SemanticPlanner
       │
       ▼
WorkPlanProposal
       │
       ▼
PlanValidator / Normalizer
       │
       ▼
ValidatedWorkPlan
       │
       ▼
Node Capability Resolver
       │
       ▼
Provider Assignment
       │
       ▼
Minimal Team Builder
       │
       ▼
TeamSpec（roster only）
       │
       ▼
DAG Materializer
       │
       ▼
Executable TaskDAG
       │
       ▼
Workspace-Aware Scheduler
       │
       ▼
DeerFlowExecutionBackend
       │
       ▼
SubagentExecutor
       │
       ▼
Node Acceptance / Invariant / Handoff
       │
       ▼
Evaluation
       │
       ▼
Result
```

职责边界：

```text
SemanticPlanner
→ proposes bounded work packages

A-SWE Plan Compiler
→ validates and compiles execution semantics

A-SWE Scheduler
→ executes the already compiled DAG

DeerFlow
→ executes concrete bounded Agent nodes
```

Planning Phase 与 Execution Phase 在 Trace 中分开显示。

---

## 16. 工程目录与模块边界

### 16.1 推荐目录

```text
a-swe-runtime/
│
├── runtime/
│   ├── engine.py
│   ├── context.py
│   └── state.py
│
├── task/
│   ├── analyzer.py
│   ├── schema.py
│   ├── contract.py
│   ├── constraints.py
│   └── validator.py
│
├── reasoning/
│   ├── backend.py
│   ├── schema.py
│   └── errors.py
│
├── workspace/
│   ├── session.py
│   ├── manager.py
│   ├── access.py
│   ├── bootstrap.py
│   ├── repository.py
│   ├── profile.py
│   ├── invariants.py
│   └── changes.py
│
├── planning/
│   ├── schema.py
│   ├── context_gate.py
│   ├── recon.py
│   ├── planner.py
│   ├── validator.py
│   ├── normalizer.py
│   ├── node_boundary.py
│   ├── handoff.py
│   ├── acceptance_compiler.py
│   └── materializer.py
│
├── capability/
│   ├── registry.py
│   ├── resolver.py
│   ├── provider.py
│   └── schema.py
│
├── team/
│   ├── builder.py
│   └── policy.py
│
├── scheduler/
│   ├── dag.py
│   ├── scheduler.py
│   ├── executor.py
│   ├── workspace_policy.py
│   └── retry.py
│
├── agents/
│   └── registry.py
│
├── skills/
│   ├── repository-navigation/
│   ├── python-debugging/
│   ├── pytest/
│   └── code-review/
│
├── evaluation/
│   ├── evaluator.py
│   ├── acceptance.py
│   ├── checks.py
│   └── reviewer.py
│
├── observability/
│   ├── events.py
│   ├── event_bus.py
│   ├── trace_store.py
│   └── metrics.py
│
├── integrations/
│   └── deerflow/
│       ├── backend.py
│       ├── reasoning.py
│       ├── config_mapper.py
│       ├── result_mapper.py
│       ├── acceptance_adapter.py
│       └── trace_adapter.py
│
├── api/
│   └── routes.py
│
├── ui/
│   └── trace-viewer/
│
├── examples/
│   ├── simple_edit/
│   ├── bug_fix/
│   └── repo_refactor/
│
└── README.md
```

### 16.2 核心依赖方向

```text
TaskRequest
   ↓
Workspace / Repository Profile
   ↓
Task Understanding
   ↓
Planning
   ↓
Capability
   ↓
Team / Provider Set
   ↓
DAG Materialization
   ↓
Scheduler
   ↓
Execution Backend
   ↓
Evaluation
```

### 16.3 依赖约束

- `workspace/` 不依赖 Team Builder；
- Repository lifecycle 属于 `workspace/` Runtime infrastructure；
- `task/` 不依赖具体 Agent；
- `reasoning/` 只定义 provider-neutral structured reasoning contract；
- Task Analyzer / SemanticPlanner 只能通过 `ReasoningBackend` 调用模型；
- `planning/` 不依赖 DeerFlow；
- SemanticPlanner 不直接读取 Agent roster；
- `capability/` 不依赖 DeerFlow；
- `team/` 只形成 roster，不拥有 topology；
- topology 只由 `planning/materializer.py` 形成；
- `scheduler/` 禁止重新 decomposition；
- `scheduler/` 只执行已编译 TaskDAG；
- `observability/` 不参与业务决策；
- 所有 DeerFlow-specific 逻辑集中在 `integrations/deerflow/`。

### 16.4 Anti-Corruption Layer

`integrations/deerflow/` 负责：

```text
TaskNode
AgentProvider
WorkspaceSession
Node Handoff Context
      ↓
DeerFlow SubagentConfig / SubagentExecutor
      ↓
SubagentResult
      ↓
NodeExecutionResult / NodeAcceptanceResult
```

A-SWE 核心 Planning / Scheduler 代码禁止散落 `deerflow.*` import。

DeerFlow integration 分为两条 Anti-Corruption seam：

```text
ReasoningBackend
→ DeerFlow ModelInvoker

ExecutionBackend
→ DeerFlow SubagentExecutor
```

P1 暂不使用 `AgentRuns` 作为细粒度 DAG Worker，因为 AgentRuns 的 Custom Agent 与 thread 绑定，而 A-SWE 需要多个不同 Provider 共享同一 WorkspaceSession / thread filesystem。

---

## 17. 项目核心技术亮点

### 17.1 Capability-Centric Runtime

以 Capability 而非 Agent 作为 Runtime 的语义调度中心。

```text
Task Requirement
      ↓
Capability
      ↓
Provider
      ↓
Agent / Skill / Tool
```

核心价值：降低任务需求与具体执行主体之间的耦合。

### 17.2 Adaptive Team Formation

根据任务：

```text
Complexity
Risk
Scope
Capability Requirement
```

动态选择不同 Agent Topology。

核心思想：

> **什么时候不使用 Multi-Agent，同样是 Runtime 的核心能力。**

### 17.3 Minimal Feasible Team Policy

一期不虚构复杂成功率预测，而使用：

```text
Capability Coverage
+
Risk Constraint
+
Minimal Team
```

形成可解释、可实现的团队选择逻辑。

### 17.4 Workspace-Aware Task DAG Scheduling

A-SWE 不仅调度依赖关系，还显式调度共享 Repository 状态：

```text
Task DAG
   +
READ / WRITE Side Effect
   +
WorkspaceSession
```

一期以“READ / READ 可并行，存在 WRITE 即串行”的保守规则保证 correctness。

这使 Scheduler 的价值不仅是并发执行，而是：

> **在共享代码工作区上安全组织多个执行主体。**

### 17.5 Control Plane / Execution Plane Separation

```text
A-SWE Control Plane
→ what / who / when

DeerFlow Execution Plane
→ how to execute safely
```

A-SWE 不重复实现 Agent Harness，而是将核心工程价值集中在 Runtime Decision 与 Scheduling。

### 17.6 Layered Observability

可观测性分为：

```text
Runtime Decision Trace
Node Execution Evidence
Backend LLM / Tool Trace
```

既能解释“为什么这样调度”，又不重复底层 Harness 的 tracing。

### 17.7 Evaluation First

任务执行结束不等于任务完成。

必须经过：

```text
Node Acceptance
Tests
Build / Import
Regression Check
Review
```

再决定最终状态。

---

## 18. 一期实施范围

### 18.1 Runtime Core

必须完成：

```text
Task Analyzer
Workspace Runtime
Capability Registry
Capability Resolver
Dynamic Team Builder
Task Scheduler
Execution Trace
Evaluation
```

### 18.2 Agent

只实现：

```text
Repo Explorer
Coder
Tester
Reviewer
```

其中：

```text
Explorer → READ
Coder    → WRITE
Tester   → READ（生成测试文件时可升级为 WRITE）
Reviewer → READ
```

具体 Workspace Access 最终以 TaskNode 行为为准，而不是仅按角色硬编码。

### 18.3 Skills

只实现：

```text
repository-navigation
python-debugging
pytest
code-review
```

可选：

```text
git
```

### 18.4 Tool / MCP

基础：

```text
filesystem
shell
code search
git read-only exploration
```

其中必须区分：

```text
Runtime-owned Git operations
→ clone / checkout / base SHA freeze / invariant / final patch

Agent-available Git operations
→ status / log / show / diff 等任务探索
```

一期不将 Repository lifecycle operation 作为普通 `git_operation` Capability 下放给 Agent。

可选：

```text
GitHub MCP
```

GitHub MCP 不应阻塞 MVP 主链路。

### 18.5 Memory

一期只保留必要的运行态 Task Context 与 WorkspaceSession。

暂不单独建设复杂：

```text
Repository Memory
Experience Memory
Long-term Vector Memory
```

Execution Trace 为后续 Experience Memory 提供数据基础。

### 18.6 Sandbox / Deployment Boundary

一期区分两类执行模式：

```text
Development / Trusted Fixture:
LocalSandboxProvider + allow_host_bash=true

Default SWE Execution:
AioSandboxProvider + Local Container Backend
```

DeerFlow 的 LocalSandbox 会直接在 Gateway / Host 文件系统上执行命令，没有容器隔离。因此：

> **LocalSandbox 只用于受信任的 Demo Fixture、本地开发和调试，不作为任意外部 Repository 的默认执行环境。**

真实 Repository-Level SWE Task 需要执行 `pytest`、`npm test`、`make`、安装依赖等潜在不可信代码，一期默认推荐使用本地 AIO Container Sandbox。

核心要求：

- 同一 `user_id + thread_id` 可以稳定映射到同一任务 Workspace；
- Sandbox release / warm reuse 不破坏 Workspace 中的 Repository 文件状态；
- 多个 Node 可以在同一 WorkspaceSession 上连续执行。

一期不承诺：

```text
Remote AIO / K8s provisioner
cross-region shared workspace
multi-worker remote filesystem persistence
```

远程 Sandbox 可能依赖显式文件同步或共享存储，必须独立验证后再进入支持范围。

Private Repository Auth 一期作为 optional integration。DeerFlow GitHub Webhook Channel 可以注入 GitHub App installation token，但 A-SWE Direct Runtime 不应默认假设自动获得该 token。MVP 必须稳定支持 public HTTPS repo 与 local fixture repo；private clone / push 需要单独设计 RepositoryCredential / GitHub integration boundary。

### 18.7 Demo Case

一期准备 3～5 个可稳定复现的 Demo。

#### Demo 1：简单修改

```text
README / config small edit
→ Coder
```

证明：

> Runtime 不会无意义创建 Multi-Agent Team。

#### Demo 2：普通 Python Bug

```text
Explorer
 ↓
Coder
 ↓
Tester
```

同时证明：

> Explorer、Coder、Tester 共享同一 WorkspaceSession，Tester 能看到 Coder 的修改。

#### Demo 3：高风险 Bug

```text
Explorer
 ↓
Coder
 ↓
Tester
 ↓
Reviewer
```

#### Demo 4：跨模块探索

```text
Explorer A ─┐
            ├─ READ / READ parallel
Explorer B ─┘
      ↓
    Coder        WRITE exclusive
      ↓
    Tester
      ↓
   Reviewer
```

Demo 重点展示：

```text
Same Runtime
Different Task
Different Agent Topology
Shared Workspace
Explainable Scheduling
```

---

## 19. 实施原则

### 19.1 核心 Runtime 自主、Infrastructure 复用

项目主要工程价值集中在：

```text
Task → Capability
Capability → Provider
Capability → Team
Team → DAG
DAG → Execution
Execution → Trace
Execution → Evaluation
```

基础 Agent Harness、Sandbox、Skill Loader 等成熟能力尽量复用。

### 19.2 Adapter 隔离

不得让 A-SWE 的核心模块大量直接引用 DeerFlow 内部类。

优先通过：

```text
Adapter
Interface
Schema
Event
```

形成明确边界。

### 19.3 可解释优先

Dynamic Team 每次选择必须可解释。

系统不能只输出：

```text
selected = tester
```

还需要输出：

```text
selected tester because regression_testing is required
```

### 19.4 Evaluation First

新增 Agent、Skill 或 Tool 时，应明确：

- 它解决什么 Capability；
- 什么时候被选择；
- 如何判断任务执行成功。

### 19.5 Progressive Complexity

项目演进：

```text
Rule-based Runtime
       ↓
LLM-assisted Runtime
       ↓
Trace-aware Runtime
       ↓
Experience-aware Runtime
       ↓
Performance-driven Runtime
```

一期停留在前三层即可。

### 19.6 Demo First

每完成一个 Runtime 能力，都优先保证：

```text
可运行
可观察
可解释
可 Demo
```

而不是继续增加抽象层级。

---

## 20. 阶段性实施计划

### Phase 0：Integration Validation / Architecture Freeze

正式实现 Phase 1 前，先验证 A-SWE 对 DeerFlow 的关键假设。

#### P0-1：DeerFlow Integration PoC

冻结审计基线：

```text
deer-flow @ c0895d295bba34f6e95188fca380f555dabed891
```

PoC 矩阵：

| PoC | 测试内容 | 必须验证 |
|---|---|---|
| POC-01 | 直接 `SubagentExecutor(repo-explorer)` | 不经过 Lead Agent 即可执行 |
| POC-02 | Explorer 写 `marker.txt` → Coder 读取 | 同 thread workspace |
| POC-03 | Coder 修改 Python 文件 → Tester pytest | 修改对后续 Node 可见 |
| POC-04 | 两个 READ Node 并发 | 能共享 Workspace / Sandbox |
| POC-05 | 两个 Subagent 并发 | 一个结束不会提前释放另一个正在使用的 Sandbox |
| POC-06 | timeout / cancel | capacity 与 sandbox lease 无泄漏 |
| POC-07 | tools / disallowed_tools | Provider 权限映射正确 |
| POC-08 | skills | discovery / activation 范围正确 |
| POC-09 | acceptance criteria | Adapter 可回接 DeerFlow checker |
| POC-10 | tool receipts | 可关联 node → execution evidence |
| POC-11 | sequential WRITE nodes | Repository 状态正确累计 |
| POC-12 | Local AIO release / reclaim | Workspace 文件仍连续可见 |

Go / No-Go Tests：

```text
POC-02
POC-03
POC-05
POC-09
```

任何一个失败，都先修正 Adapter / Workspace 设计，不直接进入 Phase 1 主实现。

#### P0-2：Repository / Workspace Bootstrap Audit

状态：

```text
Architecture Audited
Implementation PoC Pending
```

审计后冻结链路：

```text
Repository Source
      ↓
WorkspaceSession Allocate
      ↓
Repository Bootstrap
      ↓
clone / materialize
      ↓
resolve requested_ref
      ↓
checkout exact resolved_base_sha
      ↓
verify clean repository
      ↓
capture baseline workspace snapshot
      ↓
Task DAG execution
      ↓
Repository Invariant Check
      ↓
Git ChangeSet + Workspace ChangeSet
      ↓
Evaluation / Patch Result
```

架构结论：

- DeerFlow thread workspace：直接复用；
- DeerFlow bash cwd：直接复用，Repository Root 放在 `/mnt/user-data/workspace`；
- Repository clone / checkout / base SHA freeze：A-SWE 实现；
- Repository invariant：A-SWE 实现；
- filesystem snapshot / diff：复用 DeerFlow `workspace_changes`；
- Git-aware Patch：A-SWE 实现；
- LocalSandbox：仅可信开发 / Demo；
- Local AIO Container：Repository-Level SWE 默认执行环境；
- public HTTPS repo / local fixture：MVP 必须支持；
- private repo credential：optional integration。

新增 P0-2 PoC：

| PoC | 测试内容 | 必须验证 |
|---|---|---|
| POC-13 | public repo → bootstrap | clone 到 workspace root |
| POC-14 | branch / tag → SHA | `resolved_base_sha` 稳定 |
| POC-15 | bootstrap clean check | 初始 working tree clean |
| POC-16 | Agent 修改后 invariant | HEAD 仍等于 base SHA |
| POC-17 | Agent 擅自 commit / checkout | invariant fail |
| POC-18 | final Git ChangeSet | tracked + untracked 完整 |
| POC-19 | baseline snapshot timing | clone 文件不进入 task diff |
| POC-20 | Local AIO repo execution | clone / edit / pytest / diff 全链路成立 |

P0-2 的目标不是让 Agent 学会 Git，而是让 Runtime 掌握 Repository execution identity 与可复现 baseline。

#### P0-3：Planning / DAG Architecture Audit

状态：

```text
Architecture Audited
Plan Compiler Rules Audit In Progress
```

当前冻结结论：

- DeerFlow Plan Mode / Todo 不等于 SWE Task DAG；
- A-SWE 不使用 DeerFlow Lead Agent 自主 delegation 作为核心 topology authority；
- LLM 不直接生成最终可执行 TaskDAG；
- TaskSpec 中只保留 `capability_hints`；
- Repository Profile 在 Task Analyzer / Planner 前提供结构证据；
- 信息不足时允许 bounded read-only Recon Probe；
- Recon Probe 属于 Planning Infrastructure，不计入最终 TeamSpec；
- SemanticPlanner 不读取 Agent roster；
- Planner 输出 `WorkPlanProposal`；
- Runtime 通过 PlanValidator / Normalizer 生成 `ValidatedWorkPlan`；
- Capability Resolution 下沉到 WorkItem / Node Level；
- Team Builder 只生成 roster，不生成 topology；
- DAG Materializer 拥有最终执行 topology authority；
- Scheduler 不再做 task decomposition；
- DAG Edge 同时具有 control dependency 与 handoff data dependency；
- 共享 Workspace 的无依赖 WRITE Nodes 由 Materializer 确定性串行化；
- execution-time arbitrary DAG mutation 不属于 MVP；
- Plan validation 分为 Semantic 与 Execution 两阶段；
- unordered READ / WRITE 同样需要 deterministic dependency normalization；
- Retry、Repair、Replan、Fail-Closed 必须分离；
- DeerFlow workspace snapshot 不是 transactional rollback；
- dirty WRITE failure 一期禁止自动 retry；
- Verification failure 使用 bounded repair，而不是重试 Tester；
- execution-time replan 只允许发生在首次 WRITE 前；
- DeerFlow AgentAssemblyDescriptor 用于 runtime assembly attestation；
- AgentAssemblyObserver 是 fail-open observer，不能作为安全 gate；
- Executable Plan 需要 canonical fingerprint 与 repair log。
- Task Analyzer / SemanticPlanner 的模型调用通过 provider-neutral ReasoningBackend；
- DeerFlow integration 优先复用公开 ModelInvoker 做 bounded structured reasoning；
- TaskSpec 与 authoritative TaskContract 分离；
- TaskContract / Constraint Compiler 进入下一轮 P0-4 审计。

下一步继续审计：

```text
PlanValidator
+
PlanNormalizer
+
DAG Materializer
```

的 deterministic rule set、repair boundary 与 bounded replan policy。

---

#### P0-4：TaskContract / Constraint Compiler Audit

状态：

```text
Audit In Progress
```

目标：

- 冻结 ConstraintSource / authority model；
- 区分 runtime-policy / user-explicit / repository-guidance / inference；
- 定义 provenance validation；
- 定义 hard / soft constraint；
- 定义 conflict resolution；
- 定义 monotonic constraint repair；
- 定义 TaskContract 与 SemanticPlanner / PlanValidator 的接口；
- 定义 constraint trace / fingerprint。

---

### Phase 1：Adaptive SWE Runtime MVP

这是当前唯一必须完成的产品阶段。

#### P1-1：Runtime Skeleton + Workspace Runtime

完成：

- 独立 A-SWE 工程目录；
- `ExecutionBackend` Interface；
- `DeerFlowExecutionBackend`；
- `SubagentExecutor` Adapter；
- Runtime State / Context；
- `WorkspaceSession`；
- `RepositoryBinding`；
- Repository Bootstrap；
- `resolved_base_sha` freeze；
- Repository clean baseline check；
- baseline workspace snapshot；
- Repository invariant；
- Repository ChangeSet；
- Workspace identity：`task_id → user_id + thread_id`；
- shared execution capacity；
- DeerFlow compatibility integration tests。

目标：

```text
Task Node
+
WorkspaceSession
→
DeerFlow Subagent Execution
```

#### P1-2：Task Understanding + Planning Pipeline

完成：

- RepositoryProfile；
- Basic / Targeted Profiling；
- ReasoningBackend；
- DeerFlowReasoningBackend / ModelInvoker adapter；
- TaskSpec；
- TaskContractDraft；
- ConstraintCompiler；
- Compiled TaskContract；
- Task Analyzer；
- Planning Context Gate；
- Read-only Recon Probe；
- PlanningContext；
- SemanticPlanner；
- WorkPlanProposal；
- NodeBoundaryPolicy；
- PlanValidator；
- PlanNormalizer；
- Acceptance Compiler。

目标：

```text
Task + Repository Evidence
→
ValidatedWorkPlan
```

#### P1-3：WorkPlan → Capability → Team → DAG

完成：

- Capability Metadata；
- Node-Level Capability Resolver；
- Agent Registry；
- AgentProvider；
- Minimal Feasible Team Policy；
- TeamSpec（roster only）；
- NodeHandoff schema；
- DAG Materializer；
- TaskNode；
- deterministic WRITE serialization。

目标：

```text
ValidatedWorkPlan
→
Executable TaskDAG
```

#### P1-4：Workspace-Aware Execution

完成：

- Scheduler；
- Ready Node calculation；
- Dependency Handoff Routing；
- READ / WRITE Workspace Access；
- READ-only Basic Parallel Execution；
- WRITE Exclusive Execution；
- Retry；
- Cancellation；
- Failure Propagation；
- Result Aggregation；
- Node Acceptance Gate；
- WRITE Node 后 Repository Invariant Check；
- NodeHandoff generation；
- 可选 WRITE Node before / after workspace snapshot。

#### P1-5：Execution Trace

完成：

- RuntimeEvent；
- Event Bus；
- Trace Store；
- Decision Timeline；
- Node Execution Evidence；
- backend trace correlation；
- Metrics；
- 简单 Trace Viewer。

#### P1-6：Evaluation

完成：

- DeerFlow Acceptance Adapter；
- Build / Import Check；
- Test Execution；
- Regression Test；
- Reviewer；
- Git-aware Repository ChangeSet；
- DeerFlow Workspace ChangeSet；
- Evaluation Report。

#### P1-7：Demo Packaging

完成 3～5 个稳定 Demo：

```text
Simple Task
Normal Bug
High-Risk Bug
Repository-Level Exploration
```

并在 README 展示：

```text
Task
→ Workspace / Repo Profile
→ Semantic Work Plan
→ Plan Validation
→ Capability / Provider Resolution
→ Selected Team
→ Compiled DAG
→ Workspace-Aware Scheduling
→ Trace
→ Evaluation
→ Result
```

### Phase 2：Memory & Experience（非一期目标）

后续可增加：

- Repository Memory；
- Experience Memory；
- Historical Task Retrieval；
- Similar Task Matching；
- Team Execution Statistics；
- Skill Effectiveness Statistics；
- Workspace conflict statistics。

核心目标：

> 让过去的任务执行数据开始影响未来决策。

### Phase 3：Cost / Performance-Aware Selection（非一期目标）

在具有足够 Trace 数据以后再增加：

- historical success rate；
- token cost；
- execution time；
- retry cost；
- coordination cost；
- workspace contention cost；
- Team Ranking。

此时才引入真正有数据支撑的 Cost-Aware Team Selection。

### Phase 4：Experience-Driven Adaptation（远期）

可进一步研究：

```text
Execution Trace
      ↓
Experience Extraction
      ↓
Team / Skill / Scheduling Statistics
      ↓
Selection Policy Update
      ↓
Future Runtime Decision
```

在具备实际自动策略更新机制前，不将项目核心宣传为 Self-Improving Runtime。

---

## 21. 面试展示与项目包装重点

### 21.1 不将项目包装为普通 Coding Agent

项目不是：

> 一个可以自动写代码的 Multi-Agent 系统。

而是：

> **一个能够根据 Software Engineering Task 动态构建执行能力、Agent Topology 与 Workspace-aware Task DAG 的 Runtime。**

### 21.2 README 首页首先展示 Adaptive Topology

建议 README 首屏展示：

```text
Task A
"Fix README typo"

A-SWE Runtime
→ Coder
```

```text
Task B
"Fix Python API regression"

A-SWE Runtime
→ Explorer → Coder → Tester
```

```text
Task C
"Fix high-risk database lifecycle bug"

A-SWE Runtime
→ Explorer → Coder → Tester → Reviewer
```

然后展示一个并行探索案例：

```text
Explorer A ─┐
            ├─ parallel READ
Explorer B ─┘
      ↓
Coder            exclusive WRITE
```

强调：

> **Same runtime. Different task. Different agent topology. One controlled workspace.**

### 21.3 面试必须能够讲清楚的问题

至少准备以下问题：

1. 为什么选择 Runtime，而不是再做一个 Coding Agent？
2. 为什么 Capability 比 Agent 更适合作为调度中心？
3. Capability 与 Plugin 有什么区别？
4. Agent、Skill、Tool、MCP 分别是什么角色？
5. 为什么不是所有任务都使用 Multi-Agent？
6. Dynamic Team 如何选择？
7. 为什么一期使用 Minimal Feasible Team，而不是成功率预测模型？
8. Task Scheduler 如何构建和执行 DAG？
9. 为什么 SWE Agent 需要独立的 WorkspaceSession？
10. 多个 Agent 共享同一 Repository 时如何避免并发写冲突？
11. 为什么 READ / READ 可以并行，而 MVP 中 WRITE 默认独占？
12. Retry、Cancellation 与 Failure Propagation 如何处理？
13. A-SWE 为什么直接使用 DeerFlow `SubagentExecutor` 而不是让 Lead Agent 自主 delegation？
14. A-SWE Control Plane 与 DeerFlow Execution Plane 的边界是什么？
15. `SubagentResult` 如何映射为 NodeExecutionResult？
16. Tool Receipt 与完整 Trace 有什么区别？
17. 为什么 Acceptance Checker 要从 DeerFlow 回接，而不是自己重写？
18. Execution Trace 为什么不仅仅是日志？
19. Evaluation 为什么优先使用确定性检查？
20. Direct Subagent 的 Skill projection 有什么已知边界？
21. 为什么一期限制 Local Sandbox / Local AIO，而不直接承诺 Remote Sandbox？

### 21.4 代码掌握边界

需要重点掌握并能够现场解释的代码：

```text
Task Analyzer
WorkspaceSession / Workspace Manager
Capability Registry / Resolver
Dynamic Team Builder
Selection Policy
Task DAG
Workspace Access Policy
Scheduler
Execution Trace
Evaluation
DeerFlowExecutionBackend
Acceptance Adapter
Result Mapper
```

对于底层成熟基础设施，应能够解释：

```text
SubagentExecutor 如何调用
thread_id / user_id 如何绑定 workspace
Sandbox Lease 为什么复用
Tool Receipt 如何进入节点证据
为什么不重复实现 Skill / MCP / Sandbox
```

而不是将主要时间投入重复实现 Sandbox、MCP Client 或 LangGraph Engine。

### 21.5 面试中的核心架构表述

推荐表述：

> **A-SWE is the control plane. DeerFlow is the execution plane.**

进一步展开：

```text
A-SWE:
Task → Capability → Team → DAG → Scheduling → Evaluation

DeerFlow:
Subagent → Skill / Tool / MCP → Sandbox → Execution Evidence
```

这比“我在 DeerFlow 上又套了一层 Multi-Agent”更准确，也更能体现 Runtime 设计价值。

---

## 22. 项目最终定位

A-SWE Runtime 最终不定义为：

> 一个拥有很多 Agent 的软件开发系统。

更准确的定义是：

> **A task-adaptive control plane for resolving software engineering capabilities, managing a shared repository workspace, composing the minimum feasible agent team, scheduling workspace-aware task DAGs, and tracing/evaluating execution on top of a reusable agent execution plane.**

其核心执行链路为：

```text
Task
 ↓
WorkspaceSession
 ↓
Capability
 ↓
Team
 ↓
Task DAG
 ↓
Workspace-Aware Scheduling
 ↓
DeerFlow Execution Plane
 ↓
Execution Evidence
 ↓
Evaluation
```

项目核心思想可以概括为：

> **Don't build one SWE agent. Build the runtime that assembles the right execution structure for each software engineering task.**

进一步补充：

> **A-SWE owns runtime decisions and shared-workspace scheduling; DeerFlow owns safe agent execution.**

对应中文：

> **不是构建一个固定的软件工程 Agent，而是构建一个能够针对不同软件工程任务动态组装能力、执行主体和协作拓扑，并在同一 Repository Workspace 上安全调度这些执行主体的运行时控制面。**

一期项目的成功标准不是功能数量，而是：

1. 核心链路真实运行；
2. 不同任务能够产生不同 Team / DAG；
3. 同一任务内多个 Node 能共享稳定 Workspace；
4. READ / WRITE 调度规则能够避免基础并发冲突；
5. A-SWE 能通过 Adapter 直接驱动 DeerFlow SubagentExecutor；
6. Runtime Decision 能够通过 Trace 解释；
7. Node 执行具有 Tool Receipt / Acceptance 等证据；
8. 最终结果能够通过 Evaluation 验证；
9. 3～5 个 Demo Case 能稳定展示整个流程。
