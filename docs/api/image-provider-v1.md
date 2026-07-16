# Image Provider API v1

## 接口含义

Image Provider API 是业务服务与外部生图服务之间的内部适配接口。它把不同供应商的请求参数、返回格式和错误统一起来，让上层无需直接依赖供应商 SDK。首期接入：

- OpenAI GPT Image
- Google Nano Banana

上层只使用统一的 `ImageProvider` 接口，不直接依赖供应商 SDK、模型参数、会话 ID 或返回格式。

该 API 可以：

- 根据文字描述生成一张图片
- 根据一张或多张参考图编辑并生成一张新图片
- 自动选择或按请求指定 OpenAI、Google Provider
- 返回统一的图片、模型、用量、耗时和错误信息

该 API 是服务端内部接口，不直接向终端用户开放，也不负责异步任务状态、多视图生成顺序、3D Loop、业务数据持久化或产物存储。

## 接口一览

| 功能 | 请求方法 | 路径 | 含义 |
| --- | --- | --- | --- |
| 文生图 | `POST` | `/api/v1/image-provider/generate` | 根据文字描述生成一张图片 |
| 参考图编辑 | `POST` | `/api/v1/image-provider/edit` | 根据文字和参考图生成一张编辑后的图片 |

所有路径均以部署环境的服务地址为基准。`POST` 请求和所有响应使用 `application/json`；JSON 中的图片数据使用 Base64 字符串传输，进入 Provider 实现后解码为二进制。图片数据不得写入普通日志。

---

## 调用关系

```text
Text-to-Image API
    ↓
MultiViewGenerator
    ↓
ProviderRouter
    ├── OpenAIImageProvider
    └── GoogleImageProvider
            ↓
      ImageProviderResult
```

相关接口：

- [Text-to-Image API v1](./text-to-image-v1.md)
- [Multi-Views to 3D API v1](./multi-views-to-3d-v1.md)

---

## 职责边界

Provider 负责：

- 将统一请求转换成供应商请求
- 调用生图或图片编辑 API
- 将 Base64 / 图片响应转换成统一结果
- 统一供应商错误
- 返回实际 Provider、模型、请求 ID 和耗时
- 声明自身支持的能力

Provider 不负责：

- 不管理异步任务状态
- 不决定前、左、后三视图生成顺序
- 不管理文生图 → 3D Loop
- 不处理 3D 审计反馈
- 不直接保存业务任务
- 不自行选择或切换其他 Provider

这些职责分别由 `ImageTaskService`、`MultiViewGenerator`、`ProviderRouter` 和 Loop Coordinator 处理。

---

## Provider 接口

```python
from typing import Protocol


class ImageProvider(Protocol):
    name: str

    def capabilities(self) -> "ImageProviderCapabilities":
        ...

    async def generate(
        self,
        request: "ImageGenerateRequest",
    ) -> "ImageProviderResult":
        ...

    async def edit(
        self,
        request: "ImageEditRequest",
    ) -> "ImageProviderResult":
        ...

    async def health(self) -> "ImageProviderHealth":
        ...
```

V1 中一次 Provider 调用只返回一张图片。需要前、左、后三张视图时，由 `MultiViewGenerator` 依次调用 Provider。

---

## 文生图

根据文字描述生成一张新图片，不接收参考图。

- 请求方法：`POST`
- 请求路径：`/api/v1/image-provider/generate`
- 请求体：`ImageGenerateRequest`
- 成功响应：`200 OK`，返回 `ImageProviderResult`

### 请求参数

| 参数 | 类型 | 必填 | 默认值 | 含义 |
| --- | --- | --- | --- | --- |
| `request_id` | string | 是 | - | 调用方生成的唯一请求 ID，用于日志追踪和串联响应 |
| `provider` | string | 否 | `auto` | `auto` / `openai` / `google`；`auto` 由路由器选择 Provider |
| `prompt` | string | 是 | - | 本次实际执行的图片描述 |
| `quality` | string | 否 | `balanced` | `draft` / `balanced` / `final` |
| `resolution` | string | 否 | `1k` | `1k` / `2k` / `4k` |
| `aspect_ratio` | string | 否 | `1:1` | 图片宽高比，取值必须在目标 Provider 的能力声明中 |
| `output_format` | string | 否 | `png` | `png` / `jpeg` / `webp` |

### 请求示例

```json
{
  "request_id": "req_a1b2c3d4",
  "provider": "auto",
  "prompt": "一台复古桌面收音机，正视图，完整展示主体",
  "quality": "balanced",
  "resolution": "1k",
  "aspect_ratio": "1:1",
  "output_format": "png"
}
```

---

## 参考图编辑

根据文字指令和一张或多张参考图生成一张新图片，可用于改变观察角度、局部编辑或保持主体一致性。

- 请求方法：`POST`
- 请求路径：`/api/v1/image-provider/edit`
- 请求体：`ImageEditRequest`
- 成功响应：`200 OK`，返回 `ImageProviderResult`

### 请求参数

| 参数 | 类型 | 必填 | 默认值 | 含义 |
| --- | --- | --- | --- | --- |
| `request_id` | string | 是 | - | 调用方生成的唯一请求 ID，用于日志追踪和串联响应 |
| `provider` | string | 否 | `auto` | `auto` / `openai` / `google`；`auto` 由路由器选择 Provider |
| `prompt` | string | 是 | - | 对参考图执行的编辑指令 |
| `reference_images` | image[] | 是 | - | 一张或多张参考图片；数量必须符合目标 Provider 的能力声明 |
| `quality` | string | 否 | `balanced` | `draft` / `balanced` / `final` |
| `resolution` | string | 否 | `1k` | `1k` / `2k` / `4k` |
| `aspect_ratio` | string | 否 | `1:1` | 图片宽高比，取值必须在目标 Provider 的能力声明中 |
| `output_format` | string | 否 | `png` | `png` / `jpeg` / `webp` |

`reference_images` 元素：

| 参数 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `asset_id` | string | 是 | 上层资产 ID，用于结果追踪；Provider 不通过该 ID 读取资产 |
| `mime_type` | string | 是 | 图片 MIME 类型，例如 `image/png` |
| `data` | string | 是 | 图片内容的 Base64 字符串，不包含 Data URL 前缀 |

### 请求示例

```json
{
  "request_id": "req_e5f6g7h8",
  "provider": "auto",
  "prompt": "生成同一台收音机的左视图，只改变观察方向",
  "reference_images": [
    {
      "asset_id": "asset_front",
      "mime_type": "image/png",
      "data": "<base64>"
    }
  ],
  "quality": "balanced",
  "resolution": "1k",
  "aspect_ratio": "1:1",
  "output_format": "png"
}
```

V1 不在统一协议中提供任意 Provider 参数透传，也不提供跨 Provider 的 `seed` 保证。

---

## 生图与编辑响应

文生图和参考图编辑成功时均返回 `200 OK` 和以下统一结果：

```json
{
  "provider": "openai",
  "model": "gpt-image-2",
  "request_id": "req_a1b2c3d4",
  "provider_request_id": "provider_req_123",
  "image": {
    "data": "<base64>",
    "mime_type": "image/png",
    "width": 1024,
    "height": 1024
  },
  "usage": {
    "input_tokens": null,
    "output_tokens": null
  },
  "provider_session_ref": null,
  "latency_ms": 12000
}
```

### 响应字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `provider` | string | 实际执行请求的 Provider |
| `model` | string | 实际执行请求的模型 |
| `request_id` | string | 原样返回调用方传入的请求 ID |
| `provider_request_id` | string \| null | 供应商请求 ID，用于问题排查 |
| `image.data` | string | 图片内容的 Base64 字符串 |
| `image.mime_type` | string | 图片 MIME 类型 |
| `image.width` | integer | 图片宽度，单位为像素 |
| `image.height` | integer | 图片高度，单位为像素 |
| `usage.input_tokens` | integer \| null | 供应商返回的输入 token 数 |
| `usage.output_tokens` | integer \| null | 供应商返回的输出 token 数 |
| `provider_session_ref` | string \| null | 供应商会话引用，仅用于辅助追踪 |
| `latency_ms` | integer | 本次 Provider 调用耗时，单位为毫秒 |

规则：

- HTTP 响应使用 Base64；Provider 内部结果使用图片二进制，Asset Store 由上层负责
- 图片 Base64 和二进制不得写入普通日志
- `provider_request_id` 用于供应商问题排查
- `provider_session_ref` 是可选字段，不作为业务恢复的唯一状态
- Provider 未返回 usage 时允许为 null
- 返回结果前必须验证图片能正常解码

---

## Provider 能力声明

```json
{
  "provider": "openai",
  "operations": ["generate", "edit"],
  "output_formats": ["png", "jpeg", "webp"],
  "resolution_tiers": ["1k", "2k", "4k"],
  "supports_multiple_references": true,
  "supports_provider_session": false,
  "supports_transparent_background": false
}
```

`ProviderRouter` 在调用前根据能力声明校验请求。不支持的能力必须返回明确错误，不能静默降级。

---

## OpenAIImageProvider

默认模型：`gpt-image-2`

| Operation | OpenAI API |
| --- | --- |
| `generate` | Image Generations |
| `edit` | Image Edits |

字段映射：

| 统一字段 | OpenAI 字段 |
| --- | --- |
| `prompt` | `prompt` |
| `resolution` + `aspect_ratio` | `size` |
| `quality` | `quality` |
| `output_format` | `output_format` |
| `reference_images` | Edits 图片输入 |

V1 规则：

- 使用 Image API，不使用 Responses API 保存业务会话
- `generate` 和 `edit` 都只请求一张图片
- 统一质量档位映射为 `low` / `medium` / `high`
- 输出解码成图片二进制后再返回
- OpenAI 特有字段不得直接泄漏给上层请求协议

---

## GoogleImageProvider

默认模型：`gemini-3.1-flash-image`

可选模型：

| 质量 | 模型 |
| --- | --- |
| `draft` | `gemini-3.1-flash-lite-image` |
| `balanced` | `gemini-3.1-flash-image` |
| `final` | `gemini-3-pro-image` |

统一使用 Gemini Interactions API。

字段映射：

| 统一字段 | Google 字段 |
| --- | --- |
| `prompt` | `input` text block |
| `reference_images` | `input` image block |
| `aspect_ratio` | `response_format.aspect_ratio` |
| `resolution` | `response_format.image_size` |
| `output_format` | `response_format.mime_type` |

V1 规则：

- `generate` 使用文本 input
- `edit` 使用文本和图片 input
- 不依赖 `previous_interaction_id` 恢复业务任务
- 多轮修正由上层重新传入参考图
- 只返回一张主结果图；混合输出中的文字记录为 Provider 元数据
- 生成图片的来源标记由上层产物元数据保留

---

## ProviderRegistry

Registry 保存可用 Provider 实例：

```python
registry.register("openai", OpenAIImageProvider(...))
registry.register("google", GoogleImageProvider(...))

provider = registry.get("openai")
```

规则：

- Provider 名称全局唯一
- 未配置 API Key 的 Provider 不注册
- 重复注册直接报错
- Registry 不负责路由策略
- 测试环境可以注册 Mock Provider

---

## ProviderRouter

Router 根据请求和运行状态选择 Provider：

```python
provider = router.select(
    requested_provider="auto",
    operation="edit",
    quality="balanced",
    resolution="1k",
    reference_count=1,
)
```

路由顺序：

1. 如果调用方指定 Provider，优先使用指定值
2. 校验 Provider 是否已配置并支持所需能力
3. `auto` 使用服务端配置的默认 Provider
4. 默认 Provider 不可用时选择健康的备用 Provider
5. 返回实际 Provider 和模型到任务记录

Router 只负责选择，不直接调用供应商 SDK。

---

## 重试与兜底

允许重试和兜底：

- 连接失败
- 超时
- 429
- 5xx
- Provider 配额不可用

不允许重试或切换 Provider：

- 内容安全拒绝
- 非法参数或图片
- 不支持的能力
- API Key 或服务配置错误

规则：

- 单个 Provider 默认最多调用 2 次
- 重试使用指数退避和随机抖动
- 三视图中途切换 Provider 时，上层重新生成整组三视图
- Provider 不自行调用另一个 Provider；兜底由 Router 和上层服务控制

---

## 统一错误

```python
class ImageProviderError(Exception):
    code: str
    provider: str
    retryable: bool
    provider_request_id: str | None
```

错误码：

| Code | HTTP 状态 | Retryable | Description |
| --- | --- | --- | --- |
| `provider_timeout` | `504` | ✅ | Provider 调用超时 |
| `provider_rate_limited` | `429` | ✅ | 触发限流 |
| `provider_unavailable` | `503` | ✅ | Provider 服务异常 |
| `provider_quota_exhausted` | `503` | ✅ | 配额不可用，可尝试备用 Provider |
| `provider_configuration_error` | `503` | ❌ | API Key、模型或服务配置错误 |
| `invalid_provider_request` | `400` | ❌ | 请求参数或图片不合法 |
| `unsupported_capability` | `422` | ❌ | Provider 不支持请求能力 |
| `safety_blocked` | `422` | ❌ | 内容安全拒绝 |
| `invalid_provider_response` | `502` | 视情况 | 返回结果缺失或图片无法解码 |

所有接口使用相同的错误响应结构：

```json
{
  "error": {
    "code": "invalid_provider_request",
    "message": "reference_images is required",
    "provider": "google",
    "retryable": false,
    "provider_request_id": null
  }
}
```

`message` 只能包含可安全返回的摘要。上层接口只依赖统一错误码，供应商原始错误放入受控日志。

---

## 配置

```text
IMAGE_PROVIDER_DEFAULT=openai
IMAGE_PROVIDER_FALLBACK=google
IMAGE_PROVIDER_TIMEOUT_SECONDS=150
IMAGE_PROVIDER_MAX_ATTEMPTS=2

OPENAI_API_KEY=<secret>
OPENAI_IMAGE_MODEL=gpt-image-2

GOOGLE_API_KEY=<secret>
GOOGLE_IMAGE_MODEL=gemini-3.1-flash-image
```

规则：

- API Key 只从 Secret Manager 或服务端环境读取
- API Key 不进入请求、响应、数据库和日志
- 模型 ID、超时、重试和默认路由必须配置化
- 健康检查不得通过生成付费图片实现

---

## 测试要求

- 所有 Provider 实现运行同一套 contract tests
- Mock Provider 用于普通单元测试
- OpenAI / Google 响应使用脱敏 fixture 测试解析
- 真实付费 API 测试只在显式开启的集成测试中执行
- 测试超时、429、5xx、安全拒绝和无效图片解码
- 测试 Router 的指定 Provider、自动路由和兜底逻辑
- 测试三视图中途切换 Provider 时整组重新生成

---

## 验收标准

1. `OpenAIImageProvider` 和 `GoogleImageProvider` 实现相同接口
2. 上层使用统一请求完成文生图和参考图编辑
3. Provider 返回统一图片、模型、请求 ID、usage 和耗时结构
4. Provider 差异不泄漏到 Text-to-Image API 请求协议
5. Router 可以指定 Provider，也可以使用 `auto` 路由
6. 不支持的能力在调用前返回 `unsupported_capability`
7. 技术故障可以有限重试和兜底，安全拒绝不能切换 Provider
8. Provider API Key 和图片 Base64 不进入普通日志
9. 新增 Provider 时不需要修改 Text-to-Image API 和 3D Loop 协议
