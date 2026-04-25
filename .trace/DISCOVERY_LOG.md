# Lemonade 探索紀錄、落差分析與技術債彙整

> 產出日期：2026-04-25 | 版本：10.2.0 | 探索範圍：全 codebase + 外部資源

---

## 1. Web Search 發現摘要

| 來源 | 連結 | 關鍵 Takeaway |
|------|------|--------------|
| 官方文件入口 | https://lemonade-server.ai/docs/ | API 規格、設定、開發指南的單一入口點 |
| AMD 技術文章（2026） | https://www.amd.com/en/developer/resources/technical-articles/2026/lemonade-for-local-ai.html | AMD 定位 Lemonade 為「統一本地 AI API 層」，開發者無需感知底層硬體差異 |
| Hacker News 討論 | https://news.ycombinator.com/item?id=47612724 | 社群對 AMD NPU 支援與 OpenAI 相容性反應正面；「non-Nvidia ecosystem 的重要缺口填補」 |
| DEV Community | https://dev.to/max_quimby/amds-lemonade-just-made-every-nvidia-only-ai-guide-obsolete-2a3l | 定位清楚：Nvidia-only 教程時代的終結者 |
| Phoronix 10.0.1 | https://www.phoronix.com/news/Lemonade-10.0.1 | Linux NPU 安裝流程持續改善中（社群重要關注點） |
| FLM NPU Linux | https://lemonade-server.ai/flm_npu_linux.html | FLM 可在 Linux 做 NPU 推論，但 Windows 支援仍更完整 |
| GAIA 整合 Issue | https://github.com/amd/gaia/issues/372 | AMD 生態系其他專案（GAIA）已整合 Lemonade WebSocket Realtime API，可作為參考實作 |
| DeepWiki 第三方分析 | https://deepwiki.com/lemonade-sdk/lemonade | AI 生成的架構分析，可交叉驗證自行探索結果 |
| Issue #1149（ROCm） | https://github.com/lemonade-sdk/lemonade/issues/1149 | ROCm 7.2 後端相容性已知問題 |
| Issue #1358（systemd） | https://github.com/lemonade-sdk/lemonade/issues/1358 | systemd 服務無法看到 NPU driver 路徑（環境變數繼承問題）|

**重要外部限制發現**：FLM 同時支援 Embeddings 和 Reranking（`fastflowlm_server.h:10` 繼承
`IEmbeddingsServer + IRerankingServer + IAudioServer`），但官方行銷文件聚焦在 Audio，
Embeddings/Reranking 能力未被充分宣傳。

---

## 2. 既有文件與程式碼的落差

| 文件描述 | 程式碼實際情況 | 位置 |
|---------|--------------|------|
| AGENTS.md 說 FLM 能力為「Completion, Embeddings, Reranking, Audio」 | 正確。`FastFlowLMServer` 同時繼承 `IEmbeddingsServer + IRerankingServer + IAudioServer` | `src/cpp/include/lemon/backends/fastflowlm_server.h:10` |
| `server_spec.md` 指向舊格式（一行轉址至 api/README.md） | 實際 API 文件已分散至 `docs/api/` 下多個檔案（openai.md、lemonade.md、ollama.md、anthropic.md） | `docs/server/server_spec.md:1` |
| AGENTS.md 說 Anthropic endpoint 是 `POST /api/messages`（無版本 prefix） | 正確，且此 endpoint 是**唯一不在 quad-prefix 下**的 API（其他 API 皆在 4 個 prefix 下）| `src/cpp/server/anthropic_api.cpp`、`extensions.md` |
| `docs/api/lemonade.md` 的 `websocket_port` 說「OS 指派或透過 `--websocket-port` 設定」| 設定欄位名稱為 `websocket_port`（JSON key），CLI 參數名稱需確認是否正確為 `--websocket-port` | `src/cpp/server/config_file.cpp`、`docs/api/lemonade.md:486` |
| `images/upscale` endpoint 在 `docs/api/openai.md` 有完整文件 | `recon.md` 原始紀錄誤判為未文件化，實際已有詳細文件（含 RealESRGAN 模型說明） | `docs/api/openai.md:704` |
| `configuration.md` 說 `max_loaded_models` 每個 ModelType 的上限 | 程式碼實作確認；`-1` 為無限制；但文件未明確說明 FLM NPU slot（LLM/Audio/Embed 各一）不受此參數控制 | `src/cpp/server/router.cpp:282-305`、`docs/server/configuration.md:82` |
| AGENTS.md 說 `lemonade-server` 是「deprecated backwards-compatibility shim」 | 程式碼印證：`src/cpp/legacy-cli/main.cpp:46` 確實輸出 deprecation 警告 | `src/cpp/legacy-cli/main.cpp:1,46,54,75` |
| `configuration.md` 說 legacy 環境變數（`LEMONADE_PORT` 等）「僅當 config.json 不存在時讀取」 | `ConfigFile::migrate_from_env()` 驗證正確，僅首次遷移時讀取，隨後持久化到 config.json | `src/cpp/server/config_file.cpp` |
| docs 中未提及 `collection` recipe | `server_models.json` 中有兩個 collection（「Full Collection」和「Lite Collection」）以 `composite_models` 陣列組合多個子模型 | `src/cpp/resources/server_models.json:1392,1403` |

---

## 3. 程式碼中的 TODO/FIXME/HACK 彙整

透過 `grep -rn "TODO|FIXME|HACK|XXX|WORKAROUND|DEPRECATED"` 掃描，結果如下：

### 3a. 明確的 TODO（需追蹤）

| 位置 | 內容 | 分類 |
|------|------|------|
| `src/cpp/server/model_manager.cpp:202` | `cleanup_orphaned_blob()` 目前只清理單一 blob；TODO 建議擴展為跨 repo 的孤立 blob 掃描，並整合至 `cleanup_orphaned_cache()`，需搭配 `hf_hub_download()` Python integration test | 功能缺口 / 測試基礎設施 |

### 3b. Legacy/Deprecated 標記（向下相容負擔）

| 位置 | 內容 | 影響範圍 |
|------|------|---------|
| `src/cpp/legacy-cli/main.cpp:1,46,54,75` | `lemonade-server` 整個 binary 是 deprecated shim | 低，下個 major release 可移除 |
| `src/cpp/server/ollama_api.cpp:1182` | `/api/embeddings`（複數形式）標注為 Legacy embeddings API | 需保留 Ollama 相容性 |
| `src/cpp/server/model_manager.cpp:945,949` | `legacy_mmproj_to_checkpoint()` 和 `parse_legacy_mmproj()`：舊式 multimodal model 的 mmproj 欄位解析 | 可能長期存在（舊 server_models.json 格式相容） |
| `src/cpp/server/system_info.cpp:1594,1597` | FLM 版本解析的 fallback legacy parsing（JSON 解析失敗時） | 低優先，FLM 版本格式穩定後可移除 |

### 3c. OpenAI API 版本遷移標記

| 位置 | 內容 |
|------|------|
| `src/cpp/server/backends/llamacpp_server.cpp:485,496` | `max_tokens` 在 2024 年 9 月已被 OpenAI deprecated（改為 `max_completion_tokens`），目前程式碼兩個欄位並存處理 |

### 3d. 平台相關 Pragma（抑制警告）

| 位置 | 內容 |
|------|------|
| `src/cpp/tray/platform/linux_tray.cpp:445` | `#pragma GCC diagnostic ignored "-Wdeprecated-declarations"`（GTK API） |
| `src/cpp/server/utils/process_manager.cpp:466,471` | `addchdir_np` 在 macOS 26+ deprecated，改用 portable 替代方案，但保留舊路徑並抑制警告 |

---

## 4. 未解答的疑問與模糊地帶

### 4a. VAD 行為細節

`src/cpp/server/vad.cpp` 使用 `SimpleVAD`（純能量門限法，RMS）：
- `energy_threshold`：靜音/語音判定門限
- `onset_frames`：需要連續幾個 voice frame 才確認語音開始
- `hangover_frames`：語音結束後的延伸幀數（避免過早截斷）
- `min_speech_ms` / `min_silence_ms`：最短語音/靜音時長

**疑問**：VAD 的 default config 值（`energy_threshold` 等）在哪裡定義？是否可透過 Realtime API session 設定覆蓋？WebSocket Realtime 的 `turn_detection` 欄位是否對應到 VAD Config？

### 4b. FLM 後端的 Multimodal 能力

`FastFlowLMServer` 繼承 `IEmbeddingsServer + IRerankingServer + IAudioServer`，但：
- 是否支援視覺（Vision/Multimodal LLM）？
- `IAudioServer` 是否同時支援 STT 和 TTS，還是只有 STT？
- FLM 的 embedding 是否支援多語言？

### 4c. Collection Recipe 的載入行為

`collection` recipe 透過 `composite_models` 陣列組合多個子模型（如 Full Collection = Qwen3.5 + Flux + Whisper + Kokoro）。

**疑問**：
1. 呼叫 collection 模型的 `chat/completions` 時，如何決定路由到哪個子模型？
2. 是否有自動根據請求類型選擇子模型的邏輯（OmniRouter 的概念）？
3. `model_manager.cpp:1994` 中 `actual_recipe == "collection"` 的 download 邏輯是同時下載所有子模型嗎？

### 4d. Port Rebind 行為

`apply_config_side_effects()` 在 port 變更時設定 `rebind_requested_ = true`（`server.cpp:3715`），但 `server.cpp:979-1001` 的 rebind 邏輯在失敗時「restore old port and retry」。

**疑問**：rebind 失敗時（如 port 被占用），錯誤如何向客戶端回報？是否有 SSE 通知機制？

### 4e. Nuclear Eviction 的 Retry 策略

`router.cpp:385` 中 nuclear option 的觸發條件是「非 file-not-found 的任何錯誤」，evict all 後重試一次。

**疑問**：重試仍失敗時的 fallback？是否有最大重試次數？這是否可能造成雪崩效應（一個後端 crash → 清除所有模型 → 下次請求又需重新載入）？

---

## 5. 已知技術債

### 5a. 超大檔案問題

```mermaid
quadrantChart
    title 技術債分布圖
    x-axis 修改難度低 --> 修改難度高
    y-axis 影響範圍小 --> 影響範圍大
    quadrant-1 高優先重構
    quadrant-2 短期改善
    quadrant-3 觀察即可
    quadrant-4 長期規劃
    server.cpp(4035行): [0.85, 0.95]
    model_manager.cpp(3312行): [0.75, 0.80]
    ollama_api.cpp(1290行): [0.45, 0.50]
    anthropic_api.cpp(961行): [0.35, 0.45]
    legacy-cli shim: [0.15, 0.20]
    max_tokens兩版並存: [0.20, 0.35]
    orphaned_blob TODO: [0.30, 0.40]
```

| 檔案 | 行數 | 問題描述 | 建議 |
|------|------|---------|------|
| `src/cpp/server/server.cpp` | **4035** | 單一檔案包含 HTTP 路由、所有 handler、config 管理、web app mock 注入 | 按功能拆分：`image_handlers.cpp`、`model_handlers.cpp`、`admin_handlers.cpp` |
| `src/cpp/server/model_manager.cpp` | **3312** | 模型 registry、下載、user model、FLM 整合全混在一起 | 拆分：`hf_downloader.cpp`、`user_model_registry.cpp`、`flm_model_manager.cpp` |
| `src/cpp/server/ollama_api.cpp` | **1290** | Ollama 格式轉換邏輯與 route handler 混合 | 轉換邏輯獨立為 `ollama_format.cpp` |

### 5b. Subprocess 模型的固有限制

- 每次模型 load 需要啟動子程序並等待就緒（最多 600 秒超時），**不支援 streaming 進度**
- LRU eviction 需要等待 `wait_until_not_busy()`，若推論很慢則阻塞新請求
- 後端崩潰時 nuclear eviction 清除所有模型（服務中斷），設計上不支援 graceful degradation

### 5c. Plugin 化程度不足

後端新增**必須重新編譯**（無動態 `.so/.dll` 載入）。雖然 subprocess 模型讓任何語言都能包裝，但增加後端的門檻仍高（需修改 5 個地方：header、router、backend_manager、backend_versions.json、server_models.json）。

### 5d. `orphaned_blob` 清理 TODO

`model_manager.cpp:202`：`cleanup_orphaned_blob()` 僅清理單一 blob，缺乏跨 repo 的孤立 blob 掃描。TODO 中指出需要 Python integration test（使用 `hf_hub_download()`）才能驗證，這是**基礎設施依賴導致的測試債**。

---

## 6. 需要深入調查的區域

### 6a. WebSocket Realtime VAD 完整行為

- VAD 實作為純 RMS 能量門限法（`vad.cpp`），**非 ML-based**（無 Silero VAD 等模型）
- 待確認：VAD Config 的預設值與 WebSocket 訊息格式的對應
- 待確認：Realtime session 是否支援 VAD disable（靜音截斷控制）
- 相關檔案：`src/cpp/server/vad.cpp`、`src/cpp/server/realtime_session.cpp`、`src/cpp/server/streaming_audio_buffer.cpp`

### 6b. FLM 後端能力的完整邊界

依 `fastflowlm_server.h:10`，FLM 繼承 3 個 capability interface：

| Interface | 能力 | 確認狀態 |
|-----------|------|---------|
| `IEmbeddingsServer` | 文字向量嵌入 | 已確認在 header 中 |
| `IRerankingServer` | 文件重排序 | 已確認在 header 中 |
| `IAudioServer` | 語音轉錄（STT） | 已確認在 header 中 |

**未確認**：是否支援視覺/多模態（不繼承 `IImageServer`，但 FLM 文件提及多模態能力）。

### 6c. Collection Recipe 的路由與自動分派

`server_models.json` 中 Full Collection = [Qwen3.5-35B、Flux-2-Klein-9B、Whisper-Large-v3-Turbo、kokoro-v1]。
需調查 `model_manager.cpp:1994` 附近的 collection download 和 `router.cpp` 中是否有 collection 感知的路由邏輯，或依賴 OmniRouter 外部智能分派（`docs/omni-router.md` 有文件）。

### 6d. OmniRouter 整合

`sd_server.cpp:494` 有 OmniRouter tool call 的特殊處理。`docs/omni-router.md` 文件存在，但 codebase 中的整合深度未完整探索。

---

## 7. 與維護者或社群確認的問題清單

開發者上手時建議確認以下問題：

1. **VAD 可調參數**：`SimpleVAD::Config` 的預設值（`energy_threshold`、`onset_frames` 等）是否有建議調整場景？WebSocket Realtime API 的 `session.update` 訊息是否可覆蓋這些參數？

2. **FLM Embeddings 生產就緒程度**：`FastFlowLMServer` 雖實作 `IEmbeddingsServer`，FLM 的 embedding 功能是否已達生產品質？是否有對應的 integration test？

3. **Nuclear Eviction 在生產環境的行為**：`evict_all_servers()` + retry 的策略在高負載情況下是否有已知問題（雪崩效應）？是否有計畫加入 circuit breaker 機制？

4. **Collection Recipe 的請求路由**：呼叫 collection model 的 `/v1/chat/completions` 時，如何決定要送到哪個子模型？是靜態對應（文字→LLM、圖片→SD）還是依賴 OmniRouter 動態分派？

5. **FLM Linux NPU 的生產狀態**：Phoronix 報導 10.0.1 改善了 Linux NPU 安裝流程，但實際 FLM 在 Linux 是否已穩定到可在 CI 中自動測試？目前 `test/server_llm.py` 的 `--wrapped-server fastflowlm` 是否有對應 CI job？

6. **server.cpp 4035 行拆分計畫**：是否有 issue/milestone 追蹤 `server.cpp` 的模組化工作？還是維護者接受此現狀作為「單一入口」的設計？

7. **`lemonade-server` Shim 的退役時程**：`legacy-cli/main.cpp` 何時會被移除？是否綁定在某個 major version 的 breaking change 計畫中？

8. **多客戶端並發的壓力測試**：AGENTS.md Critical Invariant #11 強調一對多客戶端拓撲。是否有專門測試多個 Tauri app 同時連到同一個 lemond 的 integration test？

9. **`extra_models_dir` GGUF 掃描的效能邊界**：`discover_extra_models()` 自動掃描目錄中所有 `.gguf` 檔。若目錄下有數百個大型 GGUF 檔，是否有效能問題？是否有快取機制？

10. **Windows `WMI_helper` 的 NPU 偵測準確性**：`src/cpp/server/utils/wmi_helper.cpp` 使用 WMI 查詢 GPU/NPU 資訊。Issue #1358 顯示 systemd 下 NPU driver 路徑問題，Windows 上 WMI 是否也有類似 driver 路徑偵測問題？

---

*本文件由自動化探索流程產出，覆蓋 `src/cpp/`（server、backends、include、cli、tray）、`src/app/`、`docs/` 及外部搜尋結果。如有更新請重新執行探索流程或手動修訂。*
