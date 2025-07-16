# Pax 技術部落格 · GitHub Pages 版

這是我使用 React 製作的技術部落格前端專案，透過 GitHub Pages 將其靜態部署，提供文章列表與 Markdown 格式的技術分享內容。

🔗 **網站連結**：[https://pax131072.github.io](https://pax131072.github.io)

---

## 🛠 技術棧

* React + Vite
* React Router v6
* `react-markdown` + `rehype-highlight`（文章渲染與語法高亮）
* GitHub Pages 靜態部署

---

## 📂 專案結構概要

```
{root}/
├── articles/          # markdown 格式文章內容（slug.md）
├── assets/            # 頁面的 css 與 js
├── images             # 文章封面圖片等圖像資源
└── articleList.json   # 文章摘要資料（清單用）
```

---

## ✨ 功能說明

* 📝 **文章列表頁**：從 `articleList.json` 載入摘要資料，顯示封面圖、標題、摘要與標籤
* 📄 **單篇文章頁**：使用 React Router 動態參數載入 markdown 檔案，並進行語法高亮渲染
* 🔄 **fallback 模式**：支援使用者直接輸入 URL 前往 `/article/:slug`，系統會自動從清單補上 metadata
* 🌐 **可部署於 GitHub Pages**：無需後端支援即可運行與存取

---

## 🙋‍♂️ 作者資訊

* GitHub: [@pax131072](https://github.com/pax131072)
* Email: [pax131072@gmail.com](mailto:pax131072@gmail.com)
* LinkedIn: [沛時 張](https://www.linkedin.com/in/%E6%B2%9B%E6%99%82-%E5%BC%B5-15753736a/)
