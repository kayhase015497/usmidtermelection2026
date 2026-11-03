# 2026 美國期中選舉即時開票系統｜完整交接文件

> **用途**：讓任何接手者（備援 AI：DeepSeek / Grok / Kimi / 其他模型，或沒有 AI 的同事）在不依賴原開發者與 Claude 的情況下，理解、維運、修改這套系統。
> **資料截止**：2026-11-03（開票日）。程式與資料以 GitHub repo 目前的 `main` 為準，本文件若與程式不一致，**以程式為準**。
> **公開警告**：這個 repo 是公開的（GitHub Pages）。**絕對不要把 Token、密碼、個人聯絡方式寫進本文件或任何檔案。**

---

## 0. 先看這裡（60 秒版）

| 項目 | 內容 |
|---|---|
| 這是什麼 | 中時新聞網「2026 美國期中選舉」專題：即時開票地圖、參眾議院席次計分板、關鍵戰區列表、圖文解說區塊 |
| 前台網址 | `https://usmidtermelection2026.chinatimes.com/`（嵌入 CMS 用 iframe） |
| 後台網址 | `https://usmidtermelection2026.chinatimes.com/admin.html`（編輯室用，需要 GitHub Token） |
| 程式倉庫 | GitHub：`kayhase015497/usmidtermelection2026`（公開），分支 `main` |
| 技術 | 純靜態網站：HTML + CSS + 原生 JavaScript，無框架、無伺服器、無資料庫 |
| 資料怎麼更新 | 編輯室在後台改 → 後台透過 GitHub API 把整份 `data.json` 提交到 `main` → GitHub Pages 自動部署 → 前台每 60 秒重新讀取 |
| 部署 | GitHub Pages（Deploy from a branch：`main` / root），自訂網域 + HTTPS |

**最重要的三個事實**

1. **網站是靜態的，沒有任何 AI 或伺服器也會照常運作。** 開票夜的所有日常操作（更新席次、地圖、戰區、圖文）都是編輯室在後台完成，**不需要 AI**。AI 只在「需要改程式」時才有用。
2. **開票夜鐵則：只動資料，不動程式。** 程式變更一律排在開票前或開票後；非改不可時，走 PR、測試、再合併（見第 12 章）。
3. **所有資料都在 `data.json`，所有變更都是 git 提交。** 任何錯誤都可以從 GitHub 的 History 還原（見第 11 章）。

---

## 1. 專案目標與核心原則

- **受眾是台灣讀者**（這是所有 UI 決策的最高原則）。因此：
  - 不放州長選舉（台灣讀者不熟）、不放候選人姓名。
  - 戰區的民主黨／共和黨得票率**目前隱藏**（人手吃緊，見 15 章），只留「結果」與「開票進度」。
  - 地圖只代表**參議院**結果；眾議院用席次計分板呈現（眾議院以「選區」為單位，無法用州地圖表示）。
  - 用語白話：「本屆無參議員選舉」「計票中」「共和黨勝」。
- **語言**：繁體中文（zh-Hant）。
- **嵌入友善**：要能放進 CMS 的 iframe，也要能日後獨立成滿版網站（所有路徑用相對路徑）。
- **防呆優先**：後台為不熟悉美國選舉的同事設計，能自動算的絕不讓人手填（未決定席次、過半控制黨、最後更新時間）。
- **行動優先**：後台與前台都要在手機 375px 與桌機 1200px 好用。

---

## 2. 系統架構

```
編輯室同事（手機／電腦）
        │  開啟 admin.html，貼上 GitHub Token
        ▼
┌──────────────────────────┐        GitHub REST API (Contents API)
│ admin.html（後台，純前端） │ ─────────────────────────────────────┐
│  · 讀：GET  /contents/data.json                              │
│  · 寫：PUT  /contents/data.json （帶 sha，整份覆蓋）          │
│  · 傳檔：PUT /contents/medias/<檔名>                          │
└──────────────────────────┘                                      ▼
                                                     GitHub repo（main 分支）
                                                     data.json、medias/、index.html …
                                                                    │ push 觸發
                                                                    ▼
                                                      GitHub Pages 自動部署（約 1～5 分鐘）
                                                                    │
                          ┌─────────────────────────────────────────┴───────────┐
                          ▼                                                     ▼
          https://usmidtermelection2026.chinatimes.com/        （DNS：CNAME → kayhase015497.github.io）
          index.html（前台）每 60 秒 fetch data.json 重新渲染
                          │
                          ▼  iframe
                   中時新聞網 CMS 文章頁
```

重點：

- **沒有後端**。「後台」只是另一個靜態頁面，直接用使用者的 GitHub Token 呼叫 GitHub API 改檔案。
- **資料延遲** ≈ 後台存檔 → 部署完成（約 1～2 分鐘）→ 前台下一次輪詢（最多 60 秒）。對外可說「約 2～3 分鐘內更新」。
- 前台抓資料時加 `?_=<時間戳>` 繞過快取；後台抓資料用 `cache:'no-store'` 與 `?_=`，避免 GitHub API 的 60 秒快取讓人拿到舊版（見第 13 章教訓）。

---

## 3. 網址、倉庫、帳號與外部服務

### 3.1 倉庫

| 倉庫 | 用途 |
|---|---|
| `kayhase015497/usmidtermelection2026` | **本專題（現行）**。綁定 `usmidtermelection2026.chinatimes.com` |
| `kayhase015497/war-industry-news` | 張威翔另一個專案「戰爭即工業力」的網站（**與本專題無關**）。其中 `misc-data/us-election-2026*.html` 只剩導向／搬遷提示頁；另保留 `ver_2` 兩層版備份（見 15 章）。**不要在這個 repo 綁定中時網域，不要放 `CNAME`。** |

### 3.2 網域與 DNS

- 網域：`usmidtermelection2026.chinatimes.com`
- DNS（由中時 IT 管理）：**CNAME** `usmidtermelection2026` → `kayhase015497.github.io`
  - 歷史事件：IT 一度設成 **A 記錄**，GitHub 回報 `InvalidARecordError`，HTTPS 無法啟用。**子網域一定要用 CNAME，不能用 A 記錄**。
- GitHub Pages 設定（repo → Settings → Pages）：Source = *Deploy from a branch*、Branch = `main` / `(root)`、Custom domain = `usmidtermelection2026.chinatimes.com`、勾選 *Enforce HTTPS*。
- repo 根目錄的 `CNAME` 檔內容只有一行：`usmidtermelection2026.chinatimes.com`；`.nojekyll` 為空檔（避免 Jekyll 處理）。

### 3.3 外部服務（都是前端直接呼叫，沒有金鑰在 repo 裡）

| 服務 | 用途 | 備註 |
|---|---|---|
| GitHub Pages / GitHub API | 部署、後台讀寫資料 | 後台需要使用者的 **Classic PAT（勾選 `repo`）**，只存在瀏覽器 `sessionStorage`，關閉分頁即清除 |
| jsDelivr CDN | `d3@7`、`topojson-client@3`、`us-atlas@3/states-10m.json`（地圖） | 若載入失敗，地圖會顯示「地圖載入失敗（需要網路連線）」 |
| Google Fonts | Noto Sans TC、Noto Serif TC、Barlow Condensed | 失敗會退回系統字體 |
| Google Tag Manager | PV 統計，容器 `GTM-WHBTRJJ`（由中時 IT 提供） | 只放在前台 `index.html`，**不放後台**；被廣告攔截器擋掉也不影響頁面 |
| microlink.io、allorigins.win | 後台貼報導網址時自動抓標題 | 免費額度有限，失敗時可手動輸入標題 |
| YouTube / Vimeo | 圖文區塊可嵌入影片 | 前台會強制靜音、自動播放、循環、無控制列 |

### 3.4 相關人員（角色）

- 專案擁有者／開發：張威翔（GitHub 帳號 `kayhase015497`）。
- 規劃執行（前台頁尾署名）：楊幼蘭、張威翔、栗筱雯。
- 中時 IT 部門：負責 DNS、GTM 容器。

---

## 4. 倉庫檔案地圖

```
usmidtermelection2026/
├─ index.html      前台（約 1,450 行；CSS、HTML、JS 全在同一個檔案）
├─ admin.html      後台（約 1,550 行；同上）
├─ data.json       全部選舉資料（唯一資料來源）
├─ medias/         圖文區塊素材（mp4/gif/png/webp），後台可直接上傳
│   ├─ chinatimes-logo.png    Hero 左上角 LOGO（200×50 透明 PNG）
│   └─ midterm-basics.webp    資訊圖卡（2000×1126）
├─ docs/HANDOFF.md 本文件
├─ CLAUDE.md       功能細節備忘（給 AI 的專案說明）
├─ CNAME           自訂網域
└─ .nojekyll
```

沒有 build 步驟、沒有 npm。**直接編輯檔案、提交到 `main` 就會部署。**

---

## 5. 資料檔 `data.json` 規格

整個系統的「資料庫」。後台是它的編輯器，前台是它的檢視器。

### 5.1 結構總覽

```jsonc
{
  "meta": {
    "lastUpdated": "2026-11-03T03:33:51-05:00",  // ISO 8601，美東時間（含夏令/冬令偏移）。後台存檔時自動寫入
    "reporting": "已開出 0% 選區",                 // 前台標頭顯示的開票進度文字
    "source": "美聯社、各州選務委員會",
    "note": "即將開票",
    "showLive": false                              // ★ true 才顯示即時開票區塊（見 6.2）
  },
  "senate": {                                      // house 結構相同
    "dem": { "seats": 0, "net": -2, "label": "民主黨" },
    "rep": { "seats": 0, "net": 2,  "label": "共和黨" },
    "ind": { "seats": 0, "net": 0,  "label": "獨立/其他" },
    "uncalled": 100,       // 衍生：total − 三黨席次加總（後台自動算）
    "total": 100,          // 眾議院 435
    "threshold": 51,       // 眾議院 218
    "currentControl": "?"  // 衍生："D"/"R"/"?"（某黨 ≥ threshold 才有）
  },
  "house": { ... },
  "governor": { ... },     // 保留但完全不顯示（台灣讀者原則）
  "states": {              // 50 州，鍵是州縮寫
    "TX": {
      "winner": "R",       // "D" | "R" | "I" | null
      "called": true,      // 是否已確定；地圖只有 called=true 才上色
      "margin": 10.0,      // 領先差距 %，可為 null
      "office": "Senate",
      "noRace": true       // 選填：本屆無參議員選舉（存在才代表「無選舉」）
    }
  },
  "keyRaces": [            // 關鍵戰區
    {
      "id": "PA-S", "state": "PA", "stateName": "賓夕法尼亞州", "office": "Senate",
      "dem": { "name": "", "pct": 49.2, "votes": 0 },   // 得票率：目前隱藏，資料保留
      "rep": { "name": "", "pct": 50.1, "votes": 0 },
      "reporting": 87,      // 開票進度 %
      "called": false, "winner": null,   // winner: "D"|"R"|null；選了勝方會自動 called=true
      "competitive": true   // 前台篩選「競爭選區」用
    }
  ],
  "stories": [             // 圖文內容區塊（由上到下排列）
    {
      "id": "s-muttkbn7",
      "title": "選舉全解析", "text": "一句話說明",
      "src": "medias/xxx.mp4",       // 或完整 https 網址，或 YouTube/Vimeo 連結；副檔名自動判斷種類
      "links": [ { "url": "https://www.chinatimes.com/…", "title": "報導標題" } ],
      "hidden": true                 // 選填：true＝前台不顯示、內容保留
    }
  ]
}
```

### 5.2 欄位規則與陷阱

- **衍生欄位由後台自動維護**：`uncalled`、`currentControl`、`meta.lastUpdated`。**手動改 JSON 時要自己算好**（見第 11 章），否則前台「誰過半」會錯。
- 前台只在 `states[x].called === true` 時才給州上色；`called=false` 一律是「未決定」灰色。後台選了勝方才可選「已確定」。
- `noRace: true` 的州：地圖用**斜線紋理**顯示（見 6.5），並忽略它的 winner/called。2026 年無參議員選舉的 15 州見附錄 A。
- 圖文 `src` 路徑相對於網站根目錄（例如 `medias/abc.mp4`）。前台用 `MEDIA_BASE='./'` 解析。
- 報導連結只接受 `http://` 或 `https://`；前台顯示時另外過濾，避免 `javascript:` 等惡意網址。
- 後台存檔時會丟掉「空白的圖文」與「網址不合法的連結」。
- `governor`、各黨 `net`、`keyRaces[].dem/rep` 的 `name`、`votes` 目前前台都**不顯示**，但資料保留，勿刪。

---

## 6. 前台 `index.html` 功能與行為

### 6.1 版面（由上到下）

1. **Hero**（永遠顯示）：左上 `BREAKING · 2026 MIDTERMS` 小字 → 其下「中時新聞網」LOGO（白色圓角底板，**是連結**，`target="_top"` 回 `https://www.chinatimes.com/`）→ 置中大標「2026美國期中選舉」與副標「川普期中考　國會攻防與台海變數」→ 右上亮暗切換鈕。
2. **「即時開票結果」標頭**（僅開票後）：直播燈、最後更新時間（依讀者所在時區顯示）、開票進度標籤。
3. **參眾議院計分板**（僅開票後）：各黨席次、過半門檻、席次比例條、控制黨標籤。
4. **參議院選情地圖 ＋ 戰區列表**（僅開票後）。
5. **圖文內容區塊**（永遠顯示）：多則，桌機左右交錯排列，手機上下堆疊。
6. **資訊圖卡**（永遠顯示）：`medias/midterm-basics.webp`。
7. 頁尾（來源、立即更新鈕，僅開票後）與**署名**「規劃執行：楊幼蘭、張威翔、栗筱雯」（永遠顯示）。

順序差異：
- 開票前（隱藏即時區塊）：Hero → **資訊圖 → 圖文** → 署名。
- 開票後桌機：Hero → 標頭 → 計分板 → 地圖＋戰區 → 圖文 → 資訊圖 → 頁尾 → 署名。
- 開票後手機：…地圖 → 圖文 → 資訊圖 → 戰區列表 …（以 CSS grid areas 控制）。

### 6.2 開票前隱藏機制（重要）

- 根元素預設帶 class `ue-prelive`，CSS 會隱藏標頭、計分板、地圖、戰區、頁尾。
- `isLive()`：網址參數 `?live=1`（強制顯示）或 `?live=0`（強制隱藏）優先；否則看 `meta.showLive === true`。
- 隱藏時**不會載入地圖資料、不渲染計分板**（頁面較輕）。
- 後台「總覽」最上方有「🔴 即時開票區塊」開關，寫入 `meta.showLive`。**開票當天要記得打開。**
- 預覽／訓練：前台網址加 `?live=1`，只有開那個網址的人看得到。

### 6.3 其他網址參數

| 參數 | 作用 |
|---|---|
| `?theme=dark` / `?theme=light` | 強制亮／暗色（預設亮色；讀者按切換鈕會記在 `localStorage`） |
| `?live=1` / `?live=0` | 見 6.2 |
| `?font=b` / `?font=c` | 預覽替代字體組合（LXGW WenKai TC / Chiron GoRound TC），已定案可移除 |

### 6.4 資料載入與刷新

- `loadData()`：`fetch('./data.json?_='+Date.now())`；失敗用 `getFallbackData()`。
- 每 60 秒自動重抓（`REFRESH_INTERVAL`），「立即更新」鈕可手動抓。
- `renderAll()`：先依 `isLive()` 切換 `ue-prelive`；開票後才畫標頭、計分板、地圖、戰區；圖文僅在內容有變時才重畫（避免影片被重載）。

### 6.5 地圖（D3 + TopoJSON）

- 標題「參議院選情地圖」。每州一色，來源 `states[abbr]`：
  - 民主黨勝＝藍、共和黨勝＝紅、獨立＝紫、**未決定＝實心灰**。
  - **本屆無參議員選舉＝淺底＋斜線紋理**（SVG `<pattern id="ue-hatch">`，斜線在螢幕上固定約 10px 週期，視窗縮放會自動重算；圖例色塊用 CSS 漸層做出相同斜線）。這個設計是為了與實心灰的「未決定」明確區分，且不依賴顏色。
  - 已確定且領先差距 < 4% 的州，不透明度略降（表示膠著）。
- 滑過顯示提示框；點擊彈出該州視窗（結果＋該州在 `keyRaces` 裡的戰區）。無選舉的州顯示說明：「美國參議員任期 6 年，每 2 年改選約三分之一席次，○○州今年沒有輪到」。

### 6.6 戰區列表

- 分頁「參議院／眾議院」依 `keyRaces[].office` 篩選；篩選鈕：全部／競爭選區／已確定／未確定。
- 每條只顯示：州縮寫、州名、結果徽章（計票中／民主黨勝／共和黨勝）、「○○% 選區已開出」。
- 一次只顯示 4 條高度，其餘用捲軸（`RACE_VISIBLE=4`），底部提示「共 N 場，往下滑動查看更多」。
- **得票率與藍紅比例條已隱藏**：`SHOW_RACE_PCT = false`（`index.html` 與 `admin.html` 各一個），改成 `true` 即恢復。

### 6.7 圖文內容區塊（stories）

- 版面：媒體 ＋ 文字，桌機左右交錯，兩者之間有一條金色粗線（圓角、與兩側留空隙）；手機上下堆疊，金線變橫線。無媒體時只有文字。
- 媒體類型自動判斷：`.mp4/.webm/.mov` → 影片；YouTube/Vimeo 連結 → 嵌入；其他 → 圖片／GIF。
- **影片一律純氛圍播放**：靜音、自動播放、循環、無控制列、不可點擊暫停（嵌入影片同理，加參數並關閉指標事件）。iPhone「低耗電模式」會擋自動播放，這是系統限制。
- **報導連結**：
  - `target="_top"`，同分頁跳轉（嵌入 CMS 時，整個 CMS 頁面會跳轉）；**不加 `noreferrer`**，中時站內可看到來源。
  - **按「上一頁」回到原捲動位置**：點連結前把「最靠近畫面頂端的區塊（`data-anchor`）與偏移量」＋展開狀態存進 `sessionStorage`（30 分鐘有效），回來後等內容與圖片載入完再捲回去；使用者一捲動就停止校正；一般重新整理不受影響。
  - **超過 5 條（`LINK_LIMIT`）**：只先顯示前 5 條，其餘收在「**更多報導**」按鈕；展開後按鈕變「收起報導」。
  - **自動輪播**：超過 5 條的圖文，每 3 秒（`ROT_MS`）最上面一條上滑淡出、下一條從下方滑入，循環所有連結（動畫 `ROT_ANIM`=650ms）。規則：
    - 展開「更多報導」就停止並列出全部；收起後恢復。
    - 滑鼠停在列表上（以 `:hover` 判斷）／觸控後 6 秒內暫停；畫面外、分頁在背景時不輪播；各則錯開 0.7 秒。
    - 系統設定「減少動態效果」時不輪播（降級為靜態列表）。
    - **列表高度會鎖定**為「所有可能的 5 條組合中最高的那個」（`rotMeasure`），所以輪播時卡片高度、影片位置**不會跳動**；視窗縮放與字體載入後自動重新量測。展開時解除鎖定。
- 圖文可個別隱藏（`hidden:true`），前台不顯示、內容保留。

### 6.8 其他

- **亮暗主題**：CSS 變數；`<html data-theme>`；亮色預設（暖白紙色底）。
- **字體**：標題與大數字 Noto Serif TC、內文 Noto Sans TC、標籤與數字 Barlow Condensed（CSS 變數 `--ff-display / --ff-body / --ff-label`）。
- **GTM**：`<head>` 一段 `<script>`、`<body>` 開頭一段 `<noscript>`，容器 `GTM-WHBTRJJ`，**保持原樣**。一次進站算一次 PV，自動刷新、切換主題、切換分頁不會重複計算。

---

## 7. 後台 `admin.html` 功能與行為

### 7.1 登入與基本操作

- 第一次開啟要貼 **GitHub Classic Token（勾選 `repo`）**；可按「測試連線」檢查；Token 只在 `sessionStorage`。沒登入只能看、不能存。
- 頂部固定列：標題、同步狀態（「✓ 已同步」「● 有尚未存檔的修改」）、⚙ 工具箱、綠色「⬆ 存檔上線」鈕。快捷鍵 **Ctrl/⌘+S** 存檔。
- ⚙ 工具箱：從 GitHub 重新載入、重新設定 Token、連線診斷、開啟前台、複製嵌入碼、複製 JSON。
- 介面：手機底部分頁列／桌機左側選單，遊戲風格（粗框按鈕、血條式席次條、方塊地圖、底部彈出面板）。

### 7.2 六個分頁

| 分頁 | 功能 |
|---|---|
| **總覽** | ★「🔴 即時開票區塊」開關與預覽按鈕；參眾議院席次摘要；開票進度滑桿（會改寫「已開出 N% 選區」文字）；資料來源、頁面底部備註；最後更新時間（存檔時自動寫入，不用手改） |
| **參議院／眾議院** | 三黨「確認席次」加減鈕（可按住連續加、按久加速）；上方席次條；**未決定席次、過半控制黨自動計算**；三黨加總超過總席次時**禁止存檔** |
| **地圖** | 50 州方塊地圖；點州→底部面板：「有選舉／本屆無選舉」、誰贏（民主/共和/獨立/未定）、是否確定、領先差距；上下州切換；「📋 一鍵套用 2026 參議院沒有改選的 15 州」 |
| **戰區** | 卡片列表；新增戰區（先選院別再點州）；每張卡：**開票進度滑桿、結果（計票中/民主勝/共和勝）、激烈戰區開關、刪除**（得票率欄位目前隱藏） |
| **內容** | 圖文管理：頂端固定「快速前往」導覽列（每則一顆、捲動時標示目前位置）；白色「＋ 新增一個子區塊」；每則有「👁 在前台顯示這一則」開關、標題、文字、報導連結區、媒體區、排序▲▼、刪除 |

### 7.3 報導連結編輯

- **黃色「＋ 新增報導連結（加在最上面）」按鈕在連結區最上方**；新增的空白列出現在最上面並自動聚焦。
- 貼上網址 → 自動補 `https://` → 自動抓標題（microlink，失敗改用 allorigins，去掉「- 中時新聞網」之類後綴）；抓不到可手動輸入；改過標題後再換網址不會被覆蓋；有「↻ 重抓標題」。
- 每條有序號、「⤒ 置頂」、▲、▼、🗑。前台先顯示前 5 條，後台有提示「請把最重要的排前面」。

### 7.4 上傳媒體

- 「📤 上傳檔案」把檔案存到 repo 的 `medias/`（`PUT /contents/medias/<名稱>`），檔名自動加時間戳避免重複。
- 支援 gif / png / jpg / jpeg / webp / mp4 / webm；**上限 25MB**；GIF > 5MB 或其他 > 15MB 會跳警告。
- 上傳完還要按「存檔上線」，網頁才會引用該檔（`stories[].src`）。新檔案在網站上要等部署完成才看得到。
- **影片規格建議**（相容所有手機）：MP4、H.264、yuv420p、無音軌、720p（1280 寬）、約 3,000 kb/s、24～30fps、`+faststart`。例如：
  `ffmpeg -i in.mov -movflags +faststart -pix_fmt yuv420p -vf "scale='min(1280,iw)':-2" -an -c:v libx264 -crf 26 -preset slow out.mp4`
- DaVinci Resolve 輸出：格式選 **MP4**（不是 QuickTime/.mov）、編碼 H.264、品質限制約 3000 Kb/s、取消導出音頻、勾選網絡優化。選到 `.mov`、HEVC、ProRes 會無法上傳或無法播放。

### 7.5 存檔邏輯與多人同時編輯（合併引擎）

這是後台最複雜、也最重要的部分（函式：`pushToGitHub`、`mergeWithRemote`、`merge3`、`mergeList`、`canon`、`finalize`、`normalizeObj`）。

- 載入時記下「當時的伺服器版本」`serverBase`（經正規化）。
- **存檔時永遠先抓 GitHub 上最新版本**（`fetchRemote`，繞過快取）：
  - 若最新版 = 載入時版本 → 直接存檔。
  - 否則做**三方合併**：共同祖先（載入時）、我的版本、別人剛存的版本：
    - 我沒改的欄位 → 採別人的；別人沒改的 → 採我的；**兩人改不同欄位 → 自動合併**。
    - 清單（`stories`、`keyRaces`、`links`）以 `id`／`url` 對應；各自新增的都保留；順序若兩人都動過則視為衝突。
    - **兩人改到同一格 → 跳出「別人剛存了更新」視窗**，用中文標出是哪一格（例如「阿拉斯加｜誰贏」），每項選「別人的」或「我的」（**預設保留別人的**），可「全部用…」或取消（取消後本機修改仍保留）。
  - 衍生欄位（`uncalled`、`currentControl`、`lastUpdated`）不參與衝突，合併後重算。
  - 合併後若席次加總超過總數 → 不存檔並提示重新載入。
- 存檔瞬間又被搶先（HTTP 409/422）→ 自動重抓重試（最多 4 次）。
- 提交訊息：`chore: 更新選舉開票結果 <美東時間> ET`。
- **仍建議開票夜指定一人負責即時資料（席次、地圖、戰區），其他人只改圖文。**

---

## 8. 開票夜 Runbook（操作清單）

### 8.1 開票前（建議在前一天完成）

- [ ] GitHub Pages 設定頁：Custom domain 為綠色「DNS check successful」，*Enforce HTTPS* 已勾選。
- [ ] 打開 `https://usmidtermelection2026.chinatimes.com/`，確認 HTTPS 正常、Hero 與 LOGO、資訊圖、圖文內容都正常。
- [ ] CMS 的 iframe 嵌入碼已換成新網域；若 iframe 有 `sandbox` 屬性，需包含 `allow-top-navigation`（或 `allow-top-navigation-by-user-activation`），否則報導連結與 LOGO 連結會點了沒反應。
- [ ] **後台「地圖」分頁按「一鍵套用 2026 參議院沒有改選的 15 州」→ 存檔上線。**
- [ ] **戰區（`keyRaces`）換成真實名單**：目前資料裡仍有測試用的 PA、NV、WI、AZ，這 4 州 2026 年沒有參議員選舉，必須刪除或改掉（見 14 章）。
- [ ] 清掉測試資料：開票進度文字、`senate/house` 席次歸零、`states` 全部未定。
- [ ] 每位操作同事：用手機實際「改一輪 → 存檔上線」演練一次；重新整理後台（Mac `⌘+Shift+R`）才會載入最新版。
- [ ] 指定「資料操盤手」一人。
- [ ] 統計：GTM 預覽模式確認標籤觸發；GA4 等平台已加入網域。

### 8.2 開票開始

1. 後台「總覽」→ 打開「🔴 即時開票區塊」→「⬆ 存檔上線」→ 約 2 分鐘後前台出現計分板、地圖、戰區。
2. 之後的循環：
   - **參／眾議院分頁**：更新確認席次（各黨 ＋/− 或直接輸入）。
   - **地圖分頁**：點州 → 設誰贏 → 設「已確定」→（選填）領先差距。
   - **戰區分頁**：拉開票進度；出結果時選「民主勝／共和勝」。
   - **總覽**：開票進度滑桿（或直接改文字）。
   - 按「⬆ 存檔上線」。
3. 前台約 **2～3 分鐘**內反映（部署＋60 秒輪詢）。需要立刻確認時，前台頁尾「↻ 立即更新」。

### 8.3 開票結束後

- 視需要打開／隱藏「即時開票區塊」，更新資料來源與備註。
- 保留 `data.json` 的 git 歷史作為存證；不需要刪除任何東西。

### 8.4 開票夜不要做的事

- 不要改 `index.html` / `admin.html`（程式）。
- 不要在舊後台（`war-industry-news` 裡的 `us-election-2026-admin.html`）操作——它只是搬遷提示頁。
- 不要同時有兩人修改同一個州或同一欄位。

---

## 9. 部署、網域、HTTPS、統計

- **部署**：推送到 `main` 即觸發 GitHub Pages 建置（Settings → Pages 顯示 "pages build and deployment"）。通常 1～5 分鐘。連續多次存檔會排隊。
- **HTTPS**：憑證由 GitHub 自動發。DNS 檢查未過時無法啟用／續期，所以 **DNS 必須是 CNAME**。
- **快取**：Pages 資源通常 `max-age=600`。前台資料用時間戳繞過；換圖檔時（同檔名）在 `index.html` 裡的圖片網址加 `?v=<日期>` 版本參數（資訊圖卡目前是 `?v=20261103`）。
- **嵌入碼**：
  ```html
  <iframe src="https://usmidtermelection2026.chinatimes.com/"
    width="100%" height="900" frameborder="0"
    style="border:none;max-width:100%"></iframe>
  ```
  （後台 ⚙ → 複製嵌入碼可得）。iframe 內為內部捲動；如 CMS 支援自動高度，需另外評估。
- **獨立滿版網站**（未來可能）：所有路徑皆為相對路徑，整頁 `max-width:1280px` 置中，直接以網域根目錄部署即可；`MEDIA_BASE` 常數可調整素材路徑。

---

## 10. 常見修改對照表（要改什麼 → 去哪裡）

| 需求 | 位置 |
|---|---|
| 改大標／副標／署名／頁面文字 | `index.html` 內 `.ue-hero`、`.ue-credit`（直接寫在 HTML 裡） |
| 換 LOGO | 覆蓋 `medias/chinatimes-logo.png`（建議 400×100 以上或 SVG）；連結網址在 `.ue-logo-link` |
| 換資訊圖卡 | 覆蓋 `medias/midterm-basics.webp`，並改 `index.html` 圖片網址的 `?v=` |
| 圖文內容、報導連結 | **後台「內容」分頁**（不要改程式） |
| 隱藏／顯示某則圖文 | 後台「內容」→ 「👁 在前台顯示這一則」 |
| 連結輪播間隔、動畫、閾值 | `index.html`：`ROT_MS`、`ROT_ANIM`、`LINK_LIMIT` |
| 恢復戰區得票率 | `index.html` 與 `admin.html` 的 `SHOW_RACE_PCT` 改成 `true` |
| 戰區的列表長度（一次看幾條） | `index.html`：`RACE_VISIBLE` |
| 「無參議員選舉」的州名單 | `admin.html`：`NO_SENATE_2026`（一鍵套用用），或後台地圖逐州設定 |
| 地圖配色、斜線樣式 | `index.html`：CSS 變數 `--c-dem / --c-rep / --c-unc / --c-none-bg / --c-none-line`、函式 `setHatch()` |
| 前台資料刷新頻率 | `index.html`：`REFRESH_INTERVAL` |
| 前台資料路徑、素材路徑 | `index.html`：`DATA_URL`、`MEDIA_BASE` |
| 後台寫入的 repo／檔案 | `admin.html`：`REPO`、`FILE_PATH`（換 repo 時務必同步改） |
| 開票前隱藏即時區塊的判斷 | `index.html`：`isLive()`；CSS `.ue-prelive` |
| 上傳限制、支援格式 | `admin.html`：`MEDIA_EXT`、`uploadMedia()` |
| 標題抓取來源 | `admin.html`：`fetchTitle()`、`cleanTitle()` |

---

## 11. 沒有 AI、沒有後台時的緊急手動操作

> 只需要有 GitHub 帳號（對 repo 有寫入權限）。以下都在瀏覽器完成。

### 11.1 直接在 GitHub 網頁修改資料

1. 開 `https://github.com/kayhase015497/usmidtermelection2026/blob/main/data.json` → 右上角鉛筆（Edit）。
2. 搜尋（Ctrl+F）要改的欄位，改值，注意 JSON 語法（逗號、引號、`true/false/null` 不加引號）。
3. 右上 Commit changes → 直接提交到 `main` → 約 2 分鐘後前台更新。
4. **提交前請確認 JSON 合法**（可貼到任一 JSON 驗證工具）；壞掉的 JSON 會讓前台退回備用資料。

常用修改片段：

```jsonc
// 打開即時開票區塊
"meta": { ..., "showLive": true }

// 一個州：共和黨勝、已確定、領先 10 個百分點
"TX": { "winner": "R", "called": true, "margin": 10.0, "office": "Senate" }

// 一個州：今年沒有參議員選舉
"CA": { "winner": null, "called": false, "margin": null, "office": "Senate", "noRace": true }

// 參議院席次（手動改時要連衍生欄位一起改！）
"senate": {
  "dem": { "seats": 47 }, "rep": { "seats": 49 }, "ind": { "seats": 2 },
  "uncalled": 2,            // = 100 − 47 − 49 − 2
  "currentControl": "?"     // 某黨 ≥ 51 才填 "D" 或 "R"，否則 "?"
}

// 戰區：共和黨勝（要同時 called=true）
{ "id": "GA-S", ..., "reporting": 97, "called": true, "winner": "R" }

// 隱藏某則圖文
{ "id": "s-xxxx", ..., "hidden": true }
```

### 11.2 還原錯誤（回到之前的版本）

1. 開 `data.json` → 右上 **History**。
2. 找到要還原的那次提交（提交訊息 `chore: 更新選舉開票結果 …` 後面有時間）→ 點進去 → `⋯` → *View file*。
3. 按 *Raw*，複製全部內容 → 回到 `data.json` 編輯頁貼上 → 提交。
   （進階：本機 `git revert <提交雜湊>`。）
4. 圖文／連結被蓋掉的情況：歷史版本裡都有；可用比對找回缺少的連結（開票前曾發生並已用此法復原）。

### 11.3 用 GitHub 網頁上傳素材

repo → `medias/` 資料夾 → *Add file* → *Upload files*（單檔建議 ≤ 25MB）→ 提交。之後在後台圖文的媒體欄貼上 `medias/檔名.mp4`，或手動改 `data.json` 的 `src`。

### 11.4 直接用 API（進階／自動化）

```bash
# 讀取（回傳含 sha 與 base64 內容）
curl -s -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/kayhase015497/usmidtermelection2026/contents/data.json?ref=main"

# 寫入：body = {"message": "...", "content": "<base64>", "sha": "<上面拿到的 sha>", "branch": "main"}
# sha 不符會得到 409/422（有人先存了），要重新讀取再合併，不可直接重試覆蓋。
```

`$TOKEN` 是使用者自己的 Classic PAT（`repo` 權限）。**不要把 Token 寫進任何檔案或貼到公開地方。**

---

## 12. 開發與測試方法（給接手的 AI）

### 12.1 工作流程（本專案的約定）

1. 從最新的 `main` 開功能分支（例如 `claude/xxx`）。
2. 改程式 → 本機測試 → 更新 `CLAUDE.md`（若影響行為或設定）。
3. 提交、推送分支 → 開 PR（說明＋測試項目）→ **擁有者同意後合併（squash）**。
4. 合併後 `git pull` 確認 `main` 內容，並告知部署約 2～5 分鐘。
5. 開票夜前後不動程式，除非緊急且有測試。

### 12.2 本機預覽

```bash
cd usmidtermelection2026
python3 -m http.server 8000     # 開 http://localhost:8000/index.html
```

- 前台可直接預覽；`?live=1` 看即時區塊；`?theme=dark`。
- 後台需要有效 GitHub Token 才能存檔；測試時建議用「假的 GitHub API」攔截（見下）。

### 12.3 自動化測試（Playwright 無頭瀏覽器）

本專案所有功能都用 Playwright + Chromium 實測過。重點做法：

- 沙盒常連不到外網：**把 d3、topojson-client、us-atlas 從 npm 載下來，用 `route` 攔截 CDN 請求回傳本機檔案**；Google Fonts 直接 abort（會退回系統字體，版面測試仍有效）。
- 攔截 `data.json` 回傳測試資料；攔截 `api.github.com/.../contents/data.json` 做一個**有 SHA 檢查的假 GitHub**（GET 回 `{sha, content(base64)}`、PUT 比對 sha、不符回 409），就能測存檔與多人合併。
- 視窗寬度至少測 **375px**（手機）與 **1200px**（桌機），亮暗兩種主題，開票前（預設）與開票後（`?live=1`）。
- 驗證版面時**量數字，不要只看截圖**：例如逐畫面影格量卡片高度變動、檢查元素有沒有重疊、整頁高度是否合理（曾用「手機整頁 30,977px」發現 CSS 被切壞）。
- 延遲載入的圖片會在捲動時改變版面高度，量測前先把整頁捲過一遍、等圖載入完。
- 輪播類功能用真實時間等待，並取樣「穩定狀態」（動畫進行中會同時有 6 條）。

### 12.4 程式風格

- 前台 CSS class 一律加前綴 `ue-`（避免嵌入時污染外部樣式）；後台自成一個頁面，無此限制。
- 無框架、無建置工具；**不要引入 npm / 打包**。
- 預設不寫註解，只在「原因不明顯」處寫一行。
- 後台所有可編輯資料都在記憶體的 `data` 物件，改完呼叫 `markDirty()`；需要重畫時呼叫對應的 `renderXxx()`。

---

## 13. 踩過的坑與教訓（務必讀）

1. **用「第一個符合的字串」插入 CSS 會切壞別的規則。** 曾用 `.ue-foot{` 當插入點，結果它剛好出現在另一條規則的選擇器中間，導致「開票前隱藏」失效、卡片寬度變 2px。→ 插入前先確認**唯一匹配**（`count==1`），並用更嚴格的測試（隱藏是否生效、卡片寬度、整頁高度）把關。
2. **多人同時存檔會互相蓋掉。** 後台是整份 `data.json` 覆蓋，曾有同事兩台電腦同時新增大量報導連結而被蓋掉。→ 已實作三方合併；歷史復原可從 git history 找回（見 11.2）。
3. **GitHub API 回應有 60 秒快取**，會讓後台抓到舊 SHA。→ 抓取加 `?_=時間戳` 與 `cache:'no-store'`。
4. **子網域不能用 A 記錄**，GitHub 會回報 `InvalidARecordError`，HTTPS 發不出來。→ 一律 CNAME。
5. **iframe 內的 `target="_top"` 連結**：嵌入頁若有 `sandbox` 屬性而沒給 `allow-top-navigation`，點了沒反應。
6. **輪播時列表高度改變會讓整張卡片跳動**（影片欄位被拉伸）。→ 鎖定列表高度為「所有 5 條組合中最高者」。
7. **`overflow:hidden` 會裁掉 focus 外框**，鎖定高度後改用 `outline-offset:-2px`。
8. **`scrollIntoView` 與 sticky 列**：後台內容分頁的導覽列固定在頂端，跳轉的區塊要設 `scroll-margin-top`。
9. **自動播放影片的限制**：必須 `muted`＋`playsinline`；iOS 低耗電模式仍會擋。
10. **影片上傳灰掉不能選**：副檔名不是 mp4（常見是 DaVinci 輸出成 QuickTime `.mov`）。
11. **Windows/Linux shell 小陷阱**：`pkill -f "<pattern>"` 的 pattern 若出現在目前 shell 的指令列，會把自己殺掉；`git stash` 與後續還原要放在同一段指令裡並檢查結果，以免修改被遺忘在 stash。
12. **合併前一定要在乾淨的 `main` 上重新 `pull`**；同事開票期間會頻繁提交 `data.json`，推送被拒時用 `git pull --rebase origin main` 再推。
13. **LOGO 解析度偏低**（200×50）；高解析螢幕會稍模糊，需要設計提供 ≥400×100 或 SVG。

---

## 14. 已知問題與待辦（開票前請確認）

| 狀態 | 項目 | 說明 |
|---|---|---|
| ⚠️ 必做 | `meta.showLive` 目前為 `false` | 開票當天要在後台打開，前台才有即時區塊 |
| ⚠️ 必做 | `keyRaces` 仍是測試名單 | 含 PA、NV、WI、AZ（2026 無參議員選舉）；請換成真實戰區 |
| ⚠️ 必做 | 一鍵套用 15 州名單 | 名單依公開資訊整理，**請對照美聯社核對**；套用後記得存檔 |
| ⚠️ 必做 | 測試資料清零 | 席次、各州結果、開票進度文字 |
| 💡 建議修 | 尚未開出任何席次時，前台計分板控制黨標籤會顯示「**懸峙議會**」 | 程式在 `boardHTML()`：控制黨 `?` 時顯示「懸峙議會」；開票初期會讓讀者誤解。建議改成「尚無過半」或隱藏。後台已是「尚無過半」 |
| 💡 建議 | 眾議院沒有地圖與戰區 | 眾議院以選區為單位，只用計分板呈現；戰區的「眾議院」分頁目前會顯示「此分類目前無資料」，除非另外新增 `office:"House"` 的戰區 |
| 💡 | 資料檔殘留欄位 | `governor`、各黨 `net`、戰區 `dem/rep.name/votes` 不顯示但保留，勿刪 |
| 💡 | 報導標題自動抓取 | 依賴第三方免費服務，額度或網路問題時需手動輸入 |
| 💡 | LOGO 解析度 | 見 13-13 |
| 💡 | iPhone 低耗電模式 | 影片不會自動播放（系統限制） |

---

## 15. 決策紀錄（為什麼這樣做）

- **只有參議院上地圖（A 方案）**：美國參議院以「州」為單位，一州一個結果，台灣讀者容易理解；眾議院以選區為單位、一州多色，不適合上州地圖。州長選舉與「黨派執政州」不是這次要決定的事，會讓讀者混亂，故不放。
- **「本屆無參議員選舉」獨立呈現**：2026 年約 35 席（33 席例行改選＋ 2 席補選：俄亥俄、佛羅里達）改選，另 15 州沒有輪到；若與「尚未開出」同為灰色，讀者會誤判。→ 斜線紋理。
- **隱藏戰區得票率**：開票夜人手吃緊，同事只需填「開票進度」與「結果」。資料保留、一個旗標可恢復。
- **開票前隱藏即時區塊**：開票前只留 Hero → 資訊圖 → 圖文，開票時用後台開關打開；訓練時用 `?live=1` 預覽。
- **獨立 repo＋獨立網域**：網域只對應選舉專題，不讓張威翔另一個專案的網站被綁在中時網域底下。
- **兩層式專頁（首頁＋6 個子頁）曾做過後來取消**：公司討論後偏好原本的單頁佈局。兩層版備份在 `war-industry-news` repo 的 `misc-data/us-election-2026-ver_2*.html`（含獨立資料檔），原始提交 `51c6af2`。
- **單一 iframe**：整個專題只嵌入一個 iframe；若日後獲准獨立網站，直接用網域根目錄即可。
- **圖文影片純氛圍**：只當氛圍，不需要控制列。
- **連結同分頁跳轉＋回上一頁還原位置**：符合新聞站導流習慣。
- **最後更新時間自動寫入、席次與控制黨自動算**：防呆。
- **圖文與報導連結新增在最上面**：最新的報導排最前面，且前台只先顯示前 5 條。
- **GTM 只放前台**：避免編輯室操作灌水 PV。

---

## 16. 給備援 AI 的接手提示詞（可整段複製）

```
你現在要接手維護「2026 美國期中選舉即時開票系統」。請先完整閱讀 docs/HANDOFF.md 與 CLAUDE.md，再回覆你的理解與要先確認的問題，確認前不要修改任何檔案。

【專案】純靜態網站（HTML/CSS/原生 JS，無框架無建置），repo：kayhase015497/usmidtermelection2026，部署在 GitHub Pages，網域 usmidtermelection2026.chinatimes.com。前台 index.html、後台 admin.html、資料 data.json、素材 medias/。受眾是台灣讀者，語言為繁體中文。

【鐵則】
1. 開票夜只改資料，不改程式；除非使用者明確要求且你已測試。
2. 任何會刪除資料、覆蓋檔案、改寫歷史（force push、reset --hard）、或影響線上的操作，先向使用者確認。
3. 絕不在任何檔案、提交、PR 或回覆中出現 GitHub Token、密碼或個資。repo 是公開的。
4. 小步修改：從最新 main 開分支 → 改 → 測 → 更新 CLAUDE.md → PR → 使用者同意後 squash 合併。不要整檔重寫、不要引入框架或 npm。
5. 手機 375px 與桌機 1200px、亮色與暗色、開票前（預設）與開票後（?live=1）都要測。用數字驗證版面（元素是否重疊、整頁高度、卡片寬度），不要只看截圖。
6. 修改 CSS 時，插入位置必須是唯一匹配；修改 JS 後要檢查語法並實測。
7. 不要動 data.json，除非使用者要求；同事會隨時從後台存檔，直接改會被覆蓋或造成衝突（推送被拒時 git pull --rebase 後再推）。
8. 前台 CSS class 用 ue- 前綴；保持現有風格與命名。
9. 不確定就問，不要猜。遇到與本文件不符的現況，以程式為準並指出差異。

【你要回報的格式】做了什麼、怎麼測的（附數字）、有什麼風險、需要使用者決定什麼。用白話繁體中文，不要用過多術語。
```

建議使用者貼上提示詞後，再附上**這個 repo 的網址**與**這次要做的具體任務**。

---

## 17. 附錄

### A. 2026 年沒有參議員選舉的 15 州（後台「一鍵套用」使用）

亞利桑那 AZ、加州 CA、康乃狄克 CT、夏威夷 HI、印第安納 IN、馬里蘭 MD、密蘇里 MO、內華達 NV、紐約 NY、北達科他 ND、賓夕法尼亞 PA、猶他 UT、佛蒙特 VT、華盛頓 WA、威斯康辛 WI。

有選舉的 35 州＝ 33 州例行改選＋ 2 州補選（俄亥俄 OH、佛羅里達 FL）。**請以美聯社公布名單核對。**

### B. 美國期中選舉基本知識（給不熟悉的接手者）

- **期中選舉**：在總統任期第 2 年舉行，改選國會部分席次。
- **眾議院**：435 席，**全部改選**，任期 2 年；以「選區」為單位，過半門檻 **218**。
- **參議院**：100 席，任期 6 年，每 2 年改選約三分之一（本屆 35 席）；每州 2 席；過半門檻 **51**。
- **勝負**：簡單多數制，在各選區（或州）得票最多者當選，不看全國總票數。
- **共和黨（R）＝紅、民主黨（D）＝藍**；獨立人士（I）多與某黨組成黨團，本系統用紫色。
- **「計票中」與「已確定」**：美聯社宣布勝選（called）才算確定；領先不等於確定。

### C. 術語對照

| 用語 | 意思 |
|---|---|
| Hero | 頁面最上方的主視覺標題區 |
| 即時開票區塊 | 標頭、計分板、地圖、戰區列表、頁尾（由 `meta.showLive` 控制） |
| 戰區 / 關鍵選區 / 搖擺州 | `keyRaces`，競爭激烈的選區列表 |
| 子區塊 / 圖文 / 內容區塊 | `stories` 裡的每一則 |
| 無選舉 | `states[x].noRace=true`（本屆無參議員選舉） |
| 合併引擎 | 後台存檔時的三方合併 |
| 鎖定高度 | 連結輪播時固定列表高度避免跳動 |

### D. 歷史版本與備份

- 兩層式專頁版（ver_2）：`war-industry-news` repo，`misc-data/us-election-2026-ver_2.html`、`…-ver_2-admin.html`、`…-ver_2-data.json`；原始提交 `51c6af2`。
- 本 repo 的每一次修改都有 git 提交與 PR（PR #1 LOGO 連結、#2 隱藏戰區得票率、#3 無選舉斜線、#4 資訊圖卡更新），可由 `git log` 追溯。
