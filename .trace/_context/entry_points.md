# Stage 2.1: Entry Points 分析

## 可執行檔入口總覽

### 1. lemond（核心伺服器）
**入口**: `src/cpp/server/main.cpp:50 — int main()`

啟動流程：
```
main()
  └─ CLIParser::parse()              // 解析 --port, --host, cache_dir
  └─ ConfigFile::load(cache_dir)     // 載入 config.json，merge defaults
  └─ RuntimeConfig::set_global()     // 全域設定單例
  └─ configure_application_logging() // 設定日誌等級
  └─ Server::Server(config, cache_dir) // 建構伺服器
       ├─ ModelManager::new          // 模型管理器
       ├─ BackendManager::new        // 後端管理器
       ├─ Router::new(config, mm, bm) // 請求路由器
       ├─ setup_http_servers()       // 建立 IPv4/IPv6 httplib::Server，8 thread pool
       │    └─ setup_routes()        // 註冊所有 REST 路由（quad-prefix）
       │    └─ OllamaApi::register_routes() // Ollama 相容路由
       ├─ WebSocketServer::new()     // Realtime API (port 9000+)
       └─ NetworkBeacon 初始化       // UDP 廣播
  └─ server.run()                    // 開始監聽（阻塞）
  └─ signal_handler → _exit(0)      // SIGINT/SIGTERM 處理
```

關鍵初始化細節：
- `ConfigFile::load()` 位於 `src/cpp/server/config_file.cpp`
  - 不存在時從 defaults 建立，保留 unknown keys（前向相容）
  - `config_file.h:47` — 使用 shared_mutex 保護讀寫
- `setup_http_servers()` 位於 `src/cpp/server/server.cpp:170`
  - 建立兩個 `httplib::Server`（IPv4 + IPv6），各有 8 thread 的 ThreadPool
  - `server.h:11` — `CPPHTTPLIB_THREAD_POOL_COUNT 8`
- `setup_routes()` 位於 `server.cpp:255`
  - 使用 `register_get` / `register_post` lambda，每個 endpoint 自動在 4 個 prefix 下各一份
  - 最後呼叫 `OllamaApi::register_routes(web_server)` 掛載 Ollama 相容路由

### 2. lemonade（CLI 客戶端）
**入口**: `src/cpp/cli/main.cpp:?` (CLI11 app，非 `int main()` pattern)

CLI 命令結構（`CliConfig` struct 定義在 `cli/main.cpp`）：
- `list` — 列出可用/已下載模型
- `pull MODEL` — 下載模型（呼叫 `/api/v1/pull`）
- `delete MODEL` — 刪除模型（呼叫 `/api/v1/delete`）
- `run MODEL` — 載入並執行 chat（呼叫 `/api/v1/load` + `/api/v1/chat/completions`）
- `status` — 查詢伺服器狀態（UDP beacon 發現 + `/api/v1/health`）
- `logs` — 查看日誌
- `launch AGENT` — 啟動 AI agent（claude/codex/opencode）
- `backends` — 管理後端執行檔
- `scan` — 掃描系統 NPU/GPU
- `config` — 設定管理（`/internal/set`, `/internal/config`）
- `recipes` — 列出 recipe 設定

客戶端連線發現機制（`src/cpp/include/lemon/utils/network_beacon.h`）：
- 監聽 UDP 廣播 beacon，自動發現執行中的 lemond
- 備用：直接連 `--host/--port` 或環境變數

### 3. LemonadeServer.exe（Windows GUI）
- SUBSYSTEM:WINDOWS 應用，嵌入 lemond + 系統列圖示
- 透過 Windows startup folder 自動啟動
- **不建議**用 lemonade-app.exe 取代此元件管理生命週期

### 4. lemonade-tray（macOS/Linux）
**入口**: `src/cpp/tray/main.cpp`
- 輕量客戶端，連接執行中的 lemond（不內嵌 lemond）
- 平台程式碼在 `src/cpp/tray/platform/`

### 5. Tauri 桌面 App（lemonade-app）
**入口**: `src/app/src-tauri/` (Rust)
- 隨需開啟，透過 UDP beacon 或 explicit base URL 發現 lemond
- `window.api` 合約由 `src/app/src/renderer/tauriShim.ts` 實作

## 信號處理
`src/cpp/server/main.cpp:24-47`

- `SIGINT/SIGTERM` → `_exit(0)`（立即終止，OS 清理子程序）
- `SIGHUP` → 設定 `g_reload_requested` flag → 背景執行緒呼叫 `SystemInfoCache::invalidate_recipes()` → 重新掃描硬體

## WebSocket 伺服器入口
`src/cpp/server/websocket_server.cpp:25`

- 在 `Server::Server()` 建構時初始化
- 綁定 OS 指派的 port（從 9000 起）
- 透過 `/health` 的 `websocket_port` 欄位揭露給客戶端

## Anthropic API 入口
`src/cpp/server/anthropic_api.cpp`（透過 `server.cpp` 中 `POST /api/messages` 路由）
