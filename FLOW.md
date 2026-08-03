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
 
## 待確認事項(承接 QA.md 既有問題,新增本次討論衍生的問題)
 
- Tiling Service 的切割顆粒度(現行 5000×5000)與是否導入 COG 直讀方案,詳見 ARCHITECTURE.md「圖片切割服務」章節待確認事項,需先向學長確認後再決定是否調整本流程中的步驟 3。
- Redis 持久化策略(AOF/RDB 設定)、BullMQ stalled job 偵測參數,實作時需依此文件與 ARCHITECTURE.md 中「Async Worker」章節的建議進行設定與實測驗證。
 

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
