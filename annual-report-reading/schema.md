# annual_report_summary_v2 Schema Guide

本檔說明完整 JSON 的資料模組。正式程式驗證請使用 `annual_report_summary_v2.schema.json`。

## 核心模組

| 模組 | 說明 |
|---|---|
| `document` | 市場、文件類型、年度、日期、幣別與查核狀態 |
| `company` | 產業、商業模式、價值鏈、核心能力與護城河 |
| `headline` | 標題、摘要、基本面立場、信心與核心問題 |
| `business_segments` | 各事業部營收、獲利、產品與成長狀態 |
| `product_portfolio` | 主力產品、新量產產品、研發中產品與路線圖 |
| `research_and_development` | 研發費用、成功成果、進行中計畫與未來方向 |
| `revenue_mix` | 產品、部門、客戶、地區、應用與通路占比 |
| `customers` | 實名客戶、集中度、地區分布與客戶變化 |
| `supply_chain` | 原料、供應商、工廠、委外商、通路與風險 |
| `financial_highlights` | 損益、現金流、CapEx、存貨、應收與訂單 |
| `operating_review` | 年度營運變化、原因與持續性 |
| `industry_and_market` | 產業現況、前景、政策、技術與市占 |
| `competition` | 競爭者、競爭因素與相對位置 |
| `company_challenges` | 已發生並影響營運的困境 |
| `growth_drivers` | 新產品、擴產、客戶、區域與市場成長 |
| `future_strategy` | 管理層優先事項、擴產、區域拓展與合作 |
| `risks` | 未來可能發生的重大風險 |
| `financial_and_accounting_flags` | 財務與會計警訊 |
| `governance_flags` | 治理風險 |
| `investment_view` | Bull/Base/Bear、催化劑與追蹤指標 |
| `source_evidence` | 頁碼、章節、表格與來源類型 |
| `data_quality` | 完整度、缺失欄位、推論與限制 |

## 關鍵設計原則

1. 同一公司可以同時出現在客戶、供應商、合作夥伴與競爭者。
2. 每個角色分別記錄，不去重成單一實體。
3. 年報明確揭露、模型計算、外部研究與推論必須分開。
4. 詳細研究 JSON 與前端卡片 JSON 分層。
