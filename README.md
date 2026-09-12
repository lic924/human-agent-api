# Human Agent API

供 Core 串接的唯讀 Human Agent：個人與公共資源分析、world v0.12 單 tick 核算、RAG、LLM 工具流程與最多三輪討論。Human 提出建議，Core 決策並執行世界操作；討論期間世界暫停。

## 啟動

使用 Python 3.11+。以下 PowerShell 命令建立獨立環境，預設 **mock，不需金鑰**：

```powershell
git clone --depth 1 https://github.com/lic924/human-agent-api.git
cd human-agent-api
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
.\.venv\Scripts\python.exe -m uvicorn app.main:app --host 127.0.0.1 --port 8000
```

macOS／Linux 使用 `.venv/bin/python`，以 `cp -n .env.example .env` 建立設定。已有倉庫使用 `git pull --ff-only` 更新，重新安裝依賴並重啟服務。

開 http://127.0.0.1:8000/docs，選 **POST /discuss → Try it out → Examples「正常：使用 current_plan 核算」→ Execute**。健康檢查為 `/health`，機器可讀契約為 `/openapi.json`。

## 測試

服務保持運行，另開同目錄終端：

```powershell
.\.venv\Scripts\python.exe examples/discussion_client.py --fixture current_plan
.\.venv\Scripts\python.exe examples/discussion_client.py --fixture current_plan --rounds 3
.\.venv\Scripts\python.exe examples/discussion_client.py --request .\my_core_request.json
```

程式顯示中文回覆與秒數，保存每輪 request／response／metrics 至 `examples/manual_discussion_responses/`。缺氧、失灌、補給競爭等測試見 [START_TESTING.md](docs/START_TESTING.md)。

獨立自動驗收會自行啟停測試服務：

```powershell
.\.venv\Scripts\python.exe -m pytest -q
.\.venv\Scripts\python.exe scripts/discussion_smoke.py --all
.\.venv\Scripts\python.exe scripts/http_smoke.py --all
```

mock 驗證規則、核算與接口；語意分析需 Live。實測摘要見 [validation_report.md](docs/validation_report.md)。執行產生的報告與回覆不納入 Git。

## Core 串接

| 接口 | 請求範例 | 用途 |
|---|---|---|
| `POST /discuss` | `examples/discussion_requests/core_original_human.json` | Core v1.3 討論訊息，無計畫亦可分析 |
| `POST /discuss` | `examples/discussion_requests/current_plan.json` | 帶明確候選的量化分析 |
| `POST /human-agent/analyze` | `examples/requests/normal.json` | Human API 2.0 詳細分析與 ledger |

`content.world` 就是當次 snapshot，不需要額外物件或 API。回覆沿用 discussion_id／round／world_version，最多三輪，previous_messages 使用真實前輪訊息。未知 alive 保持未知；核算僅到灌溉後、作物操作前，不是世界狀態更新或長期生存保證。

`power` 使用 EU，`1 EU = 3.9745 kWh`；換算不改變世界係數。第二次失灌標 critical、尚未死亡；第三次植物死亡的 audit 為 unsafe_in_scope。請查看風險與可行性，HTTP 200 不代表安排安全。

完整約定見 [core_handoff_alignment.md](docs/core_handoff_alignment.md)，規則見 [spec.md](spec.md)，兩個 API 的 request／response JSON Schema 在 `docs/`。422 為格式錯誤，409 為契約／版本不相容；`/discuss` 的分析失敗／逾時為 502／504，配合 Core 45 秒 HTTP timeout。

跨機時以 `--host 0.0.0.0` 啟動，Core 填 Human 電腦可達 IP。`0.0.0.0` 不是客戶端目的位址；不同電腦不能使用對方的 localhost。正式世界與區網仍須雙方實際聯測。

## Live 設定

在自己的 `.env` 設 `AGENT_MODE=live`，填入 `LLM_API_KEY`、`EMBEDDING_API_KEY`。provider、base URL、model 必須與金鑰可用服務一致；範本不提供模型使用權或金鑰。環境變數優先於 `.env`，修改後重啟服務。

預設 LLM 為 OpenAI 相容 Responses 流程，embedding 為 OpenRouter `voyageai/voyage-4`／1024 維。協定細節見 [provider_protocols.md](docs/provider_protocols.md)。

```powershell
.\.venv\Scripts\python.exe scripts/check_providers.py
.\.venv\Scripts\python.exe scripts/discussion_smoke.py --live
```

Live 會使用付費 API；`/health` 不探測遠端模型。缺設定會回報欄位名稱，不輸出金鑰。不要提交自己的 `.env`。

## RAG 資料與維護

倉庫只包含整合運行所需的 **393 chunks、來源 manifest、世界規則、BM25 metadata 和目前 dense 索引**。啟動不下載、不建索引；完整 PDF、抽取中間檔、舊版索引、歷史回覆與開發紀錄不在目前交付目錄中。

來源可由 `data/source_manifest.jsonl` 的 URL／SHA-256 追溯，chunks 附原文位置。科學語意驗證仍為 partial；固定世界核算不依賴模型或文獻相似度。來源說明見 [parameter_review.md](docs/parameter_review.md)。

一般串接不需重建。若更換 embedding 的 provider／model／dimensions／policy，需建立相容索引：`python scripts/build_index.py --mode both`（可能付費）。此動作使用既有 chunks；若要重新抽取原文，再執行 `python scripts/acquire_sources.py` 和 `python scripts/prepare_corpus.py --local`，輸出留在本機忽略目錄。索引發布需同時保留 current.json 指向的 generation 與對應 metadata。

Git 保留既有提交歷史；新的整合副本可用上述 `--depth 1` 只取得最新版本。
