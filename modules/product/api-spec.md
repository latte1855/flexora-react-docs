# Product Module – API Spec (草稿)

主要服務預期包含：

- `GET /api/products`、`POST /api/products`：產品主檔 CRUD。
- `GET /api/item-skus`、`POST /api/item-skus`：SKU 與 UoM/稅別。
- `GET /api/price-lists`、`GET /api/item-prices`：價目表與價格。

TODO：

- [ ] 定義查詢參數（關鍵字、分類、啟用狀態）。
- [ ] 規劃批次匯入/匯出 API。
- [ ] 與報價模組共用的 lookup API（Async Select）需列出回傳格式。

> 待後端設計完成後再補充詳細 payload、錯誤碼與 workflow。
