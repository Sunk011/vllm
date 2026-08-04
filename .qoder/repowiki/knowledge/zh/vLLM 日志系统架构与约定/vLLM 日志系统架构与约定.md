---
kind: logging_system
name: vLLM 日志系统架构与约定
category: logging_system
scope:
    - '**'
source_files:
    - vllm/logger.py
    - vllm/logging_utils/__init__.py
    - vllm/logging_utils/formatter.py
    - vllm/logging_utils/access_log_filter.py
    - vllm/logging_utils/log_time.py
---

## 系统概述

vLLM 使用 Python 标准库 `logging` 模块作为核心日志框架，通过集中式配置和自定义格式化器实现结构化、可配置的日志输出。系统支持彩色终端输出、文件追踪、一次性日志等高级功能。

## 核心组件

### 主要配置文件
- `vllm/logger.py`: 日志系统的核心配置和初始化逻辑
- `vllm/logging_utils/`: 自定义格式化器和工具（包含 ColoredFormatter、NewLineFormatter 等）
- `vllm/envs.py`: 环境变量定义和控制开关

### 关键特性
1. **动态颜色支持**: 根据输出目标自动选择彩色或纯文本格式
2. **作用域控制**: 支持 process/global/local 三种日志作用域
3. **一次性日志**: 提供 debug_once、info_once、warning_once 方法避免重复日志
4. **函数调用追踪**: 内置 sys.settrace 实现的函数调用跟踪功能
5. **配置灵活性**: 支持 JSON 配置文件和环境变量双重配置方式

## 架构设计

### 日志级别策略
- 默认使用 `VLLM_LOGGING_LEVEL` 环境变量控制全局日志级别
- 自动抑制 httpx 等第三方库的冗余日志
- 支持通过 `suppress_logging` 上下文管理器临时禁用日志

### 格式化器设计
- `ColoredFormatter`: 支持 ANSI 颜色的彩色输出格式化器
- `NewLineFormatter`: 处理多行消息的标准格式化器
- 统一的输出格式：`[时间戳] [文件名:行号] 消息内容`

### 分布式感知
- 集成分布式状态检查，支持按 rank 过滤日志输出
- 提供 `_should_log_with_scope` 函数判断是否应该记录日志

## 配置机制

### 环境变量控制
- `VLLM_CONFIGURE_LOGGING`: 是否启用 vLLM 日志配置
- `VLLM_LOGGING_LEVEL`: 日志级别设置
- `VLLM_LOGGING_STREAM`: 输出流配置（stdout/stderr/文件路径）
- `VLLM_LOGGING_COLOR`: 颜色输出开关
- `VLLM_LOGGING_CONFIG_PATH`: 外部 JSON 配置文件路径
- `VLLM_TRACE_FUNCTION`: 启用函数调用追踪

### 配置优先级
1. 外部 JSON 配置文件（最高优先级）
2. 环境变量配置
3. 默认配置模板

## 使用约定

### 日志记录模式
```python
from vllm.logger import init_logger
logger = init_logger(__name__)

# 标准日志记录
logger.info("正常信息")
logger.debug("调试信息")
logger.warning("警告信息")

# 一次性日志（避免重复）
logger.info_once("只输出一次的警告", scope="local")
```

### 作用域控制
- `scope="process"`: 仅主进程输出
- `scope="global"`: 仅全局第一个 rank 输出  
- `scope="local"`: 仅本地第一个 rank 输出

### 调试工具
- `enable_trace_function_call()`: 启用函数调用追踪
- `current_formatter_type()`: 检查当前格式化器类型
- `suppress_logging()`: 临时抑制日志输出

## 约束和规范

1. **统一入口**: 所有日志必须通过 `init_logger()` 获取 logger 实例
2. **避免重复**: 优先使用 `_once` 系列方法减少日志噪音
3. **作用域意识**: 在分布式环境中正确使用 scope 参数
4. **性能考虑**: 追踪功能会显著影响性能，仅用于调试
5. **向后兼容**: 保持对旧版 `vllm.logging.NewLineFormatter` 的兼容性