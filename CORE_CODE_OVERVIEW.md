# Differential Gaussian Rasterization 核心代码概览

## 1. 核心代码文件

下表列出该仓库中直接实现 Differential Gaussian Rasterization 核心流程的 C 源文件与头文件。

| 文件 | 类型 | 作用 |
| --- | --- | --- |
| `cpu_rasterizer/rasterizer_impl.c` | C 源文件 | 顶层前向渲染入口，负责组织预处理、分桶排序、按 tile 建立范围并调用最终渲染。 |
| `cpu_rasterizer/forward.c` | C 源文件 | 实现高斯投影与渲染主体，包括 SH 颜色计算、3D/2D 协方差处理、像素级 alpha 合成。 |
| `cpu_rasterizer/auxiliary.c` | C 源文件 | 提供几何与矩阵辅助函数，包括包围矩形计算、坐标变换、视锥裁剪判定。 |
| `cpu_rasterizer/rasterizer.h` | 头文件 | 对外导出的 C API 声明，定义 `cpu_rasterizer_forward` 与可见性检测接口。 |
| `cpu_rasterizer/forward.h` | 头文件 | 声明预处理与渲染阶段函数，连接顶层调度与算法实现。 |
| `cpu_rasterizer/auxiliary.h` | 头文件 | 定义 `float2`、`float3`、`float4`、`uint2`、`dim3` 等基础数据结构及辅助函数声明。 |
| `cpu_rasterizer/config.h` | 头文件 | 提供 `NUM_CHANNELS`、`BLOCK_X`、`BLOCK_Y` 等编译期配置，影响 tile 大小与通道数。 |

## 2. 非核心代码文件

该仓库除了核心 C 实现，还包含绑定层、构建脚本和编译中间产物。

`ext.cpp`、`rasterize_points.cpp`、`rasterize_points.h` 是 PyTorch 扩展/接口层，作用是把 Python 端张量调用转发到 C 核心函数，不直接承载高斯光栅化主算法。`setup.py`、`CMakeLists.txt`、`c_diff_gaussian_rasterization/__init__.py` 属于构建与打包文件，用于编译扩展、组织 Python 包加载流程。`README.md`、`LICENSE.md` 是项目说明与许可证文本。

`out/` 目录中的 `*.mlir`、`*.ll`、`*_optimized.ll`、`*_optimized.mlir`、`*_output.s`、`*_riscv.s`、`script.sh` 是编译链路的中间表示与目标汇编产物，主要用于编译实验、优化观察或后端代码生成验证，不是手写核心算法源代码。

## 3. 核心代码逻辑讲解

### 3.1 总体流程

核心执行入口是 `cpu_rasterizer_forward`。流程可概括为预处理、分桶与排序、tile 范围识别、tile 内像素合成四个阶段。

首先，代码分配 `GeometryState`、`ImageState`、`BinningState` 三类运行时状态缓冲区，分别承载高斯几何中间量、图像累积量、分桶排序缓冲。随后调用 `cpu_rasterizer_preprocess`，把每个 3D 高斯转换为屏幕空间可渲染参数。

预处理后，程序根据每个高斯覆盖 tile 数得到前缀偏移，调用 `cpu_rasterizer_duplicateWithKeys` 把“高斯-被覆盖 tile”对展开为键值条目；键的高 32 位是 tile id，低 32 位是深度比特表示。之后以 key 排序，得到按 tile 且按深度顺序的点列表，再由 `cpu_rasterizer_identifyTileRanges` 写出每个 tile 在点列表中的连续区间。最后 `cpu_rasterizer_render` 在每个 tile 内逐像素执行高斯贡献累积并输出颜色。

### 3.2 预处理阶段（`forward.c`）

`preprocessCPU` 对每个高斯执行以下关键步骤：

1. 视锥检测：通过 `cpu_rasterizer_in_frustum` 剔除不可见点。
2. 投影与协方差：将 3D 均值投影到 NDC/像素空间，调用 `computeCov3D` 与 `computeCov2D` 得到屏幕空间椭圆参数。
3. 包围半径与覆盖 tile：由特征值估计半径，再用 `cpu_rasterizer_getRect` 得到 tile 覆盖矩形。
4. 颜色准备：若未提供预计算颜色，则 `computeColorFromSH` 按球谐系数和视线方向计算 RGB。
5. 写回中间结果：包括 `depths`、`radii`、`means2D`、`conic_opacity`、`tiles_touched`。

其中 `conic_opacity` 将 2D 协方差逆矩阵的三个独立分量与不透明度打包，供后续像素着色快速读取。

### 3.3 分桶与排序阶段（`rasterizer_impl.c`）

`cpu_rasterizer_duplicateWithKeys` 把每个高斯按其 tile 覆盖范围复制为多条记录，并写入 `point_list_keys_unsorted`、`point_list_unsorted`。排序后，同一 tile 的记录在数组中连续存放，且深度顺序稳定可用于前向 alpha 合成。`cpu_rasterizer_identifyTileRanges` 扫描有序 key，生成每个 tile 对应的 `[start, end)` 区间，避免渲染阶段全局遍历所有高斯。

### 3.4 像素合成阶段（`forward.c`）

`renderCPU` 以 tile 为单位遍历像素，对当前 tile 范围内的高斯做贡献评估。每个高斯先计算像素偏移对应的二次型指数项，再得到 alpha；若 alpha 太小则跳过。颜色按“当前透过率 `T` × alpha × 特征颜色”累积，并更新透过率 `T = T*(1-alpha)`；当 `T` 足够小时提前终止。最终像素输出为累积前景颜色与背景色 `T*bg` 之和。

### 3.5 主要数据结构

`GeometryState` 管理每个高斯的几何中间量，如深度、2D 坐标、协方差/圆锥参数、颜色、覆盖 tile 数。`BinningState` 管理展开后的键值列表与排序缓存。`ImageState` 管理 tile 到点列表区间映射、像素累积透过率、像素贡献计数。三者配合形成“高斯参数准备—tile 索引加速—像素合成输出”的完整 CPU 光栅化流水线。
