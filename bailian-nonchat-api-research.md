# 阿里云百炼（DashScope Model Studio）非对话模型 HTTP API 调研报告

> 调研时间：2026-09-30 · 来源均为阿里云官方文档（help.aliyun.com / platform.qianwenai.com 官方新文档站）
> 用途：封装成 CLI 工具（curl / Python requests），不走 OpenAI 对话接口

## 0. 通用约定（所有接口共用）

**鉴权**（所有接口一致）：
```
Authorization: Bearer $DASHSCOPE_API_KEY
```

**域名**（二选一，同一地域的 Key 和域名必须匹配）：
- 旧域名（**CLI 封装推荐**，无需业务空间 ID）：`https://dashscope.aliyuncs.com`
- 新域名（官方推荐迁移，性能更好）：`https://{WorkspaceId}.cn-beijing.maas.aliyuncs.com`（`{WorkspaceId}` = 百炼业务空间 ID）
- 新加坡：`dashscope-intl.aliyuncs.com` / `{WorkspaceId}.ap-southeast-1.maas.aliyuncs.com`
- 官方明确「现有域名仍可正常使用」→ CLI 里做成 `BASE_URL=${DASHSCOPE_BASE_URL:-https://dashscope.aliyuncs.com}` 即可。

**异步任务统一轮询**：
```
GET {BASE_URL}/api/v1/tasks/{task_id}
Authorization: Bearer $DASHSCOPE_API_KEY
```
- 状态机：`PENDING` → `RUNNING` → `SUCCEEDED` / `FAILED`（`CANCELED`/`UNKNOWN`）
- task_id 查询有效期 **24 小时**；查询接口默认限流 20 QPS
- ⚠ Paraformer API 参考页表格把查询方法写成 POST 是文档笔误，其 curl 示例及全部其他页面均为 **GET**
- 支持批量查询/取消：见《管理异步任务》 help.aliyun.com/zh/model-studio/manage-asynchronous-tasks

**结果 URL 有效期**：图片 / 视频 / 音频 / 转写 JSON 的下载链接一律 **24 小时**，拿到后立刻下载落盘。

**本地文件怎么传**（各类接口差异大，汇总）：
| 接口 | 本地文件 base64 | 公网 URL | 其他 |
|---|---|---|---|
| 录音文件识别 filetrans（paraformer-v2 / qwen-audio-3.1-asr-flash-filetrans） | ❌ 明确不支持 | ✅ `input.file_urls`（HTTP/HTTPS，单次 1 个） | RESTful 可用 `oss://` 临时 URL（配 `X-DashScope-OssResourceResolve: enable`，48h，勿生产）；本地文件须先传 OSS 或走「上传文件获取临时 URL」接口 |
| 同步 ASR（qwen-audio-3.x-asr-flash / qwen3-asr-flash） | ✅ `input_audio.data` 放 data URI（≤10MB） | ✅ 同字段直接放 URL | — |
| TTS（qwen-audio-tts / cosyvoice） | 无输入文件 | — | 纯文本输入 |
| 文生图 qwen-image-3.0 | 纯文本 prompt | — | — |
| 图像翻译 qwen-mt-image-2.0 | ❌ 仅 URL | ✅ `input.image_url`（≤100MB） | — |
| 文生视频 wan3.0-video（可选配音频） | ❌ | ✅ `input.audio_url`（wav/mp3，≤15MB） | 支持 `oss://` 临时 URL |

---

## 1. 录音文件识别（异步 ASR）

模型：`paraformer-v2`、`qwen-audio-3.1-asr-flash-filetrans`（两者**同端点同流程**，只换 model）
来源：https://help.aliyun.com/zh/model-studio/non-realtime-speech-recognition-user-guide 、https://help.aliyun.com/zh/model-studio/paraformer-recorded-speech-recognition-restful-api

### ① URL + ② Method
```
POST {BASE_URL}/api/v1/services/audio/asr/transcription
Header: X-DashScope-Async: enable   ← 必带，否则报错
```

### ③ 请求体
```json
{
  "model": "qwen-audio-3.1-asr-flash-filetrans",
  "input": { "file_urls": ["https://example.com/audio.mp3"] },
  "parameters": {
    "channel_id": [0],
    "language_hints": ["zh", "en"],
    "disfluency_removal_enabled": false,
    "diarization_enabled": false,
    "speaker_count": 2
  }
}
```
- `file_urls`：公网 HTTP/HTTPS URL 数组，**单次仅支持 1 个 URL**；URL 含空格/中文要先 URL 编码
- `language_hints` 仅 paraformer-v2 支持；`diarization_enabled` 开说话人分离（结果多 `speaker_id`）
- 文件限制：单文件 ≤12 小时、≤2GB，任意采样率，aac/wav/mp3 等主流格式
- ❌ **不支持 base64、不支持二进制流、不支持本地文件路径**（官方 FAQ 原文）

### ④⑤ 响应 + 轮询
提交成功：
```json
{"output": {"task_status": "PENDING", "task_id": "c2e5d63b-..."}, "request_id": "..."}
```
轮询 `GET /api/v1/tasks/{task_id}` 直到 SUCCEEDED：
```json
{
  "output": {
    "task_status": "SUCCEEDED",
    "results": [{
      "file_url": "...",
      "transcription_url": "https://dashscope-result-bj.oss-...json?Expires=...",
      "subtask_status": "SUCCEEDED"
    }],
    "task_metrics": {"TOTAL": 1, "SUCCEEDED": 1, "FAILED": 0}
  },
  "usage": {"duration": 9}
}
```

### ⑥ 文字在哪
`transcription_url` 指向一个 24h 有效的 JSON 文件，下载后结构：
```
transcripts[].text                          ← 整段文本
transcripts[].sentences[].text              ← 句级（begin_time/end_time 毫秒）
transcripts[].sentences[].words[].text      ← 词级
```

### curl 全流程
```bash
BASE_URL="https://dashscope.aliyuncs.com"

# 1. 提交任务
TASK_ID=$(curl -sS -X POST "$BASE_URL/api/v1/services/audio/asr/transcription" \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-DashScope-Async: enable" \
  -d '{
    "model": "qwen-audio-3.1-asr-flash-filetrans",
    "input": {"file_urls": ["https://dashscope.oss-cn-beijing.aliyuncs.com/audios/welcome.mp3"]},
    "parameters": {"channel_id": [0], "language_hints": ["zh", "en"]}
  }' | jq -r .output.task_id)

# 2. 轮询（GET）
curl -sS "$BASE_URL/api/v1/tasks/$TASK_ID" -H "Authorization: Bearer $DASHSCOPE_API_KEY"

# 3. 下载识别结果 JSON（URL 24h 有效）
curl -sS '{transcription_url}' -o transcription.json
jq '.transcripts[].text' transcription.json
```

### 本地文件的替代路径：同步 ASR（≤5 分钟音频）
`qwen-audio-3.1-asr-flash`（同步，无需轮询）走 multimodal-generation 端点，`input_audio.data` 放公网 URL：
```bash
curl -X POST "$BASE_URL/api/v1/services/aigc/multimodal-generation/generation" \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  --data '{
    "model": "qwen-audio-3.1-asr-flash",
    "input": {"messages": [{"role": "user", "content": [
      {"type": "input_audio", "input_audio": {"data": "https://example.com/audio.wav"}}
    ]}]},
    "parameters": {"format": "wav", "sample_rate": "16000"}
  }'
```
⚠ 该端点响应**没有 choices**，文本在 `output.text`（或 `output.output.sentence.text`）。
长音频（>5min）走 filetrans，本地文件必须先上传到 OSS 公网可读（或用百炼「上传文件获取临时 URL」接口拿 `oss://dashscope-instant/...`）。

---

## 2. 语音合成（TTS）

### 2a. Qwen-Audio-TTS（qwen-audio-3.1-tts-flash，HTTP 可调，推荐）
来源：https://help.aliyun.com/zh/model-studio/non-realtime-tts-user-guide 、https://help.aliyun.com/zh/model-studio/qwen-audio-tts-http-api

**① URL + ② Method**
```
POST {BASE_URL}/api/v1/services/audio/tts/SpeechSynthesizer
```
（同端点也服务 cosyvoice-v3-flash / cosyvoice-v3.5-*；MiniMax 走 multimodal-generation 端点）

**③ 请求体**（text / voice / format / sample_rate 全在 `input` 里）
```json
{
  "model": "qwen-audio-3.1-tts-flash",
  "input": {
    "text": "我家的后面有一个很大的花园。",
    "voice": "longanhuan_v3.6",
    "format": "wav",
    "sample_rate": 24000
  }
}
```
- `format`：wav / mp3 / pcm；`voice` 音色列表见官方 Qwen-Audio-TTS 音色列表页
- qwen-audio-3.1-tts-flash 支持文本内嵌情感标签（`[excited]` `[sad]` `[laughing]` 等）和指令控制

**④ 响应**：**JSON（不是二进制流）**。非流式返回音频 URL（**24h 有效**）：
```json
{"output": {"audio": {"url": "https://dashscope-result-....wav", "id": "...", "expires_at": 1234567890}}, "request_id": "..."}
```
（字段结构经 DashScope Python SDK 源码 `http_speech_synthesizer.py` 交叉确认）

**⑤ 无需轮询**（同步接口）。

**保存成文件**：
```bash
AUDIO_URL=$(curl -sS -X POST "$BASE_URL/api/v1/services/audio/tts/SpeechSynthesizer" \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen-audio-3.1-tts-flash",
    "input": {"text": "你好，这是语音合成测试。", "voice": "longanhuan_v3.6", "format": "mp3", "sample_rate": 24000}
  }' | jq -r .output.audio.url)

curl -sS "$AUDIO_URL" -o tts_output.mp3   # 24h 内下载
```

**流式（要音频字节而非 URL 时）**：加头 `X-DashScope-SSE: enable`，SSE 每个事件的 `output.audio.data` 是 **base64 音频块**，`echo` 出来 base64 -d 拼接即可；最后一个 `finish_reason:"stop"` 事件也带完整 url。

### 2b. sambert-zhida-v1：⚠ 查不到 HTTP API
现行官方文档（2026-09 抓取）中 Sambert 系列只有 **Python / Java / Android SDK 与 WebSocket** 参考，非实时语音合成用户指南的 HTTP 端点清单里没有 sambert——**官方未提供/未公开 sambert 的 RESTful HTTP 调用方式**。
**最接近的可用方案**：HTTP 场景直接改用 `qwen-audio-3.1-tts-flash` / `qwen-audio-3.0-tts-flash` 或 `cosyvoice-v3-flash`（同一端点，见 2a），能力完全覆盖 sambert。

---

## 3. 文生图

### 3a. qwen-image-3.0 / qwen-image-3.0-pro（异步）
来源：https://platform.qianwenai.com/docs/api-reference/image-generation/qwen-text-to-image-30-async （千问 AI 平台官方文档，含完整 OpenAPI）

**① URL + ② Method**（注意：不是旧版 text2image/image-synthesis！）
```
POST {BASE_URL}/api/v1/services/aigc/image-generation/generation
Header: X-DashScope-Async: enable
```
（不带该头即为同步调用，直接返回结果；异步则轮询）

**③ 请求体**（messages 格式）
```json
{
  "model": "qwen-image-3.0",
  "input": {
    "messages": [{"role": "user", "content": [{"text": "一只戴墨镜的橘猫坐在窗台，写实风格"}]}]
  },
  "parameters": {
    "size": "2048*2048",
    "n": 1,
    "prompt_extend": true,
    "watermark": false,
    "negative_prompt": "低质量",
    "seed": 42
  }
}
```
- model 枚举：`qwen-image-3.0-pro`、`qwen-image-3.0`（2.0 系列及更早同端点同步）
- size：3.0 系列总像素 512*512~2048*2048、宽高比 1:8~8:1，不指定由模型自定；n：1-6
- prompt 上限：3.0 系列 4500 Token

**④⑤ 异步响应 + 轮询**：提交返回 `output.task_id/task_status`；轮询 `GET /api/v1/tasks/{task_id}`，SUCCEEDED 后：
```json
{"output": {"task_status": "SUCCEEDED",
  "results": [{"orig_prompt": "...", "actual_prompt": "...", "url": "https://dashscope-result-*.oss-...png?Expires=..."}],
  "task_metrics": {"TOTAL":1,"SUCCEEDED":1,"FAILED":0}},
 "usage": {"image_count": 1}}
```

**图片下载与有效期**：`results[].url` 为 PNG 直链，**24 小时有效**，`curl -o` 直接下载。
```bash
TASK_ID=$(curl -sS -X POST "$BASE_URL/api/v1/services/aigc/image-generation/generation" \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" -H "Content-Type: application/json" \
  -H "X-DashScope-Async: enable" \
  -d '{"model":"qwen-image-3.0","input":{"messages":[{"role":"user","content":[{"text":"一只戴墨镜的橘猫，写实风格"}]}]},"parameters":{"size":"1664*928"}}' \
  | jq -r .output.task_id)

curl -sS "$BASE_URL/api/v1/tasks/$TASK_ID" -H "Authorization: Bearer $DASHSCOPE_API_KEY" | jq -r '.output.results[0].url'
```

### 3b. 旧版万相 wan2.x 系列（对照）
`POST {BASE_URL}/api/v1/services/aigc/text2image/image-synthesis`（wan2.5-t2i-preview 等），请求体为扁平 `{"input":{"prompt":"..."},"parameters":{"size":"1024*1024","n":1}}`，轮询/结果字段同上。来源：https://www.alibabacloud.com/help/en/model-studio/first-call-to-image-and-video-api

### 3c. qwen-mt-image-2.0（图像文字翻译）
来源：https://platform.qianwenai.com/docs/api-reference/image-translation/qwen-mt-image/create-task

```
POST {BASE_URL}/api/v1/services/aigc/image2image/image-synthesis
Header: X-DashScope-Async: enable   （qwen-mt-image-2.0 不带头可同步）
```
```json
{
  "model": "qwen-mt-image",
  "input": {
    "image_url": "https://example.com/sign.webp",
    "source_lang": "zh",
    "target_lang": "en",
    "ext": {"domainHint": "...", "terminologies": [{"src":"应用程序接口","tgt":"API"}], "config": {"imageSegment": false}}
  }
}
```
- `image_url` **仅公网 URL**（JPG/PNG/BMP/WEBP 等，≤100MB，宽高 15~8192px，URL 不能含中文），不支持 base64
- 2.0 版支持 55 种语言互译；源/目标至少一方为中英（旧版限制）
- 轮询同通用接口；结果在 **`output.image_url`**（JPG，与原图同尺寸，24h 有效）
- 图中无文字时任务仍 SUCCEEDED，`output.message = "No text detected for translation"`

---

## 4. 文生视频

模型：`wan3.0-video`（指南确认）；`wan3.0-video-prime` 模型存在（官方模型页 qianwenai.com/models/wan3.0-video-prime），**同端点同请求结构，换 model 名即可**（prime 专属参数差异指南未单列，未逐项核实）。另有 wan2.7-t2v / wan2.6-t2v 走同端点。
来源：https://help.aliyun.com/zh/model-studio/text-to-video-guide 、https://help.aliyun.com/zh/model-studio/text-to-video-api-reference

**① URL + ② Method**
```
POST {BASE_URL}/api/v1/services/aigc/video-generation/video-synthesis
Header: X-DashScope-Async: enable   ← 必须，缺了报 "current user api does not support synchronous calls"
```

**③ 请求体**
```json
{
  "model": "wan3.0-video",
  "input": {
    "prompt": "一只小猫在月光下奔跑，电影质感",
    "negative_prompt": "低质量"
  },
  "parameters": {
    "resolution": "720P",
    "ratio": "16:9",
    "duration": 15,
    "watermark": true,
    "prompt_extend": true
  }
}
```
- wan3.0-video：最长 30 秒、480P/720P/1080P、自适应宽高比、自动配音；`input.audio_url` 可选配自定义音频（wav/mp3，2~30s，≤15MB，支持公网 URL 或 `oss://` 临时 URL）
- wan2.7-t2v：duration [2,15] 默认 5；resolution 720P/1080P 默认 1080P；ratio 16:9/9:16/1:1/4:3/3:4
- wan2.6-t2v：用 `size: "1280*720"` + `shot_type` 而非 resolution/ratio

**④⑤ 轮询**：`GET /api/v1/tasks/{task_id}`，生成约 1~5 分钟，**建议 15 秒间隔**。SUCCEEDED 后：
- 视频在 **`output.video_url`**（MP4 / H.264），**24 小时有效**，`curl -o video.mp4` 直接下载
- `usage`：duration（计费秒数）、SR（分辨率档）、ratio、video_count

```bash
TASK_ID=$(curl -sS -X POST "$BASE_URL/api/v1/services/aigc/video-generation/video-synthesis" \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" -H "Content-Type: application/json" \
  -H "X-DashScope-Async: enable" \
  -d '{"model":"wan3.0-video","input":{"prompt":"一只小猫在月光下奔跑"},"parameters":{"resolution":"720P","ratio":"16:9","duration":5}}' \
  | jq -r .output.task_id)

# 循环轮询直到 SUCCEEDED
curl -sS "$BASE_URL/api/v1/tasks/$TASK_ID" -H "Authorization: Bearer $DASHSCOPE_API_KEY"
```

---

## 5. 文本向量与重排

### 5a. Embedding（OpenAI 兼容接口，猜测正确 ✅）
来源：https://help.aliyun.com/zh/model-studio/text-embedding-synchronous-api

```
POST {BASE_URL}/compatible-mode/v1/embeddings
```
```bash
curl -sS "$BASE_URL/compatible-mode/v1/embeddings" \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3.7-text-embedding",
    "input": "风急天高猿啸哀，渚清沙白鸟飞回",
    "dimensions": 1024,
    "encoding_format": "float"
  }'
```
- `input`：**字符串或字符串数组均可**（数组=批量，qwen3.7 系列最多 20 行、单行 ≤128,000 Token）
- `dimensions`：qwen3.7-text-embedding 可选 2560/2048/1536/1024(默认)/768/512/256；flash 为 1024(默认)/768/512/256
- 响应为 OpenAI 标准：`data[].embedding`（float 数组，index 对应输入顺序）、`usage.prompt_tokens`
- 免费额度：qwen3.7-text-embedding 与 flash **各 100 万 Token**（开通后 90 天内）

### 5b. Rerank（DashScope 原生接口，非 OpenAI 兼容）
来源：https://help.aliyun.com/zh/model-studio/text-rerank-api

```
POST {BASE_URL}/api/v1/services/rerank/text-rerank/text-rerank
```
```bash
curl -sS -X POST "$BASE_URL/api/v1/services/rerank/text-rerank/text-rerank" \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3.7-text-rerank",
    "input": {
      "query": "什么是文本排序模型",
      "documents": ["文本排序模型广泛用于搜索引擎", "量子计算是前沿领域", "预训练语言模型带来新进展"]
    },
    "parameters": {
      "top_n": 2,
      "instruct": "Given a web search query, retrieve relevant passages that answer the query."
    }
  }'
```
- 响应：`output.results[]` 按 `relevance_score`（0~1）降序，含 `index`（原文档下标）；`return_documents: true` 可带回原文档
- ⚠ 端点拼写是 `rerank/text-rerank/text-rerank`（两层）。旧模型 `qwen3-rerank`（非 3.7）走另一 OpenAI 风格端点 `/compatible-api/v1/reranks`（顶层 query/documents/top_n），别混

---

## 6. 附加确认：qwen-audio 系列 OpenAI 兼容传 base64 音频

**结论：可以，但按模型分两条路。**

### 路径 A：OpenAI 兼容 `/compatible-mode/v1/chat/completions`
官方明确：**仅 Qwen3-ASR-Flash 系列模型（qwen3-asr-flash）支持 OpenAI 兼容方式**做语音识别。`content` 格式：
```json
{
  "model": "qwen3-asr-flash",
  "messages": [{
    "role": "user",
    "content": [
      {"type": "input_audio", "input_audio": {"data": "data:audio/mpeg;base64,SUQzBAAAAAAA..."}}
    ]
  }],
  "stream": false,
  "asr_options": {"language": "zh", "enable_itn": false}
}
```
- `input_audio.data`：公网 URL **或** base64 Data URI（`data:<mediatype>;base64,<data>`；wav=`audio/wav`、mp3=`audio/mpeg`）
- 编码后 ≤10MB；`asr_options` 为非 OpenAI 标准参数，放请求体顶层（SDK 用 extra_body）
- 识别文本在 `choices[0].message.content`
- 来源：https://help.aliyun.com/zh/model-studio/non-realtime-speech-recognition-user-guide （「使用OpenAI兼容API」节）

### 路径 B：DashScope 原生 multimodal-generation
`qwen-audio-3.x-asr-flash` 系列：同「1. 本地文件替代路径」的 curl，`input_audio.data` 同样支持 URL / data URI。
老模型 `qwen-audio-turbo`：字段名不同，是 `{"audio": "URL 或 data:;base64,<data>"}`（注意 data URI **不带 mediatype**），音频 >30 秒只处理前 30 秒，base64 后 ≤10MB：
```bash
curl -X POST "$BASE_URL/api/v1/services/aigc/multimodal-generation/generation" \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"qwen-audio-turbo","input":{"messages":[{"role":"user","content":[
    {"audio":"data:;base64,SUQzBAAAAAAA..."},{"text":"这段音频在说什么?"}]}]}}'
```
- 来源：https://help.aliyun.com/zh/model-studio/audio-language-model

**⚠ 查不到的模型名**：`qwen-audio-3.1-asr-flash-message` 在官方文档不存在；官方列表是 `qwen-audio-3.1-asr-flash`（同步）与 `qwen-audio-3.1-asr-flash-filetrans`（异步文件转写）。

---

## 来源 URL 汇总
- 非实时语音识别指南：https://help.aliyun.com/zh/model-studio/non-realtime-speech-recognition-user-guide
- Paraformer 录音文件识别 HTTP API：https://help.aliyun.com/zh/model-studio/paraformer-recorded-speech-recognition-restful-api
- 非实时语音合成指南：https://help.aliyun.com/zh/model-studio/non-realtime-tts-user-guide
- Qwen-Audio-TTS API：https://help.aliyun.com/zh/model-studio/qwen-audio-tts-http-api
- 文生视频指南（wan3.0）：https://help.aliyun.com/zh/model-studio/text-to-video-guide
- 万相文生视频 API 参考：https://help.aliyun.com/zh/model-studio/text-to-video-api-reference
- Postman/cURL 首调（图+视频）：https://www.alibabacloud.com/help/en/model-studio/first-call-to-image-and-video-api
- Qwen-Image 异步 3.0（含 OpenAPI）：https://platform.qianwenai.com/docs/api-reference/image-generation/qwen-text-to-image-30-async
- Qwen-MT-Image 图像翻译：https://platform.qianwenai.com/docs/api-reference/image-translation/qwen-mt-image/create-task
- 通用文本向量同步 API：https://help.aliyun.com/zh/model-studio/text-embedding-synchronous-api
- 文本排序 Rerank API：https://help.aliyun.com/zh/model-studio/text-rerank-api
- 音频理解 Qwen-Audio（base64）：https://help.aliyun.com/zh/model-studio/audio-language-model
- TTS 响应字段佐证（DashScope SDK 源码）：https://github.com/dashscope/dashscope-sdk-python/blob/main/dashscope/audio/http_tts/http_speech_synthesizer.py
