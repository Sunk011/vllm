---
kind: dependency_management
name: vLLM 多语言依赖管理体系
category: dependency_management
scope:
    - '**'
source_files:
    - pyproject.toml
    - setup.py
    - requirements/common.txt
    - requirements/cuda.txt
    - requirements/cpu.txt
    - requirements/rocm.txt
    - requirements/dev.txt
    - rust/Cargo.toml
    - rust/Cargo.lock
    - .github/dependabot.yml
    - cmake/external_projects/deepgemm.cmake
    - docker/Dockerfile
---

vLLM 项目采用多语言、多层次的依赖管理策略，涵盖 Python、Rust、C++/CUDA 原生扩展以及容器化构建环境。以下是其依赖管理的核心体系：

## 1. Python 依赖管理（requirements 分层体系）

**核心文件结构：**
- `requirements/common.txt` - 所有平台共享的基础依赖
- `requirements/cuda.txt` / `requirements/rocm.txt` / `requirements/cpu.txt` / `requirements/xpu.txt` / `requirements/tpu.txt` - 按硬件后端分层的依赖
- `requirements/dev.txt` / `requirements/lint.txt` / `requirements/docs.txt` - 开发工具链依赖
- `requirements/test/*.txt` - 测试环境特定依赖
- `requirements/build/` - 构建时依赖

**版本锁定机制：**
- 使用 `.in` 源文件配合 uv 生成锁定的 `.txt` 文件（如 `cuda.in` → `cuda.txt`）
- 通过 `-r common.txt` 引用基础依赖，实现依赖继承
- 关键依赖精确版本控制：`torch==2.11.0`、`transformers >= 5.5.3`、`tokenizers >= 0.21.1`

**特殊依赖处理：**
- PyPI 镜像配置：`--extra-index-url https://flashinfer.ai/whl/` 用于 flashinfer 包
- 平台条件依赖：`lm-format-enforcer == 0.11.3` 和 `llguidance` 等根据架构条件安装
- 安全补丁：对 protobuf、starlette 等存在 CVE 的包进行版本限制

## 2. Rust 依赖管理（Cargo Workspace）

**工作区结构：**
- `rust/Cargo.toml` - 顶层 workspace 配置，统一管理 14 个子模块
- `rust/Cargo.lock` - 完整的依赖锁定文件
- 各子模块独立的 `Cargo.toml` 文件

**依赖集中管理：**
- 在 `[workspace.dependencies]` 中统一定义版本，确保一致性
- 内部 crate 通过路径引用：`vllm-chat = { path = "src/chat" }`
- Git 依赖精确版本：`llm-multimodal` 和 `openai-harmony` 指定 git commit/tag

**性能优化配置：**
- 针对 tokenizer、regex-automata 等热点包启用优化编译
- 发布模式启用 LTO：`[profile.release] lto = "thin"`

## 3. C++/CUDA 原生扩展依赖

**CMake 外部依赖：**
- `cmake/external_projects/` 目录管理第三方库：deepgemm、flashmla、triton_kernels 等
- 通过 FetchContent 机制下载和管理外部依赖
- 支持 CUDA、ROCm、CPU 多后端编译

**预编译二进制支持：**
- `setup.py` 中的 `precompiled_build_ext` 类支持跳过本地编译
- 从 wheels.vllm.ai 下载预编译的 .so 扩展
- 自动检测 CUDA/ROCm/XPU 环境并选择对应二进制

## 4. 依赖更新与自动化

**Dependabot 集成：**
- `.github/dependabot.yml` 配置每周自动检查依赖更新
- 忽略特定依赖的安全更新：torch、xformers、ray[cgraph] 等
- 分组管理次要更新：`minor-update` 组聚合相关依赖

**构建系统依赖：**
- `pyproject.toml` 定义构建依赖：cmake>=3.26.1、setuptools>=77.0.3、torch==2.11.0
- setuptools-rust 集成 Rust 编译流程
- 支持 sccache/ccache 加速编译

## 5. 容器化依赖管理

**Docker 多架构支持：**
- 多个 Dockerfile 针对不同平台：CUDA、ROCm、TPU、XPU、s390x、ppc64le
- `docker/versions.json` 统一管理版本信息
- 基于 Alpine/Ubuntu 的基础镜像优化

**依赖隔离策略：**
- 构建时使用 `no-build-isolation-package=["torch"]` 避免隔离问题
- 运行时依赖通过 requirements 文件精确控制
- 预编译 wheel 包含所有必要的二进制依赖

## 6. 第三方内核管理

**vendored 代码管理：**
- `vllm/third_party/` 目录包含 vendored 的第三方代码：triton_kernels、deep_gemm、fmha_sm100、tml_fa4
- 构建时从预编译 wheel 提取或本地编译
- 通过 `MANIFEST.in` 控制打包内容

这种多层级的依赖管理体系确保了 vLLM 在不同硬件平台、Python 版本和部署环境下的稳定性和可重复性。