
You build a SUPPLY-CHAIN BOTTLENECK news feed for a personal investing
dashboard. This is NOT a general market/price-move feed — only include news
that is about a genuine supply-chain BOTTLENECK or a STRUCTURAL change to a
supply chain.

SELECTION (last 24h). Include an item only if it is one of:
  - a bottleneck: shortage, capacity/throughput constraint, lead-time blowout,
    single-source or chokepoint risk, key material/equipment disruption,
    export controls that restrict supply; OR
  - a structural shift that rewrites industry rules or a traditional supply
    chain: a new entrant, domestic substitution breaking a monopoly, or new
    tech that reroutes the chain. Example: "China DUV immersion litho reaches
    volume shipment; SMIC / Hua Hong / CXMT place orders within the year."
Skip generic index moves, earnings beats/misses, and price commentary that
carry no supply-chain angle. Pick the 5-8 most material items.

For EACH item, RESEARCH beyond the headline and build this object. Every field
except date/headline/content is OPTIONAL — use "" or [] when unknown; never
fabricate. Try to fill "chain", "alternatives" and "signals" whenever the
story is about a real bottleneck.

```json
{
  "date": "<YYYY-MM-DD the news refers to>",
  "headline": "<concise headline, Traditional Chinese>",
  "content": "<1-3 sentence summary + why it matters, Traditional Chinese>",
  "tickers": ["<related tickers, e.g. TSM, 2317.TW; [] if none>"],
  "sentiment": "bullish | bearish | neutral  (for the related names)",
  "credibility": "高 | 中 | 低  (source reliability + corroboration)",
  "tags": ["<bottleneck keywords/themes, e.g. CoWoS, ABF substrate>"],
  "effect": "<下游如何被影響、哪些產品成本被墊高，Traditional Chinese>",
  "advise": "<簡短投資建議，Traditional Chinese>",
  "chain": {
    "upstream":   [{"name": "<company/material>", "note": "<how affected>"}],
    "midstream":  [{"name": "<company>", "note": "<how affected>"}],
    "downstream": [{"name": "<company>", "note": "<how affected>"}]
  },
  "alternatives": [
    {"name": "<incumbent>", "share": "<e.g. 95%>", "incumbent": true, "note": "..."},
    {"name": "<substitute company/product>", "share": "<e.g. ~5%>", "note": "..."}
  ],
  "signals": "<clues from recent earnings/revenue/call transcripts that
              corroborate the bottleneck, Traditional Chinese>",
  "sources": [{"title": "<source>", "url": "<link>"}]
}
```

To fill "chain" / "alternatives", trace who is upstream/mid/downstream of the
bottleneck, whether a substitute product or company exists, and the market
share of the incumbent vs the substitute. For "signals", check the last few
quarters' earnings / revenue / call transcripts of the named companies for
corroborating evidence.

STEPS:
1. Write the array to /tmp/news.json as {"items": [ ... ]}.
2. Run: python scripts/publish.py --type news --file /tmp/news.json
   (credentials come from the FIREBASE_SERVICE_ACCOUNT env var).
3. Confirm it printed "Published N news document(s)". Do NOT commit or push —
   the data lives in Firestore and the page reads it live.
4. 必須查看 firebase 之前爬過的供應鏈瓶頸新聞，不要有10天內相似或類似題材、內容的新聞
