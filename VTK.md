# VTK 与 HOOPS Visualize 在 CAE 领域“渲染器之上的工具层”对比分析

## vtk中的适配层
vtkMapper
vtkPolyDataMapper
vtkDataSetMapper
vtkCompositePolyDataMapper
vtkHierarchicalPolyDataMapper
vtkGlyph3DMapper
vtkGraphMapper
vtkPointGaussianMapper
vtkCellGridMapper
vtkCompositeCellGridMapper
vtkHyperTreeGridMapper
//图像 / 切片 mapper
vtkImageMapper
vtkImageMapper3D
vtkImageSliceMapper
vtkImageResliceMapper
//2D / 标签 / 文本 mapper
vtkMapper2D
vtkPolyDataMapper2D
vtkTextMapper
vtkLabeledDataMapper
vtkLabelPlacementMapper
vtkDynamic2DLabelMapper
vtkLabeledContourMapper

vtkGPUVolumeRayCastMapper
vtkUnstructuredGridVolumeMapper

面向CAE的功能封装层
- 深度偏移自定义组合接口 <--- 提供点线面的offset动态组合，在业务层可视化模块中根据场景，动态调整深度偏移组合。
- 填充模式修复 <-- Gems负责接口实现（这里填充模式更偏向于业务层的表述，e.g. 初始借鉴于PointWise FillMode）


## 1. 先给结论

如果只看 CAE 应用里“渲染器之上的能力层”，两者的定位非常不同：

- **VTK** 更像“可组装的工具箱”。
  - 渲染器之上的能力分散在 `Interaction`、`Views`、`Charts`、`Filters`、`IO` 等模块中。
  - 它的上层能力依赖 **pipeline + representation + view + widget** 组合出来。
  - 官方文档按“模块”和“示例”组织，适合工程师拼装定制系统。

- **HOOPS Visualize** 更像“带现成交互和场景机制的产品级图形内核”。
  - 渲染器之上的能力集中在 **Operator、Selection/Highlighting、Portfolio/Style、Exchange/OOC、View/Layout/Canvas** 等对象上。
  - 它的上层能力依赖 **Scene Graph + View system + Controls/Operators** 直接落地。
  - 官方文档按“Programming Guide/Technical Overview/Guides”组织，适合做完整 CAD/CAE 桌面应用。

从 CAE 视角看：

- **VTK 的工具层更强在数据处理、后处理、可视化算法、图表、格式支持。**
- **HOOPS 的工具层更强在交互成品度、装配场景组织、大模型浏览、产品化 UI 接口风格。**

---

## 2. 什么叫“渲染器之上的工具层”

为了避免把底层渲染和上层工具混在一起，本文采用下面这条边界：

### 2.1 不算工具层的部分

- 图元提交
- GPU backend / driver
- render pass / shader / mapper 的底层实现
- renderer / render window 本身

### 2.2 算工具层的部分

- 相机与导航
- 选择、拾取、高亮
- 尺寸测量、角度测量、标注
- 剖切、切片、探针、重切片
- 多视图、联动视图、布局
- 图表、表格、二维上下文可视化
- 领域数据导入、CAE 文件读取
- 大模型/流式/OOC/静态场景优化
- 面向应用层的对象组织方式和事件挂接方式

---

## 3. 面向 CAE 的工具层功能列表

下面这张表只列“渲染器之上”的能力，并按 CAE 软件常见需求做归类。

| 类别 | VTK | HOOPS Visualize | CAE 价值 |
|---|---|---|---|
| 视图导航 | `vtkRenderWindowInteractor` + `vtkInteractorStyle*` | `Operator` 栈，`Orbit/Pan/Zoom` 等标准算子 | 模型浏览、检查、定位 |
| 选择与拾取 | `vtkPicker`、`vtkCellPicker`、`vtkPropPicker`、Widgets picking、`vtkAnnotationLink` 联动 | `SelectionControl`、`SelectionOptionsKit`、`HighlightControl`、`KeyPath` | 零件/单元/面边点选择，高亮和查询 |
| 尺寸测量 | `vtkDistanceWidget`、`vtkAngleWidget`、`vtkBiDimensionalWidget` | `MeasurementOperator`、`Exchange::MeasurementOperator` | 尺寸、角度、面积、B-Rep 测量 |
| 剖切/切片 | `vtkImplicitPlaneWidget2`、`vtkImagePlaneWidget`、`vtkResliceCursorWidget` | section plane、clip region、bounded section、operator 驱动剖切 | 截面分析、MPR、局部检查 |
| 标注/辅助显示 | `vtkOrientationMarkerWidget`、`vtkScalarBarWidget`、caption/text widgets | markup/redline、glyph、named style、overlay、3D spriting | 方向轴、标尺、注释、UI 叠加 |
| 多视图与联动 | `vtkView`、`vtkRenderViewBase`、`vtkContextView`、`vtkDataRepresentation`、`vtkAnnotationLink` | `Window -> Canvas -> Layout -> View -> Model` | 正交视图、视图联动、结果对比 |
| 图表与二维视图 | `vtkContextView`、`vtkChartXY`、`vtkChartMatrix`、Qt table/chart | 不是 HOOPS 核心强项，更多依赖应用层 UI | 曲线、历史结果、后处理统计图 |
| 数据管线与后处理 | `Filters/*`、pipeline execution model | HOOPS 有显示层控制，但后处理算法远少于 VTK | 等值面、切片、流线、阈值、提取、统计 |
| CAE 文件输入 | `IO/CGNS`、`Exodus`、`IOSS`、`EnSight`、`FLUENTCFF`、`CONVERGECFD`、`LSDyna` 等 | `Exchange`、`Stream::File::Import`、多 CAD/CAM 格式 | CFD/FEA 数据导入 |
| 大模型与性能 | static mesh/cache、模块可选、数据格式并行 I/O、渲染后端可扩展 | `StaticModel`、display lists、culling、fixed framerate、OOC | 百万级/装配级浏览与交互 |
| 应用集成 | Qt、views、representations、examples 偏框架拼装 | GUI 平台集成、operators、controls、更接近成品 SDK | 做成桌面 CAE 产品 |

### 3.1 VTK 更适合承载的 CAE 上层能力

- 后处理可视化
  - slice / clip / contour / threshold / streamlines / tensor / temporal
- 科学绘图
  - XY 图、矩阵图、表格联动
- 多数据格式读取
  - CAE 仿真结果格式覆盖面很广
- 医学/体数据交互
  - `vtkImagePlaneWidget`、`vtkResliceCursorWidget`、`vtkInteractorStyleImage`

### 3.2 HOOPS 更适合承载的 CAE 上层能力

- CAD/CAE 模型浏览器
  - 选择、高亮、operator、markup、视图管理都更“现成”
- 装配级场景组织
  - `Segment`、`IncludeSegment`、`ReferenceGeometry`、`KeyPath`
- 大模型交互体验
  - `StaticModel`、overlay、OOC、fixed framerate
- 产品级交互 API
- `Control + Operator + Kit` 风格比 VTK 更统一

---

## 3A. 工具层功能树

这一节按你要求的方式重排：

- 第一层：工具大类
- 第二层：具体工具功能
- 每个功能都写三件事
  - 作用
  - 如何使用
  - 接口怎么用

这里的“接口怎么用”采用工程上最有用的表达方式：

- **VTK**：列出典型类和最常见挂接点
- **HOOPS**：列出典型对象和最常见入口

---

### 3A.1 场景架构层

#### A. 场景组织

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 场景根与层级组织 | 组织模型、装配、部件、子部件、显示状态，是所有 CAE 工具挂接的基础 | 先建立统一的场景层级，再把几何、属性、工具结果挂到明确节点上 | **VTK**：通常由 `vtkDataObject -> vtkDataRepresentation -> vtkView` 组成逻辑层级，渲染对象再进入 `vtkRenderer`。**HOOPS**：直接用 `SegmentKey::Subsegment()` 建立 `Segment` 树，挂到 `Model/View` 下 |
| 实例化与复用 | 让重复零件、重复单元块、重复标记共享数据，降低内存和构建时间 | 将重复出现的几何抽成库节点，再在业务场景中引用 | **VTK**：通常依赖数据共享、mapper 复用、actor 复用或应用层装配逻辑。**HOOPS**：用 `IncludeSegment()` 做段实例化，用 `ReferenceGeometry()` 做几何实例化 |
| 场景检索与定位 | 让选择、跳转、属性编辑、批量操作能精确找到目标对象 | 给节点命名或维护稳定 ID，再通过遍历/搜索定位对象 | **VTK**：常见做法是 actor/picker/selection 配合应用层 ID 映射。**HOOPS**：用 `Find()`、`SearchResults`、`KeyPath`、`SelectionResults` |
| 场景与视图解耦 | 支持同一份模型被多个视图复用，如主视图、剖视图、结果视图 | 模型数据与视图状态分离，视图只保存相机、样式、交互状态 | **VTK**：同一数据源可被多个 `Representation/View` 复用。**HOOPS**：`Model` 挂场景，`View` 挂相机与显示方式，`Layout/Canvas` 负责组织多个视图 |

#### B. 属性与样式系统

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 层级属性继承 | 让颜色、可见性、透明度、剖切状态按层级传播，减少重复设置 | 在高层节点设置公共属性，在局部节点覆盖特殊状态 | **VTK**：更多通过 actor/property/representation 显式设置，继承性弱。**HOOPS**：直接依赖 `Segment` 层级继承，常见入口是 `GetMaterialMappingControl()`、`GetVisibilityControl()` |
| 命名样式 | 把“高亮样式”“警告样式”“选中样式”抽成可复用资源 | 先定义样式，再在交互或条件渲染时应用 | **VTK**：通常由应用层维护样式对象或 property 模板。**HOOPS**：`PortfolioKey::DefineNamedStyle()`，再通过 `StyleControl` 或 `HighlightOptionsKit` 使用 |
| 条件显示/条件样式 | 支持同一场景在不同视图、不同模式下显示不同内容 | 将“显示条件”与“场景内容/样式”解耦 | **VTK**：一般由应用层逻辑控制 actor/representation 可见性。**HOOPS**：`ConditionalExpression(...)`，配合 `PushNamed(...)` 和 `IncludeSegment(..., condition)` |
| 资源库管理 | 统一管理 glyph、纹理、线型、材质等辅助显示资源 | 建立共享资源库，再由工具和模型节点引用 | **VTK**：通常散落在 mapper/property/chart 体系里。**HOOPS**：以 `Portfolio` 为中心，`DefineGlyph`、`DefineTexture`、`DefineMaterialPalette` |

---

### 3A.2 视图与导航层

#### A. 视图系统

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 单视图显示 | 承载一个主三维场景，是所有交互工具的默认宿主 | 初始化窗口、渲染器、交互器，再挂载场景 | **VTK**：`vtkRenderer + vtkRenderWindow + vtkRenderWindowInteractor`。**HOOPS**：`Window -> Canvas -> Layout -> View -> Model` |
| 多视图布局 | 支持四视口、三视图+3D、结果对比等 CAE 布局 | 把模型与多个相机/视图组合起来 | **VTK**：通常多个 `vtkRenderer` 共享一个 `RenderWindow`，或多个 view 并列。**HOOPS**：`Layout` 是原生多视图容器，支持附着多个 `View` |
| 视图主题与显示模式 | 把线框、着色、隐藏线、结果色图等包装成视图级模式 | 一个视图切换一个模式，而不是重建整个场景 | **VTK**：常通过 mapper/property/interactor style 组合。**HOOPS**：常通过 `View` 段属性、渲染算法、样式切换实现 |
| 视图联动 | 支持一个视图中的选择、剖切、窗口级别影响其他视图 | 建立共享状态对象或回调桥接 | **VTK**：`vtkAnnotationLink`、共享 `LookupTable`、共享 callback。**HOOPS**：共享 `Model`、条件样式、工具事件、同一 selection/highlight 结果传播 |

#### B. 导航工具

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 旋转/平移/缩放 | 最基本的三维浏览能力 | 设置默认交互风格或 operator 栈 | **VTK**：`vtkInteractorStyleTrackballCamera`、`vtkInteractorStyleTerrain` 等。**HOOPS**：标准 `Operator`，常见是 orbit/pan/zoom/navigation operators |
| 图像浏览模式 | 用于切片数据、医学数据、结果切面等二维/2.5D 交互 | 切换到面向图像的专用交互模式 | **VTK**：`vtkInteractorStyleImage`。**HOOPS**：通常由自定义 operator 或图像/视图逻辑实现，不像 VTK 有单独强势模块 |
| 定位与自适应视野 | 快速把相机对准模型、所选对象或全局包围盒 | 在对象更新、导入、选择后调用“fit”逻辑 | **VTK**：`renderer->ResetCamera()`、`ResetCameraClippingRange()`。**HOOPS**：`View::FitWorld()`、`ComputeFitWorldCamera()` |
| 视角切换 | 快速切换前后左右上下、等轴测等工程视角 | 在界面中把标准视角作为命令暴露 | **VTK**：多通过相机姿态直接设置。**HOOPS**：通过相机控制或视图工具设置标准朝向 |

---

### 3A.3 选择与拾取层

#### A. 基础选择

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 点选 | 选择一个零件、单元、面、边、点 | 鼠标点击屏幕坐标，转换成场景命中 | **VTK**：`vtkPropPicker`、`vtkCellPicker`、`vtkPointPicker`，或 widget 内部 picking。**HOOPS**：`SelectionControl.SelectByPoint(...)` |
| 框选/区域选 | 一次选择多个对象 | 鼠标拖框后执行区域测试 | **VTK**：rubber-band 风格、picker、selection 流程组合。**HOOPS**：`SelectionControl.SelectByArea(...)` |
| 射线/体积选 | 在三维场景中做更精确的命中测试 | 常用于面、单元、点云、体对象拾取 | **VTK**：`Pick()` / `Pick3DPoint()` / `Pick3DRay()`。**HOOPS**：ray/point/area 选择入口配合 `SelectionOptionsKit` |
| 多级选择 | 决定选的是“零件”“几何体”“子实体”还是“节点” | 在选择前定义 level，再返回匹配结果 | **VTK**：通常依赖 picker 类型和 selection 数据模型。**HOOPS**：`SelectionOptionsKit` 配置 `level/algorithm/proximity/sorting` |

#### B. 选择联动

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 结果路径返回 | 拿到“命中了哪个对象以及从哪个场景路径命中”的完整信息 | 选择后读取 path，用于高亮、属性查询、坐标变换 | **VTK**：常拿 picker 命中对象，再结合应用层层级映射。**HOOPS**：`SelectionItem.ShowPath(selectionPath)` 直接返回 `KeyPath` |
| 视图间共享选择 | 一个视图选中的对象在其他视图同步选中 | 建立共享 selection 状态 | **VTK**：`vtkAnnotationLink` 是官方核心机制。**HOOPS**：常见做法是共享 `KeyPath`/selection 结果，再驱动各视图 highlight |
| 选择过滤 | 限制只选某些类型对象，如只选面、不选标注 | 在选择前声明筛选规则 | **VTK**：通常通过 picker 范围、pick list、数据类型控制。**HOOPS**：`SelectionOptionsKit` 和场景层过滤 |

---

### 3A.4 高亮与反馈层

#### A. 视觉反馈

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 选中高亮 | 让用户确认当前选中对象 | 在 selection 结果上套一层高亮样式 | **VTK**：常用 actor/property 改色、outline actor、widget 高亮。**HOOPS**：`HighlightControl.Highlight(path, options)` |
| 悬停高亮 | 提升交互可预见性 | 鼠标 move 时先做 lightweight picking，再临时反馈 | **VTK**：一般自定义 interactor observer。**HOOPS**：可用 operator + highlight control 组合 |
| 轮廓/包围盒反馈 | 不改变对象材质，只在外层描边或框出 | 适合不破坏原始材质的选中显示 | **VTK**：常见是 outline actor、overlay prop。**HOOPS**：常通过命名样式、overlay、高亮样式实现 |
| 文字/状态反馈 | 显示当前命中的名称、属性、坐标、值 | 选择后读取对象元数据，显示到 UI 或 overlay | **VTK**：callback + text actor / Qt UI。**HOOPS**：selection path + segment/name/query + text overlay |

---

### 3A.5 测量与标注层

#### A. 测量工具

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 距离测量 | 测两点间距离 | 用户放置两个点，系统显示长度值 | **VTK**：`vtkDistanceWidget`，常配 `vtkDistanceRepresentation2D/3D`。**HOOPS**：标准 measurement operator 或 `sprk_exchange_common_measurement_op.cpp` 一类测量算子 |
| 角度测量 | 测三点夹角或两边夹角 | 用户依次放置点或拖动控制柄 | **VTK**：`vtkAngleWidget`。**HOOPS**：测量 operator，自定义或 Exchange 扩展测量 |
| 双向尺寸/正交尺寸 | 用于医学/工程中的双轴尺寸评估 | 用四个端点定义两条正交尺寸线 | **VTK**：`vtkBiDimensionalWidget`。**HOOPS**：通常由自定义 measurement/redline operator 实现 |
| 面向 B-Rep 的精确测量 | 对 CAD 曲面/边界做更工程化测量 | 依赖 CAD 内核或导入模块返回拓扑语义 | **VTK**：原生弱，通常依赖上层几何内核。**HOOPS**：`Exchange` 相关 measurement operator 更自然 |

#### B. 标注工具

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 文本标注 | 给模型添加说明、结果说明、警告信息 | 在三维点或屏幕位置插入文字 | **VTK**：text actor、caption widget、2D/3D prop。**HOOPS**：`InsertText(...)`、markup/redline 风格工具 |
| 引线/说明框 | 用于 PMI、注释、问题标记 | 文本与目标对象之间建立视觉关联 | **VTK**：caption/annotation widgets。**HOOPS**：glyph、text、style、operator 组合 |
| 坐标轴/方向标 | 帮用户建立空间方向感 | 常驻在角落，以 overlay 形式显示 | **VTK**：`vtkOrientationMarkerWidget`。**HOOPS**：常由 overlay + glyph/geometry + local camera 实现 |
| 标尺/色标 | 显示结果值范围、颜色映射、标尺参考 | 把结果色图与 legend 挂到视图 | **VTK**：`vtkScalarBarWidget`、chart/2D context。**HOOPS**：更多依赖应用层 UI 或子窗口/overlay 自建 |

---

### 3A.6 剖切、切片与探针层

#### A. 剖切与裁剪

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 平面剖切 | 用一个平面切开模型或结果场 | 用户拖动平面，实时更新显示结果 | **VTK**：`vtkImplicitPlaneWidget2`，常与 `vtkPlane`、clip filter 联动。**HOOPS**：section plane、bounded section、operator 驱动切割 |
| 有界剖切 | 切割不再是无限平面，而是有限裁剪区域 | 常用于局部窗口、局部分析、工程视图 | **VTK**：多靠 filter 组合实现。**HOOPS**：bounded section 和 clip region 是更原生的能力 |
| 多边形裁剪区域 | 按任意轮廓保留或去除区域 | 在工程图/局部视图中特别实用 | **VTK**：通常需算法+filter 自组装。**HOOPS**：clip region 直接面向显示系统 |

#### B. 切片与重切片

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 图像切片平面 | 在体数据中生成一个可拖动切片平面 | 绑定图像数据和切片 widget | **VTK**：`vtkImagePlaneWidget`，内部依赖 `vtkImageReslice`。**HOOPS**：不是最强项，通常需上层配合图像模块实现 |
| 重切片光标 | 用于正交 MPR、多视图切面联动 | 一个切片光标驱动多个视图同步 | **VTK**：`vtkResliceCursorWidget` + `vtkResliceCursorRepresentation`。**HOOPS**：需要多视图+自定义工具联动 |
| 探针/取值 | 查看某位置的结果值、体素值、标量值 | 鼠标移动或点击时读取当前值 | **VTK**：picker + probe/filter 或 image widget 自带 cursor data。**HOOPS**：selection/operator 获取位置，再由应用层查询数据值 |

---

### 3A.7 结果可视化层

#### A. 科学图表

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| XY 曲线 | 显示历史结果、节点响应、路径曲线 | 准备表格数据，再把列映射到 chart | **VTK**：`vtkContextView` + `vtkChartXY` + `vtkPlot`。**HOOPS**：一般交给 Qt/WPF 图表控件，不是核心内建强项 |
| 多曲线对比 | 比较不同工况/不同测点结果 | 多列数据共享一个 chart 或多轴 | **VTK**：`vtkChartXY::AddPlot(...)`。**HOOPS**：应用层图表控件实现更常见 |
| 表格联动 | 图表与表格、数据表之间联动浏览 | 同一份表数据同时驱动 chart 与 table view | **VTK**：`vtkTable`、`vtkQtTableView`、chart view 组合。**HOOPS**：依赖 UI 框架 |

#### B. 结果映射

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 色图映射 | 把标量结果映射为颜色显示 | 定义 LUT，再把结果数组映射到几何 | **VTK**：mapper + LUT + scalar array。**HOOPS**：material palette、vertex/face color、texture/style 等 |
| 结果 legend | 辅助解释色图范围和单位 | 色图与 legend 始终绑定显示 | **VTK**：`vtkScalarBarWidget` 及相关 actor。**HOOPS**：常通过 overlay/子窗口/应用 UI 自建 |
| 多显示模式切换 | 在线框、实体、隐藏线、结果着色之间切换 | 把显示模式做成视图级命令 | **VTK**：property/mapper/interactor style 组合。**HOOPS**：渲染模式、样式系统和 view 控制更自然 |

---

### 3A.8 数据与算法层

#### A. CAE 数据导入

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 通用 CAE 结果读取 | 读取 CFD/FEA 文件，建立显示和分析数据对象 | 先由 reader 导入，再接后处理链路 | **VTK**：`IO/CGNS`、`Exodus`、`IOSS`、`EnSight`、`FLUENTCFF` 等 reader 模块。**HOOPS**：更偏 CAD/Exchange 导入，CAE 结果读取能力不如 VTK 丰富 |
| CAD 数据导入 | 读取 B-Rep、装配、PMI、tessellation | 导入后建立装配树与显示树 | **VTK**：可读部分几何格式，但 CAD 装配/PMI 产品化弱。**HOOPS**：`Exchange::File::Import` 是核心入口 |
| 流式/分块导入 | 面对超大模型或超大结果数据逐步加载 | 先载入粗粒度数据，再按需细化 | **VTK**：取决于 reader/pipeline 设计。**HOOPS**：OOC/deferral/区域加载机制更成熟 |

#### B. 后处理算法工具

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 等值面/等值线 | 从标量场提取结果边界 | reader 后接 contour/filter | **VTK**：`vtkContourFilter` 是典型入口。**HOOPS**：通常依赖外部算法后把结果当几何导入显示 |
| 阈值/提取 | 抽取满足条件的单元或区域 | 定义标量范围，再对数据集做提取 | **VTK**：`Filters/Selection/Extraction/Threshold` 一类模块。**HOOPS**：更多由应用层处理数据，再驱动显示 |
| 流线/张量/时间序列 | 处理 CFD、应力、瞬态结果 | 以 pipeline 方式串接算法模块 | **VTK**：`Filters/FlowPaths`、`Tensor`、`Temporal` 模块。**HOOPS**：原生不是强项 |

---

### 3A.9 性能与大模型层

#### A. 显示优化

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 静态场景优化 | 为不频繁变化的模型生成更高效的内部组织 | 导入完成后标记静态模型 | **VTK**：更多依赖数据缓存、静态 mesh、mapper/backend 优化。**HOOPS**：`StaticModel`、display lists、shadow tree 类机制 |
| 裁剪 | 不绘制不可见内容，降低绘制成本 | 开启视锥/背面/范围裁剪 | **VTK**：renderer/mapper/backend 与数据结构共同承担。**HOOPS**：`CullingControl`、extent/frustum/backplane culling |
| 固定帧率交互 | 导航时优先保证交互流畅，再补足细节 | 在交互模式下启用帧预算策略 | **VTK**：一般靠应用层或 backend 策略。**HOOPS**：`Canvas.SetFrameRate` 及 related mechanisms |
| overlay 重绘 | 只重绘正在交互的标记/工具，而不是整个场景 | 把辅助绘制对象放到 overlay 层 | **VTK**：一般用 overlay prop 或 2D layer 自建。**HOOPS**：`Drawing::Overlay::*` 是原生能力 |

#### B. 大模型与 OOC

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| Out-of-core | 数据不完全驻留内存，按需调页 | 面向超大装配或超大结果 | **VTK**：要看具体 reader/pipeline 支持。**HOOPS**：`OOC::*`、延迟加载和区域跟踪是明确产品能力 |
| 实例复用优化 | 重复部件只存一份几何 | 在导入阶段就把重复体抽象出来 | **VTK**：应用层常自己做。**HOOPS**：`IncludeSegment`、`ReferenceGeometry` |
| GPU/CPU 镜像与缓存 | 提升大型场景的稳定渲染性能 | 让内核自动管理 buffer 和缓存 | **VTK**：渲染 backend 负责。**HOOPS**：文档明确强调 CPU/GPU vertex buffer 镜像与自动管理 |

---

### 3A.10 应用集成层

#### A. 事件与工具挂接

| 功能 | 作用 | 如何使用 | 接口怎么用 |
|---|---|---|---|
| 工具事件回调 | 把交互动作接入业务逻辑 | 工具发事件，应用层监听并更新状态 | **VTK**：`AddObserver(...)` + callback 是标准模式。**HOOPS**：operator 回调、selection/highlight 结果、notifier 风格接口 |
| UI 框架集成 | 把渲染与工具嵌入 Qt/WPF/MFC 等桌面程序 | 先把窗口系统接好，再挂视图和工具 | **VTK**：`QVTKRenderWidget`、Qt support、custom render window。**HOOPS**：官方 samples 直接覆盖 Qt/MFC/WPF 等 |
| 工具状态与业务对象绑定 | 让“选中零件”“测量结果”“结果曲线”进入业务模型 | 用稳定 ID 或对象引用做桥接 | **VTK**：actor/picker/annotation link 与业务层做映射。**HOOPS**：`Key/KeyPath` 更适合作为桥接句柄 |

---

## 3B. 这棵树应该怎么用

如果你的目标是做一份“CAE 工具层规格清单”，最适合直接拿走的是下面这棵两层树：

```text
1. 场景架构层
   1.1 场景组织
   1.2 实例化与复用
   1.3 场景检索与定位
   1.4 属性与样式系统

2. 视图与导航层
   2.1 单视图显示
   2.2 多视图布局
   2.3 视图联动
   2.4 旋转/平移/缩放
   2.5 图像浏览模式
   2.6 定位与标准视角

3. 选择与拾取层
   3.1 点选
   3.2 框选/区域选
   3.3 射线/体积选
   3.4 多级选择
   3.5 选择联动
   3.6 选择过滤

4. 高亮与反馈层
   4.1 选中高亮
   4.2 悬停高亮
   4.3 包围盒/轮廓反馈
   4.4 文本与状态反馈

5. 测量与标注层
   5.1 距离测量
   5.2 角度测量
   5.3 双向尺寸
   5.4 B-Rep 精确测量
   5.5 文本标注
   5.6 引线/说明框
   5.7 方向标/标尺/色标

6. 剖切、切片与探针层
   6.1 平面剖切
   6.2 有界剖切
   6.3 多边形裁剪区域
   6.4 图像切片平面
   6.5 重切片光标
   6.6 探针/取值

7. 结果可视化层
   7.1 XY 曲线
   7.2 多曲线对比
   7.3 表格联动
   7.4 色图映射
   7.5 结果 legend
   7.6 多显示模式切换

8. 数据与算法层
   8.1 CAE 数据导入
   8.2 CAD 数据导入
   8.3 流式/分块导入
   8.4 等值面/等值线
   8.5 阈值/提取
   8.6 流线/张量/时间序列

9. 性能与大模型层
   9.1 静态场景优化
   9.2 裁剪
   9.3 固定帧率交互
   9.4 overlay 重绘
   9.5 Out-of-core
   9.6 实例复用优化

10. 应用集成层
   10.1 工具事件回调
   10.2 UI 框架集成
   10.3 工具状态与业务对象绑定
```

这棵树比“只列 widgets 或 operators”更接近你说的“完整工具层”，因为它把：

- 辅助绘制工具
- 交互工具
- 结果工具
- 场景架构分层
- 性能机制

都放进了同一套 CAE 语义里。

---

## 4. 工具层架构

## 4.1 VTK 的工具层架构

VTK 在工程上更像一个“模块叠加系统”，不是单独一块工具层。

### 4.1.1 自下而上的典型关系

```text
Data / Readers / Filters
    ->
vtkAlgorithm pipeline
    ->
vtkDataRepresentation
    ->
vtkView / vtkRenderViewBase / vtkContextView
    ->
vtkRenderer / vtkRenderWindow
    ->
vtkRenderWindowInteractor + vtkInteractorStyle
    ->
vtkAbstractWidget + vtkWidgetRepresentation
    ->
Application UI (Qt / custom app)
```

### 4.1.2 每层职责

- **Data / Filters / IO**
  - 负责读取 CAE 数据、执行后处理算法、组织 `vtkDataObject`
- **Representation**
  - 负责把数据转换成 view 可显示的对象
  - 这是 VTK 区别于 HOOPS 的关键层
- **View**
  - 负责一个应用视图如何管理 representations、选择、主题
- **InteractorStyle**
  - 负责相机/输入语义
- **Widget**
  - 负责“一个具体工具”的交互行为
  - 如距离测量、角度、切片平面、方向标、小部件叠加
- **WidgetRepresentation**
  - 负责这个工具在场景中的视觉表示

### 4.1.3 对 VTK 来说，工具层不是单体，而是 5 个模块族

| 模块族 | 代表目录 | 作用 |
|---|---|---|
| 交互风格 | `Interaction/Style` | 相机、图像、拾取、rubber band 等输入语义 |
| 交互控件 | `Interaction/Widgets` | 测量、切片、方向标、标尺、轮廓、平面、slider |
| 图像交互 | `Interaction/Image` | 图像 window/level、slice、医学交互模式 |
| 视图系统 | `Views/Core`、`Views/Context2D`、`Views/Infovis` | view/representation/linking/scene |
| 图表系统 | `Charts/Core` | XY、matrix、surface、parallel coordinates |

### 4.1.4 VTK 的关键架构特征

- **工具是挂在 interactor 上的**
  - 典型代码模式：`widget->SetInteractor(iren)`
- **工具可视表示与行为分离**
  - `vtkAbstractWidget` / `vtkWidgetRepresentation`
- **view 与 data 之间插入 representation**
  - 工具能和数据管线耦合，但不直接等于 scene graph node
- **linked selection 是 view 级能力**
  - 由 `vtkAnnotationLink` 和 `vtkDataRepresentation` 共享

### 4.1.5 典型 CAE 数据流

```text
CAE file reader
    ->
filters (clip/contour/threshold/probe/streamline)
    ->
representation
    ->
view
    ->
renderer
    ->
interactor style + widget
    ->
user interaction feeds back to filter / representation / selection link
```

这意味着 VTK 的上层工具并不只是“UI 控件”，而是和数据后处理流程深度绑定。

---

## 4.2 HOOPS Visualize 的工具层架构

HOOPS 的工具层组织更“产品化”，它不是围绕 pipeline，而是围绕 **scene graph + view system + controls/operators** 展开。

### 4.2.1 自下而上的典型关系

```text
Segment / Geometry / Portfolio / Attributes
    ->
Model
    ->
View
    ->
Layout
    ->
Canvas
    ->
Window
    ->
SelectionControl / HighlightControl / Operator stack / PerformanceControl
    ->
Application UI
```

### 4.2.2 每层职责

- **Segment**
  - 真正的场景图节点
  - 同时承载几何、属性、层级、include/reference 关系
- **Model**
  - 场景根和资源上下文
- **View**
  - 相机、渲染模式、operator 绑定、选择与高亮的主要入口
- **Layout / Canvas / Window**
  - 多视图组织、窗口承载、平台集成
- **Control / Kit / Operator**
  - 面向应用层的工具接口

### 4.2.3 对 HOOPS 来说，工具层的核心族群

| 能力族 | 代表对象 | 作用 |
|---|---|---|
| 交互 | `Operator`、标准 operators、自定义 operators | 导航、测量、框选、markup |
| 选择高亮 | `SelectionControl`、`HighlightControl`、`SelectionResults` | 命中测试、路径返回、样式高亮 |
| 样式与资源 | `Portfolio`、`NamedStyle`、`Glyph` | 工具外观、标注、条件样式 |
| 视图组织 | `Window/Canvas/Layout/View/Model` | 多视图和 UI 集成 |
| 性能控制 | `PerformanceControl`、`StaticModel`、overlay、OOC | 大模型浏览 |
| 导入 | `Exchange::File::Import`、`Stream::File::Import` | CAD / tessellation 输入 |

### 4.2.4 HOOPS 的关键架构特征

- **工具直接立在 View 之上**
  - 例如 operator stack、selection、highlight 都挂在 `View` / `Window`
- **工具直接作用于 scene graph path**
  - 结果对象天然是 `Key` / `KeyPath`
- **工具与样式系统天然结合**
  - 高亮、条件样式、overlay、深度层级都是现成机制
- **性能机制是场景系统的一部分**
  - 不是外部单独拼装

### 4.2.5 典型 CAE 数据流

```text
CAD/CAE import
    ->
Segment / Model construction
    ->
View attachment
    ->
Operator / Selection / Highlight / Style
    ->
user interaction returns Key / KeyPath / measurement events
    ->
scene graph or style updates
```

HOOPS 的工具层更偏“直接操纵显示场景”。

---

## 5. 从 CAE 角度比较两套架构

| 对比项 | VTK | HOOPS Visualize |
|---|---|---|
| 上层工具的核心依托 | pipeline + representation + widget | scene graph + controls + operators |
| 最重要的中间层 | `vtkDataRepresentation` | `Model/View/Segment` |
| 交互工具挂接点 | `Interactor` / `InteractorStyle` / widget | `View` / `Window` / operator stack |
| 选择结果 | selection + picker + annotation link | `SelectionResults` + `KeyPath` |
| 数据处理能力 | 很强 | 相对弱于 VTK |
| 场景管理能力 | 中等，需要自己搭 | 很强，原生就是 scene graph |
| 产品化 UI 能力 | 需要应用层封装 | 官方 API 已明显朝应用层暴露 |
| 更适合的 CAE 任务 | 后处理平台、算法可视化平台 | CAD/CAE 浏览器、装配查看器、交互式工程软件 |

### 5.1 一个非常关键的判断

如果你的目标是：

- **做“仿真后处理系统”**，VTK 的工具层更自然。
- **做“工程模型浏览与交互平台”**，HOOPS 的工具层更自然。
- **做完整 CAD/CAE 产品**，常见模式反而是：
  - VTK 负责后处理算法与科学可视化
  - HOOPS 负责模型浏览、交互和产品级显示框架

这是本文基于文档和样例做出的工程判断。

---

## 6. 文档章节架构分析

## 6.1 VTK 官方文档是怎么组织“工具层”的

VTK 官网并没有单独开一个“Tool Layer”总章，而是拆散在下面几类入口里：

### 6.1.1 入口层

- `Documentation Home`
- `Modules`
- `Supported Data Formats`
- `Learning`
- `Examples`
- `VTK Book`

### 6.1.2 真正和工具层最相关的章节族

| 文档入口 | 关注内容 | 对应 CAE 工具层含义 |
|---|---|---|
| `Modules` | 各模块能力与依赖 | 看出工具层被拆成哪些模块 |
| `Supported Data Formats` | readers/writers/module | 看 CAE 数据入口 |
| `VTK Book Chapter 4` | graphics model / pipeline | 看 view-representation-pipeline 基础 |
| `VTK Book Chapter 7` | advanced graphics | 看渲染之上的图形/交互语境 |
| `VTK Book Chapter 12` | application classes | 看医学、工程、后处理落地场景 |
| `Examples` | widgets / plotting / imaging / visualization | 看工具如何落代码 |

### 6.1.3 如果你要按“工具层”重排 VTK 文档，建议章节树写成这样

```text
1. Tool Layer Overview
2. View & Representation Architecture
3. Interaction Styles
4. Widgets
   4.1 Measurement Widgets
   4.2 Slice / Plane / Reslice Widgets
   4.3 Orientation / Annotation Widgets
5. Linked Views and Selection
6. Charts / Context2D / Tables
7. CAE Data IO
8. CAE Post-processing Filters
9. Performance and Large Data
10. Example Programs
```

这比官网原始目录更贴近 CAE 软件开发者的阅读路径。

---

## 6.2 HOOPS 官方文档是怎么组织“工具层”的

HOOPS 的文档结构对“工具层”更友好，因为它本身就把交互、场景、性能写成产品能力。

### 6.2.1 最关键的章节族

| 文档入口 | 关注内容 | 对应工具层含义 |
|---|---|---|
| `Technical Overview` | 全景能力图 | 总览 scene graph、selection、overlays、performance |
| `Programming Guide / Database` | segment tree / key / search | 上层场景模型 |
| `Programming Guide / API Conventions` | Key / Kit / Control / Operator 风格 | 工具 API 风格 |
| `Programming Guide / Operators` | 标准交互算子与自定义算子 | 导航、测量、交互入口 |
| `Guides / Selection & Highlighting` | 选择、高亮、area select | CAE 常用交互 |
| `Performance Considerations` | display list / culling / fixed framerate | 大模型工具层支撑 |
| `Static Model Guide` | scene optimization | 静态场景组织优化 |

### 6.2.2 如果你要按“工具层”重排 HOOPS 文档，建议章节树写成这样

```text
1. Tool Layer Overview
2. Scene Graph and View System
3. API Style
4. Operators
   4.1 Navigation
   4.2 Selection
   4.3 Measurement
   4.4 Custom Operators
5. Selection / Highlight / Markup
6. Layout / View / Window Integration
7. Style / Portfolio / Overlay
8. CAD Import and Data Access
9. Performance / Static Model / OOC
10. Example Programs
```

HOOPS 文档天然就比 VTK 更容易整理成“应用工具层手册”。

---

## 7. 本地 example 程序映射分析

## 7.1 VTK 本地 example / test 能说明什么

本文重点参考了下面这些本地程序：

- `Examples/Medical/Cxx/Medical3.cxx`
- `Examples/Charts/Cxx/QChartTable.cxx`
- `Interaction/Widgets/Testing/Cxx/TestDistanceWidget.cxx`
- `Interaction/Widgets/Testing/Cxx/TestImplicitPlaneWidget2.cxx`
- `Interaction/Widgets/Testing/Cxx/TestOrientationMarkerWidget.cxx`
- `Interaction/Widgets/Testing/Cxx/TestResliceCursorWidget2.cxx`

### 7.1.1 它们揭示出的 VTK 工具层代码风格

VTK 的典型模式是：

```cpp
reader / filter -> mapper / representation -> renderer
-> renderWindow -> interactor
-> interactorStyle / widget
-> callback / observer
```

关键特征：

- 工具对象基本都需要 `SetInteractor(...)`
- 很多工具需要显式创建 `Representation`
- 业务联动靠 `AddObserver(...)` / callback
- 多视图联动要自己显式把 widgets / planes / LUT / callback 串起来

### 7.1.2 这说明 VTK 的工具层偏“可组装”

- 优点
  - 灵活
  - 可以深度接入数据处理管线
  - 非常适合做定制后处理交互
- 缺点
  - 应用层样板代码多
  - 统一性不如 HOOPS
  - 要自己搭很多“产品化 glue code”

## 7.2 HOOPS 本地 example 能说明什么

本文主要沿用了之前已经核对过的这些样例：

- `samples/code/cpp/sample.cpp`
- `samples/code/cpp/samples/database_search.cpp`
- `samples/code/cpp/samples/selection.cpp`
- `samples/code/cpp/samples/reference_geometry.cpp`
- `samples/code/cpp/samples/conditional_styles_and_includes.cpp`
- `samples/operators/sprk_exchange_common_measurement_op.cpp`
- `samples/qt_quick_sandbox/quickcanvas.cpp`
- `samples/mfc_ooc_sandbox/CHPSDoc.cpp`
- `samples/wpf_sandbox/source/SegmentBrowser/SegmentBrowser.cs`

### 7.2.1 它们揭示出的 HOOPS 工具层代码风格

HOOPS 的典型模式是：

```cpp
create model/view/window
-> build Segment scene graph
-> attach operators / controls
-> selection/highlight returns keys or key paths
-> update style / segment / view
```

关键特征：

- 视图系统骨架很统一
  - `Window -> Canvas -> Layout -> View -> Model`
- 交互用 operator stack
- 场景查询用 `Key` / `KeyPath`
- 工具效果常常直接映射成 style / segment / overlay 操作

### 7.2.2 这说明 HOOPS 的工具层偏“现成产品能力”

- 优点
  - 统一
  - 对场景交互更直接
  - 更适合做大型工程查看器
- 缺点
  - 数据后处理算法生态不如 VTK
  - 科学绘图和数据分析能力弱于 VTK

---

## 8. 站在 CAE 架构师角度，应该怎么理解两者

### 8.1 VTK 的工具层本质

VTK 的工具层不是一个“现成应用层”，而是：

- 以数据管线为中心
- 以 widget / view / chart 为补充
- 最终由应用层把它们拼成产品

因此它特别适合：

- 仿真后处理
- 科学可视化平台
- 图像和体数据交互
- 强算法驱动的 CAE 场景

### 8.2 HOOPS 的工具层本质

HOOPS 的工具层更像：

- 以场景图和视图系统为中心
- 以 operator / selection / styles 为主要应用接口
- 直接面向产品 UI 与大模型交互

因此它特别适合：

- CAD/CAE 浏览器
- 装配模型查看器
- 交互式工程软件
- 强产品交互驱动的 CAE 场景

---

## 9. 给你的最终建议

如果你的目标是“列出面向 CAE、在渲染器之上的工具层能力”，推荐最终采用下面这套统一目录：

```text
1. 工具层定义与边界
2. 视图与交互架构
3. 导航工具
4. 选择 / 拾取 / 高亮
5. 测量与标注
6. 剖切 / 切片 / 探针 / 重切片
7. 多视图与联动
8. 图表与二维结果视图
9. CAE 数据导入
10. 后处理算法工具
11. 大模型与性能工具
12. 官方文档入口结构
13. example 程序映射
14. VTK vs HOOPS 对比结论
```

如果按这套结构继续扩写：

- **VTK** 重点写 `Filters + Views + Widgets + Charts + IO`
- **HOOPS** 重点写 `Scene Graph + Operators + Selection/Highlight + Performance + Exchange`

---

## 10. 参考来源

### 10.1 VTK 官方文档

- https://docs.vtk.org/en/latest/index.html
- https://docs.vtk.org/en/latest/modules/index.html
- https://docs.vtk.org/en/latest/supported_data_formats.html
- https://book.vtk.org/en/latest/VTKBook/04Chapter4.html
- https://book.vtk.org/en/latest/VTKBook/07Chapter7.html
- https://book.vtk.org/en/latest/VTKBook/12Chapter12.html
- https://examples.vtk.org/site/Cxx/

### 10.2 HOOPS 官方文档

- https://docs.techsoft3d.com/hoops/visualize-desktop/general/technical_overview.html
- https://docs.techsoft3d.com/hoops/visualize-desktop/prog_guide/0101_database.html
- https://docs.techsoft3d.com/hoops/visualize-desktop/prog_guide/0102_api_conventions.html
- https://docs.techsoft3d.com/hps/2024/prog_guide/0601_standard_operators.html
- https://docs.techsoft3d.com/hps/2024.8.0/prog_guide/0602_custom_operators.html
- https://docs.techsoft3d.com/hps/2024/guides/04_selection_highlighting.html
- https://docs.techsoft3d.com/hoops/visualize-desktop/prog_guide/0703_performance_considerations.html
- https://docs.techsoft3d.com/hoops/visualize-desktop/guides/06_static_model.html

### 10.3 本地样例/源码

- `D:\github-repos\VTK`
- `D:\github-repos\RoadMap2026\HOOPS_Visualize_2026.1.0`
