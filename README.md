# FTMO MT5 歷史行情下載器 v8

這是一個可部署在 GitHub Pages 的瀏覽器版 FTMO MT5 歷史行情下載器。使用瀏覽器原生 WebSocket 與 Web Crypto 連線，不需要安裝 Python、不需要本機伺服器，也不需要啟動 MT5 Desktop。

程式只使用 FTMO MT5 Investor／Read-only 密碼登入，只查詢歷史行情，不會送出下單或交易命令。

## v8 主要更新：登入一次、連續查詢

v7 每次下載都會建立新連線、Bootstrap、登入、查詢、登出。v8 改為將登入工作階段與行情查詢分開：

1. 使用者輸入 MT5 Login ID 及 Investor／Read-only 密碼，按「登入 FTMO」。
2. 瀏覽器建立 WebSocket，執行 Bootstrap，匯入當次 Session AES Key，再送出唯讀登入命令 `cmd=28`。
3. 登入成功後，WebSocket 與 Session AES Key 保留在目前頁面的記憶體中。
4. 使用者可以改選商品、K 線週期及 UTC 時間範圍，重複按「下載 CSV」。每次下載只送出歷史行情命令 `cmd=11`，不會重新登入，也不會在下載完成後登出。
5. 使用者按「登出」，或連續 15 分鐘沒有操作時，程式會送出 MT5 登出命令 `cmd=2`，再關閉 WebSocket 並清除頁面中的 Session 參考。

## 閒置登出規則

- 閒置期限目前固定為 **15 分鐘**。
- 使用者在頁面內點擊、輸入或更改選項等操作，會重設閒置倒數。
- 正在執行歷史行情下載時，閒置倒數暫停；下載完成後，從完成時間重新計算 15 分鐘。
- 瀏覽器可能節流背景分頁的計時器。頁面回到前景時，程式會以實際經過的時間重新檢查期限；若已超過 15 分鐘，便會開始登出。
- 手動登出會嘗試等待 FTMO 對 `cmd=2` 的回應，收到成功回應後再關閉 WebSocket。
- 若 WebSocket 意外中斷、FTMO 伺服器提早關閉連線，程式會清除本機 Session 狀態並要求重新登入。v8 不會把密碼儲存起來，因此重新登入時需要再次輸入唯讀密碼。

### 關閉瀏覽器或分頁

頁面關閉時，`pagehide` 事件會盡力關閉 WebSocket 並清除頁面中的 Session 參考。但瀏覽器無法保證在頁面離開期間完成非同步的 MT5 `cmd=2` 登出請求。因此：

- WebSocket 傳輸連線通常會隨頁面關閉而終止；
- 這不等於 MT5 應用層一定收到並確認 `cmd=2`；
- 若頁面直接被關閉或瀏覽器程序遭終止，伺服器必須依其連線逾時／斷線處理回收工作階段。

這是瀏覽器生命週期限制，不應把 `pagehide` 當成百分之百可靠的登出保證。

## 心跳與連線中斷

v8 **不會發送未經驗證的應用層心跳封包**。瀏覽器原生 WebSocket API 不提供 JavaScript 自行送出 WebSocket Ping control frame 的介面；若 FTMO 需要特定應用層心跳，必須先確認其正式封包格式，不能猜測。

因此，雖然 v8 可在同一個 Session 中連續查詢，但不能保證 FTMO 伺服器會無限期保持連線。若 FTMO 因伺服器政策、網路設備、網路中斷或閒置逾時而關閉連線，頁面會顯示連線中斷，使用者需重新輸入唯讀密碼登入。

## 功能

- 選擇常見外匯商品，或輸入 FTMO Market Watch 中的自訂商品名稱。
- 選擇 M1、M5、M15、M30、H1、H2、H3、H4、H6、H8、H12、D1、W1、MN1。
- 選擇最近 24 小時、7 天、30 天、90 天，或自訂 UTC 起訖日期與時間。
- 匯出 OHLC、Tick Volume、Spread、伺服器有提供時的 Real Volume、UTC ISO 8601 時間（`Z` 結尾）、Unix timestamp，以及 K 線是否已收盤。
- 顯示登入狀態、WebSocket 狀態及 15 分鐘閒置倒數。
- 可手動登出；閒置 15 分鐘會自動登出。
- 帳密不會寫入 `localStorage`、GitHub repository 或診斷日誌。密碼欄位會在登入請求建立後清空；Session AES Key 僅留在當前頁面記憶體中。

## K 線週期協定映射

介面週期以分鐘數表示，但 MT5 `cmd=11` 的 16-bit 週期欄位使用 MQL5 `ENUM_TIMEFRAMES` wire value。v8 沿用 v7 已修正的映射：

| 介面週期 | 分鐘數 | `cmd=11` 傳送值 |
|---|---:|---:|
| M1 | 1 | 1 |
| M5 | 5 | 5 |
| M15 | 15 | 15 |
| M30 | 30 | 30 |
| H1 | 60 | 16385（`0x4001`）|
| H2 | 120 | 16386（`0x4002`）|
| H3 | 180 | 16387（`0x4003`）|
| H4 | 240 | 16388（`0x4004`）|
| H6 | 360 | 16390（`0x4006`）|
| H8 | 480 | 16392（`0x4008`）|
| H12 | 720 | 16396（`0x400C`）|
| D1 | 1440 | 16408（`0x4018`）|
| W1 | 10080 | 32769（`0x8001`）|
| MN1 | 43200 | 49153（`0xC001`）|

## 部署到 GitHub Pages

1. 將本專案的 `index.html` 上傳至 GitHub Pages repository，覆蓋目前版本。
2. Commit changes，等待 GitHub Pages 完成部署。
3. 使用 HTTPS 網址開啟網站。
4. 輸入 MT5 Login ID、Investor／Read-only Password；若帳戶要求 OTP，再填入 OTP。
5. 按「登入 FTMO」，等待登入成功。
6. 選擇商品、K 線週期及 UTC 時間範圍，按「下載 CSV」。可重複更改選項並下載，不必重新登入。
7. 使用完畢按「登出」。

## 時區規則

- 日期與時間輸入一律解讀為 UTC，不依裝置所在時區換算。
- CSV 的 ISO 8601 時間以 `Z` 結尾，例如 `2026-11-17T06:30:00Z`；`Z` 明確代表 UTC。
- CSV 同時輸出 Unix timestamp，方便 Python、資料庫或 AI 系統讀取。
- `timestamp_timezone` 欄位標示為 `UTC`。
- 時間戳代表每根 K 線的開盤時間。

## CSV 欄位

CSV 包含：

- `symbol`、`timeframe`、`timeframe_minutes`
- `query_start_utc_iso8601`、`query_end_utc_iso8601`
- `bar_open_time_utc_iso8601`、`bar_close_time_utc_iso8601`
- `bar_open_time_utc_unix_seconds`、`timestamp_timezone`
- `open`、`high`、`low`、`close`
- `tick_volume`、`spread_points`、`real_volume`
- `is_closed`

`tick_volume` 是報價變動／Tick 數量，不等同於集中交易所的實際成交量。非交易時段不一定會有連續 K 線。

## 資料限制與注意事項

- 商品名稱必須與 FTMO Market Watch 完全一致。若商品有後綴，請使用自訂商品欄位。
- 單次請求預估上限為 100,000 根 K 線；超出時請縮短區間或改用較大的週期。
- 伺服器可能因市場休市或歷史資料供應範圍而回傳少於估算數量的 K 線。
- K 線解碼器只接受能唯一判斷為標準 48-byte 或擴充 56-byte 記錄格式的回應；若格式不明確，程式會停止解析，避免猜測資料結構。
- Web Terminal 通訊協定為非官方、逆向分析的實作，可能隨伺服器版本變動。重要分析前，請抽樣與 FTMO WebTrader 的 OHLC 數值比對。
- v8 目前不會自動重新連線或自動重新登入。連線失效後，需使用者重新輸入唯讀密碼並登入。
- 本工具只應使用自己獲授權的 FTMO 帳戶及唯讀密碼；不要分享密碼、OTP、Session Token、Session AES Key 或完整認證封包。

## 已驗證項目

依目前使用者提供的測試紀錄：

- GitHub Pages Origin 可成功與 FTMO WebSocket 建立連線。
- Bootstrap `cmd=0` 可成功解密並取得 Session 資料。
- 唯讀登入 `cmd=28` 已取得成功回應，登入回應結構亦已解析。
- EURCHF M15 歷史行情可成功取得並解析 96 根 K 線。
- H1 初次測試因週期 wire value 錯誤而收到 `response_code=1`；v7 修正週期映射後，使用者已回報測試成功。
- 登出 `cmd=2` 可取得成功回應。

**v8 的持續登入、跨多次查詢共用同一 Session 及閒置自動登出流程尚待 FTMO 實際測試。** 建議先登入後，連續下載兩組不同商品／週期，確認第二次查詢沒有再次執行 Bootstrap 或 `cmd=28`，最後測試手動登出。
