The following tables seem redundant or unused:
 - backoffice_wallet_adjustment_logs
 - integral_prices
 - log_destinations
 - log_entries
 - dynamic_tasks
 - event_logs
 - job_pollers
 - processing_statuses
 - task_runs
 - user_notifications vs notifications vs price_notifications
 - instruments
 - global_transaction_summaries
 - wallet_cost_basis_history

# Extend wallet_snapshots - Done
1. Ranme the table, model and api to Snapshots
1. Rename wallet_snapshots.target to wallet_snapshots.period to match the naming convention of the other period based tables.
2. Add new column target_type (string)
3. Rename column wallet_id to target_id
4. Target can be "wallet" or "account" or "merchant" or "ledger"
5. After completing all the wallet snapshots for an account for the given period, the system should then create the Account snapshot using the wallet snapshots
6. After completing the account snapshots for a merchant, the system should then create the merchant snapshot using the account snapshots
7. After completing all the merchant snapshots for a ledger, the system should then create the ledger snapshot using the merchant snapshots

# Pricing Regimes - Done
1. Price Source max age should be milliseconds
2. Add a second max age for holidays (includes holidays and weekends)

# Vendors, Ledgers and Origin Wallets - Done
1. When creating a Ledger, always create a new origin wallet. no 2 ledgers should share an origin wallet.
2. When creating a Vendor, if the Ledger is not specified, then create a new ledger using `Vendor.name + "Ledger"`
3. Place a constraint in the posting service to block transfers between wallets of two different ledgers. Tarnsfers can only happen between wallets on the same ledger.

# Snapshot Meta
Similar to PostingMets we want to create a valuations table for snapshots.
```
type SnapshotMeta struct {
    SnapshotID
    ValueAssetID
    Value
}
```
The primary purpose of this is to allow fast viewing of snapshot history in the user interface by caching the valuation history in the database.

Somehow need to have the average cost monthly

# Customer Groups - Nhan
1. Drop the current customer groups
2. Create the Customer Groups table
```
type CustomerGroup struct {
    MerchantID
    Label
    PricingRegimeID
    StorageFeeRegimeID
}
```
3. Create default customer group for all merchants
4. Every customer must have exactily 1 CustomerGroup
5. Remove the DefaultStorageFeeRegime from Merchants
6. When charging storage fees to wallets, refer to the CustomerGroup for the StorageFeeRegime
7. When creating a customer, a group must always be privided in the api request

# Customer View 
Need to show the pricing markups of the customer in the customer view

# Special Assets and Liabilities View - Done
1. From GSC point of view (positive is an asset, negative is  aliabiility)
2. Show the change and running total side by side
3. horizontal column for currencies



# Split the workers - Done
Should be exection creators and execution runners

# Snapshot history Chart - Done
If assetId is not specified, query for all the assets of the target and show them with multime axies

# Failed Savings Plans in dashboard - Done

# Customer numbers - Deon
1. Create a column on CustomerLogin `CustomerNumber`
2. Allow this column to be provided in the initial API request to create the customer and allow it to be modified after.
3. If no number is specified, the create a number that follows the standard pattern using `IND_` or `CRP_` depending of the customer type is individual or corporate


# Cleanup Storage Charge Logic - Done
1. Don't log storage_charge_wallets_found if there are no wallet to process
2. Deprecate ListMetalDailySnapshots and rename to ListDailySnapshots because snapshots and storage fees have got nothing to do with metals. Snapshots are for all wallets and estorage fees are for the assets listed in the storage fee regime. This means that some metals do not have a stroage fee and some non-metals have a storag fee. Avoid labelling things to be metal or non metal in the code, they are all just assets and the treatment will be based on configuration in the different regimes 

# Acqusition only average price
We need to capture an acqusition only average price.
For each new acqusition transaction, we need to look at the "balance_before" and multiple it by the average acqusition cost before. Then add the balance before to the transaction amount and the the total cost to the transaction value (ideally the exact traded). Then using these two number calculate the new average cost.

For example:
Buy 10g of XAU for $1000: average_acqusition_price = $100/g
Sell 9g of XAU for (irrelevant): average_acqusition_price = no change
   -> The remaining 1g still has an acqusition_price of $100/g, but now a acqusition_cost of $100 instead of $1000
Buy 10g of XAU for 2000: This acqusition = $200/g, average_acqusition_price = ($2000 + (1 * $100)) / (1 + 10) = $190.909091
Sell 5g of XAU for (irrelevant): average_acqusition_price = no change
Buy 1g of XAU for 50: This acqusition = $50/g, average_acqusition_price = ($50 + (6 * $190.909091)) / (1 + 6) = $170.779221

This also needs to be done for Dispose only average price average_dispose_price

Investigate if it's possible to use posting meta for this, ie, we just add a few new columne, including a `type` column.
Or if we need to create a new entity for this.


# Activity based logs - Done
We need to create an activity log for the user. This can be created on the fly from existing data

The columns should be
 - Datetime
 - Activity: 'Auto Savings Plan' | 'Exchange' | 'Deposit' | 'Withdraw' | 'Fee'
 - Primary Asset
 - Primary Amount
 - Counter Asset
 - Counter Amount
 - Price
{for wallet of wallets}
  - {Wallet.Label}: {activity.balance_after}
{end}

This structure intends to show all transactions for a user, but for 'Auto Savings Plan' and 'Exchange', we collapse the two transactions into one row

Create an API for this with date, wallet, account and asset filters. The request must spacify wither an account or a wallet. IF specifying an account it can optionally specify an asset which will be the primary asset.

# Emails should not be required - Done

# Killswitch - Thu
