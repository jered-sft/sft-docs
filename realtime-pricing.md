We need to create a new stream that can be requested by backoffice.

# Price Streams
Clean up the incoming price streams.
Delete these workers:
 - internal\workers\integral_price_fetch
 - internal\workers\stream_price_fetch

Can also remove internal\services\price\stream.go as it will be no longer used

Merge these PKG (the only use case for the merged pkg\integral\client.go is below)
 - pkg\pricesocket\client.go
 - pkg\integral\client.go

## Create a new worker IntergralStreamIngestWorker
The worker should subscribe to the websocket in the Initialize() function
### Every tick of the websocket should:
  1. Publish the raw event into Redis, replace the existing `pricer:live:ticks:` pattern with `stream:{price_source}:ticks` using the ProcessEvent structure. (This requires also updating the current websocket handler for these events.)
  2. Enhance PriceIngestService.IngestQuote. If the Resolution == 'STREAM' then save the QuotePriceRaw and the QuotePrices to Redis (instead of the database) under a key
        - QuotePriceRaw: `stream:{price_source}:quote_price_raw`
        - QuotePrice: `stream:{price_source}:quote_prices`   
      (This should still trigger a standard ingest event: quote_price_ingest_success but the SkipDB should be true)
Note: IngestQuote is currently too heavy to run per tick. Every call opens a transaction and reloads the source config and every asset. With about 5 sockets ticking once a second, the stream path needs caching and no database work

### (cron) Every hour
  1. Take the latest value of `stream:{price_source}:quote_price_raw`
  2. Save is in the database using `PriceIngestService.IngestQuote` but set Resolution = 'HOURLY'

## Create Mt5StreamIngestWorker
This works very similarly to IntergralStreamIngestWorker
This fetches the MT5 price from a remote database.
Create a new price_source `exc_mt5`

## FetchPrice using Stream
Enhance PriceService.queryFetchedPair
If the priceSource in the filter is a stream, then first attempt to fetch the quotePrice from redis `stream:{price_source}:quote_prices`.

# API changes

## Stream Price Regimes

### Websocket handler
Websocket requests can be made to start streaming prices from the regime: the request should contain `base_asset_id`, `quote_asset_id` and `price_regime_id`. 

When the request is received, publish an event on the redis client requesting the price.

### StreamingPriceRegimeWorker
1. Create a worker that listens on for the pricing request on Redis. This worker should hold a map of live PricingRegimeNode, keyed using `{price_regime_id}:{base_asset_id}/{quote_asset_id}`.
2. A PricingRegimeNode is created using the three parameters `base_asset_id`, `quote_asset_id` and `price_regime_id`. The node should fetch the a price from the pricing service using `FetchPriceForRegime`. This will give the initial price that the PricingRegimeNode can publish back to Redis as a price stream with `SkipDB` false (the first price should be written to the events_log).
3. The PricingRegimeNode should subscribe to the `quote_price_ingest_success` redis events for the PricingRegime.SourcePreferences. Every time there is a new price from one of them, calculate the new price using the calculation QuotePriceInput that was first returned by the initial FetchPriceForRegime


## Price Sources
For each price source, there should be an api to fetch the latest QuotePriceRaw and a websocket to see the latest of each one coming in.

Don't implement anything yet. Review the code and write a plan for this. Raise uncertiantities and questions here.
