# 提案：ComfyUI 相关接口参数描述

## 动机 / 用户故事

ComfyUI 工作流中的每个节点具有不同参数，参数含义和影响需要明确记录，使调用方和用户能够理解每个参数的用途、范围和专业术语。

## 目标用户

- 配置 ComfyUI 工作流的开发者
- 阅读参数文档以理解决策影响的用户
- 构建 Parameter Capability 时需要参考参数语义的开发者

## 现有做法及不足

参数信息分散在各个节点的实现中，缺乏统一的参数描述文档。调用方和用户不清楚参数范围、选项含义或对结果的影响。

## 本期范围

本期做：

- 逐个节点描述所有对外业务参数的名称、范围、选项和专业含义
- 覆盖以下节点类型：
  - Load Image（四视图输入）
  - TransparentBGSession+（图像分割）
  - ImageResize+（四视图输入和纹理视图）
  - Hy3DModelLoader（几何模型加载）
  - Hy3DGenerateMeshMultiView（多视图网格生成）
  - Hy3DVAEDecode（几何解码）
  - Hy3DPostprocessMesh（网格后处理）
  - Hy3DRenderSingleView（单视图渲染）
  - Hy3DCameraConfig（多视图相机配置）
  - DownloadAndLoadHy3DDelightModel（去光照模型）
  - SolidMask / RepeatImageBatch / ImageCompositeMasked（图像合成辅助）
  - Hy3DDelightImage（去光照处理）
  - DownloadAndLoadHy3DPaintModel（纹理生成模型）
  - Hy3DDiffusersSchedulerConfig（调度器配置）
  - Hy3DRenderMultiView（多视图控制图渲染）
  - Hy3DSampleMultiView（多视图纹理采样）
  - UpscaleModelLoader（超分辨率模型）
  - Hy3DBakeFromMultiview（多视图纹理烘焙）
  - CV2InpaintTexture（纹理修补）
  - Hy3DExportMesh（网格导出）
  - Hy3DTorchCompileSettings（编译设置）

本期明确不做：

- 不规定参数的默认取值或推荐配置
- 不提供参数组合的最佳实践
- 不做参数自动化测试或覆盖率检查

## 关键决策与依据

待讨论：本提案作为 Parameter Capability 的补充参考文档，还是独立交付的参数说明。

## 基本概念与信息结构

### 参数描述表

每个节点参数包含以下信息：

- 参数名称
- 范围或可选项
- 作用与专业术语说明

## 原型 / 演示

### Load Image 参数示例

| 参数 | 范围/选项 | 作用与专业术语说明 |
|---|---|---|
| `front` | 图像文件 | 正面条件视图。条件视图是模型用来约束三维形状的参考图；该端口中的轮廓、遮挡关系和物体比例会被解释为从正前方观察到的几何证据。 |
| `left` | 图像文件 | 左侧条件视图。这里的"左侧"指约定相机从物体左侧观察所得的视图；如果左右方向接反，模型会收到互相矛盾的空间约束。 |
| `right` | 图像文件 | 右侧条件视图。它与左视图共同提供正面图无法表达的厚度、前后层次和遮挡信息。 |
| `back` | 图像文件 | 背面条件视图。用于约束后脑、背部、衣物背面和正面视图中完全不可见的结构。 |

### Hy3DGenerateMeshMultiView 参数示例

| 参数 | 范围/选项 | 作用与专业术语说明 |
|---|---|---|
| `guidance_scale` | `0–100`，步长 `0.01` | 控制条件引导强度。Guidance/CFG（Classifier-Free Guidance，无分类器引导）通过比较"有条件预测"和"无条件预测"来放大模型对输入图的响应；数值越大，结果通常越强制贴合条件，但互相矛盾的视图也会被更强地同时施加。 |
| `steps` | `≥1` | 设置 Flow Matching 数值求解步数。Flow Matching（流匹配）学习把简单噪声分布连续变换为目标三维潜变量分布；每一步相当于对这条连续轨迹进行一次离散积分。增加步数会减小数值积分误差，但不会提升模型容量或补充输入中不存在的信息。 |
| `seed` | `0–18446744073709551615` | 设置伪随机数生成器种子。相同模型、输入、参数和执行环境下，相同种子通常产生相同的初始噪声；改变种子会改变模型选择的具体三维拓扑和细节方案。 |

### Hy3DPostprocessMesh 参数示例

| 参数 | 范围/选项 | 作用与专业术语说明 |
|---|---|---|
| `remove_floaters` | `true` / `false` | 是否删除 Floater（浮岛）。浮岛是与主网格没有拓扑连接的小型独立连通分量；算法通常计算所有连通分量并保留主体、删除较小碎片。 |
| `remove_degenerate_faces` | `true` / `false` | 是否删除 Degenerate Face（退化面）。退化面是三个顶点共线、顶点重复或面积接近零的三角形，可能导致法线、UV、渲染和后续简化算法异常。 |
| `reduce_faces` | `true` / `false` | 是否执行网格简化。网格简化通常通过边折叠等操作减少三角面数量，并尽量最小化表面误差；面数降低后文件更小、渲染更快，但小尺度轮廓和曲率会损失。 |
| `max_facenum` | `1–10000000` | 设置网格简化后的最大 Face 数量。Face 指构成表面的多边形，在此通常为三角形；该上限仅在启用 `reduce_faces` 时参与计算。 |
| `smooth_normals` | `true` / `false` | 是否平滑顶点法线。Normal（法线）是垂直于表面的方向向量，渲染器用它计算光照；平滑法线会在相邻面之间插值着色方向，使表面看起来连续，但不会移动顶点或增加真实几何。 |

### Hy3DVAEDecode 参数示例

| 参数 | 范围/选项 | 作用与专业术语说明 |
|---|---|---|
| `box_v` | `-10–10`，步长 `0.001` | 设置查询三维空间的边界尺度。解码器通常在 `[-box_v, box_v]` 的立方体内查询隐式表面；隐式表面不是直接存储三角形，而是由空间中每个位置的占用值或符号距离值定义。边界越大，同样分辨率下每个网格单元覆盖的实际空间越大。 |
| `octree_resolution` | `8–4096`，步长 `8` | 设置隐式体积场的空间采样分辨率。Octree（八叉树）是一种把三维立方体递归分成八个子立方体的空间结构，可把计算集中在接近表面的区域；分辨率决定最终可分辨的最小空间尺度。分辨率翻倍时，完整三维网格的理论采样量约增加八倍。 |
| `num_chunks` | `1–10000000` | 设置每批送入几何解码器的三维查询点数量。Chunk（分块）是把大量空间点拆成多个小批次处理；较大数值减少批次数、通常更快，但提高峰值显存，较小数值降低峰值显存但增加循环开销。它不改变查询坐标和等值面定义。 |
| `mc_level` | `-1–1`，步长 `0.0001` | 设置等值面阈值。隐式场为每个三维点输出一个标量，Marching Cubes 会寻找数值等于 `mc_level` 的表面；改变阈值会移动表面位置，使局部结构膨胀、收缩、连接或断开。该参数不是传统意义上的细节强度。 |
| `mc_algo` | `mc`、`dmc` | 选择表面提取算法。`MC` 是 Marching Cubes（移动立方体），逐个检查体素立方体的角点符号并生成三角面；`DMC` 在该工作流中指 Differentiable Marching Cubes（可微分移动立方体），使用可微实现处理符号距离场。 |
| `enable_flash_vdm` | `true` / `false` | 是否启用 FlashVDM 体积解码。VDM 在这里指用于把三维潜变量解码为空间体积场的模块；FlashVDM 使用分层查询和自适应 Key/Value 选择减少需要计算的注意力项，从而加速高分辨率体积解码。关闭后使用更直接的体积查询方式。 |
| `force_offload` | `true` / `false` | 解码完成后是否执行 Offload。Offload 是把模型权重从 GPU 显存移动到 CPU 内存或指定卸载设备，以便后续纹理模型使用显存；代价是再次使用时需要重新传输权重。 |

## 验收标准

### 例子 1：参数表完整性

- 现状：参数文档分散或缺失。
- 提议后的行为：所有节点的所有对外业务参数均在描述表中列出。
- 验收：覆盖 Load Image、TransparentBGSession+、ImageResize+、Hy3DModelLoader、Hy3DGenerateMeshMultiView、Hy3DVAEDecode、Hy3DPostprocessMesh、Hy3DRenderSingleView、Hy3DCameraConfig、DownloadAndLoadHy3DDelightModel、SolidMask、RepeatImageBatch、ImageCompositeMasked、Hy3DDelightImage、DownloadAndLoadHy3DPaintModel、Hy3DDiffusersSchedulerConfig、Hy3DRenderMultiView、Hy3DSampleMultiView、UpscaleModelLoader、Hy3DBakeFromMultiview、CV2InpaintTexture、Hy3DExportMesh、Hy3DTorchCompileSettings 节点。

### 例子 2：参数信息准确

- 现状：无。
- 提议后的行为：每个参数描述包含名称、范围/选项和专业术语说明。
- 验收：参数名称与实际节点参数名一致；范围/选项与实际允许值匹配；术语说明使用行业通用表述。

### 例子 3：文档可检索

- 现状：无。
- 提议后的行为：文档可按节点名称或参数名称检索。
- 验收：通过搜索节点名可定位到对应参数表；通过搜索参数名可定位到对应行。
