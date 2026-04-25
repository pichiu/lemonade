# Stage 1 Web 搜尋發現

## 搜尋結果彙整

### 1. 官方文件
- **Lemonade Server Documentation**: https://lemonade-server.ai/docs/  
  完整文件入口，包含 API、指南、設定
- **Server Spec**: https://lemonade-server.ai/docs/server/server_spec/  
  官方 API 規格，已整合至 docs/api/
- **Developer Getting Started**: https://lemonade-server.ai/docs/dev-getting-started/  
  與 repo 中的 docs/dev-getting-started.md 相同
- **Server Configuration**: https://lemonade-server.ai/docs/server/configuration/  
  完整設定文件

### 2. AMD 官方技術文章
- **Lemonade by AMD: A Unified API for Local AI Developers** (2026)  
  https://www.amd.com/en/developer/resources/technical-articles/2026/lemonade-for-local-ai.html  
  Key takeaway: AMD 視 Lemonade 為統一的本地 AI API 層，讓開發者不需感知底層硬體差異

### 3. 社群討論
- **Hacker News: lemonade-sdk/lemonade** (GPU+NPU 本地 LLM 伺服器)  
  https://news.ycombinator.com/item?id=47612724  
  Key takeaway: 社群對 AMD NPU 支援與 OpenAI 相容性反應正面
- **DEV Community: AMD's Lemonade Just Made Every Nvidia-Only AI Guide Obsolete**  
  https://dev.to/max_quimby/amds-lemonade-just-made-every-nvidia-only-ai-guide-obsolete-2a3l  
  Key takeaway: AMD GPU/NPU 用戶的重要工具，填補 Nvidia-only 生態系的空缺

### 4. 平台新聞
- **Phoronix: Lemonade 10.0.1 Improves Setup Process For Using AMD Ryzen AI NPUs On Linux**  
  https://www.phoronix.com/news/Lemonade-10.0.1  
  Key takeaway: Linux NPU 支援持續改善，重點在安裝流程

### 5. NPU 專項資料
- **LLMs on Linux with FastFlowLM**  
  https://lemonade-server.ai/flm_npu_linux.html  
  Key takeaway: FLM 可在 Linux 上做 NPU 推論，但 Windows 支援更完整
- **gaia project: Add streaming transcription via Lemonade WebSocket realtime API**  
  https://github.com/amd/gaia/issues/372  
  Key takeaway: AMD 生態系其他專案（GAIA）已整合 Lemonade WebSocket Realtime API

### 6. DeepWiki
- **lemonade-sdk/lemonade | DeepWiki**  
  https://deepwiki.com/lemonade-sdk/lemonade  
  Key takeaway: 第三方 AI 生成的架構分析，可用於交叉驗證

## 關鍵技術 Takeaways

### FastFlowLM (FLM) NPU 調度機制
- FLM 使用 AMD 特有的 NPU 調度器，可同時執行最多 3 個 NPU 程序：
  - 1 個 LLM 程序
  - 1 個 audio (STT) 程序
  - 1 個 embedding 程序
- 這與 RyzenAI-LLM 不同（ryzenai-llm 獨佔整個 NPU）
- 相關程式碼: `src/cpp/server/router.cpp` NPU 互斥邏輯

### RyzenAI 混合推論架構
- Ryzen AI 300/400 系列：prompt processing 由 NPU 執行，token generation 由 iGPU/dGPU 執行
- 這是 llama.cpp 後端與 ryzenai-llm 後端的核心差異

### Embeddable Lemonade
- 這是最近新增的功能（已完成，在 roadmap 標為 Recently Completed）
- 允許第三方 app 把 Lemonade 打包進去，不需用戶單獨安裝
- 相關文件: `docs/embeddable/README.md`

### WebSocket Realtime API
- OpenAI 相容的 Realtime protocol
- 僅支援 16kHz mono PCM16 音訊格式
- Port 由 OS 指派（9000+），透過 `/health` 的 `websocket_port` 欄位揭露
- 包含 VAD (Voice Activity Detection)

### API 相容性策略
- **OpenAI**: 主要相容目標（base_url = http://localhost:13305/v1）
- **Anthropic**: `POST /api/messages`（支援 tool use 和 SSE streaming）
- **Ollama**: `/api/` prefix（無版本號），相容 Ollama client SDK

## 相關 Issues 參考
- **llama backend with rocm 7.2**: https://github.com/lemonade-sdk/lemonade/issues/1149  
  ROCm 後端相容性問題
- **lemond run by systemctl cannot see NPU drivers**: https://github.com/lemonade-sdk/lemonade/issues/1358  
  systemd 服務環境變數問題（NPU driver 路徑）
