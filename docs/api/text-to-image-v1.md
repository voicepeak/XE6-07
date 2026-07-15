# Text-to-Image API v1 提案

## Overview

提供统一的文生图代理服务，封装 OpenAI GPT Image 和 Google Nano Banana，为 Multi-Views to 3D 流程生成前、左、后三张视图。

- Base URL: `http://<server>:8000`
- Content-Type: `application/json`
- 异步任务模型：提交 → 轮询状态 → 下载三视图 → 提交 3D 任务
- 默认 Provider: `auto`
- 默认输出：PNG、1K、前/左/后三视图

---

## Provider 与模型

| Provider | 默认模型 | 用途 |
| --- | --- | --- |
| `openai` | `gpt-image-2` | 文生图、参考图编辑 |
| `google` | `gemini-3.1-flash-image` | 文生图、参考图编辑、多轮修正 |
| `auto` | 服务端选择 | 根据可用性和配置选择 Provider |

Provider API 关系：

- OpenAI 首张图使用 Image Generations，后续视图和修正使用 Image Edits
- Google 统一使用 Interactions API
- OpenAI Responses 多轮模式和 Google `previous_interaction_id` 仅作为内部优化，不暴露给调用方
- 上层只依赖本 API，不直接依赖 Provider 的模型名、会话 ID 或返回格式

---

## 三视图生成规则

三视图按以下顺序生成：

1. 根据用户 prompt 生成 `front`
2. 以 `front` 为参考生成 `left`
3. 以已有视图为参考生成 `back`

三张图片必须保持：

- 同一物体和同一几何结构
- 相同颜色、材质和部件数量
- 相同背景、光照、画面比例和主体大小
- 只改变观察方向，不改变物体设计
- 主体完整、居中、无遮挡，适合图生 3D

---

## Endpoints

### 1. 提交文生图任务 `POST /api/v1/image-tasks`

```http
POST /api/v1/image-tasks
Content-Type: application/json
```

Request:

```json
{
  "prompt": "一台复古桌面收音机，木质外壳，两个圆形旋钮",
  "provider": "auto",
  "quality": "balanced",
  "resolution": "1k",
  "views": ["front", "left", "back"],
  "source_task_id": null,
  "feedback": null,
  "loop_id": null,
  "iteration": 1,
  "client_id": "my-app"
}
```

| Field | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `prompt` | string | ✅ | - | 物体描述 |
| `provider` | string | ❌ | `auto` | `auto` / `openai` / `google` |
| `quality` | string | ❌ | `balanced` | `draft` / `balanced` / `final` |
| `resolution` | string | ❌ | `1k` | `1k` / `2k` / `4k` |
| `views` | string[] | ❌ | 三视图 | `front` / `left` / `back` |
| `source_task_id` | string | ❌ | null | 修正时引用上一轮图片任务 |
| `feedback` | string | ❌ | null | 3D 审计反馈或本轮修改要求 |
| `loop_id` | string | ❌ | 自动生成 | 关联同一个文生图 → 3D loop |
| `iteration` | int | ❌ | 1 | 当前业务迭代轮次 |
| `client_id` | string | ❌ | null | 调用方标识 |

Response 201:

```json
{
  "task_id": "img_a1b2c3d4",
  "loop_id": "loop_f5e6d7c8",
  "iteration": 1,
  "status": "pending",
  "created_at": "2026-07-15T08:30:00+00:00",
  "message": "Image task queued successfully"
}
```

---

### 2. 查询任务状态 `GET /api/v1/image-tasks/{task_id}`

```http
GET /api/v1/image-tasks/img_a1b2c3d4
```

Response 200:

```json
{
  "task_id": "img_a1b2c3d4",
  "loop_id": "loop_f5e6d7c8",
  "iteration": 1,
  "status": "completed",
  "progress": 100.0,
  "provider": "openai",
  "model": "gpt-image-2",
  "created_at": "2026-07-15T08:30:00+00:00",
  "updated_at": "2026-07-15T08:31:20+00:00",
  "results": {
    "front": "/api/v1/image-tasks/img_a1b2c3d4/result/front",
    "left": "/api/v1/image-tasks/img_a1b2c3d4/result/left",
    "back": "/api/v1/image-tasks/img_a1b2c3d4/result/back"
  },
  "error": null,
  "client_id": "my-app"
}
```

status 值：

```text
pending → queued → running → completed | failed | cancelled
```

---

### 3. 下载视图 `GET /api/v1/image-tasks/{task_id}/result/{view}`

```http
GET /api/v1/image-tasks/img_a1b2c3d4/result/front
```

Path 参数：

| Field | Values |
| --- | --- |
| `view` | `front` / `left` / `back` |

Response 200: Binary image stream.

| Header | Value |
| --- | --- |
| `Content-Type` | `image/png` |
| `Content-Disposition` | `attachment; filename="front.png"` |

---

### 4. 健康检查 `GET /api/v1/image-health`

```http
GET /api/v1/image-health
```

Response 200:

```json
{
  "status": "ok",
  "providers": {
    "openai": "available",
    "google": "available"
  },
  "queue_depth": 0,
  "running_tasks": 0,
  "api_version": "1.0.0"
}
```

---

## 与 Multi-Views to 3D 的关系

```text
用户 prompt
    ↓
POST /api/v1/image-tasks
    ↓
front.png + left.png + back.png
    ↓
POST /api/v1/tasks
    ↓
result.glb
    ↓
3D 审计
    ├── 通过 → 完成
    └── 未通过 → 提交下一轮 image-task
```

提交 3D 任务时，将三张结果图传给现有接口：

```bash
curl -X POST http://localhost:8000/api/v1/tasks \
  -F "front=@front.png" \
  -F "left=@left.png" \
  -F "back=@back.png"
```

---

## Loop 修正规则

3D 审计发现图片可修复问题时，创建新的图片任务：

```json
{
  "prompt": "一台复古桌面收音机，木质外壳，两个圆形旋钮",
  "provider": "auto",
  "source_task_id": "img_a1b2c3d4",
  "feedback": "左视图主体被裁切，完整展示收音机左侧并保持其余设计不变",
  "loop_id": "loop_f5e6d7c8",
  "iteration": 2
}
```

规则：

- 默认最多 3 轮
- `source_task_id` 指向上一轮图片任务
- 修正局部视图时，其他视图继续作为一致性参考
- 如果主体设计或几何结构不一致，重新生成整组三视图
- 如果只是某一视图裁切、遮挡或背景问题，只重新生成失败视图
- 网格修复、切片或图生 3D 算法问题不回传给文生图模型
- 达到迭代上限、时间上限或成本上限后停止

---

## Provider 路由与兜底

`provider=auto` 时由服务端选择 Provider。

允许兜底的错误：

- 超时
- 连接失败
- 429
- 5xx
- Provider 配额不可用

不允许兜底的错误：

- 内容安全拒绝
- 非法图片或参数
- 鉴权和服务端配置错误

如果在三视图生成过程中切换 Provider，必须重新生成整组三视图，避免不同模型造成物体不一致。

---

## Error Handling

Response 400:

```json
{
  "detail": "Unsupported provider 'unknown'"
}
```

Response 404:

```json
{
  "detail": "Image task not found"
}
```

Provider 调用失败：

```json
{
  "task_id": "img_a1b2c3d4",
  "status": "failed",
  "error": {
    "code": "provider_timeout",
    "message": "Image provider request timed out",
    "retryable": true
  }
}
```

内容安全拒绝：

```json
{
  "task_id": "img_a1b2c3d4",
  "status": "failed",
  "error": {
    "code": "safety_blocked",
    "message": "Image generation request was blocked",
    "retryable": false
  }
}
```

---

## curl 测试示例

```bash
# 1. 提交文生图任务
curl -X POST http://localhost:8000/api/v1/image-tasks \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "一台复古桌面收音机，木质外壳，两个圆形旋钮",
    "provider": "auto"
  }'

# 2. 轮询状态
curl http://localhost:8000/api/v1/image-tasks/<task_id>

# 3. 下载三视图
curl -o front.png http://localhost:8000/api/v1/image-tasks/<task_id>/result/front
curl -o left.png http://localhost:8000/api/v1/image-tasks/<task_id>/result/left
curl -o back.png http://localhost:8000/api/v1/image-tasks/<task_id>/result/back

# 4. 提交 3D 任务
curl -X POST http://localhost:8000/api/v1/tasks \
  -F "front=@front.png" \
  -F "left=@left.png" \
  -F "back=@back.png"
```

---

## 验收标准

1. 同一个文生图任务可以产出前、左、后三张图片
2. 三张图片的主体结构、颜色、材质和部件保持一致
3. 生成结果可以直接提交给 Multi-Views to 3D API
4. 调用方切换 OpenAI / Google 时不需要改变请求和结果处理逻辑
5. 3D 审计失败后可以通过 `source_task_id` 和 `feedback` 创建下一轮任务
6. Loop 达到上限后停止，不出现无限生成
7. Provider 技术故障可以受控兜底，内容安全拒绝不切换 Provider
8. 日志记录实际 Provider、模型、任务耗时、错误和迭代轮次，但不记录 API Key 和图片 Base64
