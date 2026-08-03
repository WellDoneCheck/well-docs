# 系統架構
系統元件概述及架構決策紀錄

## 設計原則
 
### 命名慣例
- 命名對應「架構角色/職責」,不用框架具體構件名稱,也不用產品願景名稱。
- 優先採用職責型名詞(描述「這裡負責什麼」),避免執行者/工具型名詞,除非該詞在領域內已是通用術語(如 Inference Engine)。
### 資料流分離原則
- Metadata(座標、物件儲存路徑等文字資料)與圖片位元組資料,兩條路徑全程不交叉:
  - Metadata → 走 Queue Store / job payload,隨 job 在 Async Worker 內流轉
  - 圖片位元組 → 只存在於 Object Storage,透過 key/presigned URL 存取
- 圖片位元組實際被搬動的時間點僅兩次:切割時寫入 Object Storage 一次、AI Model 推論時讀取一次,全程不進入 Core Backend 或 Async Worker 的記憶體(RAM)。

---

## 內部

### Frontend (前端)

| 元件 | 職責 |
|---|---|
| Login(登入) | 帳密輸入、送出驗證（無公開註冊） |
| Upload(圖片上傳) | 大檔案(空拍圖)選取、上傳進度、presigned URL 直傳邏輯 |
| Result Review(辨識結果審核) | human in loop，顯示 bounding box、勾選確認/刪除 |
| Map(地圖檢視) | 載入 NLSC 底圖、顯示已審核標記、詳情查詢 |
| Detection History(辨識記錄) | 唯讀瀏覽辨識紀錄列表 |
| Account(帳號管理) | 管理員對他人帳號 CRUD,僅管理員可見 |

這是一個內部後台系統，不對外公開、無SEO需求，故以CSR為主

#### 技術選型
* Next.js
  保留 file-based routing 的開發便利性
  與後端分開、避免codebase邊界模糊
* React 
  生態系龐大，地圖渲染、上傳元件等常用套件現成可用，降低造輪子成本；
  Component-based 架構利於拆分審核列表、地圖標記等元件，可以重複使用
* TailwindCSS 
  樣式直接寫在 JSX 裡，vibe coding 友善
  不用煩惱 class naming跟維護CSS檔案
* Leaflet
  底圖渲染與互動
  API簡潔、設定少
  react-leaflet 提供成熟的 React + TypeScript 整合
  但 WMS/WFS 需要透過外掛支援，不過目前應該沒有需要
  備選：OpenLayers
  有原生支援GIS 資料格式、OGC 標準服務
* Typescript
  強型別，穩定性高，與後端共用DTO，降低資料格式不一致的風險

---

### Core Backedn (主後端)

| 層 | 職責 |
|---|---|
| Controller Layer(控制器層) | 接收 HTTP request、驗證格式、轉發、包裝 response |
| Flow Dispatch(任務流程派發) | 觸發 Async Worker 開始一個 Flow、查詢 Flow 層級整體狀態 |
| Review Workflow(審核流程) | 審核結果的確認/刪除、狀態轉換 |
| Permission Policy(權限管理) | 角色權限判斷(含帳號管理職責併入於此) |
| Repositories(資料存取) | 封裝 PostgreSQL/PostGIS 查詢邏輯 |
| Authentication(登入驗證) | 登入驗證、身份確認 |
| Upload Coordination(上傳協調) | 簽發 presigned URL,讓大檔案繞過主後端直傳 Object Storage |

#### 技術選型
* Nest.js
  與前端分開、避免codebase邊界模糊、
  模組化架構、依賴注入、一致性高
  與Typescript綁定、有官方BullMQ整合
* Typescript
  強型別，穩定性高，與前端共用DTO

---

### Tilling service (圖片切割服務)

| 層 | 職責 |
|---|---|
| Streaming Decoder(串流解碼) | 串流讀取 BigTIFF、處理 strip-based 儲存、LZW+Predictor=2 差分還原、邊讀邊釋放記憶體 |
| Tile Splitting(子圖切割) | 將解碼出的橫向長條做垂直累積(Row Buffering)+ 水平切割(Column Slicing),重組成正方形子圖,處理 sliding window 重疊 |
| Tile Uploader(子圖上傳) | 將切好的子圖寫入 Object Storage |
| Tile Manifest Reporting(子圖清單回報) | 收集座標與儲存路徑,組成 metadata manifest 回報給 Async Worker |

**需要 Tile Splitting的原因:** 檔案為 strip-based 儲存(Block=寬度×1),解碼一次拿到的是橫跨全寬、僅 1 行高的長條,並非正方形。需先垂直累積夠切割高度(如 5000 行),再從累積出的寬版面橫向切出定寬視窗,才能重組成正方形子圖。此重組是解碼完成後獨立的二維視窗運算,不屬於解碼的副產品。

- 讀取水土人員提供的空拍正射影像，格式為 GeoTIFF，
  由 Pix4Dmapper 拼接輸出
- 已確認規格：BigTIFF、Strip-based 儲存、LZW 壓縮 + Predictor=2、
  座標系統 EPSG:3826
  詳見[NOTES.md](./NOTE.md###空拍圖規格)
- 已知最大檔案 7.91GB（僅為目前已檢查範圍內的最大值，非確認上限），
  無法一次載入 RAM，需以串流方式逐行讀取
- 依照 5000×5000 尺寸做 Sliding Window 切割，每個切割出的小圖需保留
  對應的地理座標，供後續辨識結果回貼地圖使用
- 已處理完的區域立即釋放記憶體，避免 RAM 佔用隨檔案大小線性增加

**待確認事項：**
  1. Sliding Window 的實作是否正確處理邊界情況（例如原圖尺寸無法被5000 整除時，最後一塊如何處理：補邊 padding 還是縮小尺寸）
  2. 每個切割出的小圖，座標轉換（像素座標 → EPSG:3826 地理座標）的計算是否正確
  3. 記憶體釋放時機是否確實在每個 window 處理完後執行，而非等到整份檔案讀完才釋放
  4. **切割顆粒度(現行 5000×5000)訂立的原始理由尚未明確**——需向學長確認當初是依據何種考量選定此尺寸(記憶體控制經驗值?UI 顯示需求?模型輸入尺寸?),此答案將直接影響是否需要調整切割顆粒度。
  5. **COG(Cloud-Optimized GeoTIFF)方案的可行性評估**——現行檔案為 strip-based,非 COG(tile-based + overview 分層),不具備直接 range-read 的效率優勢。若評估將原始檔案轉為 COG,搭配模型服務端做 windowed read,可能可以省去「切好子圖並持久化存入 Object Storage」這一步(即 Tile Uploader 這一層),改為推論當下即時讀取、讀完即丟。惟此方案需額外考量:
   - 轉檔本身的時間與資源成本(13.6GB 檔案轉檔耗時待測)
   - 前端 Result Review 頁面顯示 bounding box 疊圖時,若不持久化子圖檔案,需改為即時裁切服務,是否有可接受的延遲
   - COG windowed read 在實際 Object Storage(S3/MinIO)環境下的效能是否確實優於現行手刻方案
   - 模型端一次推論的視窗尺寸,不論是否採 COG,仍受限於模型固定輸入尺寸(如 1024×1024),COG 僅改善「讀取效率」,不改變「模型一次能吃多大範圍」這件事


#### 技術選型
* Go + GDAL (go-gdal binding 或直接透過 CGO 呼叫)
  使用 GDAL 處理底層 TIFF/BigTIFF 解析、LZW 解壓縮與 Predictor 還原，
  在此基礎上自行實作 5000x5000 的 Sliding Window 切割與座標對應邏輯。
  - 無 GIL (Global Interpreter Lock) 限制，可用 goroutine 平行處理
    多個 strip 的解壓縮與切割
  - encoding/binary 套件對二進位格式解析原生支援，處理 TIFF 檔頭與
    IFD (Image File Directory) 結構較直接
  - 編譯為單一執行檔，部署與跨平台無額外執行環境依賴

---

### Async Worker(三層)+ 佇列儲存

| 層 | 職責 |
|---|---|
| Flow Producer(流程建構) | 接收 Core Backend 觸發,建立 parent-child job 樹(定義流程形狀) |
| Job Processors(任務執行) | 從佇列取出個別 job,呼叫對應服務(Tiling Service/AI Model)執行 | 
| Result Aggregation(結果聚合) | 座標轉換(子圖內部像素座標→地理座標)、合併多子圖結果、去除跨子圖邊界重複偵測、寫入 Data Storage |

#### 技術選型

* Redis
  如果任務不複雜其實不需要，可以用PostgreSQL手搓
  但是目前看來一批很多張圖、大圖切小圖、小圖辨識，會有任務依賴關係

* BullMQ
  Redis只是底層儲存方案，狀態機、重試機制等實作不想造輪子，拿現有的來用
  與Nest.js有官方整合

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

#### 待確認事項
1. AOF/RDB 實際設定尚未於系統中落實,待實作後驗證
2. Stalled job 偵測參數(`lockDuration`、`stalledInterval`)需依實測單張子圖耗時調整

#### 技術選型
* Redis(同 Async Worker)

---

### Data Storage (資料儲存)
- 儲存使用者、任務、水井、審核紀錄等結構化資料，並支援空間查詢


#### 技術選型
* PostgreSQL
  免費 效能與生態完整優於MySQL
  有支援PostGIS算法
  授權較MySQL寬鬆
* PostGIS
  座標轉換、空間索引和函數、地圖工具整合完整
  
---


### Object Storage (物件儲存)
- 儲存空拍圖、切割後子圖等大型檔案,透過 Presigned URL 模式讓前端/Worker 直接存取,避免主後端負擔檔案傳輸流量
- 底層可為 AWS S3 或自架 MinIO,架構角色統一稱 Object Storage

#### 待確認事項
1. 是否有資源能夠存取大量空拍圖、圖資能否上雲端(資料主權/授權限制)

#### 技術選型
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
 
**職責邊界決策:** 座標轉換(EPSG:3826 轉換等)不放在 AI Model 內部,統一交由 Async Worker 的 Result Aggregation 處理。理由:保持 AI Model 為純粹的電腦視覺服務,與地理座標系統解耦,提升未來更換模型/搬遷部署環境時的可替換性。
 
* Fast API
  支援非同步
  
* Python (暫定)
  訓練時使用
  部署使用的語言還沒確定 沿用的話就不用重寫
  
* ONNX
  好打包哪裡都能跑的格式，還未確定會放在那裡跑，保留擴充性
  有需要的話也可以使用ONNX Runtime

---

## 外部

### 地圖套件
供地圖檢視之底圖使用
* NLSC
  台灣官方圖源，座標系統對應 TWD97
  基本電子地圖底圖屬「免申請」服務項目；
  但是還需要確定穩定性、可以打多少API 

* 備案
  若 NLSC 穩定性不足，可自架 tile server（如 TileServer GL）；
  若圖資可上雲端且有預算，可用商業地圖服務（如 Google Maps Platform）；
  OSM 公開 tile server 因官方使用政策明確限制重度使用、
  可能未經通知即封鎖存取，不適合作為正式產品底圖來源，
  僅可作開發階段臨時測試

---
