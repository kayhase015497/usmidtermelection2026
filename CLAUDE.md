# 2026 美國期中選舉即時開票專題

網址：`https://usmidtermelection2026.chinatimes.com/`（GitHub Pages，repo：`kayhase015497/usmidtermelection2026`）

## 結構
```
/index.html     前台（亮色預設、可切換暗色；D3 地圖、參眾院計分板、戰區列表、圖文區塊）
/admin.html     編輯室後台（手機優先；需 GitHub Classic PAT，勾選 repo）
/data.json      選舉資料（後台推送更新，前台每 60 秒自動讀取）
/medias/        圖文區塊的圖片／GIF／MP4（後台可直接上傳）
/CNAME          自訂網域（Pages 以「Deploy from a branch」部署時生效）
```
所有路徑皆為相對路徑，換網域或放在子路徑（`github.io/usmidtermelection2026/`）都能運作。

## 部署
- Settings → Pages → Source：Deploy from a branch → `main` / `(root)`；Custom domain：`usmidtermelection2026.chinatimes.com`；DNS 通過後勾選 Enforce HTTPS
- DNS（由 IT 設定）：`usmidtermelection2026` CNAME → `kayhase015497.github.io`
- push 到 `main` 後約 2～5 分鐘上線

## 資料格式（data.json）
`meta`（lastUpdated、reporting、source、note）／`senate`、`house`（dem/rep/ind seats、uncalled、total、threshold、currentControl）／`governor`（保留，不顯示）／`states`（各州 winner、called、margin）／`keyRaces`（關鍵選區）／`stories`（圖文區塊：title、text、src、links[{url,title}]）

## 後台
- 手機：底部分頁列；桌機：左側選單（總覽、參議院、眾議院、地圖、戰區、內容）
- 未確定席次、過半控制黨、最後更新時間自動計算；席次加總超過總數時禁止存檔
- 圖文影片一律純氛圍播放（靜音、自動播放、循環、無控制列）；上傳上限 25MB，建議 H.264 MP4、720p、約 3000 Kb/s、無音軌
- 報導連結貼上網址後自動抓標題（microlink → allorigins 備援），可手動修改

## 外部資源（CDN）
D3 v7、TopoJSON、US Atlas（jsDelivr）；Google Fonts（Noto Sans TC / Noto Serif TC / Barlow Condensed）

## 來源
由 `kayhase015497/war-industry-news` 的 `misc-data/us-election-2026*.html` 拆分而來（commit `2985861` 之後）。舊 repo 的版本（含 ver_2 兩層版備份）保留不動。
