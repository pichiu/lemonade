# Lemonade API 完整參考（Part 2）

> 接續 [API_SURFACE_part1.md](./API_SURFACE_part1.md)

---

## 4. OpenAI 相容 Endpoints（續）

### `POST /v1/images/generations`

文字提示生成圖像（stable-diffusion.cpp 後端）。

| 參數 | 必填 | 說明 |
|------|------|------|
| `model` | 是 | SD 模型（如 `SD-Turbo`、`SDXL-Turbo`） |
| `prompt` | 是 | 圖像描述文字 |
| `size` | 否 | `WIDTHxHEIGHT`，預設 `512x512` |
| `steps` | 否 | 推論步數（SD-Turbo 建議 4） |
| `cfg_scale` | 否 | 引導強度（SD-Turbo 建議 1.0） |
| `seed` | 否 | 隨機種子 |
| `n` | 否 | 目前僅支援 `1` |
| `response_format` | 否 | 僅支援 `b64_json` |

```bash
curl -X POST http://localhost:13305/v1/images/generations \
  -H "Content-Type: application/json" \
  -d '{"model":"SD-Turbo","prompt":"sunset mountain","size":"512x512","steps":4,"response_format":"b64_json"}'
```

**Response**：`{"created":1742927481,"data":[{"b64_json":"<base64 PNG>"}]}`

---

### `POST /v1/images/edits`

圖像局部編輯。接受 `multipart/form-data`。

| 參數 | 必填 | 說明 |
|------|------|------|
| `model` | 是 | 如 `Flux-2-Klein-4B`、`SD-Turbo` |
| `image` | 是 | 來源圖像（PNG 檔案） |
| `prompt` | 是 | 編輯描述 |
| `mask` | 否 | 遮罩圖像（白色 = 編輯區域） |
| `size` / `steps` / `cfg_scale` / `seed` | 否 | 同 generations |
| `n` | 否 | 1–10，預設 1 |
| `response_format` | 否 | 僅 `b64_json` |

```bash
curl -X POST http://localhost:13305/v1/images/edits \
  -F "model=Flux-2-Klein-4B" \
  -F "prompt=Add mountains in background" \
  -F "size=512x512" \
  -F "response_format=b64_json" \
  -F "image=@source.png"
```

---

### `POST /v1/images/variations`

生成圖像變體。接受 `multipart/form-data`，不支援 `prompt` 參數。

| 參數 | 必填 | 說明 |
|------|------|------|
| `model` | 是 | 如 `Flux-2-Klein-4B` |
| `image` | 是 | 來源圖像（PNG） |
| `size` | 否 | 輸出尺寸 |
| `n` | 否 | 1–10，預設 1 |
| `response_format` | 否 | 僅 `b64_json` |

---

### `POST /v1/images/upscale`

圖像放大（Real-ESRGAN 4x 超解析度）。接受 JSON（非 multipart）。

| 參數 | 必填 | 說明 |
|------|------|------|
| `image` | 是 | Base64 編碼 PNG |
| `model` | 是 | `RealESRGAN-x4plus` 或 `RealESRGAN-x4plus-anime` |

```bash
curl -X POST http://localhost:13305/v1/images/upscale \
  -H "Content-Type: application/json" \
  -d '{"image":"<base64>","model":"RealESRGAN-x4plus"}'
```

---

## 5. Lemonade 專有 Endpoints

所有以下 endpoint 均在 quad-prefix 下可用（`/api/v0/`、`/api/v1/`、`/v0/`、`/v1/`）。

### `POST /v1/pull`

下載並安裝模型。支援 `stream=true` 以 SSE 回報進度。

**安裝已登錄模型**：
```bash
curl -X POST http://localhost:13305/v1/pull \
  -H "Content-Type: application/json" \
  -d '{"model_name":"Qwen3-0.6B-GGUF"}'
```

**自訂模型（從 HuggingFace 登錄並安裝）**：
```json
{
  "model_name": "user.MyModel",
  "checkpoint": "unsloth/Phi-4-mini-instruct-GGUF:Q4_K_M",
  "recipe": "llamacpp",
  "reasoning": false,
  "vision": false
}
```

> 自訂模型 `model_name` 必須使用 `user.` namespace（如 `user.MyModel`）。

**SSE 串流進度格式**（`stream=true`）：
```
event: progress
data: {"file":"model.gguf","file_index":1,"total_files":2,"bytes_downloaded":1073741824,"bytes_total":2684354560,"percent":40}

event: complete
data: {"file_index":2,"total_files":2,"percent":100}
```

**Response**（非串流）：`{"status":"success","message":"Installed model: Qwen3-0.6B-GGUF"}`

---

### `GET /v1/pull/variants`

查詢 HuggingFace GGUF 倉庫的可用量化版本清單。

| 查詢參數 | 必填 | 說明 |
|---------|------|------|
| `checkpoint` | 是 | HF repo id，如 `unsloth/Qwen3-8B-GGUF` |

```bash
curl 'http://localhost:13305/v1/pull/variants?checkpoint=unsloth/Qwen3-8B-GGUF'
```

**Response**：包含 `variants[]`（每項有 `name`、`primary_file`、`size_bytes`、`sharded`），以及 `suggested_name`、`suggested_labels`、`mmproj_files`。

---

### `POST /v1/load`

將模型顯式載入記憶體（若未下載會先自動安裝）。

| 參數 | 必填 | 適用後端 | 說明 |
|------|------|---------|------|
| `model_name` | 是 | 全部 | 模型名稱 |
| `ctx_size` | 否 | llamacpp / flm / ryzenai | 上下文視窗大小 |
| `llamacpp_backend` | 否 | llamacpp | `vulkan`/`rocm`/`metal`/`cpu` |
| `llamacpp_args` | 否 | llamacpp | 自訂 llama-server 引數 |
| `whispercpp_backend` | 否 | whispercpp | `npu`/`cpu`/`vulkan` |
| `whispercpp_args` | 否 | whispercpp | 自訂 whisper-server 引數 |
| `steps` / `cfg_scale` / `width` / `height` | 否 | sd-cpp | 圖像生成參數 |
| `save_options` | 否 | 全部 | `true` 時將設定儲存至 `recipe_options.json` |

**設定優先順序**：Load 請求參數 > `recipe_options.json` > 環境變數/啟動引數 > 內建預設值。

```bash
curl -X POST http://localhost:13305/v1/load \
  -H "Content-Type: application/json" \
  -d '{"model_name":"Qwen3-0.6B-GGUF","ctx_size":8192,"llamacpp_backend":"rocm"}'
```

**Response**：`{"status":"success","message":"Loaded model: Qwen3-0.6B-GGUF"}`

---

### `POST /v1/unload`

從記憶體卸載模型（伺服器程序保持執行）。

| 參數 | 必填 | 說明 |
|------|------|------|
| `model_name` | 否 | 未指定時卸載所有已載入模型 |

```bash
# 卸載特定模型
curl -X POST http://localhost:13305/v1/unload \
  -H "Content-Type: application/json" \
  -d '{"model_name":"Qwen3-0.6B-GGUF"}'

# 卸載全部
curl -X POST http://localhost:13305/v1/unload
```

---

### `POST /v1/delete`

刪除模型（若已載入先卸載再刪除檔案）。

```bash
curl -X POST http://localhost:13305/v1/delete \
  -H "Content-Type: application/json" \
  -d '{"model_name":"Qwen3-0.6B-GGUF"}'
```

---

### `GET /v1/stats`

取得上一次請求的推論效能指標。

```bash
curl http://localhost:13305/v1/stats
```

**Response**：
```json
{
  "time_to_first_token": 2.14,
  "tokens_per_second": 33.33,
  "input_tokens": 128,
  "output_tokens": 5,
  "decode_token_times": [0.01, 0.02, 0.03],
  "prompt_tokens": 9
}
```

---

### `GET /v1/system-info`

系統硬體資訊與後端安裝狀態。

```bash
curl http://localhost:13305/v1/system-info
```

**Response 結構**：
- **系統資訊欄位**：`OS Version`、`Processor`、`Physical Memory`（Windows 額外含 `OEM System`、`BIOS Version`）
- **`devices`**：`cpu`、`amd_gpu[]`、`nvidia_gpu[]`、`amd_npu`（依硬體而異）
- **`recipes`**：每個後端（`llamacpp`/`whispercpp`/`sd-cpp`/`flm`/`ryzenai-llm`）的 `backends` 狀態
  - 狀態值：`unsupported` / `installable` / `update_required` / `installed`

---

### `POST /v1/install`

安裝或更新後端執行檔。

| 參數 | 必填 | 說明 |
|------|------|------|
| `recipe` | 是 | `llamacpp`、`flm`、`whispercpp`、`sd-cpp`、`ryzenai-llm` |
| `backend` | 是 | `vulkan`、`rocm`、`cpu`、`metal`、`default` |
| `stream` | 否 | `true` 時以 SSE 回報進度 |
| `force` | 否 | 跳過硬體過濾，強制安裝 `unsupported` 後端 |

```bash
curl -X POST http://localhost:13305/v1/install \
  -H "Content-Type: application/json" \
  -d '{"recipe":"llamacpp","backend":"vulkan"}'
```

**Response**：`{"status":"success","recipe":"llamacpp","backend":"vulkan"}`

---

### `POST /v1/uninstall`

移除後端執行檔（若有模型正在使用該後端，會先卸載模型）。

```bash
curl -X POST http://localhost:13305/v1/uninstall \
  -H "Content-Type: application/json" \
  -d '{"recipe":"llamacpp","backend":"vulkan"}'
```

---

### `GET /v1/system-stats`

即時系統資源使用量（CPU、記憶體、GPU 使用率等）。

```bash
curl http://localhost:13305/v1/system-stats
```

---

### `POST /v1/params`

查詢或更新已載入模型的推論參數。

---

### `POST /v1/log-level`

動態調整日誌等級。

```bash
curl -X POST http://localhost:13305/v1/log-level \
  -H "Content-Type: application/json" \
  -d '{"level":"debug"}'
```

有效等級：`trace`、`debug`、`info`、`warning`、`error`、`fatal`、`none`。

---

### `GET /live`

輕量存活探針，不計入模型狀態查詢，適合高頻輪詢（如 load balancer）。

> 注意：`/live` **不在** quad-prefix 下，直接以此路徑存取。

```bash
curl http://localhost:13305/live
```

**Response**：`{"status":"ok"}`（支援 HEAD 請求）

---

## 6. Ollama 相容 Endpoints

路由在 `/api/` prefix 下（**無版本號**），保留 Ollama 原始格式。

| 方法 | 路徑 | 說明 |
|------|------|------|
| `POST` | `/api/chat` | 對話補全（串流/非串流） |
| `POST` | `/api/generate` | 文字補全 + 圖像生成 |
| `GET` | `/api/tags` | 列出已下載模型 |
| `POST` | `/api/show` | 顯示模型詳情 |
| `DELETE` | `/api/delete` | 刪除模型 |
| `POST` | `/api/pull` | 下載模型（含進度） |
| `POST` | `/api/embed` | 新版嵌入格式 |
| `POST` | `/api/embeddings` | 舊版嵌入格式 |
| `GET` | `/api/ps` | 列出執行中模型 |
| `GET` | `/api/version` | 伺服器版本 |

> 不支援（回傳 501）：`/api/create`、`/api/copy`、`/api/push`

**切換至 Ollama 預設 port**：修改 `config.json` 的 `port` 為 `11434`，即可被 Ollama 整合 app 自動偵測。

**`POST /api/chat` 範例**：
```bash
curl -X POST http://localhost:13305/api/chat \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen3-0.6B-GGUF","messages":[{"role":"user","content":"Hello"}],"stream":false}'
```

---

## 7. Anthropic 相容 Endpoint

| 方法 | 路徑 | 說明 |
|------|------|------|
| `POST` | `/api/messages` | Anthropic Messages API 相容 |

支援欄位：`model`、`messages`、`system`、`max_tokens`、`temperature`、`stream`、基礎 `tools`。不支援的 Anthropic 特定欄位會被忽略並記錄警告。

> 路徑為 `/api/messages`（無版本號），接受 `?beta=true` 查詢參數。

```bash
curl -X POST http://localhost:13305/api/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: <your-key>" \
  -d '{
    "model": "Qwen3-0.6B-GGUF",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

**串流**（`"stream": true`）：回傳 SSE，格式遵循 Anthropic SSE 規範（`content_block_delta`、`message_stop` 等事件）。

**Tool Use 範例**：
```json
{
  "model": "Qwen3-0.6B-GGUF",
  "max_tokens": 1024,
  "tools": [{"name": "get_weather", "description": "Get weather", "input_schema": {"type": "object", "properties": {"location": {"type": "string"}}}}],
  "messages": [{"role": "user", "content": "What is the weather in Tokyo?"}]
}
```

---

## 8. WebSocket Realtime API

### 連線方式

1. 先呼叫 `GET /v1/health` 取得 `websocket_port`
2. 連線至 `ws://localhost:<websocket_port>/realtime?model=<model_name>`

```python
import websockets, asyncio, json

async def main():
    async with websockets.connect("ws://localhost:9000/realtime?model=Whisper-Tiny") as ws:
        msg = await ws.recv()  # session.created
        await ws.send(json.dumps({"type": "input_audio_buffer.append", "audio": "<base64 PCM16>"}))
```

### 音訊規格

| 項目 | 規格 |
|------|------|
| 取樣率 | 16 kHz |
| 聲道 | Mono（單聲道） |
| 位元深度 | 16-bit 有符號整數 PCM |
| 編碼 | Base64 |
| 建議 chunk 大小 | ~85–256 ms |

### Client → Server 訊息

| 類型 | 說明 |
|------|------|
| `session.update` | 設定 session（模型、VAD 參數） |
| `input_audio_buffer.append` | 送出音訊 chunk（base64 PCM16） |
| `input_audio_buffer.commit` | 強制觸發轉錄 |
| `input_audio_buffer.clear` | 清空緩衝區（不觸發轉錄） |

### Server → Client 訊息

| 類型 | 說明 |
|------|------|
| `session.created` | Session 建立，含 session ID |
| `session.updated` | Session 設定已更新 |
| `input_audio_buffer.speech_started` | VAD 偵測到語音開始 |
| `input_audio_buffer.speech_stopped` | VAD 偵測到語音結束，觸發轉錄 |
| `input_audio_buffer.committed` | Buffer 已提交 |
| `input_audio_buffer.cleared` | Buffer 已清除 |
| `conversation.item.input_audio_transcription.delta` | 中間（可替換）轉錄結果 |
| `conversation.item.input_audio_transcription.completed` | 最終轉錄結果 |
| `error` | 錯誤訊息 |

### VAD 設定（透過 `session.update`）

```json
{
  "type": "session.update",
  "session": {
    "model": "Whisper-Tiny",
    "turn_detection": {
      "threshold": 0.01,
      "silence_duration_ms": 800,
      "prefix_padding_ms": 250
    }
  }
}
```

設定 `turn_detection: null` 可停用伺服器端 VAD，改用手動 `commit`。

### WebSocket 日誌串流

路徑：`ws://localhost:<websocket_port>/logs/stream`（與 Realtime API 共用同一 port）

**訂閱流程**：連線後送出 `logs.subscribe` 訊息。

```json
{"type": "logs.subscribe", "after_seq": null}
```

**Server 回傳**：
- `logs.snapshot`：最多 5000 筆歷史日誌
- `logs.entry`：即時新增的日誌條目

```json
{
  "type": "logs.entry",
  "entry": {
    "seq": 1043,
    "timestamp": "2025-03-30 14:22:05.456",
    "severity": "Info",
    "tag": "Router",
    "line": "2025-03-30 14:22:05.456 [Info] (Router) Model loaded successfully"
  }
}
```

---

## 9. Internal Endpoints（管理用）

僅限 **localhost** 存取，需 `LEMONADE_ADMIN_API_KEY`（若有設定）。

| 方法 | 路徑 | 說明 |
|------|------|------|
| `POST` | `/internal/shutdown` | 優雅關閉 `lemond` 程序 |
| `GET` | `/internal/config` | 取得目前 `config.json` 完整內容 |
| `POST` | `/internal/set` | 更新 `config.json` 中的設定值 |
| `POST` | `/internal/cleanup-cache` | 清理快取目錄 |

**`POST /internal/set` 範例**：
```bash
curl -X POST http://localhost:13305/internal/set \
  -H "Content-Type: application/json" \
  -d '{"key":"log_level","value":"debug"}'
```

**`GET /internal/config` 回應**（摘要）：
```json
{
  "port": 13305,
  "host": "localhost",
  "log_level": "info",
  "ctx_size": 4096,
  "max_loaded_models": 1,
  "llamacpp": {"backend": "auto", "args": ""},
  "whispercpp": {"backend": "auto", "args": ""}
}
```

---

## 快速參考索引

| 功能 | 推薦 Endpoint |
|------|-------------|
| 對話（OpenAI SDK） | `POST /v1/chat/completions` |
| 文字生成（legacy） | `POST /v1/completions` |
| 向量嵌入 | `POST /v1/embeddings` |
| 文件重排序 | `POST /v1/reranking` |
| 語音辨識（批次） | `POST /v1/audio/transcriptions` |
| 即時語音辨識 | `WS ws://...<port>/realtime` |
| 文字轉語音 | `POST /v1/audio/speech` |
| 圖像生成 | `POST /v1/images/generations` |
| 圖像編輯 | `POST /v1/images/edits` |
| 圖像放大 | `POST /v1/images/upscale` |
| 列出模型 | `GET /v1/models` |
| 健康檢查 | `GET /v1/health` |
| 下載模型 | `POST /v1/pull` |
| 載入模型 | `POST /v1/load` |
| 卸載模型 | `POST /v1/unload` |
| 效能統計 | `GET /v1/stats` |
| 系統硬體資訊 | `GET /v1/system-info` |
| 安裝後端 | `POST /v1/install` |
| 存活探針 | `GET /live` |
| Ollama 對話 | `POST /api/chat` |
| Anthropic Messages | `POST /api/messages` |
| 關閉伺服器 | `POST /internal/shutdown` |
