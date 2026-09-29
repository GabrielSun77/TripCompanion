
https://gabrielsun77.github.io/TripCompanion/

開發交接文件 · v1
行程夥伴（Trip Companion）開發手冊
個人使用、離線優先的行程規劃與紀錄工具。單一 HTML 檔案的 PWA，無框架、無建置流程，資料全存在使用者裝置本機的 IndexedDB。這份文件記錄目前的功能範圍、資料結構、開發過程中的重大轉折，以及已知的限制，方便下一位開發者接手。

單檔 PWA
Vanilla JS
IndexedDB
Hash Router
無後端
2026-09
目錄
01
專案總覽
02
技術架構
03
檔案結構
04
資料模型
05
功能總覽
06
開發歷程
07
已知限制與技術負債
08
部署方式
09
給下一位開發者的建議
01
專案總覽
給個人使用（非團隊協作）的行程規劃與紀錄工具，同時支援旅遊與出差。核心訴求是「離線也能用」——因為出差時常遇到手機流量有限、工廠內網路被遮罩的狀況，所以整支工具刻意不依賴任何伺服器或線上服務。

目標使用情境涵蓋行程前的打包／SOP／購物清單準備、行程中的快速記錄（筆記、待辦、費用、照片）、交通資訊管理，以及行程結束後的回顧與費用統計。

定位澄清：這不是團隊協作工具，沒有帳號系統、沒有多人共用、沒有雲端同步。每一份資料只存在使用者自己那支手機、那個瀏覽器裡。這個限制是刻意的設計決策，但也是目前最大的技術負債，詳見第 07 節。

02
技術架構
整支 App 是一個自包含的 .html 檔案：CSS 寫在 <style>、JS 寫在 <script>，沒有任何外部套件、沒有 CDN 依賴、沒有 build step。可以直接用 file:// 打開，也可以放到任何靜態網站主機（例如 GitHub Pages）上用 https:// 打開。

資料層
用瀏覽器原生 IndexedDB 存放所有結構化資料，照片以 Canvas 壓縮後轉成 base64 data: URL 一併存進 IndexedDB（不是存檔案系統）。所有 CRUD 都包成 idbGet / idbPut / idbGetAll / idbGetByParent / idbDelete / idbDeleteCascade 幾個小工具函式。

路由
用 location.hash 做 client-side 路由，hashchange 事件觸發 render()，render() 依照 hash 內容呼叫對應的 viewXxx() function 產生 HTML 字串，整段塞進 #app 的 innerHTML。沒有用任何前端框架。

事件處理
全站用事件委派：每個可互動元素標上 data-action（加 data-id / data-id2 帶參數），#app 上只掛一組 onclick / onchange / onsubmit，統一由 handleAction(action, id, id2, el) 這個大型 if-chain 分派。每次 render() 重畫整個畫面後會重新綁定。

離線與安裝
搭配 manifest.json＋sw.js（Service Worker，network-first 策略、離線回退快取）可以變成真正可安裝的 PWA。這兩個檔案是選配的：本機用 file:// 開啟時會安靜地註冊失敗，不影響任何功能。

為什麼是這個架構（而不是原生 App）
這不是最初的技術選擇，是繞了一圈之後的結果，完整經過見第 06 節。簡短版：一開始用 React Native + Expo 開發，但開發過程中的「改功能→驗證」迴圈在使用者的 Windows + Android 環境下持續卡關（Expo Go 連線失敗、EAS Build 相依性問題、APK 裝上手機後打不開且無法取得 crash log），最後決定整個放棄原生方案，改寫成單檔 PWA，換掉的代價是犧牲了系統層級的排程通知（見第 07 節）。

03
檔案結構
目前交付的檔案有兩種形式：

單檔版（給使用者直接存在手機上開）
trip-companion.html	唯一檔案，包含全部 HTML／CSS／JS。可用 file:// 直接開，也可以上傳到任何地方用 https:// 開。
PWA 部署版（給上傳到 GitHub Pages 之類的靜態主機）
index.html	跟上面 trip-companion.html 內容相同，只是改名成 index.html 給靜態主機用。
manifest.json	PWA 名稱、圖示、顏色、display:standalone 設定，讓瀏覽器可以跳出「安裝應用程式」提示。
sw.js	Service Worker。網路優先、離線時退回快取，快取名稱帶版號（trip-companion-v1），要清快取記得改版號。
icon-192.png / icon-512.png	App 圖示，薄荷綠底、白色「行」字，程式生成（PIL，字型 Noto Sans CJK TC Black）。
兩份檔案目前是手動同步的（我每次改功能都是同時改兩邊，或請使用者用其中一份覆蓋）。沒有任何自動化的建置流程去產生 index.html。下一位開發者如果要接手，最優先要做的事情之一就是決定唯一的 source of truth，並考慮引入哪怕最小限度的建置腳本（見第 09 節）。

04
資料模型
IndexedDB 資料庫名稱 tripCompanionDB，目前版本 DB_VERSION = 2。所有 store（除了 trips、templates）都有 parentId 欄位並建了 byParent index，用來模擬「一對多」關聯。新增 store 只要把名字加進 STORES 陣列、bump DB_VERSION，既有使用者的資料庫會自動觸發 onupgradeneeded 補建新 store，不會動到既有資料。

Store	parentId 指向	重點欄位	用途
trips	—	name, type, startDate, endDate, location, timezone, tzManual, reminders[]	一筆行程
templates	—	name, category	可重複使用的清單模板（打包／SOP／待買／待辦）
templateItems	templateId	text	模板裡的每一行項目
checklists	tripId	name, category, templateId	行程套用模板／建立空白清單後產生的一份清單（含SOP）
checklistItems	checklistId	text, checked, order	清單裡的項目；checked 只對非SOP類別有意義
entries	tripId	type, content, amount, currency, category, done, lat, lng, placeName, timezone	快速記錄（筆記／待辦／費用／照片統一存這裡，用 type 區分）
entryPhotos	entryId	uri (base64), sortOrder	記錄附加的照片
scheduleItems	tripId	date, time, title, order	時間軸每天的安排；順序由 order 決定（可拖曳排序），非 time
transports	tripId	type, time, number, note, order	總覽的交通資訊（飛機／計程車／火車／高鐵）
注意 entries 的 lat / lng / placeName 欄位已經存在，但目前寫入時永遠是 null——地點自動定位這個 P1 功能在 RN → PWA 改寫時被跳過了，欄位留著只是為了以後補上時不用動資料結構。詳見第 07 節。

05
功能總覽
依底部分頁與行程內的次分頁列出目前已完成的功能。

行程列表
建立行程（旅遊／出差／自訂），依「進行中／即將出發／已結束」自動排序分類。
刪除行程會連帶刪除該行程下所有清單、記錄、照片、時間軸、交通資訊（cascade delete）。
總覽
行前提醒：不是系統推播，是「打開App時、若符合條件就顯示提醒橫幅」，條件是即將出發（3天內）且非SOP清單還有未勾選項目。
時區顯示與校正：預設即時讀取裝置目前時區（Intl.DateTimeFormat），會隨著使用者移動地區自動更新；系統判斷錯誤時可手動指定（常用時區快選＋自訂輸入），也可以隨時「恢復自動偵測」，不會卡死在單一時區。
交通資訊：可新增飛機／計程車／火車／高鐵四種類型，填時間、班次或車輛資訊、備註，依時間排序顯示，可編輯／刪除。
刪除行程按鈕。
清單（打包／待辦／待買）＋ SOP 流程
一般清單（打包清單／待辦／待買物品）：可從模板套用或建立空白清單，項目可勾選完成、可編輯文字、可刪除。
SOP 流程：刻意跟一般清單分開處理，因為 SOP 的性質是「查詢某件事情該怎麼做的步驟參考」，不是打勾用的待辦。UI 上獨立成自己的區塊，步驟顯示為編號列表（不是checkbox），支援拖曳排序、編輯、刪除。
清單／SOP 都可以命名與重新命名（建立時跳出輸入框，之後也能隨時改名），可以「存回模板」讓下次直接套用同一份內容，也可以直接刪除整份清單。
記錄（快速紀錄）
統一的筆記／待辦／費用輸入介面，可附加照片（拍照或選相簿，Canvas 壓縮後存 base64）。沒填文字但有照片時，自動歸類成「照片」類型。
依分類（全部／筆記／待辦／費用／照片）篩選檢視；在某個分類下按「快速記錄」會直接帶入對應的輸入模式，不用先進「全部」再手動切換。
待辦可直接點擊切換完成狀態；點擊筆記／費用／照片會開啟編輯視窗。
花費
依幣種分別加總行程中所有費用類型記錄，不做匯率換算。
時間軸
依行程天數列出每天的安排，每筆安排有時間（純標示文字）跟標題，可編輯、刪除。
排序由拖曳決定（order 欄位），不是自動依時間排序——因為改成手動排序後，時間欄位變成單純顯示用途，不影響順序。
回顧
統計筆記數、待辦完成度、照片數、費用總計；列出行程套用過的清單，可個別「存回模板」。
模板
四個分類共用同一套模板機制：打包清單／SOP流程／待買物品／待辦。
模板內容是「每行一項」的純文字編輯，套用到行程時才展開成獨立的清單項目。
編輯既有模板時可以刪除模板（新建立時不會顯示刪除按鈕）。
06
開發歷程與重大決策
這是一個真的走過彎路的專案，記錄下來是希望下一位開發者不要重複同一輪嘗試。

Phase 0 · 需求
PRD 確認：個人用、旅遊＋出差通用、離線優先
先寫了完整 PRD（P0/P1/P2 分級），P0 涵蓋行程管理、可重用清單模板、統一快速記錄、照片、購物清單、費用記錄、本機通知、離線架構；P1 涵蓋時間軸規劃、行程回顧、地點自動定位、多幣種顯示、搜尋篩選。語音輸入原本在規劃內，後來使用者主動決定放棄（做到 P1 為止，不做語音）。

Phase 1 · 原生方案（後來整個放棄）
React Native + Expo，完整實作但卡在部署驗證迴圈
選定 Android 為目標平台，用 Expo Router + expo-sqlite + expo-notifications + expo-image-picker + expo-location 完整實作了 P0+P1 全部功能，TypeScript 檢查與 expo export 都過。問題出在「改完功能要怎麼驗證」：

Windows 環境下 npm/npx 找不到、PowerShell 執行原則擋 .ps1，換成可攜式 Node.js 才解決。
Expo Go 掃 QR Code 連線失敗（同 Wi-Fi、防火牆、手機熱點都試過），改用 EAS Build 打包 APK。
EAS Build 一開始在「Install dependencies」階段失敗（npm peer-dependency 衝突），加 .npmrc（legacy-peer-deps=true）才過；另外還遇到 git 環境問題，用 EAS_NO_VCS=1 繞過。
APK build 成功、下載安裝到手機後，打開就閃退，且沒有 ADB／Android Studio 無法取得 crash log，完全無法診斷。
退而求其次改用 expo start --tunnel（透過 ngrok），又卡在 ngrok 現在需要帳號＋authtoken才能用。
結論：不是某一個 bug，是每一種「改功能→在手機上驗證」的路徑都各自卡關，且大多跟這位使用者當下的環境（Windows、沒有 Android Studio、公司網路限制）強相關，不是程式碼問題。

Phase 2 · 策略轉向
放棄 React Native，改寫成單檔 PWA
使用者明確決定「改方法開發這支 tool」而不是換一種除錯方式，於是整個技術棧換成單一 HTML 檔案：不需要 npm、不需要 build、不需要 ADB、不需要 EAS、不需要網路才能連上開發伺服器。唯一的代價是接受「系統排程通知」改成「打開App時顯示提醒橫幅」——這是使用者事先確認接受的取捨。整支 App 原封不動照搬 P0+P1 的功能範圍重寫一遍。

Phase 3 · 迭代修正
連續幾輪真實環境的 bug 修正
PWA 版本上線後經過幾輪在使用者實機上測試才發現、且在桌面模擬環境不容易重現的問題：

Modal 被同時掛進 #app 跟 body 兩層，造成點擊事件被兩套委派監聽器各處理一次 → 套用模板／建立清單時會建立出兩份重複的清單。
用 innerHTML 注入 <script> 標籤處理分類切換按鈕跟 Enter 新增項目——瀏覽器不會執行透過 innerHTML 插入的 <script>，導致這兩個互動完全失效。改成用既有的事件委派 data-action 機制處理。
清單新增項目的輸入框跟按鈕放在同一個 flex row，因為 .btn 預設 width:100%，導致按鈕把輸入框擠成幾乎看不見的一條細線——這個問題光看程式碼看不出來，是使用者截圖才抓到的排版 bug。
手機虛擬鍵盤（尤其中文輸入法）在非 <form> 的文字輸入框上，Enter 鍵行為不可靠；改成把輸入框包進標準 <form>，用瀏覽器原生的 submit 事件統一處理 Enter 跟按鈕點擊。
這幾個 bug 有一個共通點：都在桌面端用 Node.js + jsdom 模擬測試時不會發作（尤其排版擠壞、觸控／IME 行為），只有在使用者的實機上才會出現。下一位開發者接手時要有心理準備：這支 App 的正確性驗證，最終還是要有人拿真手機點過一輪。

Phase 4 · 功能擴充
SOP 獨立、編輯功能、拖曳排序、交通資訊、時區校正
穩定之後陸續加了幾個功能：把 SOP 從「打勾清單」的邏輯裡拆出來獨立處理（見第05節）；清單項目跟時間軸安排補上編輯功能（原本只有新增／刪除）；時間軸跟 SOP 步驟加上拖曳排序（用 Pointer Events 手刻，桌面／手機通用，沒有依賴任何拖曳套件）；總覽加上交通資訊區塊（新增 transports store）；時區改成預設自動偵測、可手動校正、隨時可恢復自動；模板補上刪除功能。

Phase 5 · 視覺與發布
主題色反覆調整、加上 PWA 安裝能力
介面色調經過三次調整：淺色薄荷綠 → 暗色系＋薄荷綠 → 淺色淡橘＋薄荷綠（目前版本）。另外補上 manifest.json、sw.js、App 圖示，讓這支 App 可以上傳到 GitHub Pages 之類的靜態主機，變成真正可以「安裝」到手機主畫面、支援離線快取的 PWA，而不只是一個要手動傳輸的本機 HTML 檔案。

07
已知限制與技術負債
高風險 沒有任何備份機制
所有資料只存在單一裝置、單一瀏覽器的 IndexedDB。使用者清除瀏覽資料、換手機、或瀏覽器把網站資料判定為久未使用而清掉，資料就永久消失，沒有任何匯出／匯入或雲端備份可以救回來。這是目前最大的技術負債，見第09節建議。

通知是「打開App才提醒」
不是系統層級推播通知。使用者已知情並接受這個取捨（PWA的固有限制，尤其 file:// 開啟時完全無法用 Service Worker／Push API）。

地點自動定位（P1）未實作
entries 資料結構有 lat／lng／placeName 欄位，但沒有任何程式碼呼叫 navigator.geolocation 或做 reverse geocoding。PRD 原本規劃的功能，在 RN→PWA 改寫時被跳過，沒有被記錄成一個明確的「延後」決策。

多幣種不做匯率換算
花費分頁只依幣種分別加總顯示，PRD 裡有備註過可以之後加，目前維持原樣。

拖曳排序未經真機廣泛測試
時間軸／SOP 的拖曳排序用 Pointer Events 手刻，邏輯經過桌面模擬環境驗證（事件綁定、資料持久化都正常），但因為模擬環境沒有真實版面座標，實際拖曳手感（跳動、判定時機）沒有在真機上驗證過。

單檔／PWA 版雙份維護
trip-companion.html（給單檔使用）跟 index.html（給 GitHub Pages）內容需要手動保持同步，沒有建置腳本。

其他較小的已知缺口
日期輸入沒有原生日期選擇器，靠瀏覽器 <input type="date"> 的預設 UI。
模板列表本身（viewTemplatesList）目前只能點進單一模板編輯畫面刪除，列表頁沒有直接的刪除入口。
沒有任何自動化測試套件；目前的正確性驗證是開發過程中用 Node.js + jsdom + fake-indexeddb 手寫的一次性模擬腳本，沒有留在專案裡持久化成可重複執行的測試。
08
部署方式
方式一：單檔本機使用（目前使用者主要用法）
把 trip-companion.html 傳到手機（USB／LINE傳給自己／Google Drive都測過可行，瀏覽器直接下載偶爾會卡住不建議），用手機瀏覽器開啟，可以「加到主畫面」產生捷徑。缺點：這只是捷徑，不是真正安裝的PWA，沒有離線快取、沒有安裝提示。

方式二：GitHub Pages（PWA，可安裝、離線可用）
repo 根目錄
├── index.html      ← trip-companion.html 內容，改檔名
├── manifest.json
├── sw.js
├── icon-192.png
└── icon-512.png
上傳到 public repository 後，到 repo 的 Settings → Pages，Source 選 Deploy from a branch，Branch 選 main / root，存檔後會拿到 https://帳號.github.io/repo名稱/ 這樣的網址。用 Android Chrome 打開該網址即會出現「安裝應用程式」提示。

免費帳號的 GitHub Pages 只能用 public repository。網址雖然公開，但打開的只是空的 App 介面，使用者的行程資料仍然只存在自己手機的 IndexedDB 裡，不會透過這個網站被任何人看到或存取。

09
給下一位開發者的建議
先解決備份／匯出問題。這是目前唯一真正「會弄丟使用者資料」的風險。最小可行方案：加一個「匯出成 JSON 檔」跟「匯入 JSON 檔」的功能（把整個 IndexedDB dump 成一個檔案，用 <a download> 或 File System Access API 存下來），不需要伺服器就能做。
決定單檔版跟 PWA 版的 source of truth。目前是手動同步兩份幾乎一樣的 HTML，建議至少寫一個幾行的腳本（Node.js 或 shell）在發布前自動從一份產生另一份，避免改壞其中一份忘記同步。
補地點自動定位（P1 遺漏項）。資料結構已經留好欄位，只差呼叫 navigator.geolocation.getCurrentPosition 跟一個 reverse geocoding API（要注意這一步需要網路，跟「離線優先」的定位不完全相容，需要設計成「有網路時才選填」的體驗）。
把手寫的模擬測試腳本留下來變成真正的測試。開發過程中大量使用 jsdom + fake-indexeddb 模擬瀏覽器環境驗證邏輯（但測不出真實觸控／排版問題），建議整理成專案裡的 tests/ 目錄，之後每次改動至少能防止邏輯層的回歸。
拖曳排序找機會上真機測一次。邏輯是對的，但手感沒人驗證過，第一次真機測試時特別留意這個功能。
模板列表頁補上刪除入口。目前只能進到編輯畫面才能刪，小改動但體驗缺
