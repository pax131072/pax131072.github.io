---
title: "API 設計原則與版本控制"
date: "2025-05-29"
imageUrl: "https://picsum.photos/seed/article-3/800/400"
---

良好的 API 設計需兼顧一致性、簡潔性與可擴充性。建議遵循以下原則：

- 使用 RESTful 或 GraphQL 規範
- 明確定義錯誤碼與訊息格式
- 加入 `/v1/`、`/v2/` 等版本路徑區分

這可降低破壞性變更的風險，讓前後端合作更加順暢。
