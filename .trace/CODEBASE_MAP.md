# Lemonade 程式碼地圖

## Annotated Directory Tree

```
lemonade/
│
├── CMakeLists.txt              ← 根建置設定（版本號、所有 target、依賴）
├── CMakePresets.json           ← 建置預設（default/windows/vs18/debug）
├── setup.sh / setup.ps1        ← 開發環境安裝腳本（執行前先跑這個）
├── AGENTS.md                   ← AI agent 指引（架構、Critical Invariants）
├── CLAUDE.md                   ← Claude Code 設定（@AGENTS.md）
├── DESIGN.md                   ← 前端設計系統規格（glassmorphism）
├── mkdocs.yml                  ← 文件網站建置設定
├── Dockerfile                  ← Docker container 建置
├── .pre-commit-config.yaml     ← 格式化 hooks（trailing-whitespace, check-yaml 等）
│
├── docs/                       ← 官方使用者文件（發布至 lemonade-server.ai）
│   ├── api/                    ← API 規格（OpenAI/Ollama/Anthropic endpoints）
│   ├── guide/                  ← 使用者指南
│   ├── server/                 ← 伺服器設定、整合文件
│   │   ├── configuration.md    ← config.json 完整參考
│   │   ├── server_integration.md ← 應用整合指南
│   │   └── apps/               ← 各 app 整合教學（Claude Code, Open WebUI 等）
│   ├── embeddable/             ← Embeddable Lemonade 指南
│   ├── man/man1/               ← man pages（lemond, lemonade, lemonade-server）
│   ├── lemonade-cli.md         ← CLI 指令完整參考
│   ├── dev-getting-started.md  ← 開發者上手指南
│   └── faq.md                  ← 常見問題
│
├── src/
│   │
│   ├── cpp/                    ← 全部 C++ 程式碼
│   │   │
│   │   ├── server/             ← lemond HTTP 伺服器實作
│   │   │   ├── main.cpp        ← lemond 入口（signal handling, Server 初始化）
│   │   │   ├── server.cpp      ← HTTP 路由、所有 handle_*() 函式（4035行）
│   │   │   ├── router.cpp      ← 多模型排程、LRU eviction、NPU 互斥（768行）
│   │   │   ├── model_manager.cpp ← 模型 registry、下載、路徑解析（3312行）
│   │   │   ├── anthropic_api.cpp ← Anthropic API 格式轉換
│   │   │   ├── ollama_api.cpp  ← Ollama API 相容層
│   │   │   ├── websocket_server.cpp ← Realtime WebSocket API
│   │   │   ├── realtime_session.cpp ← WebSocket session 管理
│   │   │   ├── config_file.cpp ← config.json 讀寫（atomic write）
│   │   │   ├── runtime_config.cpp ← RuntimeConfig 全域設定
│   │   │   ├── backend_manager.cpp ← 後端 binary 下載/版本管理
│   │   │   ├── system_info.cpp ← 硬體探測（GPU/NPU/CPU）
│   │   │   ├── recipe_options.cpp ← RecipeOptions 三層合併
│   │   │   ├── streaming_proxy.cpp ← HTTP 串流 proxy
│   │   │   ├── vad.cpp         ← Voice Activity Detection
│   │   │   ├── wrapped_server.cpp ← WrappedServer 基底實作
│   │   │   └── backends/       ← 各後端實作
│   │   │       ├── llamacpp_server.cpp    ← llama.cpp 後端
│   │   │       ├── fastflowlm_server.cpp  ← FLM NPU 後端
│   │   │       ├── ryzenaiserver.cpp      ← RyzenAI NPU 後端
│   │   │       ├── whisper_server.cpp     ← whisper.cpp ASR 後端
│   │   │       ├── sd_server.cpp          ← stable-diffusion.cpp 後端
│   │   │       ├── kokoro_server.cpp      ← Kokoro TTS 後端
│   │   │       └── backend_utils.cpp      ← BackendSpec 共用工具
│   │   │
│   │   ├── include/lemon/      ← 公開 C++ header 介面
│   │   │   ├── wrapped_server.h        ← 後端抽象基底類別
│   │   │   ├── server_capabilities.h   ← ICompletionServer 等 capability 介面
│   │   │   ├── router.h                ← Router 類別宣告
│   │   │   ├── server.h                ← Server 類別宣告
│   │   │   ├── model_manager.h         ← ModelManager + ModelInfo 宣告
│   │   │   ├── model_types.h           ← ModelType/DeviceType enum
│   │   │   ├── recipe_options.h        ← RecipeOptions 類別
│   │   │   ├── config_file.h           ← ConfigFile 讀寫介面
│   │   │   ├── backend_manager.h       ← BackendManager 宣告
│   │   │   ├── runtime_config.h        ← RuntimeConfig 全域設定
│   │   │   └── utils/                  ← 工具類
│   │   │       ├── http_client.h       ← libcurl 封裝
│   │   │       ├── process_manager.h   ← 跨平台子程序啟動/管理
│   │   │       ├── path_utils.h        ← 跨平台路徑工具
│   │   │       ├── network_beacon.h    ← UDP 廣播（伺服器發現）
│   │   │       └── aixlog.hpp          ← 日誌系統
│   │   │
│   │   ├── cli/                ← lemonade CLI 客戶端
│   │   │   ├── main.cpp        ← CLI 入口（CLI11 app，所有子命令）
│   │   │   ├── lemonade_client.cpp ← HTTP 呼叫 lemond API
│   │   │   ├── agent_launcher.cpp  ← 啟動 claude/codex/opencode agent
│   │   │   ├── model_selection.cpp ← 互動式模型選擇
│   │   │   └── hf_pull.cpp     ← HuggingFace 模型匯入輔助
│   │   │
│   │   ├── tray/               ← lemonade-tray（系統列客戶端）
│   │   │   ├── main.cpp        ← tray app 入口
│   │   │   ├── tray_app.cpp    ← 系統列 UI 邏輯
│   │   │   └── platform/       ← 平台特定程式碼
│   │   │
│   │   ├── legacy-cli/         ← lemonade-server shim（向下相容）
│   │   │   └── main.cpp        ← 委派給 lemond 或 lemonade
│   │   │
│   │   ├── resources/          ← 嵌入資源
│   │   │   ├── server_models.json    ← 模型 registry（165 個模型）
│   │   │   └── backend_versions.json ← 後端版本釘選
│   │   │
│   │   └── installer/          ← WiX 安裝程式設定（Windows）
│   │
│   ├── app/                    ← Tauri 桌面 app
│   │   ├── src/renderer/       ← React UI 元件
│   │   │   ├── App.tsx         ← 根元件
│   │   │   ├── ChatWindow.tsx  ← 聊天介面
│   │   │   ├── ModelManager.tsx ← 模型管理 UI
│   │   │   ├── DownloadManager.tsx ← 下載進度 UI
│   │   │   ├── BackendManager.tsx ← 後端管理 UI
│   │   │   ├── SettingsPanel.tsx ← 設定面板
│   │   │   └── tauriShim.ts    ← window.api Tauri 橋接器
│   │   └── src-tauri/          ← Rust Tauri 後端
│   │
│   └── web-app/                ← 瀏覽器版 web app（Debian 打包用）
│       └── package.json        ← 獨立依賴（不合併至 app/package.json）
│
├── test/                       ← Python 整合測試
│   ├── server_cli.py           ← CLI 測試（不需推論）
│   ├── server_endpoints.py     ← Endpoint 測試（不需推論）
│   ├── server_llm.py           ← LLM 推論測試
│   ├── server_whisper.py       ← ASR 測試
│   ├── server_sd.py            ← 圖像生成測試
│   └── utils/                  ← 測試工具（server_base.py）
│
├── examples/                   ← 使用範例
│   └── realtime_transcription.py ← WebSocket Realtime API 範例
│
└── contrib/                    ← 套件打包設定
    └── debian/                 ← Debian/Ubuntu 打包設定
```

## 「我想改 X 要看哪裡？」速查表

| 我想要... | 看這裡 | 關鍵檔案 |
|----------|--------|---------|
| 新增一個 API endpoint | `src/cpp/server/server.cpp` | `setup_routes()` 函式 + 對應 `handle_*()` |
| 新增一個 AI 後端 | `src/cpp/server/backends/` + `router.cpp` | 新建 `*_server.cpp/.h`，加入 `create_backend_server()` |
| 新增/修改內建模型 | `src/cpp/resources/` | `server_models.json` |
| 修改後端版本 | `src/cpp/resources/` | `backend_versions.json` |
| 修改伺服器預設設定 | `src/cpp/server/config_file.cpp` | `get_defaults()` 函式 |
| 修改 LRU/NPU 排程邏輯 | `src/cpp/server/router.cpp` | `load_model()`, NPU 互斥程式碼（約 270 行）|
| 修改 Ollama API | `src/cpp/server/ollama_api.cpp` | `OllamaApi::register_routes()` |
| 修改 Anthropic API | `src/cpp/server/anthropic_api.cpp` | `AnthropicApi` 類別 |
| 修改 WebSocket Realtime | `src/cpp/server/websocket_server.cpp` | `WebSocketServer` 類別 |
| 修改模型下載邏輯 | `src/cpp/server/model_manager.cpp` | `download_from_huggingface()`, `download_from_flm()` |
| 修改 CLI 命令 | `src/cpp/cli/main.cpp` | CLI11 app 定義 |
| 修改 React UI | `src/app/src/renderer/` | 對應元件 `.tsx` 檔案 |
| 修改 Tauri window.api | `src/app/src/renderer/tauriShim.ts` + `src-tauri/` | 前後端橋接 |
| 修改 Web App mock api | `src/cpp/server/server.cpp` | `serve_web_app_html` lambda，約 600 行處 |
| 修改硬體探測邏輯 | `src/cpp/server/system_info.cpp` | 系統資訊快取 |
| 修改建置設定 | `CMakeLists.txt` | 根目錄 |
| 修改 Windows 安裝程式 | `src/cpp/installer/` | WiX 設定 |
| 修改測試 | `test/` | 對應的 `server_*.py` |
| 修改文件 | `docs/` | 對應 `.md` 檔案 |

## 模組依賴關係圖

```mermaid
graph TD
    CLI[lemonade CLI<br/>src/cpp/cli/] --> |HTTP API| S
    TRAY[lemonade-tray<br/>src/cpp/tray/] --> |HTTP API| S
    APP[Tauri App<br/>src/app/] --> |window.api / HTTP| S
    WEBAPP[Web App<br/>src/web-app/] --> |HTTP| S

    subgraph lemond 伺服器
        M[main.cpp] --> S[Server<br/>server.cpp]
        S --> R[Router<br/>router.cpp]
        S --> MM[ModelManager<br/>model_manager.cpp]
        S --> BM[BackendManager<br/>backend_manager.cpp]
        S --> WS[WebSocketServer<br/>websocket_server.cpp]
        S --> OA[OllamaApi<br/>ollama_api.cpp]
        S --> AA[AnthropicApi<br/>anthropic_api.cpp]
        R --> WR[WrappedServer<br/>wrapped_server.h]
        WR --> LC[LlamaCppServer]
        WR --> FLM[FastFlowLMServer]
        WR --> RAI[RyzenAIServer]
        WR --> WH[WhisperServer]
        WR --> SD[SDServer]
        WR --> KK[KokoroServer]
    end

    LC --> |subprocess HTTP| llama[llama-server]
    FLM --> |subprocess HTTP| flmbin[flm]
    RAI --> |subprocess HTTP| ryzen[ryzenai-server]
    WH --> |subprocess HTTP| whisper[whisper-server]
    SD --> |subprocess HTTP| sdbin[sd-server]
    KK --> |subprocess HTTP| koko[koko]

    MM --> |download| HF[Hugging Face Hub]
    BM --> |download| GH[GitHub Releases]

    style lemond 伺服器 fill:#f9f,stroke:#333,stroke-width:2px
```

## 建置輸出物

| 輸出 | 平台 | 建置指令 |
|------|------|---------|
| `lemond` | Linux/macOS | `cmake --build --preset default` |
| `lemond.exe` | Windows | `cmake --build --preset windows` |
| `lemonade` | Linux/macOS | 同上 |
| `lemonade.exe` | Windows | 同上 |
| `LemonadeServer.exe` | Windows | 同上（SUBSYSTEM:WINDOWS）|
| `lemonade-tray` | Linux | 同上 |
| `lemonade-app` | Linux/macOS | `--target tauri-app` |
| `lemonade-app.exe` | Windows | `--target tauri-app` |
| `.msi` 安裝程式 | Windows | `--target wix_installer_minimal` |
| `.deb` 套件 | Linux | `cd build && cpack` |
| `.rpm` 套件 | Linux | `cd build && cpack -G RPM` |
| `.pkg` 安裝程式 | macOS | `--target package-macos` |
| `AppImage` | Linux | `--target appimage` |
