# OpenAI 标准协议、主流模型 API 与兼容性分析

日期：2026-06-23

## 1. 先给结论

所谓 `OpenAI-compatible API`，通常不是指完整复刻 OpenAI 平台上的所有接口，而是指“请求和响应形状尽量兼容 OpenAI 的部分接口”。现实里最常见、也最容易做到的是兼容 `POST /v1/chat/completions`，其次是兼容 embeddings、简单流式输出、基础 tool calling。  

如果目标是“支持 OpenAI 所有协议”，这件事不能只靠字段映射完成。OpenAI 的接口已经不只是单轮 text chat，还包含 Responses API、Chat Completions、Embeddings、Images、Audio、Realtime、Files、Vector Stores、Batch、Moderation、Fine-tuning、内置工具、结构化输出、状态管理、server-side tool、异步任务、WebSocket 等能力。其他供应商即使提供类似模型能力，也经常在消息结构、工具调用循环、流式事件、结构化输出、文件/向量资源、实时音频、推理参数、安全策略、错误码和鉴权方式上不同。

因此比较现实的设计是分层兼容：

| 层级 | 能力 | 可兼容程度 |
| --- | --- | --- |
| L0 | text-only chat | 最高，很多供应商都能做 |
| L1 | text streaming | 较高，但 chunk/event 细节不同 |
| L2 | tool/function calling | 中等，调用循环和 tool result 表达差异很大 |
| L3 | JSON mode / structured output | 中等偏低，JSON Schema 支持程度不同 |
| L4 | embeddings | 较高，但向量维度、模型语义和批量限制不同 |
| L5 | multimodal input/output | 中等，不同供应商支持的 image/audio/video 范围不同 |
| L6 | Files / vector store / batch / realtime / state | 低，通常需要网关自己实现后端服务 |
| L7 | provider-native reasoning、safety、cache、routing | 无法无损统一，只能透传或单独适配 |

一句话：text-only agent 场景可以优先按 OpenAI Chat Completions 或 Responses 的子集做；但如果希望支持 OpenAI 全协议，必须把它看成一个“能力网关 + 资源服务 + 协议转换层”，而不是一个简单 proxy。

## 2. OpenAI 标准协议中的主要接口

这里的“OpenAI 标准协议”可以分成两类：

1. 模型推理接口：Responses、Chat Completions、Embeddings、Images、Audio、Realtime。
2. 平台资源接口：Files、Vector Stores、Batch、Fine-tuning、Moderations、Models、Webhooks 等。

很多开源网关只实现第一类中的一小部分，尤其是 Chat Completions。

### 2.1 Responses API

Responses API 是 OpenAI 现在推荐的新一代主推接口。它可以理解为 Chat Completions 的演进版，但抽象层更高，覆盖 text、image input、tool calling、built-in tools、状态延续、结构化输出、streaming 等能力。

典型 endpoint：

```http
POST https://api.openai.com/v1/responses
Authorization: Bearer $OPENAI_API_KEY
Content-Type: application/json
```

典型请求：

```json
{
  "model": "gpt-4.1",
  "instructions": "You are a concise assistant.",
  "input": "Summarize the difference between TCP and UDP.",
  "max_output_tokens": 500,
  "stream": false
}
```

它的核心字段包括：

| 字段 | 含义 |
| --- | --- |
| `model` | 使用的模型 |
| `input` | 用户输入，可以是字符串，也可以是结构化 input item |
| `instructions` | 类似全局行为指令，可理解为系统/开发者侧指令 |
| `tools` | 可用工具，包括函数工具和内置工具 |
| `tool_choice` | 控制是否调用工具、调用哪个工具 |
| `text` | 控制文本输出格式，例如结构化输出配置 |
| `previous_response_id` | 让新请求接续之前的 response 状态 |
| `store` | 是否让服务端保存 response |
| `stream` | 是否使用流式输出 |
| `max_output_tokens` | 输出 token 上限 |
| `parallel_tool_calls` | 是否允许并行工具调用 |
| `include` | 额外返回内容，比如 file search 结果、logprobs、reasoning 加密内容等 |

Responses API 和旧 Chat Completions 的关键区别：

| 维度 | Chat Completions | Responses |
| --- | --- | --- |
| 输入抽象 | `messages[]` | `input` / input items |
| 输出抽象 | `choices[].message` | response output items |
| 多模态 | 支持部分内容块 | 作为统一 input/output item 设计 |
| 状态延续 | 通常由客户端拼接历史 | 可通过 `previous_response_id` 接续 |
| 内置工具 | 不完整 | 原生支持 web search、file search、computer use 等工具形态 |
| 新功能承载 | 逐步边缘化 | 新能力优先落在 Responses |

如果你要做一个面向未来的 OpenAI 协议 server，Responses API 是必须重点支持的。但如果你的用户主要用现有 OpenAI SDK 或大量兼容生态，Chat Completions 仍然是最现实的入口。

### 2.2 Chat Completions API

Chat Completions 是目前生态里最广泛被“OpenAI-compatible”复刻的接口。大量模型供应商、代理网关、AI IDE、agent 框架都支持这个形状。

典型 endpoint：

```http
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer $OPENAI_API_KEY
Content-Type: application/json
```

典型请求：

```json
{
  "model": "gpt-4.1-mini",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant."
    },
    {
      "role": "user",
      "content": "Write a short SQL query to count users by country."
    }
  ],
  "temperature": 0.2,
  "max_tokens": 300
}
```

核心字段：

| 字段 | 含义 |
| --- | --- |
| `model` | 模型名称 |
| `messages` | 对话历史 |
| `messages[].role` | `system`、`developer`、`user`、`assistant`、`tool` 等 |
| `messages[].content` | 文本或多模态 content parts |
| `tools` | 函数工具定义 |
| `tool_choice` | 控制工具调用策略 |
| `parallel_tool_calls` | 是否允许并行工具调用 |
| `response_format` | JSON mode 或 JSON Schema 结构化输出 |
| `stream` | 是否流式返回 |
| `temperature` / `top_p` | 采样参数 |
| `max_tokens` | 输出长度限制，部分新模型使用 `max_completion_tokens` |

典型响应：

```json
{
  "id": "chatcmpl_xxx",
  "object": "chat.completion",
  "created": 1710000000,
  "model": "gpt-4.1-mini",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "SELECT country, COUNT(*) AS user_count FROM users GROUP BY country;"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 32,
    "completion_tokens": 18,
    "total_tokens": 50
  }
}
```

tool calling 的典型形态：

```json
{
  "model": "gpt-4.1-mini",
  "messages": [
    {
      "role": "user",
      "content": "What's the weather in Beijing?"
    }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get current weather for a city.",
        "parameters": {
          "type": "object",
          "properties": {
            "city": {
              "type": "string"
            }
          },
          "required": ["city"]
        }
      }
    }
  ]
}
```

模型如果选择调用工具，会返回类似：

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {
      "id": "call_xxx",
      "type": "function",
      "function": {
        "name": "get_weather",
        "arguments": "{\"city\":\"Beijing\"}"
      }
    }
  ]
}
```

客户端执行工具后，再把工具结果作为 `tool` message 传回模型：

```json
{
  "role": "tool",
  "tool_call_id": "call_xxx",
  "content": "{\"temperature\":28,\"condition\":\"cloudy\"}"
}
```

Chat Completions 的兼容性最好，但也有几个坑：

- `developer` role 并不是所有兼容供应商都认识。
- `response_format` 的 JSON Schema 严格模式并不是所有模型都支持。
- `tool_calls[].function.arguments` 是 JSON 字符串，不是 JSON 对象，很多代理会处理错。
- 流式返回的 `delta` chunk 很多供应商只能近似兼容。
- 多模态 content parts 虽然形状可以兼容，但模型实际是否支持 image/audio input 取决于供应商。

### 2.3 Embeddings API

Embeddings API 用于把文本转成向量，主要用于检索、聚类、相似度计算、RAG 等场景。

典型 endpoint：

```http
POST https://api.openai.com/v1/embeddings
```

请求：

```json
{
  "model": "text-embedding-3-large",
  "input": [
    "OpenAI-compatible API",
    "vector search"
  ]
}
```

响应：

```json
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "index": 0,
      "embedding": [0.0123, -0.0456]
    }
  ],
  "model": "text-embedding-3-large",
  "usage": {
    "prompt_tokens": 8,
    "total_tokens": 8
  }
}
```

兼容难点主要不是字段，而是语义：

- 不同 embedding 模型维度不同，例如 768、1024、1536、3072 等。
- 不同模型的向量空间不可直接混用，不能把 A 模型写入向量库后用 B 模型查询。
- 归一化、距离度量、语言覆盖、多语种效果都不同。
- 批量大小、输入 token 上限和价格策略不同。

### 2.4 Images API

Images API 覆盖图像生成、图像编辑、图像变体等。

典型 endpoint：

```http
POST https://api.openai.com/v1/images/generations
POST https://api.openai.com/v1/images/edits
POST https://api.openai.com/v1/images/variations
```

生成请求：

```json
{
  "model": "gpt-image-1",
  "prompt": "A clean product photo of a transparent mechanical keyboard on a white desk.",
  "size": "1024x1024"
}
```

响应通常返回图片 URL 或 base64：

```json
{
  "created": 1710000000,
  "data": [
    {
      "b64_json": "..."
    }
  ]
}
```

兼容难点：

- 很多 text LLM 供应商没有原生图像生成接口。
- 有图像生成能力的供应商也会有自己的参数，例如风格、比例、种子、负面提示词、参考图权重。
- 安全策略差异很大，同一 prompt 在不同供应商可能被拒绝、改写或返回不同内容。
- URL 返回、base64 返回、异步任务返回三种模式并存。

### 2.5 Audio API

Audio API 覆盖语音转文本、文本转语音、翻译、说话人分离等能力。

典型 endpoint：

```http
POST https://api.openai.com/v1/audio/transcriptions
POST https://api.openai.com/v1/audio/translations
POST https://api.openai.com/v1/audio/speech
```

语音转文本请求通常是 multipart form：

```http
POST /v1/audio/transcriptions
Content-Type: multipart/form-data

file=@meeting.mp3
model=gpt-4o-transcribe
response_format=json
```

响应：

```json
{
  "text": "The meeting started at 9 AM..."
}
```

兼容难点：

- 音频 API 往往不是 JSON body，而是 multipart 上传。
- 不同供应商支持的音频格式、采样率、文件大小、时长限制不同。
- TTS 音色、语速、音频格式、流式播放协议差异很大。
- 说话人分离、时间戳、词级时间戳不是所有供应商都有。

### 2.6 Realtime API

Realtime API 用于低延迟交互，典型场景是实时语音助手、电话机器人、实时多模态交互。它通常通过 WebSocket 建立 session，也可以包含临时 client secret、音频输入输出配置、工具调用和 event 流。

典型能力：

| 能力 | 说明 |
| --- | --- |
| WebSocket session | 长连接实时交互 |
| input audio | 客户端持续发送音频 |
| output audio | 模型实时返回语音 |
| output text | 同时返回文本转写或回答 |
| tool calling | 会话中触发工具 |
| turn detection | 服务端或客户端检测说话轮次 |
| ephemeral key | 给前端使用的短期密钥 |

兼容难点非常大：

- 不是所有供应商都提供 WebSocket 实时模型接口。
- 音频格式、编码、采样率、分片事件完全不同。
- turn detection、barge-in、interrupt、voice activity detection 是实时系统能力，不只是模型能力。
- 有的供应商只能做普通 streaming text，不能替代 Realtime。

### 2.7 Files API

Files API 用于上传文件，供 fine-tuning、assistant/file search、batch 等功能使用。

典型 endpoint：

```http
POST /v1/files
GET /v1/files
GET /v1/files/{file_id}
DELETE /v1/files/{file_id}
GET /v1/files/{file_id}/content
```

请求通常包含：

| 字段 | 含义 |
| --- | --- |
| `file` | 上传的文件 |
| `purpose` | 文件用途，例如 fine-tune、assistants、batch 等 |

Files API 的关键点是它不是一次模型推理，而是平台资源管理。要兼容它，网关必须有自己的对象存储、文件生命周期、权限控制、用途校验和内容读取接口。

### 2.8 Vector Stores 与 File Search

Vector Stores 是 OpenAI 平台上的文件检索资源，用来支持 file search 等能力。

典型 endpoint：

```http
POST /v1/vector_stores
GET /v1/vector_stores
GET /v1/vector_stores/{vector_store_id}
POST /v1/vector_stores/{vector_store_id}/search
POST /v1/vector_stores/{vector_store_id}/files
POST /v1/vector_stores/{vector_store_id}/file_batches
```

它背后包含几个服务能力：

- 文件解析。
- 切块。
- embedding。
- 索引构建。
- 检索排序。
- 引用来源返回。
- 文件状态管理。
- 过期策略。

因此它不是把请求转发给其他 LLM 就能完成。除非目标供应商也有等价的托管向量检索服务，否则兼容 server 需要自己实现 RAG 后端。

### 2.9 Batch API

Batch API 用于异步批量执行大量请求，适合离线任务、成本优化和吞吐优化。

典型 endpoint：

```http
POST /v1/batches
GET /v1/batches/{batch_id}
POST /v1/batches/{batch_id}/cancel
GET /v1/batches
```

它通常需要：

- 上传 JSONL 输入文件。
- 创建 batch 任务。
- 异步调度。
- 状态查询。
- 输出文件下载。
- 失败行记录。

兼容难点：

- 很多供应商没有 OpenAI Batch 等价物。
- 即使有异步任务，输入格式、任务生命周期和输出文件格式也不同。
- 网关如果要统一 Batch，就必须自己做队列、重试、并发控制、结果汇总和文件管理。

### 2.10 Moderations API

Moderations API 用于对文本或图像进行安全分类，例如暴力、自残、仇恨、色情等风险类别。

典型 endpoint：

```http
POST /v1/moderations
```

请求：

```json
{
  "model": "omni-moderation-latest",
  "input": "User provided text or image input"
}
```

响应会包含分类结果、是否 flagged、类别分数等。

兼容难点：

- 不同供应商的安全分类标签体系不同。
- “是否违规”的阈值不同。
- 有的供应商只在生成接口里做内容拦截，不提供独立 moderation API。
- 企业内部通常还会叠加自己的安全策略，无法完全用一个标准字段表达。

### 2.11 Fine-tuning、Models、Webhooks、Admin

OpenAI 平台还包括 fine-tuning、model listing、webhooks、organization/admin、usage/cost 等接口。这些接口更偏平台管理，不是模型调用协议。

兼容它们需要供应商或网关具备：

- 训练数据上传。
- 训练任务调度。
- 模型版本管理。
- 事件回调。
- 权限和组织管理。
- 用量与账单统计。

多数 OpenAI-compatible server 不会完整实现这一层。

## 3. 其他主流模型接口调用方式

下面按主流供应商说明它们和 OpenAI 协议的差异。

### 3.1 Anthropic Claude Messages API

Claude 的核心接口是 Messages API。它不是 OpenAI Chat Completions 原生形状，但可以映射到相似的聊天模型。

典型 endpoint：

```http
POST https://api.anthropic.com/v1/messages
x-api-key: $ANTHROPIC_API_KEY
anthropic-version: 2023-06-01
Content-Type: application/json
```

典型请求：

```json
{
  "model": "claude-3-5-sonnet-latest",
  "max_tokens": 1024,
  "system": "You are a careful coding assistant.",
  "messages": [
    {
      "role": "user",
      "content": "Explain what this Python function does."
    }
  ]
}
```

典型响应：

```json
{
  "id": "msg_xxx",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "This function..."
    }
  ],
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 20,
    "output_tokens": 50
  }
}
```

Claude tool use 的形态和 OpenAI 不同：

- Claude 返回 `content[]` block，其中可能包含 `tool_use` block。
- 客户端执行工具后，把结果作为 `tool_result` block 发回。
- Anthropic 也有 server tool 和 client tool 的区分。
- strict tool use 使用自己的配置方式。

映射到 OpenAI 时的主要问题：

| Anthropic | OpenAI |
| --- | --- |
| top-level `system` | `messages[].role=system` 或 Responses `instructions` |
| `content[]` blocks | `message.content` 或 output items |
| `tool_use` block | `tool_calls[]` |
| `tool_result` block | `role=tool` message |
| `stop_reason=end_turn` | `finish_reason=stop` |
| `max_tokens` | `max_tokens` / `max_output_tokens` |

这种映射可以跑通 text-only 和常见 tool calling，但很难无损保留 Claude 特有的 extended thinking、server tools、content block 细节和 stop reason。

### 3.2 Google Gemini GenerateContent API

Gemini 的主接口是 `generateContent` 和 `streamGenerateContent`，请求结构围绕 `contents[]` 和 `parts[]` 设计。

典型 endpoint：

```http
POST https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-pro:generateContent
x-goog-api-key: $GEMINI_API_KEY
Content-Type: application/json
```

典型请求：

```json
{
  "contents": [
    {
      "role": "user",
      "parts": [
        {
          "text": "Summarize this paragraph."
        }
      ]
    }
  ],
  "systemInstruction": {
    "parts": [
      {
        "text": "Be concise."
      }
    ]
  },
  "generationConfig": {
    "temperature": 0.2,
    "maxOutputTokens": 512
  }
}
```

Gemini 的重要字段：

| 字段 | 含义 |
| --- | --- |
| `contents[]` | 对话内容 |
| `contents[].parts[]` | 文本、图片、函数调用、函数响应等 part |
| `systemInstruction` | 系统指令 |
| `tools[]` | 工具定义，例如 function declarations、code execution |
| `toolConfig` | 工具调用配置 |
| `safetySettings[]` | 安全策略 |
| `generationConfig` | 温度、top-p、输出长度、response schema 等 |
| `cachedContent` | 缓存内容引用 |

和 OpenAI 的主要差异：

- OpenAI 是 `messages[]` 或 `input`，Gemini 是 `contents[]` + `parts[]`。
- Gemini 的 function calling 嵌在 part 里，不是 OpenAI 的 `tool_calls[]`。
- Gemini 有 `safetySettings`，OpenAI 没有完全等价字段。
- Gemini 有自己的 response schema、thinking、cache 等配置。
- 多模态输入在 Gemini 里是更原生的 part 体系，映射到 OpenAI content parts 时需要谨慎。

### 3.3 AWS Bedrock Converse API

AWS Bedrock 提供 Converse API，目标是给不同模型提供统一对话接口。它本身也是一种“供应商内部统一协议”。

典型 endpoint：

```http
POST https://bedrock-runtime.$REGION.amazonaws.com/model/$MODEL_ID/converse
Authorization: AWS Signature Version 4
Content-Type: application/json
```

典型请求：

```json
{
  "modelId": "anthropic.claude-3-5-sonnet-20240620-v1:0",
  "system": [
    {
      "text": "You are a helpful assistant."
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "text": "Explain S3 in one paragraph."
        }
      ]
    }
  ],
  "inferenceConfig": {
    "temperature": 0.2,
    "maxTokens": 512
  }
}
```

Bedrock 的特点：

- `modelId` 可以是基础模型 ID、inference profile、provisioned throughput ARN 等。
- `system` 是数组。
- `messages[].content[]` 是 content block。
- 工具定义放在 `toolConfig`。
- 特定模型的额外字段放在 `additionalModelRequestFields`。
- 返回中包含 `stopReason`，可能是 `tool_use`、`max_tokens`、`guardrail_intervened`、`content_filtered` 等。
- 鉴权使用 AWS SigV4，不是简单 API key。

兼容 OpenAI 时的关键问题：

- Bedrock 是多模型托管平台，同一个 Converse 协议背后可能是 Claude、Llama、Mistral、Titan 等模型，细节仍然不一致。
- AWS guardrail、trace、performance config、prompt management 等字段没有 OpenAI 等价物。
- SigV4 鉴权和 OpenAI Bearer key 模式完全不同。
- ConverseStream 使用 AWS event stream，不是普通 SSE。

### 3.4 Mistral Chat API

Mistral 提供自己的 Chat API，同时也比较接近 OpenAI Chat Completions。

典型 endpoint：

```http
POST https://api.mistral.ai/v1/chat/completions
Authorization: Bearer $MISTRAL_API_KEY
Content-Type: application/json
```

典型请求：

```json
{
  "model": "mistral-large-latest",
  "messages": [
    {
      "role": "user",
      "content": "Give me a short project risk checklist."
    }
  ],
  "temperature": 0.2,
  "max_tokens": 512
}
```

Mistral 还提供 FIM、Embeddings、Classifiers、Files、Batch、OCR、Audio 等接口。它与 OpenAI Chat Completions 的距离比较近，所以 text chat 兼容较容易。但要完整兼容 OpenAI Responses、Vector Stores、Realtime、Images、特定 built-in tools，仍然不是简单映射。

### 3.5 Cohere Chat API

Cohere v2 Chat API 是自己的 `/v2/chat` 形状。

典型 endpoint：

```http
POST https://api.cohere.com/v2/chat
Authorization: Bearer $COHERE_API_KEY
Content-Type: application/json
```

典型请求：

```json
{
  "model": "command-a-03-2025",
  "messages": [
    {
      "role": "user",
      "content": "Draft a concise customer support reply."
    }
  ],
  "stream": false
}
```

Cohere 的特点：

- `messages[]` 支持 user、assistant、tool、system 等角色。
- `tools`、`tool_choice`、strict tools 有自己的语义。
- `response_format` 支持 JSON object 或 JSON Schema，但和 tools/documents 组合时有额外限制。
- `safety_mode` 是 Cohere 自己的安全控制。
- 返回的 `message.content[]` 和 OpenAI 的 `choices[].message.content` 不完全一样。

兼容 OpenAI 的问题：

- tool 调用和 tool result 的循环字段不同。
- `finish_reason` 取值不同，例如 tool call、error、max token 等。
- documents、citation、rerank 等能力是 Cohere 自己的强项，不一定有 OpenAI 等价字段。

### 3.6 Azure OpenAI API

Azure OpenAI 是最接近 OpenAI 协议的一类，但它也不是完全相同。

典型 endpoint：

```http
POST https://{resource}.openai.azure.com/openai/deployments/{deployment-id}/chat/completions?api-version=2025-04-01-preview
api-key: $AZURE_OPENAI_API_KEY
Content-Type: application/json
```

主要差异：

| 维度 | OpenAI | Azure OpenAI |
| --- | --- | --- |
| URL | `/v1/chat/completions` | `/openai/deployments/{deployment-id}/chat/completions?api-version=...` |
| 模型选择 | `model` | deployment-id 更关键 |
| 鉴权 | Bearer API key | `api-key` 或 Azure AD |
| 版本 | 通常由平台统一 | 显式 `api-version` |
| 扩展 | OpenAI 原生字段 | Azure 可能有 `data_sources` 等扩展 |

Azure 的兼容性强，但由于 deployment、api-version、企业网络、内容过滤、Azure-specific extensions 存在，仍然需要适配层。

### 3.7 DeepSeek API

DeepSeek 提供 OpenAI-compatible 的 chat 接口，使用方式和 OpenAI SDK 很接近。

典型 endpoint：

```http
POST https://api.deepseek.com/chat/completions
Authorization: Bearer $DEEPSEEK_API_KEY
Content-Type: application/json
```

或者配置 OpenAI SDK：

```python
from openai import OpenAI

client = OpenAI(
    api_key="DEEPSEEK_API_KEY",
    base_url="https://api.deepseek.com"
)

response = client.chat.completions.create(
    model="deepseek-chat",
    messages=[
        {"role": "user", "content": "Explain binary search."}
    ]
)
```

DeepSeek 的特点：

- Chat 接口高度兼容 OpenAI。
- 支持自己的模型名，例如 `deepseek-chat`、`deepseek-reasoner`。
- reasoning/thinking 相关字段可能通过额外参数传入。
- 不是 OpenAI 全平台协议，重点仍是推理接口兼容。

### 3.8 阿里云 Qwen / DashScope OpenAI 兼容模式

DashScope 提供 OpenAI 兼容模式，典型路径是：

```http
POST https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions
Authorization: Bearer $DASHSCOPE_API_KEY
Content-Type: application/json
```

典型请求：

```json
{
  "model": "qwen-plus",
  "messages": [
    {
      "role": "user",
      "content": "用三句话解释 Transformer。"
    }
  ],
  "stream": true
}
```

它可以和 OpenAI SDK 配合使用：

```python
from openai import OpenAI

client = OpenAI(
    api_key="DASHSCOPE_API_KEY",
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
)

completion = client.chat.completions.create(
    model="qwen-plus",
    messages=[
        {"role": "user", "content": "你好"}
    ]
)
```

兼容性特点：

- 对 Chat Completions 生态友好。
- 流式输出使用 OpenAI 风格 chunk 和 `[DONE]`。
- 错误对象也尽量接近 OpenAI。
- 但 Qwen/DashScope 原生能力，例如多模态、视频、插件、应用调用、异步任务、安全策略等，不一定能完全塞进 OpenAI Chat Completions。

## 4. 主流接口横向对比

| 供应商/协议 | 主要聊天接口 | 输入结构 | 工具调用 | 流式 | 结构化输出 | 主要不兼容点 |
| --- | --- | --- | --- | --- | --- | --- |
| OpenAI Responses | `/v1/responses` | `input` / items | `tools` + built-in tools | SSE events | `text` 配置 | 新协议，其他供应商很少完整支持 |
| OpenAI Chat | `/v1/chat/completions` | `messages[]` | `tool_calls[]` | SSE chunks | `response_format` | 生态标准，但不是全平台协议 |
| Anthropic Claude | `/v1/messages` | `messages[]` + top-level `system` + content blocks | `tool_use` / `tool_result` blocks | SSE events | tool/schema 相关能力 | content block、thinking、server tools 差异 |
| Gemini | `:generateContent` | `contents[]` + `parts[]` | function call parts | streamGenerateContent | response schema | safetySettings、parts、cache、thinking 差异 |
| AWS Bedrock | `/model/{id}/converse` | `messages[].content[]` | `toolConfig` | event stream | 依模型而定 | SigV4、guardrails、model-specific fields |
| Mistral | `/v1/chat/completions` | `messages[]` | 支持工具 | 支持 | 依模型而定 | Chat 接近，平台接口不完整等价 |
| Cohere | `/v2/chat` | `messages[]` | tools + tool_choice | 支持 | `response_format` | safety_mode、documents、citations、tool loop |
| Azure OpenAI | deployment path | 接近 OpenAI | 接近 OpenAI | 接近 OpenAI | 接近 OpenAI | deployment、api-version、Azure extensions |
| DeepSeek | OpenAI-like chat | `messages[]` | OpenAI-like | 支持 | 依模型而定 | reasoning extra fields、非全平台 |
| DashScope Qwen | compatible-mode chat | `messages[]` | OpenAI-like | 支持 | 依模型而定 | 原生 DashScope 能力无法完全映射 |

## 5. 为什么无法做到完全兼容

### 5.1 “字段兼容”不等于“能力兼容”

很多兼容 server 只做了这样的事情：

```text
OpenAI request -> provider request -> provider response -> OpenAI-like response
```

这对 text chat 有效，因为 text chat 的最小语义很简单：

```text
输入一段对话 -> 输出一段文本
```

但 OpenAI 平台中的很多接口不是简单模型调用。例如：

- File Search 需要文件解析、切块、embedding、索引和检索。
- Batch 需要异步任务队列和结果文件。
- Realtime 需要 WebSocket、音频流、turn detection、低延迟输出。
- Moderation 需要专门安全分类模型。
- Fine-tuning 需要训练平台。
- Responses 的 `previous_response_id` 需要服务端状态。

如果下游供应商没有这些能力，网关只能自己实现，不能靠转换字段实现。

### 5.2 消息模型不同

OpenAI Chat 的消息模型是：

```json
[
  {"role": "system", "content": "..."},
  {"role": "user", "content": "..."},
  {"role": "assistant", "content": "..."}
]
```

Anthropic 是：

```json
{
  "system": "...",
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "..."}
      ]
    }
  ]
}
```

Gemini 是：

```json
{
  "systemInstruction": {
    "parts": [
      {"text": "..."}
    ]
  },
  "contents": [
    {
      "role": "user",
      "parts": [
        {"text": "..."}
      ]
    }
  ]
}
```

Bedrock 是：

```json
{
  "system": [
    {"text": "..."}
  ],
  "messages": [
    {
      "role": "user",
      "content": [
        {"text": "..."}
      ]
    }
  ]
}
```

表面上都能表示“系统提示 + 用户消息”，但差异包括：

- system 是 top-level 字段、message role，还是 systemInstruction。
- content 是字符串、content blocks，还是 parts。
- assistant/tool/function 消息是否允许混排。
- 多模态输入如何表示。
- 历史消息中是否允许 system 多次出现。
- provider 对 system/developer 指令优先级的解释不同。

这些不是简单 rename 字段就能完全解决的。

### 5.3 role 语义冲突

OpenAI 现在有 `system`、`developer`、`user`、`assistant`、`tool` 等角色。其他供应商不一定有 `developer` role，也不一定允许 tool message 以相同方式出现。

可能的处理方式：

| OpenAI role | 兼容处理 | 风险 |
| --- | --- | --- |
| `system` | 映射到 provider system 字段 | 通常可行 |
| `developer` | 合并进 system | 可能改变优先级 |
| `tool` | 映射成 tool_result/function_response | 需要保留 tool call id |
| `assistant` with tool_calls | 映射成 provider tool_use/function_call | 结构不一定一致 |
| 多个 system/developer | 合并成一个字符串 | 可能改变边界和顺序 |

对 agent 来说，这些差异会影响行为稳定性。例如 developer message 原本应该高于用户消息，如果被简单拼进普通 prompt，越狱抵抗和指令优先级可能下降。

### 5.4 tool calling 循环不同

OpenAI tool calling：

```text
assistant -> tool_calls[]
client executes tools
tool -> tool result message
assistant -> final answer
```

Anthropic tool use：

```text
assistant content contains tool_use block
client executes tools
user message contains tool_result block
assistant -> final answer
```

Gemini function calling：

```text
model returns functionCall part
client sends functionResponse part
model -> final answer
```

Bedrock tool use：

```text
assistant message contains toolUse block
client sends toolResult block
assistant -> final answer
```

Cohere tool use：

```text
assistant emits tool calls
client sends tool role messages
assistant -> final answer
```

差异点包括：

- tool call id 是否必需。
- tool arguments 是 JSON string 还是 JSON object。
- tool result 是 user message、tool message，还是 content block。
- 是否支持并行 tool calls。
- 是否支持 strict schema。
- 是否支持 server-side tools。
- 工具调用中途是否能流式输出 arguments。
- 工具失败如何表达。

如果 agent 框架依赖 OpenAI 的 `tool_calls[].id` 和 `role=tool`，映射到 Anthropic/Gemini 时必须做状态跟踪。否则多工具并发、工具结果归属、重试、错误恢复都会出问题。

### 5.5 结构化输出不是同一个东西

OpenAI 有 JSON mode 和 JSON Schema 结构化输出。其他供应商也有类似能力，但实现不完全一样：

- 有的只保证输出是 JSON，不保证符合 schema。
- 有的支持 JSON Schema 子集。
- 有的把 schema 放在 generation config。
- 有的通过 tool/function schema 间接约束输出。
- 有的严格模式会影响采样和拒答行为。
- 有的模型即使接口支持，实际遵循率也不稳定。

因此下面这几个概念不能混为一谈：

| 概念 | 含义 |
| --- | --- |
| JSON prompt | 提示模型“请输出 JSON” |
| JSON mode | 接口层要求输出 JSON object |
| JSON Schema | 接口层给出 schema |
| strict schema | 模型输出必须符合 schema |
| tool schema | 工具参数 schema，不一定等价最终回答 schema |

如果一个 OpenAI-compatible server 收到：

```json
{
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "answer",
      "strict": true,
      "schema": {
        "type": "object",
        "properties": {
          "score": {"type": "number"},
          "reason": {"type": "string"}
        },
        "required": ["score", "reason"],
        "additionalProperties": false
      }
    }
  }
}
```

而下游供应商只支持普通 JSON mode，那么网关如果静默降级，就会让调用方误以为拿到的是强约束结构化结果。这对评测、自动执行、代码生成、工作流系统都很危险。

### 5.6 流式协议不同

OpenAI Chat Completions streaming 常见形态是 SSE：

```text
data: {"choices":[{"delta":{"content":"Hel"}}]}
data: {"choices":[{"delta":{"content":"lo"}}]}
data: [DONE]
```

Responses API 的 streaming 更像事件流，会有 response created、output item、content delta、tool call delta、completed 等不同事件。

Anthropic streaming 会发 message start、content block start、delta、content block stop 等事件。

Gemini `streamGenerateContent` 返回 candidate chunks。

Bedrock ConverseStream 是 AWS event stream。

差异包括：

- event type 名称不同。
- 增量字段不同。
- tool arguments 增量如何表达不同。
- reasoning/thinking 是否流式输出不同。
- usage 是否在最后返回不同。
- 错误是否在 SSE 中返回不同。
- 是否使用 `[DONE]` 不同。

做兼容时可以把它们压成 OpenAI chunk，但会丢失部分事件语义。尤其是 Responses API 和 Realtime API，不应该简单降级成 Chat Completions stream。

### 5.7 服务端状态和资源不一致

OpenAI 有不少服务端状态：

- `previous_response_id`
- stored responses
- files
- vector stores
- batches
- fine-tuning jobs
- realtime sessions
- webhooks

很多供应商的基础 LLM API 是 stateless 的：每次请求必须把上下文完整传入。  

如果网关要兼容 OpenAI 的状态接口，就必须自己维护：

- response 存储。
- conversation/thread 映射。
- 文件对象。
- 向量索引。
- batch job 状态。
- session 生命周期。
- 权限隔离。
- 清理和过期策略。

这已经是一个平台后端，不是普通反向代理。

### 5.8 reasoning / thinking 字段不统一

现在很多模型有推理增强能力，但各家表达不一样：

- OpenAI 有 reasoning 相关配置和加密 reasoning item。
- Anthropic 有 extended thinking / thinking blocks。
- Gemini 有 thinking budget 等配置。
- DeepSeek 有 reasoning 模型和 thinking 控制。
- 有的供应商会返回可见思维摘要，有的只返回最终答案，有的完全不暴露。

兼容难点：

- 字段名不同。
- 是否计费不同。
- 是否可见不同。
- 是否允许流式输出不同。
- 安全策略不同。
- 模型行为差异很大。

如果把所有供应商都伪装成同一个 `reasoning` 字段，调用方很容易误判模型能力。

### 5.9 安全策略和 moderation 不统一

安全相关差异更难统一：

- OpenAI 有独立 Moderations API。
- Gemini 有 `safetySettings`。
- Bedrock 有 guardrails。
- Cohere 有 `safety_mode`。
- Azure OpenAI 有 Azure 内容过滤和企业策略。
- 很多供应商只在生成接口中返回 content filtered，不提供同等分类分数。

同一个输入在不同平台可能出现：

- 正常回答。
- 拒答。
- 部分改写。
- 返回安全错误。
- 返回空内容并标记 filtered。
- 需要调整 safety setting 才能返回。

所以安全层的兼容必须保守，不能承诺同一 moderation 分类体系完全一致。

### 5.10 鉴权、模型命名和部署方式不同

OpenAI：

```text
Authorization: Bearer sk-...
model: gpt-4.1-mini
```

Azure OpenAI：

```text
api-key: ...
URL contains deployments/{deployment-id}
api-version=...
```

AWS Bedrock：

```text
AWS SigV4
modelId can be model id, ARN, inference profile
```

Gemini：

```text
x-goog-api-key
model in URL path
```

Anthropic：

```text
x-api-key
anthropic-version header
```

这些差异意味着兼容网关需要做 provider credential management、model alias、region routing、deployment mapping、版本控制，而不是只改 URL。

### 5.11 错误码、限流和用量统计不同

OpenAI 错误对象一般类似：

```json
{
  "error": {
    "message": "...",
    "type": "...",
    "param": null,
    "code": "..."
  }
}
```

但各供应商的错误码、HTTP 状态、重试头、限流策略、配额维度、账单维度不同。

兼容风险：

- 下游 429 不一定等价 OpenAI 429。
- provider 的 safety block 如果被包装成普通 400，调用方无法正确处理。
- usage token 统计口径不同，例如 input/output、cache read/write、reasoning token、audio token、image token。
- cost 无法只靠 token 数统一计算。

### 5.12 多模态能力边界不同

即使大家都说支持 multimodal，也可能完全不同：

| 能力 | 差异 |
| --- | --- |
| image input | 支持 URL、base64、file id、multipart 的范围不同 |
| image output | 同步返回、异步任务、URL、base64 不同 |
| audio input | 支持格式、时长、采样率不同 |
| audio output | voice、format、streaming 不同 |
| video input | 很多 OpenAI-compatible chat server 不支持 |
| file input | 有的是真文件检索，有的只是把文本塞进 prompt |

所以一个网关可以做“形状兼容”，但不能假装所有模型都有同等多模态能力。

## 6. 如果要做一个兼容 server，建议怎么设计

### 6.1 不要宣称“全兼容”，要声明 capability matrix

建议每个 provider/model 都维护能力矩阵：

```json
{
  "provider": "anthropic",
  "model": "claude-3-5-sonnet-latest",
  "openai_chat_completions": true,
  "openai_responses": "partial",
  "stream": true,
  "tools": true,
  "parallel_tool_calls": false,
  "strict_json_schema": "partial",
  "vision_input": true,
  "audio_input": false,
  "audio_output": false,
  "embeddings": false,
  "files": false,
  "vector_stores": false,
  "batch": false,
  "realtime": false,
  "reasoning_controls": "provider_specific"
}
```

### 6.2 请求处理要有三种策略

| 策略 | 适用情况 | 行为 |
| --- | --- | --- |
| translate | provider 有等价能力 | 映射字段并调用 |
| emulate | provider 没有，但网关可自己实现 | 例如 files/vector store/batch |
| reject | 无法实现或会改变语义 | 返回明确 unsupported error |

最不建议的策略是 silent ignore。比如调用方传了 `response_format.strict=true`，网关不能偷偷丢掉 strict，然后返回看似成功的结果。

### 6.3 保留 provider-specific 扩展通道

建议允许：

```json
{
  "model": "deepseek-reasoner",
  "messages": [...],
  "extra_body": {
    "thinking": {
      "type": "enabled"
    }
  }
}
```

或者：

```json
{
  "model": "bedrock/anthropic.claude-3-5-sonnet-20240620-v1:0",
  "messages": [...],
  "provider_options": {
    "bedrock": {
      "guardrailConfig": {
        "guardrailIdentifier": "...",
        "guardrailVersion": "1"
      }
    }
  }
}
```

这样可以避免为了统一协议而损失供应商原生能力。

### 6.4 返回兼容警告

响应中可以加非标准 metadata：

```json
{
  "id": "chatcmpl_xxx",
  "object": "chat.completion",
  "choices": [...],
  "usage": {...},
  "compatibility": {
    "provider": "anthropic",
    "translated_from": "openai.chat.completions",
    "warnings": [
      "developer role was merged into system",
      "parallel_tool_calls is not supported by this provider"
    ]
  }
}
```

如果要严格兼容 OpenAI SDK，metadata 也可以放在 header 或日志中，不一定直接污染响应体。

### 6.5 对 agent 场景的推荐

如果你的场景是 text-only agent，优先支持这几个接口和字段：

1. `POST /v1/chat/completions`
2. `stream`
3. `messages`
4. `tools`
5. `tool_choice`
6. `response_format`
7. `temperature`
8. `max_tokens` / `max_completion_tokens`
9. `usage`
10. OpenAI-style error object

如果要面向新 OpenAI 能力，再加：

1. `POST /v1/responses`
2. `input` / output items
3. `instructions`
4. `previous_response_id`
5. `tools`
6. built-in tool 的显式 unsupported handling
7. Responses streaming events

不建议第一版就承诺：

- 完整 Realtime。
- 完整 Files / Vector Stores。
- 完整 Images / Audio。
- 完整 Batch / Fine-tuning。
- 所有 provider 的 reasoning 字段统一。
- 所有 provider 的 safety 语义统一。

### 6.6 一个现实的接口兼容路线图

| 阶段 | 实现内容 | 目标 |
| --- | --- | --- |
| Phase 1 | Chat Completions text + non-stream | 让大多数 SDK 能跑 |
| Phase 2 | Streaming + usage + OpenAI-style errors | 支持 CLI、IDE、chat UI |
| Phase 3 | Tool calling + tool result loop | 支持 agent |
| Phase 4 | JSON mode / structured output | 支持 workflow 和评测 |
| Phase 5 | Embeddings | 支持 RAG 基础能力 |
| Phase 6 | Responses API 子集 | 对齐 OpenAI 新协议 |
| Phase 7 | Files + Vector Stores 自建 | 支持 file search/RAG |
| Phase 8 | Batch / async jobs | 支持离线大规模任务 |
| Phase 9 | Realtime / Audio / Images | 支持多模态和实时系统 |

## 7. 推荐的兼容错误设计

当字段不支持时，不要返回 200。建议返回 OpenAI-like error：

```json
{
  "error": {
    "message": "The selected model does not support strict JSON Schema output.",
    "type": "unsupported_feature",
    "param": "response_format.json_schema.strict",
    "code": "unsupported_response_format"
  }
}
```

当可以降级但有风险时，建议调用方显式允许：

```json
{
  "model": "provider/model",
  "messages": [...],
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "strict": true,
      "schema": {}
    }
  },
  "compatibility_options": {
    "allow_degraded_structured_output": false
  }
}
```

默认应当严格失败，而不是默认降级。

## 8. 总结

OpenAI-compatible API 的价值很大，因为它给 SDK、agent 框架、IDE、网关、评测系统提供了一个事实标准。但它最适合统一的是“基础文本聊天”和“部分工具调用”，不是天然适合统一所有模型平台能力。

无法完全兼容主要有三类原因：

1. 下游供应商没有对应能力，例如 Realtime、File Search、Batch、Fine-tuning。
2. 能力相似但协议语义不同，例如 tool calling、structured output、streaming、system instruction。
3. 平台层差异无法抹平，例如鉴权、部署、region、safety、usage、billing、状态资源。

所以如果要做一个 server，最佳实践不是追求“所有字段都吞下去”，而是：

- 对 Chat Completions 做高质量兼容。
- 对 Responses API 做明确子集支持。
- 对每个模型维护 capability matrix。
- 对无法支持的字段明确报错。
- 对 provider-native 能力保留扩展通道。
- 对 Files、Vector Stores、Batch、Realtime 这类平台能力单独设计后端，而不是期待供应商自动兼容。

这样做出来的系统才适合 agent 和生产工作流，不会因为“看起来兼容”而在关键路径上静默丢能力。

## 9. 参考资料

- OpenAI API Reference: https://developers.openai.com/api/reference/overview
- OpenAI Responses API: https://platform.openai.com/docs/api-reference/responses
- OpenAI Chat Completions API: https://platform.openai.com/docs/api-reference/chat
- OpenAI Embeddings API: https://platform.openai.com/docs/api-reference/embeddings
- OpenAI Images API: https://platform.openai.com/docs/api-reference/images
- OpenAI Audio API: https://platform.openai.com/docs/api-reference/audio
- OpenAI Realtime API: https://platform.openai.com/docs/api-reference/realtime
- OpenAI Files API: https://platform.openai.com/docs/api-reference/files
- OpenAI Vector Stores API: https://platform.openai.com/docs/api-reference/vector-stores
- OpenAI Batch API: https://platform.openai.com/docs/api-reference/batch
- OpenAI Moderations API: https://platform.openai.com/docs/api-reference/moderations
- OpenAI migration guide from Chat Completions to Responses: https://developers.openai.com/api/docs/guides/migrate-to-responses
- Anthropic Messages API: https://docs.anthropic.com/en/api/messages
- Anthropic Tool Use: https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview
- Anthropic Messages Streaming: https://docs.anthropic.com/en/api/messages-streaming
- Google Gemini GenerateContent API: https://ai.google.dev/api/generate-content
- AWS Bedrock Converse API: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_Converse.html
- AWS Bedrock ConverseStream API: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_ConverseStream.html
- Mistral API Reference: https://docs.mistral.ai/api/
- Cohere Chat API: https://docs.cohere.com/v2/reference/chat
- Cohere Tool Use: https://docs.cohere.com/docs/tool-use
- Azure OpenAI API Reference: https://learn.microsoft.com/en-us/azure/ai-services/openai/reference
- DeepSeek API Docs: https://api-docs.deepseek.com/
- DashScope OpenAI compatibility: https://help.aliyun.com/zh/model-studio/compatibility-of-openai-with-dashscope
