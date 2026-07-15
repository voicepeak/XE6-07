# Multi-Views to 3D API v1

## Overview

FastAPI-based proxy service that wraps a deployed ComfyUI workflow (Hunyuan3D-2.1 Multi-View to 3D Mesh) behind a clean REST API with task queuing, status tracking, and result download.

- **Base URL**: `http://<server>:8000`
- **Content-Type**: `application/json` (except file upload/download)
- **异步任务模型**: 提交 → 轮询状态 → 下载结果

---

## Endpoints

### 1. 提交生成任务 `POST /api/v1/tasks`

提交三张视图图片（前/左/后），启动 3D 模型生成任务。

```
POST /api/v1/tasks
Content-Type: multipart/form-data
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `front` | file | ✅ | 前视图 (PNG/JPG/WEBP) |
| `left` | file | ✅ | 左视图 (PNG/JPG/WEBP) |
| `back` | file | ✅ | 后视图 (PNG/JPG/WEBP) |
| `seed` | int | ❌ | 随机种子，不传自动生成 |
| `guidance_scale` | float | ❌ | CFG 引导尺度，默认 5.5 |
| `steps` | int | ❌ | 推理步数，默认 50 |
| `client_id` | str | ❌ | 调用方标识 |

**Response 201**:
```json
{
  "task_id": "a1b2c3d4e5f6",
  "status": "pending",
  "created_at": "2026-07-15T08:30:00+00:00",
  "message": "Task queued successfully"
}
```

---

### 2. 查询任务状态 `GET /api/v1/tasks/{task_id}`

```
GET /api/v1/tasks/a1b2c3d4e5f6
```

**Response 200**:
```json
{
  "task_id": "a1b2c3d4e5f6",
  "status": "running",
  "progress": 45.0,
  "created_at": "2026-07-15T08:30:00+00:00",
  "updated_at": "2026-07-15T08:35:00+00:00",
  "input_filenames": ["front.png", "left.png", "back.png"],
  "params": {
    "seed": 42,
    "guidance_scale": 5.5,
    "steps": 50
  },
  "result_url": null,
  "error": null,
  "client_id": null
}
```

**status 值**: `pending` → `queued` → `running` → `completed` | `failed` | `cancelled`

---

### 3. 下载结果 `GET /api/v1/tasks/{task_id}/result`

```
GET /api/v1/tasks/a1b2c3d4e5f6/result
```

**Response 200**: Binary file stream.

| Header | Value |
|--------|-------|
| `Content-Type` | `model/gltf-binary` 或 `application/octet-stream` |
| `Content-Disposition` | `attachment; filename="result.glb"` |

**Response 400**: Task not yet completed.

---

### 4. 任务列表 `GET /api/v1/tasks`

```
GET /api/v1/tasks?status=running&limit=10&offset=0&client_id=my-app
```

| Query | Type | Default | Description |
|-------|------|---------|-------------|
| `status` | str | - | 过滤状态 |
| `limit` | int | 50 | 分页大小 (max 200) |
| `offset` | int | 0 | 分页偏移 |
| `client_id` | str | - | 按客户端过滤 |

**Response 200**:
```json
{
  "tasks": [
    {
      "task_id": "a1b2c3d4e5f6",
      "status": "completed",
      "progress": 100.0,
      "created_at": "2026-07-15T08:30:00+00:00",
      "updated_at": "2026-07-15T08:40:00+00:00",
      "result_url": "/api/v1/tasks/a1b2c3d4e5f6/result",
      "error": null
    }
  ],
  "total": 1
}
```

---

### 5. 健康检查 `GET /health`

```
GET /health
```

**Response 200**:
```json
{
  "status": "ok",
  "comfyui_connected": true,
  "queue_depth": 0,
  "running_tasks": 0,
  "api_version": "1.0.0",
  "uptime_seconds": 3600.0
}
```

---

## Task Status Lifecycle

```
         ┌──────────┐
         │  pending │
         └────┬─────┘
              │
         ┌────▼─────┐
         │  queued  │
         └────┬─────┘
              │
         ┌────▼─────┐
         │  running │
         └────┬─────┘
              │
    ┌─────────┼─────────┐
    │         │         │
 ┌──▼──────┐┌──▼─────┐┌──▼─────────┐
 │completed││ failed ││ cancelled  │
 └─────────┘└────────┘└────────────┘
```
## Error Handling

**400 Bad Request**:
```json
{
  "detail": "Unsupported file type '.gif'. Allowed: .jpeg, .jpg, .png, .webp"
}
```

**404 Not Found**:
```json
{
  "detail": "Task not found"
}
```

**Default error**:
```json
{
  "detail": "Task status is 'running', not 'completed'"
}
```

---

## curl 测试示例

```bash
# 1. 提交任务
curl -X POST http://localhost:8000/api/v1/tasks \
  -F "front=@front.png" \
  -F "left=@left.png" \
  -F "back=@back.png" \
  -F "seed=42"

# 2. 轮询状态
curl http://localhost:8000/api/v1/tasks/<task_id>

# 3. 下载结果
curl -o model.glb http://localhost:8000/api/v1/tasks/<task_id>/result

# 4. 健康检查
curl http://localhost:8000/health
```

---

## Python 调用示例

```python
import httpx

def submit_task(server: str, front: str, left: str, back: str, seed: int | None = None):
    with httpx.Client() as client:
        files = {
            "front": open(front, "rb"),
            "left": open(left, "rb"),
            "back": open(back, "rb"),
        }
        data = {}
        if seed is not None:
            data["seed"] = str(seed)

        resp = client.post(f"{server}/api/v1/tasks", files=files, data=data)
        return resp.json()["task_id"]

def wait_for_result(server: str, task_id: str, poll_interval: float = 5.0):
    import time
    while True:
        resp = httpx.get(f"{server}/api/v1/tasks/{task_id}")
        task = resp.json()
        if task["status"] == "completed":
            return task["result_url"]
        if task["status"] in ("failed", "cancelled"):
            raise RuntimeError(f"Task {task_id} {task['status']}: {task.get('error')}")
        time.sleep(poll_interval)
```
