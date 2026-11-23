# Self Healing Schedules

## Objective
When this plan is done, the workers will meet two goals.

### 1. Batch workers can recover any missed work
A batch worker processes many customer items in one cycle, such as storage fees or savings plans. Each of these workers can recover on its own: it looks for the individual items that were never processed and processes them. The work runs in two phases.

1. **Create the execution.** A `{WalletSnapshot,StorageFee,SavingsPlan}Execution` row means a period's customer cycles are ready to run. The row holds everything those cycles need. That way, every cycle in an execution uses the same inputs. For example, every savings plan for a given day is guaranteed to use the same price.
2. **Process the items.** A separate worker looks for the individual items that still need to run. It finds them from the execution row, so the execution's presence is the only signal it needs.

The phases never share in-memory state. If a run dies partway through, the execution row is still there, and phase 2 can work out exactly which items are missing.

### 2. Every worker runs in realtime, with a cron fallback
Each worker reacts to events within seconds. A cron catches any messages that were dropped.

## Triggers
Every worker has two triggers:

- **Realtime** — a Postgres `NOTIFY` on `application_events` (or the in-process `quote_price_ingest_success` event) does the work within seconds.
- **Cron** — every 15 min, a set-based query over the last 30 days finds anything the realtime path missed (dropped listener, worker restart, failed job) and dispatches it again. On first deploy the cron also does the initial backfill.

Both paths dispatch the same job, and the job must be idempotent: running it twice, or concurrently, must have no extra effect. Correctness comes from the unique indexes and row claims below. Asynq task IDs only reduce duplicate work.

## Period and resolution
A **period** is the window of time that a row belongs to:
- a price aggregates a period;
- a schedule occupies a period;
- an execution joins the two for one regime and period.

Every one of these tables calls the column `period` and stores the same string format, so they join by plain string equality. The width of the window is the **resolution** (`DAILY`, later `HOURLY`), which `quote_prices_raw.resolution` already records.

| Resolution | Period format | Example |
|---|---|---|
| `DAILY` | `YYYY-MM-DD` (Singapore business date) | `2026-09-14` |
| `HOURLY` (future) | `YYYY-MM-DDTHH` | `2026-09-14T10` |

Each resolution has exactly one formatter in `models` (replacing `FormatDailyTimestamp`). The price ingest, the schedule generator and the execution creators all use it, and nothing builds a period string by hand. If two of them formatted a period differently, executions and schedules would silently stop matching.

Savings plans may later run hourly. Because everything keys on the period string rather than a date, that change only adds a format. Storage fees stay `DAILY`.

## Schema changes

### 1. Rename to `period`, one automatic execution per regime per period
Today the same concept has three names: `quote_prices_raw.timestamp`, `savings_plan_schedules.schedule_period` and `*.idempotency_key`. The executions' `idempotency_key` is also a `date` that is unique on its own. That breaks in two ways: a second regime's execution for the same day fails, and a date can't identify an hourly run.

All four columns become `period varchar(32)`. Execution periods are unique per regime, so re-running the same regime and period always lands on the same row, whatever the cadence. Manual runs keep `period = NULL` and stay unrestricted, as they do today. Their `processing_date` still records the day.

```sql
-- Prices
ALTER TABLE quote_prices_raw RENAME COLUMN "timestamp" TO period;
ALTER INDEX uni_quote_prices_raw_ts_source RENAME TO uni_quote_prices_raw_period_source;
ALTER INDEX idx_quote_prices_raw_timestamp RENAME TO idx_quote_prices_raw_period;

-- Schedules
ALTER TABLE savings_plan_schedules RENAME COLUMN schedule_period TO period;
ALTER TABLE savings_plan_schedules ALTER COLUMN period TYPE varchar(32);
ALTER INDEX idx_savings_plan_schedules_schedule_period RENAME TO idx_savings_plan_schedules_period;
-- uidx_savings_plan_schedule_period (savings_plan_id, period) already has the right name.

-- Savings plan executions
DROP INDEX idx_savings_plan_execution_idempotency_key;
ALTER TABLE savings_plan_execution RENAME COLUMN idempotency_key TO period;
ALTER TABLE savings_plan_execution
  ALTER COLUMN period TYPE varchar(32) USING to_char(period, 'YYYY-MM-DD');
CREATE UNIQUE INDEX uidx_savings_plan_execution_regime_period
  ON savings_plan_execution (savings_plan_regime_id, period);

-- Storage fee executions
DROP INDEX idx_storage_fee_accrual_executions_idempotency_key;
ALTER TABLE storage_fee_accrual_executions RENAME COLUMN idempotency_key TO period;
ALTER TABLE storage_fee_accrual_executions
  ALTER COLUMN period TYPE varchar(32) USING to_char(period, 'YYYY-MM-DD');
CREATE UNIQUE INDEX uidx_storage_fee_accrual_executions_regime_period
  ON storage_fee_accrual_executions (storage_fee_regime_id, period);
```
The down migration reverses each rename and casts `period` back to `date` for the two execution tables.

Code to update:
- Models and entities:
  - `QuotePriceRaw.Timestamp` → `Period`;
  - `SavingsPlanSchedule.SchedulePeriod` → `Period`;
  - `SavingsPlanExecution.IdempotencyKey` and `StorageFeeAccrualExecution.IdempotencyKey` → `Period *string`.
  
  Update the JSON tags too. That changes the backoffice and customer API fields, so regenerate swagger and tell `gbnx-admin-fe`.
- `AutomaticIdempotencyKey` and both `GetByIdempotencyKey` repository methods → `GetByPeriod(regimeID, period)`. Lookups must include the regime now that the key is unique only per regime.
- The `FormatDailyTimestamp(*execution.IdempotencyKey)` calls in `job_creator.go` and `manual_pricing_service.go` become `*execution.Period`.
- The backoffice handlers that clear the key for manual runs (`savings_plan_execution_handler.go`, `storage_fee_accrual_execution_handler.go`) set `Period = nil`.
- The price ingest, the `quote_price_raw_handler.go` filters, and the event payload of `quote_price_ingest_success`.

**Not renamed here:** `storage_fee_accruals.timestamp`, and `wallet_snapshots.scope` + `target`. They are the same concept (`target` is the period, `scope` the resolution), but they sit outside this flow. Rename them in a follow-up.

### 2. One accrual per wallet per execution
```sql
CREATE UNIQUE INDEX uidx_storage_fee_accruals_wallet_execution
  ON storage_fee_accruals (wallet_id, storage_fee_accrual_execution_id)
  WHERE type = 'accrual';
```
The accrual job inserts with `ON CONFLICT DO NOTHING`.

### 3. Drop `savings_plan_regimes.reference_price_sources`
It is unused. Remove the column, `SavingsPlanRegime.ReferencePriceSources`, and its uses in the regime service, the seeds, tests and swagger.

### 4. Exchanges reference only the schedule
Add `exchanges.savings_plan_schedule_id` and backfill it where a schedule can be matched. Then drop `savings_plan_id` and `savings_plan_execution_id`.

```sql
ALTER TABLE exchanges ADD COLUMN savings_plan_schedule_id uuid REFERENCES savings_plan_schedules(id);
CREATE UNIQUE INDEX uidx_exchanges_savings_plan_schedule_id
  ON exchanges (savings_plan_schedule_id) WHERE savings_plan_schedule_id IS NOT NULL;

UPDATE exchanges e
SET savings_plan_schedule_id = sps.id
FROM savings_plan_schedules sps
WHERE sps.savings_plan_id = e.savings_plan_id
  AND sps.savings_plan_execution_id = e.savings_plan_execution_id
  AND e.savings_plan_schedule_id IS NULL;

ALTER TABLE exchanges DROP COLUMN savings_plan_id, DROP COLUMN savings_plan_execution_id;
```
Exchanges that predate schedules keep `savings_plan_schedule_id = NULL`. Before the drop, count the rows that have `savings_plan_id` set but no schedule match, and record the number in the PR. The down migration restores the columns and fills them back from the schedule.

Code to update: `models.Exchange`, `exchange_record.go`, and the `savings_plan_id` filters in `exchange_handler.go` and `exchange_query_service.go`. The filter becomes a join through `savings_plan_schedules`.

The unique index also makes the schedule job idempotent: a schedule can never produce two exchanges.

### 5. Insert notify triggers
Same pattern as `20261005000000_transactions_insert_notify`:

```sql
CREATE OR REPLACE FUNCTION "notify_savings_plan_execution_insert"() RETURNS trigger AS $$
BEGIN
  PERFORM pg_notify('application_events', json_build_object(
    'event', 'created',
    'channel', 'savings_plan_execution',
    'target_object', 'savings_plan_execution',
    'target_id', NEW.id
  )::text);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER "trg_savings_plan_execution_insert_notify"
  AFTER INSERT ON "savings_plan_execution"
  FOR EACH ROW EXECUTE FUNCTION "notify_savings_plan_execution_insert"();
```
Add the same trigger for `storage_fee_accrual_executions` (channel `storage_fee_accrual_executions`, target object `storage_fee_accrual_execution`). Add `PgEventChannelSavingsPlanExecution` and `PgEventChannelStorageFeeAccrualExecutions` to `internal/enum/pubsub.go`. `NOTIFY` is delivered on commit, so listeners always see the row.

`wallet_snapshots` also gets an insert trigger. It is statement-level, not per row, and the [StorageFeeAccrualWorker](#realtime-wallet-snapshot-created) section shows it.

### 6. Schedule processing status
Add the `processing` status (`SavingsPlanScheduleStatusProcessing`) and a column recording when processing started:
```sql
ALTER TABLE savings_plan_schedules ADD COLUMN processing_started_at timestamptz;
CREATE INDEX idx_savings_plan_schedules_processing
  ON savings_plan_schedules (processing_started_at) WHERE status = 'processing';
```
`status` is `varchar(20)` with no check constraint, so `processing` needs no change to the column.

## SavingsPlanExecutionWorker
Creates one automatic `savings_plan_execution` row per regime per period, once that period's quote price exists. The price defines the period, so the worker works the same whether prices arrive daily or hourly.

### Realtime
1. Listen to `quote_price_ingest_success`.
2. Select the regimes where `quote_price_source = quotePrice.Source`.
3. For each regime, dispatch a job that creates the execution for that price. The job sets:
   - `period = quotePrice.Period`;
   - `processing_date` = the period's date;
   - `savings_plan_regime_id`, `quote_price_raw_id` and `prices`;
   - `status = 'pending'`.

   It inserts with `ON CONFLICT (savings_plan_regime_id, period) DO NOTHING`.

### Cron
Every 15 min, find the prices from the last 30 days that have no automatic execution for their regime. The query walks the price rows instead of a list of days, so it handles any resolution:
```sql
SELECT r.id AS savings_plan_regime_id, q.period, q.id AS quote_price_raw_id
FROM savings_plan_regimes r
JOIN quote_prices_raw q
  ON q.source = r.quote_price_source
 AND q.resolution = 'DAILY'
 AND q.created_at >= now() - interval '30 days'
WHERE NOT EXISTS (
    SELECT 1 FROM savings_plan_execution e
    WHERE e.savings_plan_regime_id = r.id
      AND e.period = q.period
  );
```
Dispatch the same create job for each row. When hourly regimes arrive, give the regime its own `resolution` and match `q.resolution` against it instead of the hard-coded `'DAILY'`, so a regime never runs on prices at another resolution.

Savings plans run only for periods that have their own price. There is no carry-forward here.

## SavingsPlanScheduleWorker
Executes `scheduled` schedules once the execution for their regime and period exists.

Schedules have no regime of their own. The regime comes from `savings_plans.savings_plan_regime_id`.

### Claiming
Executing a schedule spans more than one transaction, so the job claims schedules up front:
```sql
UPDATE savings_plan_schedules
SET status = 'processing', processing_started_at = now()
WHERE id = ANY(:ids) AND status = 'scheduled'
RETURNING id;
```
Only the returned ids are executed. Any id not returned was already claimed by another job, or is no longer `scheduled`, and is skipped. When a schedule finishes, the job sets `status` to `executed`, `failed` or `skipped`, and clears `processing_started_at`.

### Stuck-schedule cron
Every 5 min, put stuck schedules back to `scheduled` so the next dispatch picks them up again:
```sql
UPDATE savings_plan_schedules
SET status = 'scheduled', processing_started_at = NULL
WHERE status = 'processing'
  AND processing_started_at < now() - interval '30 minutes'
RETURNING id;
```
A retried schedule is safe even if the first attempt wrote its exchange before dying. Before booking, the job looks for an exchange with that `savings_plan_schedule_id`. If one exists, the job marks the schedule `executed` without booking again. The unique index on `exchanges.savings_plan_schedule_id` backs this up if two attempts race. The 30-minute timeout must be longer than the job's Asynq timeout, so that a slow job that is still running is never re-queued.

### Realtime
1. Listen to `{ channel: "savings_plan_execution", event: "created" }`.
2. Find that execution's due schedules:
```sql
SELECT sps.id
FROM savings_plan_execution spe
JOIN savings_plans sp ON sp.savings_plan_regime_id = spe.savings_plan_regime_id
JOIN savings_plan_schedules sps
  ON sps.savings_plan_id = sp.id
 AND sps.period = spe.period
WHERE spe.id = :event_target_id
  AND spe.period IS NOT NULL
  AND sps.status = 'scheduled'
  AND sps.scheduled_at <= now()
  AND sps.scheduled_at >= now() - interval '30 days';
```
3. Dispatch in chunks of 100. Task ID = `spe.id` + chunk index.

Schedules whose `scheduled_at` is still in the future when the execution is created are picked up by the cron after their cutoff.

### Cron
Every 15 min:
```sql
SELECT sps.id, spe.id AS savings_plan_execution_id
FROM savings_plan_schedules sps
JOIN savings_plans sp ON sp.id = sps.savings_plan_id
JOIN savings_plan_execution spe
  ON spe.savings_plan_regime_id = sp.savings_plan_regime_id
 AND spe.period = sps.period
WHERE sps.status = 'scheduled'
  AND sps.scheduled_at <= now()
  AND sps.scheduled_at >= now() - interval '30 days';
```
Dispatch in chunks of 100, grouped by `savings_plan_execution_id`.

## StorageFeeExecutionWorker
Creates one automatic `storage_fee_accrual_executions` row per regime per `DAILY` period.

### Price for a period
The price for period D is the latest `quote_prices_raw` row for the regime's `price_source` with `period <= D`, looking back at most 7 days. The execution keeps `period = D` and points `quote_price_raw_id` (or its `execution_price`) at that price. Whether a carry-forward day computes a fresh fee or copies the previous one is still governed by `StorageFeeRegime.RunOnWeekends`.

Public holidays are handled the same way as Saturday and Sunday. There is no holiday calendar: a holiday is simply a weekday with no price of its own. D has its own price when a `quote_prices_raw` row has `period = D`. Otherwise D is a carry-forward day, and its execution is created only when:
- D is a Saturday or Sunday, or
- a price with a later period than D exists for the source. That proves D was skipped (a holiday), not that its price is late.

Until then, a weekday with no price waits. This stops the cron from running today's accrual on yesterday's price before today's ingest arrives.

### Realtime
1. Listen to `quote_price_ingest_success`.
2. Select the regimes where `price_source = quotePrice.Source`.
3. Dispatch a create job for each regime with `period = quotePrice.Period`, inserting with `ON CONFLICT (storage_fee_regime_id, period) DO NOTHING`.

The realtime path only creates executions for periods that have their own price. Weekend and holiday executions are created by the cron, up to 15 min after the day becomes due.

### Cron
Every 15 min:
```sql
SELECT r.id AS storage_fee_regime_id, to_char(d.day, 'YYYY-MM-DD') AS period, q.id AS quote_price_raw_id
FROM generate_series(current_date - 30, current_date, interval '1 day') AS d(day)
CROSS JOIN storage_fee_regimes r
CROSS JOIN LATERAL (
  SELECT q.id, q.period
  FROM quote_prices_raw q
  WHERE q.source = r.price_source
    AND q.resolution = 'DAILY'
    AND q.period <= to_char(d.day, 'YYYY-MM-DD')
    AND q.period >= to_char(d.day - interval '7 days', 'YYYY-MM-DD')
  ORDER BY q.period DESC
  LIMIT 1
) q
WHERE (
    q.period = to_char(d.day, 'YYYY-MM-DD')             -- own price
    OR extract(isodow FROM d.day) IN (6, 7)              -- weekend
    OR EXISTS (                                          -- holiday: a later price exists
      SELECT 1 FROM quote_prices_raw later
      WHERE later.source = r.price_source
        AND later.resolution = 'DAILY'
        AND later.period > to_char(d.day, 'YYYY-MM-DD')
    )
  )
  AND NOT EXISTS (
    SELECT 1 FROM storage_fee_accrual_executions e
    WHERE e.storage_fee_regime_id = r.id
      AND e.period = to_char(d.day, 'YYYY-MM-DD')
  );
```
Dispatch the same create job for each row.

## StorageFeeAccrualWorker
Creates one `accrual` row per charged wallet per execution.

A wallet is charged by an execution when:
- its account's regime matches: `COALESCE(accounts.storage_fee_regime_id, merchants.default_storage_fee_regime_id)`;
- its asset appears in the regime's `charge_rates`;
- it existed on the execution date;
- its EOD wallet snapshot for that period exists (`wallet_snapshots.scope = 'EOD'`, `target = period`), unless the day carries forward the previous business day's fee. A wallet without a snapshot yet is skipped and picked up by a later cron run once `wallet_snapshot_success` has fired.

### Realtime
1. Listen to `{ channel: "storage_fee_accrual_executions", event: "created" }`.
2. Find the wallets still owed an accrual:
```sql
SELECT w.id
FROM storage_fee_accrual_executions sfe
JOIN storage_fee_regimes r ON r.id = sfe.storage_fee_regime_id
JOIN accounts a ON COALESCE(a.storage_fee_regime_id,
                            (SELECT m.default_storage_fee_regime_id FROM merchants m WHERE m.id = a.merchant_id))
                   = sfe.storage_fee_regime_id
JOIN wallets w ON w.account_id = a.id
WHERE sfe.id = :event_target_id
  AND w.created_at::date <= sfe.processing_date
  AND EXISTS (
    SELECT 1 FROM jsonb_array_elements(r.charge_rates) cr
    WHERE (cr->>'asset_id')::uuid = w.asset_id
  )
  AND NOT EXISTS (
    SELECT 1 FROM storage_fee_accruals sfa
    WHERE sfa.wallet_id = w.id
      AND sfa.storage_fee_accrual_execution_id = sfe.id
      AND sfa.type = 'accrual'
  );
```
3. Dispatch in chunks of 100. Task ID = `sfe.id` + chunk index.

### Cron
Every 15 min, run the same query for every automatic execution in the last 30 days: replace `sfe.id = :event_target_id` with `sfe.period >= to_char(current_date - 30, 'YYYY-MM-DD')`, and also select `sfe.id`. Dispatch in chunks of 100, grouped by `storage_fee_accrual_execution_id`.

### Realtime: wallet snapshot created
A wallet's accrual can't run until its EOD snapshot exists, so a wallet with no snapshot is skipped. The worker reacts to each individual snapshot row as it's written, not only to a whole snapshot run finishing.

This matters when one wallet's snapshot failed days ago. Later a code release or an external fix writes the missing snapshot, and that wallet's storage fee should be created at once, without waiting for the cron or a full snapshot rerun.

#### Trigger
The snapshot service upserts rows in bulk (`INSERT … ON CONFLICT (wallet_id, scope, target) DO UPDATE`). A row-level `NOTIFY` would send one event per wallet, thousands for a normal day's run. Instead, a statement-level trigger reads the inserted rows from a transition table. It sends one event per EOD period in the statement, listing that statement's wallet ids:

```sql
CREATE OR REPLACE FUNCTION "notify_wallet_snapshots_insert"() RETURNS trigger AS $$
DECLARE
  r record;
BEGIN
  FOR r IN
    SELECT target, array_agg(wallet_id) AS wallet_ids
    FROM new_rows
    WHERE scope = 'EOD'
    GROUP BY target
  LOOP
    PERFORM pg_notify('application_events', json_build_object(
      'event', 'created',
      'channel', 'wallet_snapshots',
      'target_object', 'wallet_snapshot',
      'instance', r.target,
      'wallet_ids', CASE WHEN cardinality(r.wallet_ids) <= 100 THEN r.wallet_ids END
    )::text);
  END LOOP;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER "trg_wallet_snapshots_insert_notify"
  AFTER INSERT ON "wallet_snapshots"
  REFERENCING NEW TABLE AS new_rows
  FOR EACH STATEMENT EXECUTE FUNCTION "notify_wallet_snapshots_insert"();
```

- **Only inserts fire.** For an upsert, the transition table holds only the rows that were actually inserted, not the conflicting rows that were updated. A rewritten snapshot for a wallet that already has an accrual has nothing to do.
- **Large statements drop the wallet list.** `pg_notify` payloads are capped at 8000 bytes. Above 100 wallets, `wallet_ids` is `NULL` and the listener processes the whole period. That is the normal daily run.

Add `PgEventChannelWalletSnapshots` to `internal/enum/pubsub.go`.

#### Handling
1. Listen to `{ channel: "wallet_snapshots", event: "created" }`.
2. Find the automatic executions for that period: `SELECT id FROM storage_fee_accrual_executions WHERE period = :instance`. If there are none, do nothing. The execution's own `created` event will pick the wallets up later.
3. For each execution, run the realtime wallet query above with `sfe.id` set to that execution. When `wallet_ids` is present, also add `AND w.id = ANY(:wallet_ids)`. The query's `NOT EXISTS` still drops wallets that already have an accrual.
4. Dispatch in chunks of 100. Task ID = `sfe.id` + chunk's first wallet id. The accrual job's `ON CONFLICT DO NOTHING` absorbs any overlap with the execution's own dispatch.

This replaces listening to `wallet_snapshot_success`. Each snapshot statement fires its own event, so a whole-run event adds nothing.

**Carry-forward days.** A weekend or holiday accrual doesn't need its own snapshot, because it bills the previous business day's balance. When a missing snapshot is fixed for a Friday, though, the Saturday and Sunday accruals for that wallet were skipped too (`PreviousStorageFeeBalance` found nothing). Step 2 should therefore also take any carry-forward executions after `:instance`, up to and not including the next period that has its own price. Process them in date order, so that each day's accrual exists before the next day looks it up.

### Realtime: price ingested, billable backfill
When an accrual is converted into another asset (`accumulate_as` is `normalized` or `fee_wallet_asset`), the job needs a price. If no price is available yet, the job still saves the accrual, but with `billable_amount`, `billable_outstanding` and `execution_price_id` all `NULL`. This step fills them in once the price arrives.

1. Listen to `quote_price_ingest_success`.
2. Find the unpriced accruals that the new price can price:
```sql
SELECT sfa.id
FROM storage_fee_accruals sfa
JOIN storage_fee_accrual_executions sfe ON sfe.id = sfa.storage_fee_accrual_execution_id
JOIN storage_fee_regimes r ON r.id = sfe.storage_fee_regime_id
WHERE r.price_source = :quote_price_source
  AND sfa.type = 'accrual'
  AND sfa.billable_amount IS NULL
  AND sfa.settled_by_id IS NULL
  AND sfa.timestamp >= :quote_price_period
  AND sfa.timestamp >= to_char(current_date - 30, 'YYYY-MM-DD');
```
   `timestamp >= period` is used because the price lookup falls back to earlier prices. A late price for day D can therefore also price an unpriced accrual from D+1.
3. Dispatch in chunks of 100. For each accrual, the job calls the same `billableFee` lookup the accrual job uses, at the accrual's own date. If it gets a price, the job sets `billable_amount` and `execution_price_id`. Rows that still have no price stay `NULL` for the next ingest or the cron.

`billable_outstanding` is a running total per wallet and billable asset, and `BeforeCreate` computes it only at insert. Filling in an earlier row therefore makes every later row's total wrong. The backfill job must lock the wallet and billable asset, then recompute `billable_outstanding` for that row and every later row in `(timestamp, id)` order, all in one transaction.

`settled_by_id IS NULL` keeps settled rows untouched. Settlement already skipped their `NULL` billable, so pricing them now would change an invoice that has already been issued.

The 15-min cron runs the same query without the `period` and `price_source` filters, so it also catches any ingest event that was dropped.
