# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

2026 曼谷 6 天 5 夜自由行 PWA — 零構建、單文件架構的靜態網站。所有 HTML、CSS、JavaScript 集中在 `index.html`。無框架、無依賴、無 package.json。

## Development & Deployment

- **本地開發**: 用任何靜態伺服器開啟 `index.html`（如 `python3 -m http.server`），Service Worker 需 HTTPS 或 localhost
- **部署**: push 到 `main` 分支自動透過 GitHub Actions 部署至 GitHub Pages（`.github/workflows/deploy.yml`）
- **無 build/lint/test 指令** — 零構建架構

## Architecture

### 單文件結構

`index.html` 包含所有程式碼：
- **CSS**（前 ~250 行）: CSS 變數定義色彩系統（金色主題 `#c89b3c`）、分類顏色、毛玻璃效果、卡片動畫
- **HTML**（中段）: Tab 容器 + Dashboard + Day 1-6 頁面骨架
- **JavaScript**（後段）: 資料定義 + 渲染邏輯 + 事件處理

### 核心資料結構

- **`DAYS[]`** — 6 天行程，每天含 `items[]`（時間線項目）和 `budget[]`（預算表）
- **`GLOBAL_WARNS[]`** — 全域行程警告

### 分類系統（Emoji → 類別）

`getCat()` 函數透過 emoji 正則判斷項目類別，對應不同卡片樣式：
- `transport`（藍 `#5ac8fa`）: ✈🚃🚇🚌🚕 等
- `food`（橙 `#ff9500`）: 🍽🍛🍖 等
- `spot`（綠 `#34c759`）: 📸🏛🛍💆🌃 等
- `hotel`（紫 `#af52de`）: 🏨🛁

### 狀態管理

使用 localStorage 持久化：
- `thb_rate` / `thb_rate_time` — 匯率快取（THB → TWD）

### 關鍵渲染函數

| 函數 | 用途 |
|------|------|
| `setDay(n)` | 切換 Tab（0=Dashboard, 1-6=Day） |
| `renderDay(d)` | 渲染單日時間線 + 預算 |
| `renderFoodCard()` | 食記卡片（可展開詳情） |
| `renderSpotCard()` | 景點卡片 |
| `openSheet()` | 開啟備選方案 Bottom Sheet |
| `renderDashboard()` | Dashboard（注意/預算/匯率/不辣指南） |
| `fetchRate()` | 從 er-api.com 取匯率（THB→TWD） |

### PWA 配置

- `manifest.json`: 應用名稱、主題色 `#c89b3c`（金色）、standalone 模式
- `sw.js`: Cache 名 `bangkok-v1`，Network-first 策略，修改 HTML 後需更新版號

## Key Conventions

- **Mobile-first** 設計，max-width 600px 居中
- 卡片用 `data-cat` 屬性標記分類，`has-alts` 標記有備選方案，`has-map` 標記有地圖連結
- Google Maps 連結統一用 `https://www.google.com/maps/search/?api=1&query=` 格式
- 金額一律使用泰銖（฿），匯率換算即時計算為台幣
- 新增行程項目時需在 `DAYS` 陣列對應日期的 `items` 中加入物件，emoji 決定分類
- 餐廳項目含 `nospicy` 欄位，用於「不辣推薦」顯示
