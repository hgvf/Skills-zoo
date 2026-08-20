# Defense Acquisition Intelligence Skill

從各國官方國防採購來源持續發現、抽取、正規化「國防採購事件」，輸出可直接寫入 Firestore
---

## 0. FINAL RESPONSE CONTRACT — STRICT JSON ONLY

> 本節優先於所有其他呈現、解釋、citation、markdown、對話行為。

當本 skill 執行「ingestion run」時，assistant 的最終回覆 MUST 只包含一個合法 JSON 物件：

1. 第一個非空白字元必為 `{`，最後一個必為 `}`
2. 不得有 markdown code fence、標題、表格、prose、citation、備註
3. `json.loads()` 可直接解析（不得有 trailing comma、註解、NaN、`None`、單引號）
4. 零事件時仍回傳完整結構，`"events": []`
5. 抓取失敗記在 `run.sources[].status` / `run.errors`，**絕不用 prose 解釋**
6. 過濾決策、內部推理、被排除事件 → 不出現在最終輸出
7. 使用者若明確要求「解釋 / debug skill 本身」而非執行 ingestion，允許正常 prose

批次外層固定為：

```jsonc
{
  "run":    { /* §7 Run 物件 */ },
  "events": [ /* §3 Event 物件 陣列 */ ]
}
```

---

## 1. 範圍與非目標

**收錄**：合約、修改、採購案、標案、預算請求/撥款、研發、原型/測試、量產、升級、
維持、對外軍售/出口核准、重大專案決策。

**忽略**（除非直接綁定武器計畫）：人事、演習、外交訪問、部隊調動、通用工程、
餐勤/服裝/文具、一般醫療採購、通用商用 IT、日常基地維護、招募、典禮。

**收錄國家**：US / JP / KR / TW（預設）；AU / GB 為後續擴充預留（enum 允許但目前不主動抓）。

---

## 2. Source policy — 必為官方一手來源

優先順序：
1. 官方採購 / 國防部下屬採購機關（DoW Contracts、ATLA、DAPA、國防部本部）
2. 官方軍種 / 部會
3. 官方預算 / 審計 / 國會來源
4. 官方政府採購系統
5. 其他官方政府來源
6. 美國相關網站
| 類型         | 官方來源                                 | 會公布什麼                                            | 適合網站功能                 | 價值    |
| ---------- | ------------------------------------ | ------------------------------------------------ | ---------------------- | ----- |
| **合約授予**   | War.gov Contracts                    | 廠商、金額、合約號、數量、用途、完成日期                             | Contract Timeline      | ★★★★★ |
| **國防預算**   | DoD Comptroller Budget Justification | 每個武器年度採購量、單價、改良計畫、R&D 預算                         | Procurement / Budget   | ★★★★★ |
| **武器測試**   | DOT&E                                | 實戰測試結果、缺陷、可靠度、性能是否達標                             | Test & Evaluation      | ★★★★★ |
| **重大武器計畫** | GAO Weapon Systems Assessment        | 成本超支、延誤、技術問題、預計 IOC                              | Program Health         | ★★★★★ |
| **對外軍售**   | DSCA / State Dept                    | 哪國買什麼、數量、估計金額、配套武器                               | Operators / Export Map | ★★★★★ |
| **所有聯邦採購** | USAspending                          | 小型/大型 contract、transaction history、obligation    | Contract Database      | ★★★★  |
| **招標中需求**  | SAM.gov                              | 還沒簽約的 RFI/RFP、軍方想買什麼                             | Upcoming Programs      | ★★★★  |
| **國會研究**   | CRS                                  | F-35、B-21、DDG(X)、Columbia 等完整 program background | Research / Context     | ★★★★★ |
7. 其他國家相關網站
| 國家      | 最值得抓的政府來源           | 能拿到的軍武資料           | 公開程度    | 對 weapon-research 價值 |
| ------- | ------------------- | ------------------ | ------- | -------------------- |
| 🇯🇵 日本 | **防衛裝備廳 ATLA**      | 招標、得標、契約、預定採購、R&D  | 很高      | ★★★★★                |
| 🇰🇷 韓國 | **DAPA 防衛事業廳**      | 武器計畫、開發決策、採購、合約、量產 | **非常高** | ★★★★★                |
| 🇹🇼 台灣 | 國防部 + 政府電子採購網 + 預算書 | 預算、部分採購、軍購、國造進度    | 中等      | ★★★★                 |


**媒體不可作為事實依據**，只能用來發現一手來源。每個 stored event MUST 至少一筆
`sources[].official = true` 的 URL。來源清單見 `config/sources.yaml`。

---

## 3. Event 物件 — 完整 schema

> 這是輸出 JSON 中 `events[]` 每一個元素的完整結構。註解即規則。

```jsonc
{
  // ── 識別 / 主鍵 ────────────────────────────────────────────
  "event_id": "us_2026-08-18_mk41_bae_canister",
  // 建議必填；作為 Firestore doc id。省略時系統依 §4 自動產生。
  // 重貼同 id = 覆蓋（daily rerun 安全）。

  // ── 分類 / 篩選 ────────────────────────────────────────────
  "country":    "US",                          // enum §5.1
  "agency":     "Naval Sea Systems Command",   // 選填
  "service":    "Navy",                        // enum §5.2；選填
  "event_type": "contract_modification",       // enum §5.3
  "publication_date": "2026-08-18",            // YYYY-MM-DD；清單排序主鍵
  "event_date":       "2026-08-18",            // YYYY-MM-DD；缺 publication_date 時 fallback
  "event_date_basis": "publication_date",      // "explicit" | "publication_date"

  // ── 標題 / 摘要（中文為主，英文原文為輔）─────────────────
  "title_zh":       "BAE Systems 獲 1.692 億美元 MK 41 VLS 發射筒增購合約", // *必填之一
  "title":          "BAE Systems MK 41 VLS canister production contract modification", // *必填之一
  "title_original": "Contracts for Aug. 18, 2026",   // 原始公告標題
  "summary_zh":     "……中文改寫摘要，用於清單卡片主要內文……",
  "summary":        "……English original/normalized summary……",
  "source_text_original": "……原文完整相關段落，備查用不直接顯示……",

  // ── 行動 ───────────────────────────────────────────────────
  "action": {
    "type": "production",                      // enum §5.4
    "description_zh": "……中文行動說明（卡片行動框）……",
    "description":    "……English……",
    "expected_completion_date": "2031-04-30"   // YYYY-MM-DD | null
  },

  // ── 相關計畫（可多個）─────────────────────────────────────
  "programs": [
    {
      "program_id":       "mk-41-vls",                          // 選填；conservative slug
      "canonical_name":   "MK 41 Vertical Launching System",    // 英文標準名（輔）
      "name_zh":          "MK 41 垂直發射系統",                  // 中文名（主，顯示於 chip）
      "program_name_raw": "MK 41 VLS canister and ancillary equipment", // 原始字串
      "variant":          null,                                 // e.g. "Lot 18-19"
      "category":         "weapon_component",                   // enum §5.5
      "aliases":          ["MK 41 VLS"]
    }
  ],

  // ── 承包商（決定股票標記，見 §6）──────────────────────────
  "contractor": {
    "contractor_id":  "bae-systems-land-armaments",
    "name":           "BAE Systems Land & Armaments",
    "name_raw":       "BAE Systems Land & Armaments L.P",       // 原文照抄
    "country":        "US",
    "public_company": false,
    "ticker":         null,                                     // 直接 ticker（有值→前端藍字）
    "exchange":       null,                                     // NYSE|NASDAQ|LSE|TSE|KRX|TWSE...
    "parent_company": "BAE Systems",
    "parent_ticker":  "BA.",
    "ticker_basis":   "listed_parent"                           // enum §5.6
  },

  // ── 合約金額 ───────────────────────────────────────────────
  "contract": {
    "contract_number": "N00024-24-C-5324",   // 顯示「合約 …」；參與 dedup / auto event_id
    "amount":          169216659,             // 純數字，來源幣別下的金額（不做 FX 換算）
    "currency":        "USD",                 // ISO 4217
    "amount_raw":      "$169,216,659",        // 詳情 modal 優先顯示此原字串
    "amount_type":     "modification_value",  // enum §5.7
    "quantity":        null,
    "quantity_unit":   null
  },

  // ── 標籤 / 重要度 / 品質 ──────────────────────────────────
  "importance_score": 82,                                       // 0–100 整數，見 §8
  "tags": ["mk-41","vls","navy","production","fms","missile-launcher"],
  "quality": {
    "needs_review": false,
    "issues": []                                                // needs_review=true 時，卡片顯示這些
  },

  // ── 來源（至少一筆 official + url）────────────────────────
  "sources": [
    {
      "publisher":        "U.S. Department of War",
      "url":              "https://www.war.gov/……",
      "source_type":      "contract",                           // contract|press_release|budget|...
      "official":         true,
      "publication_date": "2026-08-18"
    }
  ],

  // ── 處理時間戳 ─────────────────────────────────────────────
  "ingested_at": "2026-08-19T00:12:34Z"        // RFC3339；選填，pipeline 自動補
}
```

**唯一硬性必填**：`title_zh` 或 `title` 至少一個。其餘可為 `null` 或省略，但對應前端區塊
會顯示為空或「未揭露」；`sources[]` 至少一筆 official 才允許寫入。

---

## 4. Event ID / 主鍵 / 去重

**Doc id 產生規則**：

- 若給 `event_id` → `slug(event_id)`
- 否則 → `slug(country + "_" + (publication_date || event_date || "nodate") + "_" + (contract_number || title[:24]))`

**同 id 重貼 = 覆蓋**，daily rerun 天然去重。

**額外指紋去重**（詳見 `config/dedup.md`）：

1. exact key `{country}:{contract_number}:{event_date}:{event_type}`
2. exact key `{country}:{official_document_id}:{event_type}`
3. fallback fingerprint：`sha256("|".join([country, event_type, event_date,
   normalized_program, normalized_contractor, contract_number, amount, quantity]).lower())`

不得**只用 title** 去重。

---

## 5. Enum 對照表

### 5.1 `country`
`US` | `JP` | `KR` | `TW` | `AU` | `GB`
（前端會轉中文顯示；其他值原樣顯示）

### 5.2 `service`
`Navy` | `Army` | `Air Force` | `Joint` | `Marine Corps` | `Space Force` | `Coast Guard`

### 5.3 `event_type`
`contract_award` | `contract_modification` | `procurement_plan` | `tender` |
`budget_request` | `budget_appropriation` | `development` | `test` | `production` |
`upgrade` | `sustainment` | `foreign_military_sale` | `export_approval` |
`program_decision` | `other_acquisition`

### 5.4 `action.type`
`new_procurement` | `follow_on_procurement` | `research` | `development` |
`prototype` | `testing` | `production` | `mass_production` | `upgrade` |
`modernization` | `repair` | `sustainment` | `spares` | `training` | `support` |
`foreign_sale` | `program_approval` | `solicitation` | `other`

### 5.5 `programs[].category`
`aircraft` | `helicopter` | `missile` | `munition` | `air_defense` |
`ground_vehicle` | `naval` | `submarine` | `drone` | `radar_sensor` |
`electronic_warfare` | `space` | `cyber` | `c4isr` | `engine_propulsion` |
`weapon_component` | `support` | `unknown`

**分類提示**：AH-64 → `helicopter`；AIM-120 → `missile`；JDAM → `munition`；
Aegis radar upgrade → `radar_sensor`；MT7 engine → `engine_propulsion`。

### 5.6 `contractor.ticker_basis`
| 值 | 意義 | 前端行為 |
|---|---|---|
| `direct` | 承包商本身上市 | 顯示 `ticker` 藍字 |
| `listed_parent` | 承包商為未上市子公司，但母公司上市 | 顯示 `parent_ticker` 藍字並標「(母)」 |
| `private` | 私人公司 | 顯示「非上市」灰標籤 |
| `government` | 政府/軍方/國研單位（NCSIST、DARPA、大學實驗室） | 顯示「政府·學研」灰標籤 |
| `unresolved` | 無可靠對照 | 顯示「未對照」灰標籤 |

### 5.7 `contract.amount_type`
`award_value` | `modification_value` | `ceiling` | `estimated_value` |
`budget_request` | `appropriation` | `unknown`

**Ceiling ≠ 實支**。IDIQ maximum → `ceiling`；勿計入實際支出統計。

---

## 6. Contractor + Ticker 解析規則

`contractor.name_raw` **必須逐字保留原文**。ticker 對照優先讀
`config/contractors.json`；**絕對不得從公司名相似度猜 ticker**。

決策順序：

1. 承包商本身在 registry 中有 direct listing → `ticker` + `ticker_basis="direct"`
2. 承包商是未上市子公司、其 legal parent 在 registry 中上市
   → `ticker=null`、`parent_company/parent_ticker` 填母公司、`ticker_basis="listed_parent"`
3. 明確為私人公司 → `ticker=null`、`ticker_basis="private"`
4. 大學 / 政府 / 國研 → `ticker=null`、`ticker_basis="government"`
5. 其他無對照 → 全 null、`ticker_basis="unresolved"`

**不因為 ticker 解析失敗而阻擋 ingestion**。**不因為母公司上市就把合約承包商改成母公司**——
合約主體固定為原始承包商，母公司資料只是 enrichment。

---

## 7. Run 物件（批次外層 metadata）

```jsonc
{
  "run_date":     "2026-08-18",            // 執行日
  "generated_at": "2026-08-19T00:15:00Z",  // RFC3339
  "status":       "success",               // success | partial_success | failed
  "countries":    ["US","JP","KR","TW"],   // 本次涵蓋國家
  "sources": [
    {
      "source_id":       "us_war_contracts",
      "status":          "checked",        // checked | fetch_failed | parse_failed
      "new_documents":   3,
      "relevant_events": 7
    }
  ],
  "errors": [],                            // 機器可讀 error 物件/字串陣列
  "excluded_count": 12                     // 選填；本次被過濾掉的事件數（僅計數，不含明細）
}
```

`run` 目前**不入庫、不渲染**，只作 pipeline audit。任何一個 source `fetch_failed`
且其他成功 → `status = "partial_success"`；全部失敗 → `"failed"`。

---

## 8. Importance scoring（確定性規則，禁用 LLM 直覺）

從 0 開始累加，上限 100：

| 條件 | 加分 |
|---|---:|
| 新武器 / 新命名專案 | +30 |
| 採購 / 量產動作 | +25 |
| 研發 / 原型 / 重大測試 | +20 |
| 金額 ≥ USD 500M 等值 | +15 |
| 金額 ≥ USD 100M 等值（與 500M 互斥，只取高） | +10 |
| 明確 quantity | +10 |
| 主要作戰平台 / 飛彈 / 防空系統 | +10 |
| 對外軍售 / 出口決策 | +10 |
| 重大計畫核准 / 取消 / 重整 | +15 |
| 純例行 sustainment | +5 |

**500M / 100M 兩級不疊加**。FX 換算只用於 scoring（可用 `config/fx.yaml` 的近似匯率），
**絕不寫入事實 `contract.amount`**。

---

## 9. Daily pipeline workflow

```
SOURCE REGISTRY  →  DISCOVER NEW DOCS  →  FETCH  →  RELEVANCE FILTER (§10)
       →  SPLIT INTO EVENTS  →  EXTRACT (prompts/extract_event.md)
       →  ENTITY LINK (prompts/entity_link.md)
       →  CONTRACTOR/TICKER (prompts/contractor_ticker.md, §6)
       →  ZH NORMALIZE (prompts/normalize_zh.md)
       →  VALIDATE schema (§3)  →  IMPORTANCE (§8)
       →  DEDUP (§4)  →  FINAL BATCH (prompts/final_output.md)
```

**Lookback**：daily feeds 3 天、weekly/budget feeds 14 天。存在的用意是撐過失敗執行
與週末發佈差異，dedup 保證不重複。

---

## 10. Relevance filter

僅收錄含**具體採購行動**的文件。正向 keyword：awarded / contract / modification /
procurement / quantity / lot / prototype / test / low-rate initial production /
modernization / depot repair / spare parts / military sale / export approval /
program approval / budget line tied to a named platform。

範例：
- ✅ `$636.1M Apache depot-level repair and logistics support`
- ✅ `12 UH-72B helicopters`
- ✅ `KF-21 follow-on production decision`
- ❌ `base dining services contract`
- ❌ `minister visits an exercise`
- ❌ `generic cybersecurity license renewal with no weapon linkage`

**不因為機關名稱就推斷相關**（DoW 也會發薪資合約）。

---

## 11. 抽取原則（給 LLM 的硬性紀律）

- 只採用來源明確支持的事實；缺失一律 `null`
- **絕不編造**：quantity / contract_number / 變體型號 / 承包商 / 金額 / 完工日
- 保留原幣別；`amount_raw` 保留原字串
- 保留 `program_name_raw` 逐字原文
- 區分：`ceiling` vs `award_value`；`procurement_plan` vs `contract_award`；
  `contract_modification` vs 新約；`sustainment` vs 採購；`production` vs `development`
- 一份文件多筆合約 → **拆成多個 event**；同一合約多裝備 → 一個 event + `programs[]` 陣列
- Traditional Chinese (`zh-TW`) 用於 `title_zh` / `summary_zh` / `action.description_zh`；
  武器代號、公司法定名稱、合約號、數字、日期**保持原樣不翻譯**

---

## 12. Quality flags

不做模糊的「confidence score」。改用：

```jsonc
"quality": {
  "needs_review": true,
  "issues": ["ceiling_vs_award_ambiguous", "quantity_unclear"]
}
```

`needs_review = true` 觸發條件：program identity 模糊 / 來源內部矛盾 / OCR 損毀 /
quantity 語意不清 / ceiling vs award 不清 / 來源內容不完整。前端會把 `issues[]`
的字串顯示在卡片上。

---

## 13. Firestore write model

```
defense_events/{event_id}     ← 唯一真實來源（source of truth）
programs/{program_id}         ← enrichment cache / index
contractors/{contractor_id}   ← enrichment cache / index
ingestion_runs/{run_id}       ← run metadata
```

Program / contractor 表為衍生索引；**不得因為 entity 解析失敗就阻擋 event 寫入**。

---

## 14. Failure & security

- 抓取失敗 → `run.sources[].status = "fetch_failed"` + `run.errors[]`，**不編造「無更新」**
- 頁面結構改變 → 記錄 parser failure + URL，缺欄一律 `null`，不猜
- 只用**合法公開資訊**：不試圖取得機密文件、洩漏資料、憑證、非公開採購系統

---

## 15. Daily 完成準則

當 (1) 所有 enabled sources 已檢查或明確標記失敗、(2) 新文件已評估、
(3) 相關事件已抽取並通過 §3 schema、(4) 去重完成、(5) 每筆有 official provenance、
(6) 批次 JSON 已產生且 `json.loads()` 可解析、(7) `ingestion_runs/` 已記錄 → **success**。
其中任一 source 失敗但其他成功 → **partial_success**。

---

## 16. 最小可用範例

```json
{
  "run": {
    "run_date": "2026-08-20",
    "generated_at": "2026-08-20T00:05:00Z",
    "status": "success",
    "countries": ["TW"],
    "sources": [{ "source_id": "tw_mnd_press", "status": "checked", "new_documents": 1, "relevant_events": 1 }],
    "errors": []
  },
  "events": [
    {
      "event_id": "tw_2026-08-20_ncsist_hf3",
      "country": "TW",
      "event_type": "contract_award",
      "publication_date": "2026-08-20",
      "event_date": "2026-08-20",
      "event_date_basis": "publication_date",
      "title_zh": "中科院獲雄三增程型量產撥款",
      "title": "NCSIST awarded HF-3 ER production funding",
      "summary_zh": "國防部撥款中科院執行雄風三型增程型量產,金額約新台幣 32 億元。",
      "action": { "type": "mass_production", "description_zh": "量產撥款", "expected_completion_date": null },
      "programs": [{
        "program_id": "hf-3", "canonical_name": "Hsiung Feng III", "name_zh": "雄風三型",
        "program_name_raw": "雄風三型增程型", "variant": "ER", "category": "missile", "aliases": ["HF-3 ER"]
      }],
      "contractor": {
        "contractor_id": "ncsist", "name": "NCSIST", "name_raw": "中山科學研究院",
        "country": "TW", "public_company": false,
        "ticker": null, "exchange": null, "parent_company": null, "parent_ticker": null,
        "ticker_basis": "government"
      },
      "contract": {
        "contract_number": null, "amount": 3200000000, "currency": "TWD",
        "amount_raw": "新台幣 32 億元", "amount_type": "appropriation", "quantity": null, "quantity_unit": null
      },
      "importance_score": 70,
      "tags": ["anti-ship","hf-3","taiwan","mass-production"],
      "quality": { "needs_review": false, "issues": [] },
      "sources": [{
        "publisher": "中華民國國防部", "url": "https://www.mnd.gov.tw/……",
        "source_type": "press_release", "official": true, "publication_date": "2026-08-20"
      }]
    }
  ]
}
```
