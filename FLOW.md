# FLOW
系統流程

## 一般使用者
1. 登入
2. 上傳圖片
3. 等待辨識
4. 確認辨識結果
5. 在地圖上查看
---

## 系統內部流程(新增,細化上述「等待辨識」實際發生的步驟)
 
1. **使用者上傳圖片**：經 Core Backend 的 Upload Coordination 驗證後,透過 presigned URL **直接**傳輸至 Object Storage,不經過 Core Backend 主機記憶體。
2. **Core Backend 開啟辨識流程**：Flow Dispatch 呼叫 Async Worker,觸發 Flow Producer 建立本次 Flow 的 parent-child job 樹,並開始可供前端查詢的狀態追蹤。
3. **Tiling Service 分割圖片**：Streaming Decoder 串流讀取 Object Storage 上的原圖、Tile Splitting 切割成子圖,Tile Uploader 將子圖存回 Object Storage,Tile Manifest Reporting 將座標與路徑等 metadata 回報給 Async Worker(此步驟座標暫存於 Queue Store,尚未寫入 Data Storage)。
4. **AI Model 辨識**：Job Processors 逐一呼叫 AI Model 對各子圖執行推論,AI Model 僅回傳 tile 內部像素座標與信心分數,不涉及地理座標轉換。
5. **Async Worker 結果聚合**：Result Aggregation 將所有子圖的辨識結果進行座標轉換(像素座標→地理座標)、合併、去除跨子圖邊界重複偵測,**此為整個流程中資料第一次、也是唯一一次寫入 Data Storage 的時間點**。
6. **回傳前端顯示**：前端透過查詢 Flow Dispatch 取得進度與最終結果(見下方「前端進度顯示機制」)。
---
 
## 前端進度顯示機制
 
**決策:優先採用輪詢(polling),暫不採用 SSE/WebSocket 等即時推播機制。**
 
**理由:**
- 原先考慮 SSE 的兩個理由中,「系統重啟可接續 flow」與前端推播機制**無關**——斷點續傳能力來自 Queue Store(Redis)對 Job/Flow 狀態的持久化,即使完全不做任何前端推播也同樣成立。SSE 唯一實際解決的是使用者體驗的即時性。
- 輪詢成本:若查詢對象為 Queue Store 中的輕量狀態(而非重運算的資料庫 join),延遲可忽略,系統負擔小。
- SSE 成本:需額外橋接元件(job 事件→Core Backend→SSE 連線)、仍須保留輪詢 API 作為斷線重連 fallback、長連線的維運複雜度(逾時設定、連線數上限)。
- 辨識流程為分鐘級別(單張子圖約 30 秒,整體視子圖數量),秒級輪詢與即時推播的使用者體感差異極小。
**後續評估時機:** 若上線後有具體使用者回饋顯示輪詢頻率不足以滿足需求,再評估疊加 SSE,屆時輪詢 API 仍保留作為 fallback,不會是重工。
 
---

## 水土審核 Copilot 流程(自然語言空間問答)

1. 使用者於 Copilot 對話介面輸入自然語言指令(如查詢條件、地理範圍)
2. Core Backend 的 Query Translation 呼叫 LLM API,將自然語言解析為結構化查詢意圖(JSON),LLM 不生成可執行查詢語法、不接觸水井座標資料
3. Query Builder 依白名單欄位與運算子,將結構化意圖組成參數化 PostGIS 查詢
4. 若查詢涉及地理範圍限定(如「沿海」),向 NLSC WFS 取得對應行政區界資料(Lazy Fetch and Cache,首次查詢時取得並快取,ST_Transform 轉換為 EPSG:3826 後再與水井座標做 ST_DWithin)
5. 執行查詢,結果以清單/地圖呈現給使用者
6. 使用者可指定欄位與格式(CSV/XLSX),由 Export Service 產生清冊供下載;未指定時使用預設模板;僅於使用者明確要求時才產生

---

## 跨期比對流程(4D 時空光軸)

1. 使用者針對同一區域進行多次上傳,各次上傳各自走過現有的系統內部流程(見上方),各自完成 Result Aggregation 寫入 Data Storage
2. Result Aggregation 寫入完成後,觸發 Temporal Change Detection,將本次新確認的水井與歷史紀錄比對,判斷是否為新增
3. 前端 Timeline 元件讀取各期資料,以時間軸/雙視窗同步方式呈現同一位置跨期的水井新增歷程

---

## AI 生成模擬圖流程

1. 審核人員於 Result Review 頁面,針對俯視辨識困難的疑似點位,手動觸發生成
2. 請求排入 Queue Store 成為獨立 job(單一 job,不建立 parent-child job 樹)
3. Generative Simulation 元件執行生成式模型推論(實作細節待學長確認)
4. 前端沿用既有輪詢機制查詢生成狀態,完成後顯示模擬圖,並附「AI 生成模擬,僅供參考」之提示

---
 
## 待確認事項(承接 QA.md 既有問題,新增本次討論衍生的問題)
 
- Tiling Service 的切割顆粒度(現行 5000×5000)與是否導入 COG 直讀方案,詳見 ARCHITECTURE.md「圖片切割服務」章節待確認事項,需先向學長確認後再決定是否調整本流程中的步驟 3。
- Redis 持久化策略(AOF/RDB 設定)、BullMQ stalled job 偵測參數,實作時需依此文件與 ARCHITECTURE.md 中「Async Worker」章節的建議進行設定與實測驗證。
- 跨期比對中「同一口井」的距離判斷門檻,待井的實際尺寸與模型框選誤差實測後訂定。
- AI 生成模擬圖的排隊/輪詢機制,待與學長確認是否採用此設計。
 

## 空拍圖
1. 使用者上傳空拍圖(web → core backend)
2. Core Backend 建立整體流程的初始 Flow(core backend → async worker, via BullMQ FlowProducer)
3. tiling、計算子圖座標(async worker 呼叫 tiling service)
4. 子圖影像直接寫入物件存儲(tiling service → object storage,presigned URL)
5. 座標等 metadata 回傳給 async worker(tiling service → async worker,只有座標,不含影像)
6. Async Worker 在同一 job 完成後,建立 detection 的 child jobs(async worker 內部邏輯,FlowProducer)
7. pre-process(AI model)
8. detection(AI model)
9. post-process(AI model)
10. 辨識結果、水井座標回傳給 async worker(AI model → async worker)
11. Async Worker 呼叫 core backend 內部 API,寫入辨識結果與座標(async worker → core backend → 資料庫,只有 metadata)
12. 回前端顯示(core backend → web)