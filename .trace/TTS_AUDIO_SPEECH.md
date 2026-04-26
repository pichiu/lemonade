# `/v1/audio/speech` 技術深度研究文件

> **研究範圍**：`/v1/audio/speech` TTS 端點的完整技術棧，含 Custom Models 可用性分析與 koko subprocess ONNX 格式驗證。
> **關鍵檔案**：`src/cpp/server/server.cpp`, `src/cpp/server/router.cpp`, `src/cpp/server/model_manager.cpp`, `src/cpp/server/backends/kokoro_server.cpp`, `src/cpp/include/lemon/server_capabilities.h`

---

## 1. 請求完整流程

```
Client POST /v1/audio/speech
  {"model": "kokoro-v1", "input": "Hello", "voice": "af_heart"}
         │
         ▼
server.cpp:353  register_post("audio/speech", handle_audio_speech)
         │
         ▼
server.cpp:1881  handle_audio_speech()
  ├─ 驗證 "model" 欄位存在（必填）
  ├─ 驗證 "input" 欄位存在（必填）
  ├─ 驗證 "stream_format" == "audio"（若有）
  ├─ 驗證 "response_format" 在 MIME_TYPES 中（若有）
  ├─ auto_load_model_if_needed("kokoro-v1")  ← 若未載入則觸發載入
  └─ router_->audio_speech(request_json, sink)
         │
         ▼
router.cpp:677  Router::audio_speech()
  └─ execute_streaming(request, sink, lambda)
         │
         ▼
router.cpp:577  execute_streaming()
  ├─ 從 request["model"] 取出模型名稱
  ├─ find_server_by_model_name()  ← 在 loaded_servers_ 中查找
  └─ 呼叫 lambda(WrappedServer*)
         │
         ▼
router.cpp:679  dynamic_cast<ITextToSpeechServer*>(server)
  ├─ 失敗 → throw UnsupportedOperationException  ← 非 TTS backend 時
  └─ 成功 → tts_server->audio_speech(request, sink)
         │
         ▼
kokoro_server.cpp:171  KokoroServer::audio_speech()
  ├─ tts_request["model"] = "kokoro"  ← 強制覆蓋 model 欄位
  ├─ 若有 stream_format → tts_request["stream"] = true
  └─ forward_streaming_request("/v1/audio/speech", ...)
         │
         ▼
koko subprocess (127.0.0.1:<port>)
  OpenAI-compatible /v1/audio/speech
  → 回傳 mp3 / wav / opus / pcm 音訊串流
```

**支援的回傳格式**（`server.cpp:118`）：

| `response_format` | Content-Type |
|---|---|
| `mp3`（預設） | `audio/mpeg` |
| `opus` | `audio/opus` |
| `aac` | `audio/aac` |
| `flac` | `audio/flac` |
| `wav` | `audio/wav` |
| `pcm`（streaming） | `audio/l16;rate=24000;endianness=little-endian` |

---

## 2. Backend 能力介面架構

```
ICapability (virtual base)
    │
    ├── ICompletionServer   → chat_completion(), completion()
    ├── IEmbeddingsServer   → embeddings()
    ├── IRerankingServer    → reranking()
    ├── IAudioServer        → audio_transcriptions()  ← speech-to-TEXT (ASR)
    ├── ITextToSpeechServer → audio_speech()          ← TEXT-to-speech (TTS)
    └── IImageServer        → image_generations(), image_edits(), image_variations()

WrappedServer : ICompletionServer (abstract base)
    │
    ├── LlamaCppServer   : WrappedServer
    ├── WhisperServer    : WrappedServer, IAudioServer
    ├── FastFlowLMServer : WrappedServer, IEmbeddingsServer, IRerankingServer, IAudioServer
    ├── SDServer         : WrappedServer, IImageServer
    ├── RyzenAIServer    : WrappedServer
    └── KokoroServer     : WrappedServer, ITextToSpeechServer  ← 唯一實作 TTS
```

**Router 分派機制**（`router.cpp:679`）：

```cpp
auto tts_server = dynamic_cast<ITextToSpeechServer*>(server);
if (!tts_server) throw UnsupportedOperationException(...);
tts_server->audio_speech(request, sink);
```

`dynamic_cast` 在執行期檢查繼承關係。**任何**繼承 `ITextToSpeechServer` 的 backend 都能服務此端點——架構上已預留擴充空間，但目前只有 `KokoroServer`。

---

## 3. Backend 能力對照表

| Backend | TTS `/audio/speech` | ASR `/audio/transcriptions` | LLM chat | Embeddings | Image |
|---|---|---|---|---|---|
| `KokoroServer` | ✅ **唯一** | ✗ | ✗（stub，回傳錯誤） | ✗ | ✗ |
| `FastFlowLMServer` | ✗ | ✅ | ✅ | ✅ | ✗ |
| `WhisperServer` | ✗ | ✅ | ✗ | ✗ | ✗ |
| `LlamaCppServer` | ✗ | ✗ | ✅ | ✅ | ✗ |
| `SDServer` | ✗ | ✗ | ✗ | ✗ | ✅ |
| `RyzenAIServer` | ✗ | ✗ | ✅ | ✗ | ✗ |

> FastFlowLM 的 "Audio" 是 **speech-to-text（ASR）**，實作的是 `IAudioServer` 而非 `ITextToSpeechServer`，**不能**用於 TTS。

---

## 4. Backend 實體化：封閉的 if/else 鏈

`router.cpp:181` `Router::create_backend_server()` 是唯一實體化 backend 的地方：

```cpp
if      (recipe == "whispercpp")  → WhisperServer
else if (recipe == "kokoro")      → KokoroServer
else if (recipe == "sd-cpp")      → SDServer
else if (recipe == "flm")         → FastFlowLMServer
else if (recipe == "ryzenai-llm") → RyzenAIServer
else                              → LlamaCppServer  ← 預設 catch-all
```

**沒有**工廠 map、沒有 `register_backend()` hook、沒有 `dlopen` plugin 機制。新增 backend 必須修改此函式並重新編譯。

---

## 5. Custom Models 系統

### 5.1 模型來源優先序

```
server_models.json（唯讀，隨 binary 發布）
    > user_models.json（cache_dir/user_models.json，使用者可編輯）
    > extra_models_dir（.gguf 自動掃描，僅 llamacpp）
    > flm list（FastFlowLM 動態清單）
```

### 5.2 user_models.json 格式

```json
{
    "MyKokoroModel": {
        "checkpoint": "org/repo",
        "recipe": "kokoro",
        "size": 0.5,
        "labels": ["tts"]
    }
}
```

API/CLI 引用時加 `user.` 前綴：`user.MyKokoroModel`

支援的 `recipe` 值：`llamacpp` | `whispercpp` | `sd-cpp` | `kokoro` | `ryzenai-llm` | `flm`

### 5.3 Checkpoint 解析流程（kokoro recipe 專屬）

`model_manager.cpp:688`：

```
checkpoint: "org/repo"
    │
    ▼
HuggingFace cache: <cache_dir>/models--org--repo/snapshots/<hash>/
    │
    ▼
遞迴搜尋第一個名為 "index.json" 的檔案
    │
    ▼
resolved_path() = 該 index.json 的完整路徑
```

與 llamacpp（直接指向 `.gguf` 檔）和 whispercpp（指向 `.bin` 檔）的解析方式不同。

---

## 6. KokoroServer 啟動流程

`kokoro_server.cpp:48`：

```
1. backend_manager_->install_backend("kokoro", "cpu")
   └─ 從 lemonade-sdk/Kokoros releases 下載 koko binary（版本 b16）
      ├─ Linux:   kokoros-linux-x86_64.tar.gz
      └─ Windows: kokoros-windows-x86_64.tar.gz
      （macOS 不支援，直接 throw）

2. fs::path model_path = model_info.resolved_path()
   └─ 指向 index.json 的完整路徑

3. json model_index = JsonUtils::load_from_file(model_path)
   └─ 讀取 index.json，取出：
      model_index["model"]   → ONNX 模型相對路徑
      model_index["voices"]  → voices .bin 相對路徑

4. model_dir = model_path.parent_path()

5. 設定環境變數：
   ESPEAK_DATA_PATH = <exe_dir>/espeak-ng-data
   LD_LIBRARY_PATH  = <exe_dir>:$LD_LIBRARY_PATH  （Linux）

6. 啟動子行程：
   koko -m <model_dir>/<model_index["model"]>
        -d <model_dir>/<model_index["voices"]>
        openai --ip 127.0.0.1 --port <chosen_port>

7. wait_for_ready("/")
   └─ 每 100ms poll GET http://127.0.0.1:<port>/，最多等 300s
```

---

## 7. koko Subprocess 技術規格（黑盒驗證）

> 以下為直接 trace `lemonade-sdk/Kokoros` 原始碼驗證所得。

### 7.1 ONNX 模型 Tensor 介面

`kokoros/src/onn/ort_koko.rs` 確認：

**Input Tensors：**

| 名稱 | Shape | Dtype | 說明 |
|---|---|---|---|
| `"tokens"` / `"input_ids"` | `[batch, seq_len]` | `i64` | 音素 token 序列 |
| `"style"` | `[batch, style_dim]` | `f32` | 聲音風格 embedding（從 voices.bin 查表） |
| `"speed"` | `[1]` | `f32` | 語速係數 |

**Output Tensors（自動偵測策略）：**

```
sess.outputs().len() == 1  →  Standard 策略
    output: "audio" 或 "waveforms"  (f32 waveform)

sess.outputs().len() > 1   →  Timestamped 策略
    output[0]: "waveform" 或 "audio"  (f32 waveform)
    output[1]: "durations"            (f32)
```

**重點**：策略依輸出數量**自動偵測**，無 hardcode 限制特定 checkpoint。

### 7.2 Voices `.bin` 檔格式

`kokoros/src/tts/koko.rs` 確認：

```
副檔名：.bin（實際是 NPZ 格式，即 NumPy compressed archive）
載入：NpzReader::new(File::open(voices_path))

每個 voice entry：
  key:   voice 名稱 (e.g. "af_sky", "af_heart", "am_echo")
  value: Array3<f32>，shape [511, 1, 256]
         ↑ 511 個 style embedding（按 token 序列長度索引）
         每個 embedding：[1][256] f32

查表：mix_styles() → styles[style_name][tokens_len][0]
```

### 7.3 `index.json` 完整格式（`mikkoph/kokoro-onnx` 實測）

```json
{
    "model_name": "kokoro",
    "model": "kokoro-v1.0.onnx",
    "voices": "voices-v1.0.bin"
}
```

| 欄位 | Lemonade 讀取？ | koko 讀取？ | 說明 |
|---|---|---|---|
| `model_name` | ✗（未讀取） | ✅ | 模型識別名，僅供 koko 內部使用 |
| `model` | ✅ | — | ONNX 檔相對路徑，拼接成 `-m` 參數 |
| `voices` | ✅ | — | voices .bin 相對路徑，拼接成 `-d` 參數 |

**注意**：Lemonade `kokoro_server.cpp:69` 的 log 寫 `model_index["model"]`（ONNX 路徑），不是 `model_name`。

---

## 8. Custom Models 可用性完整分析

### 8.1 端到端可行性矩陣

| 自訂場景 | Lemonade 層 | koko 層 | 整體可行？ | 條件 |
|---|---|---|---|---|
| 換量化版本（Q8/Q4 kokoro ONNX） | ✅ | ✅ | **可行** | 相同 tensor 介面 |
| Fine-tuned Kokoro（同架構） | ✅ | ✅ | **可行** | 相同 I/O interface |
| 自訂 voices（新語者/語言） | ✅ | ✅ | **可行** | NPZ 格式，shape `[511, 1, 256]` f32 |
| 不同架構 TTS ONNX（VITS 等） | ✅ | ✗ | **不可行** | tensor 名稱/shape 不符 |
| 指向目錄而非 .bin 檔 | ✅ | ✗ | **不可行** | `NpzReader::new(File::open(...))` 失敗 |
| 用 FastFlowLM 做 TTS | ✗ | — | **不可行** | FLM 未實作 `ITextToSpeechServer` |

### 8.2 已知 Bug：kokoro recipe 沒有自動加 `"tts"` label

`model_manager.cpp:1643`，`register_user_model()` 的自動 label 邏輯：

```cpp
if (recipe == "sd-cpp")      labels.insert("image");   // ✅ 有
if (recipe == "whispercpp")  labels.insert("audio");   // ✅ 有
// kokoro → 只加 "custom"，沒有 "tts"             // ❌ 遺漏
```

**後果鏈**：

```
labels: ["custom"]（無 "tts"）
    │
    ▼
model_types.h:109  get_model_type_from_labels()
    "custom" 不符任何特殊 label → return ModelType::LLM  ← 應為 ModelType::TTS
    │
    ▼
router.cpp:315  find_lru_server_by_type(LLM)
    自訂 kokoro 模型會被 LLM LRU 驅逐策略影響  ← 資源管理錯誤
    │
    ▼
router.cpp:493  get_max_model_limits()
    {"tts": max} 有獨立上限，但自訂 kokoro 走 LLM bucket  ← 計數錯誤
```

**TTS 功能本身仍可運作**（`dynamic_cast<ITextToSpeechServer*>` 只看繼承，不看 ModelType），但資源管理行為不正確。

**Workaround**：`user_models.json` 必須手動加 `"labels": ["tts"]`：

```json
{
    "MyKokoroModel": {
        "checkpoint": "org/repo",
        "recipe": "kokoro",
        "size": 0.5,
        "labels": ["tts"]
    }
}
```

---

## 9. 正確的自訂 Kokoro 模型設定步驟

### Step 1：確認 HuggingFace repo 結構

目標 repo 必須包含：

```
org/repo/
    index.json          ← 必須有此檔案（koko 模型清單）
    <model>.onnx        ← Kokoro 架構 ONNX（inputs: tokens/style/speed）
    <voices>.bin        ← NPZ 格式，shape [511, 1, 256] f32
```

`index.json` 內容：
```json
{
    "model_name": "kokoro",
    "model": "<model>.onnx",
    "voices": "<voices>.bin"
}
```

### Step 2：寫入 `user_models.json`

路徑：`<lemonade_cache_dir>/user_models.json`

```json
{
    "MyKokoroModel": {
        "checkpoint": "org/repo",
        "recipe": "kokoro",
        "size": 0.5,
        "labels": ["tts"]
    }
}
```

> `"labels": ["tts"]` **必填**，用於修正 Bug #8.2，確保 ModelType 正確設為 TTS。

### Step 3：下載與執行

```bash
lemonade pull user.MyKokoroModel
lemonade run user.MyKokoroModel
```

或直接用 API：

```bash
curl -X POST http://localhost:13305/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"model": "user.MyKokoroModel", "input": "Hello world", "voice": "af_heart"}' \
  --output output.mp3
```

---

## 10. 關鍵檔案索引

| 檔案 | 行號 | 重要內容 |
|---|---|---|
| `src/cpp/server/server.cpp` | 353 | TTS 路由註冊 |
| `src/cpp/server/server.cpp` | 1881 | `handle_audio_speech()` 完整 handler |
| `src/cpp/server/router.cpp` | 181 | `create_backend_server()` if/else 鏈 |
| `src/cpp/server/router.cpp` | 677 | `Router::audio_speech()` + `dynamic_cast` |
| `src/cpp/server/model_manager.cpp` | 688 | kokoro `index.json` 路徑解析 |
| `src/cpp/server/model_manager.cpp` | 1643 | `register_user_model()` label 自動化（Bug 所在） |
| `src/cpp/server/backends/kokoro_server.cpp` | 48 | `KokoroServer::load()` 完整啟動流程 |
| `src/cpp/server/backends/kokoro_server.cpp` | 171 | `KokoroServer::audio_speech()` 請求轉發 |
| `src/cpp/include/lemon/server_capabilities.h` | 48 | `ITextToSpeechServer` 介面定義 |
| `src/cpp/include/lemon/model_types.h` | 85 | `get_model_type_from_labels()` |
| `src/cpp/resources/server_models.json` | 1412 | `kokoro-v1` 唯一內建 TTS 模型 |
| `lemonade-sdk/Kokoros` `koko/src/main.rs` | — | CLI 參數：`-m` ONNX, `-d` voices.bin |
| `lemonade-sdk/Kokoros` `kokoros/src/onn/ort_koko.rs` | — | ONNX tensor 介面 |
| `lemonade-sdk/Kokoros` `kokoros/src/tts/koko.rs` | — | Voices NPZ 格式載入 |

---

## 11. 待修正事項

| # | 問題 | 位置 | 修正方式 |
|---|---|---|---|
| 1 | `register_user_model()` 未自動加 `"tts"` label for kokoro recipe | `model_manager.cpp:1643` | 仿照 `sd-cpp`/`whispercpp` 加 `if (recipe == "kokoro") labels.insert("tts");` |
| 2 | `docs/api/openai.md:842` 說「No other model is supported」與實際可用性不符 | `docs/api/openai.md` | 更新文件說明 custom kokoro checkpoint 的支援條件 |
