# 系統架構
系統元件概述及架構決策紀錄

## 設計原則

### 命名慣例
- 優先採用職責型名詞(描述「這裡負責什麼」),避免執行者/工具型名詞,除非該詞在領域內已是通用術語(如 Inference Engine)。

### 資料流分離原則
- Metadata(座標、物件儲存路徑等文字資料)與圖片位元組資料,兩條路徑全程不交叉:
  - Metadata → 走 Queue Store / job payload,隨 job 在 Async Worker 內流轉
  - 圖片位元組 → 只存在於 Object Storage,透過 key/presigned URL 存取
- 圖片位元組實際被搬動的時間點僅兩次:切割時寫入 Object Storage 一次、AI Model 推論時讀取一次,全程不進入 Core Backend 或 Async Worker 的記憶體(RAM)。

### LLM 輸入輸出邊界原則
- 水土審核 Copilot 涉及外部 LLM API 呼叫時,LLM 僅接觸使用者輸入的自然語言,不接觸水井座標或任何可定位到個別水井的資料。
- 查詢結果的呈現(清單、地圖、清冊)全部在系統內部完成,不經過 LLM。若未來需要生成自然語言總結,僅以固定模板代入聚合後的統計數字,不傳遞單筆座標。

---

## 內部

### Frontend (前端)

| 元件 | 職責 |
|---|---|
| Login(登入) | 帳密輸入、送出驗證(無公開註冊) |
| Upload(圖片上傳) | 大檔案(空拍圖)選取、上傳進度、presigned URL 直傳邏輯 |
| Result Review(辨識結果審核) | human in loop,顯示 bounding box、勾選確認/刪除;顯示辨識失敗區域(見 Queue Store - DLQ);觸發 AI 生成模擬圖 |
| Map(地圖檢視) | 載入 NLSC 底圖、顯示已審核標記、詳情查詢 |
| Detection History(辨識記錄) | 唯讀瀏覽辨識紀錄列表 |
| Account(帳號管理) | 管理員對他人帳號 CRUD,僅管理員可見 |
| Copilot(自然語言查詢助理) | 對話式輸入介面,顯示查詢結果清單/地圖,觸發清冊匯出 |
| Timeline(多期時空比對) | 時光滑桿、雙視窗同步比對,顯示跨期水井新增歷程 |
| Perspective Simulation(平視模擬顯示) | 顯示生成式模擬圖,附「AI 生成模擬,僅供參考」提示 |

這是一個內部後台系統,不對外公開、無SEO需求,故以CSR為主

**技術選型**
* Next.js — 保留 file-based routing 的開發便利性;與後端分開、避免codebase邊界模糊
* React — 生態系龐大,地圖渲染、上傳元件等常用套件現成可用,降低造輪子成本;Component-based 架構利於拆分審核列表、地圖標記等元件
* TailwindCSS — 樣式直接寫在 JSX 裡,vibe coding 友善;不用煩惱 class naming跟維護CSS檔案
* Leaflet — 底圖渲染與互動,API簡潔、設定少,react-leaflet 提供成熟的 React + TypeScript 整合;但 WMS/WFS 需要透過外掛支援,目前應該沒有需要;備選:OpenLayers(有原生支援GIS 資料格式、OGC 標準服務)
* TypeScript — 強型別,穩定性高,與後端共用DTO,降低資料格式不一致的風險

---

### Core Backend (主後端)

| 層 | 職責 |
|---|---|
| Controller Layer(控制器層) | 接收 HTTP request、驗證格式、轉發、包裝 response |
| Flow Dispatch(任務流程派發) | 觸發 Async Worker 開始一個 Flow、查詢 Flow 層級整體狀態 |
| Review Workflow(審核流程) | 審核結果的確認/刪除、狀態轉換 |
| Permission Policy(權限管理) | 角色權限判斷(含帳號管理職責併入於此) |
| Data Access(資料存取) | 封裝 PostgreSQL/PostGIS 查詢邏輯;查詢時將座標由 EPSG:3826 轉換為 EPSG:4326(見下方獨立說明) |
| Authentication(登入驗證) | 登入驗證、身份確認 |
| Upload Coordination(上傳協調) | 簽發 presigned URL,讓大檔案繞過主後端直傳 Object Storage |
| Query Translation(自然語言查詢轉譯) | 呼叫 LLM API,將自然語言解析為結構化查詢意圖(JSON);不生成可執行查詢語法 |
| Query Builder(查詢組建) | 依白名單欄位與運算子,將結構化查詢意圖組成參數化 PostGIS 查詢 |
| Export Service(清冊匯出) | 依使用者指定欄位/格式(CSV/XLSX)產生清冊檔案;無指定時採預設模板;僅使用者明確要求時產生 |

**技術選型**
* Nest.js — 與前端分開、避免codebase邊界模糊;模組化架構、依賴注入、一致性高;與Typescript綁定、有官方BullMQ整合
* TypeScript — 強型別,穩定性高,與前端共用DTO

#### Data Access — 座標系統轉換

**這個元件做什麼**

查詢時將 Data Storage 儲存的 EPSG:3826 座標轉換為 EPSG:4326,供前端 Leaflet 顯示。

**為什麼要這樣做**

EPSG:3826 為投影座標系(單位公尺),去重比對等距離運算需要以此為準,若用 EPSG:4326 的度數直接計算會因緯度不同而失真,因此 Data Storage 僅儲存 EPSG:3826,轉換只在最終輸出給前端這一步發生,不在資料庫內多存一份、也不交由前端轉換,避免轉換邏輯分散到多處維護。

**待辦**

無

**注意事項**

未來 GeoJSON/Shapefile 匯出功能同樣從 EPSG:3826 出發、依匯出格式各自轉換。若未來因大量標記點造成即時轉換效能瓶頸,可評估加 EPSG:4326 快取欄位或 database view。

#### Query Translation / Query Builder — 自然語言轉查詢

**這個元件做什麼**

Query Translation 呼叫 LLM API,將使用者輸入的自然語言解析為結構化查詢意圖(如地理範圍、信心度門檻、登記狀態、匯出欄位/格式)。Query Builder 依白名單欄位與運算子,將結構化意圖組成參數化 PostGIS 查詢並執行。

**為什麼要這樣做**

若讓 LLM 直接生成可執行查詢語法(如 SQL 片段),查詢範圍不可控,存在查詢注入風險。限制 LLM 只輸出結構化意圖、由後端白名單邏輯組裝查詢,可以讓查詢範圍完全可控,同時符合 LLM 輸入輸出邊界原則——LLM 全程不接觸水井座標與查詢結果。

**待辦**

- 若查詢需求擴及地區類型不只沿海(如流域、保護區等),Query Translation 對應到外部圖層的規則需評估是否通用化,而非逐一寫死

**注意事項**

- LLM 部署方式:demo/比賽階段採外部 API,正式導入實務端視水土單位硬體資源評估自架開源模型
- 決策紀錄詳見 DECISIONS.md

---

### Tiling service (圖片切割服務)

| 層 | 職責 |
|---|---|
| Streaming Decoder(串流解碼) | 串流讀取 BigTIFF、處理 strip-based 儲存、LZW+Predictor=2 差分還原、邊讀邊釋放記憶體 |
| Tile Splitting(子圖切割) | 將解碼出的橫向長條做垂直累積(Row Buffering)+ 水平切割(Column Slicing),重組成正方形子圖,處理 sliding window 重疊 |
| Tile Uploader(子圖上傳) | 將切好的子圖寫入 Object Storage |
| Tile Manifest Reporting(子圖清單回報) | 收集座標與儲存路徑,組成 metadata manifest 回報給 Async Worker(見下方獨立說明) |

**需要 Tile Splitting的原因:** 檔案為 strip-based 儲存(Block=寬度×1),解碼一次拿到的是橫跨全寬、僅 1 行高的長條,並非正方形。需先垂直累積夠切割高度(如 5000 行),再從累積出的寬版面橫向切出定寬視窗,才能重組成正方形子圖。此重組是解碼完成後獨立的二維視窗運算,不屬於解碼的副產品。

- 讀取水土人員提供的空拍正射影像,格式為 GeoTIFF,由 Pix4Dmapper 拼接輸出
- 已確認規格:BigTIFF、Strip-based 儲存、LZW 壓縮 + Predictor=2、座標系統 EPSG:3826,詳見[NOTES.md](./NOTE.md###空拍圖規格)
- 已知最大檔案 7.91GB(僅為目前已檢查範圍內的最大值,非確認上限),無法一次載入 RAM,需以串流方式逐行讀取
- 依照 5000×5000 尺寸做 Sliding Window 切割,每個切割出的小圖需保留對應的地理座標,供後續辨識結果回貼地圖使用
- 已處理完的區域立即釋放記憶體,避免 RAM 佔用隨檔案大小線性增加

**待確認事項:**
  1. Sliding Window 的實作是否正確處理邊界情況(例如原圖尺寸無法被5000 整除時,最後一塊如何處理:補邊 padding 還是縮小尺寸)
  2. 每個切割出的小圖,座標轉換(像素座標 → EPSG:3826 地理座標)的計算是否正確
  3. 記憶體釋放時機是否確實在每個 window 處理完後執行,而非等到整份檔案讀完才釋放
  4. **切割顆粒度(現行 5000×5000)訂立的原始理由尚未明確**——需向學長確認當初是依據何種考量選定此尺寸(記憶體控制經驗值?UI 顯示需求?模型輸入尺寸?),此答案將直接影響是否需要調整切割顆粒度。
  5. **COG(Cloud-Optimized GeoTIFF)方案的可行性評估**——現行檔案為 strip-based,非 COG(tile-based + overview 分層),不具備直接 range-read 的效率優勢。若評估將原始檔案轉為 COG,搭配模型服務端做 windowed read,可能可以省去「切好子圖並持久化存入 Object Storage」這一步(即 Tile Uploader 這一層),改為推論當下即時讀取、讀完即丟。惟此方案需額外考量:
   - 轉檔本身的時間與資源成本(13.6GB 檔案轉檔耗時待測)
   - 前端 Result Review 頁面顯示 bounding box 疊圖時,若不持久化子圖檔案,需改為即時裁切服務,是否有可接受的延遲
   - COG windowed read 在實際 Object Storage(S3/MinIO)環境下的效能是否確實優於現行手刻方案
   - 模型端一次推論的視窗尺寸,不論是否採 COG,仍受限於模型固定輸入尺寸(如 1024×1024),COG 僅改善「讀取效率」,不改變「模型一次能吃多大範圍」這件事
  6. **切割總耗時尚未實測**——影響是否需要將子圖回報方式改為逐張即時回報(見下方 Tile Manifest Reporting 說明)

**技術選型**
* Go + GDAL(go-gdal binding 或直接透過 CGO 呼叫) — 使用 GDAL 處理底層 TIFF/BigTIFF 解析、LZW 解壓縮與 Predictor 還原,在此基礎上自行實作 5000x5000 的 Sliding Window 切割與座標對應邏輯
  - 無 GIL 限制,可用 goroutine 平行處理多個 strip 的解壓縮與切割
  - encoding/binary 套件對二進位格式解析原生支援,處理 TIFF 檔頭與 IFD 結構較直接
  - 編譯為單一執行檔,部署與跨平台無額外執行環境依賴

#### Tile Manifest Reporting — 子圖清單回報

**這個元件做什麼**

收集座標與儲存路徑,組成 metadata manifest 回報給 Async Worker。切割全部完成後一次性回報所有子圖的 metadata。

**為什麼要這樣做**

地理座標已足以判斷子圖間的空間鄰接關係,供後續去重比對使用,不需額外攜帶原圖網格索引(row, col),避免多存一份衍生資訊造成與地理座標不同步的風險。一次性回報的實作與判定邏輯較單純;管線化(逐張回報)需額外的「切割完成訊號」機制與 Result Aggregation 分階段處理,複雜度較高,在切割耗時未實測前不值得優先做。

**待辦**

實測切割總耗時。若切割時間相對於「N 張子圖辨識總時間」占比顯著,評估改為逐張即時回報的管線化方案,搭配新增「tiling complete + 總數 N」訊號機制作為 Flow 完成判定條件之一,並將 Result Aggregation 拆分為可逐筆處理(座標轉換)與需等待全部到齊才處理(去重、寫入)兩階段。

**注意事項**

一次性回報方案是管線化方案的子集,未來升級不需重構既有架構,只是回報與 job 建立時機由一次性改為逐筆。

---

### Async Worker(三層)+ 佇列儲存

#### Flow Producer(流程建構)

**這個元件做什麼**

接收 Core Backend 觸發,建立 parent-child job 樹。Tiling Service 完成整份原圖切割後才知道總共產生幾張子圖,此時一次建立 N 個 detection child job。判定整個 Flow 是否完成。

**為什麼要這樣做**

子圖總數在切割前無法預知,job 樹必須在切割完成後才能定案,不採預先建立的方式。Flow 完成條件為「N 個 detection job 全部成功或進入 DLQ」,而非要求全部成功,避免單一子圖失敗導致整個 Flow 永遠無法判定完成。

**待辦**

無

**注意事項**

需確保每個 detection job 最終都會落在「成功」或「DLQ」兩個終態之一,否則 Flow 完成判定會卡住。

---

#### Job Processors(任務執行)

**這個元件做什麼**

從佇列取出個別 job,呼叫對應服務(Tiling Service/AI Model)執行。控制對 AI Model 的並發呼叫數量。

**為什麼要這樣做**

並發數上限取決於硬體資源、雲端預算、AI Model 實際吞吐量,無法預先算出精確值,需以保守值起跑、依實測調整。

**待辦**

並發數(concurrency)具體數值待實測後決定。

**注意事項**

無

---

#### Result Aggregation(結果聚合)

**這個元件做什麼**

按順序執行四個子步驟:
1. 座標轉換:像素座標(AI Model 回傳的 tile 內部座標)→ EPSG:3826 地理座標
2. 去重候選比對:找出可能是同一口井的跨子圖重複偵測
3. 合併策略:候選重複配對送人工審核,不自動合併
4. 寫入 Data Storage:整個流程中資料第一次、也是唯一一次寫入

**為什麼要這樣做**

座標轉換需要子圖的地理錨點,隨 Tile Manifest Reporting 流轉,圖片位元組本身不經過此層。去重候選比對下推給 PostGIS 的 `ST_DWithin` 處理,避免在 Async Worker 記憶體內做全域比對造成 OOM,且能利用空間索引提升效能。合併策略交由人工判斷,因為信心分數門檻、IOU 閾值等參數需要實測數據才能訂定,現階段不假設已有答案。

**待辦**
- Sliding Window 重疊寬度尚未定義具體數值
- 「同一口井」的距離判斷門檻,待井的實際尺寸與模型框選誤差實測後訂定

**注意事項**
- 座標轉換只做到 EPSG:3826,不在這層轉 4326(見 Core Backend - Repositories)
- 只處理同一次 Flow 內的跨子圖去重;跨時間比對(是否為新挖的井)由下方 Temporal Change Detection 元件另行處理,兩者比對範圍與時機不同,不合併為同一邏輯
- 去重邏輯不可放進 AI Model:模型每次呼叫只處理一張子圖、無法得知其他子圖結果,放進模型會違反其無狀態、可替換的職責邊界

**技術選型**

* Redis — 如果任務不複雜其實不需要,可以用PostgreSQL手搓;但目前看來一批很多張圖、大圖切小圖、小圖辨識,會有任務依賴關係
* BullMQ — Redis只是底層儲存方案,狀態機、重試機制等實作不想造輪子,拿現有的來用;與Nest.js有官方整合

---

#### Temporal Change Detection(跨期比對)

**這個元件做什麼**

Result Aggregation 完成寫入 Data Storage 後觸發,將本次新確認的水井與歷史紀錄比對,判斷是否為同一位置新出現的水井,支援 4D 時空光軸功能的新增歷程呈現。

**為什麼要這樣做**

與 Result Aggregation 內部的跨子圖去重是不同範圍與時機的問題:內部去重發生在資料寫入 Data Storage **之前**,比對範圍侷限於同一次 Flow 切割出的子圖之間;此元件發生在資料寫入**之後**,比對範圍是全部歷史已確認紀錄。兩者語意不同,不合併為同一邏輯,避免 Result Aggregation 承擔超出其原本職責邊界的比對責任。

比對邏輯同樣下推給 PostGIS 的 `ST_DWithin`,與現有去重比對共用同一套空間運算方式,理由一致(避免記憶體內全域比對造成 OOM、利用空間索引)。

**待辦**

- 「同一口井」跨期距離判斷門檻,待井的實際尺寸與模型框選誤差實測後訂定(與 Result Aggregation 待辦性質相同,但屬不同比對範圍,需分別驗證)

**注意事項**

本元件將原先排除於架構範圍外的跨時間比對能力正式納入,決策修訂紀錄詳見 DECISIONS.md。

**技術選型**

同 Result Aggregation(PostGIS `ST_DWithin`)

---

### Queue Store (佇列儲存)
獨立於 Async Worker 之外的元件,非其內部一層。底層為 Redis,命名為 Queue Store 而非 Task Queue / Job Queue,理由:
- 對齊 Redis 官方自稱 "in-memory data structure store"
- Store(暫時性工作狀態)vs Storage(正式持久紀錄)的語意區分:Redis 遺失僅影響進行中流程重建,不影響已確認資料

**角色定位:被動狀態儲存,非協調決策者**——決策(如「job 該不該重試」「是否全部子圖已完成」)由 Async Worker 判斷,Redis 僅負責存放判斷所需的狀態。

**持久化策略:** 因單張子圖推論成本高(約 30 秒/張)、重建成本高,採 AOF `appendfsync always`(頻率允許,毫秒級延遲相對 30 秒運算可忽略),輔以 RDB 定期快照作異地備份。

**可靠性設計要點:**
- Job 完成狀態透過 processor function 的 return value 交由 BullMQ 原子寫入(結果+完成狀態同一事務),避免「寫結果」與「標記完成」分成兩步驟之間的當機空隙導致重複運算
- `lockDuration` 需略大於單一 job 實際耗時(如 60-90 秒),避免正常處理中的 job 被誤判為 stalled 而提早搶走重新分配
- Job 處理邏輯需具備冪等性:執行前先檢查該 tile 是否已有結果紀錄,避免重跑時產生重複結果
- 建議配置至少一個 Redis replica,搭配 Sentinel 做故障轉移,避免單點硬碟損毀造成資料全失
- 失敗重試機制暫不獨立成層,屬 BullMQ 內建設定值(attempts/backoff),除非未來出現需要客製化錯誤處理路徑的情境
- Job 重試次數用盡後,標記為死信狀態(DLQ),見下方獨立說明

**待確認事項**
1. AOF/RDB 實際設定尚未於系統中落實,待實作後驗證
2. Stalled job 偵測參數(`lockDuration`、`stalledInterval`)需依實測單張子圖耗時調整

**技術選型**
* Redis(同 Async Worker)

#### Dead Letter Queue (DLQ)

**這個元件做什麼**

Job 重試次數用盡後,標記為死信狀態,前端據此顯示該子圖對應區域辨識失敗。

**為什麼要這樣做**

死信是 job 的一種新增狀態,與現有 job 狀態(等待中/處理中/完成)性質相同,是否進 DLQ 由 Job Processors 判斷,Redis 僅存放狀態,符合 Queue Store 既有的被動狀態儲存定位。前端顯示失敗而不做自動重跑,是因為 AI 沒能判斷的區域,同樣交給人工複核,與既有 Review Workflow 的職責一致。

**待辦**

失敗原因分類(暫時性失敗如網路瞬斷/AI Model 過載,vs 永久性失敗如檔案損毀/格式錯誤)現階段不分類,統一以固定 attempts 次數重試到用盡才進 DLQ,待運行後依實際失敗案例評估是否需要分類判斷。

**注意事項**

手動重試觸發的重新排入,會牽動 Flow 完成狀態機、資料唯一寫入原則、去重比對邏輯(比對對象需改為查詢 Data Storage 既有紀錄而非暫存區)、地理錨點保留期限(Queue Store 為暫時性儲存,Flow 完成後 job payload 可能已清除)等多處既有設計,目前不實作,僅做前端顯示。

**技術選型**

無新增,沿用 Queue Store 主體技術(Redis)

---

### Data Storage (資料儲存)
- 儲存使用者、任務、水井、審核紀錄等結構化資料,並支援空間查詢
- 座標統一以 EPSG:3826 儲存,見 Core Backend - Repositories 的座標系統轉換說明
- 水井紀錄需關聯所屬上傳批次(Flow/期別),供 Temporal Change Detection 跨期比對與前端 Timeline 時間軸顯示使用

**技術選型**
* PostgreSQL — 免費,效能與生態完整優於MySQL,有支援PostGIS算法,授權較MySQL寬鬆
* PostGIS — 座標轉換、空間索引和函數、地圖工具整合完整;用於 Result Aggregation / Temporal Change Detection 的去重與跨期比對(`ST_DWithin`)與 Repositories 的座標轉換(`ST_Transform`)

---

### Object Storage (物件儲存)
- 儲存空拍圖、切割後子圖等大型檔案,透過 Presigned URL 模式讓前端/Worker 直接存取,避免主後端負擔檔案傳輸流量
- 底層可為 AWS S3 或自架 MinIO,架構角色統一稱 Object Storage

**子圖保留政策**

僅保留 AI 判定有偵測結果的子圖;完全無偵測結果的子圖於 Flow 完成、人工審核結束後不予保留。理由:無偵測結果的子圖佔儲存空間比例最大,且除疊圖顯示外無其他用途,而疊圖顯示現階段不在實作範圍內。

**待確認事項**
1. 是否有資源能夠存取大量空拍圖、圖資能否上雲端(資料主權/授權限制)
2. 無偵測結果子圖的清除時機與負責元件,待實作階段確定

**技術選型**
* AWS S3 — 快速、不限大小、要的時候再讀,不佔用 RAM 空間
* 備案:若圖資不可上境外雲端,改用自架 S3-compatible 物件儲存(如 MinIO),API 相容故程式碼改動成本低

---

### AI Model (AI模型)
| 層 | 職責 |
|---|---|
| Inference API(推論API) | 接收 Async Worker 請求、驗證格式、回傳結果(對應 FastAPI endpoint) |
| Pre-processing(圖片前處理) | 圖片轉換為模型輸入格式(tensor 等) |
| Inference Engine(模型推論) | 執行模型推論(ONNX Runtime) |
| Post-processing(圖片後處理) | 整理辨識結果,**僅回傳 tile 內部像素座標**,不做地理座標轉換 |

**職責邊界決策:** 座標轉換(EPSG:3826 轉換等)不放在 AI Model 內部,統一交由 Async Worker 的 Result Aggregation 處理。理由:保持 AI Model 為純粹的電腦視覺服務,與地理座標系統解耦,提升未來更換模型/搬遷部署環境時的可替換性。同理,跨子圖去重邏輯也不放在 AI Model,因模型每次呼叫僅處理單一子圖、無法得知其他子圖結果。

**技術選型**
* FastAPI — 支援非同步
* Python(暫定) — 訓練時使用;部署使用的語言還沒確定,沿用的話就不用重寫
* ONNX — 好打包哪裡都能跑的格式,還未確定會放在那裡跑,保留擴充性;有需要的話也可以使用ONNX Runtime

#### Generative Simulation(生成式平視模擬)

**這個元件做什麼**

針對俯視辨識困難的疑似點位,以生成式擴散模型(Diffusion Model)模擬地面平視視角畫面,供審核人員做形態交叉比對參考。使用者於 Result Review 頁面手動觸發,非批次預先生成。

**為什麼要這樣做**

此元件的資源特性與現有偵測用 Inference Engine 不同(GPU 需求、推論耗時以秒至分計),且觸發時機、職責性質(生成模擬 vs 偵測)也不同,獨立為元件而非併入現有 Inference Engine。採使用者觸發而非批次預先生成,因為此功能僅服務於少數疑難點位,批次生成會對多數用不到的結果浪費運算成本,且需額外的閒置資源偵測機制,複雜度高於使用者觸發設計。

**待辦**

- 模型選型、訓練/生成所需資料來源、實作細節,由學長負責評估與實作
- 執行機制建議沿用 Queue Store(BullMQ)排入獨立 job、前端輪詢查詢狀態,與現有辨識流程機制一致,待與學長確認是否採用

**注意事項**

UI 須顯示「AI 生成模擬圖,僅供參考」之免責聲明,列為必要功能需求。生成內容為模型推測畫面,非實際現場影像,僅作輔助形態比對參考,不能取代人工審核之判斷依據。

**技術選型**

待學長評估後補充

---

## 外部

### 地圖套件
供地圖檢視之底圖使用
* NLSC — 台灣官方圖源,座標系統對應 TWD97;基本電子地圖底圖屬「免申請」服務項目;但是還需要確定穩定性、可以打多少API

* 備案
  若 NLSC 穩定性不足,可自架 tile server(如 TileServer GL);
  若圖資可上雲端且有預算,可用商業地圖服務(如 Google Maps Platform);
  OSM 公開 tile server 因官方使用政策明確限制重度使用、可能未經通知即封鎖存取,不適合作為正式產品底圖來源,僅可作開發階段臨時測試

### 地理參照資料(Copilot 地理範圍查詢用)
* NLSC 村里界圖 WFS — 免申請、可即時查詢,用於解析「沿海」等地理範圍限定條件(以行政區界定義範圍,非精確海岸線幾何)。採 Lazy Fetch and Cache:首次用到某區域時才查詢取得,取得後快取供後續同區域查詢重複使用,不做自動同步(理由:行政區界資料變動頻率低,快取新鮮度風險小;即時查詢避免依賴持續同步機制)。取得的幾何資料需以 `ST_Transform` 轉換為 EPSG:3826 後,再與水井座標做 `ST_DWithin` 運算
* Demo 階段僅涵蓋雲林縣範圍;若未來需要精確海岸線距離運算(而非行政區界),需另行確認 NLSC 是否提供對應圖層

**已評估但不採用(跨期比對外部歷史圖資來源)**

評估以下免費外部資源作為 4D 時空光軸功能的歷史圖資輔助來源,結論皆不採用,詳細評估紀錄見 DECISIONS.md:
* 中央研究院 GIS 專題中心「台灣百年歷史地圖」— 典藏涵蓋日治時期至 2000 年代初,無法涵蓋近期
* Sentinel-2 免費衛星影像(EOX 加值服務)— 涵蓋 2016-2021 年,解析度 10 公尺,不足以辨識水井尺度目標
* Google Maps Platform — API 未開放歷史時間點影像存取

結論:免費資源無法同時滿足「近期」與「足夠解析度辨識水井」兩個條件,故跨期比對僅採用系統自有的多期上傳資料(見 Async Worker - Temporal Change Detection)

---