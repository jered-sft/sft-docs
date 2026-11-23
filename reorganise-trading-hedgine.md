# Vendor Based
Currently all of the ext_trades and accounting is organised by ledger_id.
We want to change this to be organised by vendor_id.

## Migration
The information currently in the database is not important, so just backfill vendor_id using the first vendor_id from the ledger_id.
