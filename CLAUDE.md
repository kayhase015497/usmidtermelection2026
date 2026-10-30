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
## 版面順序
- 桌機：計分板 → 地圖＋戰區列表 → 圖文區塊（全寬）→ 資訊圖卡（全寬）
- 手機：計分板 → 地圖 → 圖文區塊 → 資訊圖卡 → 戰區列表
- 資訊圖卡：`medias/midterm-basics.webp`（寫死在 `index.html` 的 `.ue-info`，換圖直接覆蓋同名檔即可）

## 圖文區塊的報導連結
- 連結用 `target="_top"` 在同一個分頁跳轉（嵌入 CMS 的 iframe 時，會讓整個 CMS 頁面跳轉，而不是在小框裡開啟）；不加 `noreferrer`，中時站內可看到來源
- 按瀏覽器「上一頁」回來時會回到原本捲動的位置：點連結前把「最靠近畫面頂端的區塊（`data-anchor`）與偏移量」存入 sessionStorage，回來後等內容與圖片載入完再捲回去；使用者一開始捲動就停止校正；只在點連結後 30 分鐘內有效，一般重新整理不受影響
- 後台「內容」分頁頂端有固定的「快速前往」導覽列（每則圖文一顆，隱藏的會變淡，捲動時自動標示目前位置）；每條報導連結除了 ▲▼，左邊還有「⤒ 置頂」
- 圖文可個別隱藏：後台每則有「👁 在前台顯示這一則」開關，關閉會在資料裡存 `stories[].hidden: true`，前台不顯示、內容保留，可隨時打開（「眾議院決戰」已隱藏，內容合併到「參議院關鍵戰」）
- 每則圖文的報導連結超過 5 條（`LINK_LIMIT`）時，前台只先顯示前 5 條，其餘收在「更多報導」按鈕裡，點開展開、再點「收起報導」；展開狀態也會在「點連結後按上一頁」時一併還原。後台排序請把重要的放前面
- 若 CMS 的 iframe 設有 `sandbox` 屬性，需包含 `allow-top-navigation` 或 `allow-top-navigation-by-user-activation`，否則 `_top` 連結會被瀏覽器擋下

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

## 流量統計（GTM）
- `index.html` 內放有 Google Tag Manager（容器 `GTM-WHBTRJJ`，由 IT Bruce 提供）：`<head>` 一段 `<script>`、`<body>` 開頭一段 `<noscript>`，請保持原樣
- **`admin.html` 不放**，避免編輯室操作被算進 PV
- 每次進站算一次 PV；資料每 60 秒的自動更新、切換亮暗色、切換分頁都不會重複計算
- GA4 等統計平台需加入網域 `usmidtermelection2026.chinatimes.com`；上線驗證用 GTM 預覽模式（Tag Assistant）

## 開票前隱藏即時開票介面
- `data.json` 的 `meta.showLive`：`true` 才顯示即時開票區塊（標題下方的「即時開票結果」標頭、計分板、地圖、戰區列表、頁尾）；沒有這個欄位或為 `false` ＝隱藏，前台只顯示 **Hero → 資訊圖 → 圖文內容區塊**
- 隱藏時不會載入地圖資料、不會渲染計分板，頁面較輕；開啟後恢復原本版面（手機：圖文 → 資訊圖 → 戰區列表）
- 後台「總覽」最上方有「🔴 即時開票區塊」開關，開票當天打開並「存檔上線」即可
- 預覽／訓練：前台網址加 `?live=1` 強制顯示（只有開啟該網址的人看得到）；`?live=0` 強制隱藏

## 多人同時使用後台
- 存檔時，後台會先抓 GitHub 上的最新版本，再把「你這次自己改的欄位」疊上去（三方合併：載入時的版本／我的／別人剛存的），所以兩個人改不同地方（例如甲改席次、乙改圖文）不會互相蓋掉
- 兩人改到**同一格**時，會跳出「別人剛存了更新」視窗，逐項選「別人的」或「我的」（預設保留別人的）；可「全部用…」或取消存檔，取消後你的修改仍保留
- 清單（圖文、戰區、報導連結）以 `id`／`url` 對應：各自新增的項目都保留；一方刪除、另一方修改，或兩人排序不同，也會提示選擇
- 存檔瞬間又被別人搶先時會自動重抓重試；合併後若席次加總超過總數，不會存檔並提示重新載入
- 每次存檔仍是一個 git 提交，任何版本可到 GitHub 的 `data.json` → History 找回
- 開票夜建議仍指定一人負責即時資料（席次、地圖、戰區），其他人改圖文

## 地圖（參議院選情）
- 地圖標題為「參議院選情地圖」：每州一色＝該州參議員選舉結果（眾議院以計分板與戰區列表呈現，不上地圖）
- `states.XX.noRace: true` ＝「本屆無參議員選舉」（2026 年約 15 州沒有輪到）：地圖以淡灰色顯示，圖例標示，點擊顯示說明文字；後台地圖分頁可單州切換，或按「一鍵套用 2026 參議院沒有改選的 15 州」（AZ、CA、CT、HI、IN、MD、MO、NV、NY、ND、PA、UT、VT、WA、WI；名單需與美聯社核對）
- 後台圖文「報導連結」每條都有 ▲▼ 排序箭頭與序號，前台依此順序顯示

## 外部資源（CDN）
D3 v7、TopoJSON、US Atlas（jsDelivr）；Google Fonts（Noto Sans TC / Noto Serif TC / Barlow Condensed）

## 來源
由 `kayhase015497/war-industry-news` 的 `misc-data/us-election-2026*.html` 拆分而來（commit `2985861` 之後）。舊 repo 的版本（含 ver_2 兩層版備份）保留不動。
