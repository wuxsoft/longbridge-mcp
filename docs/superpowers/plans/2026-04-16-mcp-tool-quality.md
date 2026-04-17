# MCP Tool Quality: Defaults, Missing Params, Descriptions

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Align MCP tool definitions with CLI quality — add default values, missing parameters, and rich descriptions so LLMs can use tools correctly without guessing.

**Architecture:** All changes are in `src/tools/*.rs` (param structs + function bodies) and `src/tools/mod.rs` (tool descriptions + wiring). Fields that have sensible defaults become `Option<T>` with `unwrap_or()` in the function body. The doc comment on each field communicates the default to the LLM via the JSON schema. No new files, no new dependencies.

**Tech Stack:** Rust, rmcp (schemars for JSON schema generation, serde for deserialization)

**Reference:** CLI definitions in `../longbridge-terminal/src/cli/mod.rs` are the source of truth for descriptions, defaults, and parameter completeness.

---

## Phase 1: Default Values (make required fields optional with sensible defaults)

### Task 1: quote.rs — Add defaults to param structs

**Files:**
- Modify: `src/tools/quote.rs:25-152`

The pattern for each field: change to `Option<T>`, update doc comment to include `(default: X)`, use `unwrap_or()` in the function body.

- [ ] **Step 1: Update `SymbolCountParam`**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct SymbolCountParam {
    /// Security symbol in CODE.MARKET format, e.g. "TSLA.US", "700.HK"
    pub symbol: String,
    /// Number of results to return (default: 20, max: 1000)
    pub count: Option<usize>,
}
```

- [ ] **Step 2: Update `CandlesticksParam`**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct CandlesticksParam {
    /// Security symbol in CODE.MARKET format, e.g. "TSLA.US", "700.HK"
    pub symbol: String,
    /// Candlestick period (default: "day"): 1m, 5m, 15m, 30m, 60m, day, week, month, year
    pub period: Option<String>,
    /// Number of candlesticks to return (default: 100, max: 1000)
    pub count: Option<usize>,
    /// Forward-adjust prices for splits/dividends (default: false)
    pub forward_adjust: Option<bool>,
    /// Trade session filter (default: "intraday"): "intraday" or "all" (includes pre/post market)
    pub trade_sessions: Option<String>,
}
```

- [ ] **Step 3: Update `HistoryCandlesticksByOffsetParam`**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct HistoryCandlesticksByOffsetParam {
    /// Security symbol in CODE.MARKET format, e.g. "TSLA.US", "700.HK"
    pub symbol: String,
    /// Candlestick period (default: "day"): 1m, 5m, 15m, 30m, 60m, day, week, month, year
    pub period: Option<String>,
    /// Forward-adjust prices for splits/dividends (default: false)
    pub forward_adjust: Option<bool>,
    /// Query forward in time from reference (default: false = backward)
    pub forward: Option<bool>,
    /// Reference datetime (yyyy-mm-ddTHH:MM:SS), omit to start from latest
    pub time: Option<String>,
    /// Number of candlesticks to return (default: 100, max: 1000)
    pub count: Option<usize>,
    /// Trade session filter (default: "intraday"): "intraday" or "all"
    pub trade_sessions: Option<String>,
}
```

- [ ] **Step 4: Update `HistoryCandlesticksByDateParam`**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct HistoryCandlesticksByDateParam {
    /// Security symbol in CODE.MARKET format, e.g. "TSLA.US", "700.HK"
    pub symbol: String,
    /// Candlestick period (default: "day"): 1m, 5m, 15m, 30m, 60m, day, week, month, year
    pub period: Option<String>,
    /// Forward-adjust prices for splits/dividends (default: false)
    pub forward_adjust: Option<bool>,
    /// Start date (yyyy-mm-dd), optional
    pub start: Option<String>,
    /// End date (yyyy-mm-dd), optional
    pub end: Option<String>,
    /// Trade session filter (default: "intraday"): "intraday" or "all"
    pub trade_sessions: Option<String>,
}
```

- [ ] **Step 5: Update `CalcIndexesParam`**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct CalcIndexesParam {
    /// Security symbols, e.g. ["700.HK", "AAPL.US"]
    pub symbols: Vec<String>,
    /// Calc indexes (default: ["PeTtmRatio","PbRatio","DividendRatioTtm","TurnoverRate","TotalMarketValue"]). Full list: LastDone, ChangeValue, ChangeRate, Volume, Turnover, YtdChangeRate, TurnoverRate, TotalMarketValue, CapitalFlow, Amplitude, VolumeRatio, PeTtmRatio, PbRatio, DividendRatioTtm, FiveDayChangeRate, TenDayChangeRate, HalfYearChangeRate, FiveMinutesChangeRate, ExpiryDate, StrikePrice, UpperStrikePrice, LowerStrikePrice, OutstandingQty, OutstandingRatio, Premium, ItmOtm, ImpliedVolatility, WarrantDelta, CallPrice, ToCallPrice, EffectiveLeverage, LeverageRatio, ConversionRatio, BalancePoint, OpenInterest, Delta, Gamma, Theta, Vega, Rho
    pub indexes: Option<Vec<String>>,
}
```

- [ ] **Step 6: Update quote.rs function bodies to apply defaults**

Every function that uses these structs must apply `unwrap_or()`. Example for `candlesticks`:

```rust
pub async fn candlesticks(token: &str, p: CandlesticksParam) -> Result<CallToolResult, McpError> {
    let period = parse::parse_period(p.period.as_deref().unwrap_or("day"))?;
    let count = p.count.unwrap_or(100);
    let adjust = parse::parse_adjust_type(p.forward_adjust.unwrap_or(false));
    let sessions = parse::parse_trade_sessions(p.trade_sessions.as_deref().unwrap_or("intraday"))?;
    // ... rest unchanged
}
```

Apply the same pattern to: `trades` (count default 20), `history_candlesticks_by_offset`, `history_candlesticks_by_date`, `calc_indexes` (default 5 indexes).

- [ ] **Step 7: Build to verify**

Run: `cargo build 2>&1`
Expected: compiles successfully

- [ ] **Step 8: Commit**

```bash
git add src/tools/quote.rs
git commit -m "feat(tools): add default values to quote param structs"
```

---

### Task 2: trade.rs — Add defaults to param structs

**Files:**
- Modify: `src/tools/trade.rs:13-81`

- [ ] **Step 1: Update `SubmitOrderParam` — default order_type and time_in_force**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct SubmitOrderParam {
    /// Security symbol in CODE.MARKET format, e.g. "TSLA.US"
    pub symbol: String,
    /// Order type (default: "LO"): LO (Limit), ELO (Enhanced Limit), MO (Market), AO (At-auction), ALO (At-auction Limit), ODD (Odd Lot), LIT (Limit If Touched), MIT (Market If Touched), TSLPAMT (Trailing Limit/Amount), TSLPPCT (Trailing Limit/Percent), SLO (Special Limit)
    pub order_type: Option<String>,
    /// Order side: "Buy" or "Sell"
    pub side: String,
    /// Quantity to submit as decimal string, e.g. "100"
    pub submitted_quantity: String,
    /// Time in force (default: "Day"): "Day", "GTC" (Good Til Cancelled), "GTD" (Good Til Date)
    pub time_in_force: Option<String>,
    /// Limit price as decimal string (required for LO/ELO/ALO/ODD/LIT orders), e.g. "150.00"
    pub submitted_price: Option<String>,
    /// Trigger price for LIT/MIT orders
    pub trigger_price: Option<String>,
    /// Limit offset for TSLPAMT/TSLPPCT orders
    pub limit_offset: Option<String>,
    /// Trailing amount for TSLPAMT orders
    pub trailing_amount: Option<String>,
    /// Trailing percent for TSLPPCT orders (0-1, e.g. "0.05" for 5%)
    pub trailing_percent: Option<String>,
    /// Expiry date for GTD orders (yyyy-mm-dd)
    pub expire_date: Option<String>,
    /// Extended hours trading: "RTH_ONLY" (default), "ANY_TIME", "OVERNIGHT"
    pub outside_rth: Option<String>,
}
```

- [ ] **Step 2: Update `EstimateMaxQtyParam`**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct EstimateMaxQtyParam {
    /// Security symbol in CODE.MARKET format, e.g. "TSLA.US"
    pub symbol: String,
    /// Order side: "Buy" or "Sell"
    pub side: String,
    /// Order type (default: "LO"): LO, ELO, MO, AO, ALO
    pub order_type: Option<String>,
    /// Limit price as decimal string (required for LO orders)
    pub price: Option<String>,
}
```

- [ ] **Step 3: Update `CashFlowParam` — make dates optional with 30-day default**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct CashFlowParam {
    /// Start time in RFC3339 format (default: 30 days ago), e.g. "2024-01-01T00:00:00Z"
    pub start_at: Option<String>,
    /// End time in RFC3339 format (default: now), e.g. "2024-03-31T23:59:59Z"
    pub end_at: Option<String>,
}
```

- [ ] **Step 4: Update `HistoryOrdersParam` — make dates optional**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct HistoryOrdersParam {
    /// Filter by symbol (optional), e.g. "TSLA.US"
    pub symbol: Option<String>,
    /// Start time in RFC3339 format (default: 30 days ago)
    pub start_at: Option<String>,
    /// End time in RFC3339 format (default: now)
    pub end_at: Option<String>,
}
```

- [ ] **Step 5: Update trade.rs function bodies to apply defaults**

For `submit_order`:
```rust
let order_type_str = p.order_type.as_deref().unwrap_or("LO");
let order_type = order_type_str
    .parse::<OrderType>()
    .map_err(|e| McpError::invalid_params(format!("invalid order_type: {e}"), None))?;
let tif_str = p.time_in_force.as_deref().unwrap_or("Day");
let tif = tif_str
    .parse::<TimeInForceType>()
    .map_err(|e| McpError::invalid_params(format!("invalid time_in_force: {e}"), None))?;
```

For `estimate_max_purchase_quantity`:
```rust
let order_type_str = p.order_type.as_deref().unwrap_or("LO");
```

For `cash_flow` and `history_orders`/`history_executions` — generate default date range:
```rust
fn default_start() -> time::OffsetDateTime {
    time::OffsetDateTime::now_utc() - time::Duration::days(30)
}

fn default_end() -> time::OffsetDateTime {
    time::OffsetDateTime::now_utc()
}
```

Then in `cash_flow`:
```rust
let start = match p.start_at {
    Some(ref s) => parse::parse_rfc3339(s)?,
    None => default_start(),
};
let end = match p.end_at {
    Some(ref s) => parse::parse_rfc3339(s)?,
    None => default_end(),
};
```

Apply the same pattern to `history_orders` and `history_executions`.

- [ ] **Step 6: Build to verify**

Run: `cargo build 2>&1`
Expected: compiles successfully

- [ ] **Step 7: Commit**

```bash
git add src/tools/trade.rs
git commit -m "feat(tools): add default values to trade param structs"
```

---

### Task 3: calendar.rs — Make start/end optional

**Files:**
- Modify: `src/tools/calendar.rs`

- [ ] **Step 1: Update `FinanceCalendarParam`**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct FinanceCalendarParam {
    /// Event category: "financial", "report", "dividend", "ipo", "macrodata", "closed"
    pub category: String,
    /// Market code filter (optional): HK, US, CN, SG, JP, UK, DE, AU
    pub market: Option<String>,
    /// Start date yyyy-mm-dd (default: today)
    pub start: Option<String>,
    /// End date yyyy-mm-dd (default: 30 days from start)
    pub end: Option<String>,
}
```

- [ ] **Step 2: Update `finance_calendar` function body**

```rust
pub async fn finance_calendar(
    token: &str,
    p: FinanceCalendarParam,
) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let today = time::OffsetDateTime::now_utc().date();
    let start = p.start.unwrap_or_else(|| {
        today
            .format(time::macros::format_description!("[year]-[month]-[day]"))
            .expect("failed to format date")
    });
    let end = p.end.unwrap_or_else(|| {
        let end_date = today + time::Duration::days(30);
        end_date
            .format(time::macros::format_description!("[year]-[month]-[day]"))
            .expect("failed to format date")
    });
    let mut params: Vec<(&str, &str)> = vec![
        ("date", start.as_str()),
        ("date_end", end.as_str()),
        ("types[]", p.category.as_str()),
    ];
    let market_upper;
    if let Some(ref m) = p.market {
        market_upper = m.to_uppercase();
        params.push(("markets[]", market_upper.as_str()));
    }
    http_get_tool(&client, "/v1/quote/finance_calendar", &params).await
}
```

- [ ] **Step 3: Build and commit**

```bash
cargo build 2>&1
git add src/tools/calendar.rs
git commit -m "feat(tools): add default values to calendar param"
```

---

## Phase 2: Missing Parameters (add fields CLI has that MCP lacks)

### Task 4: quote.rs — Add missing intraday params

**Files:**
- Modify: `src/tools/quote.rs` (add `IntradayParam` struct, update `intraday` function)
- Modify: `src/tools/mod.rs` (update `intraday` tool wiring)

- [ ] **Step 1: Replace `SymbolParam` usage for intraday with new `IntradayParam`**

Add to `quote.rs` after existing param structs:

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct IntradayParam {
    /// Security symbol in CODE.MARKET format, e.g. "TSLA.US"
    pub symbol: String,
    /// Trade session filter (default: "intraday"): "intraday" or "all" (includes pre/post market)
    pub trade_sessions: Option<String>,
}
```

- [ ] **Step 2: Update `intraday` function in quote.rs**

```rust
pub async fn intraday(token: &str, p: IntradayParam) -> Result<CallToolResult, McpError> {
    let (ctx, _) = QuoteContext::new(create_config(token));
    let result = ctx
        .intraday(p.symbol)
        .await
        .map_err(Error::longbridge)?;
    tool_json(&result)
}
```

Note: The SDK's `intraday()` method does not support session filtering or historical dates — it always returns today's full intraday data. We add `trade_sessions` to the param struct for documentation purposes even though we can't pass it to the SDK yet. If the SDK adds support later, the param is already in place.

- [ ] **Step 3: Update mod.rs — change intraday tool to use `IntradayParam`**

In `mod.rs`, change the import and tool signature:

```rust
// In the use block at top, add IntradayParam
use crate::tools::quote::{
    CalcIndexesParam, CandlesticksParam, CreateWatchlistGroupParam, DeleteWatchlistGroupParam,
    HistoryCandlesticksByDateParam, HistoryCandlesticksByOffsetParam, IntradayParam,
    MarketDateRangeParam, MarketParam, SecurityListParam, SymbolCountParam, SymbolDateParam,
    SymbolParam, SymbolsParam, UpdateWatchlistGroupParam, WarrantListParam,
};

// Change the intraday tool:
async fn intraday(
    &self,
    ctx: RequestContext<RoleServer>,
    Parameters(p): Parameters<IntradayParam>,
) -> Result<CallToolResult, McpError> {
    let token = extract_access_token(&ctx)?;
    measured_tool_call("intraday", || quote::intraday(&token, p)).await
}
```

- [ ] **Step 4: Build and commit**

```bash
cargo build 2>&1
git add src/tools/quote.rs src/tools/mod.rs
git commit -m "feat(tools): add trade_sessions param to intraday tool"
```

---

### Task 5: fundamental.rs — Add missing params to existing tools

**Files:**
- Modify: `src/tools/fundamental.rs`
- Modify: `src/tools/mod.rs`

- [ ] **Step 1: Update `FinancialReportParam` — add kind**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct FinancialReportParam {
    /// Security symbol, e.g. "AAPL.US"
    pub symbol: String,
    /// Report type (default: "annual"): "annual" or "quarterly"
    pub report_type: Option<String>,
    /// Statement type (default: all three): "IS" (income statement), "BS" (balance sheet), "CF" (cash flow), or omit for all
    pub kind: Option<String>,
}
```

- [ ] **Step 2: Add `ValuationHistoryParam`**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct ValuationHistoryParam {
    /// Security symbol, e.g. "AAPL.US"
    pub symbol: String,
    /// Valuation indicator (default: "pe"): "pe", "pb", "ps", "dvd_yld"
    pub indicator: Option<String>,
    /// Historical range in years (default: 1): 1, 3, 5, or 10
    pub range: Option<i32>,
}
```

- [ ] **Step 3: Add `ShareholderParam` with sort/filter**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct ShareholderParam {
    /// Security symbol, e.g. "AAPL.US"
    pub symbol: String,
    /// Filter by holding change (default: "all"): "all", "inc" (increased), "dec" (decreased)
    pub range: Option<String>,
    /// Sort field (default: "chg"): "chg" (change), "owned" (holdings), "time" (report date)
    pub sort: Option<String>,
    /// Sort order (default: "desc"): "desc" or "asc"
    pub order: Option<String>,
}
```

- [ ] **Step 4: Add `FundHolderParam` with count**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct FundHolderParam {
    /// Security symbol, e.g. "AAPL.US"
    pub symbol: String,
    /// Number of results to return (default: 20, use -1 for all)
    pub count: Option<i32>,
}
```

- [ ] **Step 5: Add `IndustryValuationParam` with currency**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct IndustryValuationParam {
    /// Security symbol, e.g. "AAPL.US"
    pub symbol: String,
    /// Currency for valuation (default: "USD"): "USD", "HKD", "CNY", "SGD"
    pub currency: Option<String>,
}
```

- [ ] **Step 6: Add `OperatingParam` with report filter**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct OperatingParam {
    /// Security symbol, e.g. "AAPL.US"
    pub symbol: String,
    /// Report kind filter: "af" (annual), "saf" (semi-annual), "q1", "q3", or omit for all
    pub report: Option<String>,
}
```

- [ ] **Step 7: Update function signatures and bodies**

Update `financial_report` to pass `kind` param:
```rust
pub async fn financial_report(
    token: &str,
    p: FinancialReportParam,
) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let cid = symbol_to_counter_id(&p.symbol);
    let mut params: Vec<(&str, &str)> = vec![("counter_id", cid.as_str())];
    let report_type = p.report_type.unwrap_or_default();
    if !report_type.is_empty() {
        params.push(("report", report_type.as_str()));
    }
    let kind = p.kind.unwrap_or_default();
    if !kind.is_empty() {
        params.push(("kind", kind.as_str()));
    }
    http_get_tool(&client, "/v1/quote/financial-reports", &params).await
}
```

Update `valuation_history` to use new param:
```rust
pub async fn valuation_history(
    token: &str,
    p: ValuationHistoryParam,
) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let cid = symbol_to_counter_id(&p.symbol);
    let indicator = p.indicator.unwrap_or_else(|| "pe".to_string());
    let range_str = p.range.unwrap_or(1).to_string();
    http_get_tool(
        &client,
        "/v1/quote/valuation/detail",
        &[
            ("counter_id", cid.as_str()),
            ("indicator", indicator.as_str()),
            ("range", range_str.as_str()),
        ],
    )
    .await
}
```

Update `shareholder`:
```rust
pub async fn shareholder(token: &str, p: ShareholderParam) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let cid = symbol_to_counter_id(&p.symbol);
    let range = p.range.unwrap_or_else(|| "all".to_string());
    let sort = p.sort.unwrap_or_else(|| "chg".to_string());
    let order = p.order.unwrap_or_else(|| "desc".to_string());
    http_get_tool(
        &client,
        "/v1/quote/shareholders",
        &[
            ("counter_id", cid.as_str()),
            ("range", range.as_str()),
            ("sort", sort.as_str()),
            ("order", order.as_str()),
        ],
    )
    .await
}
```

Update `fund_holder`:
```rust
pub async fn fund_holder(token: &str, p: FundHolderParam) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let cid = symbol_to_counter_id(&p.symbol);
    let count_str = p.count.unwrap_or(20).to_string();
    http_get_tool(
        &client,
        "/v1/quote/fund-holders",
        &[("counter_id", cid.as_str()), ("count", count_str.as_str())],
    )
    .await
}
```

Update `industry_valuation`:
```rust
pub async fn industry_valuation(
    token: &str,
    p: IndustryValuationParam,
) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let cid = symbol_to_counter_id(&p.symbol);
    let currency = p.currency.unwrap_or_else(|| "USD".to_string());
    http_get_tool(
        &client,
        "/v1/quote/industry-valuation-comparison",
        &[
            ("counter_id", cid.as_str()),
            ("currency", currency.as_str()),
        ],
    )
    .await
}
```

Update `operating`:
```rust
pub async fn operating(token: &str, p: OperatingParam) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let cid = symbol_to_counter_id(&p.symbol);
    let mut params: Vec<(&str, &str)> = vec![("counter_id", cid.as_str())];
    let report = p.report.unwrap_or_default();
    if !report.is_empty() {
        params.push(("report", report.as_str()));
    }
    http_get_tool(&client, "/v1/quote/operatings", &params).await
}
```

- [ ] **Step 8: Update mod.rs — wire new param types**

In `mod.rs`, update imports and tool signatures for all changed fundamentals:

```rust
// Change these tool signatures to use new param types:
// valuation_history: Parameters<fundamental::SymbolParam> → Parameters<fundamental::ValuationHistoryParam>
// shareholder: Parameters<fundamental::SymbolParam> → Parameters<fundamental::ShareholderParam>
// fund_holder: Parameters<fundamental::SymbolParam> → Parameters<fundamental::FundHolderParam>
// industry_valuation: Parameters<fundamental::SymbolParam> → Parameters<fundamental::IndustryValuationParam>
// operating: Parameters<fundamental::SymbolParam> → Parameters<fundamental::OperatingParam>
```

- [ ] **Step 9: Build and commit**

```bash
cargo build 2>&1
git add src/tools/fundamental.rs src/tools/mod.rs
git commit -m "feat(tools): add missing params to fundamental tools"
```

---

### Task 6: market.rs — Add missing params

**Files:**
- Modify: `src/tools/market.rs`
- Modify: `src/tools/mod.rs`

- [ ] **Step 1: Add `BrokerHoldingParam` with period**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct BrokerHoldingParam {
    /// Security symbol, e.g. "700.HK" (HK market only)
    pub symbol: String,
    /// Period for top buyer/seller ranking (default: "rct_1"): "rct_1" (1 day), "rct_5" (5 days), "rct_20" (20 days), "rct_60" (60 days)
    pub period: Option<String>,
}
```

- [ ] **Step 2: Add `AhPremiumParam` with kline_type and count**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct AhPremiumParam {
    /// Security symbol of HK dual-listed stock, e.g. "939.HK", "1398.HK"
    pub symbol: String,
    /// K-line type (default: "day"): 1m, 5m, 15m, 30m, 60m, day, week, month, year
    pub kline_type: Option<String>,
    /// Number of K-lines to return (default: 100)
    pub count: Option<i32>,
}
```

- [ ] **Step 3: Add `AnomalyParam` with symbol and count**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct AnomalyParam {
    /// Market code (default: "HK"): HK, US, CN, SG
    pub market: Option<String>,
    /// Filter to a specific symbol (optional)
    pub symbol: Option<String>,
    /// Number of results (default: 50, max: 100)
    pub count: Option<i32>,
}
```

- [ ] **Step 4: Add `ConstituentParam` with limit, sort, order**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct ConstituentParam {
    /// Index symbol, e.g. "HSI.HK", "DJI.US"
    pub symbol: String,
    /// Number of results (default: 50)
    pub limit: Option<i32>,
    /// Sort indicator (default: "change"): "change", "price", "turnover", "inflow", "turnover-rate", "market-cap"
    pub sort: Option<String>,
    /// Sort order (default: "desc"): "desc" or "asc"
    pub order: Option<String>,
}
```

- [ ] **Step 5: Update function signatures and bodies**

`broker_holding`:
```rust
pub async fn broker_holding(token: &str, p: BrokerHoldingParam) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let cid = symbol_to_counter_id(&p.symbol);
    let period = p.period.unwrap_or_else(|| "rct_1".to_string());
    http_get_tool(
        &client,
        "/v1/quote/broker-holding",
        &[("counter_id", cid.as_str()), ("period", period.as_str())],
    )
    .await
}
```

`ah_premium`:
```rust
pub async fn ah_premium(token: &str, p: AhPremiumParam) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let cid = symbol_to_counter_id(&p.symbol);
    let kline_type = p.kline_type.unwrap_or_else(|| "day".to_string());
    let count_str = p.count.unwrap_or(100).to_string();
    http_get_tool(
        &client,
        "/v1/quote/ahpremium/klines",
        &[
            ("counter_id", cid.as_str()),
            ("kline_type", kline_type.as_str()),
            ("count", count_str.as_str()),
        ],
    )
    .await
}
```

`anomaly`:
```rust
pub async fn anomaly(token: &str, p: AnomalyParam) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let market = p.market.unwrap_or_else(|| "HK".to_string()).to_uppercase();
    let count_str = p.count.unwrap_or(50).to_string();
    let mut params: Vec<(&str, &str)> = vec![
        ("market", market.as_str()),
        ("category", "0"),
        ("count", count_str.as_str()),
    ];
    let cid;
    if let Some(ref sym) = p.symbol {
        cid = symbol_to_counter_id(sym);
        params.push(("counter_id", cid.as_str()));
    }
    http_get_tool(&client, "/v1/quote/changes", &params).await
}
```

`constituent`:
```rust
pub async fn constituent(token: &str, p: ConstituentParam) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let cid = index_symbol_to_counter_id(&p.symbol);
    let limit_str = p.limit.unwrap_or(50).to_string();
    let sort = p.sort.unwrap_or_else(|| "change".to_string());
    let order = p.order.unwrap_or_else(|| "desc".to_string());
    http_get_tool(
        &client,
        "/v1/quote/index-constituents",
        &[
            ("counter_id", cid.as_str()),
            ("limit", limit_str.as_str()),
            ("sort", sort.as_str()),
            ("order", order.as_str()),
        ],
    )
    .await
}
```

- [ ] **Step 6: Update mod.rs — wire new param types**

Change tool signatures:
- `broker_holding`: `Parameters<market::SymbolParam>` → `Parameters<market::BrokerHoldingParam>`
- `ah_premium`: `Parameters<market::SymbolParam>` → `Parameters<market::AhPremiumParam>`
- `anomaly`: `Parameters<market::MarketParam>` → `Parameters<market::AnomalyParam>`
- `constituent`: `Parameters<market::IndexSymbolParam>` → `Parameters<market::ConstituentParam>`

- [ ] **Step 7: Build and commit**

```bash
cargo build 2>&1
git add src/tools/market.rs src/tools/mod.rs
git commit -m "feat(tools): add missing params to market tools"
```

---

### Task 7: content.rs + alert.rs + portfolio.rs — Add missing params

**Files:**
- Modify: `src/tools/content.rs`
- Modify: `src/tools/alert.rs`
- Modify: `src/tools/portfolio.rs`
- Modify: `src/tools/mod.rs`

- [ ] **Step 1: content.rs — Add count param to news and topic**

Add `NewsParam`:
```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct NewsParam {
    /// Security symbol in CODE.MARKET format, e.g. "TSLA.US"
    pub symbol: String,
    /// Maximum number of articles (default: 20)
    pub count: Option<usize>,
}
```

Add `TopicListParam`:
```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct TopicListParam {
    /// Security symbol in CODE.MARKET format, e.g. "TSLA.US"
    pub symbol: String,
    /// Maximum number of topics (default: 20)
    pub count: Option<usize>,
}
```

Update `news` function (the SDK's `ContentContext::news` may not support count — pass it if available, otherwise just note in doc):
```rust
pub async fn news(token: &str, p: NewsParam) -> Result<CallToolResult, McpError> {
    let ctx = ContentContext::new(create_config(token));
    let result = ctx.news(p.symbol).await.map_err(Error::longbridge)?;
    // SDK returns all results; truncate to count if specified
    let count = p.count.unwrap_or(20);
    let truncated: Vec<_> = result.into_iter().take(count).collect();
    tool_json(&truncated)
}
```

Update `topic` function similarly:
```rust
pub async fn topic(token: &str, p: TopicListParam) -> Result<CallToolResult, McpError> {
    let ctx = ContentContext::new(create_config(token));
    let result = ctx.topics(p.symbol).await.map_err(Error::longbridge)?;
    let count = p.count.unwrap_or(20);
    let truncated: Vec<_> = result.into_iter().take(count).collect();
    tool_json(&truncated)
}
```

- [ ] **Step 2: alert.rs — Add optional symbol filter to alert_list**

Currently `alert_list` takes no params. It cannot be changed to take a param struct without modifying the tool signature in mod.rs. We'll add an `AlertListParam`:

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct AlertListParam {
    /// Filter by symbol (optional), e.g. "TSLA.US". Omit to list all alerts.
    pub symbol: Option<String>,
}
```

Update `alert_list`:
```rust
pub async fn alert_list(token: &str, p: AlertListParam) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let result = http_get_tool(&client, "/v1/notify/reminders", &[]).await?;

    // If symbol filter is specified, filter client-side
    if let Some(ref symbol) = p.symbol {
        let cid = symbol_to_counter_id(symbol);
        let text = result
            .content
            .first()
            .and_then(|c| c.as_text())
            .map(|t| t.text.as_str())
            .unwrap_or("{}");
        if let Ok(mut data) = serde_json::from_str::<serde_json::Value>(text) {
            if let Some(lists) = data
                .get_mut("lists")
                .or_else(|| data.get_mut("list"))
                .and_then(|v| v.as_array_mut())
            {
                lists.retain(|item| {
                    item["counter_id"].as_str() == Some(cid.as_str())
                });
            }
            return Ok(CallToolResult::success(vec![
                rmcp::model::Content::text(data.to_string()),
            ]));
        }
    }
    Ok(result)
}
```

- [ ] **Step 3: portfolio.rs — Add missing params to profit_analysis_detail**

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct ProfitAnalysisDetailParam {
    /// Security symbol, e.g. "700.HK"
    pub symbol: String,
    /// Start date yyyy-mm-dd (optional)
    pub start: Option<String>,
    /// End date yyyy-mm-dd (optional)
    pub end: Option<String>,
    /// Currency filter, e.g. "HKD", "USD", "CNH" (optional)
    pub currency: Option<String>,
}
```

Update function:
```rust
pub async fn profit_analysis_detail(
    token: &str,
    p: ProfitAnalysisDetailParam,
) -> Result<CallToolResult, McpError> {
    let client = create_http_client(token);
    let cid = symbol_to_counter_id(&p.symbol);
    let mut params: Vec<(&str, &str)> = vec![("counter_id", cid.as_str())];
    if let Some(ref s) = p.start {
        params.push(("start_date", s.as_str()));
    }
    if let Some(ref e) = p.end {
        params.push(("end_date", e.as_str()));
    }
    if let Some(ref c) = p.currency {
        params.push(("currency", c.as_str()));
    }
    http_get_tool(&client, "/v1/portfolio/profit-analysis/detail", &params).await
}
```

- [ ] **Step 4: Update mod.rs — wire all new param types**

Changes needed:
- `news`: `Parameters<content::SymbolParam>` → `Parameters<content::NewsParam>`
- `topic`: `Parameters<content::SymbolParam>` → `Parameters<content::TopicListParam>`
- `alert_list`: add `Parameters<alert::AlertListParam>` param
- `profit_analysis_detail`: already uses `portfolio::ProfitAnalysisDetailParam` (struct definition changed in place)

For `alert_list` in mod.rs, change from:
```rust
async fn alert_list(&self, ctx: RequestContext<RoleServer>) -> Result<CallToolResult, McpError> {
    let token = extract_access_token(&ctx)?;
    measured_tool_call("alert_list", || alert::alert_list(&token)).await
}
```
To:
```rust
async fn alert_list(
    &self,
    ctx: RequestContext<RoleServer>,
    Parameters(p): Parameters<alert::AlertListParam>,
) -> Result<CallToolResult, McpError> {
    let token = extract_access_token(&ctx)?;
    measured_tool_call("alert_list", || alert::alert_list(&token, p)).await
}
```

- [ ] **Step 5: Build and commit**

```bash
cargo build 2>&1
git add src/tools/content.rs src/tools/alert.rs src/tools/portfolio.rs src/tools/mod.rs
git commit -m "feat(tools): add missing params to content, alert, portfolio tools"
```

---

## Phase 3: Tool Descriptions (align with CLI detail level)

### Task 8: mod.rs — Enrich all tool descriptions

**Files:**
- Modify: `src/tools/mod.rs` (every `#[tool(description = "...")]` attribute)

The principle: each description should include (1) what the tool does, (2) key return fields, (3) market/parameter constraints. Keep it as a single string — no multi-line in the macro.

- [ ] **Step 1: Quote tool descriptions**

```rust
#[tool(description = "Get basic security info. Returns: name, exchange, currency, lot_size, total_shares, circulating_shares, EPS, BPS, dividend")]
async fn static_info(...)

#[tool(description = "Get real-time price quotes. Returns: symbol, last_done, prev_close, open, high, low, volume, turnover, trade_status, pre_market_quote (US), post_market_quote (US), overnight_quote (US)")]
async fn quote(...)

#[tool(description = "Get option quotes for up to 500 symbols. Returns: symbol, last_done, prev_close, open, high, low, volume, turnover, implied_volatility, delta, gamma, theta, vega")]
async fn option_quote(...)

#[tool(description = "Get warrant quotes. Returns: symbol, last_done, prev_close, open, high, low, volume, turnover, implied_volatility, delta, premium")]
async fn warrant_quote(...)

#[tool(description = "Get Level 2 order book depth — up to 10 price levels of asks and bids. Returns: price, volume, order_num per level. HK stocks have 10 levels, US have 1")]
async fn depth(...)

#[tool(description = "Get broker queue at each bid/ask price level (HK market). Returns: broker IDs at each level. Use 'participants' tool to resolve broker IDs to names")]
async fn brokers(...)

#[tool(description = "Get market participant broker IDs and names. Use to interpret broker IDs from 'brokers' and 'broker_holding' tools")]
async fn participants(...)

#[tool(description = "Get recent tick-by-tick trades. Returns: timestamp, price, volume, direction (up/down/neutral), trade_type")]
async fn trades(...)

#[tool(description = "Get intraday minute-by-minute price and volume lines for today. Returns: timestamp, price, volume, turnover, avg_price")]
async fn intraday(...)

#[tool(description = "Get OHLCV candlestick (K-line) data. Returns: timestamp, open, high, low, close, volume, turnover")]
async fn candlesticks(...)

#[tool(description = "Get historical candlestick data by offset from a reference time. Returns: timestamp, open, high, low, close, volume, turnover")]
async fn history_candlesticks_by_offset(...)

#[tool(description = "Get historical candlestick data by date range. Returns: timestamp, open, high, low, close, volume, turnover")]
async fn history_candlesticks_by_date(...)

#[tool(description = "Get trading calendar — which days are trading days vs holidays for a market")]
async fn trading_days(...)

#[tool(description = "Get option chain expiry dates for an underlying symbol. Returns list of available expiry dates")]
async fn option_chain_expiry_date_list(...)

#[tool(description = "Get option chain detail for a specific expiry date. Returns: strike prices, call/put symbols, Greeks (delta, gamma, theta, vega), open_interest")]
async fn option_chain_info_by_date(...)

#[tool(description = "Get intraday capital inflow/outflow time series. Returns: timestamp, inflow, outflow for each interval")]
async fn capital_flow(...)

#[tool(description = "Get capital distribution snapshot — large/medium/small order holder flows. Returns: buy_large, sell_large, buy_medium, sell_medium, buy_small, sell_small")]
async fn capital_distribution(...)

#[tool(description = "Get trading session schedule for all markets. Returns: market, trade_sessions with begin/end times")]
async fn trading_session(...)

#[tool(description = "Get market sentiment temperature (0-100, higher = more bullish). Returns: temperature value and timestamp")]
async fn market_temperature(...)

#[tool(description = "Get historical market temperature time series. Returns: date, temperature for each day in range")]
async fn history_market_temperature(...)

#[tool(description = "Get all watchlist groups and their securities. Returns: group id, name, and securities list")]
async fn watchlist(...)

#[tool(description = "Get regulatory filings (8-K, 10-Q, 10-K, etc.) for a symbol. Returns: filing_id, title, filing_type, published_at")]
async fn filings(...)

#[tool(description = "Get warrant issuer information. Returns: issuer name, issuer id")]
async fn warrant_issuers(...)

#[tool(description = "Get filtered warrant list for an underlying symbol. Returns: symbol, last_done, change_rate, volume, expiry_date, strike_price, premium, implied_volatility")]
async fn warrant_list(...)

#[tool(description = "Calculate financial indexes for symbols. Returns: per-symbol values for requested indexes (PE, PB, dividend yield, turnover rate, etc.)")]
async fn calc_indexes(...)

#[tool(description = "Create a new watchlist group with optional initial securities. Returns: group id")]
async fn create_watchlist_group(...)

#[tool(description = "Delete a watchlist group by id")]
async fn delete_watchlist_group(...)

#[tool(description = "Update a watchlist group — rename or add/remove/replace securities")]
async fn update_watchlist_group(...)

#[tool(description = "Get security list for a market. Currently only supports US market overnight-eligible securities")]
async fn security_list(...)
```

- [ ] **Step 2: Trade tool descriptions**

```rust
#[tool(description = "Get account cash balance and asset summary. Returns: currency, net_assets, total_cash, buy_power, init_margin, maintenance_margin, risk_level, and per-currency cash breakdown")]
async fn account_balance(...)

#[tool(description = "Get current stock positions across all sub-accounts. Returns: symbol, name, quantity, available_quantity, cost_price, currency, market")]
async fn stock_positions(...)

#[tool(description = "Get current fund (mutual fund) positions. Returns: symbol, name, current_net_asset_value, cost_net_asset_value, currency, holding_units")]
async fn fund_positions(...)

#[tool(description = "Get margin ratio requirements for a symbol. Returns: im_factor (initial margin), mm_factor (maintenance margin), fm_factor (forced liquidation margin)")]
async fn margin_ratio(...)

#[tool(description = "Get all orders placed today. Returns: order_id, symbol, side, order_type, price, quantity, status, submitted_at")]
async fn today_orders(...)

#[tool(description = "Get detailed information about a specific order by order_id. Returns: full order details including fills, status history")]
async fn order_detail(...)

#[tool(description = "Cancel an open order by order_id. Returns confirmation message")]
async fn cancel_order(...)

#[tool(description = "Get today's trade executions (fills). Returns: order_id, symbol, side, price, quantity, trade_done_at")]
async fn today_executions(...)

#[tool(description = "Get historical orders between dates (excludes today's orders). Returns: order_id, symbol, side, order_type, price, quantity, status")]
async fn history_orders(...)

#[tool(description = "Get historical trade executions between dates. Returns: order_id, symbol, side, price, quantity, trade_done_at")]
async fn history_executions(...)

#[tool(description = "Get cash flow records — deposits, withdrawals, dividends, settlements. Returns: flow_name, symbol, business_type, balance, currency, business_time")]
async fn cash_flow(...)

#[tool(description = "Submit a buy/sell order. Returns: order_id on success")]
async fn submit_order(...)

#[tool(description = "Replace/modify an existing open order — change quantity and/or price")]
async fn replace_order(...)

#[tool(description = "Estimate maximum buy/sell quantity for a symbol given current account balance. Returns: cash_max_qty, margin_max_qty")]
async fn estimate_max_purchase_quantity(...)
```

- [ ] **Step 3: Fundamental tool descriptions**

```rust
#[tool(description = "Get financial statements (income statement, balance sheet, cash flow). Returns: report periods with line items (revenue, net_income, total_assets, etc.)")]
async fn financial_report(...)

#[tool(description = "Get institution rating summary — analyst consensus (Strong Buy/Buy/Hold/Underperform/Sell count) and average target price")]
async fn institution_rating(...)

#[tool(description = "Get detailed historical institution ratings and target price changes. Returns: analyst name, institution, rating, target_price, date")]
async fn institution_rating_detail(...)

#[tool(description = "Get dividend history — past dividend payments. Returns: ex_date, pay_date, amount, currency, dividend_type")]
async fn dividend(...)

#[tool(description = "Get detailed dividend distribution scheme — record dates, tax rates, per-share amounts")]
async fn dividend_detail(...)

#[tool(description = "Get EPS forecast and analyst estimate history. Returns: fiscal_year, consensus_eps, analyst estimates over time")]
async fn forecast_eps(...)

#[tool(description = "Get financial consensus estimates — revenue, EPS, net income forecasts by fiscal year. Returns: consensus values with high/low/mean")]
async fn consensus(...)

#[tool(description = "Get valuation overview — current PE, PB, PS, dividend yield with 5-year range and peer comparison")]
async fn valuation(...)

#[tool(description = "Get historical valuation time series. Returns: date, indicator value over selected time range")]
async fn valuation_history(...)

#[tool(description = "Get industry valuation comparison — PE, PB, PS for a symbol vs industry peers")]
async fn industry_valuation(...)

#[tool(description = "Get industry valuation distribution — percentile ranking of PE/PB/PS among peers")]
async fn industry_valuation_dist(...)

#[tool(description = "Get company overview — name, founding date, CEO, employees, IPO date, business description, headquarters address")]
async fn company(...)

#[tool(description = "Get company executives and board members — name, title, age, compensation")]
async fn executive(...)

#[tool(description = "Get institutional shareholders. Returns: shareholder name, % shares held, share change, report date")]
async fn shareholder(...)

#[tool(description = "Get funds and ETFs that hold a given symbol. Returns: fund name, ticker, currency, weight (position ratio), report date")]
async fn fund_holder(...)

#[tool(description = "Get corporate actions — stock splits, buybacks, name changes, rights issues. Returns: action type, date, details")]
async fn corp_action(...)

#[tool(description = "Get investor relations events and announcements — subsidiary/parent company relationships")]
async fn invest_relation(...)

#[tool(description = "Get company operating metrics by report period — revenue breakdown, user metrics, segment data")]
async fn operating(...)
```

- [ ] **Step 4: Market, Calendar, Content, Alert, Portfolio, Statement descriptions**

```rust
// Market
#[tool(description = "Get current market open/close status for all exchanges. Returns: market, status (pre_open/trading/closing/closed), trade_session times")]
async fn market_status(...)

#[tool(description = "Get top broker holding positions for a HK stock. Returns: broker name, holding quantity, holding ratio, change. HK market only")]
async fn broker_holding(...)

#[tool(description = "Get full broker holding detail list for a HK stock — all brokers and their positions")]
async fn broker_holding_detail(...)

#[tool(description = "Get daily holding history for a specific broker on a HK stock. Returns: date, holding quantity, change")]
async fn broker_holding_daily(...)

#[tool(description = "Get A/H share premium historical K-line data. Only for HK stocks dual-listed in A-share markets (e.g. 939.HK, 1398.HK). Returns: date, premium_rate")]
async fn ah_premium(...)

#[tool(description = "Get A/H share premium intraday time-share data for dual-listed stocks. Returns: timestamp, premium_rate")]
async fn ah_premium_intraday(...)

#[tool(description = "Get trade statistics — price distribution by buy/sell/neutral volume. Returns: buy_volume, sell_volume, neutral_volume, total_volume")]
async fn trade_stats(...)

#[tool(description = "Get market anomaly alerts — unusual price or volume changes. Returns: symbol, name, change_rate, volume_ratio, anomaly type")]
async fn anomaly(...)

#[tool(description = "Get constituent stocks of an index (e.g. HSI.HK, DJI.US, SPX.US). Returns: symbol, name, weight, change_rate")]
async fn constituent(...)

// Calendar
#[tool(description = "Get finance calendar events — earnings, dividends, IPOs, macro data releases, market closures. Returns: event date, type, details. Use category to filter event type")]
async fn finance_calendar(...)

// Portfolio
#[tool(description = "Get exchange rates for all supported currency pairs. Returns: from_currency, to_currency, rate")]
async fn exchange_rate(...)

#[tool(description = "Get portfolio profit and loss analysis summary — total P/L across all holdings")]
async fn profit_analysis(...)

#[tool(description = "Get detailed profit and loss analysis for a specific symbol — cost basis, realized/unrealized P/L, transaction flows")]
async fn profit_analysis_detail(...)

// Alert
#[tool(description = "Get all configured price alerts. Returns: alert_id, symbol, condition, threshold, enabled status")]
async fn alert_list(...)

#[tool(description = "Add a price alert — triggered when price or percentage change reaches threshold")]
async fn alert_add(...)

#[tool(description = "Delete a price alert by alert_id")]
async fn alert_delete(...)

#[tool(description = "Enable a previously disabled price alert by alert_id")]
async fn alert_enable(...)

#[tool(description = "Disable a price alert by alert_id (keeps the alert but stops triggering)")]
async fn alert_disable(...)

// Content
#[tool(description = "Get latest news articles for a symbol. Returns: news_id, title, published_at, source, likes, comments")]
async fn news(...)

#[tool(description = "Get full news article content by news_id. Returns: title, body (HTML), published_at, source")]
async fn news_detail(...)

#[tool(description = "Get community discussion topics for a symbol. Returns: topic_id, title, body, likes, comments, created_at")]
async fn topic(...)

#[tool(description = "Get discussion topic detail by topic_id. Returns: full topic content with body")]
async fn topic_detail(...)

#[tool(description = "Get replies to a discussion topic. Returns: reply_id, body, author, created_at")]
async fn topic_replies(...)

#[tool(description = "Create a new community discussion topic. Returns: topic_id")]
async fn topic_create(...)

#[tool(description = "Create a reply to a discussion topic. Returns: reply_id")]
async fn topic_create_reply(...)

#[tool(description = "Get regulatory filing detail by filing_id. Returns: full filing content (HTML or text)")]
async fn filing_detail(...)

// Statement
#[tool(description = "List available account statements (daily/monthly). Returns: file_key, date, statement_type — use file_key with statement_export to download")]
async fn statement_list(...)

#[tool(description = "Export account statement content by file_key. Returns: statement sections (equity_holdings, cash_flow, etc.)")]
async fn statement_export(...)
```

- [ ] **Step 5: Update server instructions in `#[tool_handler]`**

```rust
#[tool_handler(
    name = "longbridge-mcp",
    version = "0.1.0",
    instructions = "Longbridge OpenAPI MCP Server — 90 tools for market data, trading, and financial analysis. Symbol format: CODE.MARKET (e.g. TSLA.US, 700.HK, D05.SG, 600519.SH). Most parameters have sensible defaults; only symbol is usually required."
)]
impl ServerHandler for Longbridge {}
```

- [ ] **Step 6: Build and commit**

```bash
cargo build 2>&1
git add src/tools/mod.rs
git commit -m "feat(tools): enrich all tool descriptions with return fields and constraints"
```

---

### Task 9: Final verification

- [ ] **Step 1: Format and lint**

```bash
cargo +nightly fmt
cargo clippy --all-features --all-targets
```

Fix any warnings.

- [ ] **Step 2: Run server and verify JSON schema**

```bash
pkill -f 'target/debug/longbridge-mcp' 2>/dev/null
LONGBRIDGE_HTTP_URL=https://openapi.longbridge.xyz cargo run &
sleep 3
```

Then initialize and list tools:
```bash
SESSION=$(curl -s -X POST http://127.0.0.1:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"0.1"}}}' \
  -D - 2>/dev/null | grep -i mcp-session-id | cut -d' ' -f2 | tr -d '\r')

curl -s -X POST http://127.0.0.1:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer TOKEN" \
  -H "Mcp-Session-Id: $SESSION" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' | grep '^data:' | grep -v '^data: $' | sed 's/^data: //' | python3 -m json.tool > /tmp/tools.json
```

Verify: (1) total tools still 90, (2) default values visible in schema, (3) descriptions enriched, (4) new optional fields present.

- [ ] **Step 3: Test a tool call with defaults (omitting optional params)**

```bash
curl -s -X POST http://127.0.0.1:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer TOKEN" \
  -H "Mcp-Session-Id: $SESSION" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"candlesticks","arguments":{"symbol":"TSLA.US"}}}'
```

Expected: returns 100 daily candlesticks (defaults applied) instead of error about missing required fields.

- [ ] **Step 4: Commit if any fixes were needed**

```bash
cargo +nightly fmt
git add -u
git commit -m "fix: address clippy warnings and formatting"
```
