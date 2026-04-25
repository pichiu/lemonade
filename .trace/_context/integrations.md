# Stage 2.5: 外部整合分析

## 推論後端子程序

這是最核心的外部整合——每個 AI 後端作為子程序執行。

| 後端 | 子程序執行檔 | 通訊方式 | 下載來源 |
|------|------------|---------|---------|
| llama.cpp | `llama-server` | HTTP localhost | GitHub Releases |
| FastFlowLM | `flm` | HTTP localhost | AMD FLM 發布渠道 |
| RyzenAI | `ryzenai-server` | HTTP localhost | AMD 發布渠道 |
| whisper.cpp | `whisper-server` | HTTP localhost | GitHub Releases |
| stable-diffusion.cpp | `sd-server` | HTTP localhost | GitHub Releases |
| Kokoro TTS | `koko` | HTTP localhost | GitHub Releases |

**版本管理**: `src/cpp/resources/backend_versions.json` 釘選每個後端的版本號

**下載機制**: `BackendManager` 負責下載/驗證/安裝後端可執行檔
- `src/cpp/server/backend_manager.cpp`
- 驗證 `version.txt` 與 `backend_versions.json` 比對

**程序管理**: `ProcessHandle` 結構體（`src/cpp/include/lemon/utils/process_manager.h`）
- 跨平台程序啟動（Windows: CreateProcess, Linux/macOS: fork+exec）
- `WrappedServer::is_process_running()` 監控子程序狀態

## Hugging Face Hub 整合

**位置**: `src/cpp/server/model_manager.cpp`

```
download_from_huggingface(model_info, progress_callback)
  └─ libcurl HTTP 下載
  └─ HF Hub API 取得 file list
  └─ 支援斷點續傳（Range header）
  └─ 儲存至 HF_HUB_CACHE 目錄
```

配置（優先順序）：
1. `models_dir` config.json 設定
2. `HF_HUB_CACHE` 環境變數
3. `HF_HOME` 環境變數
4. 平台預設（`~/.cache/huggingface/`）

下載進度透過 SSE 回報給客戶端（`DownloadProgress` struct，`model_manager.h:38`）

## libcurl HTTP 客戶端

**位置**: `src/cpp/include/lemon/utils/http_client.h`、`src/cpp/server/utils/http_client.cpp`

`utils::HttpClient` 是對 libcurl 的封裝，用於：
- 轉發請求到後端子程序（`HttpClient::post()` / `stream_post()`）
- 下載模型檔案（`HttpClient::download()`）
- 後端健康檢查（`HttpClient::get()`）

全域 timeout 設定：`utils::HttpClient::set_default_timeout(config->global_timeout())`

**失敗處理**：
- 後端健康檢查超時（`wait_for_ready()` 最多 600 秒）
- 連線失敗由 `forward_request()` 拋出 exception，Router 捕捉後決定是否 nuclear evict

## libwebsockets（WebSocket Realtime API）

**位置**: `src/cpp/server/websocket_server.cpp`

用於 OpenAI-compatible Realtime API（實時語音轉錄）：
- 綁定 OS 指派 port（9000+）
- VAD（Voice Activity Detection）—— `src/cpp/server/vad.cpp`
- `realtime_session.cpp` 管理每個 WebSocket session 的音訊緩衝
- `streaming_audio_buffer.cpp` 緩衝音訊資料

## OpenAI API 相容客戶端

Lemonade 提供的 API 與 OpenAI API 相容，因此所有 OpenAI 官方/相容客戶端可直接使用：
- base_url: `http://localhost:13305/v1`（或 `/api/v1`）
- api_key: 任意字串（除非設定 `LEMONADE_API_KEY`）

已驗證相容的客戶端：openai-python, openai-node, claude-code CLI

## Anthropic API 相容

**位置**: `src/cpp/server/anthropic_api.cpp`

- Endpoint: `POST /api/messages`
- 支援：message completion、tool use、SSE streaming
- 格式轉換：Anthropic Messages API ↔ OpenAI Chat Completions

## Ollama API 相容

**位置**: `src/cpp/server/ollama_api.cpp`

- `/api/chat`, `/api/generate`, `/api/tags`, `/api/show`
- `/api/delete`, `/api/pull`, `/api/embed`, `/api/embeddings`
- `/api/ps`, `/api/version`
- 保留 Ollama 的格式（不加 `/v1` prefix）

## CI/CD 整合（GitHub Actions）

**位置**: `.github/workflows/`

主要 workflow：
| Workflow | 觸發 | 功能 |
|----------|------|------|
| `cpp_server_build_test_release.yml` | push/PR | 建置 Windows/Linux/macOS，跑整合測試 |
| `docs_and_style.yml` | push/PR | Black 格式化，Pylint，mkdocs 建置 |
| `build-and-push-container.yml` | push main | 建置並推送 Docker image |
| `launchpad-ppa.yml` | push | 發布 Ubuntu PPA |
| `linux_distro_builds.yml` | push | 多 Linux distro 建置測試 |
| `publish-website.yml` | release | 發布文件網站 |

## SignPath 程式碼簽署

**位置**: `.signpath/policies/lemonade/`

Windows 可執行檔透過 SignPath.io 簽署（由 SignPath Foundation 提供憑證）

## Docker 整合

**位置**: `Dockerfile`、`.dockerignore`

- 多階段建置
- 用於在容器化環境中跑 Lemonade Server

## Discord/社群通訊

**位置**: 外部（不在 codebase 中）  
- https://discord.gg/5xXzkMu8Zk
- 維護者 email: lemonade@amd.com

## 平台系統整合

### Windows
- `src/cpp/server/utils/wmi_helper.cpp` — WMI 查詢系統資訊（GPU 型號、NPU 可用性）
- Windows startup folder — 自動啟動 LemonadeServer.exe
- SUBSYSTEM:WINDOWS GUI — 無終端視窗

### Linux
- systemd service `lemond.service`
- `/var/lib/lemonade/.cache/lemonade/` 作為 cache dir
- `src/cpp/server/system_info.cpp` — DRM/ioctl 查詢 AMD GPU 資訊
- `src/cpp/include/lemon/amdxdna_accel.h` — XDNA NPU 驅動介面

### macOS
- `.pkg` installer
- SecureTransport SSL
- Metal 後端（llama.cpp metal build）

## 失敗處理策略

| 整合 | 失敗情況 | 處理方式 |
|------|---------|---------|
| 後端子程序啟動失敗 | 非 file-not-found 錯誤 | nuclear option：evict all + retry |
| 後端子程序啟動失敗 | file-not-found | 直接拋出（不 evict） |
| 後端推論請求失敗 | HTTP 連線失敗 | 拋出 exception，返回 500 |
| HF 下載失敗 | 網路錯誤 | 拋出，SSE 回報錯誤 |
| Config 寫入失敗 | 磁碟錯誤 | `ConfigFile::save()` atomic write（temp+rename）|
