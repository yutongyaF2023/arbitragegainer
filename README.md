# Arbitrage Gainer

**Team 6:** Sahana Rangarajan (ssrangar), April Yang (yutongya), Audrey Zhou (yutongz7)

**Miro Board link:** https://miro.com/app/board/uXjVNfeHKWc=/?share_link_id=780735329174

## Table of Contents

1. [Trading Strategy](#trading-strategy)
2. [Arbitrage Opportunity](#positive-test-cases)
3. [Order Management](#order-management)
4. [Domain Services](#domain-services)
   - 4a. [Crosstraded Cryptocurrencies](#crosstraded-cryptocurrencies)
   - 4b. [Historical Spread Calculation](#historical-spread-calculation)
5. [REST Endpoints](#rest-endpoints)

## Trading Strategy

The trading strategy is a bounded context representing a **core subdomain**. It consists of the following workflows, all of which can be found in [TradingStrategy.fs](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs):

- `updateTransactionsVolume` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs#L163)): processing an update to the transactions daily volume
- `updateTransactionsAmount` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs#L180)): processing an update to the transactions total amount
- `acceptNewTradingStrategy` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs#L191)): process a new trading strategy provided by the user
- `activateAcceptedTradingTrategy` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs#L204)): activate an accepted trading strategy for use in trading
- `resetForNewDay` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs#L214)): the daily volume should be reset so as to accurately track whether the maximal daily volume is reached
- `reactivateUponNewDay` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs#L223)): a strategy that was deactivated due to reaching the maximal daily volume should be reactivated when the daily volume is reset

Note that there are also two "helper workflows", `processNewTransactionVolume` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs#L150)) and `processNewTransactionAmount` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs#L156)). These functions' purpose is to link this bounded context to the order management bounded context by translating the events between both of them. This will be improved for the next milestone when we fully integrate the bounded contexts.

### Side Effects

- **User input**:
  - `startTrading` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs#L269)): The user can start trading with the `/tradingstart` REST API endpoint.
  - `stopTrading` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs#L274)): The user can stop trading with the `/tradingstop` REST API endpoint.
  - `newTradingStrategy` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs#L225)): The user can input new trading strategy parameters through the `/tradingstrategy` REST API endpoint.

### Error Handling

- Here, the error handling primarily occurs in ensuring whether the trading strategy parameters provided by the user are valid. If invalid or missing inputs are found, then an error response is returned instead of the 200 success; see [code](https://github.com/yutongyaF2023/arbitragegainer/blob/main/TradingStrategy.fs#L244) for reference.

## Arbitrage Opportunity

The ArbitrageOpportunity is a bounded context representing a **core subdomain**. It consists of the following workflows, all of which can be found in [ArbitrageOpportunity.fs](https://github.com/yutongyaF2023/arbitragegainer/blob/main/ArbitrageOpportunity.fs):

- `subscribeToRealTimeDataFeed` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/ArbitrageOpportunity.fs#L101)): Subcribing to real-time data
  feed for top N currency pairs.

  - **Contains error handling when external service (Polygon) throws exception**

- `retrieveDataFromRealTimeFeed` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/ArbitrageOpportunity.fs#L106)): Upon receiving real-time
  data, process the data and starting trading

  - **Contains error handling when external service (WebSocket) throws exception**

- `assessRealTimeArbitrageOpportunity` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/ArbitrageOpportunity.fs#L115)): Based on user provided
  trading strategy, identify arbitrage opportunities and emit orders correspondingly

- `unsubscribeRealTimeDataFeed` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/ArbitrageOpportunity.fs#L207)): When trading strategy is
  deactivated, pause trading and unsubscribe from real-time data feed

### Side Effects

Each workflow listed above includes error handling to manage potential failures and ensure the system's robustness. The following side effect areas have been identified:

- **Database Operations**:
  - `fetchCryptoPairsFromDB`([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/ArbitrageOpportunity.fs#L164-L181))
- **API Calls to Polygon for Real Time Data Retrieval**:
  - `authenticatePolygon`([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/ArbitrageOpportunity.fs#L184))
  - `connectWebSocket`([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/ArbitrageOpportunity.fs#L201))
  - `subscribeData`([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/ArbitrageOpportunity.fs#L213))
  - `subscribeToRealTimeDataFeed`([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/ArbitrageOpportunity.fs#L228))
  - `unsubscribeRealTimeDataFeed`([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/ArbitrageOpportunity.fs#L238))
  - `receiveMsgFromWSAndTrade`([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/ArbitrageOpportunity.fs#L267-L292))

## Order Management

The OrderManagement a bounded context representing a **generic subdomain**. The workflows within this bounded context can all be found within [OrderManagement.fs](https://github.com/yutongyaF2023/arbitragegainer/blob/main/OrderManagement.fs):

### General Functionalities

- `createOrderAsync` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/OrderManagement.fs#L202)): Processes each order by capturing details, initiating buy/sell orders, and recording them in the database, ultimately generating a list of `OrderInitialize` events.

- `createOrders` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/OrderManagement.fs#L217)): Taking in an `OrderEmitted` event and processes a list of orders.

- `processOrderUpdate` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/OrderManagement.fs#L235)): Retrieve and processes update about the transaction volume and amount, maintaining up-to-date records of transactions.

- `userNotification` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/OrderManagement.fs#L303)): Notifies users about their order status when only one side of the order is filled.

- `createAndProcessOrders` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/OrderManagement.fs#L313)): A wrapper workflow that creates and processes the orders based on specific handling rules.

### Side Effects and Error Handling

Each workflow listed above includes error handling to manage potential failures and ensure the system's robustness. The following side effect areas have been identified and error-handled:

**API Calls for Submitting Orders to Exchanges**:

- `initiateBuySellOrderAsync` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/OrderManagement.fs#L52)) : responsible for sending new orders to the respective exchange APIs. It handles errors such as network failures or exchange-specific errors and returns a detailed message about the failure. This function is integral to the order creation workflow and ensures that orders are placed successfully on the market.

**Order Update Broadcast**:

- `processOrderUpdate` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/OrderManagement.fs#L110)) : push updates from the exchange to our system. This function waits for a specified delay before attempting to retrieve the order status. If the connection to the exchange fails or the exchange returns an error, this function will handle it gracefully and log the error for further inspection.

**User Notifications**:

- `userNotification` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/OrderManagement.fs#L303)) : scheduled for a more detailed implementation in Milestone IV. It will handle user notifications for various order-related events. It will ensure that notifications are sent out reliably and will manage failures by retrying or logging issues as appropriate.

**Bitfinex API Functions:**

- `submitOrder` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/api/BitfinexAPI.fs#L9)) : Sends an order to the Bitfinex exchange and handles responses or errors accordingly. It ensures that any network or API-specific errors are caught and reported back to the caller.

- `retrieveOrderTrades` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/api/BitfinexAPI.fs#L23)) : Retrieves the trades for a given order from Bitfinex. It includes error handling for failed HTTP requests and unexpected response formats.

**Kraken API Functions:**

- `submitOrder` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/api/KrakenAPI.fs#L18)) : Submits an order to the Kraken exchange, handling potential errors such as connection issues or invalid responses. It ensures the caller receives a clear error message for any issues encountered.
- `queryOrderInformation` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/api/KrakenAPI.fs#L35)) : Queries detailed information about specific orders on Kraken. It handles errors robustly, including logging and reporting back any issues with the connection or the API response.

**Bitstamp API Functions:**

- `buyMarketOrder` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/api/BitstampAPI.fs#L22)) and `sellMarketOrder` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/api/BitstampAPI.fs#L29)) : These functions initiate market orders on Bitstamp and are equipped to handle errors such as network failures or unexpected API changes.
- `orderStatus` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/api/BitstampAPI.fs#L36)) : Checks the status of an order on Bitstamp, handling any potential errors during the request or in parsing the response.

**Database Operations**:

- `recordOrderInDatabaseAsync` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/OrderManagement.fs#L86)) function takes care of persisting order details into the database. It provides comprehensive error handling for database connectivity issues or write failures, ensuring that order details are not lost and system integrity is maintained.

- `addOrderToDatabase` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/database/DatabaseOperations.fs#L9)) : Attempts to add an order entity to the database and includes comprehensive error handling for database operation failures.
- `getOrderFromDatabase` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/database/DatabaseOperations.fs#L13)) : Retrieves an order entity from the database, handling cases where the order might not exist or there's an issue with the database connection.
- `updateOrderInDatabase` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/database/DatabaseOperations.fs#L17)) : Updates an existing order entity in the database, with error handling for any issues that might occur during the update process.
- `deleteOrderFromDatabase` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/database/DatabaseOperations.fs#L21)) : Removes an order entity from the database, ensuring that any errors are caught and handled appropriately.

In each of these modules, functions are designed to manage side effects—such as making HTTP requests or database operations—and include error handling to ensure the robustness of the system. The error handling strategies involve catching exceptions, validating API responses, and ensuring that any failures do not disrupt the overall workflow and are logged for further analysis.
- `databaseOperations` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/OrderManagement.fs#L189)): Handles database interactions, crucial for maintaining persistent data and state of the trading operations.

### Side Effects and Error Handling

Each workflow listed above includes error handling to manage potential failures and ensure the system's robustness. The following side effect areas have been identified and error-handled:

1. **API Calls to External Services**: `initiateBuySellOrderAsync`
2. **Database Operations**: `recordOrderInDatabaseAsync`
3. **Trade Execution**: `tradeExecution`
4. **Order Fulfillment**: `orderFulfillment`
5. **User Notifications**: `userNotification`
6. **Order Update Broadcast**: `pushOrderUpdate`
7. **Error Detection and Management**: `handleOrderError`




## Domain Services

We have classified the following functionalities in our system as domain services, each consisting of a few simple workflows.

### Crosstraded Cryptocurrencies

- `updateCrossTradedCryptos` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/CrossTradedCryptos.fs#L93)): retrieve cross-traded cryptocurrencies
- `uploadCryptoPairsToDB` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/CrossTradedCryptos.fs#L141)): note that this workflow will be implemented fully in the next milestone, as it represents a side effect.
- **Side Effects**:
  - `crossTradedCryptos` [link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/CrossTradedCryptos.fs#L153): A user-accessible REST API endpoint, `/crosstradedcryptos`, was added to retrieve updated cross traded currencies
  - The workflow `uploadCryptoPairsToDB` (now found [here](https://github.com/yutongyaF2023/arbitragegainer/blob/main/CrossTradedCryptos.fs#L141)) now uploads database results to an Azure Storage table.
  - The input currency lists for each exchange are loaded in from external txt files; see `validCurrencyPairsFromFile`([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/CrossTradedCryptos.fs#L85)).
- **Error Handling**:
  - Railway-oriented error handling was added in checking the new input files starting at `validCurrencyPairsFromFile` and propagating into the REST API endpoint when cross traded currencies are updated (see [here](https://github.com/yutongyaF2023/arbitragegainer/blob/main/CrossTradedCryptos.fs#L156)). The properties checked are whether the files exist (`validateFileExistence` [here](https://github.com/yutongyaF2023/arbitragegainer/blob/main/CrossTradedCryptos.fs#L71)) and whether the file is the proper type (`validateFileType` [here](https://github.com/yutongyaF2023/arbitragegainer/blob/main/CrossTradedCryptos.fs#L77)).
  - The database operation's response is checked for errors in the [endpoint](https://github.com/yutongyaF2023/arbitragegainer/blob/main/CrossTradedCryptos.fs#L162).

### Historical Spread Calculation

- `calculateHistoricalSpreadWorkflow` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/HistoricalSpreadCalc.fs#L77)): performs the historical spread calculation — the historical value file's quotes are separated into 5ms buckets, and arbitrage opportunities are identified from these buckets.
- **Side Effects**:
  - The historical value file is now loaded in with JSON type providers; see the `HistoricalValues` [type](https://github.com/yutongyaF2023/arbitragegainer/blob/main/HistoricalSpreadCalc.fs#L51) and `loadHistoricalValuesFile` [here](https://github.com/yutongyaF2023/arbitragegainer/blob/main/HistoricalSpreadCalc.fs#L95).
  - This calculation can be invoked by the REST API endpoint `/historicalspread` ([link](https://github.com/yutongyaF2023/arbitragegainer/blob/main/HistoricalSpreadCalc.fs#L95)).
  - The results are persisted in a database; see `persistOpportunitiesInDB` [here](https://github.com/yutongyaF2023/arbitragegainer/blob/main/HistoricalSpreadCalc.fs#L163).
- **Error Handling**:
  - We validate that the historicalData.txt file exists [here](https://github.com/yutongyaF2023/arbitragegainer/blob/main/HistoricalSpreadCalc.fs#L175)
  - We validate that the JSON type provider has all the fields we require [here](https://github.com/yutongyaF2023/arbitragegainer/blob/main/HistoricalSpreadCalc.fs#L72)
  - We handle DB errors [here](https://github.com/yutongyaF2023/arbitragegainer/blob/main/HistoricalSpreadCalc.fs#L185)
  - See the REST API endpoint to see the errors propagate into failures.

### REST Endpoints
- `/tradingstrategy`: This POST endpoint accepts a new trading strategy, specified as the following required query parameters:
  - `trackedcurrencies`: the number of cryptocurrencies to track (**int**)
  - `minpricespread`: minimal price spread value (**float**)
  - `minprofit`: minimal trasaction profit (**float**)
  - `maxamount`: maximal transaction amount (**float**)
  - `maxdailyvol`: maximal daily transactions volume (**float**)
- `/tradingstart`: This POST endpoint activates the trading strategy and begins the trading flow.
- `/tradingstop`: This POST endpoint deactivates the trading strategy and halts the trading flow.
- `/crosstradedcurrencies`: This GET endpoint retrieves pairs of currencies traded at all the currencies contained within the exchange files expected in the same location as the code. It also logs the output into a text file (`crossTradedCurrencies.txt`) and perpetuates the results in an Azure table.

### Azure Service Bus
- **[Service Bus Setup Code Pointer]()**
- **[tradingqueue](https://portal.azure.com/#@andrewcmu.onmicrosoft.com/resource/subscriptions/075cf1cf-2912-4a8b-8d6f-fbb9c461bc2b/resourceGroups/ArbitrageGainer/providers/Microsoft.ServiceBus/namespaces/ArbitrageGainer/queues/tradingqueue/overview)**: connecting **TradingStrategy** and **ArbitrageOpportunity** bounded contexts
   - message type:
      - "Stop trading" for stopping trading
      - stringified trading strategy parameters for starting trading with specified trading strategy
   - send message code pointer
   - receive message code pointer
- **[orderqueue](https://portal.azure.com/#@andrewcmu.onmicrosoft.com/resource/subscriptions/075cf1cf-2912-4a8b-8d6f-fbb9c461bc2b/resourceGroups/ArbitrageGainer/providers/Microsoft.ServiceBus/namespaces/ArbitrageGainer/queues/orderqueue/overview)**: connecting **ArbitrageOpportunity** and **OrderManagement** bounded contexts
   - message type:
      - stringified order details for creating orders
   - send message code pointer
   - receive message code pointer 
