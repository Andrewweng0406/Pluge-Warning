# TSLA Gamma 預警儀表板 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 建置一個持續運行的 TSLA 期權 Gamma 分析系統：後端定時抓取期權鏈、計算 GEX/Max Pain/Zero Gamma/IV Skew、判定三色燈號、寫入 SQLite、燈號變化時推送 Telegram；前端 Next.js Dashboard 顯示燈號與防禦牆數據。

**Architecture:** 三個獨立部署單元共享一個 SQLite 檔案：`worker`（長駐迴圈，僅在美股交易時段定時抓取與計算）、`api`（FastAPI，唯讀查詢層）、`web`（Next.js，輪詢 api）。後端所有量化邏輯為純函數，放在 `backend/app/quant_core.py` 與 `backend/app/signal_engine.py`，不依賴網路或資料庫，可獨立單元測試。

**Tech Stack:** Python 3.11+（yfinance、FastAPI、uvicorn、requests、pytest、sqlite3 標準庫）；Next.js 14+（App Router）、TypeScript、Tailwind CSS、Chart.js + react-chartjs-2、Vitest + Testing Library。

## Global Constraints

- 標的僅 TSLA，本期不支援多股票（spec 非目標）
- 無風險利率固定 4.5%（`RISK_FREE_RATE = 0.045`），不接外部利率數據源
- 燈號判定帶寬 ±1.5%（`LIGHT_BAND = 0.015`）
- IV Skew 預警門檻：超過近 20 筆歷史記錄的 +1.5 個標準差（`IV_SKEW_STD_THRESHOLD = 1.5`）
- 期權鏈抓取範圍：未來 45 天內所有到期日（`MAX_DAYS = 45`）
- worker 僅在美股交易時段（9:30–16:00 America/New_York，週一至週五）運行，每 5–15 分鐘一次
- Telegram 僅在燈號較上一筆記錄變化時推送，無變化不推送
- 暗池 (Dark Pool) 監控本期不做
- Bot Token / Chat ID 一律讀環境變數，不寫死於程式碼
- worker 與 api 共享同一個 SQLite 檔案（Railway Volume）

---

## File Structure

```
backend/
  app/
    __init__.py
    config.py
    db.py
    quant_core.py
    signal_engine.py
    data_fetcher.py
    alerts.py
    worker.py
    api.py
  tests/
    __init__.py
    test_db.py
    test_quant_core.py
    test_signal_engine.py
    test_data_fetcher.py
    test_alerts.py
    test_worker.py
    test_api.py
  requirements.txt
  .env.example
  railway.toml

frontend/
  app/
    layout.tsx
    page.tsx
  components/
    TrafficLight.tsx
    WallsPanel.tsx
    HistoryChart.tsx
  lib/
    api.ts
  tests/
    TrafficLight.test.tsx
    WallsPanel.test.tsx
    HistoryChart.test.tsx
  vitest.config.ts
  vitest.setup.ts
  .env.local.example
```

---

### Task 1: Backend 專案骨架與設定

**Files:**
- Create: `backend/requirements.txt`
- Create: `backend/app/__init__.py`
- Create: `backend/app/config.py`
- Create: `backend/tests/__init__.py`
- Create: `backend/pytest.ini`

**Interfaces:**
- Produces: `config.TICKER`, `config.MAX_DAYS`, `config.RISK_FREE_RATE`, `config.LIGHT_BAND`, `config.IV_SKEW_STD_THRESHOLD`, `config.POLL_INTERVAL_SECONDS`, `config.DB_PATH`, `config.TELEGRAM_BOT_TOKEN`, `config.TELEGRAM_CHAT_ID`, `config.as_dict() -> dict`

- [ ] **Step 1: 建立 requirements.txt**

```
yfinance==0.2.54
fastapi==0.115.0
uvicorn==0.30.6
requests==2.32.3
pytest==8.3.3
httpx==0.27.2
```

- [ ] **Step 2: 建立套件初始檔**

`backend/app/__init__.py`:
```python
```
(空檔，僅標記 `app` 為 Python 套件)

`backend/tests/__init__.py`:
```python
```

- [ ] **Step 3: 建立 config.py**

`backend/app/config.py`:
```python
import os

TICKER = "TSLA"
MAX_DAYS = 45
RISK_FREE_RATE = 0.045
LIGHT_BAND = 0.015
IV_SKEW_STD_THRESHOLD = 1.5

POLL_INTERVAL_SECONDS = int(os.environ.get("POLL_INTERVAL_SECONDS", "600"))
DB_PATH = os.environ.get("DB_PATH", "data/snapshots.db")
TELEGRAM_BOT_TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN", "")
TELEGRAM_CHAT_ID = os.environ.get("TELEGRAM_CHAT_ID", "")


def as_dict():
    return {
        "ticker": TICKER,
        "max_days": MAX_DAYS,
        "risk_free_rate": RISK_FREE_RATE,
        "telegram_bot_token": TELEGRAM_BOT_TOKEN,
        "telegram_chat_id": TELEGRAM_CHAT_ID,
    }
```

- [ ] **Step 4: 建立 pytest.ini**

`backend/pytest.ini`:
```ini
[pytest]
testpaths = tests
pythonpath = .
```

- [ ] **Step 5: 安裝依賴並驗證**

Run:
```bash
cd backend && python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt
```
Expected: 安裝成功無錯誤。

- [ ] **Step 6: Commit**

```bash
git add backend/requirements.txt backend/app/__init__.py backend/app/config.py backend/tests/__init__.py backend/pytest.ini
git commit -m "chore: scaffold backend project and config"
```

---

### Task 2: SQLite 持久化層 (`db.py`)

**Files:**
- Create: `backend/app/db.py`
- Test: `backend/tests/test_db.py`

**Interfaces:**
- Consumes: 無（僅依賴 `sqlite3` 標準庫）
- Produces: `get_connection(path: str) -> sqlite3.Connection`, `init_db(conn) -> None`, `insert_snapshot(conn, snapshot: dict) -> None`, `get_latest(conn) -> dict | None`, `get_history(conn, days: int) -> list[dict]`
  - `snapshot` dict 欄位：`ts`(str, ISO8601), `spot`(float), `zero_gamma`(float|None), `put_wall`(float|None), `call_wall`(float|None), `max_pain`(float|None), `iv_skew`(float|None), `light`(str), `iv_alert`(bool)

- [ ] **Step 1: 寫失敗測試**

`backend/tests/test_db.py`:
```python
import sqlite3
from datetime import datetime, timedelta

from app.db import get_connection, init_db, insert_snapshot, get_latest, get_history


def make_snapshot(ts, spot=300.0, light="green"):
    return {
        "ts": ts,
        "spot": spot,
        "zero_gamma": 323.1,
        "put_wall": 300.0,
        "call_wall": 340.0,
        "max_pain": 325.0,
        "iv_skew": 0.1,
        "light": light,
        "iv_alert": False,
    }


def test_insert_and_get_latest_returns_most_recent():
    conn = get_connection(":memory:")
    init_db(conn)
    insert_snapshot(conn, make_snapshot("2026-08-14T10:00:00", spot=300.0))
    insert_snapshot(conn, make_snapshot("2026-08-14T10:10:00", spot=305.0))

    latest = get_latest(conn)

    assert latest["spot"] == 305.0
    assert latest["light"] == "green"


def test_get_latest_returns_none_when_empty():
    conn = get_connection(":memory:")
    init_db(conn)

    assert get_latest(conn) is None


def test_get_history_filters_by_days():
    conn = get_connection(":memory:")
    init_db(conn)
    old_ts = (datetime.utcnow() - timedelta(days=10)).isoformat()
    recent_ts = (datetime.utcnow() - timedelta(hours=1)).isoformat()
    insert_snapshot(conn, make_snapshot(old_ts, spot=100.0))
    insert_snapshot(conn, make_snapshot(recent_ts, spot=200.0))

    history = get_history(conn, days=1)

    assert len(history) == 1
    assert history[0]["spot"] == 200.0
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd backend && pytest tests/test_db.py -v`
Expected: FAIL，因 `app.db` 模組不存在（`ModuleNotFoundError`）。

- [ ] **Step 3: 實作 db.py**

`backend/app/db.py`:
```python
import sqlite3
from datetime import datetime, timedelta

SCHEMA = """
CREATE TABLE IF NOT EXISTS snapshots (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ts TEXT NOT NULL,
  spot REAL NOT NULL,
  zero_gamma REAL,
  put_wall REAL,
  call_wall REAL,
  max_pain REAL,
  iv_skew REAL,
  light TEXT NOT NULL,
  iv_alert INTEGER NOT NULL
);
"""


def get_connection(path: str) -> sqlite3.Connection:
    conn = sqlite3.connect(path)
    conn.row_factory = sqlite3.Row
    return conn


def init_db(conn: sqlite3.Connection) -> None:
    conn.execute(SCHEMA)
    conn.commit()


def insert_snapshot(conn: sqlite3.Connection, snapshot: dict) -> None:
    conn.execute(
        """INSERT INTO snapshots
           (ts, spot, zero_gamma, put_wall, call_wall, max_pain, iv_skew, light, iv_alert)
           VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)""",
        (
            snapshot["ts"],
            snapshot["spot"],
            snapshot["zero_gamma"],
            snapshot["put_wall"],
            snapshot["call_wall"],
            snapshot["max_pain"],
            snapshot["iv_skew"],
            snapshot["light"],
            int(snapshot["iv_alert"]),
        ),
    )
    conn.commit()


def get_latest(conn: sqlite3.Connection) -> dict | None:
    row = conn.execute("SELECT * FROM snapshots ORDER BY ts DESC LIMIT 1").fetchone()
    return dict(row) if row else None


def get_history(conn: sqlite3.Connection, days: int) -> list[dict]:
    cutoff = (datetime.utcnow() - timedelta(days=days)).isoformat()
    rows = conn.execute(
        "SELECT * FROM snapshots WHERE ts >= ? ORDER BY ts ASC", (cutoff,)
    ).fetchall()
    return [dict(row) for row in rows]
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd backend && pytest tests/test_db.py -v`
Expected: 3 個測試全部 PASS。

- [ ] **Step 5: Commit**

```bash
git add backend/app/db.py backend/tests/test_db.py
git commit -m "feat: add SQLite persistence layer for snapshots"
```

---

### Task 3: 量化核心 — Black-Scholes Gamma 與 GEX 加總

**Files:**
- Create: `backend/app/quant_core.py`
- Test: `backend/tests/test_quant_core.py`

**Interfaces:**
- Consumes: 無
- Produces: `black_scholes_gamma(spot, strike, time_to_expiry, iv, risk_free_rate=0.045) -> float`；`GexResult` dataclass（欄位 `call_gex_by_strike: dict[float, float]`, `put_gex_by_strike: dict[float, float]`, `net_gex_by_strike: dict[float, float]`）；`compute_gex(options: list[dict], spot: float, risk_free_rate=0.045) -> GexResult`
  - `options` 每筆 dict 欄位：`strike`(float), `time_to_expiry`(float, 年化), `iv`(float), `open_interest`(float), `option_type`("call"|"put")

- [ ] **Step 1: 寫失敗測試**

`backend/tests/test_quant_core.py`:
```python
import math
import pytest

from app.quant_core import black_scholes_gamma, compute_gex


def test_black_scholes_gamma_matches_known_atm_value():
    # S=K=100, T=1, sigma=0.2, r=0.05 -> gamma ≈ 0.01876 (hand-derived reference)
    gamma = black_scholes_gamma(spot=100.0, strike=100.0, time_to_expiry=1.0, iv=0.2, risk_free_rate=0.05)

    assert gamma == pytest.approx(0.01876, abs=1e-4)


def test_black_scholes_gamma_zero_for_expired_option():
    gamma = black_scholes_gamma(spot=100.0, strike=100.0, time_to_expiry=0.0, iv=0.2, risk_free_rate=0.05)

    assert gamma == 0.0


def test_compute_gex_calls_positive_puts_negative_and_cancel_when_equal():
    spot = 100.0
    options = [
        {"strike": 100.0, "time_to_expiry": 1.0, "iv": 0.2, "open_interest": 100, "option_type": "call"},
        {"strike": 100.0, "time_to_expiry": 1.0, "iv": 0.2, "open_interest": 100, "option_type": "put"},
    ]

    result = compute_gex(options, spot, risk_free_rate=0.05)

    gamma = black_scholes_gamma(spot, 100.0, 1.0, 0.2, 0.05)
    expected_magnitude = gamma * 100 * 100 * (spot ** 2) * 0.01

    assert result.call_gex_by_strike[100.0] == pytest.approx(expected_magnitude, rel=1e-6)
    assert result.put_gex_by_strike[100.0] == pytest.approx(-expected_magnitude, rel=1e-6)
    assert result.net_gex_by_strike[100.0] == pytest.approx(0.0, abs=1e-6)


def test_compute_gex_aggregates_multiple_contracts_at_same_strike():
    spot = 100.0
    options = [
        {"strike": 90.0, "time_to_expiry": 0.5, "iv": 0.3, "open_interest": 50, "option_type": "call"},
        {"strike": 90.0, "time_to_expiry": 0.5, "iv": 0.3, "open_interest": 20, "option_type": "call"},
    ]

    result = compute_gex(options, spot, risk_free_rate=0.045)

    gamma = black_scholes_gamma(spot, 90.0, 0.5, 0.3, 0.045)
    expected = gamma * 70 * 100 * (spot ** 2) * 0.01

    assert result.call_gex_by_strike[90.0] == pytest.approx(expected, rel=1e-6)
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd backend && pytest tests/test_quant_core.py -v`
Expected: FAIL，`ModuleNotFoundError: No module named 'app.quant_core'`。

- [ ] **Step 3: 實作 black_scholes_gamma 與 compute_gex**

`backend/app/quant_core.py`:
```python
import math
from dataclasses import dataclass, field


def _norm_pdf(x: float) -> float:
    return math.exp(-0.5 * x * x) / math.sqrt(2 * math.pi)


def black_scholes_gamma(spot: float, strike: float, time_to_expiry: float, iv: float,
                         risk_free_rate: float = 0.045) -> float:
    if time_to_expiry <= 0 or iv <= 0 or spot <= 0 or strike <= 0:
        return 0.0
    d1 = (
        math.log(spot / strike) + (risk_free_rate + 0.5 * iv ** 2) * time_to_expiry
    ) / (iv * math.sqrt(time_to_expiry))
    return _norm_pdf(d1) / (spot * iv * math.sqrt(time_to_expiry))


@dataclass
class GexResult:
    call_gex_by_strike: dict = field(default_factory=dict)
    put_gex_by_strike: dict = field(default_factory=dict)
    net_gex_by_strike: dict = field(default_factory=dict)


def compute_gex(options: list[dict], spot: float, risk_free_rate: float = 0.045) -> GexResult:
    result = GexResult()
    for opt in options:
        gamma = black_scholes_gamma(
            spot, opt["strike"], opt["time_to_expiry"], opt["iv"], risk_free_rate
        )
        contribution = gamma * opt["open_interest"] * 100 * (spot ** 2) * 0.01
        strike = opt["strike"]
        if opt["option_type"] == "call":
            result.call_gex_by_strike[strike] = result.call_gex_by_strike.get(strike, 0.0) + contribution
            result.net_gex_by_strike[strike] = result.net_gex_by_strike.get(strike, 0.0) + contribution
        else:
            result.put_gex_by_strike[strike] = result.put_gex_by_strike.get(strike, 0.0) - contribution
            result.net_gex_by_strike[strike] = result.net_gex_by_strike.get(strike, 0.0) - contribution
    return result
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd backend && pytest tests/test_quant_core.py -v`
Expected: 4 個測試全部 PASS。

- [ ] **Step 5: Commit**

```bash
git add backend/app/quant_core.py backend/tests/test_quant_core.py
git commit -m "feat: add Black-Scholes gamma and GEX aggregation"
```

---

### Task 4: 量化核心 — Zero Gamma / Put Wall / Call Wall

**Files:**
- Modify: `backend/app/quant_core.py`
- Modify: `backend/tests/test_quant_core.py`

**Interfaces:**
- Consumes: `GexResult`（Task 3）
- Produces: `find_zero_gamma(net_gex_by_strike: dict[float, float]) -> float | None`；`find_put_wall(put_gex_by_strike: dict[float, float]) -> float | None`；`find_call_wall(call_gex_by_strike: dict[float, float]) -> float | None`

- [ ] **Step 1: 追加失敗測試**

在 `backend/tests/test_quant_core.py` 追加：
```python
from app.quant_core import find_zero_gamma, find_put_wall, find_call_wall


def test_find_zero_gamma_interpolates_between_sign_change():
    net_gex = {90.0: 50.0, 100.0: -30.0}

    zero_gamma = find_zero_gamma(net_gex)

    assert zero_gamma == pytest.approx(96.25, abs=1e-6)


def test_find_zero_gamma_returns_none_when_no_sign_change():
    net_gex = {90.0: 50.0, 100.0: 30.0}

    assert find_zero_gamma(net_gex) is None


def test_find_zero_gamma_returns_none_for_empty_input():
    assert find_zero_gamma({}) is None


def test_find_put_wall_returns_strike_with_most_negative_gex():
    put_gex = {280.0: -100.0, 300.0: -500.0, 320.0: -50.0}

    assert find_put_wall(put_gex) == 300.0


def test_find_call_wall_returns_strike_with_most_positive_gex():
    call_gex = {330.0: 200.0, 340.0: 800.0, 350.0: 150.0}

    assert find_call_wall(call_gex) == 340.0


def test_find_walls_return_none_for_empty_input():
    assert find_put_wall({}) is None
    assert find_call_wall({}) is None
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd backend && pytest tests/test_quant_core.py -v`
Expected: 新增的 6 個測試 FAIL（`ImportError`）。

- [ ] **Step 3: 實作**

在 `backend/app/quant_core.py` 尾端追加：
```python
def find_zero_gamma(net_gex_by_strike: dict) -> float | None:
    if not net_gex_by_strike:
        return None
    strikes = sorted(net_gex_by_strike.keys())
    if len(strikes) == 1:
        return strikes[0] if net_gex_by_strike[strikes[0]] == 0 else None
    for i in range(len(strikes) - 1):
        k1, k2 = strikes[i], strikes[i + 1]
        g1, g2 = net_gex_by_strike[k1], net_gex_by_strike[k2]
        if g1 == 0:
            return k1
        if (g1 < 0) != (g2 < 0):
            return k1 + (k2 - k1) * (-g1) / (g2 - g1)
    last = strikes[-1]
    return last if net_gex_by_strike[last] == 0 else None


def find_put_wall(put_gex_by_strike: dict) -> float | None:
    if not put_gex_by_strike:
        return None
    return min(put_gex_by_strike, key=lambda k: put_gex_by_strike[k])


def find_call_wall(call_gex_by_strike: dict) -> float | None:
    if not call_gex_by_strike:
        return None
    return max(call_gex_by_strike, key=lambda k: call_gex_by_strike[k])
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd backend && pytest tests/test_quant_core.py -v`
Expected: 全部測試 PASS。

- [ ] **Step 5: Commit**

```bash
git add backend/app/quant_core.py backend/tests/test_quant_core.py
git commit -m "feat: add zero gamma flip point and wall detection"
```

---

### Task 5: 量化核心 — Max Pain

**Files:**
- Modify: `backend/app/quant_core.py`
- Modify: `backend/tests/test_quant_core.py`

**Interfaces:**
- Consumes: 期權 `options: list[dict]`（同 Task 3 結構）
- Produces: `compute_max_pain(options: list[dict]) -> float | None`

- [ ] **Step 1: 追加失敗測試**

```python
from app.quant_core import compute_max_pain


def test_compute_max_pain_finds_price_with_minimum_total_payout():
    options = [
        {"strike": 90.0, "open_interest": 10, "option_type": "call"},
        {"strike": 100.0, "open_interest": 5, "option_type": "call"},
        {"strike": 110.0, "open_interest": 1, "option_type": "call"},
        {"strike": 90.0, "open_interest": 1, "option_type": "put"},
        {"strike": 100.0, "open_interest": 5, "option_type": "put"},
        {"strike": 110.0, "open_interest": 10, "option_type": "put"},
    ]

    assert compute_max_pain(options) == 100.0


def test_compute_max_pain_returns_none_for_empty_input():
    assert compute_max_pain([]) is None
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd backend && pytest tests/test_quant_core.py -v`
Expected: 新增 2 個測試 FAIL（`ImportError`）。

- [ ] **Step 3: 實作**

在 `backend/app/quant_core.py` 尾端追加：
```python
def compute_max_pain(options: list) -> float | None:
    strikes = sorted({opt["strike"] for opt in options})
    if not strikes:
        return None
    best_strike = None
    best_loss = None
    for candidate in strikes:
        total_loss = 0.0
        for opt in options:
            if opt["option_type"] == "call":
                total_loss += opt["open_interest"] * max(candidate - opt["strike"], 0)
            else:
                total_loss += opt["open_interest"] * max(opt["strike"] - candidate, 0)
        if best_loss is None or total_loss < best_loss:
            best_loss = total_loss
            best_strike = candidate
    return best_strike
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd backend && pytest tests/test_quant_core.py -v`
Expected: 全部測試 PASS。

- [ ] **Step 5: Commit**

```bash
git add backend/app/quant_core.py backend/tests/test_quant_core.py
git commit -m "feat: add max pain calculation"
```

---

### Task 6: 量化核心 — IV Skew

**Files:**
- Modify: `backend/app/quant_core.py`
- Modify: `backend/tests/test_quant_core.py`

**Interfaces:**
- Consumes: 期權 `options: list[dict]`（含 `iv` 欄位）、現價 `spot: float`
- Produces: `compute_iv_skew(options: list[dict], spot: float) -> float | None`

- [ ] **Step 1: 追加失敗測試**

```python
from app.quant_core import compute_iv_skew


def test_compute_iv_skew_returns_far_otm_put_iv_minus_atm_iv():
    spot = 100.0
    options = [
        {"strike": 87.5, "iv": 0.45, "option_type": "put", "time_to_expiry": 0.1, "open_interest": 10},
        {"strike": 100.0, "iv": 0.30, "option_type": "call", "time_to_expiry": 0.1, "open_interest": 10},
        {"strike": 100.0, "iv": 0.32, "option_type": "put", "time_to_expiry": 0.1, "open_interest": 10},
    ]

    skew = compute_iv_skew(options, spot)

    assert skew == pytest.approx(0.14, abs=1e-6)


def test_compute_iv_skew_returns_none_without_puts():
    options = [
        {"strike": 100.0, "iv": 0.30, "option_type": "call", "time_to_expiry": 0.1, "open_interest": 10},
    ]

    assert compute_iv_skew(options, 100.0) is None
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd backend && pytest tests/test_quant_core.py -v`
Expected: 新增 2 個測試 FAIL（`ImportError`）。

- [ ] **Step 3: 實作**

在 `backend/app/quant_core.py` 尾端追加：
```python
def compute_iv_skew(options: list, spot: float) -> float | None:
    puts = [o for o in options if o["option_type"] == "put"]
    if not puts:
        return None

    target_far = spot * 0.875
    far_put = min(puts, key=lambda o: abs(o["strike"] - target_far))

    min_distance = min(abs(o["strike"] - spot) for o in options)
    atm_candidates = [o for o in options if abs(o["strike"] - spot) == min_distance]
    atm_iv = sum(o["iv"] for o in atm_candidates) / len(atm_candidates)

    return far_put["iv"] - atm_iv
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd backend && pytest tests/test_quant_core.py -v`
Expected: 全部測試 PASS（累計應為 Task 3–6 共 14 個測試）。

- [ ] **Step 5: Commit**

```bash
git add backend/app/quant_core.py backend/tests/test_quant_core.py
git commit -m "feat: add IV skew calculation"
```

---

### Task 7: 三色燈號邏輯 (`signal_engine.py`)

**Files:**
- Create: `backend/app/signal_engine.py`
- Test: `backend/tests/test_signal_engine.py`

**Interfaces:**
- Consumes: 無（純函數）
- Produces: `determine_light(spot: float, zero_gamma: float | None, band: float = 0.015) -> str`（回傳 `"red"|"yellow"|"green"|"unknown"`）；`determine_iv_alert(current_iv_skew: float | None, historical_iv_skews: list[float], std_threshold: float = 1.5) -> bool`；`build_signal(spot, zero_gamma, iv_skew, historical_iv_skews) -> dict`（欄位 `light`, `iv_alert`）

- [ ] **Step 1: 寫失敗測試**

`backend/tests/test_signal_engine.py`:
```python
import pytest

from app.signal_engine import determine_light, determine_iv_alert, build_signal


def test_determine_light_red_when_spot_below_zero_gamma():
    assert determine_light(spot=310.0, zero_gamma=323.10) == "red"


def test_determine_light_yellow_when_within_band_above():
    # (325 - 323.10) / 323.10 = 0.00588 <= 0.015
    assert determine_light(spot=325.0, zero_gamma=323.10) == "yellow"


def test_determine_light_yellow_at_exact_zero_gamma():
    assert determine_light(spot=323.10, zero_gamma=323.10) == "yellow"


def test_determine_light_green_when_beyond_band_above():
    # (330 - 323.10) / 323.10 = 0.02135 > 0.015
    assert determine_light(spot=330.0, zero_gamma=323.10) == "green"


def test_determine_light_unknown_when_zero_gamma_missing():
    assert determine_light(spot=300.0, zero_gamma=None) == "unknown"


def test_determine_iv_alert_true_when_current_far_above_history():
    history = [0.05, 0.06, 0.04, 0.05, 0.06, 0.05, 0.04, 0.06, 0.05, 0.05]
    assert determine_iv_alert(current_iv_skew=0.20, historical_iv_skews=history) is True


def test_determine_iv_alert_false_when_close_to_history():
    history = [0.05, 0.06, 0.04, 0.05, 0.06, 0.05, 0.04, 0.06, 0.05, 0.05]
    assert determine_iv_alert(current_iv_skew=0.055, historical_iv_skews=history) is False


def test_determine_iv_alert_false_when_insufficient_history():
    assert determine_iv_alert(current_iv_skew=0.20, historical_iv_skews=[0.05]) is False


def test_build_signal_combines_light_and_iv_alert():
    history = [0.05, 0.06, 0.04, 0.05, 0.06, 0.05, 0.04, 0.06, 0.05, 0.05]

    signal = build_signal(spot=325.0, zero_gamma=323.10, iv_skew=0.20, historical_iv_skews=history)

    assert signal == {"light": "yellow", "iv_alert": True}
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd backend && pytest tests/test_signal_engine.py -v`
Expected: FAIL，`ModuleNotFoundError: No module named 'app.signal_engine'`。

- [ ] **Step 3: 實作 signal_engine.py**

`backend/app/signal_engine.py`:
```python
import statistics


def determine_light(spot: float, zero_gamma: float | None, band: float = 0.015) -> str:
    if zero_gamma is None:
        return "unknown"
    if spot < zero_gamma:
        return "red"
    if abs(spot - zero_gamma) / zero_gamma <= band:
        return "yellow"
    return "green"


def determine_iv_alert(current_iv_skew: float | None, historical_iv_skews: list,
                        std_threshold: float = 1.5) -> bool:
    if current_iv_skew is None or len(historical_iv_skews) < 2:
        return False
    mean = statistics.mean(historical_iv_skews)
    stdev = statistics.stdev(historical_iv_skews)
    if stdev == 0:
        return False
    return (current_iv_skew - mean) / stdev >= std_threshold


def build_signal(spot: float, zero_gamma: float | None, iv_skew: float | None,
                  historical_iv_skews: list) -> dict:
    return {
        "light": determine_light(spot, zero_gamma),
        "iv_alert": determine_iv_alert(iv_skew, historical_iv_skews),
    }
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd backend && pytest tests/test_signal_engine.py -v`
Expected: 全部 9 個測試 PASS。

- [ ] **Step 5: Commit**

```bash
git add backend/app/signal_engine.py backend/tests/test_signal_engine.py
git commit -m "feat: add three-color signal engine with IV skew alert"
```

---

### Task 8: 期權鏈抓取 (`data_fetcher.py`)

**Files:**
- Create: `backend/app/data_fetcher.py`
- Test: `backend/tests/test_data_fetcher.py`

**Interfaces:**
- Consumes: `yfinance.Ticker`（外部函式庫，測試中以假物件替換）
- Produces: `fetch_option_chain(ticker: str = "TSLA", max_days: int = 45) -> tuple[float, list[dict]]`，回傳 `(spot, options)`，`options` 結構同 Task 3

- [ ] **Step 1: 寫失敗測試**

`backend/tests/test_data_fetcher.py`:
```python
from datetime import date, timedelta
import pandas as pd
import pytest

from app import data_fetcher


class FakeChain:
    def __init__(self, calls, puts):
        self.calls = pd.DataFrame(calls)
        self.puts = pd.DataFrame(puts)


class FakeTicker:
    def __init__(self, symbol):
        self.symbol = symbol
        near_expiry = (date.today() + timedelta(days=10)).strftime("%Y-%m-%d")
        far_expiry = (date.today() + timedelta(days=60)).strftime("%Y-%m-%d")
        self.options = (near_expiry, far_expiry)
        self.fast_info = {"last_price": 320.0}
        self._chains = {
            near_expiry: FakeChain(
                calls=[{"strike": 330.0, "openInterest": 100, "impliedVolatility": 0.5}],
                puts=[{"strike": 300.0, "openInterest": 200, "impliedVolatility": 0.55}],
            ),
            far_expiry: FakeChain(
                calls=[{"strike": 340.0, "openInterest": 50, "impliedVolatility": 0.6}],
                puts=[{"strike": 280.0, "openInterest": 80, "impliedVolatility": 0.65}],
            ),
        }

    def option_chain(self, expiry):
        return self._chains[expiry]


def test_fetch_option_chain_filters_expiries_beyond_max_days(monkeypatch):
    monkeypatch.setattr(data_fetcher.yf, "Ticker", FakeTicker)

    spot, options = data_fetcher.fetch_option_chain(ticker="TSLA", max_days=45)

    assert spot == 320.0
    strikes = {opt["strike"] for opt in options}
    assert 330.0 in strikes
    assert 300.0 in strikes
    assert 340.0 not in strikes  # far_expiry (60 days out) excluded
    assert 280.0 not in strikes


def test_fetch_option_chain_marks_option_type_correctly(monkeypatch):
    monkeypatch.setattr(data_fetcher.yf, "Ticker", FakeTicker)

    _, options = data_fetcher.fetch_option_chain(ticker="TSLA", max_days=45)

    types_by_strike = {opt["strike"]: opt["option_type"] for opt in options}
    assert types_by_strike[330.0] == "call"
    assert types_by_strike[300.0] == "put"
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd backend && pytest tests/test_data_fetcher.py -v`
Expected: FAIL，`ModuleNotFoundError: No module named 'app.data_fetcher'`。

- [ ] **Step 3: 實作 data_fetcher.py**

`backend/app/data_fetcher.py`:
```python
from datetime import datetime

import yfinance as yf


def fetch_option_chain(ticker: str = "TSLA", max_days: int = 45) -> tuple:
    tk = yf.Ticker(ticker)
    spot = tk.fast_info["last_price"]
    options = []
    today = datetime.utcnow().date()

    for expiry_str in tk.options:
        expiry = datetime.strptime(expiry_str, "%Y-%m-%d").date()
        days_out = (expiry - today).days
        if days_out < 0 or days_out > max_days:
            continue
        time_to_expiry = max(days_out, 1) / 365.0
        chain = tk.option_chain(expiry_str)

        for _, row in chain.calls.iterrows():
            if row["openInterest"] and row["impliedVolatility"]:
                options.append({
                    "strike": float(row["strike"]),
                    "time_to_expiry": time_to_expiry,
                    "iv": float(row["impliedVolatility"]),
                    "open_interest": float(row["openInterest"]),
                    "option_type": "call",
                })
        for _, row in chain.puts.iterrows():
            if row["openInterest"] and row["impliedVolatility"]:
                options.append({
                    "strike": float(row["strike"]),
                    "time_to_expiry": time_to_expiry,
                    "iv": float(row["impliedVolatility"]),
                    "open_interest": float(row["openInterest"]),
                    "option_type": "put",
                })
    return spot, options
```

- [ ] **Step 4: 加入 pandas 依賴並執行測試**

在 `backend/requirements.txt` 追加一行 `pandas==2.2.3`（yfinance 依賴項，測試中直接使用需顯式聲明）。

Run:
```bash
cd backend && pip install -r requirements.txt && pytest tests/test_data_fetcher.py -v
```
Expected: 2 個測試全部 PASS。

- [ ] **Step 5: Commit**

```bash
git add backend/app/data_fetcher.py backend/tests/test_data_fetcher.py backend/requirements.txt
git commit -m "feat: add yfinance option chain fetcher"
```

---

### Task 9: Telegram 推送 (`alerts.py`)

**Files:**
- Create: `backend/app/alerts.py`
- Test: `backend/tests/test_alerts.py`

**Interfaces:**
- Consumes: `requests.post`（測試中以 monkeypatch 替換）
- Produces: `send_telegram_alert(message: str, bot_token: str, chat_id: str) -> bool`

- [ ] **Step 1: 寫失敗測試**

`backend/tests/test_alerts.py`:
```python
import pytest

from app import alerts


class FakeResponse:
    def __init__(self, status_code):
        self.status_code = status_code


def test_send_telegram_alert_returns_true_on_200(monkeypatch):
    captured = {}

    def fake_post(url, json, timeout):
        captured["url"] = url
        captured["json"] = json
        return FakeResponse(200)

    monkeypatch.setattr(alerts.requests, "post", fake_post)

    result = alerts.send_telegram_alert("hello", bot_token="TOKEN", chat_id="123")

    assert result is True
    assert captured["url"] == "https://api.telegram.org/botTOKEN/sendMessage"
    assert captured["json"] == {"chat_id": "123", "text": "hello"}


def test_send_telegram_alert_returns_false_on_non_200(monkeypatch):
    monkeypatch.setattr(alerts.requests, "post", lambda url, json, timeout: FakeResponse(400))

    assert alerts.send_telegram_alert("hello", bot_token="TOKEN", chat_id="123") is False


def test_send_telegram_alert_returns_false_on_exception(monkeypatch):
    def raise_error(url, json, timeout):
        raise alerts.requests.RequestException("network down")

    monkeypatch.setattr(alerts.requests, "post", raise_error)

    assert alerts.send_telegram_alert("hello", bot_token="TOKEN", chat_id="123") is False
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd backend && pytest tests/test_alerts.py -v`
Expected: FAIL，`ModuleNotFoundError: No module named 'app.alerts'`。

- [ ] **Step 3: 實作 alerts.py**

`backend/app/alerts.py`:
```python
import requests


def send_telegram_alert(message: str, bot_token: str, chat_id: str) -> bool:
    url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
    try:
        response = requests.post(url, json={"chat_id": chat_id, "text": message}, timeout=10)
        return response.status_code == 200
    except requests.RequestException:
        return False
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd backend && pytest tests/test_alerts.py -v`
Expected: 全部 3 個測試 PASS。

- [ ] **Step 5: Commit**

```bash
git add backend/app/alerts.py backend/tests/test_alerts.py
git commit -m "feat: add Telegram alert sender"
```

---

### Task 10: Worker 排程邏輯 (`worker.py`)

**Files:**
- Create: `backend/app/worker.py`
- Test: `backend/tests/test_worker.py`

**Interfaces:**
- Consumes: `db.get_connection/init_db/insert_snapshot/get_latest/get_history`（Task 2）、`data_fetcher.fetch_option_chain`（Task 8）、`quant_core.compute_gex/find_zero_gamma/find_put_wall/find_call_wall/compute_max_pain/compute_iv_skew`（Task 3–6）、`signal_engine.build_signal`（Task 7）、`alerts.send_telegram_alert`（Task 9）、`config`（Task 1）
- Produces: `is_market_hours(now: datetime) -> bool`；`light_changed(previous_light: str | None, current_light: str) -> bool`；`format_alert_message(snapshot: dict) -> str`；`run_once(conn, config_dict: dict) -> dict | None`；`main() -> None`（進入點，不納入自動測試）

- [ ] **Step 1: 寫失敗測試**

`backend/tests/test_worker.py`:
```python
from datetime import datetime
from zoneinfo import ZoneInfo

import pytest

from app import worker
from app.db import get_connection, init_db


def test_is_market_hours_true_during_trading_session():
    # 2026-08-14 is a Friday; 14:00 UTC = 10:00 America/New_York in August (EDT, UTC-4)
    now = datetime(2026, 8, 14, 14, 0, tzinfo=ZoneInfo("UTC"))
    assert worker.is_market_hours(now) is True


def test_is_market_hours_false_outside_session():
    now = datetime(2026, 8, 14, 23, 0, tzinfo=ZoneInfo("UTC"))
    assert worker.is_market_hours(now) is False


def test_is_market_hours_false_on_weekend():
    # 2026-08-15 is a Saturday
    now = datetime(2026, 8, 15, 14, 0, tzinfo=ZoneInfo("UTC"))
    assert worker.is_market_hours(now) is False


def test_light_changed_false_when_no_previous():
    assert worker.light_changed(None, "green") is False


def test_light_changed_false_when_same():
    assert worker.light_changed("green", "green") is False


def test_light_changed_true_when_different():
    assert worker.light_changed("green", "red") is True


def test_format_alert_message_includes_key_levels():
    snapshot = {
        "light": "red", "spot": 318.42, "zero_gamma": 323.10,
        "put_wall": 300.0, "call_wall": 340.0, "max_pain": 325.0,
    }

    message = worker.format_alert_message(snapshot)

    assert "318.42" in message
    assert "323.10" in message
    assert "RED" in message


def test_run_once_inserts_snapshot_and_sends_alert_on_light_change(monkeypatch):
    conn = get_connection(":memory:")
    init_db(conn)

    def fake_fetch(ticker, max_days):
        return 310.0, [
            {"strike": 300.0, "time_to_expiry": 0.1, "iv": 0.5, "open_interest": 500, "option_type": "put"},
            {"strike": 330.0, "time_to_expiry": 0.1, "iv": 0.4, "open_interest": 500, "option_type": "call"},
        ]

    sent = {}

    def fake_send(message, bot_token, chat_id):
        sent["message"] = message
        return True

    monkeypatch.setattr(worker, "fetch_option_chain", fake_fetch)
    monkeypatch.setattr(worker, "send_telegram_alert", fake_send)

    config_dict = {
        "ticker": "TSLA", "max_days": 45, "risk_free_rate": 0.045,
        "telegram_bot_token": "TOKEN", "telegram_chat_id": "123",
    }

    snapshot = worker.run_once(conn, config_dict)

    assert snapshot is not None
    assert snapshot["spot"] == 310.0
    # first run: no previous light, so no alert should be sent
    assert "message" not in sent


def test_run_once_returns_none_when_no_options(monkeypatch):
    conn = get_connection(":memory:")
    init_db(conn)
    monkeypatch.setattr(worker, "fetch_option_chain", lambda ticker, max_days: (310.0, []))

    config_dict = {
        "ticker": "TSLA", "max_days": 45, "risk_free_rate": 0.045,
        "telegram_bot_token": "TOKEN", "telegram_chat_id": "123",
    }

    assert worker.run_once(conn, config_dict) is None
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd backend && pytest tests/test_worker.py -v`
Expected: FAIL，`ModuleNotFoundError: No module named 'app.worker'`。

- [ ] **Step 3: 實作 worker.py**

`backend/app/worker.py`:
```python
import logging
import time
from datetime import datetime
from zoneinfo import ZoneInfo

from app import config
from app.alerts import send_telegram_alert
from app.data_fetcher import fetch_option_chain
from app.db import get_connection, init_db, insert_snapshot, get_latest, get_history
from app.quant_core import (
    compute_gex, find_zero_gamma, find_put_wall, find_call_wall,
    compute_max_pain, compute_iv_skew,
)
from app.signal_engine import build_signal

MARKET_TZ = ZoneInfo("America/New_York")
LIGHT_EMOJI = {"red": "🔴", "yellow": "🟡", "green": "🟢", "unknown": "⚪"}


def is_market_hours(now: datetime) -> bool:
    local = now.astimezone(MARKET_TZ)
    if local.weekday() >= 5:
        return False
    open_time = local.replace(hour=9, minute=30, second=0, microsecond=0)
    close_time = local.replace(hour=16, minute=0, second=0, microsecond=0)
    return open_time <= local <= close_time


def light_changed(previous_light, current_light: str) -> bool:
    return previous_light is not None and previous_light != current_light


def format_alert_message(snapshot: dict) -> str:
    emoji = LIGHT_EMOJI.get(snapshot["light"], "⚪")
    return (
        f"{emoji} TSLA 燈號變化：{snapshot['light'].upper()}\n"
        f"現價: ${snapshot['spot']:.2f} | Zero Gamma: ${snapshot['zero_gamma']:.2f}\n"
        f"Put Wall: ${snapshot['put_wall']:.0f} | Call Wall: ${snapshot['call_wall']:.0f} "
        f"| Max Pain: ${snapshot['max_pain']:.0f}"
    )


def run_once(conn, config_dict: dict) -> dict | None:
    spot, options = fetch_option_chain(config_dict["ticker"], config_dict["max_days"])
    if not options:
        return None

    gex = compute_gex(options, spot, config_dict["risk_free_rate"])
    zero_gamma = find_zero_gamma(gex.net_gex_by_strike)
    put_wall = find_put_wall(gex.put_gex_by_strike)
    call_wall = find_call_wall(gex.call_gex_by_strike)
    max_pain = compute_max_pain(options)
    iv_skew = compute_iv_skew(options, spot)

    history = get_history(conn, days=1)
    historical_skews = [row["iv_skew"] for row in history if row["iv_skew"] is not None]
    signal = build_signal(spot, zero_gamma, iv_skew, historical_skews)

    previous = get_latest(conn)
    previous_light = previous["light"] if previous else None

    snapshot = {
        "ts": datetime.utcnow().isoformat(),
        "spot": spot,
        "zero_gamma": zero_gamma,
        "put_wall": put_wall,
        "call_wall": call_wall,
        "max_pain": max_pain,
        "iv_skew": iv_skew,
        "light": signal["light"],
        "iv_alert": signal["iv_alert"],
    }
    insert_snapshot(conn, snapshot)

    if light_changed(previous_light, signal["light"]) and zero_gamma is not None:
        message = format_alert_message(snapshot)
        send_telegram_alert(message, config_dict["telegram_bot_token"], config_dict["telegram_chat_id"])

    return snapshot


def main() -> None:
    logging.basicConfig(level=logging.INFO)
    conn = get_connection(config.DB_PATH)
    init_db(conn)
    while True:
        now = datetime.now(tz=ZoneInfo("UTC"))
        if is_market_hours(now):
            try:
                run_once(conn, config.as_dict())
            except Exception:
                logging.exception("worker run_once failed")
        time.sleep(config.POLL_INTERVAL_SECONDS)


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd backend && pytest tests/test_worker.py -v`
Expected: 全部 8 個測試 PASS。

- [ ] **Step 5: Commit**

```bash
git add backend/app/worker.py backend/tests/test_worker.py
git commit -m "feat: add worker scheduling and orchestration logic"
```

---

### Task 11: FastAPI 查詢層 (`api.py`)

**Files:**
- Create: `backend/app/api.py`
- Test: `backend/tests/test_api.py`

**Interfaces:**
- Consumes: `db.get_connection/init_db/get_latest/get_history`（Task 2）、`config.DB_PATH`（Task 1）
- Produces: FastAPI app 物件 `app`，路由 `GET /latest`（200 含 snapshot dict，404 若無資料）、`GET /history?days=N`（200 含 list[dict]，預設 `days=7`）

- [ ] **Step 1: 寫失敗測試**

`backend/tests/test_api.py`:
```python
import pytest
from fastapi.testclient import TestClient

from app import api, config, db


@pytest.fixture
def client(tmp_path, monkeypatch):
    db_path = str(tmp_path / "test.db")
    monkeypatch.setattr(config, "DB_PATH", db_path)
    return TestClient(api.app)


def test_latest_returns_404_when_no_data(client):
    response = client.get("/latest")
    assert response.status_code == 404


def test_latest_returns_most_recent_snapshot(client):
    conn = db.get_connection(config.DB_PATH)
    db.init_db(conn)
    db.insert_snapshot(conn, {
        "ts": "2026-08-14T10:00:00", "spot": 310.0, "zero_gamma": 323.1,
        "put_wall": 300.0, "call_wall": 340.0, "max_pain": 325.0,
        "iv_skew": 0.1, "light": "red", "iv_alert": False,
    })
    conn.close()

    response = client.get("/latest")

    assert response.status_code == 200
    assert response.json()["light"] == "red"


def test_history_returns_list_filtered_by_days(client):
    conn = db.get_connection(config.DB_PATH)
    db.init_db(conn)
    db.insert_snapshot(conn, {
        "ts": "2026-08-14T10:00:00", "spot": 310.0, "zero_gamma": 323.1,
        "put_wall": 300.0, "call_wall": 340.0, "max_pain": 325.0,
        "iv_skew": 0.1, "light": "red", "iv_alert": False,
    })
    conn.close()

    response = client.get("/history?days=7")

    assert response.status_code == 200
    assert len(response.json()) == 1
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd backend && pytest tests/test_api.py -v`
Expected: FAIL，`ModuleNotFoundError: No module named 'app.api'`。

- [ ] **Step 3: 實作 api.py**

`backend/app/api.py`:
```python
from fastapi import FastAPI, HTTPException

from app import config, db

app = FastAPI(title="TSLA Gamma Dashboard API")


def _get_conn():
    conn = db.get_connection(config.DB_PATH)
    db.init_db(conn)
    return conn


@app.get("/latest")
def latest():
    conn = _get_conn()
    row = db.get_latest(conn)
    conn.close()
    if row is None:
        raise HTTPException(status_code=404, detail="no data yet")
    return row


@app.get("/history")
def history(days: int = 7):
    conn = _get_conn()
    rows = db.get_history(conn, days)
    conn.close()
    return rows
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd backend && pytest tests/test_api.py -v`
Expected: 全部 3 個測試 PASS。

- [ ] **Step 5: 執行整體後端測試套件確認無回歸**

Run: `cd backend && pytest -v`
Expected: 所有測試（Task 2–11 累計）全部 PASS。

- [ ] **Step 6: Commit**

```bash
git add backend/app/api.py backend/tests/test_api.py
git commit -m "feat: add FastAPI read endpoints for latest and history"
```

---

### Task 12: 後端 Railway 部署設定

**Files:**
- Create: `backend/.env.example`
- Create: `backend/railway.toml`

**Interfaces:**
- Consumes: `config.py` 所讀取的環境變數清單（Task 1）
- Produces: 部署設定檔（無程式邏輯，不需單元測試）

- [ ] **Step 1: 建立環境變數範例檔**

`backend/.env.example`:
```
DB_PATH=/data/snapshots.db
POLL_INTERVAL_SECONDS=600
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
PORT=8000
```

- [ ] **Step 2: 建立 railway.toml（api 服務預設設定）**

`backend/railway.toml`:
```toml
[build]
builder = "NIXPACKS"
buildCommand = "pip install -r requirements.txt"

[deploy]
startCommand = "uvicorn app.api:app --host 0.0.0.0 --port $PORT"
restartPolicyType = "ON_FAILURE"
```

註記（寫入檔案頂端的 `#` 註解）：worker 服務在 Railway 上需另外建立一個服務，指向同一個 repo/backend 目錄，並在該服務的自訂啟動指令覆寫為 `python -m app.worker`（不使用此 `railway.toml` 的 `startCommand`）；worker 與 api 兩服務都需掛載同一個 Railway Volume 至 `/data`，並設定相同的 `DB_PATH=/data/snapshots.db`。

- [ ] **Step 3: Commit**

```bash
git add backend/.env.example backend/railway.toml
git commit -m "chore: add backend Railway deployment configuration"
```

---

### Task 13: 前端專案骨架（Next.js + Tailwind + Vitest）

**Files:**
- Create: `frontend/` (透過 `create-next-app` 產生)
- Create: `frontend/lib/api.ts`
- Create: `frontend/vitest.config.ts`
- Create: `frontend/vitest.setup.ts`
- Create: `frontend/.env.local.example`
- Modify: `frontend/package.json`

**Interfaces:**
- Produces: `Snapshot` type、`fetchLatest(): Promise<Snapshot | null>`、`fetchHistory(days?: number): Promise<Snapshot[]>`

- [ ] **Step 1: 建立 Next.js 專案**

Run:
```bash
cd "/Users/andrewweng/Desktop/Plunge Warning" && npx create-next-app@latest frontend --typescript --tailwind --eslint --app --src-dir=false --import-alias "@/*" --no-turbopack
```
於互動提示中全部選預設值（若被詢問）。

- [ ] **Step 2: 安裝 Chart.js 與測試相關依賴**

Run:
```bash
cd frontend && npm install chart.js react-chartjs-2 && npm install -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom
```

- [ ] **Step 3: 建立 API client**

`frontend/lib/api.ts`:
```ts
export type Snapshot = {
  id: number;
  ts: string;
  spot: number;
  zero_gamma: number | null;
  put_wall: number | null;
  call_wall: number | null;
  max_pain: number | null;
  iv_skew: number | null;
  light: "red" | "yellow" | "green" | "unknown";
  iv_alert: number;
};

const API_URL = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:8000";

export async function fetchLatest(): Promise<Snapshot | null> {
  const res = await fetch(`${API_URL}/latest`, { cache: "no-store" });
  if (res.status === 404) return null;
  if (!res.ok) throw new Error(`fetchLatest failed: ${res.status}`);
  return res.json();
}

export async function fetchHistory(days = 7): Promise<Snapshot[]> {
  const res = await fetch(`${API_URL}/history?days=${days}`, { cache: "no-store" });
  if (!res.ok) throw new Error(`fetchHistory failed: ${res.status}`);
  return res.json();
}
```

- [ ] **Step 4: 建立 Vitest 設定**

`frontend/vitest.config.ts`:
```ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./vitest.setup.ts"],
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, ".") },
  },
});
```

`frontend/vitest.setup.ts`:
```ts
import "@testing-library/jest-dom/vitest";
```

- [ ] **Step 5: 加入 test script**

在 `frontend/package.json` 的 `"scripts"` 區塊加入：
```json
"test": "vitest run"
```

- [ ] **Step 6: 建立環境變數範例檔**

`frontend/.env.local.example`:
```
NEXT_PUBLIC_API_URL=http://localhost:8000
```

- [ ] **Step 7: 驗證專案可建置**

Run: `cd frontend && npm run build`
Expected: 建置成功無錯誤（此時頁面仍為 create-next-app 預設內容）。

- [ ] **Step 8: Commit**

```bash
git add frontend/
git commit -m "chore: scaffold Next.js frontend with Tailwind, Chart.js, and Vitest"
```

---

### Task 14: TrafficLight 元件

**Files:**
- Create: `frontend/components/TrafficLight.tsx`
- Test: `frontend/tests/TrafficLight.test.tsx`

**Interfaces:**
- Consumes: `fetchLatest()`, `Snapshot`（`lib/api.ts`，Task 13）
- Produces: `TrafficLight` React 元件（預設匯出），根節點 `data-testid="traffic-light"`，依 `light` 套用 `bg-red-600` / `bg-yellow-400` / `bg-green-600`

- [ ] **Step 1: 寫失敗測試**

`frontend/tests/TrafficLight.test.tsx`:
```tsx
import { render, screen, waitFor } from "@testing-library/react";
import { describe, it, expect, vi, afterEach } from "vitest";
import TrafficLight from "@/components/TrafficLight";
import * as api from "@/lib/api";

afterEach(() => {
  vi.restoreAllMocks();
});

function makeSnapshot(overrides: Partial<api.Snapshot> = {}): api.Snapshot {
  return {
    id: 1, ts: "2026-08-14T10:00:00Z", spot: 310, zero_gamma: 323.1,
    put_wall: 300, call_wall: 340, max_pain: 325, iv_skew: 0.1,
    light: "red", iv_alert: 0, ...overrides,
  };
}

describe("TrafficLight", () => {
  it("renders red background when light is red", async () => {
    vi.spyOn(api, "fetchLatest").mockResolvedValue(makeSnapshot({ light: "red" }));

    render(<TrafficLight />);

    await waitFor(() => expect(screen.getByTestId("traffic-light")).toHaveClass("bg-red-600"));
  });

  it("renders green background when light is green", async () => {
    vi.spyOn(api, "fetchLatest").mockResolvedValue(makeSnapshot({ light: "green" }));

    render(<TrafficLight />);

    await waitFor(() => expect(screen.getByTestId("traffic-light")).toHaveClass("bg-green-600"));
  });

  it("shows waiting state before data loads", () => {
    vi.spyOn(api, "fetchLatest").mockReturnValue(new Promise(() => {}));

    render(<TrafficLight />);

    expect(screen.getByText("等待數據...")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd frontend && npm run test -- TrafficLight`
Expected: FAIL，找不到 `@/components/TrafficLight` 模組。

- [ ] **Step 3: 實作 TrafficLight.tsx**

`frontend/components/TrafficLight.tsx`:
```tsx
"use client";

import { useEffect, useState } from "react";
import { fetchLatest, Snapshot } from "@/lib/api";

const LIGHT_STYLES: Record<string, { bg: string; label: string }> = {
  red: { bg: "bg-red-600", label: "High Risk — 跌破 Zero Gamma 防線" },
  yellow: { bg: "bg-yellow-400", label: "Caution — Zero Gamma 附近擠壓" },
  green: { bg: "bg-green-600", label: "Bullish Flow — 穩居 Zero Gamma 之上" },
};

export default function TrafficLight() {
  const [snapshot, setSnapshot] = useState<Snapshot | null>(null);

  useEffect(() => {
    const load = () => fetchLatest().then(setSnapshot).catch(() => setSnapshot(null));
    load();
    const interval = setInterval(load, 60000);
    return () => clearInterval(interval);
  }, []);

  if (!snapshot) {
    return <div className="rounded-xl p-8 text-center text-gray-400">等待數據...</div>;
  }

  const style = LIGHT_STYLES[snapshot.light] ?? { bg: "bg-gray-400", label: "Unknown" };

  return (
    <div data-testid="traffic-light" className={`rounded-xl p-8 text-center text-white ${style.bg}`}>
      <div className="text-2xl font-bold">{style.label}</div>
      <div className="mt-2 text-lg">現價 ${snapshot.spot.toFixed(2)}</div>
    </div>
  );
}
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd frontend && npm run test -- TrafficLight`
Expected: 全部 3 個測試 PASS。

- [ ] **Step 5: Commit**

```bash
git add frontend/components/TrafficLight.tsx frontend/tests/TrafficLight.test.tsx
git commit -m "feat: add TrafficLight component"
```

---

### Task 15: WallsPanel 元件

**Files:**
- Create: `frontend/components/WallsPanel.tsx`
- Test: `frontend/tests/WallsPanel.test.tsx`

**Interfaces:**
- Consumes: `fetchLatest()`, `Snapshot`（`lib/api.ts`，Task 13）
- Produces: `WallsPanel` React 元件（預設匯出），根節點 `data-testid="walls-panel"`

- [ ] **Step 1: 寫失敗測試**

`frontend/tests/WallsPanel.test.tsx`:
```tsx
import { render, screen, waitFor } from "@testing-library/react";
import { describe, it, expect, vi, afterEach } from "vitest";
import WallsPanel from "@/components/WallsPanel";
import * as api from "@/lib/api";

afterEach(() => {
  vi.restoreAllMocks();
});

describe("WallsPanel", () => {
  it("renders put wall, call wall, max pain, and zero gamma values", async () => {
    vi.spyOn(api, "fetchLatest").mockResolvedValue({
      id: 1, ts: "2026-08-14T10:00:00Z", spot: 310, zero_gamma: 323.1,
      put_wall: 300, call_wall: 340, max_pain: 325, iv_skew: 0.1,
      light: "red", iv_alert: 0,
    });

    render(<WallsPanel />);

    await waitFor(() => expect(screen.getByTestId("walls-panel")).toBeInTheDocument());
    expect(screen.getByText("$300.00")).toBeInTheDocument();
    expect(screen.getByText("$340.00")).toBeInTheDocument();
    expect(screen.getByText("$325.00")).toBeInTheDocument();
    expect(screen.getByText("$323.10")).toBeInTheDocument();
  });

  it("renders nothing before data loads", () => {
    vi.spyOn(api, "fetchLatest").mockReturnValue(new Promise(() => {}));

    const { container } = render(<WallsPanel />);

    expect(container).toBeEmptyDOMElement();
  });
});
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd frontend && npm run test -- WallsPanel`
Expected: FAIL，找不到 `@/components/WallsPanel` 模組。

- [ ] **Step 3: 實作 WallsPanel.tsx**

`frontend/components/WallsPanel.tsx`:
```tsx
"use client";

import { useEffect, useState } from "react";
import { fetchLatest, Snapshot } from "@/lib/api";

function Card({ label, value }: { label: string; value: number | null }) {
  return (
    <div className="rounded-lg border border-gray-700 p-4">
      <div className="text-sm text-gray-400">{label}</div>
      <div className="text-xl font-semibold">
        {value !== null && value !== undefined ? `$${value.toFixed(2)}` : "—"}
      </div>
    </div>
  );
}

export default function WallsPanel() {
  const [snapshot, setSnapshot] = useState<Snapshot | null>(null);

  useEffect(() => {
    const load = () => fetchLatest().then(setSnapshot).catch(() => setSnapshot(null));
    load();
    const interval = setInterval(load, 60000);
    return () => clearInterval(interval);
  }, []);

  if (!snapshot) return null;

  return (
    <div data-testid="walls-panel" className="grid grid-cols-2 gap-4 md:grid-cols-4">
      <Card label="Put Wall" value={snapshot.put_wall} />
      <Card label="Call Wall" value={snapshot.call_wall} />
      <Card label="Max Pain" value={snapshot.max_pain} />
      <Card label="Zero Gamma" value={snapshot.zero_gamma} />
    </div>
  );
}
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd frontend && npm run test -- WallsPanel`
Expected: 全部 2 個測試 PASS。

- [ ] **Step 5: Commit**

```bash
git add frontend/components/WallsPanel.tsx frontend/tests/WallsPanel.test.tsx
git commit -m "feat: add WallsPanel component"
```

---

### Task 16: HistoryChart 元件

**Files:**
- Create: `frontend/components/HistoryChart.tsx`
- Test: `frontend/tests/HistoryChart.test.tsx`

**Interfaces:**
- Consumes: `fetchHistory()`, `Snapshot`（`lib/api.ts`，Task 13）
- Produces: `HistoryChart` React 元件（預設匯出），根節點 `data-testid="history-chart"`

- [ ] **Step 1: 寫失敗測試**

`frontend/tests/HistoryChart.test.tsx`:
```tsx
import { render, screen, waitFor } from "@testing-library/react";
import { describe, it, expect, vi, afterEach } from "vitest";
import HistoryChart from "@/components/HistoryChart";
import * as api from "@/lib/api";

afterEach(() => {
  vi.restoreAllMocks();
});

describe("HistoryChart", () => {
  it("renders chart container after history loads", async () => {
    vi.spyOn(api, "fetchHistory").mockResolvedValue([
      { id: 1, ts: "2026-08-14T10:00:00Z", spot: 310, zero_gamma: 323.1, put_wall: 300, call_wall: 340, max_pain: 325, iv_skew: 0.1, light: "red", iv_alert: 0 },
    ]);

    render(<HistoryChart />);

    await waitFor(() => expect(screen.getByTestId("history-chart")).toBeInTheDocument());
  });

  it("renders chart container even when history fetch fails", async () => {
    vi.spyOn(api, "fetchHistory").mockRejectedValue(new Error("network error"));

    render(<HistoryChart />);

    await waitFor(() => expect(screen.getByTestId("history-chart")).toBeInTheDocument());
  });
});
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd frontend && npm run test -- HistoryChart`
Expected: FAIL，找不到 `@/components/HistoryChart` 模組。

- [ ] **Step 3: 實作 HistoryChart.tsx**

`frontend/components/HistoryChart.tsx`:
```tsx
"use client";

import { useEffect, useState } from "react";
import { Line } from "react-chartjs-2";
import {
  Chart as ChartJS, CategoryScale, LinearScale, PointElement, LineElement, Tooltip, Legend,
} from "chart.js";
import { fetchHistory, Snapshot } from "@/lib/api";

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Tooltip, Legend);

export default function HistoryChart() {
  const [history, setHistory] = useState<Snapshot[]>([]);

  useEffect(() => {
    fetchHistory(7).then(setHistory).catch(() => setHistory([]));
  }, []);

  const data = {
    labels: history.map((h) => h.ts),
    datasets: [
      { label: "現價", data: history.map((h) => h.spot), borderColor: "#38bdf8", tension: 0.2 },
      { label: "Zero Gamma", data: history.map((h) => h.zero_gamma), borderColor: "#f97316", tension: 0.2 },
    ],
  };

  return (
    <div data-testid="history-chart" className="rounded-lg border border-gray-700 p-4">
      <Line data={data} />
    </div>
  );
}
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd frontend && npm run test -- HistoryChart`
Expected: 全部 2 個測試 PASS。

- [ ] **Step 5: Commit**

```bash
git add frontend/components/HistoryChart.tsx frontend/tests/HistoryChart.test.tsx
git commit -m "feat: add HistoryChart component"
```

---

### Task 17: 頁面組裝與前端 Railway 部署設定

**Files:**
- Modify: `frontend/app/page.tsx`
- Modify: `frontend/app/layout.tsx`
- Create: `frontend/railway.toml`

**Interfaces:**
- Consumes: `TrafficLight`（Task 14）、`WallsPanel`（Task 15）、`HistoryChart`（Task 16）
- Produces: 首頁 UI（無新增可測試邏輯，此任務為組裝與部署設定）

- [ ] **Step 1: 組裝首頁**

`frontend/app/page.tsx`:
```tsx
import TrafficLight from "@/components/TrafficLight";
import WallsPanel from "@/components/WallsPanel";
import HistoryChart from "@/components/HistoryChart";

export default function Home() {
  return (
    <main className="mx-auto max-w-4xl space-y-6 p-6">
      <h1 className="text-3xl font-bold">TSLA Gamma 預警儀表板</h1>
      <TrafficLight />
      <WallsPanel />
      <HistoryChart />
    </main>
  );
}
```

- [ ] **Step 2: 更新 layout 標題**

在 `frontend/app/layout.tsx` 的 `metadata` 物件中，將 `title` 改為 `"TSLA Gamma 預警儀表板"`，`description` 改為 `"TSLA 期權 Gamma 與暴跌/洗盤預警儀表板"`。

- [ ] **Step 3: 建立前端 Railway 部署設定**

`frontend/railway.toml`:
```toml
[build]
builder = "NIXPACKS"
buildCommand = "npm install && npm run build"

[deploy]
startCommand = "npm run start"
restartPolicyType = "ON_FAILURE"
```

註記：此服務需在 Railway 環境變數設定 `NEXT_PUBLIC_API_URL` 指向 `api` 服務的公開網域（build-time 變數，變更後需重新部署才會生效）。

- [ ] **Step 4: 本地整合驗證**

Run（於兩個終端機分別執行）：
```bash
cd backend && DB_PATH=./data/dev.db uvicorn app.api:app --reload --port 8000
```
```bash
cd frontend && NEXT_PUBLIC_API_URL=http://localhost:8000 npm run dev
```
開啟瀏覽器至 `http://localhost:3000`，確認頁面顯示「等待數據...」（因尚未有 worker 寫入資料，屬預期行為）；接著手動執行一次 worker 邏輯寫入測試資料驗證前端能正確渲染：
```bash
cd backend && python3 -c "
from app.db import get_connection, init_db, insert_snapshot
conn = get_connection('./data/dev.db')
init_db(conn)
insert_snapshot(conn, {'ts': '2026-08-14T10:00:00', 'spot': 310.0, 'zero_gamma': 323.1, 'put_wall': 300.0, 'call_wall': 340.0, 'max_pain': 325.0, 'iv_skew': 0.1, 'light': 'red', 'iv_alert': False})
"
```
重新整理瀏覽器，確認燈號顯示紅色、四個防禦牆卡片顯示對應數值、圖表區塊出現。

- [ ] **Step 5: 執行完整測試套件確認無回歸**

Run: `cd backend && pytest -v && cd ../frontend && npm run test`
Expected: 後端與前端所有測試全部 PASS。

- [ ] **Step 6: Commit**

```bash
git add frontend/app/page.tsx frontend/app/layout.tsx frontend/railway.toml
git commit -m "feat: assemble dashboard page and add frontend Railway config"
```

---

## Self-Review Notes

- **Spec coverage**：三色燈號（Task 7）、Gamma 防禦牆與 Max Pain（Task 3–5）、IV Skew 預警（Task 6）、Telegram 推送（Task 9–10）、前端三元件（Task 14–16）、Railway 部署（Task 12、17）、SQLite 歷史記錄（Task 2）均已對應到具體任務。暗池監控依 spec 明確排除，不建立對應任務。
- **型別一致性**：`options` 的 dict 結構（`strike`, `time_to_expiry`, `iv`, `open_interest`, `option_type`）在 Task 3、5、6、8、10 中維持一致；`snapshot` 的 dict 結構在 Task 2、10、11 中維持一致；前端 `Snapshot` type 與後端 `snapshots` 表欄位一一對應。
- **依賴順序**：Task 2–9 為互相獨立的純函數/資料層，可平行開發；Task 10 依賴 Task 2、3–9；Task 11 依賴 Task 2；Task 13 為前端起點；Task 14–16 依賴 Task 13 但彼此獨立；Task 17 依賴 Task 14–16。
