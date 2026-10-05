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

    # repository identity
    repo_url: str | None = None
    base_ref: str | None = None

    # runtime workspace
    workspace_root: str
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

Repository clone / checkout / base ref 固定等 Workspace Bootstrap 细节在正式实现前继续进行 P0 审计；当前架构先固定 `WorkspaceSession` 作为上层接口。

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

## 4. Task Understanding：任务理解模块

### 4.1 模块目标

Task Analyzer 将自然语言形式的软件工程需求转换为结构化 `TaskSpec`。

系统接收到任务后，不应直接进入 Coding，而需要先判断：

- 任务类型；
- 技术栈；
- Repository 影响范围；
- 是否需要代码修改；
- 是否需要测试；
- 是否存在较高 Regression Risk；
- 是否需要 Repository Exploration；
- 是否需要 Reviewer；
- 任务复杂度；
- 所需 Capability。

### 4.2 TaskSpec 数据结构

示例任务：

> Flask 项目在并发请求下偶发出现数据库连接泄漏，请定位问题，修复并添加 regression test。

结构化输出：

```yaml
task:
  type: bug_fix
  description: database connection leak under concurrent requests

scope:
  repository_level: true
  expected_files: unknown

domains:
  - python
  - flask
  - database
  - concurrency

requirements:
  investigation: high
  coding: medium
  testing: high
  review: true

risk:
  level: high
  regression: high

capabilities:
  required:
    - repo_exploration
    - python_debugging
    - database_analysis
    - code_modification
    - regression_testing

estimated_complexity: medium
```

### 4.3 实现方式

一期采用：

```text
LLM Structured Output
        +
Rule Validation
        +
Lightweight Heuristics
```

建议使用严格 Schema：

```python
class TaskSpec(BaseModel):
    task_type: TaskType
    domains: list[str]
    repository_level: bool
    complexity: Complexity
    risk: RiskLevel
    required_capabilities: list[str]
    testing_required: bool
    review_required: bool
```

### 4.4 Rule Validation

LLM 输出不能直接作为最终调度依据，需要执行规则修正。

例如：

```text
如果 task_type == bug_fix
→ 至少需要 code_modification

如果 repository_level == true
→ 默认加入 repo_exploration

如果 risk == high
→ 强制 review_required = true

如果用户明确要求 regression test
→ 强制加入 regression_testing
```

---

## 5. Capability Registry：能力注册中心

### 5.1 核心设计原则

A-SWE Runtime 不将 Agent 作为最基本调度单位，而将：

> **Capability**

作为 Runtime 的语义调度单位。

Capability 表示：

> **完成任务所需要的“能力”。**

例如：

```text
repo_exploration
code_search
python_debugging
database_analysis
code_modification
test_generation
regression_testing
code_review
```

Agent、Skill 和 Tool 是 Capability Provider。

### 5.2 Capability 与 Capability Provider

推荐数据模型：

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

- Capability：任务需要做什么；
- Agent：谁来执行；
- Skill：执行时应加载哪些领域知识 / SOP；
- Tool：真正执行外部动作；
- MCP：Tool 的一种外部接入来源，而不是 Capability 本身。

### 5.3 Capability 定义示例

```yaml
capability:
  id: repo_exploration
  description: understand repository structure and locate relevant code

providers:
  agents:
    - repo_explorer

  skills:
    - repository_navigation

  tools:
    - read_file
    - search_code
    - list_tree

  mcp_tools:
    - github.search_code
```

### 5.4 与 Plugin System 的边界

Capability-Centric Runtime 与“Everything is a Plugin”不是同一层概念。

```text
Plugin System
解决：系统组件如何注册、加载、替换、组合

Capability System
解决：当前任务需要什么能力，以及应该选哪些 Provider
```

可以理解为：

```text
Infrastructure Layer
Plugin / Registry / Extension
          │
          ▼
Runtime Semantic Layer
Capability
          │
          ▼
Provider Selection
```

因此 A-SWE 不应重新实现一套底层 Plugin Framework。

若底层 Harness 已经支持 Plugin / Tool / Skill 注册机制，A-SWE 仅维护 Capability 到 Provider 的语义映射。

### 5.5 一期 Capability 范围

一期建议只支持：

```text
repo_exploration
code_search
bug_diagnosis
code_modification
test_generation
regression_testing
code_review
git_operation
```

必要时再增加：

```text
database_analysis
architecture_analysis
```

---

## 6. Capability Resolver：能力解析器

### 6.1 模块职责

Capability Resolver 输入：

```text
TaskSpec
+
Capability Registry
+
Available Providers
```

输出：

```text
ResolvedCapabilities
```

主要解决：

> 每一个 Required Capability 应由哪些 Agent、Skill 和 Tool 提供。

### 6.2 解析示例

任务要求：

```text
repo_exploration
python_debugging
code_modification
regression_testing
```

Resolver 输出：

```yaml
resolved_capabilities:

  repo_exploration:
    agents:
      - repo_explorer
    skills:
      - repository_navigation
    tools:
      - search_code
      - read_file

  python_debugging:
    agents:
      - coder
    skills:
      - python_debugging

  code_modification:
    agents:
      - coder

  regression_testing:
    agents:
      - tester
    skills:
      - pytest
    tools:
      - shell
```

### 6.3 Provider Selection

一期使用简单、可解释的 Provider Selection：

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
Compatibility Filter
        │
        ▼
Priority / Cost Rule
        │
        ▼
Resolved Provider
```

不需要一期就实现复杂学习模型。

### 6.4 核心价值

Capability Resolver 将：

```text
Task Requirement
```

与：

```text
具体 Agent / Skill / Tool
```

解耦。

这使 Dynamic Team Builder 不需要直接理解所有底层工具细节。

---

## 7. Dynamic Team Builder：动态团队构建模块

### 7.1 建设目标

Dynamic Team Builder 根据：

- TaskSpec；
- ResolvedCapabilities；
- Complexity；
- Risk；
- Repository Scope；
- Testing Requirement；
- Review Requirement；

动态生成 `TeamSpec`。

### 7.2 核心原则

Multi-Agent 不是默认选择。

> **能够由一个 Agent 完成的任务，不应为了“Multi-Agent”而强行创建多个 Agent。**

### 7.3 示例拓扑

#### 简单任务

任务：

> 修改 README 中的一处描述错误。

```text
Coder
```

#### 普通 Bug

```text
Explorer
   ↓
Coder
   ↓
Tester
```

#### 高风险 Bug

```text
Explorer
   ↓
Coder
   ↓
Tester
   ↓
Reviewer
```

#### Repository-Level 重构

```text
          Explorer
         /       \
 Explorer         Explorer
         \       /
           Coder
             ↓
           Tester
             ↓
          Reviewer
```

### 7.4 TeamSpec

```yaml
team:
  strategy: explore_code_test_review

  agents:
    - id: explorer_1
      role: explorer

    - id: coder_1
      role: coder

    - id: tester_1
      role: tester

    - id: reviewer_1
      role: reviewer

  constraints:
    testing_required: true
    review_required: true
```

### 7.5 动态团队的核心解释能力

Team Builder 除了输出“选了谁”，还必须输出：

```text
为什么选
为什么不选
```

例如：

```yaml
selection_reason:
  explorer:
    selected: true
    reason: repository_level_task

  tester:
    selected: true
    reason: regression_test_required

  reviewer:
    selected: true
    reason: high_risk_change

  architect:
    selected: false
    reason: architecture_change_not_detected
```

该信息后续直接进入 Execution Trace。

---

## 8. MVP Team Selection Policy：最小可行团队策略

### 8.1 为什么一期不使用复杂 Cost Utility

理论上可以将团队选择定义为：

```text
success probability
-
token cost
-
latency
-
coordination cost
```

但一期没有足够历史数据可靠估计：

```text
P(success | task, team)
```

因此一期不构造形式上复杂但缺乏数据支撑的成功率模型。

### 8.2 一期选择目标

一期 Team Builder 使用：

> **满足 Capability Coverage 与风险约束前提下，选择最小可行 Team。**

形式化表达：

```text
Minimize:
    Team Complexity / Estimated Execution Cost

Subject to:
    CapabilityCoverage == 100%
    RiskConstraints == satisfied
    TaskConstraints == satisfied
```

### 8.3 规则示例

```text
规则 1：
单文件低风险修改
→ Coder

规则 2：
Repository-Level Task
→ Explorer 必选

规则 3：
需要测试
→ Tester 必选

规则 4：
High Risk
→ Reviewer 必选

规则 5：
明显跨模块重构
→ 允许多个 Explorer 并行

规则 6：
如果 Coder 已覆盖简单代码检索能力
且任务规模很小
→ 不额外创建 Explorer
```

### 8.4 后续演进

当系统积累历史执行数据后，可升级为：

```text
Task Features
     +
Historical Execution Metrics
     ↓
Team Ranking
     ↓
Selected Team
```

但不属于一期必做范围。

---

## 9. Task Scheduler：任务调度模块

### 9.1 模块职责

Team Builder 决定：

> 谁参与任务。

Task Scheduler 决定：

> 这些 Agent 以什么顺序、依赖关系和并行关系执行，以及它们能否安全地同时访问同一个 Workspace。

Scheduler 将任务转换为 Task DAG，同时维护 Workspace side-effect constraint。

### 9.2 示例 DAG

数据库连接泄漏任务：

```text
                 Inspect Repository
                        │
          ┌─────────────┴─────────────┐
          ▼                           ▼
 Locate DB Layer                Inspect Tests
      READ                          READ
          │                           │
          ▼                           │
Analyze Connection Lifecycle         │
      READ                            │
          │                           │
          └─────────────┬─────────────┘
                        ▼
                    Root Cause
                       READ
                        │
                        ▼
                     Implement
                       WRITE
                        │
                        ▼
                  Regression Test
                       READ
                        │
                        ▼
                      Review
                       READ
```

其中两个探索节点均为只读，可以并行执行；代码修改节点具有 WRITE 副作用，需要独占共享 Workspace。

### 9.3 TaskNode

建议统一定义：

```python
class TaskNode(BaseModel):
    id: str
    type: str
    description: str

    assigned_agent: str | None
    required_capabilities: list[str]
    dependencies: list[str]

    workspace_access: WorkspaceAccess

    # Phase 1 optional; default unknown.
    affected_paths: list[str] | None = None

    acceptance_criteria: list[str] = []

    status: NodeStatus
    retry_policy: RetryPolicy
```

一期：

```python
class WorkspaceAccess(str, Enum):
    READ = "read"
    WRITE = "write"
```

### 9.4 Workspace 并发安全规则

DeerFlow 可以解决共享 Sandbox 的生命周期、lease 与 shell scope，但不会替 A-SWE 判断两个业务节点能否安全并行修改同一份 Repository。

因此 A-SWE Scheduler 必须显式承担共享 Workspace 冲突控制。

一期采用保守策略：

| Node A | Node B | 是否允许并行 |
|---|---|---:|
| READ | READ | 是 |
| READ | WRITE | 否 |
| WRITE | READ | 否 |
| WRITE | WRITE | 否 |

即：

> **MVP 只允许 READ / READ 并行，任何 WRITE 节点都获得 Workspace 独占执行权。**

这种策略会牺牲部分并行度，但能优先保证 Repository 状态一致性。

### 9.5 后续细粒度并行

二期以后可增加：

```python
affected_paths = [
    "src/auth/**"
]
```

若两个 WRITE Node 的作用域可证明不相交，可允许并行：

```text
Coder A → src/auth/**
Coder B → docs/**
```

但一期不实现复杂路径冲突预测、Git worktree fan-out 或自动 merge。

### 9.6 Scheduler 执行职责

一期 Scheduler 必须支持：

- Task decomposition；
- Dependency DAG；
- Sequential execution；
- READ-only basic parallel execution；
- Agent assignment；
- Node status tracking；
- Workspace access arbitration；
- Retry；
- Failure propagation；
- Cancellation；
- Result aggregation；
- Acceptance gate。

### 9.7 Node 执行流程

单节点标准流程：

```text
NodeReady
   │
   ▼
WorkspaceAccessCheck
   │
   ▼
ExecutionBackend.execute_node()
   │
   ▼
NodeExecutionResult
   │
   ▼
Acceptance Check
   │
   ├── holds → NodeCompleted
   │
   ├── unverified → policy decision
   │
   └── failed → retry / fail
   │
   ▼
Release Workspace Access
```

### 9.8 一期不追求

暂不做：

- complex distributed scheduler；
- dynamic worker autoscaling；
- RL scheduling；
- large-scale Agent swarm；
- parallel write merge；
- cross-machine distributed workspace locking。

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

A-SWE Runtime 一期完整执行链路：

```text
User SWE Task
      │
      ▼
Task Analyzer
      │
      ▼
TaskSpec
      │
      ├─────────────────────┐
      ▼                     ▼
Workspace Runtime      Capability Resolver
      │                     │
      ▼                     ▼
WorkspaceSession     ResolvedCapabilities
      │                     │
      │                     ▼
      │              Dynamic Team Builder
      │                     │
      │                     ▼
      │                  TeamSpec
      │                     │
      └──────────┬──────────┘
                 ▼
            Task Scheduler
                 │
                 ▼
              Task DAG
                 │
                 ▼
        Workspace Access Gate
                 │
                 ▼
      DeerFlowExecutionBackend
                 │
                 ▼
         SubagentExecutor
                 │
                 ▼
         SubagentResult
                 │
        ┌────────┴─────────┐
        ▼                  ▼
Node Acceptance       Execution Evidence
        │                  │
        └────────┬─────────┘
                 ▼
          Runtime Trace
                 │
                 ▼
             Evaluation
                 │
                 ▼
               Result
```

其中：

```text
A-SWE
→ Task / Capability / Team / DAG / Scheduling / Evaluation

DeerFlow
→ Subagent / Skill / Tool / MCP / Sandbox / low-level execution
```

每一个主要 Runtime Decision 均产生 A-SWE Runtime Event；底层 LLM / Tool span 通过 backend trace correlation 下钻查看。

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
│   └── validator.py
│
├── workspace/
│   ├── session.py
│   ├── manager.py
│   ├── access.py
│   └── bootstrap.py
│
├── capability/
│   ├── registry.py
│   ├── resolver.py
│   ├── provider.py
│   └── schema.py
│
├── team/
│   ├── builder.py
│   ├── policy.py
│   └── topology.py
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
│   ├── explorer.py
│   ├── coder.py
│   ├── tester.py
│   └── reviewer.py
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
Task
  ├─────────────→ Workspace
  │
  ▼
Capability
  ↓
Team
  ↓
Scheduler
  ├─────────────→ Workspace Access Policy
  │
  ▼
Execution Backend
  ↓
Evaluation
```

Observability 作为横切能力：

```text
Task ──────────────┐
Workspace ─────────┤
Capability ────────┤
Team ──────────────┤
Scheduler ─────────┼──→ RuntimeEvent → Trace Store
Execution ─────────┤
Evaluation ────────┘
```

### 16.3 依赖约束

建议：

- `task/` 不依赖具体 Agent；
- `workspace/` 不依赖 Team Builder；
- `capability/` 不依赖 DeerFlow；
- `team/` 不直接调用 Tool；
- `scheduler/` 只通过 `ExecutionBackend` 执行节点；
- `scheduler/` 负责共享 Workspace 的并发正确性；
- `observability/` 不参与业务决策；
- `evaluation/` 不直接创建 Agent；
- 所有 DeerFlow-specific 逻辑集中在 `integrations/deerflow/`。

### 16.4 Anti-Corruption Layer

`integrations/deerflow/` 是 A-SWE 与 DeerFlow 之间的 Anti-Corruption Layer。

它负责把：

```text
A-SWE TaskNode
A-SWE AgentProvider
A-SWE WorkspaceSession
```

转换成：

```text
DeerFlow SubagentConfig
DeerFlow SubagentExecutor inputs
```

并把：

```text
SubagentResult
AcceptanceVerdict
backend trace id
```

转换回：

```text
NodeExecutionResult
NodeAcceptanceResult
```

A-SWE 核心代码禁止直接散落 `deerflow.*` import。

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
git
code search
```

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

一期明确支持：

```text
LocalSandboxProvider
or
AioSandboxProvider + LocalContainerBackend
```

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

在开始实际 Repository-Level Demo 前继续确认：

```text
Repository Source
      ↓
WorkspaceSession creation
      ↓
clone / checkout / base ref freeze
      ↓
/mnt/user-data/workspace
      ↓
Task DAG execution
      ↓
git diff / patch result
```

该阶段的目标是冻结 Repo ingress / base revision / final diff 的工程边界，而不是扩展 Agent 功能。

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

#### P1-2：Task → Capability

完成：

- TaskSpec；
- Task Analyzer；
- Rule Validation；
- Capability Registry；
- Capability Resolver。

目标：

```text
Natural Language Task
→
Structured Capability Requirement
```

#### P1-3：Capability → Team

完成：

- Agent Registry；
- AgentProvider；
- TeamSpec；
- Minimal Feasible Team Policy；
- Dynamic Team Builder。

目标：

```text
Different Tasks
→
Different Teams
```

#### P1-4：Team → DAG → Workspace-Aware Execution

完成：

- TaskNode；
- DAG；
- Scheduler；
- READ / WRITE Workspace Access；
- READ-only Basic Parallel Execution；
- WRITE Exclusive Execution；
- Retry；
- Cancellation；
- Failure Propagation；
- Result Aggregation；
- Node Acceptance Gate。

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
→ Workspace
→ Capability
→ Selected Team
→ DAG
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
