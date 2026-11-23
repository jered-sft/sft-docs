# Price streams: stream ingest workers + backoffice regime streams

Status: plan, questions answered 2026-11-01 (see "Decisions"); awaiting go-ahead.

## Decisions (2026-11-01)
1. `NormalizedPrice` gets optional `Bid`/`Ask`.
2. Regime streams: backoffice /ws only, any authenticated backoffice user.
3. Keep `price:live` + `pricer:live:instrument:*` (legacy LivePrice shape, incl. daily stats) alongside `stream:{source}:ticks` for now.
4. Workers store only the pairs the feed sends; regimes do all crossing (no XAU/SGD derivation in workers).
5. `mt5_stream` source, polled from exc. Hard-coded markups in `exc.go` removed; spreads/markups come from
   PriceSourceConfig + PricingRegime, configured manually (no seed migration).
6. Stream ingest events go on their own channel `price_source_stream`; `price_source` stays DAILY/HOURLY/manual.
7. Redis-first lookup only when the Redis quote is inside `[notBefore, notAfter]`; otherwise DB (HOURLY rows).
8. Hourly cron saves only if the Redis raw is at most 1h old (skips feed-down and weekends naturally).
9. Latest-raw API: stream sources Redis then DB fallback; non-stream sources DB only.
10. Nodes re-run `FetchPriceForRegime` per relevant event (no frozen route, no `RepriceRoute`); timed, `processing_ms` in every message.
11. Redis lease lock per stream ingest worker, safe against dead holders (see 4a).
12. Nodes torn down when their demand heartbeat key expires (short TTL).

Consequence of 4: any regime pricing metal/SGD from `integral_stream`/`mt5_stream` needs `max_crosses >= 1` with USD as a
crossing asset — check existing regimes' config before deploy (manual config, like 5).
Supersedes "Pricing regime price stream (backoffice websocket)" further down:
nodes now live in a worker, not the backoffice-api process.

## Findings from the code review
- `integral_price_fetch` and `stream_price_fetch` are not registered in `server/workers/worker.go`
  (dropped in 64832991). Nothing publishes `pricer:live:*` today, so `price:live` on /ws is silent.
- `stream_price_fetch` (MT5) is a 1-minute poll of the exc DB (`resources.ExcResource`), not a websocket,
  and writes its rows under source `integral_stream`. There is no MT5 price source.
- `IngestQuote` takes `NormalizedPrice{MidRate}` and derives bid/ask from `PriceSourceConfig` spreads.
  Stream ticks are two-sided; `IngestStreamQuotes` exists to keep their bid/ask.
- `IngestQuote` opens a transaction and reloads the source config + all assets on every call.
  Per tick (~5 sockets x 1/s) that has to be cached or skipped for STREAM.
- `publishIngestSuccess` currently suppresses Redis fan-out for STREAM (writes event_logs only).
  The new design inverts that: Redis yes, event_logs no.
- `PricingRegimeWorker` subscribes to every `quote_price_ingest_success` and drops non-DAILY in CreateJobs.
- `PriceService` has no Redis client; `NewPriceService(db)` is built in many places.
- `whereQuoteTimeWithin` / `FormatResolutionTimestamp` / `ReadyQuoteRaw` know only DAILY and STREAM.
- No function re-evaluates a `QuotePriceInput` tree with new leaf prices.
- wshub `broadcast` sends every pattern message to every client; only the chart manager routes per client.
- `livePricePair` (`services/price/live.go`) has no callers.
- `pkg/integral` (HTTP) is still used by `livemarket_handler` (candlestick) and `IntegralLiveResource.DailyStats`.

## Plan

### 1. Cleanup and merge
- [ ] Delete `internal/workers/integral_price_fetch`, `internal/workers/stream_price_fetch`, `internal/services/price/stream.go` (+ tests).
- [ ] Delete what only they used: `resources.IntegralStreamResource` (`resources/integral.go`, builder at `resources.go:178`),
      `PriceIngestService.IngestStreamQuotes`, `StreamPriceRepository` if unused, `livePricePair`.
- [ ] Merge `pkg/pricesocket` into `pkg/integral`: `client.go` (HTTP) + `socket.go` (ws `Client`, `TickData`, `GetURLs`, `ParseTickTime`).
- [ ] `LivePriceStreamer`: keep its legacy publishing (instrument keys, `pricer:live:ticks:*`, DailyStats refresh) but drive it from
      the Integral ingest worker's tick loop instead of its own sockets (one socket set, two sinks).
- [ ] `exc.go`: drop the hard-coded markups and the SGD-metal derivation; emit raw XAU/XAG/XPT-USD + USD/SGD bid/ask.

### 2. Models
- [ ] `PriceResolutionHourly = "HOURLY"`; period/timestamp in stream layout truncated to the hour.
      Teach `FormatResolutionTimestamp` and `whereQuoteTimeWithin` (HOURLY joins the instant branch).
- [ ] `PriceSourceMT5Stream = "mt5_stream"` (Q5); `IsStreamPriceSource` covers both; add to `KnownPriceSources` + config seed.
- [ ] `NormalizedPrice` gains optional `Bid`/`Ask`; `quotePricesFromPairs` uses them when set, else spreads (Q1).
- [ ] Redis key/channel helpers in one place: `stream:{source}:ticks`, `stream:{source}:quote_price_raw`,
      `stream:{source}:quote_prices`, plus the regime request/price channels.

### 3. `PriceIngestService.IngestQuote` STREAM branch
- [ ] `Resolution == STREAM`: no transaction. Config + ticker IDs from a short-TTL in-memory cache; build rows as today; then
      - `SET stream:{source}:quote_price_raw` = JSON raw (fresh uuid so events can reference it),
      - `HSET stream:{source}:quote_prices {base_id}/{quote_id}` = JSON QuotePrice (one field per pair),
      - publish `quote_price_ingest_success` with `SkipDB: true`.
- [ ] Service needs a Redis client for this; error when missing.
- [ ] STREAM events publish on `enum.ChannelPriceSourceStream = "price_source_stream"`; PricingRegimeWorker untouched.

### 4a. Redis lease (`pkg/redis/lease.go`)
- [ ] `Lease{key, token, ttl}`: acquire = `SET key token NX PX ttl`; renew and release are Lua compare-token scripts
      (only the holder can extend or delete). TTL 15s, renew every 5s.
- [ ] Holder that fails to renew (lost token or Redis error past TTL) cancels its stream context immediately, so two pods never ingest at once.
- [ ] Standbys retry acquire every 5s. A dead pod's lock expires within 15s; a clean shutdown releases at once.
- [ ] `MemoryClient` support for tests; table tests for acquire/renew/steal-after-expiry/release-by-non-owner.

### 4. `IntegralStreamIngestWorker` (`internal/workers/integral_stream_ingest`)
- [ ] `Initialize`: honour `INTEGRAL_PRICE_STREAM_DISABLED`; start a goroutine that holds the lease `lock:stream:integral_stream`
      and, while held, runs per-symbol socket loops with reconnect + Telegram alert
      (lifted from `IntegralLiveResource.Stream` / `LivePriceStreamer.onStreamFailure`).
- [ ] Per tick: publish `ProcessEvent{Channel: "stream:integral_stream:ticks", Event: "price_stream_tick", Raw: tick, SkipDB: true}`;
      merge into the latest-per-symbol snapshot; build a STREAM `QuotePriceRaw` (RawData = snapshot, Normalized = pairs in
      storage units); `IngestQuote`.
- [ ] Cron `0 * * * *`: read `stream:integral_stream:quote_price_raw`; skip (`workers.Skipped("stale_stream_price")`) if older than 1h;
      else clear ID, Resolution HOURLY + hour period, `IngestQuote` (DB path, event_logs written). Idempotent via the
      (period, source) unique index. Runs on any pod (reads Redis only), no lease needed.
- [ ] Register in `server/workers/worker.go`.

### 5. `Mt5StreamIngestWorker` (`internal/workers/mt5_stream_ingest`)
- [ ] Same shape; lease `lock:stream:mt5_stream`; while held, poll `ExcResource` every few seconds for snapshots newer than the
      last seen (seeded from the Redis raw's timestamp) feeding the same per-tick path.
- [ ] Share tick→raw→ingest + hourly cron with step 4 in a small helper package.

### 6. `PriceService.queryFetchedPair` reads Redis first
- [ ] `PriceService.WithRedis(client)`; ServiceBuilder wires it.
- [ ] When `f.source` is a stream source and the resolution filter is empty/STREAM: `HGET stream:{source}:quote_prices`;
      use it only if its time is within `[notBefore, notAfter]` (Q7), else fall through to SQL. `QuotePriceID` nil, Source set.

### 7. Regime streams (websocket → worker)
- [ ] wshub: inbound `subscribe:price_regime` / `unsubscribe:price_regime` `{price_regime_id, base_asset_id, quote_asset_id}`.
      Hub tracks subscribers per key, publishes the request to Redis and keeps a heartbeat key with TTL (the `chart:params:` pattern);
      drops it on last unsubscribe/disconnect.
- [ ] Hub PSUBSCRIBEs the regime price channel and sends only to that key's subscribers. Wired from the backoffice /ws handler
      only (customer /ws ignores the message type).
- [ ] `StreamingPriceRegimeWorker` (`internal/workers/streaming_price_regime`): `Initialize` subscribes to the request channel,
      `price_source` and `price_source_stream`; holds `map[key]*PricingRegimeNode`, key `{price_regime_id}:{base_asset_id}/{quote_asset_id}`.
      Every worker pod may run nodes (output is idempotent); a pod only creates a node on request, so no lease.
- [ ] `PricingRegimeNode`: create → price → publish first price `SkipDB: false`. On each ingest event whose source is in the
      regime's preference (or the regime has an any-source entry) → mark dirty; a per-node loop re-prices at most once per
      ~250ms (coalesces a tick burst across sources). Price = reload regime (pk lookup, so edits are seen) +
      `FetchPriceForRegime(now)`; publish if bid/ask changed, `SkipDB: true`.
- [ ] Each message: `{price_regime_id, base_asset_id, quote_asset_id, bid, ask, mid, timestamp, sources, calculation_type,
      input, processing_ms}`; `processing_ms` = regime reload + FetchPriceForRegime wall time. A miss publishes an error message
      (`PriceNotFoundError.Tried`) rather than going silent.
- [ ] Teardown: hub heartbeat key TTL 30s, refreshed every 10s; node loop checks it each 10s and stops when gone.

### 8. Price source APIs (backoffice)
- [ ] `GET /api/price-sources/{source}/latest-raw`: stream source → Redis raw, else newest DB `QuotePriceRaw`; non-stream → DB.
- [ ] ws `subscribe:price_source {source}` → forwards `stream:{source}:ticks` (stream) or ingest events (others).
- [ ] Swagger regenerated.

### 9. Verify
- [ ] Unit: STREAM ingest branch (miniredis), Redis-first lookup incl. age bounds, lease, node coalescing + teardown,
      hourly cron (fresh / stale / duplicate), hub routing. `/verify`, `make lint`.
- [x] Measure `FetchPriceForRegime` against the dev DB for a crossed metal/SGD route with stream sources (Redis hit) and with a ✅ 2026-11-23
      DAILY fallback (DB). If p95 is well above ~20ms, lower the coalesce rate or cache the regime + holiday lookup.
- [x] Manual: worker + backoffice running, subscribe from a ws client, watch ticks and regime prices. ✅ 2026-11-23

## Open questions (answered 2026-11-01, see Decisions)
1. Bid/ask through `IngestQuote`: optional Bid/Ask on `NormalizedPrice` (my pick), or mid-only + source spreads?
2. Who may subscribe to regime streams: backoffice /ws only, or customers too? Gate on `super_backoffice`?
3. `price:live` FE event + `pricer:live:instrument:*` keys (LivePrice with high/low/yesterday close): moving to
   `stream:{source}:ticks` with a ProcessEvent payload breaks that contract. Drop it, or keep the old shape until FEs move?
   Still need the DailyStats refresh?
4. Pairs per tick: keep deriving XAU/SGD etc. from XAU/USD x USD/SGD in the worker, or store only what the feed sends and let regimes cross?
5. MT5 has no websocket: is polling the exc DB every N s acceptable, under a new `mt5_stream` source? Hard-coded markups in
   `exc.go` stay, or move to source-config spreads?
6. STREAM ingest events now hit Redis on `price_source` every tick. Fine there (regime worker filters), or a separate channel?
7. Redis-first lookup with a past `BeforeDatetime` (executions priced at a period): Redis value used only inside the window,
   so historical lookups keep hitting the DB (HOURLY rows) — confirm.
8. Hourly cron: skip when the Redis raw is older than ~1h (feed down / weekend)? Run on weekends?
9. Latest-raw API for non-stream sources (OER, GSC, manual): newest DB row, or stream sources only?
10. Node route frozen at creation (as specified): a fallback-source route never switches back, max ages aren't re-checked,
    regime edits aren't seen. Accept, or re-run `FetchPriceForRegime` on each relevant event (simpler, always correct, one DB query
    per event per node)?
11. Worker replicas: with more than one worker pod each opens the sockets and ingests every tick. Single replica, or a Redis lock?
12. Node teardown by heartbeat-key TTL (like charts) OK? Cap on live nodes?
