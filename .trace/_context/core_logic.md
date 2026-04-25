# Stage 2.3: 核心領域邏輯分析

## 核心 Abstraction：WrappedServer

**檔案**: `src/cpp/include/lemon/wrapped_server.h`

這是整個系統的最核心 abstraction。每個後端繼承 `WrappedServer`，後者又繼承 `ICompletionServer`。

```
ICapability (虛擬基底)
  ├─ ICompletionServer  chat_completion(), completion()
  ├─ IEmbeddingsServer  embeddings()
  ├─ IRerankingServer   reranking()
  ├─ IAudioServer       audio_transcriptions()
  ├─ ITextToSpeechServer audio_speech()
  └─ IImageServer       image_generations(), image_edits(), image_variations()

WrappedServer : ICompletionServer
  ├─ LlamaCppServer   : WrappedServer, IEmbeddingsServer, IRerankingServer
  ├─ FastFlowLMServer : WrappedServer, IEmbeddingsServer, IAudioServer
  ├─ RyzenAIServer    : WrappedServer
  ├─ WhisperServer    : WrappedServer, IAudioServer
  ├─ SDServer         : WrappedServer, IImageServer
  └─ KokoroServer     : WrappedServer, ITextToSpeechServer
```

**執行時能力檢查**：
```cpp
// server_capabilities.h:62
template<typename T>
bool supports_capability(ICapability* server) {
    return dynamic_cast<T*>(server) != nullptr;
}
// 使用範例: supports_capability<IEmbeddingsServer>(server)
```

## 核心邏輯一：NPU 互斥排程
**檔案**: `src/cpp/server/router.cpp:260-320`

這是 Lemonade 最重要的業務邏輯——管理多個 NPU 後端的共存規則：

| 後端 Recipe | NPU 策略 | 可共存 |
|------------|---------|--------|
| `ryzenai-llm` | 獨佔整個 NPU | 不可與任何 NPU 後端共存 |
| `whispercpp` (NPU) | 獨佔整個 NPU | 不可與任何 NPU 後端共存 |
| `flm` | 分槽共享 NPU | 最多 1 LLM + 1 audio + 1 embedding（同時） |

FLM 共存規則實作（`router.cpp:282-305`）：
```
載入 FLM 模型時：
  1. 若存在 ryzenai-llm 或 whispercpp(NPU) → evict
  2. 若已有同 ModelType 的 FLM → evict（每槽最多 1 個）
  3. 允許不同 ModelType 的 FLM 共存
```

## 核心邏輯二：LRU 多模型管理
**檔案**: `src/cpp/server/router.cpp:75-150`

- `loaded_servers_: vector<unique_ptr<WrappedServer>>`
- 每個 `WrappedServer` 維護 `last_access_time_`（`std::chrono::steady_clock`）
- 每次推論後 `update_access_time()` 更新
- `max_loaded_models` 設定每個 ModelType 的上限（預設 1）

LRU eviction 選擇：
```
find_lru_server_by_type(ModelType type)
  → 在 loaded_servers_ 中找到 last_access_time_ 最舊的同類型 server
  → wait_until_not_busy()   // 等待推論完成
  → server->unload()        // 停止子程序
  → loaded_servers_.erase() // 移除
```

## 核心邏輯三：子程序 Subprocess 模型
**檔案**: `src/cpp/include/lemon/utils/process_manager.h`

每個後端以獨立子程序執行：
- `WrappedServer::load()` 啟動 `llama-server`/`whisper-server`/etc.
- 取得一個 OS 指派的 port（`choose_port()`）
- `wait_for_ready()` polling health endpoint 直到就緒（最多 600 秒）
- 所有推論請求以 HTTP proxy 方式轉發（`forward_request()` / `forward_streaming_request()`）

**設計理由**：子程序隔離可防止後端 crash 導致 lemond 崩潰，也允許各後端使用不同語言/runtime。

## 核心邏輯四：RecipeOptions 三層繼承
**檔案**: `src/cpp/include/lemon/recipe_options.h`

設定優先級（高 → 低）：
```
1. load 請求的 per-request options（最高優先）
2. 模型 registry 中的 model_info.recipe_options
3. config.json 的全域 defaults（最低優先）
```

`RecipeOptions::inherit()` 實作三層合併：
```cpp
// 呼叫者覆蓋被呼叫者（非空值優先）
RecipeOptions effective = request_opts.inherit(
    model_opts.inherit(config_defaults)
);
```

已知的 recipe options keys（`recipe_options.cpp`）：
- `ctx_size` — 上下文視窗大小
- `gpu_layers` — GPU offload 層數
- `backend` — 後端變體選擇（vulkan/rocm/cpu/metal）
- `args` — 直接傳給後端的額外 CLI 參數

## 核心邏輯五：ModelManager 模型解析
**檔案**: `src/cpp/server/model_manager.cpp`

模型識別系統（三層 aliasing）：
```
Public API 名稱 → canonical 名稱 → checkpoint 路徑

例：
"Gemma-4-E2B-it-GGUF" (public)
  → 內部 canonical 名稱（可能相同，也可能含版本後綴）
  → ~/.cache/lemonade/models/HF-repo/model.gguf (path)
```

Registry 載入邏輯（`model_manager.cpp`）：
1. `src/cpp/resources/server_models.json` — 內建 100+ 個模型
2. `user_models.json`（cache dir）— 使用者自訂/匯入模型
3. `recipe_options.json`（cache dir）— 使用者儲存的每模型設定
4. `discover_extra_models()` — 掃描 `extra_models_dir` 中的 GGUF 檔案

模型過濾（`filter_models_by_backend()`）：
- 依目前系統的 NPU/GPU 硬體可用性過濾
- `disable_model_filtering=true` 可關閉過濾
- 被過濾的模型記錄在 `filtered_out_models_` map

## 核心邏輯六：load 並發控制
**檔案**: `src/cpp/server/router.cpp`

```
load_mutex_：std::mutex
is_loading_：bool
load_cv_：std::condition_variable
```

保證同一時間只有一個 load 操作進行。load 期間釋放鎖（lock.unlock()）允許其他推論請求繼續，load 完成後重新獲鎖（lock.lock()）加入 loaded_servers_。

## 核心邏輯七：Web App 的 window.api mock
**檔案**: `src/cpp/server/server.cpp:600-680`

當 lemond 提供 Web App 時，它在 HTML 的 `</head>` 前注入 mock `window.api`，模擬 Tauri 的 `window.api` 合約：
- `getSettings()` / `saveSettings()` — 使用 localStorage
- `getServerPort()` — 返回 HTTP server port
- `openExternal(url)` — `window.open()`
- `isWebApp: true` — 讓 renderer 知道是 web 模式

這讓同一份 React renderer 可以在 Tauri 和純 web 環境中無縫執行。

## 核心設計模式

| 模式 | 實作位置 | 說明 |
|------|---------|------|
| Strategy | `WrappedServer` 繼承體系 | 不同後端的可替換推論策略 |
| Proxy | `forward_request()` / `forward_streaming_request()` | HTTP proxy 轉發給子程序 |
| Observer | `NetworkBeacon` UDP broadcast | 客戶端發現伺服器 |
| Factory | `Router::create_backend_server()` | 根據 recipe 建立對應 backend |
| Template Method | `WrappedServer::wait_for_ready()` | 可被子類覆蓋的健康檢查 |
| LRU Cache | `Router::loaded_servers_` + `last_access_time_` | 多模型記憶體管理 |
