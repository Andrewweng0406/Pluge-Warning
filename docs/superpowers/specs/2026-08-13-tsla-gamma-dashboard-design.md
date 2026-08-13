# TSLA Gamma 暴跌/洗盤預警儀表板 — 設計文件

日期：2026-08-13

## 目標

打造一個極簡、數據驅動的 TSLA 專屬儀表板，核心是三色風險燈號 + 關鍵 Gamma 防禦牆，取代 SpotGamma / Unusual Whales 等複雜昂貴平台。

## 範圍

本次實作涵蓋四層：數據抓取、量化核心邏輯、前端 Dashboard、Telegram 自動預警。暗池 (Dark Pool) 監控本期不做（免費數據源無法取得真實機構暗池成交），僅預留未來付費數據源的擴充點。

## 架構

三個 Railway 服務，共享一個 SQLite Volume：

1. **worker**（Python，長駐進程）：定時抓取 → 計算 → 寫入 SQLite → 燈號變化時推送 Telegram
2. **api**（Python FastAPI）：讀 SQLite，提供 REST JSON 給前端
3. **web**（Next.js + Tailwind + Chart.js）：Dashboard 前端，輪詢 api

worker 與 api 分開部署的原因：兩者生命週期與擴展需求不同（長駐迴圈 vs 無狀態請求），worker 卡住不應影響前端讀取歷史數據。

## 數據源

- **行情/期權鏈**：`yfinance`，抓取 TSLA 未來 45 天內所有到期日的 calls/puts。yfinance 不提供官方 Greeks，自行用 Black-Scholes 計算。
- **無風險利率**：固定近似值 4.5%（寫死於 config，不接外部數據源）。
- **暗池**：本期不實作。

## 量化核心邏輯 (`quant_core.py`)

**Gamma 計算**：Black-Scholes 公式，輸入為現價、strike、到期時間（年化）、yfinance 提供的 `impliedVolatility`、固定無風險利率 4.5%。

**GEX（Gamma Exposure）**：
```
GEX_per_strike = Σ [Gamma × OpenInterest × 100 × 現價² × 0.01 × (Call: +1, Put: -1)]
```
按 strike 加總得到每個價位的淨 GEX。

**Zero Gamma Flip Point**：將各 strike 的累積 GEX 按價格排序，找累積曲線穿越零軸的兩個相鄰 strike，線性插值得到 flip 價位。

**Put Wall / Call Wall**：GEX 絕對值最大的 put strike / call strike（反映做市商對沖壓力最大的價位），非單純 OI 最大值。

**Max Pain**：標準演算法——對每個候選到期價，計算所有 call/put 在該價格下對買方的總內在價值損失，找總損失最小的價位。

**IV Skew 預警**：比較 Far OTM Put（約現價 10–15% 價外）IV 與 ATM IV 的差值，與近 20 筆歷史記錄的均值/標準差比較，超過 +1.5 個標準差視為「IV Skew 拉升」。

## 三色燈號邏輯 (`signal_engine.py`)

```
spot = 現價
zero_gamma = Zero Gamma Flip Point

if spot < zero_gamma:
    🔴 紅燈 High Risk
elif |spot - zero_gamma| / zero_gamma <= 0.015:
    🟡 黃燈 Caution（雙側 ±1.5% 擠壓區皆算）
elif spot > zero_gamma * 1.015:
    🟢 綠燈 Bullish Flow
```

黃燈狀態下若同時觸發 IV Skew 拉升信號，標註為「黃燈 + IV 預警」子狀態。

## 數據流與排程

worker 僅在美股交易時段（9:30–16:00 ET）運行，每 5–15 分鐘一次：

1. 抓 TSLA 現價 + 期權鏈
2. 計算 GEX / Max Pain / Zero Gamma / Put Wall / Call Wall / IV Skew
3. 判定燈號
4. 寫入 SQLite `snapshots` 表
5. 若燈號與上一筆記錄不同 → 發送 Telegram 通知
6. 若無變化 → 僅存數據，不推送

## 資料庫結構

```sql
CREATE TABLE snapshots (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ts DATETIME,
  spot REAL,
  zero_gamma REAL,
  put_wall REAL,
  call_wall REAL,
  max_pain REAL,
  iv_skew REAL,
  light TEXT,        -- 'red' | 'yellow' | 'green'
  iv_alert BOOLEAN
);
```

## API 端點

- `GET /latest` — 最新一筆快照
- `GET /history?days=N` — 歷史數據，供前端趨勢圖使用

## Telegram 推送 (`alerts.py`)

- 用 `requests` 呼叫 Telegram Bot API `sendMessage`
- 觸發條件：燈號變化（如 🟢→🟡、🟡→🔴）
- Bot Token、Chat ID 存於 Railway 環境變數，不寫死於程式碼
- 訊息範例：
```
🔴 TSLA 警報：跌破 Zero Gamma 防線
現價: $318.42 | Zero Gamma: $323.10
Put Wall: $300 | Call Wall: $340 | Max Pain: $325
```

## 前端 Dashboard（Next.js + Tailwind + Chart.js）

三個核心組件：

1. `TrafficLight` — 大色塊 + 當前燈號文字說明，每 30–60 秒輪詢 `/latest`
2. `WallsPanel` — Put Wall / Call Wall / Max Pain / Zero Gamma 四個數值卡片
3. `HistoryChart` — Chart.js 折線圖，疊加現價與 Zero Gamma 歷史軌跡，來自 `/history`

## 錯誤處理

- yfinance 抓取失敗/限流 → worker 記錄日誌，跳過本輪，不寫入錯誤數據，不誤觸發燈號變化推送
- 期權鏈數據為空（如非交易日）→ 直接跳過該輪
- Telegram 發送失敗 → 記錄日誌並重試一次，不阻塞下一輪 worker 迴圈

## 測試策略

- `quant_core.py`：用固定的模擬期權鏈數據（寫死幾個 strike 的 OI/IV）做單元測試，驗證 GEX 加總、Zero Gamma 插值、Max Pain 計算結果與手算預期一致
- `signal_engine.py`：邊界值測試三色燈號判定（剛好在 ±1.5% 邊界、跨越 zero_gamma 等）

## 部署

Railway 三服務：worker、api、web，SQLite 檔案存於共享 Railway Volume，掛載給 worker 與 api 兩個服務。

## 非目標（本期不做）

- 暗池大單監控（需付費數據源，如 Cheddar Flow / FlowAlgo）
- 多股票支援（僅 TSLA）
- 用戶系統/多人訂閱
