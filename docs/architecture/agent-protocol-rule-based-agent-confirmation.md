# 架构：Agent 协议、RuleBasedAgent 与确认机制

## 动机 / 用户故事

让 Agent 从 MS1 起贯穿主链，但以规则实现替代真实 LLM：Agent 负责提出下一步动作、解释结果、请求确认；不能直接修改状态或执行外部工具。

## 目标用户

- 主链工作流中被 Agent 引导的最终用户
- 集成 Agent 输出的 Workflow Engine
- 后续需要扩展为真实 LLM Agent 的开发者

## 现有做法及不足

无。当前项目处于 MS1 空骨架阶段，尚未定义 Agent 协议或 Agent 实现。

## 本期范围

本期做：

- 定义 `PrintAgentProtocol/v0`、`AgentDecision`、`AgentAction`、`AgentMessage`、`ConfirmationRequest` 协议结构
- 实现 `RuleBasedAgent`，按输入模态、WorkflowState 与工具结果进行路由
- 至少支持 STL / GLB 上传：跳过生成，进入规范化与审计
- 将审计阻断、切片风险、设备离线和确认缺失转为用户可理解说明与合法下一步
- Agent 输出进入 Workflow Engine 前必须经过 schema 校验

本期明确不做：

- 不接入真实 LLM、RAG、多 Agent 或个性化推荐
- 不允许 Agent 直接生成 / 修改 G-code、修改 Mesh 或控制打印机

## 关键决策与依据

### 备选方案

1. 规则引擎直接推进状态，不经过 Agent 抽象层
   - 优点：实现简单，无额外抽象
   - 缺点：后续接入 LLM 时需要重构决策路由
2. Agent 抽象层 + RuleBasedAgent 实现
   - 优点：协议稳定后可直接替换为 LLM Agent，Workflow Engine 无须修改
   - 缺点：MS1 增加一层间接调用

### 本期选择

选择方案 2（Agent 抽象层 + RuleBasedAgent）。原因：MS1 固定协议后，后续 LLM 接入只需实现同一协议，不影响 Workflow Engine 和已有流程。

## 基本概念与信息结构

### AgentDecision

- `schemaVersion`: 协议版本号
- `action`: 建议的动作类型
- `message`: 面向用户的说明
- `confirmation`: 可选，需要用户确认的操作

### AgentAction

- 动作类型枚举（路由到上传、审计、切片、确认、打印等）
- 包含进入 Workflow Engine 所需的上下文

### ConfirmationRequest

- 需要确认的操作描述
- 确认前后的允许动作差异

## 原型 / 演示

不适用。

## 验收标准

### 例子 1：上传 STL / GLB 模型

- 现状：没有 Agent 路由，上传后的处理流程未定义。
- 提议后的行为：RuleBasedAgent 识别输入模态为 STL/GLB，跳过生成步骤，路由到规范化与审计。
- 验收：上传 STL 或 GLB 文件后，AgentDecision 包含 action 指向审计流程，不包含生成动作；AgentDecision.schemaVersion 存在且可通过 schema 校验。

### 例子 2：审计阻断

- 现状：审计失败时没有统一的用户反馈机制。
- 提议后的行为：Agent 将审计失败原因转化为用户可理解的说明，并给出合法下一步（如修复模型）。
- 验收：模拟审计失败，Agent 返回 AgentMessage 包含失败解释和下一步建议；Agent 不能绕过 Workflow Engine 直接推进到下一状态。

### 例子 3：确认缺失

- 现状：需要用户确认的步骤没有统一的确认机制。
- 提议后的行为：Agent 生成 ConfirmationRequest，等待用户确认后才进入后续阶段；确认前允许的动作与确认后不同。
- 验收：未确认状态下，Agent 输出的允许动作列表不包含需要确认后才能执行的操作；确认后动作列表更新。

### 例子 4：非法状态跳转

- 现状：无。
- 提议后的行为：AgentDecision 被 Workflow Engine 的 Policy Validator 校验，非法的状态跳转被拒绝。
- 验收：构造非法 AgentDecision（目标状态不可从当前状态到达），Workflow Engine 拒绝该决策并返回错误。
