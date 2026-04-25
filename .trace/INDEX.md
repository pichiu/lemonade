# Lemonade 專案總覽與速查

## 一段話總結

Lemonade 是一個本地 AI 推論伺服器，讓用戶在自己的 AMD NPU/GPU 上完全免費、私密地執行 LLM、語音辨識、圖像生成與 TTS，對外暴露與 OpenAI、Anthropic、Ollama 相容的 REST API，讓任何 AI 應用無須修改即可切換至本地端。

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| 主要語言 | C++17 | C++17 標準 | 伺服器主體 (`src/cpp/`) |
| 前端框架 | React | 19.x | Tauri app UI (`src/app/src/`) |
| 前端語言 | TypeScript | 5.3 | 型別安全前端 |
| 桌面框架 | Tauri | v2 (Rust) | 跨平台桌面 GUI (`src/app/src-tauri/`) |
| HTTP 框架 | cpp-httplib | 0.26.0+ | REST API 伺服器 |
| JSON 函式庫 | nlohmann/json | 3.11.3+ | C++ JSON 處理 |
| CLI 框架 | CLI11 | 2.4.2+ | C++ CLI 解析 |
| 建置系統 | CMake | 3.12+ | 主建置入口 (`CMakeLists.txt`) |
| HTTP 客戶端 | libcurl | 8.5.0+ | 模型下載 / 後端 proxy |
| WebSocket | libwebsockets | 4.3.3+ | Realtime API |
| LLM 後端 | llama.cpp | b8766 (vulkan) | GPU 推論 |
| NPU 後端 | FastFlowLM (flm) | v0.9.39 | AMD XDNA2 NPU 推論 |
| NPU 後端 | RyzenAI-LLM | v1.7.0 | AMD NPU 混合推論 |
| ASR 後端 | whisper.cpp | v1.8.2 | 語音辨識 |
| 圖像後端 | stable-diffusion.cpp | master-569 | 圖像生成 |
| TTS 後端 | Kokoro | b16 | 文字轉語音 |
| 壓縮 | zstd | 1.5.5+ | 資產壓縮 |
| 打包 (JS) | Webpack | 5 | Web app 建置 |
| Rust 套件管理 | Cargo | — | Tauri 依賴 |
| 容器化 | Docker | — | `Dockerfile` |
| CI/CD | GitHub Actions | — | `.github/workflows/` |
| 測試 | Python | 3.x | `test/` 整合測試 |
| 文件 | MkDocs | — | `docs/` + `mkdocs.yml` |
| 程式碼簽署 | SignPath.io | — | Windows 可執行檔 |

## 關鍵指令速查

### 建置
```bash
# 安裝依賴並設定建置目錄
./setup.sh                              # Linux / macOS
./setup.ps1                             # Windows

# 建置 C++ 伺服器
cmake --build --preset default          # Linux / macOS
cmake --build --preset windows          # Windows

# 建置含 Tauri 桌面 app
cmake --build --preset default --target tauri-app

# 建置 Web app
cmake --build --preset default --target web-app

# 建置 Linux .deb
cd build && cpack

# 建置 macOS .pkg
cmake --build --preset default --target package-macos
```

### 測試
```bash
pip install -r test/requirements.txt

# CLI 測試（不需推論後端）
python test/server_cli.py

# Endpoint 測試（不需推論後端）
python test/server_endpoints.py

# LLM 推論測試
python test/server_llm.py --wrapped-server llamacpp --backend vulkan

# 語音辨識測試
python test/server_whisper.py

# 圖像生成測試
python test/server_sd.py
```

### 執行
```bash
# 啟動伺服器
lemond

# 使用 CLI
lemonade list                           # 列出模型
lemonade pull Gemma-4-E2B-it-GGUF      # 下載模型
lemonade run Gemma-4-E2B-it-GGUF       # 執行 chat
lemonade status                         # 查詢伺服器狀態
lemonade backends                       # 查看後端狀態
lemonade launch claude                  # 啟動 Claude Code agent
lemonade config set port=13305          # 修改設定
```

### 格式化 / Lint
```bash
black test/                             # Python 格式化（版本 26.1.0）
pre-commit run --all-files              # 執行所有 pre-commit hooks
```

## 文件地圖

| 文件 | 內容 |
|------|------|
| [INDEX.md](./INDEX.md) | 本文件：總覽、技術棧、指令速查 |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 系統架構、元件圖、設計決策 |
| [DATA_MODEL.md](./DATA_MODEL.md) | 資料模型、ModelInfo、config 結構 |
| [API_SURFACE.md](./API_SURFACE.md) | 所有 API endpoint、request/response 範例 |
| [CODEBASE_MAP.md](./CODEBASE_MAP.md) | 目錄結構、「我想改 X 去哪裡」速查表 |
| [DEV_GUIDE.md](./DEV_GUIDE.md) | 開發者上手、環境建置、測試策略 |
| [DISCOVERY_LOG.md](./DISCOVERY_LOG.md) | 探索紀錄、技術債、待解疑問 |

中繼檔案（AI 分析用）：`.trace/_context/`

## 專案專屬術語表

| 術語 | 定義 |
|------|------|
| **lemond** | 核心 HTTP 伺服器可執行檔，純推論服務，不含 UI |
| **lemonade** | CLI 客戶端工具，與 lemond 通訊 |
| **LemonadeServer.exe** | Windows GUI 應用，內嵌 lemond + 系統列圖示 |
| **lemonade-tray** | macOS/Linux 系統列客戶端（不內嵌 lemond）|
| **lemonade-app** | Tauri 桌面 app（隨需開啟，不是伺服器）|
| **WrappedServer** | 後端抽象基底類別，封裝後端子程序的生命週期 |
| **Router** | 管理多個 WrappedServer，做 LRU 排程和 NPU 互斥 |
| **ModelManager** | 模型 registry 管理，下載，路徑解析 |
| **BackendManager** | 後端可執行檔的下載和版本管理 |
| **Recipe** | 模型 + 後端的配對方式（如 `llamacpp`, `flm`, `ryzenai-llm`）|
| **RecipeOptions** | 推論時的設定參數（ctx_size, gpu_layers, backend, args）|
| **ModelType** | 模型用途分類（LLM, EMBEDDING, AUDIO, IMAGE, TTS, RERANKING）|
| **DeviceType** | 硬體設備旗標（CPU \| GPU \| NPU，bitmask）|
| **NPU exclusivity** | RyzenAI/whispercpp 獨佔 NPU；FLM 分槽共享 |
| **Nuclear option** | 後端載入失敗時清空所有模型並重試的策略 |
| **Quad-prefix** | 每個 endpoint 必須在 /api/v0/, /api/v1/, /v0/, /v1/ 各一份 |
| **UDP beacon** | lemond 廣播自身存在，CLI/tray 透過此發現伺服器 |
| **FLM** | FastFlowLM，AMD 的 NPU 推論框架 |
| **XDNA2** | AMD Ryzen AI NPU 硬體架構 |
| **Embeddable** | 可打包進第三方 app 的 Lemonade binary 版本 |
| **Collection** | 由多個子模型組成的 recipe（experience recipes）|
| **user. prefix** | 用戶自訂模型的命名慣例（如 `user.MyModel`）|
