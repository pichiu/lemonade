# Stage 2.2: Request / Data Flow 分析

## 代表性 Use Case：Chat Completion（串流）

以下 trace `POST /v1/chat/completions` 從接收到輸出的完整路徑。

### 第一層：HTTP 接收與認證
`src/cpp/server/server.cpp`

```
httplib::Server (8 thread pool)
  └─ pre_routing_handler (server.cpp:256)
       ├─ log_request()              // 非 health/stats 路由才記錄
       └─ authenticate_request()     // 驗證 LEMONADE_API_KEY
            ├─ 若 admin_api_key_ 匹配 → 允許所有路由
            └─ 若 api_key_ 匹配 → 允許 /api, /v0, /v1 路由（不含 /internal）
```

### 第二層：路由分派
`src/cpp/server/server.cpp` `register_post("chat/completions", ...)`

```
handle_chat_completions(req, res)  // server.cpp
  ├─ 解析 req.body → nlohmann::json request_json
  ├─ 抽取 model 欄位
  ├─ should_disable_thinking()      // 處理 enable_thinking / thinking 欄位
  ├─ auto_load_model_if_needed()    // 若模型未載入，呼叫 router_.load_model()
  │    └─ Router::load_model()      // (見下方)
  ├─ 判斷 stream 欄位
  │    ├─ stream = true  → 串流路徑
  │    └─ stream = false → 非串流路徑
  └─ [串流路徑]
       res.set_chunked_content_provider("text/event-stream", ...)
       └─ router_.chat_completion_stream(request_body, sink)
```

### 第三層：Router 推論分派
`src/cpp/server/router.cpp`

```
Router::chat_completion_stream(request_body, sink)
  └─ execute_streaming(request_body, sink, streaming_func)
       ├─ 取得 std::lock_guard<std::mutex>     // load_mutex_
       ├─ 從 request_body 解析 model 欄位
       ├─ resolve_model_name()                 // 公開名稱 → canonical 名稱
       ├─ find_server_by_model_name()          // 在 loaded_servers_ 中尋找
       ├─ 若未找到 → 自動載入（auto-load path）
       ├─ server->update_access_time()         // 更新 LRU 時間戳
       ├─ server->set_busy(true)               // 標記 busy（防止被 evict）
       └─ server->forward_streaming_request()  // 轉發到後端子程序
```

### 第四層：WrappedServer 轉發
`src/cpp/server/wrapped_server.cpp`

```
WrappedServer::forward_streaming_request(endpoint, request_body, sink, sse, timeout)
  └─ utils::HttpClient::stream_post(
           url = "http://127.0.0.1:{port}/v1/chat/completions",
           body = request_body,
           callback = [&sink](chunk) { sink.write(chunk) }
       )
       // http_client.cpp — libcurl chunked transfer
```

### 第五層：後端子程序（以 LlamaCppServer 為例）
`src/cpp/server/backends/llamacpp_server.cpp`

```
llama-server process（獨立子程序）
  └─ 監聽 http://127.0.0.1:{random_port}/v1/chat/completions
  └─ 接收請求，推論，以 SSE 格式串流回傳
  └─ lemond 的 HttpClient 逐 chunk 轉發給原始客戶端
```

## Model 載入流程
`src/cpp/server/router.cpp — Router::load_model()`

```
load_model(model_name, model_info, options, do_not_upgrade)
  ├─ resolve_model_name()          // 公開名稱 → canonical
  ├─ RecipeOptions 三層合併:
  │    call options > model_info.recipe_options > config defaults
  ├─ std::unique_lock<load_mutex_>
  ├─ 等待其他載入完成 (load_cv_.wait)
  ├─ is_loading_ = true
  ├─ NPU 互斥檢查：
  │    ├─ ryzenai-llm / whispercpp → evict_all_npu_servers()
  │    └─ flm → 允許共存（max 1 LLM + 1 audio + 1 embed）
  ├─ LRU 容量檢查：
  │    └─ count_servers_by_type(type) >= max_loaded_models → evict_lru_server()
  ├─ create_backend_server(model_info) → unique_ptr<WrappedServer>
  ├─ lock.unlock()                 // 釋放鎖（允許其他請求繼續）
  ├─ new_server->load(...)         // 啟動子程序（30~60 秒）
  │    └─ 失敗時：
  │         ├─ file not found → 直接拋出
  │         └─ 其他錯誤 → evict_all_servers() + retry（nuclear option）
  ├─ lock.lock()
  ├─ loaded_servers_.push_back(new_server)
  └─ is_loading_ = false; load_cv_.notify_all()
```

## Model 下載流程
`src/cpp/server/model_manager.cpp`

```
ModelManager::download_model(model_name, model_data, ...)
  ├─ resolve_model_name()          // canonical 名稱
  ├─ 若為 user. 前綴 → register_user_model()
  ├─ 取得 ModelInfo（來自 models_cache_）
  └─ download_registered_model(info, ...)
       ├─ 若 recipe == "flm" → download_from_flm()
       │    └─ 呼叫 `flm download MODEL`（subprocess）
       └─ 其他 → download_from_huggingface()
            └─ libcurl 下載，SSE 回報進度
```

## 資料轉換層次

| 層次 | 角色 | 轉換內容 |
|------|------|---------|
| HTTP 接收 | `Server` | req.body → nlohmann::json |
| 請求前處理 | `Server` | enable_thinking → /no_think prepend |
| 路由分派 | `Router` | model 名稱解析、server 選擇、busy 標記 |
| Proxy 轉發 | `WrappedServer` | JSON → HTTP body → libcurl → chunked SSE |
| 後端回應 | `llama-server` etc. | 模型輸出 → OpenAI SSE 格式 |
| 遙測收集 | `WrappedServer::set_telemetry()` | token 計數、首 token 時間、TPS |

## Ollama 相容路由流程
`src/cpp/server/ollama_api.cpp`

```
OllamaApi::register_routes(web_server)
  └─ 掛載 /api/chat, /api/generate, /api/tags, /api/show, /api/pull, ...
  └─ 內部全部呼叫 router_→ 同一套推論邏輯
  └─ 格式轉換：Ollama JSON ↔ OpenAI JSON
```

## Anthropic API 流程
`src/cpp/server/anthropic_api.cpp`（透過 `POST /api/messages`）

```
AnthropicApi
  └─ 格式轉換：Anthropic messages → OpenAI messages 格式
  └─ 呼叫 router_.chat_completion() / chat_completion_stream()
  └─ 格式轉換：OpenAI response → Anthropic response 格式
```
