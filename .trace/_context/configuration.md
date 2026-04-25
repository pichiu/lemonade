# Stage 2.6: 設定與環境分析

## 設定載入優先順序

```
高優先 ─────────────────────────────────── 低優先
[lemond CLI 參數] > [config.json] > [legacy env vars] > [defaults]
```

### 詳細說明

1. **lemond CLI 參數**（`src/cpp/server/cli_parser.cpp`）
   - `--port PORT`、`--host HOST`
   - 只在 server 無法正常啟動時使用
   - **自動持久化**回 config.json（不需重起）

2. **config.json**（`src/cpp/server/config_file.cpp`）
   - 儲存位置（依平台）：
     - Linux (systemd): `/var/lib/lemonade/.cache/lemonade/config.json`
     - Linux (standalone): `~/.cache/lemonade/config.json`
     - Windows: `%USERPROFILE%\.cache\lemonade\config.json`
     - macOS: `/Library/Application Support/lemonade/.cache/config.json`
   - 不存在時自動以 defaults 建立
   - 使用 temp+rename atomic write（執行緒安全）
   - 保留 unknown keys（前向相容）

3. **Legacy 環境變數**（`src/cpp/server/config_file.cpp — migrate_from_env()`）
   - 僅當 config.json 不存在時讀取
   - 之後不再讀取（已持久化到 config.json）
   - 原 `LEMONADE_PORT`、`LEMONADE_HOST` 等

4. **Defaults**（`src/cpp/server/config_file.cpp — get_defaults()`）
   - 從內建資源讀取基礎 defaults
   - Linux 可有 `/usr/share/lemonade/defaults.json` 蓋過（distro 打包用）

## 執行時設定動態更新

透過 `lemonade config set key=value` 或 `POST /internal/set`：

```
Server::handle_config_set()
  └─ config_->set(changes)          // RuntimeConfig::set()
  └─ ConfigFile::save(cache_dir, config_json)  // 立即持久化
  └─ apply_config_side_effects(applied_changes) // 立即生效
       ├─ port/host 變更 → rebind HTTP servers
       ├─ log_level 變更 → AixLog 重設
       ├─ extra_models_dir 變更 → model_manager 更新
       └─ *_bin 變更 → 後端 binary 熱替換
```

**RuntimeConfig**（`src/cpp/include/lemon/runtime_config.h`）是全域設定單例，`RuntimeConfig::set_global()` 設定，子元件透過 `config_` 指標存取。

## 所有設定欄位

### 頂層設定

| Key | Type | Default | 說明 |
|-----|------|---------|------|
| `port` | int | 13305 | HTTP server port |
| `host` | string | "localhost" | 綁定地址（"0.0.0.0" = 所有介面）|
| `log_level` | string | "info" | trace/debug/info/warning/error/fatal/none |
| `global_timeout` | int | 300 | 所有 HTTP/推論/就緒 timeout（秒）|
| `max_loaded_models` | int | 1 | 每個 ModelType 的最大同時載入數，-1=無限 |
| `no_broadcast` | bool | false | 停用 UDP 廣播 |
| `extra_models_dir` | string | "" | 額外 GGUF 掃描目錄 |
| `models_dir` | string | "auto" | HF cache 目錄（auto=平台預設）|
| `ctx_size` | int | 4096 | 全域預設 LLM context 大小 |
| `offline` | bool | false | 跳過所有下載 |
| `no_fetch_executables` | bool | false | 不下載後端可執行檔 |
| `disable_model_filtering` | bool | false | 顯示所有模型（不依硬體過濾）|
| `enable_dgpu_gtt` | bool | false | 硬體過濾時納入 GTT 記憶體 |
| `websocket_port` | int | 0 | WebSocket port（0=OS指派）|

### Backend 設定

每個後端有獨立的設定區塊（`llamacpp`、`whispercpp`、`sdcpp`、`flm`、`ryzenai`、`kokoro`）。

`*_bin` 欄位值：
- `"builtin"` — 使用 lemonade 內建版本（測試過的穩定版）
- `"latest"` — 解析最新 GitHub release（實驗性）
- `"v1.8.2"` / `"b8664"` — 釘選特定版本
- `"/path/to/dir"` — 使用本地自訂 binary

## API Key 與安全性

| 環境變數 | 用途 | 覆蓋範圍 |
|---------|------|---------|
| `LEMONADE_API_KEY` | 一般 API 認證 | `/api/*`, `/v0/*`, `/v1/*` |
| `LEMONADE_ADMIN_API_KEY` | 管理員認證 | 以上 + `/internal/*` |

認證邏輯在 `Server::authenticate_request()`（`server.cpp`）：
- 未設定任何 key → 不需認證
- 僅設定 `LEMONADE_API_KEY` → 所有路由都需要此 key
- 僅設定 `LEMONADE_ADMIN_API_KEY` → 內部路由需要此 key，一般路由不需認證
- 兩者都設定 → 內部路由需要 admin key，一般路由接受任一 key

客戶端（CLI、tray、app）優先使用 `LEMONADE_ADMIN_API_KEY`，備用 `LEMONADE_API_KEY`。

## 模型相關設定檔案

| 檔案 | 位置 | 用途 |
|------|------|------|
| `server_models.json` | `src/cpp/resources/` | 內建模型 registry（100+ 個模型，版本控制）|
| `user_models.json` | cache dir | 用戶自訂/匯入模型（跨版本保留）|
| `recipe_options.json` | cache dir | 用戶儲存的每模型推論設定 |
| `backend_versions.json` | `src/cpp/resources/` | 後端版本釘選（版本控制）|

## 日誌設定

**日誌系統**: AixLog 函式庫（`src/cpp/include/lemon/utils/aixlog.hpp`）

日誌等級：trace > debug > info > warning > error > fatal > none

輸出目標（`src/cpp/server/logging_config.cpp`）：
- **Console stdout** — 直接伺服器模式
- **File log** — `~/.cache/lemonade/lemond.log`
- **WebSocket log hub** — 讓客戶端即時接收日誌（`GET /api/v1/logs/stream`）

`SIGHUP` 訊號觸發 `SystemInfoCache::invalidate_recipes()`（不重啟重新掃描硬體）。

## Feature Flag 類設定

| 設定 | 效果 | 使用情境 |
|------|------|---------|
| `disable_model_filtering=true` | 顯示所有模型 | 除錯、開發 |
| `offline=true` | 跳過所有網路請求 | 隔離網路環境 |
| `no_fetch_executables=true` | 不自動下載後端 | 自管 binary |
| `no_broadcast=true` | 不廣播 UDP | 多伺服器環境 |
| `enable_dgpu_gtt=true` | 納入 GTT 做 VRAM 計算 | dGPU + GTT 混合環境 |

## 前端設定（Per-Client）

根據 Critical Invariant #11，所有前端設定**不存在於 lemond**，存在各客戶端本地：

- **Tauri app**: `app_settings.json`（Tauri 本地存儲）
- **Web app**: `localStorage`（`lemonade-settings` key）
- **CLI**: 環境變數 + 命令列參數

設定內容包含：API URL、API Key、theme、layout（chat/model manager 比例）、zoom level。
