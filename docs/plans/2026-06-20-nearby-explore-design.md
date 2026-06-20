# 「附近吃逛」分頁設計

> 日期：2026-06-20
> 範圍：在 Dashboard（day0）新增第 6 個子頁 `🧭 附近`，以 GPS 定位列出 500m 內的候選地點（吃飯／逛街／景點／其他），協助「不知道去哪、不知道吃什麼」時就近決策。複用既有米其林地圖引擎。

## 目標與情境

- 使用情境：在曼谷當下打開 App → 進「總覽 → 🧭 附近」→ 按 🧑 定位 → 下方清單以**距離最近**排序，只列 **500m 內**的候選 → 選一個去逛街或吃飯。
- 與米其林分頁的關係：沿用其地圖 + GPS + `haversine` 距離排序的模式，但資料換成**自訂候選清單**，並加上**硬性 500m 範圍**與**地標參考點**。

## 關鍵決策（已與使用者確認）

| 主題 | 決策 |
|------|------|
| 資料來源 | 獨立維護的候選清單 `NB_PLACES[]`；另含地標錨點 `NB_LANDMARKS[]`（飯店、機場、鄭王廟、大皇宮、洽圖洽市集）幫助判斷相對位置 |
| 地標角色 | **只在地圖當參考點**，不進「最近清單」、不參與 500m 判定 |
| 分類 | 🍽️ 吃飯(food) / 🛒 逛街(shop) / 📷 景點(spot) / ✨ 其他(other，涵蓋按摩/酒吧/咖啡)，上方提供多選 toggle 篩選 |
| 分頁位置 | Dashboard（day0）第 6 個子頁 |
| 500m 空清單 | 顯示最近 N（5）筆並提示「500m 內無候選，以下為最近 5 個」 |
| 定位方式 | **只用 GPS**（不做手動插 pin） |

## 資料模型

新增於 `MICHELIN_DATA` 區塊之後：

```js
const NB_CATS = {
  food:  {emoji:'🍽️', label:'吃飯', color:'#ff9500'},
  shop:  {emoji:'🛒', label:'逛街', color:'#34c759'},
  spot:  {emoji:'📷', label:'景點', color:'#5ac8fa'},
  other: {emoji:'✨', label:'其他', color:'#c89b3c'},
};

// 候選清單（會進 500m 排序，日後自行維護擴充）
const NB_PLACES = [
  // {name, cat:'food'|'shop'|'spot'|'other', lat, lng, area, note, pid}
  // 必填 name/cat/lat/lng；選填 area/note/pid（有 pid 才能彈 Place Card）
];

// 地標錨點（只在地圖當參考，不進清單）
const NB_LANDMARKS = [
  {name:'素萬那普機場',  emoji:'✈️', lat:13.6900, lng:100.7501},
  {name:'大皇宮',       emoji:'🏛', lat:13.7500, lng:100.4914},
  {name:'鄭王廟',       emoji:'⛩',  lat:13.7437, lng:100.4888},
  {name:'洽圖洽市集',    emoji:'🛍', lat:13.7999, lng:100.5509},
  {name:'Prestige 飯店', emoji:'🏨', lat:13.7413, lng:100.5390},
  {name:'Kimpton 飯店',  emoji:'🏨', lat:13.7376, lng:100.5435},
];
```

地標座標沿用 `PLACES[]` 中已驗證的經緯度。

## 複用既有資產

- 純函式：`haversine()`、`fmtDist()`、`mapUrl()`、`showPlaceCard()`、`onMarkerTap()`
- CSS：`.mich-card`、`.mich-pill`、`.map-legend`、`.map-locate-btn`、`gm-loc-dot`
- 命名空間：新增獨立 `nb*` 狀態與函式，平行於米其林，**互不干擾**，米其林分頁原樣不動。

## UI 版面（Dashboard 子頁）

子頁列：`⚠️注意 💰預算 💱匯率 🔤翻譯 🍽米其林 🧭附近`（`#day0 .day-subtabs` 已支援橫向捲動）。

由上到下：
1. 標題 `🧭 附近吃逛` + 一行說明「按 🧑 定位，列出 500m 內的候選」
2. 分類篩選 pill（多選 toggle，全不選 = 全部）：`[全部] [🍽️吃飯] [🛒逛街] [📷景點] [✨其他]`
3. 圖例：候選四色點 + 「◇ 地標」灰色標示
4. 地圖 `#nbmap`（右上角 🧑 GPS 按鈕；定位後畫半徑 500m 金色圈）
5. 計數列（三態文案，見下）
6. 結果清單 `.nb-list`（距離排序卡片）

結果卡片：左側分類 emoji + 名稱、分類 · area、距離 badge（`📍 ${fmtDist}`）、Google Maps 導航連結；有 `pid` 時點卡彈 `showPlaceCard`。

## 距離邏輯與資料流

`filterNearby()`（於「定位完成」「切換分類」時觸發）：

```js
function filterNearby(){
  let list = NB_PLACES.filter(p => activeCats.size===0 || activeCats.has(p.cat));
  if(!nbUserPos){ renderList(list, {located:false}); return; }          // 未定位
  list = list.map(p => ({...p, _d: haversine(nbUserPos.lat,nbUserPos.lng,p.lat,p.lng)}))
             .sort((a,b)=>a._d-b._d);
  const within = list.filter(p => p._d <= 500);
  if(within.length) renderList(within, {mode:'in500', n:within.length});
  else              renderList(list.slice(0,5), {mode:'fallback'});
}
```

計數列文案三態：
- 未定位：`按 🧑 定位，列出 500m 內的候選`
- 有結果：`📍 500m 內 N 個 · 依距離排序`
- 空 fallback：`⚠️ 500m 內無候選 · 以下為最近 5 個`（卡片仍顯示實際距離）

`NB_LANDMARKS` 完全不參與此運算。

進入子頁：比照米其林自動靜默請求一次定位（`nbRequestLocation()`）即列出 500m 內候選；失敗／未授權則顯示提示，🧑 按鈕可隨時重新定位。

## 地圖標記

- **候選 pin**：`PinElement`，背景色 = `NB_CATS[cat].color`，白邊。點擊 → 有 `pid` 彈 `showPlaceCard`，否則開 Google Maps。
- **地標錨點**：`AdvancedMarkerElement` + 自訂 content（半透明灰底圓角膠囊 + emoji），視覺上刻意異於候選；點擊直接開 Google Maps。
- **你的位置**：`gm-loc-dot` 藍點 + 精度圈 + 半徑 500m 金色圈（`--accent`）。
- **重繪**：`updateNbMarkers(list)` 只重畫候選 marker；地標 marker 初始化畫一次後不動；切換分類只變候選。
- **fitBounds**：定位後框住「你的位置 + 要顯示的候選」。

## 程式碼整合點（皆在 `index.html`）

1. 資料定義加在 `MICHELIN_DATA` 之後。
2. `renderDashboard()`：子頁列加 `🧭 附近`；新增 `<div class="tab-section" data-section="nearby">` 容器。
3. 子頁切換 click handler：`if(sec==='nearby'){setTimeout(()=>{renderNbFilters();filterNearby();nbRequestLocation();},100);}`
4. 新函式群：`nbState`、`renderNbFilters()`、`filterNearby()`、`renderList()`、`initNbMap()`、`updateNbMarkers()`、`nbRequestLocation()`，置於米其林函式群之後。
5. CSS：複用米其林既有類別；僅新增少量 `.nb-*`（分類 emoji、地標膠囊 marker）。
6. **SW 版本**：`sw.js` 的 `bangkok-v22` → `bangkok-v23`。

## 測試（手動）

- `python3 -m http.server` → localhost → 總覽 → 🧭附近。
- DevTools → Sensors 模擬 GPS（例如 Chit Lom 13.7445,100.5420）驗證 500m 排序。
- 驗證項：分類 toggle 即時刷新、500m 空清單 fallback 文案、地標不進清單、候選彈 Place Card、地圖 fitBounds + 500m 圈。

## 資料維護

候選清單由使用者自行在 `NB_PLACES[]` 增修；必填 `name/cat/lat/lng`，選填 `area/note/pid`（加 `pid` 才有照片卡）。

**重要原則：候選清單不放已在行程裡的餐廳/景點**，以「行程外的逛街選項」為主（供臨時不知去哪時就近挑選）。初版放 8 個行程外逛街商場（Siam Paragon／Siam Center／MBK／Platinum／Pratunam／Terminal 21／EmQuartier／Emporium）。地標（飯店/機場/大皇宮/鄭王廟/洽圖洽）維持在 `NB_LANDMARKS[]` 僅作地圖參考點。
