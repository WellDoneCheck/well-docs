# FLOW
系統流程

##　一般使用者
1. 登入
2. 上傳圖片
3. 等待辨識
4. 確認辨識結果
5. 在地圖上查看

## 空拍圖
1. 使用者上傳空拍圖(web → core backend)
2. Core Backend 建立整體流程的初始 job(core backend → async worker, via BullMQ FlowProducer)
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

