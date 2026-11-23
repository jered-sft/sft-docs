# Pricing regimes (plan, not started — one PR)

Decisions (user, 2026-10-14/15):
- Source preference is an ordered list of `{price_source, execution_max_age_hours}`.
- Spreads are per price source, applied at ingest, and only to single-price feeds; whether that price is mid, bid or ask is configurable.
- Offsets live on the pricing regime as a buy/sell list, apply in both directions, and the most specific entry wins.
- All rates are fractions in the API and DB (`0.025` = 2.5%).
- Migration: per ledger, one "Default" regime (`gsc_10am_reference_price` 72h, `open_exchange_rates` 168h) plus one "Manual" regime; everything points at them.
- Replace existing call sites; ledger-accounting valuations use plain non-merchant `FetchPrice`.
- `integral_stream` becomes its own source; its workers are unregistered.
- Rename `integral_10am` / `models.Integral10am` → `gsc_10am_reference_price` / `models.GSC10amReferencePrice` everywhere (code, DB, API, docs).
- Drop `execution_prices.args` and `pair_price_offsets`.
- One PR (not in production).

## 1. `price_source_configs` (global — quote_prices are not ledger-scoped)

| Column | Type | Notes |
|---|---|---|
| id, created_at, updated_at | | EntityBase |
| price_source | varchar(64) unique not null | one row per `models.PriceSource` |
| single_price_as | varchar(8) not null default 'mid' | mid / bid / ask |
| spread_up | decimal(20,8) not null default 0 | fraction; produces ask |
| spread_down | decimal(20,8) not null default 0 | fraction; produces bid |

Applied only in `IngestQuote` (single-price feeds: `open_exchange_rates` daily, `gsc_10am_reference_price`). Two-sided feeds (stream, manual) ignore it.
- mid → bid = p × (1 − down), ask = p × (1 + up)
- bid → bid = p, ask = p × (1 + up)
- ask → ask = p, bid = p × (1 − down)

- `quotePricesFromPairs` takes the config. A missing row means mid 0/0, which is today's behaviour.
- `quote_prices_raw` gets `single_price_as`, `spread_up`, `spread_down` copied at ingest, so each row records how its bid/ask were made.
- Config changes affect only future ingests. Re-running a raw re-derives its prices with the current config.
- Backoffice: `GET /api/price-sources` includes config; `PATCH /api/price-sources/:source` (super admin). Seed one row per source.

### Stream source
- Add `PriceSourceIntegralStream = "integral_stream"`. `PersistStreamQuotePrices` / `IngestStreamQuotes` take the source instead of hardcoding `open_exchange_rates`.
- Unregister `integral_price_fetch` and `stream_price_fetch` (the latter already has a nil fetcher, a no-op) in `server/workers/worker.go`.
  Keep the packages so re-enabling is one line. Drop the now-unused `INTEGRAL_PRICE_STREAM_DISABLED` check? (keep; harmless)
- No Redis live-cache branch in regime lookups: with the stream off nothing writes Redis prices, and no regime lists `integral_stream`.
  `GetLiveLatestPrice` / `liveOrDBFetcher` are removed; live requests read the DB through the regime.

### Rename `integral_10am` → `gsc_10am_reference_price`
- Constant: `models.Integral10am` → `models.GSC10amReferencePrice = "gsc_10am_reference_price"` in [price_source.go](../internal/models/price_source.go) and `KnownPriceSources`.
- Go identifiers that carry the old name:
  - `IngestIntegral10amQuote` → `IngestGSC10amReferencePrice` (gsc_integral_fetch ingest/job_creator/job_executor, `etl_handler`)
  - error strings "ingest integral 10am" → "ingest gsc 10am reference price"
  - `metrics_service` `tenAMSource`
  - the job_creator comment
- Rename the `gsc_integral_fetch` package/worker too? It fetches GSC, not Integral. Keep the package name (not asked); only the source string changes.
- Tests (string literals and constants):
  - savings_plan_execute, storage_charge_execute, gsc_integral_fetch
  - metrics_service, ingest_service, quote_price_raw_handler
  - `models/price_source_test`
  - `workers/engine_test`, `event_log_repository_test`, `jobqueue_test` (job key), `pubsub_test` (event name)
- Docs/API: `backoffice-api.md` (price-sources example, quote_price_raw example), `api-changes.md` (schedule_value), `price/README.md`, `price/service.go` PriceFetchArgs comment, swagger regen.
- DB (in the migration, before the regime backfill so it maps the new name):
  - `quote_prices_raw.source`; the unique (timestamp, source) index is unaffected
  - `event_logs.context` where it equals the old name
  - `job_pollers` / process job keys that embed the source, if persisted (check the column during implementation)
  - `savings_plan_regimes.quote_price_source`, `reference_price_sources`, `storage_fee_regimes.price_source` are dropped by this migration anyway
  - `execution_prices` rows are deleted anyway
  - Down migration reverses the string updates.

## 2. `pricing_regimes` (ledger-scoped)

| Column | Type | Notes |
|---|---|---|
| id, created_at, updated_at | | EntityBase |
| ledger_id | uuid not null FK ledgers, index | |
| name | varchar(255) not null | unique (ledger_id, name) |
| description | text not null default '' | |
| source_preference | jsonb not null | `[{price_source, execution_max_age_hours}]`; ≥1, known sources, no dups, hours 0–720 |
| offsets | jsonb not null default '[]' | `[{buy_asset_id?, sell_asset_id?, markup}]`; markup fraction ≥ 0 |

Types: `models.PriceSourcePreference`, `models.PriceOffsets` (Valuer/Scanner like `PriceSourceList`), `PricingRegime.Validate()`.
Validation:
- no duplicate (buy, sell) keys
- buy ≠ sell
- asset ids exist (checked in the service)

### Source preference resolution (`at` = instant priced)
- For each entry in order, take that source's newest quote with `at − max_age_hours ≤ quote_time ≤ at`. Lookup order is direct, then inverted, then crossing (both legs from that source).
- The first entry that resolves wins. Never forward-looking.
- quote_time: STREAM = its instant; DAILY = 00:00 SGT of its date. So a same-day daily quote matches with 0h when `at` is that day's midnight (savings/manual), and is "N hours old" for an intraday `at` (exchange).
- `date_fallback` = the hit is not stamped on `at`'s SGT day.

### Offsets (both directions)
For a request base→quote:
- ask (customer buys base, pays quote) uses the entry for buy = base, sell = quote
- bid (customer sells base, receives quote) uses the entry for buy = quote, sell = base

Per side, the most specific entry wins: buy+sell > buy only > sell only > neither. No match = 0.
- ask' = ask × (1 + m_ask)
- bid' = bid / (1 + m_bid) — division, so XAU→SGD and SGD→XAU stay exact reciprocals of each other

Examples:
- `{buy: XAU, sell: SGD, markup: 0.025}` makes gold 2.5% dearer for SGD buyers whichever way the pair is asked.
- `{buy: XAG, markup: 0.03}` covers silver bought with anything.
- `{markup: 0.01}` is the fallback.

## 3. PriceService

- `FetchPriceForRegime(ctx, regime, base, quote, at)` → `*QuotePriceResult` with offsets applied. `Input` stays the market route. The result also carries `PricingRegimeID`, `AskMarkup` and `BidMarkup`.
- `FetchPriceForMerchant(ctx, merchantID, base, quote, at)` → uses `merchants.default_pricing_regime_id`.
- `FetchPriceForAccount(ctx, accountID, base, quote, at)` → `PricingRegimeForAccount` (today the merchant default; the hook for a later per-account override). Origin accounts error.
- `PricingRegimeForMerchant` / `PricingRegimeForAccount` are exported so a loop resolves the regime once.
- `FetchExecutionPrice(ctx, regime, base, quote, at)` and `RecordSuppliedExecutionPrice(ctx, regime, base, quote, bid, ask, ts)` replace the args+spread signatures.
- `FetchPrice(args)` stays: it's the internal primitive and serves the non-merchant callers. The `Source` filter is reused per preference entry; add a private `notBefore`.
- Delete `RecordedFetchArgs`, `executionPriceArgs`, `executionPriceLookupKey`, `savingsExchangeSpread`, `GetLiveLatestPrice`, `liveOrDBFetcher`.

## 4. `execution_prices`

- Add `pricing_regime_id uuid not null` FK, `as_of varchar(32)` (pinned stamp; NULL for live requests), and copies of the applied `ask_markup` and `bid_markup`.
- Drop `args`, `lookup_key`, `exchange_spread`. Index (pricing_regime_id, base_asset_id, quote_asset_id, as_of).
- Reuse the newest row on (regime, base, quote, as_of) when:
  - `created_at >= regime.updated_at`
  - its quote_prices rows are unchanged
  - the date-fallback re-check passes, as today
- Live requests never reuse.
- Migration deletes existing rows.
- Backoffice: list filter `pricing_regime_id`; response drops `args`/`exchange_spread` and adds the regime id and markups.

## 5. Feature tables

| Table | Change |
|---|---|
| `merchants` | add `default_pricing_regime_id` not null FK (same ledger) |
| `savings_plan_regimes` | add `pricing_regime_id` not null FK; drop `quote_price_source`, `reference_price_sources` |
| `storage_fee_regimes` | add `pricing_regime_id` not null FK; drop `price_source`; strip `exchange_spread` from `charge_rates` |
| `pair_price_offsets` | drop table, model, handler, entities, seed, route constant |

- On create/update, the chosen regime's `ledger_id` must equal the merchant's ledger.
- `StorageChargeRate.ExchangeSpread` and `ChargedPrice` are removed; the conversion markup comes from regime offsets.
- `GetByPriceSource(manual)` → regimes whose pricing regime's first source is `manual`.
- Manual pricing previously forced `PreventCrossing`. With the Manual regime's list being only `manual`, crossing can use only manual legs, so the route still points at manual rows. No per-entry crossing flag.

## 6. Call sites

Booking — feature's own regime:
- SavingsProcessor `collectPricesAndRun`, `ProcessSavingsPlanByIDWithOptions` → `FetchExecutionPrice(savingsRegime.PricingRegime, …, processingDate)`
- ManualPricingService `executionPrices` → same; supplied path → `RecordSuppliedExecutionPrice(regime, …)`
- StorageFeeAccrual `billableFee` → `FetchExecutionPrice(storageRegime.PricingRegime, …)`
- StorageFeeSettlement conversion (×2) and `metalSellQuote` (×2) → `FetchPriceForRegime(storageRegime.PricingRegime, …)`

Savings trigger (missed in earlier drafts): `savings_plan_execute` picks regimes by the ingested source.
- `job_executor`: `GetByPriceSource(source)`
- `job_creator`: `uniqueQuoteSources`, `hasDueSchedulesForSource` on `quote_price_source`

These become "savings regimes whose pricing regime's **first** preference entry is that source". Only the primary source triggers a run; later entries are fallbacks used during that run. The open question below asks what happens when the primary never arrives.

Account known → `FetchPriceForAccount`:
- exchange `GetExchangeQuote`, `processExchange`, `usdPerUnit`
- limit order `getCurrentExchangeRate`
- customer `wallet_handler.getDashboardData`, `WalletService.GetWalletBalanceHistory`, `CalculatePortfolioSummaryForWallets`
- customer `savings_handler_pricing.latestPair`

Merchant known → `FetchPriceForMerchant` (Account where the row has one):
- `account_detail_data_helpers.convertAssetToTargetCurrency`, `CustomerStatisticsService` (×3)
- `metrics_service.fetchSavingsPlanDailyPurchasePrice` (its 4-step fallback chain becomes the regime's preference list)
- `CurrentValue` / `ValueAtDate` take a regime: `wallet_current_value_handler`, backoffice `wallet_handler`, `metrics_service`
- `savings_stream_prices_handler.latestPair`, `etl_handler.simulateExecuteSavingsPlans`

Non-merchant, stays on plain `FetchPrice` (no regime, no offsets):
- `external_position_service` (×4), `wallet_funding_valuation`, `posting_meta_service.quoteAsk`
- `AssetAmountToUSD` (rough; any source, any age — `FetchPrice(nil)` as now)

## 7. Lifecycle

- `CreateLedgerWithOrigin` creates the "Default" and "Manual" regimes.
- Merchant create: `ensureDefaultPricingRegime` sets `default_pricing_regime_id` to the ledger's Default; the auto-created Default storage regime points at it.
- Merchant update accepts `default_pricing_regime_id`.
- Deleting a regime that is in use fails on the FK (RESTRICT) → 400 "in use".

## 8. Backoffice API

- `/api/pricing-regimes`: GET list (filter `ledger_id`, `name`), GET `/:id`, POST, PATCH `/:id`, DELETE `/:id`. No X-Merchant-Id.
  Any backoffice user can read; writes need `VerifySuperAdmin(db)`, same as `/api/ledgers`.
- `/api/price-sources`: GET + PATCH config.
- Savings/storage regime and merchant DTOs: `pricing_regime_id` replaces the source fields. Update swagger and `backoffice-api.md`.

## 9. Migration (single up/down pair)

Up:
0. Rename `integral_10am` → `gsc_10am_reference_price` in stored values (see section 1).
1. Create `price_source_configs`; insert one row per known source (incl. `integral_stream`) with mid, 0, 0.
2. `quote_prices_raw`: add the copied columns; relabel `resolution = 'STREAM' AND source = 'open_exchange_rates'` → `integral_stream`.
3. Create `pricing_regimes`. Per ledger, insert:
   - "Default": `[{gsc_10am_reference_price,72},{open_exchange_rates,168}]`, `offsets = []`
   - "Manual": `[{manual,0}]`, `offsets = []`
4. Point the existing rows at the new regimes:
   - `merchants.default_pricing_regime_id` → ledger Default
   - savings regimes with `quote_price_source = 'manual'` → ledger Manual; the others → Default
   - storage regimes → Default
   - Then set NOT NULL, drop the old source columns, and strip `charge_rates[].exchange_spread`.
5. Clear what references `execution_prices`: `exchanges.execution_price_id`, `storage_fee_accruals.execution_price_id`, and savings `prices[]`. Then `DELETE FROM execution_prices` and apply the column changes.
6. Drop `pair_price_offsets`.

Down: recreate the dropped columns/tables with defaults (`open_exchange_rates`, spread 0) and relabel `integral_stream` back. Data is not restored; the file says so.

## 10. Seeds, tests, docs

- Seeds: `seed_if_empty`, savings/storage regime seeds, pricing regime + source config seeds, drop `seed_pair_price_offsets`, `scripts/ops/seed`.
- Unit tests:
  - ingest spread for mid, bid and ask anchors; two-sided feeds untouched
  - preference order and max_age_hours boundaries (exactly at the limit, just past it); never forward-looking; daily vs intraday `at`
  - offsets: specificity per side, both directions (XAU→SGD vs SGD→XAU reciprocal), fallback-only
  - execution price reuse invalidated by regime `updated_at`
  - ledger mismatch rejected; account → merchant default
- Update savings/storage/manual-pricing/exchange/limit-order/execution-price tests.
- Docs: `docs/execution_prices.md`, `internal/services/price/README.md`, `backoffice-api.md`, new `docs/pricing_regimes.md`.
- `/verify`, `make check-migrations`, `make lint`.

## Open points (proposed defaults — will proceed with these unless told otherwise)

- Bid offset applied as `bid / (1 + markup)` (exact reciprocal of the ask in the other direction), not `bid × (1 − markup)`.
- Daily quote time = 00:00 SGT of its date.
- Ingest spreads apply to `gsc_10am_reference_price` too (single-price feed), even though its config starts at 0/0.
- Savings runs trigger only on ingest of the pricing regime's first source. If it never arrives (e.g. a GSC outage), nothing runs and the fallback entries are never reached. Options:
  - accept that and run manually
  - also trigger on fallback sources after a cutoff time (needs a scheduled check)

  Proposed: accept it for this PR.
- Keep the `gsc_integral_fetch` package and worker name; only the source value is renamed.

## Review
