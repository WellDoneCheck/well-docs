# 系統架構
系統元件概述及架構決策紀錄

## 內部

### 前端
- 登入介面（無公開註冊）
- 大檔案圖片上傳（空拍圖，體積大）
- 辨識結果的人工審核介面（可能需要顯示 bounding box、可勾選確認/刪除）
- 地圖檢視（載入 NLSC 底圖、顯示標記點、點擊查詳情）
- 這是一個內部後台系統，不對外公開、無SEO需求，故以CSR為主

####　技術選型
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

### 主後端
- 使用者認證（登入、無公開註冊、管理員手動建帳號）
- 圖片上傳的接收與轉發（大檔案，需要與 Object Storage / Queue 協調）
- 佇列任務派發與狀態追蹤（把推論工作丟給 Async Worker，追蹤進度）
- 審核流程的 CRUD（確認/刪除辨識結果）
- 地圖資料查詢（結合 PostgreSQL + GIS 的空間查詢）
- 權限控管（一般使用者 vs 系統管理員）

####　技術選型
* Nest.js
  與前端分開、避免codebase邊界模糊、
  模組化架構、依賴注入、一致性高
  與Typescript綁定、有官方BullMQ整合
* Typescript
  強型別，穩定性高，與前端共用DTO


### 圖片切割服務
- 讀取水土人員提供的空拍正射影像 (Orthomosaic)，格式為 GeoTIFF，
  由 Pix4Dmapper 拼接輸出
- 已確認規格：BigTIFF、Strip-based 儲存、LZW 壓縮 + Predictor=2、
  座標系統 EPSG:3826
  詳見[NOTES.md](./NOTE.md###空拍圖規格)
- 已知最大檔案 7.91GB（僅為目前已檢查範圍內的最大值，非確認上限），
  無法一次載入 RAM，需以串流方式逐行讀取
- 依照 5000×5000 尺寸做 Sliding Window 切割，每個切割出的小圖需保留
  對應的地理座標，供後續辨識結果回貼地圖使用
- 已處理完的區域立即釋放記憶體，避免 RAM 佔用隨檔案大小線性增加
- 與主後端 / Async Worker 的關係：此服務接收 Async Worker 派發的切割任務，
  輸出結果（小圖 + 座標）交由 Async Worker 收集並轉交模型端推論

#### 技術選型
* Go + GDAL (go-gdal binding 或直接透過 CGO 呼叫)
  使用 GDAL 處理底層 TIFF/BigTIFF 解析、LZW 解壓縮與 Predictor 還原，
  在此基礎上自行實作 5000x5000 的 Sliding Window 切割與座標對應邏輯。
  - 無 GIL (Global Interpreter Lock) 限制，可用 goroutine 平行處理
    多個 strip 的解壓縮與切割
  - encoding/binary 套件對二進位格式解析原生支援，處理 TIFF 檔頭與
    IFD (Image File Directory) 結構較直接
  - 編譯為單一執行檔，部署與跨平台無額外執行環境依賴

  **待確認事項（暫列為開放問題）：**
  1. Sliding Window 的實作是否正確處理邊界情況（例如原圖尺寸無法被
     5000 整除時，最後一塊如何處理：補邊 padding 還是縮小尺寸）
  2. 每個切割出的小圖，座標轉換（像素座標 → EPSG:3826 地理座標）
     的計算是否正確
  3. 記憶體釋放時機是否確實在每個 window 處理完後執行，
     而非等到整份檔案讀完才釋放



### Async Worker
- 任務排隊：避免同時大量圖片湧入時，模型直接被打爆
- 大圖切割與子任務聚合：大圖切成小圖子任務、推論完再聚合結果
- 失敗重試：模型推論可能因暫時性錯誤失敗，需要重試機制
- 與主後端解耦：主後端不該被推論耗時卡住
  
* Redis
  如果任務不複雜其實不需要，可以用PostgreSQL手搓
  但是目前看來一批很多張圖、大圖切小圖、小圖辨識，會有任務依賴關係

* BullMQ
  Redis只是底層儲存方案，狀態機、重試機制等實作不想造輪子，拿現有的來用
  與Nest.js有官方整合

  
### SQL
- 儲存使用者、任務、水井、審核紀錄等結構化資料，並支援空間查詢

* PostgreSQL
  免費 效能與生態完整優於MySQL
  有支援PostGIS算法
  授權較MySQL寬鬆
* PostGIS
  座標轉換、空間索引和函數、地圖工具整合完整


### 模型
- 接收 Async Worker 傳來的圖片，執行模型推論，回傳偵測結果
- 
* Fast API
  支援非同步
* Python (暫定)
  訓練時使用
  部署使用的語言還沒確定 沿用的話就不用重寫
  
* ONNX
  好打包哪裡都能跑的格式，還未確定會放在那裡跑，保留擴充性
  有需要的話也可以使用ONNX Runtime

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

### 物件存儲(選用)
- 儲存空拍圖等大型檔案，透過 Presigned URL 模式讓前端/Worker 直接存取，
  避免主後端負擔檔案傳輸流量

* AWS S3
  不確定有沒有資源能夠存取大量的空拍圖
  快速 不限大小 要的時候再讀 這樣不用占用RAM的空間
  且不確定圖能不能上雲端

* 備案
  若圖資因資料主權/授權限制不可上境外雲端，可改用自架的
  S3-compatible 物件儲存（如 MinIO），API 相容故程式碼改動成本低
