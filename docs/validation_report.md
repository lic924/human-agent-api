# 整合驗證摘要

## 自動驗收

2026-09-12 精簡後，以 Git index 匯出的 83 檔乾淨副本（約 3.9 MiB）重新執行：**130 passed（13.08 秒）**，Core 11 個與 Human 六個 mock HTTP 案例全部通過。副本沒有完整 PDF、抽取中間檔、舊索引或個人金鑰；393 chunks、BM25 與唯一現行 dense generation 均成功載入，文件連結檢查通過。

```powershell
python -m pytest -q
python scripts/discussion_smoke.py --all
python scripts/http_smoke.py --all
```

- 130 項測試：世界核算、RAG、provider mock、Core 契約、三輪引用及人工客戶端。
- Core mock HTTP 11 個案例：原始請求、三輪討論、current_plan、個人資源不足、缺氧、補給競爭、第二／第三次失灌、已失敗。
- Human mock HTTP 六情境：正常、個人資源不足、提早缺氧、補給競爭、第三次失灌、資料不足。

mock 不需要金鑰，不代表模型已做語意分析。測試腳本自行啟停本機服務；生成的 JSON 報告與回覆保留於本機，Git 忽略這些執行產物。

## Live 既有驗證

2026-09-12 實際原始 Core 請求及三輪討論皆 HTTP 200：22.69／28.53／31.75／34.13 秒。第二／三輪各有 2／3 項有效前輪提案評論。此為單次量測，非延遲保證。

可用 `python scripts/discussion_smoke.py --live` 在自己的設定下重新驗證；需要有效金鑰，會呼叫遠端 API。Plant 歷史訊息為明示測試資料，並未連到正式 Plant 服務。

## 交付範圍

目前交付只依賴原文 chunks 與現行索引，啟動不需要完整 PDF 或抽取中間檔。來源 manifest、chunks hash、索引相容檢查保留。科學證據仍為 partial，只有 reviewed_claims 中的有限結論經原文核對。

正式 Core／世界引擎差異與第二台設備 LAN 需雙方服務聯測。本次檔案精簡不更改模型設定、世界規則或核算公式。
