# Stage 1 偵察報告

## 專案基本資訊

**專案名稱**: Lemonade  
**版本**: 10.2.0 (CMakeLists.txt)  
**授權**: Apache 2.0  
**維護**: 社群 + AMD 贊助，主要維護者 10+ 名  
**官方網站**: https://lemonade-server.ai  
**Discord**: https://discord.gg/5xXzkMu8Zk  

## 一句話描述

Lemonade 是一個本地 AI 伺服器，提供與雲端 API 相同的多模態能力（文字生成、語音、圖像），透過 AMD NPU/GPU 硬體加速完全本地私密執行。

## 技術棧

| 類別 | 技術 | 版本/說明 |
|------|------|-----------|
| 主要語言 | C++17 | 伺服器主體 |
| 前端框架 | React 19 + TypeScript | Tauri app 與 Web app |
| 桌面框架 | Tauri v2 (Rust) | 跨平台桌面 GUI |
| HTTP 框架 | cpp-httplib | REST API 伺服器 |
| JSON 函式庫 | nlohmann/json 3.11.3+ | C++ JSON 處理 |
| CLI 框架 | CLI11 2.4.2+ | C++ CLI 解析 |
| 建置系統 | CMake 3.12+ | 搭配 Ninja/Visual Studio |
| LLM 後端 | llama.cpp | GPU 推論 (Vulkan/ROCm/Metal/CPU) |
| NPU 後端 1 | FastFlowLM (flm) | AMD XDNA2 NPU |
| NPU 後端 2 | RyzenAI-LLM | AMD NPU (Windows) |
| 語音後端 | whisper.cpp | ASR 轉錄 |
| 圖像後端 | stable-diffusion.cpp | 圖像生成 |
| TTS 後端 | Kokoro | 文字轉語音 |
| HTTP 客戶端 | libcurl 8.5.0+ | 下載模型 |
| 壓縮 | zstd 1.5.5+ | 資產壓縮 |
| WebSocket | libwebsockets 4.3.3+ | Realtime API |
| SSL | Schannel (Win) / SecureTransport (Mac) / OpenSSL (Linux) |
| 套件管理 (JS) | npm + Webpack 5 | Web app 建置 |
| Rust 套件 | Cargo | Tauri 相依性 |
| 容器化 | Docker | 附 Dockerfile |
| CI/CD | GitHub Actions | 多個 workflow |
| 測試 | Python pytest-style | 整合測試 |
| 文件 | MkDocs + markdown | docs/ 目錄 |

## 目錄結構（3 層深度）

```
lemonade/
├── .devcontainer/          # VS Code devcontainer 設定
├── .github/
│   ├── ISSUE_TEMPLATE/
│   ├── actions/
│   └── workflows/          # 14 個 CI/CD workflow
├── .signpath/              # 程式碼簽署政策
├── contrib/
│   └── debian/             # Debian 打包設定
├── data/                   # 靜態資料資產
├── docs/                   # 官方文件（mkdocs 驅動）
│   ├── api/                # API 規格文件
│   ├── guide/              # 使用者指南
│   ├── man/man1/           # man page（lemond, lemonade, lemonade-server）
│   ├── news/               # 版本新聞
│   └── server/             # 伺服器設定文件
├── examples/               # 使用範例（realtime_transcription.py 等）
├── src/
│   ├── app/                # Tauri 桌面 app
│   │   ├── assets/         # 圖示、靜態資源
│   │   ├── src/renderer/   # React 元件（ChatWindow, ModelManager 等）
│   │   └── src-tauri/      # Rust Tauri 後端
│   ├── cpp/                # C++ 伺服器主體
│   │   ├── cli/            # lemonade CLI 工具原始碼
│   │   ├── include/lemon/  # 標頭檔（router, wrapped_server 等）
│   │   ├── legacy-cli/     # 向下相容 shim
│   │   ├── resources/      # server_models.json, backend_versions.json
│   │   ├── server/         # HTTP 伺服器實作（server.cpp, router.cpp 等）
│   │   └── tray/           # 系統列托盤 app
│   └── web-app/            # 瀏覽器版 web app（獨立 webpack）
└── test/                   # Python 整合測試
    ├── cpp/                # C++ 單元測試
    └── utils/              # 測試工具
```

## 架構模式識別

**模式**: Plugin-based + Subprocess Model  
- 核心是 `lemond` HTTP 伺服器，透過 `Router` 管理多個 `WrappedServer` 實例
- 每個後端（llama.cpp、fastflowlm、whisper.cpp 等）以**獨立子程序**執行
- Lemonade 透過 HTTP proxy 轉發請求給這些子程序
- 前端（Tauri、Web app、CLI）透過 HTTP API 與 `lemond` 通訊
- **一對多客戶端架構**：單一 `lemond` 同時服務多個客戶端

## 既有文件摘要

### docs/dev-getting-started.md
完整的開發入門指南，包含 Prerequisites、建置步驟、架構概覽、測試方式。  
**路徑**: `docs/dev-getting-started.md`

### docs/server/configuration.md  
`config.json` 所有欄位說明，後端設定細節，API Key 認證層次。  
**路徑**: `docs/server/configuration.md`

### docs/server/server_integration.md
應用整合指南：Standalone、App-Managed 兩種整合模式。  
**路徑**: `docs/server/server_integration.md`

### AGENTS.md / CLAUDE.md  
給 AI agent 看的完整架構說明，含 Critical Invariants。  
**路徑**: `AGENTS.md`

### DESIGN.md  
前端設計系統規格（glassmorphism 設計語言、CSS tokens）。  
**路徑**: `DESIGN.md`

## 落差分析

| 既有文件描述 | 程式碼實際情況 | 位置 |
|-------------|---------------|------|
| AGENTS.md 說 `lemonade-server` 是向下相容 shim | 確實存在 `src/cpp/legacy-cli/` 目錄 | `src/cpp/legacy-cli/` |
| AGENTS.md 說 WebSocket port 9000+ | 程式碼確認為 OS 指派，從 9000 起 | `src/cpp/include/lemon/websocket_server.h` |
| docs/ 說 server_spec 已移到 api/README.md | `docs/server/server_spec.md` 只有一行轉址 | `docs/server/server_spec.md:1` |
| 文件說支援 `images/upscale` | server.cpp 確有 register_post("images/upscale") | `src/cpp/server/server.cpp:430` |

## 統計資訊

- 總檔案數: ~208（3 層深度 find）
- 主要 C++ 原始碼行數：
  - server.cpp: 4035 行
  - model_manager.cpp: 3312 行
  - router.cpp: 768 行
- 後端 header 數: 6 個（llama, fastflowlm, ryzenai, whisper, sd, kokoro）
- API endpoint 數: ~20 個路徑，每個在 4 個 prefix 下各一份
- 模型資料庫: server_models.json（100+ 個 recipe）
