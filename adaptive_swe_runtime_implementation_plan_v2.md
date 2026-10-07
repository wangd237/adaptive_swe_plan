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
    async def prepare_node(
        self,
        node: TaskNode,
        policy: NodeExecutionPolicy,
        workspace: WorkspaceSession,
    ) -> NodeExecutionPreparation:
        ...

    async def execute_prepared(
        self,
        preparation: NodeExecutionPreparation,
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
NodeExecutionPolicy
NodeExecutionPreparation
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

    # inference hints only; TaskContract owns authoritative obligations
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

Task Analyzer 不再直接产出带 authority 的 `ConstraintSource / hard` 字段。

原因：

> **模型可以提出 constraint candidate，但不能自己证明来源，也不能自己决定执行权威。**

Analyzer 只输出候选：

```python
class ConstraintCandidate(BaseModel):
    key: str
    operator: str
    value: object

    # model hint only
    origin_hint: str | None = None
    modality_hint: str | None = None

    evidence_quote: str | None = None
    evidence_locator: str | None = None

class TaskContractDraft(BaseModel):
    candidates: list[ConstraintCandidate]
```

其中：

- `origin_hint` 不是 provenance；
- `modality_hint` 不是 enforcement；
- `evidence_quote / locator` 是 Compiler 做 provenance validation 的输入；
- Runtime Policy constraint 不由 Analyzer 抽取，而由 Runtime deterministic 注入；
- Analyzer / Recon inference 默认只进入 TaskSpec / PlanningContext，不自动成为 Hard Contract。

核心原则：

> **No model-supplied field is allowed to self-elevate into execution authority.**

### 4.5 Task Analyzer Rule Validation

LLM 输出不能直接作为最终调度依据。

例如：

```text
如果 task_type == bug_fix
→ capability_hints 至少包含 code_modification

如果用户明确要求 regression test
→ TaskSpec.testing_required = true  # planning hint
→ TaskContract candidate: verification.required = regression

如果 risk == high
→ TaskSpec.review_required = true   # planning hint
→ Runtime Policy Rule 可生成 RUNTIME_DERIVED review.required

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

### 4.8 TaskContract / Constraint Compiler

TaskSpec 解决：

> **系统如何理解任务。**

TaskContract 解决：

> **系统最终必须满足什么、禁止什么，以及这些条件由谁授权、如何合并、如何验证。**

#### 4.8.1 Provenance Authenticity ≠ Instruction Authority

Constraint Compiler 必须把下面三个概念分开：

```text
Origin
→ 这条约束来自哪里？

Provenance
→ Runtime 能否证明它确实来自那里？

Enforcement
→ 最终执行时它有多强？
```

DeerFlow 的 Project Context 已证明这种区分是必要的：

- server-owned provenance 能证明该块确实由 middleware 注入；
- 其中 project instructions 仍然是 untrusted user text；
- provenance authenticity 不会自动提升 instruction authority。

因此 A-SWE 不使用单一 `source + hard` 模型。

#### 4.8.2 Immutable TaskRequest Envelope

P1 创建 immutable task source：

```python
class TaskRequestEnvelope(BaseModel):
    request_id: str
    raw_text: str
    content_hash: str
```

Task Analyzer / Constraint Extractor 读取同一个 `raw_text`。

类似 DeerFlow 保留 `original_user_content`：

> Runtime 后续判断 user-explicit constraint 时，必须回到原始用户文本，而不能从经过 prompt wrapping / middleware enrichment / Planner paraphrase 后的文本反推用户意图。

一期不允许外部调用者直接提供 Compiler-owned provenance metadata。

#### 4.8.3 Constraint Origin

最终 CompiledConstraint 只使用 Runtime 签发的 Origin：

```python
class ConstraintOrigin(str, Enum):
    RUNTIME_POLICY = "runtime_policy"
    USER_EXPLICIT = "user_explicit"
    REPOSITORY_GUIDANCE = "repository_guidance"
    RUNTIME_DERIVED = "runtime_derived"
```

注意：

```text
ANALYZER_INFERRED
RECON_INFERRED
```

不作为最终 authoritative origin。

它们可以：

- 影响 TaskSpec；
- 触发 deterministic Runtime Policy Rule；
- 影响 Planner context；

但不能直接变成 Hard Constraint。

例如：

```text
Analyzer:
risk = high

Runtime rule:
RISK-REVIEW-001
high risk → require review

Compiled origin:
RUNTIME_DERIVED
```

而不是：

```text
origin = ANALYZER_INFERRED
hard = true
```

#### 4.8.4 Constraint Provenance

建议：

```python
class ConstraintEvidenceRef(BaseModel):
    source_kind: Literal[
        "task_request",
        "runtime_policy",
        "repository_file",
        "task_spec",
    ]

    source_id: str
    source_hash: str

    locator: str
    quote: str | None = None

class ConstraintProvenance(BaseModel):
    origin: ConstraintOrigin
    evidence: list[ConstraintEvidenceRef]

    provenance_verified: bool

    extraction_method: Literal[
        "deterministic",
        "llm",
    ]
```

重要区别：

```text
provenance_verified = true
```

只表示：

> Compiler 可以证明“这段 source text / policy rule / repository range”确实存在。

它不表示：

> LLM 对该文本语义的解析一定正确。

这种区别必须在 Trace 中保留。

#### 4.8.5 Compiler-Owned Provenance Stamp

Analyzer 若输出：

```yaml
origin_hint: user_explicit
evidence_quote: "不要修改数据库 schema"
```

ConstraintCompiler 必须：

1. 在 immutable `TaskRequestEnvelope.raw_text` 中验证 evidence；
2. 校验 locator / quote 一致；
3. canonicalize constraint；
4. 最后由 Runtime 创建 `ConstraintProvenance(origin=USER_EXPLICIT)`。

如果 evidence 无法验证：

```text
origin_hint = user_explicit
→ 不得保留 USER_EXPLICIT authority
```

结果可以：

```text
drop candidate
or
demote to analyzer inference
or
emit compiler diagnostic
```

但不能信任模型自报 provenance。

#### 4.8.6 Enforcement 与 Origin 分离

建议：

```python
class ConstraintEnforcement(str, Enum):
    LOCKED = "locked"
    HARD = "hard"
    SOFT = "soft"
```

含义：

```text
LOCKED
→ Runtime safety / operator policy
→ 不允许 Planner / User / Repo Guidance 放宽

HARD
→ 必须满足，否则 TaskContract unsatisfied

SOFT
→ planning preference / repository convention
→ 可以被更高权威 requirement 覆盖
```

Enforcement 由 Compiler / Policy Rule 决定，不接受 LLM 的 `hard=true` 作为 authority。

典型映射：

| Origin | 默认 Enforcement |
|---|---|
| Runtime Safety Policy | LOCKED |
| User explicit MUST / MUST NOT | HARD |
| Runtime-derived mandatory quality gate | HARD |
| Repository guidance | SOFT |
| Analyzer / Recon inference | 不进入 Contract |

Runtime policy 可以显式定义某条 derived rule 是 HARD 还是 SOFT。

#### 4.8.7 Repository Guidance Promotion Rule

Repository 中的：

```text
AGENTS.md
CONTRIBUTING.md
README
source comments
```

不能通过写出：

```text
THIS IS MANDATORY
IGNORE USER REQUEST
```

自行提升 authority。

P1 规则：

> **Repository text can never self-promote above SOFT.**

若希望某类 Repository metadata 形成 Hard Constraint，必须由 Runtime Policy 显式识别并 promotion。

例如未来：

```text
machine-readable generated-file manifest
+
Runtime rule
→ forbid direct edits
```

而不是因为自然语言 README 自称 mandatory 就提升。

P1 为控制复杂度：

- root-level recognized guidance 可以进入 global TaskContract；
- path-scoped nested guidance 可在目标路径明确后作为 Node Context 处理；
- 不在 Planning 初期实现复杂 hierarchical AGENTS resolution。

#### 4.8.8 Constraint Registry

ConstraintCompiler 不使用任意字符串 + 通用 DSL。

一期维护小型 typed registry：

```python
class ConstraintSpec(BaseModel):
    key: str
    merge_strategy: str
    verification_mode: str

class DeliverableEffect(str, Enum):
    REPORT_ONLY = "report_only"
    REPOSITORY_MUTATION = "repository_mutation"
    EXTERNAL_SIDE_EFFECT = "external_side_effect"

class DeliverableRequirement(BaseModel):
    description: str
    effect: DeliverableEffect
```

建议首批 key：

```text
deliverables.required  # value = DeliverableRequirement
repo.paths.allowed
repo.paths.forbidden
actions.forbidden
verification.required
review.required
change.max_files
target.exact_path
preference.test_command
semantic.requirement
```

未知 user-explicit requirement 不允许静默丢弃。

无法 canonicalize 到已知 key 时：

```text
→ semantic.requirement
→ HARD
→ verification_mode = semantic
```

保证用户要求仍留在 Contract，只是确定性验证能力下降。

#### 4.8.8.1 TaskExecutionAuthority Projection

CompiledTaskContract 还需要投影出一个 coarse-grained execution authority：

```python
class TaskExecutionAuthority(BaseModel):
    repository_mutation_allowed: bool

    # P1 默认空；外部写操作默认不开放
    external_side_effects_allowed: frozenset[str]

    granting_constraint_ids: tuple[str, ...]
    fingerprint: str
```

其中：

```text
deliverables.required(effect = REPOSITORY_MUTATION)
→ repository_mutation_allowed = true
```

例如：

```text
"Fix the connection leak"
→ DeliverableRequirement(
     description="fix connection leak",
     effect=REPOSITORY_MUTATION
   )
```

其 evidence 仍回到 immutable TaskRequest 验证。

而：

```text
"Analyze why the connection leaks"
→ DeliverableRequirement(
     description="analysis report",
     effect=REPORT_ONLY
   )
→ repository_mutation_allowed = false
```

外部写副作用 P1 默认由 Runtime Policy LOCKED deny。

核心原则：

> **TaskContract 决定任务允许产生哪类业务效果；Planner 不能通过 capability_hints 自行扩大 execution authority。**

#### 4.8.9 Constraint Merge Algebra

Constraint conflict 不能只用一个总的 source precedence。

不同 constraint family 使用不同 merge algebra。

##### Allow Scope

```text
allowed paths
→ SET INTERSECTION
```

例如：

```text
Runtime: src/**
User: src/auth/**
→ effective = src/auth/**
```

低层约束不能扩大高层允许集合。

##### Forbidden Scope / Actions

```text
forbidden paths/actions
→ SET UNION
```

限制只会增加，不会被低权威来源取消。

##### Required Obligations

```text
deliverables / verification / review
→ SET UNION
```

兼容义务共同保留。

##### Maximum Budget

```text
max_changed_files
max_runtime_cost
→ MIN
```

低层请求不能突破 Runtime ceiling。

##### Exact Choice

```text
target.exact_path
single required mode
→ EXACT / conflict detection
```

两个 incompatible HARD 值：

```text
→ CONTRACT_UNSATISFIABLE
```

##### Soft Preference

```text
preferred test command / style preference
→ PRIORITY SELECT
```

一期 soft precedence：

```text
USER_EXPLICIT
>
REPOSITORY_GUIDANCE
>
Runtime default preference
```

但 LOCKED/HARD 不通过 soft precedence 解决，而通过 constraint algebra / conflict detection 处理。

#### 4.8.10 Conflict Resolution

典型矩阵：

| Conflict | P1 行为 |
|---|---|
| User 请求 Runtime LOCKED 禁止动作 | `CONTRACT_POLICY_CONFLICT`，不静默降级执行 |
| 两个 user-explicit HARD 约束互斥 | `CONTRACT_UNSATISFIABLE` |
| User HARD vs Repository SOFT | User wins，记录 overridden-guidance warning |
| Repository SOFT vs Repository SOFT | deterministic priority / warning，不升级 Hard |
| Runtime-derived HARD + User HARD 兼容 | union |
| Runtime-derived HARD + User HARD 不兼容 | unsatisfiable / policy conflict，不能偷偷删除一方 |
| Unknown source / unverifiable provenance | 不得提升到 authoritative constraint |

如果系统运行在 interactive mode：

```text
CONTRACT_UNSATISFIABLE
→ 可以进入 clarification
```

Headless / Demo MVP：

```text
→ fail with explicit diagnostics
```

#### 4.8.11 Monotonic Constraint Repair

ConstraintCompiler 允许的自动 repair 必须满足：

> **只收窄权限、增加验证或规范化表达，不改变用户核心目标。**

允许：

```text
normalize path spelling
deduplicate equivalent constraint
runtime ceiling clamp
merge forbidden sets
add mandatory review derived from policy
promote deterministic verification obligation
```

禁止：

```text
删除 user HARD requirement
替换用户目标文件
把禁止动作改成允许
为了让计划可执行而弱化 acceptance
```

后者必须进入：

```text
contract conflict
or
replan / clarification
```

#### 4.8.12 CompiledTaskContract

建议：

```python
class CompiledConstraint(BaseModel):
    id: str

    key: str
    operator: str
    value: object

    enforcement: ConstraintEnforcement
    provenance: ConstraintProvenance

    verification_mode: Literal[
        "deterministic",
        "semantic",
        "none",
    ]

    contributors: list[str]

class CompiledTaskContract(BaseModel):
    task_request_hash: str
    runtime_policy_hash: str
    repository_base_sha: str

    constraints: list[CompiledConstraint]

    compiler_repairs: list[dict]
    warnings: list[dict]

    fingerprint: str
```

CompiledTaskContract 是 immutable runtime artifact。

#### 4.8.13 Contract → Planner Coverage

SemanticPlanner 不只生成 WorkItem，还需要声明 positive obligation coverage。

建议扩展：

```python
class WorkKind(str, Enum):
    DISCOVERY = "discovery"
    IMPLEMENTATION = "implementation"
    VERIFICATION = "verification"
    REVIEW = "review"

class WorkItemProposal(BaseModel):
    id: str
    objective: str

    work_kind: WorkKind
    capability_hints: list[str]
    depends_on: list[str]

    coverage_claims: list[str] = []
    acceptance_intent: list[str] = []
```

其中 `coverage_claims` 引用 `CompiledConstraint.id`，表示 Planner 声明“该 WorkItem 计划覆盖此 obligation”，不是完成证明。

SemanticPlanValidator 对 positive obligation 建立 `PlanCoverageMap`：

```python
class CoverageMode(str, Enum):
    RUNTIME_ENFORCED = "runtime_enforced"
    PLANNER_DECLARED = "planner_declared"

class PlanCoverageEntry(BaseModel):
    constraint_id: str
    mode: CoverageMode
    work_item_ids: tuple[str, ...]
```

规则：

```text
所有 LOCKED / HARD positive obligation
→ 必须存在 Runtime-owned enforcement
  或至少一个合法 coverage_claim
```

但 `PLANNER_DECLARED` 只证明：

> Planner 没有把 obligation 遗忘。

它不证明：

> objective 在语义上真的足以满足 obligation。

负向 constraint 不要求 Planner 每个节点重复声明，而由 ExecutionPlanValidator / NodeExecutionPolicy 全局应用。

最终 SATISFIED 只来自 Execution / Evaluation evidence。

#### 4.8.14 Contract → Execution Policy

Constraint 不应只存在于 Planner Prompt 中。

每个 CompiledConstraint 需要声明可执行 enforcement phase：

```python
class EnforcementPhase(str, Enum):
    PLAN_VALIDATION = "plan_validation"
    PRE_TOOL_GUARD = "pre_tool_guard"
    POST_NODE_INVARIANT = "post_node_invariant"
    FINAL_EVALUATION = "final_evaluation"
    SEMANTIC_REVIEW = "semantic_review"
```

例如：

```text
repo.paths.allowed = src/auth/**
```

可以编译为：

```text
PLAN_VALIDATION
→ mutation node 的 declared scope 不得明显超出允许范围

PRE_TOOL_GUARD
→ write_file / str_replace 的 path 在执行前拦截

POST_NODE_INVARIANT
→ Git ChangeSet 检查实际 changed paths

FINAL_EVALUATION
→ 再次确认最终 Patch 未越界
```

再例如：

```text
verification.required = regression
```

对应：

```text
PLAN_VALIDATION
→ 必须存在 verification obligation coverage

POST_NODE_INVARIANT / Acceptance
→ 检查真实 test evidence

FINAL_EVALUATION
→ contract verdict
```

semantic requirement：

```text
PLAN_VALIDATION
→ 必须有 plan coverage

SEMANTIC_REVIEW
→ Reviewer / semantic judge

FINAL_EVALUATION
→ 无法确认则 UNVERIFIED
```

因此 ExecutionPlanValidator 应输出：

```text
TaskContract
      ↓
NodeExecutionPolicy
      ├─ tool allowlist
      ├─ contract guard rules
      ├─ workspace access
      ├─ acceptance
      └─ post-node invariants
```

##### DeerFlow Pre-Tool Guard Reuse

DeerFlow pinned baseline 的 `GuardrailMiddleware` 在工具执行前能够获取：

```text
tool_name
tool_input
thread_id
run_id
is_subagent
tool provenance
```

并支持 fail-closed decision。

因此 P1 不需要自己重写 Tool execution middleware；建议实现 DeerFlow-side：

```text
A-SWE ContractGuardrailProvider
```

由 Adapter 将当前 Node 的 contract policy 与 DeerFlow execution identity 关联。

典型可 pre-enforce：

- `write_file(path=...)`；
- `str_replace(path=...)`；
- 禁止修改 manifest / generated file；
- 禁止特定 tool；
- node-scoped action deny。

但要注意：

> `bash(command=...)` 是自由 shell，不能仅凭简单 path argument policy 证明其无写副作用。

因此 bash-related constraint 仍需：

- conservative WRITE scheduling；
- command allow / exact test command policy（可验证时）；
- post-node Git ChangeSet；
- final contract evaluation。

##### Defense in Depth

Contract enforcement 采用：

```text
Plan-time prevention
        +
Pre-tool enforcement
        +
Post-node invariant
        +
Final evaluation
```

而不是依赖单一 Prompt 或单一 guard。

DeerFlow Guardrail provider 异常时，A-SWE constrained execution 必须采用 fail-closed semantics。

#### 4.8.15 Enforcement Guarantee Classification

并非所有 Constraint 都能在执行前完全阻止。

因此建议给 CompiledConstraint 增加：

```python
class EnforcementGuarantee(str, Enum):
    PREVENTIVE = "preventive"
    DETECTIVE = "detective"
    SEMANTIC = "semantic"
```

例如：

| Constraint | Guarantee |
|---|---|
| `write_file` path scope | PREVENTIVE + DETECTIVE |
| final changed-path scope | DETECTIVE |
| generic `bash` does not mutate source | DETECTIVE in P1 |
| semantic behavior requirement | SEMANTIC |
| required deterministic test command | PREVENTIVE where command allowlist applies + DETECTIVE via receipt |

TaskContract UI / Trace 不应把“只能事后检测”的约束宣传为已 sandbox-enforced。

#### 4.8.16 Contract → Evaluation

TaskContract 必须贯穿到最终 Evaluation。

```text
CompiledTaskContract
      ↓
Execution
      ↓
Git ChangeSet / Receipts / Tests / Reviewer Evidence
      ↓
TaskContractEvaluator
      ↓
ContractVerdict
```

约束结果至少：

```text
SATISFIED
VIOLATED
UNVERIFIED
NOT_APPLICABLE
```

确定性约束：

- changed paths；
- max changed files；
- forbidden file modification；
- required test evidence；

优先代码检查。

semantic requirement：

```text
→ reviewer / semantic evaluator
→ 无法确认时保持 UNVERIFIED
```

沿用 DeerFlow acceptance checker 的原则：

> **undecidable ≠ passed**

#### 4.8.17 Contract Fingerprint / Trace

Constraint Trace 至少记录：

```text
CandidateExtracted
ProvenanceValidated
ConstraintCanonicalized
ConstraintMerged
ConstraintOverridden
ConstraintConflict
TaskContractCompiled
TaskContractEvaluated
```

TaskContract fingerprint 必须绑定：

```text
TaskRequest hash
Runtime policy snapshot hash
Repository base SHA
Relevant repository guidance hashes
Compiled constraint set
Compiler version / rule-set version
```

最终 Plan fingerprint 再包含：

```text
task_contract_hash
```

从而回答：

> 当前 DAG 到底是在什么任务约束集合下被编译出来的？

#### 4.8.18 Constraint Compiler Pipeline

最终冻结为：

```text
Immutable TaskRequest
        │
        ├─────────────→ Runtime Policy Projection
        │
        ├─────────────→ User Constraint Extraction
        │
        └─────────────→ Repository Guidance Extraction
                              │
                              ▼
                    Constraint Candidates
                              │
                              ▼
                    Provenance Validation
                              │
                              ▼
                  Canonicalization / Registry
                              │
                              ▼
                    Enforcement Assignment
                              │
                              ▼
                      Typed Merge Algebra
                              │
                              ▼
                     Conflict Detection
                              │
                              ▼
                   Monotonic Normalization
                              │
                              ▼
                    CompiledTaskContract
                              │
                 ┌────────────┴─────────────┐
                 ▼                          ▼
          Semantic Planner             Final Evaluator
```

核心原则：

> **The model extracts candidate constraints; the runtime authenticates, merges, enforces, and evaluates them.**

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
class WorkKind(str, Enum):
    DISCOVERY = "discovery"
    IMPLEMENTATION = "implementation"
    VERIFICATION = "verification"
    REVIEW = "review"

class WorkItemProposal(BaseModel):
    id: str
    objective: str

    work_kind: WorkKind
    capability_hints: list[str]
    depends_on: list[str]

    # planner declaration only; not satisfaction evidence
    coverage_claims: list[str] = []

    acceptance_intent: list[str] = []

class WorkPlanProposal(BaseModel):
    items: list[WorkItemProposal]
    rationale: str
```

#### WorkKind 与 WorkspaceAccess 是正交维度

`WorkKind` 表示**语义执行阶段**：

```text
DISCOVERY
→ 理解 / 定位 / 诊断

IMPLEMENTATION
→ 产生任务要求的业务修改

VERIFICATION
→ 独立验证修改结果

REVIEW
→ 修改后审查 / 风险检查
```

`WorkspaceAccess` 表示**物理共享 Workspace 的并发副作用类别**。

二者不能互相推导。

典型例子：

| WorkKind | Tool | WorkspaceAccess |
|---|---|---|
| DISCOVERY | read_file / grep | READ |
| IMPLEMENTATION | str_replace | WRITE |
| VERIFICATION | bash pytest | WRITE |
| REVIEW | read_file / grep | READ |

因此：

> **Tester 因 bash 获得 WRITE lock，不代表它属于 IMPLEMENTATION；Reviewer 即使是 READ，也不代表它应该在 Writer 前执行。**

### 4.12 Node Boundary Policy

DAG Node 是：

> **bounded work package**

而不是 Todo item / reasoning step。

一期拆分原则：

| 情况 | 是否拆成独立 Node |
|---|---:|
| 可以真正并行 | 是 |
| 语义上属于明显不同的 specialization / responsibility | 是 |
| READ → WRITE side-effect boundary | 候选边界，不强制 |
| WRITE → verification boundary | 是 |
| mandatory Review gate | 是 |
| 有独立 acceptance condition | 是 |
| 强依赖且属于同一 bounded objective | 否 |
| 拆分会重复 Repository discovery 且无独立验证收益 | 否 |
| 仅因为“未来可能由不同 Provider 执行” | 否 |
| 仅因为“未来可能由同一 Provider 执行” | 否 |

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
- diagnosis + modification 本身构成一个 bounded objective；
- 拆分会导致重复 Repository discovery；
- 没有独立 verification / parallelism / specialization benefit；

则可以在 Provider Resolution 之前就编译成一个：

```text
Diagnose and Implement
workspace_access = WRITE
```

代价是该 Node 在整个执行期间独占 Workspace。

因此 NodeBoundaryPolicy 的目标不是“尽量多拆”，而是：

> **在 specialization / parallelism / verification benefit 与 handoff / duplicate discovery / coordination cost 之间选择最小合理 work package。**

#### Provider-Neutral Boundary Invariant

SemanticPlanner / SemanticPlanValidator 不读取 Agent roster，因此 NodeBoundaryPolicy 不允许使用：

```text
same provider
different provider
provider can cover both
```

作为 WorkItem 拆分 / 合并依据。

正确顺序：

```text
Task semantics
      ↓
bounded WorkItems
      ↓
Semantic validation
      ↓
Provider Resolution
      ↓
Provider assignment
```

Provider Assignment 只能给既有 WorkItem 分配 execution carrier；不能为了减少 Provider 数量重新合并 WorkItem，也不能为了制造 Multi-Agent 再拆 WorkItem。

P1 不做 provider-aware post-resolution work-item coalescing。若未来引入，只能作为独立、可验证的 topology optimization pass。

#### Runtime-Owned Verification / Review Gates

当 TaskContract 要求 verification / review，而 Planner 未提供合法独立 gate 时，Runtime 可以单调注入 gate。

Verification：

```text
all IMPLEMENTATION nodes
        ↓
__aswe_verify
WorkKind = VERIFICATION
required capability = regression_testing
```

Review：

```text
if verification exists:
    verification gate(s) → __aswe_review
else:
    implementation node(s) → __aswe_review

WorkKind = REVIEW
required capability = code_review
```

Runtime gate 的 capability 不是 LLM hint，而是 compiler-owned requirement。

不得自动把普通 Planner WorkItem 的 objective 拆成两个新语义任务；只有 Contract 已明确要求的 gate 才允许 Runtime 注入。

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
- WorkKind vocabulary / consistency；
- capability vocabulary；
- capability authority 不得超出 TaskExecutionAuthority；
- mutation WorkItem 必须绑定 repository-mutation deliverable coverage；
- coverage_claim references 是否只指向存在的 positive constraints；
- positive obligation structural coverage；
- node count / plan budget；
- deterministic NodeBoundaryPolicy；
- mandatory verification / review gate presence。

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
coverage_claim references nonexistent / negative-only constraint
WorkKind / capability contradiction
capability authority exceeds TaskExecutionAuthority
mutation WorkItem has no mutation-authorizing coverage claim
REVIEW mixed with business mutation capability
VERIFICATION mixed with business implementation capability when independent verification is required
explicit dependency contradicts mandatory phase ordering
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
CompiledTaskContract requires verification
AND no VERIFICATION work item
→ append runtime-owned verification gate

CompiledTaskContract requires review
AND no REVIEW work item
→ append runtime-owned review gate

duplicate dependency edge
→ canonical dedupe

unordered workspace-conflicting nodes
→ materialization-time deterministic phase/order edge
```

不再根据 `effect_hint` repair，因为 WorkspaceAccess 已由 Capability + ToolEffect 编译，Planner 不拥有该 authority。

规则只能增强 safety / completeness，不能静默删除用户需求，也不能重写核心任务目标。

#### Mandatory Gate Budget

Runtime 在调用 SemanticPlanner 前就知道 TaskContract 是否要求 verification / review。

因此 planner budget 应先扣除 runtime-owned mandatory gates：

```text
max_work_items = 8
mandatory_gate_count = required_verification + required_review

planner_work_item_budget
= max_work_items - mandatory_gate_count
```

这样避免 Planner 已经生成 8 个 Node 后，Runtime 再注入两个 gate 导致总预算失控。

Runtime-injected gate 使用 reserved id namespace，例如：

```text
__aswe_verify
__aswe_review
```

Planner 不允许创建 `__aswe_` 前缀 ID。

#### Warning / Optimization

例如：

```text
多个强依赖 READ items 可以合并
重复 Repository discovery
过度细粒度 decomposition
cross-node handoff overhead
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

A-SWE Runtime 将 Capability 作为 Runtime 的**语义调度词汇**。

Capability 只回答：

> **一个 bounded work package 需要具备什么能力？**

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

一期必须避免让 Capability Registry 同时承担 Provider Registry 的职责。

因此：

```text
CapabilitySpec
≠ eligible agent list
≠ concrete tool list
≠ skill allowlist
```

这些实现细节属于 Provider Contract。

### 5.2 P1 的 Provider 模型

一期只把：

> **AgentProvider**

作为可被 Scheduler / Team Builder 选择的 execution carrier。

Tool 与 Skill 不再和 Agent 并列为“可调度 Capability Provider”。

关系冻结为：

```text
WorkItem
   │
   ▼
Required Capabilities
   │
   ▼
AgentProvider
   │
   ├── CapabilityBinding
   │       ├── required tools
   │       ├── optional tools
   │       └── preferred skills
   │
   └── DeerFlow Subagent
```

其中：

- AgentProvider：真正承担一个 DAG Node 的执行主体；
- Tool：Provider 完成 Capability 所依赖的 execution resource；
- Skill：可选的 workflow / SOP / domain enhancement；
- MCP：Tool 的外部接入机制，不是 Capability，也不是 P1 的独立 schedulable provider。

未来如果需要 deterministic ToolNode，再单独增加：

```text
DirectToolProvider
```

但不进入 MVP。

### 5.3 CapabilitySpec

Capability Registry 只保存 provider-neutral semantic metadata：

```python
class CapabilityAuthorityClass(str, Enum):
    READ_ONLY = "read_only"
    REPOSITORY_MUTATION = "repository_mutation"
    EXTERNAL_SIDE_EFFECT = "external_side_effect"

class CapabilitySpec(BaseModel):
    id: str
    description: str

    # physical shared-workspace lower bound
    workspace_effect_floor: Literal["read", "write"]

    # semantic/business authority required to select this capability
    authority_class: CapabilityAuthorityClass

    default_acceptance_kind: str | None = None
```

例如：

```yaml
capability:
  id: code_modification
  description: modify repository source code
  workspace_effect_floor: write
  authority_class: repository_mutation
```

明确不保存：

```text
eligible_agents
required_tools
preferred_skills
```

原因：

> 同一 Capability 可以被不同 Provider 用不同 Tool / Skill 组合实现。

例如：

```text
code_search

Provider A
→ grep + glob + read_file

Provider B
→ repository-search MCP tool
```

如果 concrete tools 写进 CapabilitySpec，会把“语义能力”错误绑定到一种实现。

### 5.4 CapabilityBinding

Provider 对自己支持的每个 Capability 声明实现契约：

```python
class CapabilityBinding(BaseModel):
    capability_id: str

    # P1: hard dependencies must be eagerly model-callable
    required_tools: tuple[str, ...] = ()

    # optional resources may include deferred MCP tools
    optional_tools: tuple[str, ...] = ()

    preferred_skills: tuple[str, ...] = ()

    notes: str | None = None
```

例如 Tester：

```yaml
capability: regression_testing

required_tools:
  - bash

preferred_skills:
  - pytest
```

Repo Explorer：

```yaml
capability: repo_exploration

required_tools:
  - ls
  - glob
  - grep
  - read_file

preferred_skills:
  - repository-navigation
```

因此 authoritative resolution 变为：

```text
WorkItem Capability
      ↓
Candidate AgentProvider
      ↓
Provider CapabilityBinding
      ↓
Concrete Resource Requirements
```

#### Tool Contract ID 与 Exposed Name 分离

`CapabilityBinding.required_tools / optional_tools` 在 P1 仍保持轻量字符串，但其语义冻结为 **A-SWE Tool Contract ID**，不是未经验证的 DeerFlow exposed name。

为降低配置噪声，标准 SWE Tool 可以让 contract id 与 exposed name 同名，例如 `read_file`、`bash`、`str_replace`；但 DeerFlow Adapter 内部必须维护受信任映射：

```text
Tool Contract ID
      ↓
Expected Backend Implementation
      ↓
Resolved Tool Identity
      ↓
Exposed Name
```

Pinned DeerFlow 的 `get_available_tools()` 会按 exposed name 去重，且 config-defined tool 先于 built-in / MCP / ACP / plugin。因此 `name == read_file` 并不能证明实际 implementation 是 DeerFlow 标准 `read_file_tool`。

核心原则：

> **Tool name is a routing key; Tool identity is the capability/effect authority.**

同名 implementation 发生变化时，A-SWE 必须将其视为 inventory drift，而不是继续沿用旧 ToolEffect。

### 5.4.1 CapabilityBinding Merge

一个 WorkItem 可以要求多个 Capability，而一个 AgentProvider 可以同时覆盖它们。

对同一个 Provider：

```text
required_tools
→ stable ordered union

optional_tools
→ stable ordered union
→ 再减去已经 required 的 names

preferred_skills
→ stable ordered union
```

例如：

```text
bug_diagnosis
required: grep, read_file

code_modification
required: read_file, str_replace

合并后
required: grep, read_file, str_replace
```

required wins over optional：

```text
Capability A: bash optional
Capability B: bash required
→ bash required
```

禁止通过集合排序破坏声明稳定顺序；fingerprint 使用 canonical representation，execution config 保持 deterministic first-occurrence order。

### 5.4.2 Optional Tool Selection

`optional_tools` 表示 Provider 的可选增强资源，不代表默认全部开放。

P1 默认：

> **optional tool 不自动进入 Node allowed_tools。**

只有 ToolSelectionPolicy 显式选中后，才进入：

```text
selected_optional_tools
```

选择条件至少包括：

- backend candidate inventory 中存在；
- operator policy 未禁止；
- TaskContract 未禁止；
- 不会在没有明确收益时扩大 side-effect class；
- 不会无理由扩大 external side-effect surface。

例如：

```text
repo_exploration
required = read_file, grep
optional = bash

没有明确需求
→ bash 不开放
```

这保持 least privilege，也避免“optional bash”把本可并行 READ Node 无意义升级成 WRITE。

### 5.4.3 Required Tool Delivery Boundary

Pinned DeerFlow 的 deferred tool 机制具有明确边界：

```text
tool_search.enabled
AND candidate tool is MCP
→ deferred

ordinary built-in / configured non-MCP tool
→ eager
```

DeerFlow `build_deferred_tool_setup()` 只对：

```python
is_mcp_tool(tool)
```

返回 true 的 candidate 建 DeferredToolCatalog。

因此 P1 冻结：

> **CapabilityBinding.required_tools 只允许 EAGER hard dependency。**

一期 core SWE capability：

```text
repo_exploration
code_search
code_modification
test_generation
regression_testing
code_review
```

应尽量只依赖 DeerFlow eager built-ins。

MCP tools 在 P1 只能作为：

```text
optional_tools
```

被 ToolSelectionPolicy 选择后：

```text
selected optional MCP tool
      ↓
SubagentConfig static selection
      ↓
DeferredToolCatalog
      ↓
tool_search infrastructure helper
      ↓
runtime promotion
```

因此第一模型调用的 hard feasibility 不需要把“当前不可见但未来可能 promotion”误判为 required-tool missing。

未来如果业务确实需要：

> **某个 MCP tool 是完成 Node 的硬条件**

再引入：

```text
ToolRequirement.delivery = EAGER | DEFERRED_OK
```

并定义 catalog-level attestation；不在 MVP 预先建设。

### 5.4.4 Capability Authority Validation

Planner 的：

```text
capability_hints
```

仍然只是 candidate。

不能：

```text
Planner says code_modification
→ automatically grant write tools
```

否则模型仍然可以通过 capability hint 自我扩权。

SemanticPlanValidator / Capability Compiler 必须把 CapabilitySpec.authority_class 与 TaskExecutionAuthority 对齐。

P1 规则：

```text
READ_ONLY
→ 可进入普通 planning selection

REPOSITORY_MUTATION
→ TaskExecutionAuthority.repository_mutation_allowed 必须为 true

EXTERNAL_SIDE_EFFECT
→ 必须存在 explicit allowed external action
→ P1 默认 deny
```

此外，Planner-owned mutation WorkItem 必须至少有一个：

```text
coverage_claim
→ 指向 effect = REPOSITORY_MUTATION 的 positive deliverable constraint
```

否则：

```text
CAPABILITY_AUTHORITY_VIOLATION
→ PLAN_INVALID
```

这不是说 coverage claim 已证明任务完成，而只是要求：

> 每个业务 mutation Node 都必须说明它服务于哪个被 TaskContract 授权的 mutation obligation。

Runtime-owned gate 是例外：

- verification gate 的 regression_testing 属于 READ_ONLY semantic authority；
- review gate 的 code_review 属于 READ_ONLY semantic authority；
- 若未来 Runtime Policy 注入真正 mutation gate，必须由对应 Runtime-derived constraint 授权。

### 5.4.5 Authority Effect ≠ ToolEffect

两者必须严格分离：

```text
CapabilityAuthorityClass
→ 这个 WorkItem 在业务语义上被允许做什么

ToolEffect
→ 实际开放的工具在物理上可能产生什么副作用
```

例如：

```text
regression_testing

CapabilityAuthorityClass = READ_ONLY
workspace_effect_floor = READ

Provider requires bash
ToolEffect(bash) = WORKSPACE_MUTATING

最终：
semantic authority = no business source mutation
WorkspaceAccess = WRITE / exclusive
```

因此 Tester 可以获得 WRITE lock，却仍然没有业务源码修改 authority。

其执行后：

```text
Git ChangeSet
+
TaskContract
+
post-node invariant
```

必须确认没有超出 verification 允许的 mutation 范围。

同理：

```text
code_modification
CapabilityAuthorityClass = REPOSITORY_MUTATION
workspace_effect_floor = WRITE
```

只有 TaskContract 授权 repository mutation 才能进入 ValidatedWorkPlan。

### 5.5 Capability / Tool Side-Effect Authority

最终 `workspace_access` 不由 LLM 决定，也不能只从 Capability 名称推断。

必须同时考虑：

```text
Capability Workspace Effect Floor
+
Selected Provider's CapabilityBinding
+
Required Tool Effects
+
Backend / Sandbox Contract
        ↓
Effective WorkspaceAccess
```

Capability `workspace_effect_floor` 只是物理 Workspace access 的 lower bound：

```text
repo_exploration   → READ
code_search        → READ
bug_diagnosis      → READ
code_modification  → WRITE
test_generation    → WRITE
regression_testing → READ semantic intent
code_review        → READ
```

其中：

```text
regression_testing + bash
→ WRITE / exclusive
```

因为 DeerFlow `bash` 是通用 shell execution；pinned 源码明确说明 local bash path validation 不实施 bash-command write prevention。

ToolEffect：

```python
class ToolEffect(str, Enum):
    READ_ONLY = "read_only"
    WORKSPACE_MUTATING = "workspace_mutating"
    EXTERNAL_SIDE_EFFECT = "external_side_effect"
    UNKNOWN = "unknown"
```

P1 保守分类必须基于**已解析 implementation**：

| Resolved implementation | ToolEffect |
|---|---|
| DeerFlow standard `ls/glob/grep/read_file` | READ_ONLY |
| DeerFlow standard `write_file/str_replace` | WORKSPACE_MUTATING |
| DeerFlow standard `bash` | WORKSPACE_MUTATING |
| 未识别 config / MCP / ACP / extension implementation | UNKNOWN |

禁止通过 `tool.name == "read_file"` 直接赋予 `READ_ONLY`。同名 Tool 若无法确认 implementation identity，则 hard dependency 直接 preflight mismatch；仅作为 optional resource 时按 `UNKNOWN` 处理。

最终：

```text
workspace_effect_floor == READ
AND every tool in the final effective Node allowlist
    is provably READ_ONLY
→ WorkspaceAccess.READ

otherwise
→ WorkspaceAccess.WRITE
```

即：

> **MVP 只有“可证明只读”的 Node 才能并发读取 Workspace。**

这里必须使用最终：

```text
required_tools
+
selected_optional_tools
+
Node-visible non-infrastructure execution tools
```

而不是只检查 `required_tools`。

因为：

> **Tool 只要被开放给模型，就必须按“可能被调用”计算 side effect。

因此最终 WorkspaceAccess 在 NodePolicy materialization **最后一步**确定；NodeToolPolicy 允许的业务 Tool 集发生变化时，必须重新计算 fingerprint 与 WorkspaceAccess。**

LLM 的 `effect_hint` 仍然只是 hint，可以更保守，但不能降低 Runtime 编译结果。

### 5.6 与 Plugin / Extension System 的边界

Capability-Centric Runtime 与“Everything is a Plugin”不是同一层概念。

```text
Plugin / Extension System
→ 系统组件如何注册、加载、替换

Capability System
→ 当前 WorkItem 需要什么语义能力

Provider Contract
→ 某个执行主体如何实现这些能力
```

A-SWE 不重新实现 DeerFlow Plugin / Extension Framework。

### 5.7 一期 Capability 范围

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

Repository clone / checkout / base SHA / final patch 属于 Workspace Runtime，不属于普通 Agent Capability。

---

## 6. Capability Resolver：能力解析器

### 6.1 模块职责

Capability Resolver 的 authoritative input：

```text
ValidatedWorkPlan
+
Capability Registry
+
AgentProvider Registry
+
BackendInventorySnapshot
+
CompiledTaskContract
```

对每一个 WorkItem 形成：

```text
ProviderAssignment
+
NodeResourceRequirements
```

建议：

```python
class NodeResourceRequirements(BaseModel):
    required_capabilities: tuple[str, ...]
    required_tools: tuple[str, ...]
    optional_tools: tuple[str, ...]
    preferred_skills: tuple[str, ...]
    required_sandbox_features: tuple[str, ...]

class ProviderAssignment(BaseModel):
    work_item_id: str
    provider_id: str

    provider_contract_fingerprint: str
    planning_inventory_fingerprint: str

    resources: NodeResourceRequirements

    preflight_status: Literal["preflight_feasible"]
    preflight_diagnostics: tuple[str, ...]

    fingerprint: str
```

`ProviderAssignment` 是 planning artifact。

它记录：

> **为什么在当时的 Backend snapshot 下选择了这个 Provider。**

它不承诺未来执行时 deployment 永远不变。

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

Resolver 不再输出“Agent / Skill / Tool 三类 provider 组合”。

它执行：

```text
Required Capabilities
      ↓
AgentProvider semantic coverage
      ↓
CapabilityBinding expansion
      ↓
Required / Optional Tools
Preferred Skills
      ↓
Backend Preflight
      ↓
Provider Assignment
```

### 6.3 Backend Inventory Snapshot

Compile-time 不能把 A-SWE Registry 中声明的 Tool 当成真实运行时事实。

DeerFlow 的实际 Tool catalog 来自：

- config-defined tools；
- built-ins；
- model-dependent tools；
- MCP cache / personal MCP；
- ACP；
- plugin tools；
- tool groups；
- sandbox / host-bash availability。

因此 DeerFlow Adapter 需要给 Core 提供 provider-neutral inventory snapshot：

```python
class BackendToolInfo(BaseModel):
    contract_id: str
    name: str
    source: str
    delivery: Literal["eager", "deferred"]

    # Adapter-owned opaque identity of the resolved implementation.
    implementation_id: str
    schema_hash: str | None = None

    # Display/source metadata is informative; implementation_id is used for matching.
    provenance: str | None = None

    effect: ToolEffect

class BackendInventorySnapshot(BaseModel):
    backend_id: str
    captured_at: datetime

    candidate_agent_types: frozenset[str]
    candidate_tools: dict[str, BackendToolInfo]
    candidate_skill_names: frozenset[str]
    configured_model_names: frozenset[str]

    sandbox_features: frozenset[str]
    max_parallel_executions: int

    fingerprint: str
```

注意命名：

> **candidate_tools，而不是 actual_bound_tools。**

`candidate_tools` 建议按 `Tool Contract ID` 索引。`name/source/provenance` 便于解释，但 hard feasibility 与 ToolEffect 不能只依赖这些 display fields，必须匹配 `implementation_id`。

Inventory 中的 `delivery` 用于区分当前 deployment 下的：

```text
eager tool
deferred MCP tool
```

但它仍然只是 planning-time snapshot，不是 runtime assembly proof。

Inventory 只能回答：

> 当前 deployment 看起来有没有这些资源？

不能证明：

> 当前用户、当前 Node 最终 assembly 后一定拿得到这些资源。

因为后续还有：

- operator SubagentConfig；
- runtime authorization；
- skill authorization；
- dynamic MCP/config drift；
- deferred-tool assembly；
- middleware-declared tools。

DeerFlow inventory 构建属于 Adapter；Core 不直接 import `deerflow.tools.get_available_tools`。

### 6.4 Feasibility 分层

A-SWE 不把“Provider 声明自己会做”直接等价为可执行。

冻结四层状态：

```text
DECLARED
→ ProviderContract 声明 semantic capability

PREFLIGHT_FEASIBLE
→ 当前 Backend inventory / static policy 看起来可执行

ASSEMBLED_MATCH
→ DeerFlow 实际 assembly 与 NodeExecutionPolicy 一致

EXECUTED
→ Node 真正执行并产生 evidence
```

因此：

```text
Declared Capability
≠ Executable Capability
≠ Successful Execution
```

### 6.5 Provider Preflight

每个 Candidate AgentProvider 至少检查：

1. required capability 是否都有 CapabilityBinding；
2. binding.required_tools 的 Tool Contract ID 是否都能唯一解析到预期 `implementation_id`；
3. required tool 是否存在于 backend inventory 且 `delivery == eager`；
4. resolved implementation 的 exposed name 是否仍满足 operator SubagentConfig 静态 allow / deny；
5. Sandbox / backend 是否支持 required execution primitive；
6. TaskContract 是否禁止该 Tool / effect；
7. resolved model 是否存在；
8. authorization-enabled deployment 下，required tools 做 identity-aware authorization preflight；
9. authorization-enabled deployment 下，resolved model 做 `model:use` preflight；
10. selected optional tools 是否满足 least-privilege / effect policy；
11. preferred skills 是否与 Node required-tool closure 兼容；
12. 由**最终 resolved tool identity set**编译出的 WorkspaceAccess 是否满足 Node policy。

通过后才进入：

```text
PREFLIGHT_FEASIBLE
```

但 runtime authorization 仍可能在真正 assembly 时进一步收窄。

### 6.5.1 Execution-Time Live Revalidation

`PREFLIGHT_FEASIBLE` 是 planning-time 判断，会过期。

因此 Scheduler 在 NodeReady 后、不获取 Workspace WRITE lock 之前调用：

```text
ExecutionBackend.prepare_node()
```

DeerFlow Adapter 在这一阶段重新解析当前 deployment，并形成：

```python
class NodeExecutionPreparation(BaseModel):
    execution_id: str
    node_id: str
    provider_id: str

    compiled_policy_fingerprint: str

    planning_inventory_fingerprint: str
    live_inventory_fingerprint: str

    effective_policy_fingerprint: str

    backend_snapshot_id: str

    drift_observed: bool
    drift_diagnostics: tuple[str, ...]

    status: Literal["prepared"]
```

`backend_snapshot_id` 是 provider-neutral opaque id。Core 不通过它读取 DeerFlow object；它只让 Adapter 在 `execute_prepared()` 时取回本次准备阶段冻结的 concrete resources。

### 6.5.2 Fingerprint Drift Semantics

不能：

```text
planning fingerprint != live fingerprint
→ automatically fail
```

因为无关资源变化也会改变 fingerprint。

正确规则：

```text
fingerprint changed
      ↓
re-run hard feasibility against live snapshot
      │
      ├─ all constraints still hold
      │     → PREPARED
      │     → record BACKEND_DRIFT_OBSERVED
      │
      └─ required condition no longer holds
            → BACKEND_PREFLIGHT_STALE
            → fail before workspace mutation
```

Fingerprint 变化本身不是 failure；重新验证后 required condition 不再成立才是 failure。

### 6.5.3 Monotonic Runtime Narrowing

Execution preparation 只允许继续收窄 compiled Node policy。

允许：

- optional tool disappeared → drop optional tool；
- preferred skill disappeared → warning / drop；
- operator timeout 变小 → lower effective timeout；
- operator max_turns 变小 → lower effective turns；
- authorization 移除 optional tool → drop；
- new unrelated tools appear → ignore。

禁止：

- 自动添加新 Tool；
- 用新 Tool 替代 required Tool；
- 扩大 path / action authority；
- 提高 operator ceiling；
- 自动换 Provider；
- 改 WorkItem objective。

准备后形成：

```text
Compiled NodeExecutionPolicy
      ↓ monotonic narrowing
Effective NodeExecutionPolicy
```

`WorkspaceAccess` 一期保持 compiled upper-bound lock class，不因 runtime narrowing 从 WRITE 动态降回 READ。

### 6.5.4 DeerFlow Execution Snapshot Pinning

Pinned DeerFlow `SubagentExecutor` 自身已经明确采用：

> **one AppConfig snapshot per execution**

A-SWE Adapter 应强化而不是破坏这个性质。

`prepare_node()` 一次性解析并冻结：

```text
AppConfig snapshot
effective SubagentConfig
concrete base tool objects
resolved model
LoadedExtensions generation
user / auth identity
inventory fingerprint
```

随后 `execute_prepared()` 必须把同一份 AppConfig、tools、SubagentConfig、extensions snapshot 传给 `SubagentExecutor`。

禁止：

```text
prepare with config A
execute later with get_app_config() → config B
```

### 6.5.5 Authorization 是 Live Gate，不伪装成 Snapshot

AuthorizationProvider 不能被 A-SWE 宣称为冻结 policy snapshot。

DeerFlow 当前执行链会在 assembly、middleware-declared tools、tool call 与 Skill activation 等位置继续重新授权。

因此 planning/preparation preflight 只是早失败优化；真正 authority 仍来自运行时 DeerFlow authorization / guardrail。

### 6.5.6 Provider Rebinding Boundary

如果 live revalidation 失败，P1 不自动 switch 到另一个 Provider。

原因：ProviderAssignment、Tool policy、WorkspaceAccess、prompt/skills、fingerprint 都属于已编译 execution plan。

结果：

```text
BACKEND_PREFLIGHT_STALE
→ explicit node admission failure
```

发生在首次 business WRITE 前时，上层可使用已有 bounded replan policy 重新编译；Workspace 已有 mutation 时，MVP fail closed。

未来若做 Provider Rebinding，也必须是带 Trace 与新 fingerprint 的显式 Plan Repair，而不是 Scheduler 私下替换。

### 6.6 Provider Selection

一期：

```text
Required Capability
        │
        ▼
Semantic Coverage
        │
        ▼
CapabilityBinding Expansion
        │
        ▼
Backend Inventory Preflight
        │
        ▼
Static Operator Policy Check
        │
        ▼
Contract / ToolEffect Compatibility
        │
        ▼
Minimal / Priority Rule
        │
        ▼
Provider Assignment
```

不需要一期实现学习型 success probability。

### 6.7 Single Source of Truth

Capability Registry 不维护：

```text
eligible_agents
```

AgentProvider Registry 才声明：

```text
provider → capability bindings
```

若需要 capability → providers 的反向查询：

```text
Runtime derives reverse index
```

而不是维护双向配置。

否则容易出现：

```text
Capability says Coder eligible
AgentRegistry says Coder does not support capability
```

这种 drift。

### 6.8 核心价值

最终解耦成：

```text
SemanticPlanner
→ bounded work semantics

Capability Registry
→ semantic vocabulary

AgentProvider Contract
→ implementation choice

Backend Inventory
→ current deployment preflight

DeerFlow Assembly Attestation
→ actual runtime evidence
```

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

> **若一个 Provider 对多个既有 WorkItem 都是可行 execution carrier，Team roster 可以只包含这一个 Provider；但这不合并 WorkItem，也不表示共享 Agent session。**

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

Provider 只有进入：

```text
PREFLIGHT_FEASIBLE
```

集合以后，才参与 Team Selection。

然后在：

```text
WorkItem Capability Coverage
Task / Contract Constraints
Provider Static Compatibility
Backend Preflight
```

全部满足的前提下：

> **选择最小可行 AgentProvider Set。**

形式化：

```text
Minimize:
    Provider Count
    + Estimated Coordination Cost

Subject to:
    WorkItemCapabilityCoverage == 100%
    EveryAssignment == PREFLIGHT_FEASIBLE
    ContractConstraints == satisfied
```

### 8.2 Feasibility 先于 Ranking

一期禁止这种逻辑：

```text
Provider A score higher
但 required tool 缺失
→ 仍然选 A
```

必须先做 hard feasibility filter，再做 ranking。

```text
All Providers
      ↓
Hard Feasibility
      ↓
Feasible Providers
      ↓
Priority / Cost
      ↓
Selection
```

### 8.3 一期不构造虚假 Success Probability

一期没有足够历史数据可靠估计：

```text
P(success | task, provider, repo)
```

因此不使用形式复杂但无数据基础的模型。

### 8.4 规则示例

```text
低风险单文件修改
且 Coder capability bindings 覆盖全部 required capabilities
且 required tools preflight 可用
→ Team = {Coder}

Diagnosis 需要 repo_exploration + bug_diagnosis
且 Coder 没有完整 binding
→ Explorer 进入候选

CompiledTaskContract.verification.required exists
→ Runtime / validated plan 必须有 VERIFICATION WorkItem
→ 必须有 Provider 能覆盖 regression_testing
→ 若其实现依赖 bash，则 Node workspace_access = WRITE

CompiledTaskContract.review.required exists
→ Runtime / validated plan 必须有 REVIEW WorkItem
→ 必须有 code_review-compatible Provider
```

注意：

> Team Selection Rule 不负责决定 Provider 先后关系。

### 8.5 Provider Reuse

如果同一个 Provider 可以覆盖多个不同 WorkItem：

```text
Provider Count
```

在 Team roster 中只计算一次。

但每个 WorkItem / Node 仍是独立 execution assignment，并独立生成：

```text
NodeExecutionPolicy
```

因此：

```text
same Provider
≠ same Node
≠ same tool allowlist
≠ same workspace access
≠ same Agent session
≠ implicit context continuity
```

Pinned DeerFlow native subagent 是 one-shot execution：

- child graph 使用 `checkpointer=False`；
- 不继承 parent conversation history；
- 每次 invocation 构建 fresh child state；
- subagent 自身不会替下一个 Node 保留隐藏 reasoning / conversation context。

跨 Node 连续性只能来自：

```text
explicit NodeHandoff
+
shared Workspace artifacts
+
deterministic execution evidence
```

因此 Provider reuse 只表示：

> **复用同一个 execution implementation / role contract。**

它不表示：

> **复用一个持续存在的 Agent 实例。**

Team Selection 中的 `Provider Count` 是 roster complexity 指标，不是 LLM invocation count，也不是 handoff count。

例如同一个 Coder：

```text
Diagnosis Node
→ read_file / grep only
→ READ

Implementation Node
→ read_file / write_file / str_replace
→ WRITE
```

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

`workspace_access` 由 Runtime 根据：

```text
Capability Workspace Effect Floor
+
Node Required Tools
+
ToolEffect Registry
+
Backend Capability
```

联合编译。

```text
only provably read-only capabilities/tools
→ READ

any write-capable / unknown tool
→ WRITE
```

因此：

```text
regression_testing + bash
→ WRITE

code_review + read_file/grep
→ READ

repo_exploration + read_file/glob/grep
→ READ
```

这不是说“运行测试等价于修改业务代码”，而是表示：

> Scheduler 无法证明 shell execution 对共享 Workspace 无副作用，因此需要独占。

LLM `effect_hint` 不具有 authority。

### 9.4 Dependency / Workspace Conflict Normalization

Planner dependency 先做：

```text
existence validation
cycle detection
phase consistency
transitive sanity check
```

Workspace Mutex 只能回答：

> **两个 Node 能不能同时执行？**

它不能回答：

> **哪个 Node 应该先观察 Repository 状态？**

因此 DAG Materializer 先依据 `WorkKind` 冻结语义阶段，再用 `WorkspaceAccess` 解决同阶段或未显式排序的资源冲突。

#### 9.4.1 WorkKind Phase Order

P1 定义语义偏序：

```text
DISCOVERY
    ↓
IMPLEMENTATION
    ↓
VERIFICATION
    ↓
REVIEW
```

这不是要求所有任务都必须包含四阶段，而是：

> 当两个节点共享同一任务状态、存在潜在 workspace interaction 且没有用户/Planner 明确 dependency 时，Runtime 不得生成违背这一语义方向的顺序。

典型：

```text
Explorer READ
→ Coder WRITE

Coder WRITE
→ Tester WRITE/exclusive

Tester WRITE/exclusive
→ Reviewer READ
```

注意：

```text
WorkspaceAccess(Tester) = WRITE
WorkspaceAccess(Reviewer) = READ
```

并不会推出：

```text
Reviewer → Tester
```

因为语义顺序由 WorkKind 决定。

#### 9.4.2 Explicit Dependency Validation

Planner 显式 dependency 优先保留，但必须通过 phase consistency。

例如：

```text
REVIEW → IMPLEMENTATION
```

若 REVIEW 的语义是最终修改后审查，则属于：

```text
PHASE_ORDER_CONTRADICTION
```

进入 bounded replan，而不是 Runtime 静默反转 Edge。

同理：

```text
VERIFICATION → IMPLEMENTATION
```

对独立 post-change verification 是非法方向。

若 Planner 真正想表达：

> “先检查现有 tests，再决定怎么改”

则前一个 WorkItem 应标为：

```text
DISCOVERY
```

而不是 VERIFICATION。

#### 9.4.3 Cross-Phase Missing Edges

若两个相关 Node phase 有明确偏序但 Planner 漏边：

```text
DISCOVERY + IMPLEMENTATION
→ add DISCOVERY → IMPLEMENTATION

IMPLEMENTATION + VERIFICATION
→ add IMPLEMENTATION → VERIFICATION

VERIFICATION + REVIEW
→ add VERIFICATION → REVIEW
```

这种 Edge 注入是 monotonic ordering repair：

- 不改变 objective；
- 不删除 WorkItem；
- 不扩大权限；
- 只阻止语义阶段倒序或错误并行。

所有 injected edges 必须进入：

```text
PlanRepair log
```

#### 9.4.4 Same-Phase Workspace Conflict

当 WorkKind 相同、Planner 未显式排序，但 WorkspaceAccess 冲突：

##### WRITE / WRITE

```text
Node A
  ↓
Node B
```

按 validated planner ordinal 等稳定顺序串行。

##### READ / WRITE

不再采用全局：

```text
READ → WRITE
```

规则。

同一 phase 中使用稳定 planner ordinal：

```text
earlier ordinal
      ↓
later ordinal
```

因为一旦二者都属于同一语义阶段，Runtime 没有足够 authority 根据 READ/WRITE 猜测业务数据依赖。

若真实语义需要固定顺序，Planner 必须显式依赖，或由 phase/gate rule 推导。

#### 9.4.5 READ / READ

无 dependency 且均为：

```text
WorkspaceAccess.READ
```

时可以并行。

#### 9.4.6 WorkKind 与 Access Validation

典型 consistency：

| WorkKind | 允许的物理 Access | 说明 |
|---|---|---|
| DISCOVERY | READ 为主 | 若编译成 WRITE，需明确 ToolEffect 原因并产生 warning |
| IMPLEMENTATION | READ / WRITE | 通常 WRITE |
| VERIFICATION | READ / WRITE | bash testing 常被物理编译成 WRITE |
| REVIEW | READ 为主 | P1 不允许 Reviewer 拥有 business mutation capability |

因此：

> **WorkKind 是语义 phase；WorkspaceAccess 是并发资源 class。二者都进入 DAG Materialization，但职责不能混用。**

#### 9.4.7 Ordering Algorithm

P1 Materializer：

```text
1. preserve validated explicit edges
2. reject explicit phase inversion
3. inject mandatory gate edges
4. add missing cross-phase edges when nodes are workspace-related
5. recompute acyclic
6. for remaining unordered conflicting same-phase pairs:
      stable ordinal serialization
7. recompute transitive reduction / canonical edge ordering
8. emit PlanRepair log
```

其中“workspace-related”一期可保守定义为：

> 同一个 WorkspaceSession 中、至少一个节点不是 pure READ-independent branch。

不做复杂 file-level dependency inference。

原则：

> **Semantic phase determines direction; Workspace policy determines concurrency.**

### 9.5 READ Parallelism

只有 dependency-free 且语义 phase 允许并行的 READ / READ 才允许真实并行：

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
BackendPreflightStale
ProviderAssemblyMismatch
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
| backend preflight stale | fail before execution；pre-WRITE 时可 bounded replan |
| provider assembly mismatch | fail closed；P1 不自动换 Provider |
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

Compile-time Provider feasibility 基于：

```text
ProviderContract
+
BackendInventorySnapshot
+
Operator SubagentConfig
```

它只能达到：

```text
PREFLIGHT_FEASIBLE
```

DeerFlow 真正组装 Subagent 时还会执行：

- `SubagentConfig.tools` allowlist；
- `disallowed_tools` denylist；
- runtime authorization filter；
- Skill authorization；
- MCP / deferred tool assembly；
- middleware-declared tools；
- model resolution；
- Sandbox policy。

因此真实 assembly 可能与 preflight 不同。

#### Descriptor 不是默认白送

Pinned DeerFlow 中：

```text
SubagentExecutor.assembly_descriptor
```

默认是 `None`。

只有当前 extension snapshot 中存在：

```text
AgentAssemblyObserver
```

时，`_describe_assembly()` 才会真正构建 descriptor。

这是有意的性能优化，因为 descriptor 需要 hash：

- tool descriptions；
- tool JSON schemas；
- middleware policy；
- prompt；
- skills；
- model policies。

因此 A-SWE 不能假设 descriptor 永远存在。

#### A-SWE Attestation Extension

好消息是 `AgentAssemblyObserver` 属于公开：

```text
deerflow_extension_api.ExtensionRegistry
```

contract。

P1 增加一个极薄的 DeerFlow extension：

```python
@extension(api="0.2.0", name="a_swe_attestation")
def install(registry, config):
    registry.agent_assembly_observer(ASWEAssemblyObserver())
```

Observer 本身只做轻量记录即可。

它的关键作用之一是让 DeerFlow 构建：

```text
AgentAssemblyDescriptor
```

随后 A-SWE Adapter 从**当前 SubagentExecutor 实例**读取：

```text
executor.assembly_descriptor
```

避免自行重建 assembly 描述。

禁止：

- subclass SubagentExecutor 只为取 assembly；
- 直接调用 private `_describe_assembly()`；
- 根据 prompt / config 猜 actual tools。

#### AssemblyAttestation

建议映射：

```python
class AssemblyAttestation(BaseModel):
    provider_id: str
    node_id: str

    effective_model: str
    actual_tool_names: frozenset[str]
    candidate_skill_names: frozenset[str]

    middleware_names: tuple[str, ...]
    deferred_tool_names: frozenset[str]

    backend_fingerprint: str

    matches_node_policy: bool
    diagnostics: list[str]
```

DeerFlow `build_assembly_descriptor()` 会将：

```text
explicit bound tools
+
middleware.tools
```

合并后写入 descriptor，因此 descriptor.tools 可以覆盖最终 build-time declared tool set，而不是只看 `SubagentConfig.tools`。

至少检查：

```text
required_tools ⊆ actual tool descriptors
```

以及：

```text
actual business tools
⊆ NodeExecutionPolicy.allowed_business_tools

actual infrastructure tools
⊆ NodeExecutionPolicy.infrastructure_tool_names
```

同时可以检查：

- expected Contract Guard middleware / policy 是否存在；
- resolved model 是否符合 Node model policy；
- expected runtime ceilings 是否进入 effective policies。

Skill 只做 availability evidence：

```text
preferred_skills ⊆ enabled_skills
→ informational / warning
```

不能把它当成 Skill was used 的证明。

若 required tool 缺失：

```text
PROVIDER_ASSEMBLY_MISMATCH
```

Node 不得因为模型自报 completed 而被 A-SWE 接受。

#### Attestation 不是 Security Gate

DeerFlow 的 `AgentAssemblyObserver` 是 fail-open notification hook。

同时 descriptor 是 Agent assembly 结束时产生，不能把它当成真正的 pre-execution authorization barrier。

因此：

```text
Security / authority
→ operator config narrowing
→ runtime authorization
→ ContractGuardrailProvider
→ sandbox policy

AssemblyAttestation
→ runtime evidence
→ compatibility / validity gate
```

即：

> **Attestation 是 post-build / reproducibility evidence；真正阻止无效 Node 进入第一轮 LLM 的是 ASWENodeToolPolicyMiddleware 的 model-call admission gate。**

因此职责冻结为：

```text
BackendInventory
→ planning-time preflight

NodeToolPolicy first-model gate
→ runtime execution admission

DeerFlow AssemblyDescriptor
→ post-build attestation / reproducibility

ContractGuardrail
→ argument-sensitive action enforcement
```

不要让 AssemblyDescriptor 承担它当前 API 无法承担的 pre-execution control responsibility。

### 9.13 Node-Scoped Least Privilege

ExecutionPlanValidator 为每个 Node 编译独立 policy：

```python
class NodeExecutionPolicy(BaseModel):
    provider_id: str
    required_capabilities: tuple[str, ...]

    required_tools: tuple[str, ...]
    selected_optional_tools: tuple[str, ...]

    allowed_business_tools: tuple[str, ...]
    infrastructure_tool_names: tuple[str, ...]
    denied_tools: tuple[str, ...]

    preferred_skills: tuple[str, ...]

    tool_effects: dict[str, ToolEffect]
    workspace_access: WorkspaceAccess

    model_policy: str

    contract_guard_rules: tuple[str, ...]
    post_node_invariants: tuple[str, ...]

    timeout_seconds: int
    max_turns: int

    provider_contract_fingerprint: str
    planning_inventory_fingerprint: str
    task_contract_fingerprint: str

    fingerprint: str
```

`allowed_business_tools` 是该 Node 的**最大业务执行工具集**，不是“建议工具”。

```text
allowed_business_tools
=
required_tools
+
selected_optional_tools
```

Framework-generated helper 单独放入：

```text
infrastructure_tool_names
```

避免把业务权限和 Harness 自身 discovery machinery 混在一起。

例如 Recon Probe：

```text
allowed:
ls
glob
grep
read_file

denied:
bash
write_file
str_replace
task
```

#### Adapter 必须 Monotonic Narrowing

不能：

```text
Node allowed_tools
→ 直接覆盖 operator SubagentConfig.tools
```

必须：

```text
Operator allow
∩ Node allow

Operator deny
∪ Node deny
```

然后 Runtime authorization 还可以继续收窄。

因此：

> **NodeExecutionPolicy 永远不能扩大 DeerFlow Operator 已经设置的权限。**

#### Final Tool Visibility Backstop

Pinned DeerFlow 中 `SubagentConfig.tools` 只过滤 explicit / regular tools。

随后仍可能加入：

- generated `tool_search`；
- generated `describe_skill`；
- middleware-declared tools，例如 model-dependent / extension tool。

DeerFlow 自身的 `tool_declarations.py` 也明确说明：

> LangChain 会在 host explicit tool list 过滤之后，再把 `middleware.tools` 折入最终 ToolNode。

因此：

```text
SubagentConfig.tools
≠ complete final model-visible schema allowlist
```

P1 若要真正实现 Node-scoped least privilege，需要第二道 name-level policy。

##### 不能使用 packaged Extension Middleware 做 enforcement

公开 `deerflow_extension_api` 的 middleware contribution 在 pinned baseline 是 observational contract：

- host 强制向下游传 original request；
- 不能 veto tool call；
- 不能改写 model-visible tools；
- contributor failure fail-open。

因此它适合：

```text
Assembly Observer
Trace / Metrics
```

不适合：

```text
Node security enforcement
```

##### Trusted Configured Middleware

DeerFlow 另外提供 operator-owned：

```text
extensions.middlewares
```

这是 trusted `AgentMiddleware` customization：

- 直接实例化真实 AgentMiddleware；
- 同时进入 lead / subagent chain；
- 不经过 observational isolation wrapper；
- 可以修改 ModelRequest；
- 可以 veto ToolCall；
- 配置加载失败会让 agent build fail loudly。

A-SWE P1 可以增加：

```text
ASWENodeToolPolicyMiddleware
```

作为 trusted configured middleware。

##### NodePolicyStore

`SubagentExecutor` 没有任意 extra runtime context 参数。

因此 P1 不滥用：

```text
authz_attributes
knowledge_scope
```

承载 A-SWE 业务 policy。

采用进程内、短生命周期：

```python
NodePolicyStore[node_execution_id] = NodeExecutionPolicy
```

Adapter 为每次 Node attempt 生成唯一：

```text
run_id = aswe:<task>:<node>:<attempt>:<uuid>
```

并传给 `SubagentExecutor.run_id`。

Middleware 从：

```text
runtime.context["run_id"]
```

查当前 Node policy。

生命周期：

```text
register immutable policy
      ↓
attach mutable runtime outcome
      ↓
execute node
      ↓
Adapter reads terminal outcome
      ↓
finally remove binding
```

建议：

```python
class NodePolicyRuntimeOutcome:
    admission_checked: bool
    admission_failure: str | None
    missing_required_tools: tuple[str, ...]
    denied_tool_calls: list[dict]
```

Policy 本身 immutable；runtime outcome 单独存放，不允许 middleware 原地改写 compiled policy。

要求：

- concurrency-safe；
- immutable value；
- bounded / cleanup-safe；
- cancellation 也必须 finally cleanup；
- 普通 DeerFlow run 查不到 A-SWE policy → pass-through；
- `aswe:` managed run 若 policy 丢失 → fail closed。

这个设计只承诺：

> **single-process P1 runtime。**

分布式 Worker 后续必须把 PolicyStore 换成显式 durable / remote policy carrier。

##### Middleware Enforcement

Pinned middleware order 采用：

> **first in list = outermost**

而 DeerFlow 在 Subagent chain 中先加入：

```text
SkillToolPolicyMiddleware
DeferredToolFilterMiddleware
...
configured extensions.middlewares
```

因此 `ASWENodeToolPolicyMiddleware` 作为 configured middleware 位于这些 schema filter 的内侧。

它在每一次 model call 看到的 `request.tools` 已经经过：

- active-skill tool policy；
- deferred MCP schema hiding；
- 前序 authorization / assembly filtering；
- middleware-declared tool folding。

随后 A-SWE 再执行自己的 final Node policy。

在 model-call boundary：

```text
current request.tools
        │
        ├─ verify eager required tools present
        │
        └─ intersect:
           Node allowed business tools
           +
           allowed framework infrastructure tools
        ↓
final model-visible tool schemas
```

###### First-Model Admission Gate

第一次 model call 前：

```text
required eager tools
      ⊆
current request.tools
```

必须成立。

否则：

```text
PROVIDER_ASSEMBLY_MISMATCH
```

并且不能让模型“先试试看”。

实现上不发明第二套 Agent termination protocol，但也**不抛普通 Exception / model request AdmissionError**。

Pinned DeerFlow 的 `LLMErrorHandlingMiddleware` 会捕获内层普通 `Exception`，进入自己的 LLM/provider retry-classification 与 fallback 路径。若 A-SWE 在这里直接 raise：

```text
Provider policy mismatch
→ 被包装成 LLM failure
```

Failure taxonomy 会失真。

因此 trusted Node middleware 在 required-tool closure 失效时采用：

```text
ASWENodeToolPolicyMiddleware
      ↓
DO NOT call downstream model handler
      ↓
return non-empty synthetic AIMessage / ModelResponse
      ↓
additional_kwargs:
  deerflow_error_fallback = true
  aswe_policy_failure = true
  aswe_failure_code = PROVIDER_ASSEMBLY_MISMATCH
  aswe_node_execution_id = <id>
```

设计理由：

1. synthetic response 不调用 `handler(request)`，因此 upstream LLM provider 零调用；
2. response content 必须非空，避免触发 DeerFlow empty-response retry；
3. DeerFlow `SubagentExecutor._extract_llm_error_fallback()` 已经把 terminal `deerflow_error_fallback=true` AIMessage 映射为 `SubagentStatus.FAILED`；
4. A-SWE 自有 marker 只用于精确 FailureTaxonomy，不修改 DeerFlow terminal contract；
5. 普通 DeerFlow LLM fallback 没有 `aswe_policy_failure`，Adapter 不会误分类。

Adapter 在 Result Mapping 时：

```text
SubagentResult.status == FAILED
AND terminal AIMessage.aswe_policy_failure == true
→ map as A-SWE policy / assembly failure

otherwise
→ preserve ordinary DeerFlow LLM / execution failure
```

如果未来 DeerFlow 提供正式公开的 typed node-admission failure contract，再评估替换；P1 不依赖不存在的 admission seam。
4. 不调用 LLM provider；
5. 不重试；
6. 生成带 `deerflow_error_fallback=true` 的 terminal AIMessage；
7. `SubagentExecutor` 现有 `_extract_llm_error_fallback` 将结果映射为 FAILED；
8. Adapter 从 NodePolicy runtime outcome 映射成 A-SWE 的 `PROVIDER_ASSEMBLY_MISMATCH`。

这条 DeerFlow internal exception 只能存在于：

```text
integrations/deerflow/
```

Anti-Corruption Layer 内，并通过 pinned compatibility test 保护。

Core 只看：

```text
NodeAdmissionFailure
reason = PROVIDER_ASSEMBLY_MISMATCH
```

###### Revalidation on Every Model Turn

required tool availability 不只首轮检查。

Skill activation / authorization state可能在后续模型轮次继续收窄 Tool view。

所以每次 model call 都重新验证：

```text
required eager tools ⊆ current request.tools
```

若中途不再成立：

```text
fail before next provider call
```

避免 Agent 在失去必要执行能力后继续消耗模型 token 并产生虚假完成报告。

在 tool-call boundary：

```text
tool call name
      ↓
Node policy check
      ↓
ALLOW / DENY
```

因此形成：

```text
Static pruning:
SubagentConfig.tools

Dynamic final visibility:
ASWENodeToolPolicyMiddleware

Identity authorization:
DeerFlow AuthorizationProvider

Argument-sensitive contract:
ContractGuardrailProvider
```

四层职责不同，禁止合并成一个大 Policy 类。

##### Framework Infrastructure Tools

至少区分：

```text
business tools
vs
framework infrastructure tools
```

例如 DeerFlow 生成的：

```text
tool_search
describe_skill
```

不应因为它们不是 CapabilityBinding.required_tools 就被误判成业务越权。

但 infrastructure allowlist 只能由 DeerFlow Adapter 根据当前 assembly mode 生成，不能由 LLM / Provider 自报。

`tool_search` 本身的 deferred catalog 已基于 SubagentConfig + authorization 后的候选集构建，因此允许该 helper 不等价于允许它重新引入已被 Node static pruning 移除的 ordinary tools。

#### Skill Narrowing

Node 可以只开放：

```text
preferred_skills
```

作为 `SubagentConfig.skills` discoverability scope。

如果 operator config 也有 Skill allowlist：

```text
Operator skill allow
∩ Node skill allow
```

但 Skill 缺失在 P1 默认不导致 Provider hard failure。

#### Runtime Ceiling Narrowing

```text
effective max_turns
= min(operator max_turns, node max_turns)

effective timeout
= min(operator timeout, node timeout)
```

A-SWE 不能提高 operator ceiling。

#### WorkspaceAccess

对于使用 `bash` 的 Node：

```text
workspace_access = WRITE
```

除非未来 Backend 明确提供可证明 read-only 的 shell contract。

Tester 因此默认不会和普通 READ Explorer 并行共享可变 Workspace。

---

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
ExecutionBackend.prepare_node()
   │
   ├── live backend revalidation
   ├── monotonic runtime narrowing
   ├── snapshot pinning
   └── stale preflight → fail before workspace lock
   │
   ▼
WorkspaceAccessCheck
   │
   ▼
WRITE? capture pre-attempt snapshot
   │
   ▼
ExecutionBackend.execute_prepared()
   │
   ├── NodePolicyStore bind
   ├── DeerFlow Subagent assembly
   ├── first/every-model admission gate
   └── tool / contract enforcement
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

> 系统使用什么语义能力词汇。

Agent Registry 定义：

> 哪些 execution carrier 可以实现这些 Capability，以及每种实现需要哪些 execution resources。

Agent Registry 不直接保存 DeerFlow runtime object。

### 10.2 一期 AgentProvider

一期只保留：

```text
Repo Explorer
Coder
Tester
Reviewer
```

`SWE Lead` 作为 A-SWE Runtime Coordinator 的逻辑角色存在，不实现为负责自主 delegation 的长期 DeerFlow Lead Agent。

### 10.3 Provider Contract

建议：

```python
class AgentProvider(BaseModel):
    id: str
    role: str

    backend: Literal["deerflow"] = "deerflow"
    backend_agent_type: str

    capability_bindings: dict[str, CapabilityBinding]

    model_policy: Literal["operator_config_or_inherit"] = "operator_config_or_inherit"

    cost_class: Literal["low", "medium", "high"] = "medium"

    fingerprint: str
```

明确不保存：

```text
workspace_access
actual_tools
actual_skills
```

原因：

- workspace access 是 Node + selected binding + ToolEffect 的编译结果；
- actual tools 只有 DeerFlow assembly 后才能知道；
- Skill allowlist 只表示 discoverability，不表示实际 activation/use。

### 10.4 Provider 示例

Repo Explorer：

```yaml
provider:
  id: repo_explorer
  role: explorer
  backend_agent_type: repo-explorer

capability_bindings:
  repo_exploration:
    required_tools:
      - ls
      - glob
      - grep
      - read_file
    preferred_skills:
      - repository-navigation

  code_search:
    required_tools:
      - grep
      - read_file
```

Tester：

```yaml
provider:
  id: tester
  role: tester
  backend_agent_type: tester

capability_bindings:
  regression_testing:
    required_tools:
      - bash
    preferred_skills:
      - pytest

  test_generation:
    required_tools:
      - read_file
      - write_file
      - str_replace
```

注意：

> Tester 不再静态声明 `workspace_access=read`。

`regression_testing + bash` 会在 Node 编译阶段得到 `WorkspaceAccess.WRITE`。

### 10.5 Operator Config 与 A-SWE Policy 的关系

DeerFlow 的 `SubagentConfig.tools` 是 operator / deployment 侧的静态 allowlist。

A-SWE Node policy 只能继续收窄它，不能扩权。

正确关系：

```text
Operator SubagentConfig
        ∩
A-SWE Node Allowlist
        ∩
Runtime Authorization
        ↓
Actual Bound Tools
```

Adapter 的 monotonic narrowing：

```python
base = get_subagent_config(provider.backend_agent_type)

if base.tools is None:
    narrowed_allow = node_policy.allowed_business_tools
else:
    narrowed_allow = intersection(base.tools, node_policy.allowed_business_tools)

narrowed_deny = union(
    base.disallowed_tools,
    node_policy.denied_tools,
)

if not required_tools <= (narrowed_allow - narrowed_deny):
    raise ProviderStaticContractMismatch
```

实际实现必须保留稳定顺序，不要求用无序 set 直接输出。

核心原则：

> **A-SWE may narrow operator authority; it must never widen it.**

同样：

```text
effective max_turns
→ min(operator limit, node budget)

effective timeout
→ min(operator limit, node budget)
```

A-SWE 不应通过 Node policy 抬高 operator 已配置的 execution ceiling。

### 10.6 Skill Contract

Provider binding 中的：

```text
preferred_skills
```

一期表示：

> 希望该 Skill 对此 Capability 可发现 / 可激活。

不是：

> 该 Skill 必须被实际加载后任务才算完成。

因此 Skill 缺失默认：

```text
warning / lower preference
```

而不是 hard feasibility failure。

如果未来确实需要：

```text
MUST_ACTIVATE skill X
```

必须单独定义 SkillRequirement，并验证 runtime skill-usage evidence；不能用 `enabled_skills` 冒充 activation 证据。

#### Preferred Skill Compatibility

Pinned DeerFlow 中，Skill 只有在 slash / in-context 激活后才施加 `allowed-tools` policy；但一旦生效，它会动态收窄 model-visible tools 与实际 tool calls。

因此 preferred skill 也不能只检查“存在”。例如：

```text
Node required tool = bash
preferred skill allowed-tools = [read_file]
```

该 Skill 一旦单独激活，就可能让 Node 失去完成 Capability 所需的 `bash`。

P1 采用保守规则：

- `allowed_tools is None`：兼容，表示 legacy allow-all；
- explicit `allowed_tools` 覆盖 Node 全部 required business tool exposed names：兼容；
- explicit `allowed_tools` 缺任一 required business tool：不向该 Node 暴露该 preferred skill，并记录 warning。

DeerFlow framework-always-available names（如 `read_file` / `describe_skill` / `tool_search`）仍由 DeerFlow 自身 Skill policy 处理。

注意：这只是 discoverability compatibility，不证明 Skill 实际被激活。

### 10.7 Model Policy

P1 的 AgentProvider 不自行指定高低模型，也不做 provider-level model routing。

使用：

```text
operator-configured Subagent model
or
inherit runtime/default model
```

这样避免：

```text
Capability Resolver
→ 顺便变成 Model Router
```

当前 pinned DeerFlow 有一个必须记录的 direct-executor compatibility boundary：

- Lead Agent / DeerFlowClient 会显式执行 `model:use` authorization；
- `SubagentExecutor` 自身直接 `create_chat_model(...)`；
- `create_chat_model()` 不负责 model authorization。

因此 A-SWE DeerFlow Adapter 在 authorization-enabled deployment 下必须进行：

```text
Resolved Model
      ↓
Model Authorization Preflight
      ↓
SubagentExecutor
```

P1 不复制 Lead Agent 的“deny → fallback to another model”私有逻辑，也不依赖 private `_authorize_model_name` 作为核心接口。

A-SWE 采用更严格、可解释的策略：

```text
model:use ALLOW
→ execute

model:use DENY
→ PROVIDER_MODEL_UNAUTHORIZED
→ provider infeasible / node admission failure
```

provider error：

```text
authorization.fail_closed = true
→ fail closed

authorization.fail_closed = false
→ preserve DeerFlow fail-open policy
```

实现只复用 DeerFlow 已有 lower-level authorization primitives：

- `AuthorizationProvider`；
- `AuthzRequest(resource="model", action="use")`；
- `build_principal_from_context`；
- async provider resolution pattern。

这样不复制 RBAC policy engine，也不会在 A-SWE 背后静默替用户换模型。

### 10.8 Provider Contract Fingerprint

A-SWE 自己维护：

```text
ProviderContractFingerprint
```

绑定：

- provider id / role；
- backend agent type；
- capability bindings；
- required / optional tools；
- preferred skills；
- model policy；
- runtime ceilings。

最终 Trace 关联：

```text
ProviderContractFingerprint
        ↓
NodeExecutionPolicyFingerprint
        ↓
DeerFlowAssemblyFingerprint
```

从而区分：

> 我声明了什么、我编译了什么、Backend 实际装配了什么。

### 10.9 DeerFlow 映射

执行时：

```text
AgentProvider
     │
     ▼
CapabilityBinding
     │
     ▼
NodeExecutionPolicy
     │
     ▼
DeerFlowExecutionBackend
     │
     ├─ load operator SubagentConfig
     ├─ monotonic narrowing
     ├─ model authorization preflight
     └─ backend inventory / identity context
             │
             ▼
      Effective SubagentConfig
             │
             ▼
       SubagentExecutor
```

A-SWE Agent Registry 不实例化 LangGraph Agent，也不管理 Sandbox。

### 10.10 Agent Registry 不负责

Agent Registry 不负责：

- DAG Scheduling；
- Workspace Lock；
- Backend resource discovery；
- Runtime authorization；
- Sandbox 生命周期；
- Tool 实际执行；
- Skill 文件加载；
- MCP Session；
- Evaluation。

这些职责分别归 Resolver / Backend Adapter / Scheduler / Workspace Runtime / Evaluation。

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

Skill 在 P1 是 Provider 的 optional execution enhancement。

例如：

```text
Capability: python_debugging

Coder CapabilityBinding
+
preferred skill: python-debugging
```

或者：

```text
Capability: regression_testing

Tester CapabilityBinding
+
preferred skill: pytest
```

Adapter 可以将 Node 相关 Skill 收窄成 DeerFlow `SubagentConfig.skills` discoverability allowlist。

但：

> **allowlisted ≠ loaded ≠ used。**

### 11.4 Progressive Loading

继续复用 DeerFlow Harness 已有 Progressive Loading：

```text
Skill Metadata
      ↓
Discovery
      ↓
Model chooses to load / activate
      ↓
SKILL.md
      ↓
Reference / Script on Demand
```

A-SWE 不自行实现 Skill Loader。

### 11.5 DeerFlow Direct Subagent 的 Skill Boundary

DeerFlow 标准设计中，Subagent 的 `skills` 配置主要限制：

- Skill discovery；
- Skill activation；
- active skill 的 allowed-tools policy。

它不是：

```text
eager skill loading list
```

也不是：

```text
per-subagent filesystem isolation declaration
```

A-SWE 一期直接使用 `SubagentExecutor`，因此必须明确：

> **Phase 1 的 Skill allowlist 是 discoverability / activation policy，不宣称为每个 Subagent 独立的 filesystem security boundary。**

### 11.6 Skill Availability 与 Skill Usage

DeerFlow `AgentAssemblyDescriptor.enabled_skills` 只能证明：

> 该 Skill 在 assembly 时处于可发现的 enabled set。

不能证明：

> Agent 真正加载 / 激活 / 采用了该 Skill。

DeerFlow 对显式 skill activation 存在 server-owned usage metadata：

```text
skill_usage
skill_usages
```

并包含 skill path / content hash / activation mode 等 evidence。

但 P1 不把 Skill activation 设为普通 Node 的 hard feasibility requirement。

因此：

```text
preferred skill missing
→ warning

preferred skill enabled but unused
→ normal

hard MUST_ACTIVATE skill
→ P1 默认不支持
```

后续若增加 hard skill usage contract：

```text
SkillRequirement(MUST_ACTIVATE)
      ↓
Runtime skill-usage evidence
      ↓
content hash / activation provenance
```

而不是：

```text
enabled_skills contains X
→ claim X was used
```

### 11.7 一期 Skills

一期只建设或复用少量真正能提升执行质量的 Skill：

```text
repository-navigation
python-debugging
pytest
code-review
git
```

不以 Skill 数量作为项目复杂度指标。

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
│   ├── provenance.py
│   ├── constraint_registry.py
│   ├── constraint_compiler.py
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
│   ├── bindings.py
│   ├── tool_effects.py
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
│   ├── registry.py
│   ├── provider.py
│   └── contract.py
│
├── skills/
│   ├── repository-navigation/
│   ├── python-debugging/
│   ├── pytest/
│   └── code-review/
│
├── evaluation/
│   ├── evaluator.py
│   ├── contract_evaluator.py
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
│       ├── inventory.py
│       ├── preflight.py
│       ├── preparation.py
│       ├── snapshot_store.py
│       ├── model_auth.py
│       ├── node_policy_store.py
│       ├── node_policy_outcome.py
│       ├── node_tool_policy.py
│       ├── config_mapper.py
│       ├── assembly_attestation.py
│       ├── attestation_extension.py
│       ├── result_mapper.py
│       ├── acceptance_adapter.py
│       ├── contract_guardrail.py
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

### 17.3.1 Provider Feasibility Ladder

A-SWE 不把 Registry 声明当成 execution truth。

```text
DECLARED
   ↓
PREFLIGHT_FEASIBLE
   ↓
ASSEMBLED_MATCH
   ↓
EXECUTED
   ↓
ACCEPTED / EVALUATED
```

这一分层让系统可以解释：

> 一个 Agent“理论上会做”、当前部署“看起来能做”、运行时“实际拿到了什么”、以及最终“有没有做成”，是四个不同问题。

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
| POC-21 | Contract Guard：write_file 越界 path | tool 执行前 fail-closed deny |
| POC-22 | Contract Guard：str_replace forbidden manifest | tool 执行前 deny |
| POC-23 | Tester + bash | Node 被编译为 WRITE / exclusive |
| POC-24 | Verification bash 产生 source mutation | post-node Git invariant 能检测并 fail |
| POC-25 | Guard provider error under A-SWE constrained run | fail closed |
| POC-26 | ordinary DeerFlow non-A-SWE run | A-SWE Node policy 不误作用于无关 run |

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
- 旧的全局 READ→WRITE 默认顺序被废弃；
- WorkKind 与 WorkspaceAccess 分离；
- DAG ordering 优先遵守 DISCOVERY→IMPLEMENTATION→VERIFICATION→REVIEW 语义 phase；
- Reviewer READ 不得因为 access class 被错误排到 Tester/Writer 前；
- same-phase conflict 才使用 stable planner ordinal 做保守串行；
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

P0-3 新增 deterministic compiler PoC：

| PoC | 测试内容 | 必须验证 |
|---|---|---|
| POC-P3-01 | Coder WRITE + Tester bash WRITE | IMPLEMENTATION → VERIFICATION |
| POC-P3-02 | Tester WRITE + Reviewer READ | VERIFICATION → REVIEW，而不是 READ → WRITE |
| POC-P3-03 | Explorer READ + Coder WRITE | DISCOVERY → IMPLEMENTATION |
| POC-P3-04 | 两个同 phase WRITE 且无 edge | stable ordinal serialization |
| POC-P3-05 | Planner 显式 REVIEW → IMPLEMENTATION | PHASE_ORDER_CONTRADICTION |
| POC-P3-06 | 两个独立 DISCOVERY READ | 保持并行 |
| POC-P3-07 | mandatory verification/review 缺失 | Runtime 注入 reserved gate nodes |
| POC-P3-08 | Planner 使用 __aswe_ id | hard reject |
| POC-P3-09 | read-only analysis task + planner code_modification hint | CAPABILITY_AUTHORITY_VIOLATION |
| POC-P3-10 | bug-fix contract grants repository mutation | implementation capability 可通过 authority validation |
| POC-P3-11 | Tester bash modifies source | 虽有 WRITE lock但 post-node business-mutation invariant fail |
| POC-P3-12 | mutation WorkItem 无 mutation deliverable coverage | PLAN_INVALID |

下一步继续审计：

```text
Coverage Validation
+
Runtime Gate Compilation
+
Canonical Plan Fingerprint
```

的 deterministic rule set、repair boundary 与 bounded replan policy。

---

#### P0-4：TaskContract / Constraint Compiler Audit

状态：

```text
Architecture Audited
Implementation PoC Pending
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
- provenance authenticity 与 instruction authority 分离；
- compiler-owned provenance stamping；
- `LOCKED / HARD / SOFT` enforcement；
- repository guidance 默认 SOFT 且禁止 self-promotion；
- typed merge algebra；
- user/runtime/repository conflict matrix；
- unknown explicit requirement → semantic.requirement，不静默丢弃；
- Contract → Planner coverage → Execution Policy → Final Evaluation 闭环；
- undecidable contract leaf → UNVERIFIED，不自动视为满足。
- Contract constraint 必须声明 enforcement phase 与 guarantee；
- DeerFlow GuardrailMiddleware 可复用为 PRE_TOOL_GUARD；
- structured file tool path constraint 可执行前拦截；
- generic bash 不具备 read-only guarantee；
- WorkspaceAccess 必须综合 Capability + ToolEffect；
- P1 仅将“可证明只读”的 Node 编译为 READ；
- 使用 bash 的 Tester 默认 WRITE / exclusive；
- sensitive constraint 采用 plan + pre-tool + post-node + final evaluation defense-in-depth。

---

#### P0-5：Capability / Provider Contract Audit

状态：

```text
Architecture Audited
Implementation PoC Pending
```

冻结结论：

- Capability Registry 只保存 semantic vocabulary；
- P1 只有 AgentProvider 是 schedulable execution provider；
- Tool / Skill 是 AgentProvider 的 supporting resources；
- concrete Tool requirement 从 Provider CapabilityBinding 产生，而不是写死在 CapabilitySpec；
- Provider Registry 是 capability mapping 的 single source of truth；
- BackendInventorySnapshot 只提供 compile-time candidate resource preflight；
- `SubagentConfig.tools` 是 name-level allowlist，不是 actual-tool proof；
- A-SWE Node policy 只能 intersection / union 式继续收窄 operator SubagentConfig；
- A-SWE 不允许通过 Node policy 抬高 operator max_turns / timeout；
- Skill allowlist 只代表 discovery / activation scope，enabled skill 不等于 skill was used；
- Skill 在 P1 默认 optional enhancement，不参与 hard feasibility；
- DeerFlow AssemblyDescriptor 是 runtime evidence，不是默认存在；
- A-SWE 通过公开 AgentAssemblyObserver extension 开启 descriptor generation；
- AssemblyAttestation 是 compatibility / validity gate，不替代 authorization；
- direct SubagentExecutor 的 model authorization 需要 Adapter 显式 preflight；
- P1 对 denied model 使用 strict provider-infeasible 语义，不静默 fallback；
- `SubagentConfig.tools` 不能覆盖 generated / middleware-declared tool 的全部 visibility；
- packaged extension middleware 是 observational，不能承担 Node enforcement；
- strict Node tool visibility 使用 trusted `extensions.middlewares` + ASWENodeToolPolicyMiddleware；
- NodePolicyStore 用 unique A-SWE run_id 做进程内短生命周期 correlation；
- P1 明确限制为 single-process policy carrier；distributed carrier 后置。
- required hard Tool dependency 一期必须是 eager；deferred MCP 仅作为 optional enhancement；
- NodeToolPolicy 在每次 model call 重新验证 required eager tool availability；
- required-tool mismatch 使用 synthetic marked ModelResponse short-circuit，且不调用 upstream LLM handler；
- `deerflow_error_fallback` 负责复用 DeerFlow FAILED terminalization，A-SWE marker 负责精确 failure mapping；
- AssemblyDescriptor 定位为 attestation/reproducibility evidence，不作为 admission gate。
- `PREFLIGHT_FEASIBLE` 只是 planning-time 判断；NodeReady 后必须 live revalidate；
- DeerFlow execution 使用单一 AppConfig / Tool / Extension snapshot，避免 execution 内 config TOCTOU；
- fingerprint drift 本身不是 failure，重新验证 requirement 失败才是 `BACKEND_PREFLIGHT_STALE`；
- runtime 只允许 monotonic narrowing，不允许隐式扩权或 Provider rebinding；
- authorization 保持 live gate，不伪造 frozen policy snapshot。
- DeerFlow native Subagent 是 one-shot / no-checkpoint execution；
- Provider reuse 不等于 Agent session reuse；
- Semantic NodeBoundaryPolicy 必须 provider-neutral；
- Provider Assignment 不允许反向合并 / 拆分 WorkItem；
- cross-node context 只通过 explicit NodeHandoff / shared Workspace evidence 传递。

新增 PoC：

| PoC | 测试内容 | 必须验证 |
|---|---|---|
| POC-27 | operator tools ∩ A-SWE Node tools | A-SWE 不会扩大 SubagentConfig 权限 |
| POC-28 | Provider 声明 required tool 但 backend inventory 缺失 | preflight fail，不进入 Team |
| POC-29 | 安装 A-SWE assembly observer | executor.assembly_descriptor 非空且 fingerprint 可读取 |
| POC-30 | runtime policy 在首轮前移除 required eager tool | NodeToolPolicy synthetic short-circuit；LLM provider 零调用；SubagentResult=FAILED |
| POC-31 | preferred skill enabled 但未 activation | 不误报 skill-used，也不把 Node 判失败 |
| POC-32 | authorization deny resolved model | direct-executor Adapter 在 LLM 调用前 strict fail，不静默 fallback |
| POC-33 | ordinary DeerFlow run | A-SWE attestation extension 不改变普通 Agent execution semantics |
| POC-34 | middleware-declared tool 不在 Node allowlist | model-visible schema 被 ASWENodeToolPolicyMiddleware 移除 |
| POC-35 | unauthorized tool call 绕过 model visibility | tool-call boundary 再次 deny |
| POC-36 | generated tool_search / describe_skill | 只允许 adapter-classified infrastructure helper |
| POC-37 | A-SWE managed run policy store miss | fail closed |
| POC-38 | Node cancellation / timeout | NodePolicyStore entry 一定 cleanup |
| POC-39 | optional bash 未选择 | READ Node 不因 Provider optional declaration 被升级 WRITE |
| POC-40 | optional bash 被选择 | WorkspaceAccess 自动升级 WRITE 且 fingerprint 改变 |
| POC-41 | required tool 被标记 deferred MCP | P1 compile-time reject：hard required tool 必须 eager |
| POC-42 | Skill activation 后收窄掉 required tool | 下一次 model call synthetic short-circuit；LLM 不再调用 |
| POC-43 | Node admission mismatch | terminal AIMessage 带 A-SWE marker；SubagentResult=FAILED；Adapter 精确映射 PROVIDER_ASSEMBLY_MISMATCH |
| POC-44 | NodeToolPolicy ordinary run store miss | pass-through；普通 DeerFlow 不受影响 |
| POC-45 | planning 后新增无关 Tool | inventory drift 被记录，但 Node 仍可 PREPARED |
| POC-46 | planning 后 required Tool 被移除 | prepare_node 返回 BACKEND_PREFLIGHT_STALE，LLM/Workspace 零副作用 |
| POC-47 | operator timeout/max_turns 收紧但仍可满足 | effective policy 单调收窄 |
| POC-48 | prepare 后 config 热更新 | execute_prepared 仍使用 prepare 阶段 pinned AppConfig snapshot |
| POC-49 | live authorization 在 execution 中改变 | runtime auth gate 生效，不依赖 stale planning verdict |
| POC-50 | Provider stale 但另一个 Provider 可用 | P1 不隐式切换，显式 fail/replan |
| POC-51 | 同一 Coder Provider 顺序执行两个 Node | 第二个 Node 不自动继承第一个模型上下文 |
| POC-52 | Node B 依赖 Node A | 只有显式 NodeHandoff / Workspace evidence 进入 B |
| POC-53 | 单 Provider 覆盖全部 WorkItem | TeamSpec 单成员，但 DAG Node 数与语义边界保持不变 |

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
- Constraint Provenance Validation；
- Constraint Registry / Merge Algebra；
- ConstraintCompiler；
- CompiledTaskContract / fingerprint；
- Contract coverage validation；
- TaskExecutionAuthority projection；
- Capability authority validation；
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

- provider-neutral CapabilitySpec；
- CapabilityAuthorityClass / workspace_effect_floor；
- CapabilityBinding；
- Node-Level Capability Resolver；
- Agent Registry；
- AgentProvider / ProviderContract；
- BackendInventorySnapshot；
- Provider preflight feasibility；
- ProviderContract fingerprint；
- Minimal Feasible Team Policy；
- TeamSpec（roster only）；
- NodeHandoff schema；
- DAG Materializer；
- TaskNode；
- ToolEffect Registry；
- Capability + Tool Effect workspace access compiler；
- deterministic workspace-conflict serialization。

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
- NodeExecutionPreparation；
- live Backend revalidation；
- Backend snapshot pinning；
- `BACKEND_PREFLIGHT_STALE` classification；
- Dependency Handoff Routing；
- READ / WRITE Workspace Access；
- provably READ-only Basic Parallel Execution；
- WRITE / UNKNOWN-MUTATING Exclusive Execution；
- ContractGuardrailProvider integration；
- Pre-tool constraint guard；
- monotonic SubagentConfig narrowing；
- identity-aware Tool / Model authorization preflight；
- NodePolicyStore；
- trusted ASWENodeToolPolicyMiddleware；
- first-model / every-model required-tool admission gate；
- synthetic policy-failure ModelResponse short-circuit；
- A-SWE failure marker → structured failure mapping；
- final model-visible tool filtering；
- tool-call name-level deny backstop；
- A-SWE AgentAssemblyObserver extension；
- DeerFlow AssemblyAttestation；
- PROVIDER_ASSEMBLY_MISMATCH classification；
- Post-node Git-aware contract invariant；
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
- TaskContractEvaluator / ContractVerdict；
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
22. 为什么 CapabilitySpec 不直接保存 required_tools / eligible_agents？
23. CapabilityBinding 为什么属于 Provider，而不是 Capability？
24. Backend Inventory 与 DeerFlow Assembly Attestation 有什么区别？
25. 为什么 SubagentConfig.tools 不能当作实际 Tool 可用性证明？
26. A-SWE 如何保证 Node policy 不扩大 operator SubagentConfig 权限？
27. enabled skill 与 skill activation / usage 有什么区别？
28. 为什么 AssemblyObserver 不能作为 execution safety gate？
29. direct SubagentExecutor 的 model authorization 边界是什么？
30. 为什么 SubagentConfig.tools 不是最终 schema allowlist？
31. packaged Extension middleware 与 trusted extensions.middlewares 有什么权限差异？
32. 为什么 NodeToolPolicy 与 ContractGuardrail 要分开？
33. 为什么 WorkspaceAccess 要按最终 allowed tools，而不是 required tools 计算？
34. NodePolicyStore 为什么一期只承诺 single-process？
35. 为什么 planning-time preflight 之后还需要 prepare_node？
36. Backend inventory fingerprint 变化为什么不应该自动判失败？
37. 为什么 authorization 不能被当成 frozen snapshot？
38. 为什么 P1 不在 runtime 自动切换另一个 Provider？
39. DeerFlow 的 one-AppConfig-snapshot 设计如何减少 TOCTOU？

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
BackendInventory / ProviderPreflight
SubagentConfig Narrowing
AssemblyAttestation
ContractGuardrailProvider
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
Task → Contract → Capability → Provider Preflight → Team → DAG → Scheduling → Evaluation

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
