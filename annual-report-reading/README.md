# Annual Report Buy-Side Skill Pack

這套 Skill 用於將台股、美股、日股、韓股的年度法定揭露文件，轉換為統一、可追溯、可供網站卡片呈現的結構化 JSON。

## 檔案結構

| 檔案 | 用途 |
|---|---|
| `SKILL.md` | 主 Skill；角色、任務、完整工作流程與輸出要求 |
| `schema.md` | `annual_report_summary_v2` 完整 JSON Schema 說明 |
| `market_mapping.md` | 台、美、日、韓年報章節映射與關鍵字 |
| `extraction_rules.md` | 產品、研發、客戶、供應鏈、競爭者、風險擷取規則 |
| `investment_analysis.md` | Buy-side 判斷、重大性排序、Bull/Base/Bear 框架 |
| `validation.md` | JSON 驗證、數字一致性、證據與幻覺防範規則 |
| `card_rendering.md` | 將完整研究 JSON 轉成網站文字卡片的規則 |
| `annual_report_summary_v2.schema.json` | 可直接用於程式驗證的 JSON Schema |

## 建議流程

```text
年度法定文件
  ↓
SKILL.md 執行分析
  ↓
annual_report_summary_v2 JSON
  ↓
validation.md 驗證
  ↓
card_rendering.md 轉為前端卡片
```

## 使用方式

將 `SKILL.md` 作為主系統提示，並一併提供其他規則檔案。若模型支援工具式 Skill，可讓主 Skill 按需求讀取其餘檔案。

最終輸出僅允許合法 JSON，不得輸出 Markdown、註解或補充說明。
