# 提案：ComfyUI 内部参数能力

## 动机 / 用户故事

ComfyUI 参数分散在不同节点中。算法组需要把节点参数转换为可校验、可版本化和可追溯的内部参数 Schema，同时只向后端开放少量稳定的业务参数。

## 目标用户

- 维护生成算法与 ComfyUI 工作流的算法工程师
- 调用算法服务稳定 API 的后端开发人员

后端是唯一调用方。终端用户能否调整某个业务参数由后端产品逻辑决定，不由算法服务读取用户角色。

## 本期范围

- 为每个已发布算法能力定义内部 `ParameterSchema`。
- 记录参数名称、说明、类型、默认值、范围、枚举和组合约束。
- 将参数划分为 `backend_configurable` 与 `algorithm_internal`。
- 按“算法默认值 -> 已发布预设 -> 后端覆盖”解析最终参数。
- 在创建算法任务时冻结完整参数快照。
- 拒绝未知参数、越界参数和冲突组合。
- 记录参数 Schema 版本、资源影响和实际生效值。
- 将业务参数映射到 ComfyUI 节点参数，映射关系仅算法内部可见。

## 明确不做

- 不向后端开放任意节点参数、graph、连线或服务器路径。
- 不把终端 user/admin 角色带入算法参数权限模型。
- 不允许后端修改 `algorithm_internal` 参数。
- 不静默忽略未知参数。
- 不允许已提交任务原地修改参数。

## 基本概念

```text
ParameterDefinition {
  name
  description
  type
  default
  constraints
  exposure       # backend_configurable | algorithm_internal
  resourceImpact
  version
}
```

后端 API Schema 只包含 `backend_configurable` 参数。ComfyUI 节点名和内部参数名不得出现在算法服务公共响应中。

## 验收标准

### 例子 1：后端查询稳定参数

- 后端获得业务参数的名称、类型、默认值与约束。
- 返回内容不包含 ComfyUI 节点 ID、内部运行参数或服务器路径。

### 例子 2：参数覆盖

- 后端提交合法业务参数后，算法服务生成完整 resolved parameters 快照。
- 未覆盖参数使用已发布默认值或预设值。

### 例子 3：非法参数

- 未知字段、越界值、冲突组合或 `algorithm_internal` 参数覆盖均在任务创建前被拒绝。
- 响应包含稳定错误码和具体无效字段。

### 例子 4：任务可复现

- 每个任务保存 Schema 版本、工作流版本和完整参数快照。
- 内部默认值变化不会改写历史任务。

### 例子 5：版本兼容

- 同一 Schema 版本内参数名称、类型和语义保持稳定。
- 删除参数或改变语义时必须发布新版本并提供变更说明。
