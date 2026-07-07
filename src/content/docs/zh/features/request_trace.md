---
title: "Request Trace"
sidebar:
  order: 91
---

## 概述

Request Trace 功能可以将 xLLM 服务处理的每个请求的输入和输出完整记录到文件，用于问题排查、请求回放和性能分析。

支持记录的内容：
- 原始 HTTP 请求体（可直接用于 replay）
- 内部处理后的 prompt 和 token IDs
- 采样参数（temperature、top_p 等）
- 生成的文本和 token 序列
- 请求元数据（request_id、时间戳等）

## 启动参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--enable_request_trace` | bool | `false` | 是否开启请求 trace |
| `--request_trace_path` | string | `./request_trace.jsonl` | 输出路径（JSONL 模式为文件路径，per-file 模式为目录路径） |
| `--request_trace_per_file` | bool | `false` | 是否每个请求单独写一个文件 |

## 快速开始

### JSONL 模式（默认）

所有请求追加写入同一个文件，每行一条 JSON 记录：

```bash
./xllm_server --port 8010 \
    --enable_request_trace=true \
    --request_trace_path=/data/traces/requests.jsonl
```

### Per-file 模式

每个请求写入独立文件 `<dir>/<request_id>.json`，格式化输出便于阅读：

```bash
./xllm_server --port 8010 \
    --enable_request_trace=true \
    --request_trace_per_file=true \
    --request_trace_path=/data/traces/requests/
```

## 运行时动态开关

`enable_request_trace` 支持运行时修改，无需重启服务。

通过 brpc 内置的 `/flags` 接口动态开启/关闭：

```bash
# 开启 trace
curl "http://localhost:8010/flags/enable_request_trace?setvalue=true"

# 关闭 trace
curl "http://localhost:8010/flags/enable_request_trace?setvalue=false"

# 查看当前值
curl "http://localhost:8010/flags/enable_request_trace"
```

注意事项：
- `request_trace_path` 和 `request_trace_per_file` 仅在首次开启 trace 时生效，后续修改需要重启服务。
- 动态开启后，仅新进入的请求会被完整记录（包含 `raw_request`）。开启瞬间正在处理中的请求可能缺少原始 HTTP body。

## 输出格式

每条记录包含以下字段：

```json
{
  "request_id": "cmpl-abc123",
  "x_request_id": "user-supplied-id",
  "x_request_time": "2026-07-07T10:00:00.000+0800",
  "service_request_id": "",
  "timestamp": "2026-07-07T10:00:01.234+0800",
  "stream": false,

  "raw_request": {
    "path": "/v1/chat/completions",
    "body": {
      "model": "qwen3-32b",
      "messages": [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Hello!"}
      ],
      "temperature": 0.7,
      "max_tokens": 1024,
      "stream": false
    }
  },

  "input": {
    "prompt": "<|im_start|>system\nYou are a helpful assistant.<|im_end|>\n<|im_start|>user\nHello!<|im_end|>\n<|im_start|>assistant\n",
    "prompt_token_ids": [151644, 8948, ...],
    "prompt_tokens_num": 25,
    "max_tokens": 1024,
    "n": 1,
    "best_of": 1,
    "temperature": 0.7,
    "top_p": 1.0,
    "top_k": -1,
    "frequency_penalty": 0.0,
    "presence_penalty": 0.0,
    "repetition_penalty": 1.0
  },

  "output": {
    "sequences": [
      {
        "index": 0,
        "text": "Hello! How can I help you today?",
        "token_ids": [9707, 0, 2585, 649, 358, ...],
        "finish_reason": "stop"
      }
    ],
    "usage": {
      "prompt_tokens": 25,
      "generated_tokens": 9,
      "total_tokens": 34
    },
    "finished": true,
    "cancelled": false
  }
}
```

### 字段说明

| 字段 | 说明 |
|------|------|
| `raw_request.path` | 原始 HTTP 请求路径 |
| `raw_request.body` | 原始 HTTP 请求体（可直接用于 replay） |
| `input.prompt` | chat template 渲染后的完整 prompt 字符串 |
| `input.prompt_token_ids` | tokenize 后的 token ID 序列 |
| `output.sequences[].text` | 生成的文本（streaming 请求为累积后的完整文本） |
| `output.sequences[].token_ids` | 原始生成 token 序列（tool call 解析前） |
| `output.finished` | 请求是否正常完成 |
| `output.cancelled` | 请求是否被取消 |

## Streaming 请求处理

对于 streaming 请求，trace 会累积所有 delta 输出，在请求完成或取消后写出完整记录。输出中的 `text` 和 `token_ids` 是所有 chunk 拼接后的完整结果。

## 性能影响

### Trace 关闭时

零开销。`enabled()` 检查直接读 gflags 值后 early return。

### Trace 开启时

| 环节 | 开销 | 说明 |
|------|------|------|
| 原始 body 拷贝 | 一次 string copy | 在 HTTP handler 入口，通常 < 1KB（纯文本请求） |
| JSON 序列化 | 数百微秒 ~ 数毫秒 | 取决于 prompt 长度和输出长度 |
| 异步入队 | < 1 微秒 | mutex lock + deque push + notify |
| 文件 I/O | 不阻塞请求 | 后台线程批量写入 + flush |

关键设计：
- 所有文件 I/O 在独立后台线程执行，不阻塞 response 回调
- 后台线程按 batch 处理队列中积压的记录，减少 flush 次数
- `stream_mutex_` 和 `queue_mutex_` 分离，streaming 累积不与写队列竞争

## 用于请求回放

dump 文件中的 `raw_request` 字段包含完整的原始 HTTP 请求体和路径，可以直接提取后发送给服务进行回放：

```bash
# 从 JSONL 文件提取并回放单条请求
jq -r 'select(.request_id == "cmpl-abc123") | .raw_request.body' \
    /data/traces/requests.jsonl | \
    curl -X POST http://localhost:8010/v1/chat/completions \
         -H "Content-Type: application/json" \
         -d @-
```

```python
import json
import requests

with open("/data/traces/requests.jsonl") as f:
    for line in f:
        record = json.loads(line)
        raw = record.get("raw_request")
        if raw:
            resp = requests.post(
                f"http://localhost:8010{raw['path']}",
                json=raw["body"],
            )
            print(resp.status_code, record["request_id"])
```

## 磁盘空间估算

每条记录大小取决于 prompt 和输出长度。粗略估算：

| 请求类型 | 单条记录大小 |
|----------|-------------|
| 短对话（< 100 tokens） | 2-5 KB |
| 中等对话（1K tokens） | 20-50 KB |
| 长上下文（32K tokens） | 500 KB - 1 MB |

建议配合日志轮转或定期清理使用。

## 故障排查

| 问题 | 排查方法 |
|------|----------|
| 文件未生成 | 检查 `--request_trace_path` 目录是否存在且有写权限 |
| 记录缺少 `raw_request` | 确认 trace 在请求进入前已开启（动态开启有过渡期） |
| 磁盘写入慢 | trace 使用异步写入，不影响请求延迟；检查磁盘 IOPS |
| 进程异常退出丢数据 | 后台线程会尽量 drain 队列，但 SIGKILL 可能丢失未 flush 数据 |
