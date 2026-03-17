# 非核心代码文件说明

本文档专门说明本仓库中“非 Differential Gaussian Rasterization 核心算法实现”的文件及其作用。这里的“核心算法”指 `cpu_rasterizer/` 下直接执行高斯预处理、分桶排序、光栅化合成的 C 源码与头文件；其余文件均归入非核心代码。

## 1. 构建与工程配置文件

`CMakeLists.txt`、`setup.py` 用于配置 C/C++ 扩展编译与 Python 打包安装流程，不承载光栅化算法本体。

## 2. Python 包与绑定入口文件

`c_diff_gaussian_rasterization/__init__.py` 负责 Python 侧模块导出与加载入口，是调用封装层而非算法主体实现。

## 3. C++/PyTorch 封装与接口文件

`ext.cpp`、`rasterize_points.cpp`、`rasterize_points.h` 负责把外部调用（通常是 PyTorch 张量接口）桥接到 C 光栅化内核，属于绑定层和 API 适配层。

## 4. 文档与许可证文件

`README.md` 用于项目介绍、安装与使用说明，`LICENSE.md` 用于许可证声明，`CORE_CODE_OVERVIEW.md` 是核心代码解读文档；三者均为说明性内容。

## 5. out 目录中的中间产物与脚本

`out/` 目录下文件均不属于手写核心算法源码，主要用于编译链路分析、目标代码生成验证和产物检查。

### 5.1 MLIR 文件（中间表示）

`out/auxiliary.mlir`、`out/auxiliary_optimized.mlir`、`out/forward.mlir`、`out/forward_optimized.mlir`、`out/rasterizer_impl.mlir`、`out/rasterizer_impl_optimized.mlir`：分别对应 `auxiliary`、`forward`、`rasterizer_impl` 模块在优化前后的 MLIR 表示，用于编译器 IR 级别分析与优化观察。

### 5.2 LLVM IR 文件（低层中间表示）

`out/auxiliary.ll`、`out/auxiliary_optimized.ll`、`out/forward.ll`、`out/forward_optimized.ll`、`out/rasterizer_impl.ll`、`out/rasterizer_impl_optimized.ll`：对应各模块优化前后 LLVM IR，用于进一步的后端优化、代码生成调试与性能排查。

### 5.3 汇编文件（目标代码）

`out/auxiliary_output.s`、`out/forward_output.s`、`out/rasterizer_impl_output.s`：通用目标汇编输出，用于检查函数级代码生成结果。

`out/auxiliary_riscv.s`、`out/forward_riscv.s`、`out/rasterizer_impl_riscv.s`：RISC-V 目标汇编输出，用于跨架构后端验证或教学/实验场景下的指令级分析。

### 5.4 脚本文件

`out/script.sh`：驱动或复现实验编译流程的脚本文件，用于自动化生成上述 IR/汇编产物。

## 6. 结论

非核心代码主要承担工程构建、语言绑定、文档说明、编译中间产物输出和实验脚本自动化职责，核心算法本体仍集中在 `cpu_rasterizer/` 目录的 C 代码中。
