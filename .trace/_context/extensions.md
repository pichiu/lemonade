# Stage 2.4: Extension Points 分析

## 新增後端（最主要的擴展點）

**位置**: `src/cpp/include/lemon/backends/`、`src/cpp/server/backends/`

要新增一個新的推論後端，需要：

### 步驟 1：建立 Header 和實作
```cpp
// src/cpp/include/lemon/backends/my_backend.h
class MyBackend : public WrappedServer, public ICompletionServer {
    // 必實作的 virtual methods：
    void load(model_name, model_info, options, do_not_upgrade) override;
    void unload() override;
    json chat_completion(const json& request) override;
    json completion(const json& request) override;
    json responses(const json& request) override;
    // 選擇性 capability：繼承 IEmbeddingsServer, IAudioServer 等
};
```

### 步驟 2：在 Router::create_backend_server() 加入分支
`src/cpp/server/router.cpp:150-220`
```cpp
} else if (model_info.recipe == "my-recipe") {
    new_server = std::make_unique<backends::MyBackend>(log_level, model_manager_, backend_manager_);
}
```

### 步驟 3：在 BackendManager 中加入 install 邏輯
`src/cpp/include/lemon/backend_manager.h`、`src/cpp/server/backend_manager.cpp`

### 步驟 4：在 backend_versions.json 加入版本
`src/cpp/resources/backend_versions.json`

### 步驟 5：在 server_models.json 加入 recipe
`src/cpp/resources/server_models.json`

## 新增 API Endpoint（Quad-Prefix 模式）

**位置**: `src/cpp/server/server.cpp — Server::setup_routes()`

Critical Invariant：每個 endpoint 必須在 4 個 prefix 下各一份：
```cpp
// 正確做法（使用 register_get / register_post）：
register_get("my-endpoint", handler);
// 等同於：
web_server.Get("/api/v0/my-endpoint", handler);
web_server.Get("/api/v1/my-endpoint", handler);
web_server.Get("/v0/my-endpoint", handler);
web_server.Get("/v1/my-endpoint", handler);
```

`register_post` 額外加入 405 GET handler（HEAD request 支援）。

對應的 Router method 需要：
1. 在 `Router` header 新增宣告 (`src/cpp/include/lemon/router.h`)
2. 在 `Router` 實作中加入推論邏輯
3. 對應的 `WrappedServer` capability interface（若後端也需要實作）

## 自訂模型匯入

**位置**: `src/cpp/server/model_manager.cpp`

用戶可透過兩種方式新增自訂模型：

### 1. user_models.json（無需重啟）
- cache dir 下的 `user_models.json`
- 與 `server_models.json` 同格式
- 透過 `POST /v1/pull` 或 `lemonade pull CHECKPOINT` 建立
- `register_user_model()` 方法 — `model_manager.cpp`

### 2. extra_models_dir（GGUF 掃描）
- `config.json` 的 `extra_models_dir` 欄位
- `discover_extra_models()` 自動掃描目錄中的 `.gguf` 檔案
- 自動產生 recipe 為 `llamacpp` 的 ModelInfo

## 後端 Binary 熱替換

**位置**: `src/cpp/server/server.cpp — apply_config_side_effects()`

當用戶透過 `lemonade config set llamacpp.vulkan_bin=b8664` 改變 backend binary：

```
apply_config_side_effects(applied_changes)
  └─ handle_bin_change(section, bin_key, new_value)
       ├─ 卸載所有使用此後端的模型
       ├─ 下載/替換 binary（若版本不符）
       └─ 重新載入模型（best-effort）
```

不需要重啟 `lemond`，即時生效。

## Ollama API 擴展

**位置**: `src/cpp/server/ollama_api.cpp`、`src/cpp/include/lemon/ollama_api.h`

`OllamaApi` 是一個獨立的 API adapter，透過 `register_routes(httplib::Server&)` 掛載到 http_server。

新增 Ollama 相容路由：
```cpp
// ollama_api.cpp 中加入新的 route handler
web_server.Post("/api/new-endpoint", [this](...) { ... });
// 格式轉換：Ollama 格式 ↔ OpenAI 格式
// 呼叫 router_->existing_method()
```

## Anthropic API 擴展

**位置**: `src/cpp/server/anthropic_api.cpp`

獨立 adapter，透過 `POST /api/messages` 掛載（不在 quad-prefix 下，這是唯一例外）。

## WebSocket 擴展

**位置**: `src/cpp/server/websocket_server.cpp`、`src/cpp/include/lemon/websocket_server.h`

`WebSocketServer` 與 HTTP server 並列執行，綁定獨立 port。Realtime session 管理在 `src/cpp/server/realtime_session.cpp`。

## 前端 window.api 擴展

**Tauri 路徑**: `src/app/src/renderer/tauriShim.ts`  
- 每個新 Tauri invoke 命令需要在此加入對應方法

**Web App 路徑**: `src/cpp/server/server.cpp`（注入的 mock window.api）  
- 同步更新 Tauri 和 web mock 的方法簽名（`server.cpp:600-680`）

## 系統資訊探測（SIGHUP 觸發）

**位置**: `src/cpp/server/system_info.cpp`

`SystemInfoCache::invalidate_recipes()` 重新探測：
- CPU 型號（NPU 可用性判斷）
- GPU 型號和 VRAM（llama.cpp backend 選擇）
- NPU 驅動（FLM/RyzenAI 可用性）

此機制讓系統資訊可在不重啟的情況下刷新（例如驅動安裝後）。

## Plugin 化程度評估

Lemonade 目前**不是**完全 plugin 化的架構（沒有動態載入 .so/.dll 的機制）。後端新增需要重新編譯。但透過 subprocess 模型，可以將任何語言/框架的推論伺服器包裝為 `WrappedServer` 子類，只需實作 HTTP forward 邏輯。
