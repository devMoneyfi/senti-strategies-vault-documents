# Senti Quant Stage — API Reference (khám phá từ Dashboard)

> Tài liệu này được ghi nhận trực tiếp từ các request/response trên **https://stage.sentitrade.xyz** (đã đăng nhập).  
> Base URL BE: `https://be-dev.sentitrade.xyz`  
> Base URL RTAPI: `https://rtapi.sentitrade.xyz`  
> Giao thức: mọi endpoint đều là **POST** (trừ `/v1/account` là GET) với header `Content-Type: application/json`, body là JSON object.  
> Phiên bản ghi: 2026-06-04.

---

## Mục lục

1. [Lấy danh sách Strategies (EA Catalog)](#1-lấy-danh-sách-strategies-ea-catalog)
2. [Lấy danh sách MT5 Accounts + Active Strategies](#2-lấy-danh-sách-mt5-accounts--active-strategies)
3. [Lấy thông tin chi tiết 1 MT5 Account — Portfolio totals, Equity, DrawDown](#3-lấy-thông-tin-chi-tiết-1-mt5-account--portfolio-totals-equity-drawdown)
4. [Tính các KPI từ deals (Portfolio totals / Win rate / Profit factor / Drawdown)](#4-tính-các-kpi-từ-deals-portfolio-totals--win-rate--profit-factor--drawdown)
5. [Phụ lục — Các endpoint quan sát thêm](#5-phụ-lục--các-endpoint-quan-sát-thêm)

---

## 1. Lấy danh sách Strategies (EA Catalog)

### Endpoint

```
POST https://be-dev.sentitrade.xyz/operations/get-ea-catalog
```

### Request body

```json
{}
```

### Response

Mảng `json` (type `EADefinition[]`). Ví dụ một entry:

```json
{
  "id": "b3e114c9-f99c-4e41-af6b-0391dbafa399",
  "createdAt": "2026-05-05T07:46:52.850Z",
  "updatedAt": "2026-05-22T09:20:37.375Z",
  "name": "EMA MIRROR v1.00",
  "description": "Mean reversion strategy with EMA",
  "filename": "EMA_MIRROR_v1.00.ex5",
  "version": "1.0.0",
  "versionGroup": "ema_mirror",
  "previousEaVersionId": null,
  "supportedSymbols": ["XAUUSD", "BTCUSD"],
  "supportedTimeframes": ["M1", "M5", "M15", "M30", "H1", "H4", "D1", "W1", "MN"],
  "isActive": true,
  "defaultInputs": [ /* see bảng bên dưới */ ],
  "presets": [ /* see bảng bên dưới */ ]
}
```

#### `defaultInputs` — Schema cấu hình của EA

Mỗi phần tử:

| Field | Kiểu | Giải thích |
|---|---|---|
| `name` | string | Tên param (ví dụ `InpMagicNumber`) |
| `type` | string | `number` / `boolean` / `string` |
| `label` | string | Nhãn hiển thị |
| `locked` | boolean | `true` → không cho sửa ở UI |
| `description` | string | Ghi chú |
| `defaultValue` | any | Giá trị mặc định |

#### `presets` — Bộ cấu hình có sẵn

| Field | Kiểu | Giải thích |
|---|---|---|
| `id` | string (uuid) | |
| `name` | string | Tên preset (ví dụ `"XAUUSD - M15 - Aggressive Vantage"`) |
| `symbol` | string | Cặp |
| `timeframe` | string | TF |
| `inputs` | object | Giá trị override cho từng `defaultInputs.name` |

### Khi nào dùng

- Trang **Strategies** (catalog) — hiển thị toàn bộ EA đang public, user chọn để deploy.

---

## 2. Lấy danh sách MT5 Accounts + Active Strategies

Hai endpoint này thường gọi song song (dashboard/strategies đều gọi cả hai).

### 2a. Danh sách MT5 Accounts

```
POST https://be-dev.sentitrade.xyz/operations/get-mt5-accounts
```

#### Request body

```json
{}
```

#### Response

Mảng `json` (type `MT5Account[]`).

```json
[{
  "id": "ac3c7eb1-9f58-487e-9d54-482f10cba759",
  "login": "257424789",
  "label": null,
  "broker": "Exness",
  "server": "Exness-MT5Real36",
  "accountType": "REAL",
  "brokerAccountTypeName": "Standard Cent",
  "isActive": true,
  "accessMode": "MASTER",
  "lastKnownBalance": null,
  "lastKnownEquity": null,
  "lastSyncAt": null,
  "createdAt": "2026-05-30T11:59:04.522Z",
  "terminal": {
    "assignedPort": 18201,
    "terminalStatus": "ACTIVE",
    "nodeName": "Node-WIN-01"
  },
  "activeEas": [
    {
      "name": "GRID DCA v1.02",
      "status": "RUNNING"
    }
  ]
}]
```

| Field | Giải thích |
|---|---|
| `id` | UUID — chính là `mt5AccountId` dùng cho RTAPI và các endpoint khác |
| `login` | Số tài khoản MT5 |
| `broker`, `server` | Thông tin broker |
| `isActive` | Tài khoản đang được theo dõi |
| `accessMode` | `MASTER` / `SLAVE`... |
| `terminal` | Thông tin terminal đang chạy EA |
| `activeEas[]` | Tóm tắt nhanh EA đang deploy (`name`, `status`) — đủ để hiển thị list, không phải config chi tiết |

### 2b. Chi tiết Active EAs đang deploy

```
POST https://be-dev.sentitrade.xyz/operations/get-active-eas
```

#### Request body

```json
{}
```

#### Response

Mảng `json` (type `ActiveEA[]`). Entry đầy đủ:

```json
{
  "id": "989f99ec-06ea-4b81-910a-f3c91c3337c9",
  "createdAt": "2026-05-30T11:59:32.529Z",
  "updatedAt": "2026-05-30T11:59:33.674Z",
  "mt5AccountId": "ac3c7eb1-9f58-487e-9d54-482f10cba759",
  "eaDefinitionId": "40acbb02-e6c0-48dc-960b-0a630a9cca42",
  "userId": "7f57f489-2c8a-4cc6-a366-7c5ec0e52d47",
  "chartId": "134246159608930594",
  "symbol": "XAUUSD",
  "timeframe": "M1",
  "tplFilename": "GRID_DCA_v1_02_XAUUSDc_M1.tpl",
  "inputsUsed": {
    "EODTo": 0,
    "BaseLot": 0.01,
    "BuyOnly": false,
    "ChainSL": 50000,
    "EODFrom": 19,
    "SellOnly": false,
    "TPPerLot": 500,
    "MagicNumber": 35,
    "CommentPrefix": "GRID_DCA_v1.02",
    "EODForceClose": 7,
    "RestartAfterSL": 0,
    "Zone1_GridStep": 5000,
    "Zone2_GridStep": 5000,
    "Zone3_GridStep": 10000,
    "Zone4_GridStep": 10000,
    "OpenNewChainAfterTP": 5,
    "Zone1_To_Zone2_Distance": 20000,
    "Zone2_To_Zone3_Distance": 30000,
    "Zone3_To_Zone4_Distance": 30000
  },
  "status": "RUNNING",
  "deployedAt": "2026-05-30T11:59:32.529Z",
  "stoppedAt": null,
  "eaDefinition": { /* giống EADefinition từ get-ea-catalog */ },
  "mt5Account": { "id": "...", "login": "257424789", "label": null }
}
```

| Field | Giải thích |
|---|---|
| `id` | UUID của deployment |
| `mt5AccountId` | FK → `MT5Account.id` |
| `eaDefinitionId` | FK → `EADefinition.id` |
| `symbol`, `timeframe` | Cặp & TF đang chạy |
| `tplFilename` | Template ex5 đã upload |
| `inputsUsed` | **Cấu hình thực tế đang dùng** (MagicNumber, BaseLot, TPPerLot, GridStep...) |
| `status` | `RUNNING` / `STOPPED` / ... |
| `deployedAt`, `stoppedAt` | Thời điểm |
| `eaDefinition` | Toàn bộ schema của EA (dùng để build form cấu hình UI) |
| `mt5Account` | Thông tin login/broker tương ứng (để hiển thị) |

### Gợi ý join

Để ra "danh sách account + strategy đang chạy":

```
foreach account in get-mt5-accounts:
    account.activeEas ← join với get-active-eas
        bởi activeEA.mt5AccountId == account.id
        lấy thêm: symbol, timeframe, inputsUsed, status, eaDefinition.name
```

---

## 3. Lấy thông tin chi tiết 1 MT5 Account — Portfolio totals, Equity, DrawDown

### 3a. Trạng thái tài khoản real-time (Portfolio totals / Equity)

```
GET https://rtapi.sentitrade.xyz/tm/{mt5AccountId}/v1/account
```

Ví dụ thực tế:

```
GET https://rtapi.sentitrade.xyz/tm/ac3c7eb1-9f58-487e-9d54-482f10cba759/v1/account
```

#### Response

```json
{
  "MSG": "ACCOUNT_STATUS",
  "COMPANY": "Exness Technologies Ltd",
  "CURRENCY": "USC",
  "NAME": "MoneyFi",
  "SERVER": "Exness-MT5Real36",
  "LOGIN": 257424789,
  "TRADE_MODE": 2,
  "LEVERAGE": 2000,
  "LIMIT_ORDERS": 200,
  "TRADE_ALLOWED": true,
  "MARGIN_MODE": 2,
  "BALANCE": 123631.10,
  "CREDIT": 0.00,
  "PROFIT": -83.70,
  "EQUITY": 123547.40,
  "MARGIN": 11.15,
  "MARGIN_FREE": 123536.25,
  "MARGIN_LEVEL": 1108048.43,
  "ERROR_ID": 0,
  "ERROR_DESCRIPTION": "The operation completed successfully"
}
```

| Field | Giải thích | Dùng cho |
|---|---|---|
| `BALANCE` | Số dư gốc | Portfolio totals |
| `PROFIT` | Floating P&L (unrealized) | Day P&L / Unrealized P&L |
| `EQUITY` | `BALANCE + PROFIT` | Equity hiện tại |
| `MARGIN`, `MARGIN_FREE`, `MARGIN_LEVEL` | Margin đang dùng / còn lại / % | Margin monitor |
| `LEVERAGE` | Đòn bẩy | |

> Đây là nguồn **duy nhất** có BALANCE/EQUITY/PROFIT chính xác thời gian thực.  
> Dashboard polling mỗi 30s (thấy trong UI: *"Portfolio + KPIs aggregated from 1 account(s) (polling every 30s)"*).

### 3b. Để tính DrawDown / Equity curve — cần lịch sử deals

#### Trạng thái đồng bộ

```
POST https://be-dev.sentitrade.xyz/operations/get-sync-state-for-account
```

```json
{ "mt5AccountId": "ac3c7eb1-9f58-487e-9d54-482f10cba759" }
```

Response:

```json
{
  "lastSyncedDealTime": "2026-06-04T03:47:34.000Z",
  "lastSyncRunAt": "2026-06-04T04:10:26.587Z",
  "syncStatus": "HEALTHY",
  "dealsCount": 688,
  "isBackfilling": false
}
```

| Field | Giải thích |
|---|---|
| `lastSyncedDealTime` | Thời điểm deal mới nhất đã đồng bộ |
| `lastSyncRunAt` | Lần sync cuối chạy |
| `syncStatus` | `HEALTHY` / `DEGRADED` / ... |
| `dealsCount` | Tổng số deal đã có trong warehouse |
| `isBackfilling` | Đang backfill lịch sử cũ hay không |

#### Lấy deals từ warehouse (đầy đủ lịch sử)

```
POST https://be-dev.sentitrade.xyz/operations/get-deals-from-warehouse
```

```json
{
  "mt5AccountId": "ac3c7eb1-9f58-487e-9d54-482f10cba759",
  "fromTime": "2026-06-03T00:00:00.000Z",
  "toTime": "2026-06-04T03:47:34.000Z"
}
```

| Param | Bắt buộc | Giải thích |
|---|---|---|
| `mt5AccountId` | ✓ | UUID account |
| `fromTime` | tùy | ISO8601 UTC — nếu bỏ trả theo mặc định |
| `toTime` | tùy | ISO8601 UTC |

Response body format: `{ "json": [ <DEAL_OBJ>, ... ], "meta": { "values": {...}, "v": 1 } }`

Mỗi `DEAL_OBJ`:

| Field | Giải thích |
|---|---|
| `TICKET` | ID deal |
| `ORDER` | ID order gốc |
| `POSITION_ID` | ID position |
| `SYMBOL` | Cặp, ví dụ `XAUUSDc` |
| `TYPE` | `DEAL_TYPE_BUY` / `DEAL_TYPE_SELL` |
| `ENTRY` | `DEAL_ENTRY_IN` (mở) / `DEAL_ENTRY_OUT` (đóng) |
| `VOLUME` | Lot |
| `PRICE` | Giá thực hiện |
| `COMMISSION`, `SWAP`, `PROFIT`, `FEE` | Số tiền |
| `TIME` | Thời gian (string `yyyy.MM.dd HH:mm:ss`) |
| `MAGIC` | Magic number của EA |
| `COMMENT` | Comment (ví dụ `"GRID_DCA_v1.02_M1_B_00"` — chứa hướng & level) |

> `ENTRY == DEAL_ENTRY_OUT` có `PROFIT` chính là realized P&L của trade đó.  
> Gom theo thời gian → reconstruct equity curve → **Max Drawdown**.

#### Lấy deals gần nhất từ Kong (fallback/realtime)

```
POST https://be-dev.sentitrade.xyz/operations/get-recent-deals-from-kong
```

```json
{ "mt5AccountId": "ac3c7eb1-9f58-487e-9d54-482f10cba759" }
```

Response tương tự `get-deals-from-warehouse` (trả mảng `json[]`).  
Dùng khi cần deals mới nhất mà warehouse chưa backfill kịp.

### Công thức chuẩn cho mọi time range (30D / 90D / All-Time)

```python
# ─── INPUTS cần thiết ───
# 1. DEAL_TYPE_BALANCE.PROFIT = initial deposit (120,872 USC)
# 2. Danh sách deals từ get-deals-from-warehouse (hoặc file đã lưu)
# 3. (Optional) /v1/account → BALANCE hiện tại để verify

# ─── BƯỚC 1: Xác định starting balance ───
# Nếu DEAL_TYPE_BALANCE có trong range → lấy trực tiếp
initial = 120_872  # USC

# Nếu range bắt đầu sau initial deposit (ví dụ TODAY):
#   initial = DEAL_TYPE_BALANCE.PROFIT + sum(P&L của tất cả deals TRƯỚC start_date)
#   HOẶC: initial = current_balance - sum(P&L của deals TRONG range)
#   (verified: Jun 4 midnight = 123,501 USC)

# ─── BƯỚC 2: Build equity curve từ deals ───
balance = initial
peak = initial
max_dd_pct = 0

# Track open positions để mark-to-market
open_positions = {}

for deal in deals_sorted_by_TIME:
    if deal.TYPE == 'DEAL_TYPE_BALANCE':
        continue

    pid = deal.POSITION_ID

    if deal.ENTRY == 'DEAL_ENTRY_IN':
        # Mở vị thế mới
        open_positions[pid] = {
            'type': deal.TYPE,        # DEAL_TYPE_BUY=Long, DEAL_TYPE_SELL=Short
            'entry_price': deal.PRICE,
            'vol': deal.VOLUME,
            'time': deal.TIME,
        }
        balance += deal.COMMISSION + deal.SWAP

    elif deal.ENTRY == 'DEAL_ENTRY_OUT':
        # Đóng vị thế — realize P&L
        balance += deal.PROFIT + deal.COMMISSION + deal.SWAP

        # ─── Mark-to-market tại deal PRICE cho các vị thế còn mở ───
        mark_price = deal.PRICE
        floating = 0
        for opid, pos in open_positions.items():
            if opid != pid:  # bỏ qua position vừa đóng
                if pos['type'] == 'DEAL_TYPE_BUY':   # Long
                    floating += (mark_price - pos['entry_price']) * pos['vol'] * 100
                else:                                   # Short
                    floating += (pos['entry_price'] - mark_price) * pos['vol'] * 100

        equity = balance + floating
        peak = max(peak, equity)
        dd_pct = (peak - equity) / peak * 100

        if dd_pct > max_dd_pct:
            max_dd_pct = dd_pct
            max_dd_time = deal.TIME
            max_dd_peak = peak
            max_dd_trough = equity
            max_dd_floating = floating

        open_positions.pop(pid, None)

# ─── KẾT QUẢ ───
# 30D (verified 2026-06-04):
#   Method A (balance-only):       0.27%  ← thiếu floating
#   Method B (+ mark-to-market):   0.88%  ← gần UI, thiếu 0.19%
#   UI (M1 interpolation):        -1.07%  ← exact
#
# Gap 0.19% do: giá di chuyển giữa các deal → M1 candles fill được
# Method B đủ tốt nếu không cần precision cao.
# Để khớp 100% cần M1 candles + interpolation (xem section bên dưới).
```

### Starting Balance cho từng time range

| Range | Starting Balance | Cách tính |
|---|---|---|
| **All-Time** | 120,872 USC ($1,208.72) | `DEAL_TYPE_BALANCE.PROFIT` trực tiếp |
| **30D** | 120,872 USC ($1,208.72) | Cũng là `DEAL_TYPE_BALANCE` (account mở May 30) |
| **TODAY** | 123,501 USC ($1,235.01) | `current_balance - sum(P&L trong range)` |
| **7D** | ~122,392 USC | Sum P&L May 30 → Jun 3 midnight |

> **Công thức tổng quát**:
> ```
> initial_range = DEAL_TYPE_BALANCE.PROFIT
>   + sum(ENTRY_OUT.PROFIT + COMMISSION + SWAP của tất cả deals TRƯỚC start_date)
>
> HOẶC (if có current_balance):
>   initial_range = current_balance - sum(P&L của deals TRONG range)
> ```

### Verification results (30D)

| Method | Max DD | Cần price API | Ghi chú |
|---|---|---|---|
| Balance-only | 0.27% | ❌ | Thiếu floating P&L → DD thấp hơn thật |
| **+ Mark-to-market (deal PRICE)** | **0.88%** | ❌ | **Gần UI, đủ dùng production** |
| UI (M1 interpolation) | -1.07% | ✅ | Exact — dùng 559 M1 candles/day |

### Khi nào cần price API (`/v1/history/prices`)

| Use case | Cần M1? | Thay thế |
|---|---|---|
| Tính DD lịch sử (30D/All-Time) | ❌ Không | Dùng deal PRICE làm mark |
| Tính DD chính xác như UI | ✅ Cần | M1 interpolation |
| Equity curve mượt (chart) | ✅ Cần | M1 candles |
| Floating P&L hiện tại | ✅ Cần | `/v1/account` PROFIT field |
| Open positions detail | ❌ Không | `/v1/order/list` |

> `/v1/order/list` chỉ trả **OPENED** positions (không có closed history). Không dùng được để tính DD lịch sử.

---

## 4. Tính các KPI từ deals (Portfolio totals / Win rate / Profit factor / Drawdown)

Tất cả các KPI có thể tự tính từ `get-deals-from-warehouse` (hoặc `get-recent-deals-from-kong`) + `BALANCE` ban đầu từ `/v1/account`.

> **Quy ước quan trọng**: mỗi trade có **2 deals** — `ENTRY_IN` (mở) và `ENTRY_OUT` (đóng).  
> KPI chỉ tính trên `ENTRY == DEAL_ENTRY_OUT` (đã realized).  
> Nếu lấy cả `ENTRY_IN` sẽ double-count volume.

### 4a. Mapping từ block UI → API

| UI Block | Nguồn dữ liệu | Tính tay hay có sẵn |
|---|---|---|
| **Portfolio totals** (Balance, Equity, P&L) | `GET /v1/account` | Có sẵn |
| **ROI** | `BALANCE` ban đầu + `EQUITY` hiện tại | `(EQUITY - BALANCE) / BALANCE` |
| **Win rate** | deals `ENTRY_OUT`, nhóm `PROFIT > 0` / `PROFIT < 0` | Đếm count |
| **Profit factor** | `SUM(PROFIT > 0)` / `ABS(SUM(PROFIT < 0))` | Tổng |
| **Gross profit / Gross loss** | Tổng `PROFIT > 0` / `PROFIT < 0` | Tổng |
| **Avg win / Avg loss** | Tổng/count theo dấu | Tổng + count |
| **Expectancy** | `(winRate * avgWin) - ((1 - winRate) * avgLoss)` | Công thức |
| **Total volume** | `SUM(VOLUME)` (chỉ `ENTRY_OUT`) | Tổng |
| **Notional volume** | `SUM(VOLUME * PRICE)` (chỉ `ENTRY_OUT`) | Tổng |
| **Long / Short** | `TYPE == DEAL_TYPE_BUY` → Long, `SELL` → Short (chỉ `ENTRY_OUT`) | Đếm |
| **Robot / Manual** | `MAGIC != 0` → Robot, `MAGIC == 0` → Manual | Đếm |
| **Closed deals** | Đếm số dòng `ENTRY_OUT` | Đếm |
| **Equity curve / Max DD** | Reconstruct từ `BALANCE` ban đầu + cumulative sum `PROFIT` theo `TIME` | Tự tính |

### 4b. Kết quả verified (khớp 100% với UI Performance page)

> **Ngày verify**: 2026-06-04 12:17 UTC+7  
> **Account**: 257424789 (Exness Standard Cent)  
> **EA**: GRID DCA v1.02, MagicNumber=35, XAUUSD, M1  
> **Contract Size**: `XAUUSDc trên Exness Cent = 100`  
> **Data currency**: USC (US Cents) — chia 100 để ra USD  
> **Initial deposit**: 120,872 USC = $1,208.72 (DEAL_TYPE_BALANCE, 2026-05-30 11:07:17)

#### KPI Table (30-day cumulative)

| # | KPI | Công thức | Kết quả tính | UI Value | Khớp |
|---|---|---|---|---|---|
| 1 | **Closed Deals** | `count(ENTRY_OUT)` | 345 | 345 | ✅ |
| 2 | **Win / Loss** | `count(ENTRY_OUT where PROFIT>0)` / `count(ENTRY_OUT where PROFIT≤0)` | 248 / 95 | 248 / 95 | ✅ |
| 3 | **Win Rate** | `wins / total` | 71.9% | 71.9% | ✅ |
| 4 | **Gross Profit** | `sum(ENTRY_OUT.PROFIT where PROFIT>0)` | 45.37 USD | 45.37 | ✅ |
| 5 | **Gross Loss** | `sum(ENTRY_OUT.PROFIT where PROFIT≤0)` | -17.27 USD | -17.27 | ✅ |
| 6 | **Profit Factor** | `gross_profit / |gross_loss|` | 2.63 | 2.63 | ✅ |
| 7 | **Avg Win** | `gross_profit / wins` | 0.18 USD | 0.18 | ✅ |
| 8 | **Avg Loss** | `gross_loss / losses` | -0.18 USD | -0.18 | ✅ |
| 9 | **Expectancy** | `avg_win × wr + avg_loss × (1-wr)` | 0.08 USD | 0.08 | ✅ |
| 10 | **Long / Short** | `ENTRY_IN=BUY` / `ENTRY_IN=SELL` (trong closed positions) | 144 / 201 | 144 / 201 | ✅ |
| 11 | **Robot / Manual** | `MAGIC≠0` / `MAGIC=0` | 345 / 0 | 345 / 0 | ✅ |
| 12 | **Total P&L** | `balance - initial_deposit` | 28.10 USD | 28.10 | ✅ |
| 13 | **ROI** | `pnl / initial × 100` | 2.3% | 2.3% | ✅ |
| 14 | **Total Balance** | `initial + sum(ENTRY_OUT.PROFIT)` | 1,236.82 USD | 1,236.82 | ✅ |

#### TODAY KPI Verification (2026-06-04)

> **Verify time**: ~09:18 UTC+7 — 65 deals, 559 M1 candles

| # | KPI | Computed | Ghi chú |
|---|---|---|---|
| 1 | **Closed Deals** | 27 | `ENTRY_OUT` có paired `ENTRY_IN` |
| 2 | **Open Positions** | 11 | Chưa có `ENTRY_OUT` |
| 3 | **Win / Loss** | 22 / 4 | Win rate: 84.6% |
| 4 | **Total P&L** | +181.5 USC ($1.81) | realized only |
| 5 | **Gross Profit** | 192.1 USC | 22 wins |
| 6 | **Gross Loss** | -10.6 USC | 4 losses |
| 7 | **Profit Factor** | 18.12 | rất cao do ít losses |
| 8 | **Avg Win** | 8.7 USC | |
| 9 | **Avg Loss** | -2.6 USC | |
| 10 | **Expectancy** | 7.0 USC/trade | |
| 11 | **Max DD** | -0.25% | UI: -0.24% (chênh 0.01% rounding) |
| 12 | **Starting Balance** | 123,500.90 USC | derived: current_balance - today_pnl |
| 13 | **Peak Equity** | 123,509.94 USC | at 02:11 |
| 14 | **Trough Equity** | 123,198.07 USC | at 04:43 (giờ giá đạt 4483.676 — cao nhất ngày) |
| 15 | **Ending Equity** | 123,424.39 USC | balance 123,500.90 + float -76.51 |

#### Position Classification Rules

```
Long  = ENTRY_IN type is BUY  (mở BUY → đóng SELL)
Short = ENTRY_IN type is SELL (mở SELL → đóng BUY)

Robot   = MAGIC ≠ 0 (EA gán MagicNumber khi mở lệnh)
Manual  = MAGIC == 0
```

> **Lưu ý quan trọng**: `DEAL_TYPE_BALANCE` (initial deposit) có `ENTRY = "DEAL_ENTRY_IN"` —
> khi filter `ENTRY == DEAL_ENTRY_OUT` sẽ tự loại deposit, không bị double-count.
> Tuy nhiên khi tính balance từ deals, phải **skip** `DEAL_TYPE_BALANCE` vì đã dùng làm `initial`.

#### Balance Calculation Algorithm

```python
initial = deals.find(DEAL_TYPE_BALANCE).PROFIT  # 120,872 USC
balance = initial

for deal in deals sorted by TIME:
    if deal.TYPE == 'DEAL_TYPE_BALANCE':
        continue  # đã đếm ở initial

    if deal.ENTRY == 'DEAL_ENTRY_IN':
        open_positions[deal.POSITION_ID] = deal
        balance += deal.COMMISSION + deal.SWAP

    elif deal.ENTRY == 'DEAL_ENTRY_OUT':
        if deal.POSITION_ID in open_positions:
            balance += deal.PROFIT + deal.COMMISSION + deal.SWAP
            del open_positions[deal.POSITION_ID]
```

> Kết quả: `balance = 123,682 USC = $1,236.82` — khớp UI.

#### Equity Curve & Max Drawdown

```python
# ─── Thuật toán đã verified với UI ───
# Input: initial_balance, M1 candles, today_deals
# Output: equity curve + Max DD

balance = initial_balance
peak_equity = initial_balance
max_dd_pct = 0
max_dd_time = ""
max_dd_peak = initial_balance
max_dd_trough = initial_balance

# Group deals by TIME
deals_by_time = {}
for deal in today_deals:
    if deal.ENTRY == 'DEAL_ENTRY_OUT':
        deals_by_time.setdefault(deal.TIME, []).append(deal)

# Build open positions lookup
open_positions = {}
for deal in today_deals:
    if deal.TYPE == 'DEAL_TYPE_BALANCE':
        continue
    pid = deal.POSITION_ID
    if deal.ENTRY == 'DEAL_ENTRY_IN':
        open_positions[pid] = {
            'type': deal.TYPE, 'price': deal.PRICE, 'vol': deal.VOLUME, 'time': deal.TIME
        }
    else:
        open_positions.pop(pid, None)

# Walk M1 candles
for candle in m1_candles:
    # Apply realized P&L at this minute
    for deal in deals_by_time.get(candle.TIME, []):
        balance += deal.PROFIT

    # Floating P&L for open positions
    floating = 0
    for pid, pos in open_positions.items():
        if pos['time'] <= candle.TIME:
            if pos['type'] == 'DEAL_TYPE_BUY':   # Long
                floating += (candle.CLOSE - pos['price']) * pos['vol'] * 100
            else:                                  # Short
                floating += (pos['price'] - candle.CLOSE) * pos['vol'] * 100

    equity = balance + floating

    # Max DD tracking
    if equity > peak_equity:
        peak_equity = equity
    dd_pct = (peak_equity - equity) / peak_equity * 100
    if dd_pct > max_dd_pct:
        max_dd_pct = dd_pct
        max_dd_time = candle.TIME
        max_dd_peak = peak_equity
        max_dd_trough = equity

# ─── Verified TODAY ───
# Max DD: 0.25% at 04:43  | UI: -0.24% (chênh 0.01% = rounding)
# Peak: 123,509.94 USC at 02:11
# Trough: 123,198.07 USC at 04:43
# Starting balance: 123,500.90 USC
```

> **Lưu ý**: UI Max DD (-0.24%) dùng M1 price interpolation.
> Chart data được **tính client-side** từ 3 nguồn:
> 1. `/v1/history/prices` — M1 candles cho price interpolation
> 2. `/operations/get-deals-from-warehouse` — deal timestamps
> 3. `/v1/account` — current balance → derive starting balance
>
> Không có endpoint chart API riêng. Dashboard note cũ *"Sharpe / Max DD require OHLCV equity reconstruction"* đã được implement qua M1 interpolation.

### 4c. Ví dụ pipeline (TypeScript)

```typescript
// 1. Lấy deals + initial deposit
const dealsRes = await fetch('https://be-dev.sentitrade.xyz/operations/get-deals-from-warehouse', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ mt5AccountId, fromTime: '2026-05-30T00:00:00.000Z' }),
}).then(r => r.json());

const allDeals = dealsRes.json.sort((a, b) => a.TIME.localeCompare(b.TIME));
const deposit = allDeals.find(d => d.TYPE === 'DEAL_TYPE_BALANCE');
const initial = deposit.PROFIT; // 120,872 USC

// 2. Phân loại deals
const closedDeals = allDeals.filter(d => d.ENTRY === 'DEAL_ENTRY_OUT');
const wins = closedDeals.filter(d => d.PROFIT > 0);
const losses = closedDeals.filter(d => d.PROFIT <= 0);

// 3. KPIs
const winRate = wins.length / closedDeals.length;
const grossProfit = wins.reduce((s, d) => s + d.PROFIT, 0) / 100; // USD
const grossLoss = losses.reduce((s, d) => s + d.PROFIT, 0) / 100;
const profitFactor = Math.abs(grossProfit / grossLoss);
const avgWin = grossProfit / wins.length;
const avgLoss = grossLoss / losses.length;
const expectancy = (winRate * avgWin) + ((1 - winRate) * avgLoss);

// 4. Balance
const balance = initial + closedDeals.reduce((s, d) => s + d.PROFIT + d.COMMISSION + d.SWAP, 0);
const pnl = balance - initial;
const roi = pnl / initial;

// 5. Long/Short (dùng ENTRY_IN type, không phải ENTRY_OUT type)
const positions = new Map();
for (const d of allDeals.filter(d => d.TYPE !== 'DEAL_TYPE_BALANCE')) {
  if (!positions.has(d.POSITION_ID)) positions.set(d.POSITION_ID, []);
  positions.get(d.POSITION_ID).push(d);
}
let longCount = 0, shortCount = 0;
for (const [, ds] of positions) {
  const entryIn = ds.find(d => d.ENTRY === 'DEAL_ENTRY_IN');
  const entryOut = ds.find(d => d.ENTRY === 'DEAL_ENTRY_OUT');
  if (entryIn && entryOut) {
    if (entryIn.TYPE === 'DEAL_TYPE_BUY') longCount++;
    else shortCount++;
  }
}
```

### 4d. Lưu ý khi triển khai BE

### Lưu ý: Starting Balance

Có 2 cách lấy starting balance:

| Method | Giá trị | Dùng khi |
|---|---|---|
| `DEAL_TYPE_BALANCE.PROFIT` trong deals | 120,872 USC | Có đủ lịch sử deals (30 ngày đầu) |
| `current_balance - today_realized_pnl` | 123,500.90 USC | Chỉ có deals hôm nay (derive từ `/v1/account`) |

> **Quan trọng**: `/v1/account` trả `BALANCE` hiện tại (đã bao gồm tất cả realized P&L).
> Để về starting balance: `initial = current_balance - sum(all_realized_pnl)`.
> Điều này giải thích tại sao `initial` 123,500.90 ≠ `DEAL_TYPE_BALANCE` 120,872 — chênh 2,629 USC = P&L 30 ngày đầu chưa có trong deals lấy được.
- **Deal mất mát vì close-by / hedge**: một position có thể có nhiều `ENTRY_OUT` (partial close). Công thức trên vẫn đúng vì mỗi `ENTRY_OUT` là một realized trade riêng.
- **Time format**: `TIME` trả về dạng `"2026.06.04 00:00:05"` (không phải ISO). Sort theo string vẫn đúng vì format `yyyy.MM.dd HH:mm:ss` là lexicographic.
- **`DEAL_TYPE_BALANCE` có `ENTRY = DEAL_ENTRY_IN`**: Khi filter `ENTRY == DEAL_ENTRY_OUT` sẽ tự loại deposit. Tuy nhiên khi tính balance, phải skip `DEAL_TYPE_BALANCE` để tránh double-count.
- **PROFIT field**: `ENTRY_OUT.PROFIT` là realized P&L (broker tính sẵn, bao gồm spread). Đơn vị USC (US Cents) — chia 100 ra USD.
- **COMMISSION & SWAP**: Có ở cả ENTRY_IN và ENTRY_OUT. Khi tính balance cần cộng cả hai.
- **`VOLUME` đơn vị**: Exness-Cent dùng `0.01` lot = 1 microlot. Don vị trong API là lots (vd: 0.01, 0.05, 0.13).
- **Notional volume**: `SUM(VOLUME * PRICE)` trên `ENTRY_OUT` — đơn vị USC.
- **Robot/Manual**: `MAGIC === 0` → manual, `MAGIC !== 0` → robot. EA `GRID DCA v1.02` có `MagicNumber: 35`.
- **Long/Short**: Phân loại bằng `ENTRY_IN.TYPE` (không phải `ENTRY_OUT.TYPE`). `ENTRY_IN=BUY → Long`, `ENTRY_IN=SELL → Short`.
- **Backfill**: Khi `isBackfilling: true` → nên chờ sync xong trước khi tính KPI.
- **Open positions (chưa có ENTRY_OUT)**: Không计入 Closed Deals, không计入 Long/Short. Vẫn计入 floating P&L khi tính equity.

### 4e. Checklist "đủ data chưa" trước khi tính

```typescript
const sync = await fetch('https://be-dev.sentitrade.xyz/operations/get-sync-state-for-account', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ mt5AccountId }),
}).then(r => r.json());

if (sync.syncStatus !== 'HEALTHY') {
  // cảnh báo: data chưa đáng tin cậy
}
if (sync.isBackfilling) {
  // chờ backfill xong
}
// Kiểm tra dealsCount đủ lớn, lastSyncedDealTime đủ gần hiện tại
```

### 4f. Các API endpoint mới phát hiện (2026-06-04)

| Endpoint | Method | Body | Response | Ghi chú |
|---|---|---|---|---|
| `trigger-sync-for-account` | POST | `{ mt5AccountId }` | `{ status: "queued" }` | Trigger sync thủ công — dùng khi cần force sync ngay |
| `get-fx-rates` | POST | `{}` | Tỷ giá FX | Dùng để convert USC → USD display |
| `get-brokers` | POST | `{}` | Danh sách brokers | |
| `get-article-by-slug` | POST | `{ slug }` | Chi tiết bài viết | |
| `get-ea-reviews` | POST | `{ eaDefinitionId }` | Reviews của EA | |
| `get-my-ea-review` | POST | `{ eaDefinitionId }` | Review của user hiện tại | |
| `/v1/history/prices` | GET | `?symbol=XAUUSDc&timeframe=PERIOD_M1&from_date=YYYY.MM.DD` | M1/M5/M15/H1 candles | `from_date` hỗ trợ, `to_date` trả 0 (chưa hoạt động). Limit 1000 candles/request |
| `/v1/symbol/info` | GET | `?symbol=XAUUSDc` | Thông tin symbol (contract size, digits...) | |
| `/v1/order/list` | GET | — | Danh sách orders đang chờ | |

---

## 5. Phụ lục — Các endpoint quan sát thêm

| Endpoint | Method | Body | Response | Ghi chú |
|---|---|---|---|---|
| `get-published-articles` | POST | `{}` | Danh sách bài viết | Dashboard + Strategies đều gọi |
| `get-article-by-slug` | POST | `{ slug }` | Chi tiết bài viết theo slug | |
| `get-disclaimer-status` | POST | `{}` | Trạng thái disclaimer | |
| `get-ea-reviews` | POST | `{ eaDefinitionId }` | Reviews của EA | |
| `get-my-ea-review` | POST | `{ eaDefinitionId }` | Review của user | |
| `get-brokers` | POST | `{}` | Danh sách brokers hỗ trợ | |
| `trigger-sync-for-account` | POST | `{ mt5AccountId }` | `{ status: "queued" }` | Force sync thủ công |
| `get-fx-rates` | POST | `{}` | Tỷ giá FX | Convert USC → USD |
| `/v1/symbol/info` | GET | `?symbol=XAUUSDc` | Contract size, digits, spread... | |
| `/v1/order/list` | GET | — | Orders đang chờ | |
| `/v1/history/prices` | GET | `?symbol=XAUUSDc&timeframe=PERIOD_M1&from_date=YYYY.MM.DD` | M1 OHLCV candles | Limit 1000 candles, `to_date` chưa hoạt động |
| `auth/me` | GET | — | User profile | Auth |
| `auth/email/login` | POST | `{email, password}` | token | Login |

### Lưu ý chung

- BE `be-dev.sentitrade.xyz` chỉ chấp nhận **POST** với body JSON rỗng `{}` cho các endpoint `operations/*`.  
- RTAPI `rtapi.sentitrade.xyz` dùng **path param** `{mt5AccountId}` và **GET**.  
- Các `id` quan trọng trong hệ thống:
  - `MT5Account.id` == `mt5AccountId` — dùng xuyên suốt RTAPI + operations.
  - `EADefinition.id` == `eaDefinitionId` — chiết xuất từ `get-ea-catalog` trước khi deploy.
  - `ActiveEA.id` — deployment instance (1 account + 1 EA definition = 1 row).
- Dashboard polling interval ~30s (thấy từ UI text). Frontend không cache đáng kể — mỗi lần navigate gọi lại từ đầu.
