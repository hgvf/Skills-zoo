# Website Card Rendering Rules

完整研究 JSON 用於資料庫；網站卡片只呈現最重要內容。

## 1. 卡片順序

1. 標題與一句話結論
2. 關鍵字／題材
3. 公司與產業定位
4. 主力產品
5. 新產品與研發
6. 營收結構
7. 客戶與地區
8. 供應鏈
9. 產能與擴產
10. 競爭格局
11. 公司目前困境
12. 產業前景與成長動能
13. 財務與營運重點
14. 風險與紅旗
15. 投資判斷

## 2. 卡片顯示限制

| 類型 | 最多項目 |
|---|---:|
| Tags | 8 |
| 主力產品 | 8 |
| 新量產產品 | 6 |
| 過去一年研發成果 | 10 |
| 進行中研發 | 10 |
| 實名客戶 | 10 |
| 地區 | 8 |
| 供應鏈節點 | 12 |
| 競爭者 | 10 |
| 公司困境 | 8 |
| 成長驅動 | 8 |
| 風險 | 8 |
| 催化劑 | 8 |

## 3. 卡片文字規則

- 標題：45 個中文字內。
- 一句話摘要：80 個中文字內。
- 每個 item：100 個中文字內。
- 一個 item 只說一件事。
- 優先保留數字、產品名、公司名、時間。
- 不使用「表現亮眼」、「未來可期」等空泛語句。

## 4. 優先排序

前端顯示順序依：

1. 對未來營收或毛利影響
2. 是否有明確公司或產品名稱
3. 是否有量化證據
4. 是否為近 1～3 年催化劑
5. 證據可信度

## 5. 卡片狀態

每張卡片可加：

- `positive`
- `negative`
- `mixed`
- `neutral`

不得只因管理層語氣樂觀就標記 positive。

## 6. 建議前端轉換格式

```json
{
  "header": {
    "date": "YYYY-MM-DD",
    "ticker": "string|null",
    "company_name": "string",
    "stance": "bullish|slightly_bullish|neutral|slightly_bearish|bearish",
    "confidence": "high|medium|low",
    "title": "string"
  },
  "summary": "string",
  "tags": ["string"],
  "cards": [
    {
      "type": "products|rd|revenue_mix|customers|supply_chain|capacity|competition|challenges|outlook|financials|risks|investment_view",
      "title": "string",
      "status": "positive|negative|mixed|neutral",
      "items": [
        {
          "label": "string|null",
          "text": "string",
          "metric": "string|null",
          "source_reference": "string|null"
        }
      ]
    }
  ]
}
```
