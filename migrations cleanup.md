# Squash migrations + close missing constraints

Not in production; every database holds test data. Rather than carry nullable columns that only
exist because old test rows lacked values, fix the models, regenerate one `initial_migration`,
drop the databases and redeploy.

Open questions (answer before step D):
- Is the database behind `config.prod.yaml` empty / unused? If it holds data, the squash cannot reach it without a rebuild or baseline.
- Branch: `feature/drop-dead-tables` (already deletes the dead models) or a fresh branch after it merges?

## What a plain regenerate would lose
`make generate-migration` = `atlas migrate diff --env gorm`, which reads only `internal/models`. SQL-only objects must be carried over by hand:
- Triggers/functions: `posting_chain_link`, `posting_chain_advance`, `posting_chain_immutable` (ledger chain — fills `postings.balance_after`, `wallets.last_posting_id`/`balance`); `notify_*_insert` on `transactions`, `exchanges`, `accounts`, `savings_plan_execution`, `storage_fee_accrual_executions`, `wallet_snapshots` (or `snapshots`, if the generic-snapshots rename lands first — take whatever is current at squash time).
- Partial indexes with no model tag: `uidx_storage_fee_accruals_wallet_execution` (`WHERE type = 'accrual'`), `idx_savings_plan_schedules_processing` (`WHERE status = 'processing'`).
- Seed rows (Default ledger, Default/Manual pricing regimes) were no-ops on an empty DB anyway; bootstrap must create them.

`scripts/dev/check_migrations.sh` already accepts a squash (every base file deleted, exactly one pair added), but the new timestamp must still sort after `main`'s newest — e.g. `20261030000000_initial_migration`, not the old `20260910072842`.

## Plan

### A. Models (code only)
- [ ] `Posting.BalanceAfter` — add `not null`, keep `default:null` (NOT NULL is checked after BEFORE triggers, so `posting_chain_link` still fills it)
- [ ] `Merchant.LedgerID` — add `not null`
- [ ] `SavingsPlanExecution.SavingsPlanRegimeID` — add `not null` after confirming every create path sets it (`manual_pricing_service.go`, `savings_plan_execute/job_creator.go`, test fixtures)
- [ ] `BackofficeUser.EmailOtpPrefix` — add `not null`
- [ ] Sweep all models for non-pointer Go fields whose tag lacks `not null`; decide each
- [ ] Add `where:` index tags for the two partial indexes so the models describe the full schema
- [ ] Optional: CHECK `(period IS NULL) = (quote_price_raw_id IS NULL)` on `savings_plan_execution` and `storage_fee_accrual_executions`

### B. Regenerate
- [ ] Delete every `migrations/*.sql` and `migrations/atlas.sum` (includes the uncommitted `drop_dead_tables` pair — the model deletions cover it)
- [ ] `make generate-migration NAME=initial_migration` against a clean atlas dev DB (`DB_ATLAS_NAME`)
- [ ] Append the trigger/function SQL from the old files to the generated `.up.sql`; matching `DROP TRIGGER`/`DROP FUNCTION` to the `.down.sql`
- [ ] `atlas migrate hash` to rebuild `atlas.sum`

### C. Verify (local forks only)
- [ ] Fork A: apply the old chain from `main`. Fork B: apply the new single file
- [ ] `atlas schema diff` A → B: only the intended constraints and the dead-table drops may differ
- [ ] Insert a posting on Fork B and confirm `balance_after`, `previous_posting_id` and `wallets.last_posting_id` are set; confirm a `pg_notify` fires
- [ ] `make setup-local` on Fork B — fresh DB boots and seeds (ledger, Default pricing regime, merchants)
- [ ] `make check-migrations`, `/verify` (lint + tests)

### D. Deploy
- [ ] Before each drop: `make psql PRINT=1` to confirm the target (`GF_APP_ENV` overrides `config.yaml`; `config.prod.yaml` points at Azure)
- [ ] Drop + recreate dev, staging, and local forks (`make db-list`); `make migrate`; seed. River recreates its own tables on boot. Alert DB is a separate stream — untouched unless also reset
- [ ] Tell the team: branches with unmerged migrations must regenerate on top of the squash; old version rows match nothing in the new set
- [ ] Update docs citing old migration versions (e.g. `20260806000000` in CLAUDE.md)

## Review
_(fill in when done)_
