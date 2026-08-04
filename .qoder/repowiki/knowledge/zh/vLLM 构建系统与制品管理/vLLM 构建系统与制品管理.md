---
kind: build_system
name: vLLM 构建系统与制品管理
category: build_system
scope:
    - '**'
source_files:
    - setup.py
    - pyproject.toml
    - CMakeLists.txt
    - docker/Dockerfile
    - .buildkite/ci_config.yaml
    - tools/build_rust.py
    - build_rust.sh
    - requirements/common.txt
    - requirements/cuda.txt
    - requirements/rocm.txt
    - requirements/cpu.txt
    - requirements/xpu.txt
    - requirements/tpu.txt
    - cmake/utils.cmake
    - cmake/cpu_extension.cmake
    - docker/docker-bake.hcl
    - docker/versions.json
---

vLLM 采用多语言混合构建体系，以 Python setuptools 为统一入口，集成 CMake（C++/CUDA/HIP）、Rust（PyO3）与 Docker 镜像构建，并通过 BuildKite CI 流水线完成跨平台发布。核心设计围绕「预编译轮子 + 本地可选编译」的双模策略，兼顾开发灵活性与安装速度。

**1. 构建系统架构**
- **Python 层**: `setup.py` 是统一入口，通过 `VLLM_TARGET_DEVICE` 环境变量自动检测 CUDA/ROCm/CPU/XPU/TPU 后端，动态选择依赖文件（`requirements/cuda.txt`、`rocm.txt`、`cpu.txt` 等）和扩展模块。
- **C++/CUDA 层**: 使用 CMake 管理原生扩展，支持 CUDA 与 HIP（ROCm）双后端，通过 `CMakeLists.txt` 中的 `define_extension_target` 宏定义各算子目标（`_C_stable_libtorch`、`cumem_allocator`、`spinloop`、`fs_io_C` 等），并按 GPU 架构条件编译（SM75-SM120 系列）。CUTLASS、DeepGEMM、FlashInfer 等第三方库通过 FetchContent 拉取。
- **Rust 层**: 基于 `setuptools-rust` 构建 PyO3 绑定（`vllm._rust_tool_parser`）和独立二进制（`vllm-rs`），由 `tools/build_rust.py` 统一编排，支持 `--debug`/`--release` 模式。
- **Docker 层**: 多阶段构建（base → rust-build → csrc-build → extensions-build → build → vllm-base → test → vllm-openai），每个阶段职责分离，利用 BuildKit 缓存和并行构建加速。

**2. 关键构建流程**
- **本地构建**: `python setup.py bdist_wheel --py-limited-api=cp38`，自动触发 CMake 配置与编译，支持 sccache/ccache 加速。
- **预编译轮子**: 通过 `VLLM_USE_PRECOMPILED=1` 跳过本地编译，从 `wheels.vllm.ai` 或 AMD PyPI 索引下载对应架构的 `.whl`，提取 `.so` 文件和 Rust 前端。
- **版本管理**: 使用 `setuptools-scm` 从 Git 标签生成版本号，附加设备后缀（`+cu129`、`+rocm6.2`、`+cpu`、`+xpu`、`+tpu`），支持 `VLLM_VERSION_OVERRIDE` 覆盖。
- **依赖解析**: `get_requirements()` 根据 `_is_cuda()/hip()/cpu()/xpu()/tpu()` 选择不同 `requirements/*.txt`，并处理 CUDA 版本特定的包名替换（如 `humming-kernels[cu13]` → `[cu12]`）。

**3. 多后端支持**
- **CUDA**: 默认后端，支持 SM7.5-SM12.0，按架构裁剪编译（如 Hopper 专属的 Machete/Marlin/NVFP4 内核），CUDA ≥12.3 启用 FA3，≥12.9 启用 FlashMLA。
- **ROCm**: 通过 `VLLM_TARGET_DEVICE=rocm` 切换，使用 HIP 编译器，链接 `libamdhip64`，支持 gfx906-gfx1201 架构。
- **CPU**: 仅构建纯 C++ 扩展（`_C`、`_C_AVX512`、`_C_AVX2`），Linux x86_64/aarch64 自动捆绑 tcmalloc 提升性能。
- **XPU/TPU**: 通过 `torch.version.xpu`/`torch.version.hip` 自动检测，分别加载对应后端依赖。

**4. 容器化与 CI**
- **Docker 构建**: `docker/Dockerfile` 定义 7 个阶段，`docker/docker-bake.hcl` 和 `docker/versions.json` 参数化 CUDA/Python/Ubuntu 版本，支持 manylinux 基础镜像。
- **BuildKite CI**: `.buildkite/ci_config.yaml` 定义触发规则（修改 `CMakeLists.txt`、`setup.py`、`csrc/`、`cmake/` 时全量重跑），分 `premerge` 与 `postmerge` 仓库，支持 ROCm/CUDA/XPU 并行测试。
- **镜像变体**: 提供 `vllm-openai`（OpenAI API Server）、`vllm-openai-nonroot`（非 root 用户）、`vllm-sagemaker`（SageMaker 适配）等目标。

**5. 构建优化与约束**
- **并行控制**: `MAX_JOBS` 控制 Ninja 并发，`NVCC_THREADS` 限制 nvcc 线程数，`CARGO_BUILD_JOBS=4` 防止 Rust 构建耗尽文件描述符。
- **缓存策略**: sccache 用于 C++/Rust 增量编译，ccache 作为备选；`.deps` 目录缓存 FetchContent 依赖，`/opt/uv/cache` 缓存 Python 包。
- **强制要求**: GCC ≥11.3（PyTorch C++20 头文件需求），CMake ≥3.26，Ninja 优先，Python 3.10-3.14。
- **安全约束**: 禁用 root 运行（nonroot 变体），限制 wheel 大小（`VLLM_MAX_SIZE_MB=500`），校验 wheel 完整性（sha256sum）。

**6. 扩展点与插件机制**
- **自定义后端**: 通过 `VLLM_TARGET_DEVICE` 和环境变量注入新后端，在 `setup.py` 中注册新的 `CMakeExtension`。
- **可选功能**: extras_require 定义 `zen`、`helion`、`grpc`、`otel` 等可选依赖，按需安装。
- **第三方内核**: `cmake/external_projects/` 管理 DeepGEMM、FlashMLA、Qutlass 等外部项目，支持源码目录覆盖（`VLLM_CUTLASS_SRC_DIR`）。

该构建系统通过分层抽象（Python 入口 → CMake/Rust → Docker）和多后端条件编译，实现了单一代码库对异构硬件的统一支持，同时保持开发体验与生产部署的一致性。