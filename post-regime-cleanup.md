Now that we have Price Source Config to manage in the incoming price processing and Price Regimes to govern the pricing models, we need to deprecate the following:
 - internal\models\price_pair.go (seems to not be references at all)
 - pair_price_offsets (replaced with the price_source_config)
 - PricingAudit (replaced with ExecutionPrice)



The default Pricing Regime for a merchant should be called "Default Pricing Regime"

# Refactor PriceSourceConfig
## SinglePriceAs is redundant
It is not necessary to hold this SinglePriceAs column as the same effect is achieved by sett a `0%` value on either the SpreadUp or SpreadDown.

## Multi Spread
The Price Source config needs to be restructured to allow different spread up and spread down for different pairs

Create a new column on the PriceSourceConfig: `spreads`
```
[
    {
        base_ticker: string | null | undefined
        quote_ticker: string | null | undefined
        spread_up: number
        spread_down: number
        comment: string | null | undefined // just a text field for the user to write a comment
    }
]
```
For example
```
[
    {
        spread_up: 0,
        spread_down: 0.02,
        comment: "The default spread for all prices from this source"
    },
    {
        base_ticker: 'XAU',
        spread_up: 0.01,
        spread_down: 0.03,
        comment: "Wider spread for gold"
    }
]
```
Only the most specific rule should apply. If there a two rules of the same specificty (ie rule A only has base_ticker and rule B only has quote_ticker), then use the one that's higher in the list.