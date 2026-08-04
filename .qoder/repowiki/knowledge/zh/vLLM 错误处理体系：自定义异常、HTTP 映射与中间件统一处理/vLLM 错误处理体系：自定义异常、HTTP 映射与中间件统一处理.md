---
kind: error_handling
name: vLLM 错误处理体系：自定义异常、HTTP 映射与中间件统一处理
category: error_handling
scope:
    - '**'
source_files:
    - vllm/exceptions.py
    - vllm/v1/engine/exceptions.py
    - vllm/entrypoints/serve/utils/server_utils.py
    - vllm/entrypoints/serve/utils/error_response.py
    - vllm/entrypoints/openai/api_server.py
    - csrc/quickreduce/quick_reduce.h
---

## 1. 系统/方法概述
vLLM 采用「Python 自定义异常 + FastAPI 全局异常处理器 + Rust/C++ 原生层抛出 std::runtime_error」的分层错误处理架构。Python 层通过 `vllm.exceptions` 定义领域语义化的异常类，由 `entrypoints/serve/utils/server_utils.py` 中的 FastAPI exception handlers 统一捕获并转换为 OpenAI 兼容的 `ErrorResponse`；C++/CUDA 扩展在底层通过 `throw std::runtime_error(...)` 上报错误，由 PyTorch 绑定层向上抛出。

## 2. 核心文件与包
- **Python 自定义异常**：`vllm/exceptions.py`
  - `VLLMValidationError(ValueError)`：请求参数校验失败，携带 `parameter` / `value` 上下文
  - `VLLMNotFoundError(Exception)`：资源未找到（如 LoRA adapter）
  - `LoRAAdapterNotFoundError(VLLMNotFoundError)`：LoRA adapter 缺失的专用异常
  - `VLLMUnprocessableEntityError(ValueError)`：内容无法处理（如图片 URL 404/DNS 失败）
- **引擎级异常**：`vllm/v1/engine/exceptions.py`
  - `EngineGenerateError(Exception)`：单次 generate() 可恢复错误
  - `EngineDeadError(Exception)`：EngineCore 崩溃不可恢复，支持 `suppress_context` 抑制 ZMQ 堆栈
- **FastAPI 异常处理器**：`vllm/entrypoints/serve/utils/server_utils.py`
  - `engine_error_handler`：捕获 EngineDeadError / EngineGenerateError，调用 `terminate_if_errored` 触发服务终止检查
  - `generation_error_handler`：对已知 GenerationError 静默返回 500，不打印堆栈
  - `exception_handler`：通用 Exception → ErrorResponse 转换
  - `http_exception_handler`：HTTPException → OpenAI ErrorInfo
  - `validation_exception_handler`：RequestValidationError → 400，提取 Pydantic `loc` 并清理内部标记，优先取 `ctx.error` 中的 VLLMValidationError.parameter
- **错误响应构造器**：`vllm/entrypoints/serve/utils/error_response.py`
  - `create_error_response(message|Exception, err_type, status_code, param)`：根据异常类型映射到 HTTP 状态码与 OpenAI error.type
  - `sanitize_message`：剥离文件路径、traceback、内存地址等敏感信息
- **Rust/C++ 原生层**：`csrc/core/exception.hpp`（仅宏定义）、`csrc/quickreduce/quick_reduce.h` 中 `throw std::runtime_error(...)` 作为底层错误上报

## 3. 架构与约定
- **异常分类与语义**：
  - 用户输入/校验错误 → `VLLMValidationError` / `ValueError` / `TypeError` / `OverflowError` → 400 BadRequestError
  - 资源不存在 → `VLLMNotFoundError` / `LoRAAdapterNotFoundError` → 404 NotFoundError
  - 内容不可处理 → `VLLMUnprocessableEntityError` → 422 UnprocessableEntityError
  - 生成过程错误 → `GenerationError`（含 KV cache 加载失败等）→ 按 `exc.status_code` 返回
  - 引擎崩溃 → `EngineDeadError`（不可恢复）→ 记录日志后触发进程终止检查
  - 其他未知异常 → 500 InternalServerError
- **OpenAI 兼容响应格式**：所有错误最终封装为 `ErrorResponse(error=ErrorInfo{message, type, code, param})`，通过 JSONResponse 返回
- **流式响应特殊处理**：StreamingResponse 中抛出的异常不会自动触发 FastAPI 异常处理器，而是以错误 chunk 形式发送，依赖 watchdog 后台任务检测 errored 状态
- **Pydantic 验证错误增强**：`validation_exception_handler` 从 `error["ctx"]["error"]` 中提取 `VLLMValidationError` 的 `parameter`，并对 `loc` tuple 调用 `clean_loc_for_param` 过滤 pydantic-core 内部标记（如 `function-wrap[...]`、`union` 等），输出干净的 dotted path
- **日志策略**：可通过 `req.app.state.args.log_error_stack` 控制是否记录完整堆栈；`generation_error_handler` 刻意不记录堆栈以避免污染日志

## 4. 约定与约束
- **必须使用 vllm.exceptions 中的自定义异常**：测试用例（如 `tests/entrypoints/openai/chat_completion/test_serving_chat.py`、`tests/multimodal/media/test_unprocessable_entity_error.py`）显式断言这些异常类型被正确抛出和捕获
- **FastAPI 全局异常处理器注册位置**：`api_server.py` 通过 `app.exception_handler(...)` 注册上述 handler，确保所有路由统一错误处理
- **错误消息脱敏**：所有对外错误消息必须经过 `sanitize_message`，测试覆盖路径泄露、traceback 剥离、内存地址清理等场景
- **C++/CUDA 层禁止吞掉异常**：底层通过 `throw std::runtime_error(...)` 明确报错，不允许静默失败
- **EngineDeadError 不可恢复**：注释明确标注 "Unrecoverable"，处理器会调用 `terminate_if_errored` 触发优雅退出
- **EngineGenerateError 可恢复**：注释标注 "Recoverable"，允许单请求失败不影响整体服务