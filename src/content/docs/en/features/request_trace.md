---
title: "Request Trace"
sidebar:
  order: 91
---

## Overview

The Request Trace feature records the complete input and output of each request processed by the xLLM service to a file, for troubleshooting, request replay, and performance analysis.

Supported recorded content:
- Original HTTP request body (can be used directly for replay)
- Internally processed prompt and token IDs
- Sampling parameters (temperature, top_p, etc.)
- Generated text and token sequences
- Request metadata (request_id, timestamp, etc.)

## Startup Parameters

| Parameter | Type | Default | Description |
|------|------|--------|------|
| `--enable_request_trace` | bool | `false` | Whether to enable request trace |
| `--request_trace_path` | string | `./request_trace.jsonl` | Output path (file path for JSONL mode, directory path for per-file mode) |
| `--request_trace_per_file` | bool | `false` | Whether to write each request to a separate file |

## Quick Start

### JSONL Mode (Default)

All requests are appended to the same file, one JSON record per line:

```bash
./xllm_server --port 8010 \
    --enable_request_trace=true \
    --request_trace_path=/data/traces/requests.jsonl
```

### Per-file Mode

Each request is written to a separate file `<dir>/<request_id>.json`, pretty-printed for readability:

```bash
./xllm_server --port 8010 \
    --enable_request_trace=true \
    --request_trace_per_file=true \
    --request_trace_path=/data/traces/requests/
```

## Runtime Dynamic Toggle

`enable_request_trace` can be modified at runtime without restarting the service.

Toggle it on/off dynamically via the built-in brpc `/flags` interface:

```bash
# Enable trace
curl "http://localhost:8010/flags/enable_request_trace?setvalue=true"

# Disable trace
curl "http://localhost:8010/flags/enable_request_trace?setvalue=false"

# View current value
curl "http://localhost:8010/flags/enable_request_trace"
```

Notes:
- `request_trace_path` and `request_trace_per_file` only take effect the first time trace is enabled; subsequent changes require restarting the service.
- After a dynamic enable, only newly arrived requests are fully recorded (including `raw_request`). Requests being processed at the moment of enabling may lack the original HTTP body.

## Output Format

Each record contains the following fields:

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

### Field Description

| Field | Description |
|------|------|
| `raw_request.path` | Original HTTP request path |
| `raw_request.body` | Original HTTP request body (can be used directly for replay) |
| `input.prompt` | Full prompt string after chat template rendering |
| `input.prompt_token_ids` | Tokenized token ID sequence |
| `output.sequences[].text` | Generated text (for streaming requests, the accumulated full text) |
| `output.sequences[].token_ids` | Original generated token sequence (before tool call parsing) |
| `output.finished` | Whether the request completed normally |
| `output.cancelled` | Whether the request was cancelled |

## Streaming Request Handling

For streaming requests, trace accumulates all delta outputs and writes the complete record after the request finishes or is cancelled. The `text` and `token_ids` in the output are the full results after concatenating all chunks.

## Performance Impact

### When Trace is Disabled

Zero overhead. The `enabled()` check reads the gflags value and returns early.

### When Trace is Enabled

| Stage | Overhead | Description |
|------|------|------|
| Raw body copy | One string copy | At the HTTP handler entry, typically < 1KB (plain text request) |
| JSON serialization | Hundreds of microseconds ~ several milliseconds | Depends on prompt length and output length |
| Async enqueue | < 1 microsecond | mutex lock + deque push + notify |
| File I/O | Does not block requests | Background thread batch writing + flush |

Key design:
- All file I/O runs on a dedicated background thread, not blocking response callbacks.
- The background thread processes queued records in batches to reduce flush count.
- `stream_mutex_` and `queue_mutex_` are separated; streaming accumulation does not compete with the write queue.

## Request Replay

The `raw_request` field in the dump file contains the complete original HTTP request body and path, which can be extracted and sent directly to the service for replay:

```bash
# Extract and replay a single request from a JSONL file
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

## Disk Space Estimation

Each record size depends on the prompt and output length. Rough estimate:

| Request Type | Single Record Size |
|----------|-------------|
| Short conversation (< 100 tokens) | 2-5 KB |
| Medium conversation (1K tokens) | 20-50 KB |
| Long context (32K tokens) | 500 KB - 1 MB |

It is recommended to use this with log rotation or periodic cleanup.

## Troubleshooting

| Problem | Troubleshooting Method |
|------|----------|
| File not generated | Check that the `--request_trace_path` directory exists and is writable |
| Record missing `raw_request` | Confirm trace was enabled before the request arrived (dynamic enable has a transition period) |
| Slow disk writes | Trace uses async writes and does not affect request latency; check disk IOPS |
| Process abnormal exit loses data | The background thread tries to drain the queue, but SIGKILL may lose unflushed data |
