# Adaptive Agent Runtime for Software Engineering 实施方案书

> 版本：MVP 面试项目收敛版  
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

项目当前默认通过 Adapter 对接 DeerFlow Harness，复用其 LangGraph Runtime、基础 Subagent、Skills、MCP、Sandbox、Context Management 等基础能力；A-SWE Runtime 自身保持独立，核心新增能力集中在任务理解、Capability Resolution、动态团队构建、任务调度、Execution Trace 和 Evaluation。

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
3. 识别任务所需 Capability；
4. 从 Agent、Skill、Tool、MCP Tool 中解析 Capability Provider；
5. 根据任务复杂度、风险和能力覆盖情况构建最小可行 Agent Team；
6. 生成 Task DAG；
7. 调度 Agent、Skill 与 Tool 执行；
8. 记录完整 Execution Trace；
9. 对 Patch、测试结果和执行结果进行自动 Evaluation；
10. 输出结构化任务结果与运行指标。

### 2.2 一期建设重点

一期只聚焦六个核心能力：

```text
1. Task Analyzer
2. Capability Registry / Resolver
3. Dynamic Team Builder
4. Task Scheduler
5. Execution Trace / Observability
6. Evaluation
```

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
- 大规模 Web UI 平台化建设。

一期重点是：

> **Runtime 核心链路能够真实运行，并能够通过 3～5 个 Demo Case 清晰展示不同任务产生不同执行拓扑。**

---

## 3. 总体系统架构

A-SWE Runtime 采用“核心运行时 + 基础设施适配层”的设计。

```text
┌─────────────────────────────────────────────────────────┐
│                    A-SWE Runtime                         │
│                                                         │
│  Task Analyzer                                          │
│  Capability Registry / Resolver                         │
│  Dynamic Team Builder                                   │
│  Task Scheduler                                         │
│  Execution Trace / Observability                        │
│  Evaluation                                             │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                  Runtime Adapter                         │
│                                                         │
│              DeerFlow Adapter                           │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                 DeerFlow Harness                        │
│                                                         │
│  LangGraph Runtime                                      │
│  Subagents                                              │
│  Skills                                                 │
│  MCP                                                    │
│  Tools                                                  │
│  Sandbox                                                │
│  Context Management                                     │
│  Basic Memory                                           │
└─────────────────────────────────────────────────────────┘
```

### 3.1 A-SWE Runtime 层

A-SWE Runtime 是项目主体，负责：

- Task Understanding；
- Capability Modeling；
- Capability Provider Resolution；
- Dynamic Team Formation；
- DAG Planning；
- Agent Assignment；
- Execution Scheduling；
- Runtime Trace；
- Evaluation；
- 运行指标采集。

### 3.2 Runtime Adapter 层

Adapter 负责隔离 A-SWE Runtime 与具体 Harness 实现。

一期默认：

```text
A-SWE Runtime
      │
      ▼
DeerFlowAdapter
      │
      ▼
DeerFlow Harness
```

Adapter 建议提供统一接口，例如：

```python
class RuntimeBackend:
    async def create_agent(...): ...
    async def run_agent(...): ...
    async def execute_tool(...): ...
    async def load_skill(...): ...
    async def run_in_sandbox(...): ...
```

A-SWE 核心模块不直接依赖 DeerFlow 内部对象。

### 3.3 DeerFlow Harness 层

底层 Harness 主要复用：

- LLM Agent Runtime；
- LangGraph；
- Subagent 执行；
- Tool Calling；
- Skill Loading；
- MCP Client；
- Sandbox；
- Context Management；
- 文件与 Artifact 能力。

实施原则：

> 能通过 Adapter 使用的基础设施能力，不在 A-SWE 中重复实现。

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

> 这些 Agent 以什么顺序、依赖关系和并行关系执行。

Scheduler 将任务转换为 Task DAG。

### 9.2 示例 DAG

数据库连接泄漏任务：

```text
                 Inspect Repository
                        │
          ┌─────────────┴─────────────┐
          ▼                           ▼
 Locate DB Layer                Inspect Tests
          │                           │
          ▼                           │
Analyze Connection Lifecycle         │
          │                           │
          └─────────────┬─────────────┘
                        ▼
                    Root Cause
                        │
                        ▼
                     Implement
                        │
                        ▼
                  Regression Test
                        │
                        ▼
                      Review
```

其中：

```text
Locate DB Layer
```

和：

```text
Inspect Tests
```

可并行执行。

### 9.3 TaskNode

建议定义统一 Task Node：

```python
class TaskNode(BaseModel):
    id: str
    type: str
    description: str
    assigned_agent: str | None
    required_capabilities: list[str]
    dependencies: list[str]
    status: NodeStatus
    retry_policy: RetryPolicy
```

### 9.4 一期 Scheduler 能力

必须支持：

- Task decomposition；
- Dependency DAG；
- Sequential execution；
- Basic parallel execution；
- Agent assignment；
- Node status tracking；
- Retry；
- Failure propagation；
- Result aggregation。

### 9.5 一期不追求

暂不做：

- 复杂 distributed scheduler；
- 动态 worker autoscaling；
- RL scheduling；
- 超大规模 Agent swarm。

---

## 10. Agent Registry：智能体注册中心

### 10.1 设计定位

Capability Registry 定义：

> 系统需要 / 拥有什么能力。

Agent Registry 定义：

> 哪些执行主体能够提供这些能力。

### 10.2 一期 Agent

一期只保留：

```text
Repo Explorer
Coder
Tester
Reviewer
```

`SWE Lead` 可以作为 Runtime Coordinator 的逻辑角色，而不一定独立实现为一个长期 Agent。

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
```

### 10.4 Agent 实例化

```text
Capability Requirement
        ↓
Candidate Agent
        ↓
Metadata Match
        ↓
Team Builder Decision
        ↓
Runtime Backend Instantiate
```

Agent Registry 不直接负责调度。

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

Execution Trace 是一期核心模块，而不是附加日志系统。

其作用包括：

- 解释 Runtime 为什么做出某个决策；
- 记录 Agent、Skill、Tool 的实际执行过程；
- 为调试 Scheduler 与 Dynamic Team 提供依据；
- 为 Evaluation 提供运行数据；
- 为未来 Experience Memory / Adaptive Selection 提供原始数据。

整体关系：

```text
Runtime Decision
      │
      ▼
 Runtime Event
      │
      ▼
   Event Bus
      │
      ▼
  Trace Store
      │
      ├── Timeline
      ├── Metrics
      ├── Debugging
      └── Future Experience
```

### 13.2 Runtime Event

建议所有关键动作统一转换为 `RuntimeEvent`。

```python
class RuntimeEvent(BaseModel):
    event_id: str
    task_id: str
    event_type: str
    timestamp: datetime
    source: str
    payload: dict
```

### 13.3 一期事件类型

至少记录：

```text
TaskReceived
TaskAnalyzed

CapabilityRequired
CapabilityResolved

TeamPlanningStarted
AgentSelected
SkillSelected
ToolSelected
TeamCreated

DAGCreated
NodeStarted
NodeCompleted
NodeFailed
NodeRetried

AgentStarted
AgentCompleted
ToolCalled
ToolReturned

EvaluationStarted
EvaluationCompleted

TaskCompleted
TaskFailed
```

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

### 13.5 Trace Metrics

一期至少记录：

```text
Task Duration
Agent Count
LLM Call Count
Tool Call Count
Retry Count
Node Failure Count
Token Usage（若底层可获取）
Selected Skills
Selected Tools
Final Status
```

### 13.6 Trace UI

一期 UI 不需要复杂平台化，可以实现一个简单任务详情页：

```text
Task: Fix database connection leak
Status: Running

Task Analysis
────────────────────────
Type           bug_fix
Complexity     medium
Risk           high

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
✓ inspect repository
✓ locate DB layer
✓ analyze lifecycle
→ implement patch
○ regression test
○ review

Metrics
────────────────────────
Agents           4
LLM Calls        8
Tool Calls      27
Retries          1
```

对于面试 Demo，Trace UI 是项目展示的重要组成部分。

---

## 14. Evaluation 与 Review 体系

### 14.1 模块定位

Evaluation 用于判断：

> Agent 是否真正完成了软件工程任务。

Evaluation 不等同于正式 Benchmark。

一期不做大规模 Benchmark，但每一次任务执行必须有内部 Evaluation。

### 14.2 Evaluation Pipeline

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

### 14.3 EvaluationResult

```yaml
evaluation:
  execution_status: completed

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

### 14.4 Reviewer 关注点

Reviewer 主要检查：

- Patch 是否解决原始问题；
- 是否存在无关修改；
- 是否违反 Repository Coding Convention；
- 是否可能引入 Regression；
- 是否缺少测试；
- 是否存在明显安全 / 性能问题；
- 是否可以进一步简化。

### 14.5 一期 Evaluation 原则

优先使用：

```text
确定性检查 > LLM 自评
```

例如：

```text
pytest
build
import
lint
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
      ▼
Capability Resolver
      │
      ▼
ResolvedCapabilities
      │
      ▼
Dynamic Team Builder
      │
      ▼
TeamSpec
      │
      ▼
Task Scheduler
      │
      ▼
Task DAG
      │
      ▼
Adaptive Execution
      │
 ┌────┴───────────────┐
 │                    │
 ▼                    ▼
Runtime Events     Agent / Skill / Tool
 │                    │
 ▼                    ▼
Trace Store         Sandbox
 │                    │
 └──────────┬─────────┘
            ▼
        Evaluation
            │
            ▼
          Result
```

每一个主要阶段均产生 Runtime Event。

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
│       ├── adapter.py
│       ├── agent_adapter.py
│       ├── skill_adapter.py
│       └── tool_adapter.py
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

### 16.2 模块边界

核心依赖方向：

```text
Task
  ↓
Capability
  ↓
Team
  ↓
Scheduler
  ↓
Runtime Backend
```

Observability 作为横切能力：

```text
Task ──────────────┐
Capability ────────┤
Team ──────────────┤
Scheduler ─────────┼──→ RuntimeEvent → Trace Store
Execution ─────────┤
Evaluation ────────┘
```

### 16.3 依赖约束

建议：

- `task/` 不依赖具体 Agent；
- `capability/` 不依赖 DeerFlow；
- `team/` 不直接调用 Tool；
- `scheduler/` 只通过 Runtime Backend 执行；
- `observability/` 不参与业务决策；
- 所有 DeerFlow-specific 逻辑集中在 `integrations/deerflow/`。

这样未来理论上可替换底层 Harness。

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

### 17.4 Task DAG Scheduling

将复杂软件工程任务显式转换为 DAG，支持：

- dependency；
- parallelism；
- retry；
- failure propagation；
- result aggregation。

### 17.5 Execution Trace / Observability

不仅记录 Agent 输出，还记录 Runtime Decision：

```text
Why this capability?
Why this agent?
Why this skill?
Why multi-agent?
Why reviewer?
```

使系统具备较强可解释性和可调试性。

### 17.6 Evaluation First

任务执行完成不等于任务完成。

必须经过：

```text
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

一期只保留必要的运行态 Task Context。

暂不单独建设复杂：

```text
Repository Memory
Experience Memory
Long-term Vector Memory
```

Execution Trace 为后续 Experience Memory 提供数据基础。

### 18.6 Demo Case

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

#### Demo 4：跨模块任务

```text
Explorer × N
     ↓
   Coder
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

### Phase 1：Adaptive SWE Runtime MVP

这是当前唯一必须完成的阶段。

#### P1-1：Runtime Skeleton

完成：

- 独立 A-SWE 工程目录；
- Runtime Backend Interface；
- DeerFlow Adapter；
- Runtime State / Context。

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
- TeamSpec；
- Minimal Feasible Team Policy；
- Dynamic Team Builder。

目标：

```text
Different Tasks
→
Different Teams
```

#### P1-4：Team → DAG → Execution

完成：

- TaskNode；
- DAG；
- Scheduler；
- Basic Parallel Execution；
- Retry；
- Result Aggregation。

#### P1-5：Execution Trace

完成：

- RuntimeEvent；
- Event Bus；
- Trace Store；
- Execution Timeline；
- Metrics；
- 简单 Trace Viewer。

#### P1-6：Evaluation

完成：

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
Repository-Level Task
```

并在 README 展示：

```text
Task
→ Capability
→ Selected Team
→ DAG
→ Trace
→ Result
```

### Phase 2：Memory & Experience（非一期目标）

后续可增加：

- Repository Memory；
- Experience Memory；
- Historical Task Retrieval；
- Similar Task Matching；
- Team Execution Statistics；
- Skill Effectiveness Statistics。

核心目标：

> 让过去的任务执行数据开始影响未来决策。

### Phase 3：Cost / Performance-Aware Selection（非一期目标）

在具有足够 Trace 数据以后再增加：

- historical success rate；
- token cost；
- execution time；
- retry cost；
- coordination cost；
- Team Ranking。

此时才引入真正有数据支撑的 Cost-Aware Team Selection。

### Phase 4：Experience-Driven Adaptation（远期）

可进一步研究：

```text
Execution Trace
      ↓
Experience Extraction
      ↓
Team / Skill Statistics
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

> **一个能够根据 Software Engineering Task 动态构建执行能力和 Agent Topology 的 Runtime。**

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

强调：

> **Same runtime. Different task. Different agent topology.**

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
9. Retry 与 Failure Propagation 如何处理？
10. Execution Trace 为什么不仅仅是日志？
11. Evaluation 为什么优先使用确定性检查？
12. A-SWE Runtime 与底层 Harness 的边界在哪里？

### 21.4 代码掌握边界

需要重点掌握并能够现场解释的代码：

```text
Task Analyzer
Capability Registry / Resolver
Dynamic Team Builder
Selection Policy
Task DAG
Scheduler
Execution Trace
Evaluation
DeerFlow Adapter
```

对于底层成熟基础设施，应能够解释：

```text
如何调用
为什么复用
接口边界是什么
```

而不是将主要时间投入重复实现 Sandbox、MCP Client 或 LangGraph Engine。

---

## 22. 项目最终定位

A-SWE Runtime 最终不定义为：

> 一个拥有很多 Agent 的软件开发系统。

更准确的定义是：

> **A task-adaptive runtime for resolving software engineering capabilities, composing the minimum feasible agent team, scheduling task DAGs, and tracing/evaluating the full execution process.**

其核心执行链路为：

```text
Task
 ↓
Capability
 ↓
Team
 ↓
DAG
 ↓
Execution
 ↓
Trace
 ↓
Evaluation
```

项目核心思想可以概括为：

> **Don't build one SWE agent. Build the runtime that assembles the right execution structure for each software engineering task.**

对应中文：

> **不是构建一个固定的软件工程 Agent，而是构建一个能够针对不同软件工程任务动态组装能力、执行主体和协作拓扑的运行时。**

一期项目的成功标准不是功能数量，而是：

1. 核心链路真实运行；
2. 不同任务能够产生不同 Team / DAG；
3. Runtime Decision 能够通过 Trace 解释；
4. 执行结果能够通过 Evaluation 验证；
5. 3～5 个 Demo Case 能稳定展示整个流程。
