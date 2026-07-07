# NOTE
一些零碎的筆記

水土人員用ARCGIS

### 空拍圖規格
- 已透過 gdalinfo 與檔頭 hex 檢查確認技術規格：
  - **BigTIFF**（版本號 0x2B，非經典 TIFF 的 0x2A），因未壓縮資料量約 13.6GB，
    遠超過經典 TIFF 的 4GB 定址上限
  - **Strip-based 儲存**（Block=寬度x1，逐掃描線儲存，無內部分塊 Tiling）
  - **LZW 壓縮 + Predictor=2**（水平差分預測，解壓縮後需額外做差分還原
    才能得到正確像素值）
  - 座標系統為 **EPSG:3826**（TWD97 / TM2 121度分帶），座標資訊確認內嵌於
    TIFF 本身（gdalinfo 的 Files 欄位僅列出 .tif，未關聯外部 .tfw/.prj）
  - 含 Alpha 遮罩 band，標記空拍未覆蓋區域