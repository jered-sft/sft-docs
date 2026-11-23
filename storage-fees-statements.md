# StorageFeeSettlementWorker
1. Always deduct the fee in the asset that it has been accumulated in.
2. Need to select a wallet to exchange into
  - For now just choose the first wallet of that asset
3. At exactly midnight at the end of the month, as soon as the last snapshot for the wallet has been created, we charge the SGD, even if the customer has $0 in their SGD account (it just becomes negative).
4. At 10am the next day, we do a "sold for storage" to exactly cover the storage charge

Important Info! Please understand this!

There is potentially 3 assets / wallets involved in a "Sold for storage" event:
1. AccruingWallet: eg XAU (and the asset_id of the AccruingWallet = AccruingAsset)
    - This is the wallet holding the assets that incur a storage fee.
2. AccruedFeeAsset: eg SGD ()
    - The asset is defined in the StorageFeeAccrual.BillableAssetID 
        - Should be equal to StorageFeeRegime AccumulateAs + AccumulateAssetI
    - The wallet should be defined in a new column Account.StorageChargeInterimWallet (allow null). 
        - Defaults to the first wallet of the account for the accrued asset id
        - If the account does not have a wallet for the asset_id then create one and assign it to StorageChargeInterimWallet
3. The asset of the payment wallet: eg USD
    - This is defined in Account.StorageChargeWallet
    - Defaults to the accruing wallet

Here is the most complex example we need to be able to accomodate:
1. The user holds 500g of gold, at 2.5%: (500 * 0.025) / 365 = 0.034246575342465g per day
    a. GSC wants to accumulate SGD every day (assuming price of S$165/g): 0.034246575342465 * 165 = S$5.650685 per day
    b. Over 30 days, thats S$169.5205 SGD
2. We deduct S$169.52 form their SGD wallet at 24:00:00.01 on the last day of the month (ie 00:00:00.01 of the first day of the next month).
    - This is actually done as soon as the last wallet snapshot for the month is ready so should be a LISTEN trigger, not a cron.
    - Note in many cases the StorageChargeInterimWallet will be negative after this which is allowed. Fees always bypass the negative check.
3.  At the end of the month the user want to pay fees from their USD wallet (StorageChargeWalletID). 
    Then on the first business day of the next month, when the price is available (for the GSC reference price this is 10 am) we can convert the USD to SGD at whatever the price is that day.

# Update StorageFeeAccrual
Rename columns:
    - BillableAssetID to AccruedFeeAsset
    - BillableAmount to AccruedFeeAmount

# Update StorageFeeSettlement
Rename columns:
    - posting_id to fee_posting_id

Add new columns for:
    - exchange_id (allow null): Populated when an exchange is used to cover the storage fee
    - status:
        - 'pending_charge' Fee has not yet been deducted from the wallet of the accumulated fee asset. eg the fee should be charged to the SGD wallet because the fee is accumulated in SGD
        - 'pending_exchange' The fee has been charged to the accumulated fee asset wallet, but waiting an exchange to cover the payment.
        - 'complete' the fee has been charged and the exchange has been done
        - 'failed_charge'
        - 'failed_exchange'

# New SoldForStorageWorker
1. Separate the "Sold for storage" process from the StorageFeeSettlementWorker
2. Create a new SoldForStorageExecution that holds the price to be used for the SoldForStorage.
    - scan through the pending StorageFeeSettlements and find all the distinct pairs that need to be exchanged.
3. In the storage fee regime, add 2 new columns:
    - sfs_price_regime_id that is used for "Sold For Storage"
    - sfs_execution_at
        - 'immediate': process the SoldForStorage as soon as the price is available
        - 'manual': wait for a manual trigger via api, no matter how long it takes
        - '{time}' (eg 17:00 SGT): process the SoldForStorageExecution at the specified time (only if the price is already available). triggering manually before the time will also cause it to process
5. In the case of a manual trigger, accept an optional parameter to override the price. Log the override and original price in the SystemLog as 'audit'. The price override can be for any of the asset pairs waiting to be processed


# New SoldForStorageExecution
Create an API handler for this so it cam be viewed in teh backoffice.
We need:
1. List with pagination and filtering
2. Manual execution
3. Preview: find all the StorageFeeSettlement that will be exchanged, make a list of all the exchanged that will happen when manual is clicked