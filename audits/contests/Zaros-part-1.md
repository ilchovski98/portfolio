# Introduction
Public contest organised by Codehawks.

Participated as Team AuditTemple.

Find contest details: [`here`](https://codehawks.cyfrin.io/c/2024-07-zaros).

# About Zaros

Zaros is a Perpetuals DEX powered by Boosted (Re)Staking Vaults. It seeks to maximize LPs yield generation, while offering a top-notch trading experience on Arbitrum (and Monad in the future).

# Risk Classification

The risk classification for this audit were according to the platform's rules at the time of the audit.

# Findings Summary

| ID     | Title                                                              | Severity |
| ------ | ------------------------------------------------------------------ | -------- |
| [H-01] | Liquidations reset open interest and skew, which breaks price calculations for an entire Perp Market  | High     |
| [M-01] | Incorrect margin requirement calculation when simulating trades           | Medium     |
| [M-02] | FillOffchainOrders() can revert more times than necessary | Medium   |
| [M-03] | ChainlinkUtil checks for sequencerUptimeFeed are incorrect | Medium   |
| [M-04] | priceFeedHeartBeat cannot be updated | Medium   |
| [L-01] | UpgradeBranch.sol does not use _disableInitializers() | Low   |
| [L-02] | User can withdraw all of his funds while having an active market order thus DOSing keeper | Low   |
| [L-03] | Offchain keepers cannot provide native currency in a reasonable way to cover the Chainlink fee | Low   |
| [L-04] | weETH and WSTETH do not have usd price feeds but have ETH ones | Low   |
| [L-05] | createTradingAccountAndMulticall() donates the native currency that is sent to the contract | Low   |

# Findings

# [H-01] Liquidations reset open interest and skew, which breaks price calculations for an entire Perp Market

## Summary

[`liquidateAccounts()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/LiquidationBranch.sol#L209) function inside `Liquidationbranch.sol`  updates the open interest and skew values of the perpetual market with uninitialized values (which default to 0), compromising price calculations for that market.

## Vulnerability Details

[`liquidateAccounts()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/LiquidationBranch.sol#L209) function is responsible for liquidating all market positions, whose margin collateral ratio has dropped below the healthy threshold defined by the protocol.

Reference to code: [link](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/LiquidationBranch.sol#L209)

```solidity

function liquidateAccounts(uint128[] calldata accountsIds) external {   
	...

	// iterate over every account being liquidated; intentionally not caching
	// length as reading from calldata is faster
	for (uint256 i; i < accountsIds.length; i++) {    
			...
	            
			// iterate over memory copy of active market ids
			// intentionally not caching length as expected size < 3 in most cases
			for (uint256 j; j < ctx.activeMarketsIds.length; j++) {      
					...
		                
		      // update perp market's open interest and skew; we don't enforce ipen
		      // interest and skew caps during liquidations as:
		      // 1) open interest and skew are both decreased by liquidations
		      // 2) we don't want liquidation to be DoS'd in case somehow those cap
		      //    checks would fail
@>			  perpMarket.updateOpenInterest(ctx.newOpenInterestX18, ctx.newSkewX18);
	     }
	
       ...
    }
}
```

The function iterates through every account and then through every market position of that account and after doing all the necessary checks and actions it calls `updateOpenInterest()` as the final step of each nested loop.

The function itself updates two important storage variables of a `perpMarket`:

```solidity
function updateOpenInterest(Data storage self, UD60x18 newOpenInterest, SD59x18 newSkew) internal {
	  self.skew = newSkew.intoInt256().toInt128();
	  self.openInterest = newOpenInterest.intoUint128();
}
```

* `openInterest` - stores the total open positions in the market and is used to track if the market limits have been reached
* `skew` - the current skew of a market. This value is very important since it is used for the calculation of the market price through [`perpMarket.getMarkPrice()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/leaves/PerpMarket.sol#L98) . It is a vital function used throughout the whole protocol for creating orders, filling orders, liquidations and etc.

The problem here is that when `updateOpenInterest()` is called inside `liquidateAccounts` the provided parameters - `ctx.newOpenInterestX18` & `ctx.newSkewX18` are both in their default (uninitialized) state, which is 0. If you go through the flow of the function you can see that both fields of the `ctx` struct are not set anywhere and the `updateOpenInterest` call is actually the first time where they get used (with their initial 0 values). Compare this with [`SettlementBranch._fillOrder()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/SettlementBranch.sol#L456-L460)  and you will see how those should get calculated before being passed to the update function

The final result is the reset of the `openInterest` & `skew` for the whole perpMarket when a single market position of an trading account gets liquidated which affects all other existing positions in this marker.

## Impact

* Because `openInterest` is nullified the `PerpMarket.checkOpenInterestLimits()` function will not track the required limits properly and all other logic depending on that variable will be distorted market wide.
* Because `skew` is nullified, the `PerpMarket.getMarkPrice()` function - which is a cornerstone for most operations in Zaros - will calculate prices at significantly distorted rates, affecting almost anything in the system - order creation, settlement, liquidations etc.

## Tools Used

Manual Review

## Recommended Mitigation

Similar to [`SettlementBranch._fillOrder()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/SettlementBranch.sol#L456-L460), first calculate the new values of `openInterest` & `skew` and then provide them as arguments to the update function.

Since this is the liquidation function and it should not revert, the `checkOpenInterestLimits()` function used in `SettlementBranch._fillOrder()` should also be modified so that it does not revert during liquidation in case some limits are reached. You can add an additional parameter to it to conditionally check for limits when needed:

```solidity
function checkOpenInterestLimits(
    Data storage self,
    SD59x18 sizeDelta,
    SD59x18 oldPositionSize,
    SD59x18 newPositionSize,
+   bool checkLimits
)
  internal
	view
	returns (UD60x18 newOpenInterest, SD59x18 newSkew)
{
		...
}
```

# [M-01] Incorrect margin requirement calculation when simulating trades

## Summary

&#x20;

The `simulateTrade` function of `OrderBranch.sol` is used to check in advance if the market position will become liquidatable after the order gets executed. However it incorrectly compares the balances (after the trade) against the margin requirements(before the trade) - which leads to imprecise validation.

## Vulnerability Details

The `simulateTrade` function of `OrderBranch.sol` simulates market order execution using current market conditions to check if the order can be filled, without violating the required collateral thresholds. This is how the most vital part of the check looks like:

<https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/OrderBranch.sol#L123-L137>

```solidity
 function simulateTrade(
        uint128 tradingAccountId,
        uint128 marketId,
        uint128 settlementConfigurationId,
        int128 sizeDelta
    )
        public
        view
        returns (
           ....
        )
    {
        ....

        // int128 -> SD59x18
        ctx.sizeDeltaX18 = sd59x18(sizeDelta);

       ....

        // calculate & output required initial & maintenance margin for the simulated trade
        // and account's unrealized PNL
        (requiredInitialMarginUsdX18, requiredMaintenanceMarginUsdX18, ctx.accountTotalUnrealizedPnlUsdX18) =
            tradingAccount.getAccountMarginRequirementUsdAndUnrealizedPnlUsd(marketId, ctx.sizeDeltaX18);

        // use unrealized PNL to calculate & output account's margin balance
        marginBalanceUsdX18 = tradingAccount.getMarginBalanceUsd(ctx.accountTotalUnrealizedPnlUsdX18);
        {
            // get account's current required margin maintenance (before this trade)
            (, ctx.previousRequiredMaintenanceMarginUsdX18,) =
                tradingAccount.getAccountMarginRequirementUsdAndUnrealizedPnlUsd(0, SD59x18_ZERO);

            // prevent liquidatable accounts from trading
            if (TradingAccount.isLiquidatable(ctx.previousRequiredMaintenanceMarginUsdX18, marginBalanceUsdX18)) {
                revert Errors.AccountIsLiquidatable(tradingAccountId);
            }
        }
        
        ....
    }
```

This is the order of operation in the above snippet:

* `tradingAccount.getAccountMarginRequirementUsdAndUnrealizedPnlUsd()` is called with the size of the trade so that it can simulate the value of `accountTotalUnrealizedPnlUsdX18` after the execution of the order
* after that `marginBalanceUsdX18` is calculated, using `accountTotalUnrealizedPnlUsdX18` AFTER the trade
* another block of code follows that calls again `tradingAccount.getAccountMarginRequirementUsdAndUnrealizedPnlUsd()` , however this time with 0 values, meaning that the calculated values will be the current ones and the changes after order execution are NOT reflected
* finally a `isLiquidatable` check is made on the account, which compares if `marginBalanceUsdX18` (AFTER the trade) is lower than the maintenance `margin` (BEFORE the trade)

The problem is in the last step, because the `isLiquidatable` check is conducted between values from 2 different stages, instead of the same stage.

Example:

* Let’s assume:

  * the calculated price of collateral is `1$` to keep it simple
  * `maintenanceMarginRate` is `1.2`
  * `marginBalanceUsd` without `PnL` is `1500$`
  * the simulated order reduces the position size from 1000 to 500 tokens
  * based on the above the calculated `requiredMaintenanceMargin` (AFTER the trade) = (`position size` \* `price` \* `maintenanceMarginRate` )⇒ 500 \* 1\$ \* 1.2 = `600$`
  * the `PnL` (AFTER the trade) due to the price change is reduced in half from `800$` to `400$` . This means that the calculated `marginBalance` (AFTER the trade) is `1500$ - 400$` ⇒ `1100$`
  * `previousRequiredMaintenanceMargin` (BEFORE the trade) = (`position size` \* `price` \* `maintenanceMarginRate` )⇒ 1000 \* 1\$ \* 1.2 = `1200$`
  * `isLiquidatable()` checks that `previousRequiredMaintenanceMargin` > `marginBalance` ( 1200\$ > 1100\$) ⇒ which returns `true` and respectively reverts the function, because the simulation concludes that after the trade the position will not be healthy anymore

  As you can see calculating `requiredMaintenanceMargin` and `marginBalanceUsd` from 2 different stages produces inaccurate results. If the proper `requiredMaintenanceMargin` (AFTER the trade) was used the the calculation would succeed ( because (600\$ is not > 1100\$) and the call will not revert and market order creation will not be prevented.
## Impact

Market order creation will be inaccurately reverted and will prevent healthy accounts from trading

## Tools Used

Manual Review

## Recommended Mitigation

Update the `simulateTrade()` function so that it properly to compares the values from the same stage:
```solidity
 	(requiredInitialMarginUsdX18, requiredMaintenanceMarginUsdX18, ctx.accountTotalUnrealizedPnlUsdX18) =
            tradingAccount.getAccountMarginRequirementUsdAndUnrealizedPnlUsd(marketId, ctx.sizeDeltaX18);

  // use unrealized PNL to calculate & output account's margin balance
  marginBalanceUsdX18 = tradingAccount.getMarginBalanceUsd(ctx.accountTotalUnrealizedPnlUsdX18);
-  {
-      // get account's current required margin maintenance (before this trade)
-      (, ctx.previousRequiredMaintenanceMarginUsdX18,) =
-          tradingAccount.getAccountMarginRequirementUsdAndUnrealizedPnlUsd(0, SD59x18_ZERO);

-      //@audit - this should use the marginBalanceUsdX18 calculated from the PnL before the trade
-      // prevent liquidatable accounts from trading
-     if (TradingAccount.isLiquidatable(ctx.previousRequiredMaintenanceMarginUsdX18, marginBalanceUsdX18)) {
+			if (TradingAccount.isLiquidatable(requiredMaintenanceMarginUsdX18, marginBalanceUsdX18)) {
         revert Errors.AccountIsLiquidatable(tradingAccountId);
    }

```

# [M-02] FillOffchainOrders() can revert more times than necessary

## Summary

&#x20;

Off-chain keepers calls to [`fillOffchainOrders()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/SettlementBranch.sol#L186) could be DOSed on purpose or revert more times than necessary, thus preventing the execution of possibly time-sensitive orders and bringing losses to users.

## Vulnerability Details

Off-chain keepers call [`fillOffchainOrders()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/SettlementBranch.sol#L186) to execute multiple off-chain orders for a specific market that are eligible for execution.

The function loops through all off-chain order struct argument details and executes them one by one.

```solidity
  function fillOffchainOrders(
      uint128 marketId,
      OffchainOrder.Data[] calldata offchainOrders,
      bytes calldata priceData
  )
      external
      onlyOffchainOrdersKeeper(marketId)
  {
	  ...

	  for (uint256 i; i < offchainOrders.length; i++) {
		  ...
	  
       _fillOrder(
          ctx.offchainOrder.tradingAccountId,
          marketId,
          SettlementConfiguration.OFFCHAIN_ORDERS_CONFIGURATION_ID,
          sd59x18(ctx.offchainOrder.sizeDelta),
          ctx.fillPriceX18
      );
	  }
  }

```

There are many reasons however, an off-chain order would revert and would stop the execution of all off-chain orders it is batched with.

Such reasons could include:

* a user canceling his off-chain orders by increasing his nonce
* a user transferring his trading account NFT to another EOA making the initial signatures for his off-chain orders invalid
* reasons connected with any of the if statements inside the off-chain orders loop.

This would cause a delay in the execution of orders that could be crucial for users such as stop-loss, take-profit or other orders.

In times of high volatility time is of essence and a difference by a couple of minutes could have huge impact on the portfolio of users.

Furthermore, a malicious user could track the price changes provided by Chainlink and in times where a market experiences high volatility (big drop or big spike in price) he could predict if there would be many stop-loss or take-profit orders about to be executed by a keeper and try to delay them (notice without front-running - without watching transactions in a mempool since this is not possible on Arbitrum) by creating many off-chain orders, signing them and then either call [`cancelAllOffchainOrders()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/OrderBranch.sol#L365) or simply transfer the trading account NFT to another EOA that he owns.

If some of his off-chain orders get included in a batch of other valid off-chain orders, he will be able to successfully stop their execution.

## Impact

Delaying the execution of possibly time sensitive offchain orders and bringing potential losses to users.

## Tools Used

Manual Review

## Recommended Mitigation

Without having access to the off-chain keeper logic that includes/discards off-chain orders in a batch, it is not possible to know the severity of the DOS (if the [`fillOffchainOrders()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/SettlementBranch.sol#L186) call would revert all the time or it will revert far less times).

From the smart contract perspective the issue is easily fixable by skipping the offchain order loop iteration with `continue` instead of using `revert`. This way if an order is invalid for whatever reason the loop could simply continue executing the rest of the orders instead of stopping the whole execution.

# [M-03] ChainlinkUtil checks for sequencerUptimeFeed are incorrect

## Summary

&#x20;

[ChainlinkUtil.getPrice() ](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/external/chainlink/ChainlinkUtil.sol#L26-L57)are not handling an edge case where sequencer price feed is in an “invalid round”.

## Vulnerability Details

[ChainlinkUtil.getPrice()](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/external/chainlink/ChainlinkUtil.sol#L26-L57) has 2 checks that aim to validate that sequencerUptimeFeed (the sequencer on L2) is running and that the time since the update is more than the grace period.

Reference in code: [link](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/external/chainlink/ChainlinkUtil.sol#L41-L57)

```solidity
function getPrice(
    IAggregatorV3 priceFeed,
    uint32 priceFeedHeartbeatSeconds,
    IAggregatorV3 sequencerUptimeFeed
)
    internal
    view
    returns (UD60x18 price)
{
	...
	
  if (address(sequencerUptimeFeed) != address(0)) {
      try sequencerUptimeFeed.latestRoundData() returns (
          uint80, int256 answer, uint256 startedAt, uint256, uint80
      ) {
          bool isSequencerUp = answer == 0;
@>        if (!isSequencerUp) {
              revert Errors.OracleSequencerUptimeFeedIsDown(address(sequencerUptimeFeed));
          }

          uint256 timeSinceUp = block.timestamp - startedAt;
@>        if (timeSinceUp <= Constants.SEQUENCER_GRACE_PERIOD_TIME) {
              revert Errors.GracePeriodNotOver();
          }
      } catch {
          revert Errors.InvalidSequencerUptimeFeedReturn();
      }
  }
	
	...
}

```

The [chainlink docs](https://docs.chain.link/data-feeds/l2-sequencer-feeds) say that `startedAt`  is going to be 0 “if a round is invalid”.

[![Screenshot-2024-07-30-at-21-23-41.png](https://i.postimg.cc/wBXf5stY/Screenshot-2024-07-30-at-21-23-41.png "Screenshot-2024-07-30-at-21-23-41.png")](https://postimg.cc/bGv9prHm)

**"invalid round"** is described to mean there was a problem updating the sequencer's status, possibly due to network issues or problems with data from oracles, and is shown by a `startedAt` time of 0 and `answer` is 0. Further explanation can be seen as given by an official chainlink engineer as seen here in the chainlink public discord

<https://discord.com/channels/592041321326182401/605768708266131456/1213847312141525002>

[![Screenshot-2024-07-30-at-21-51-32.png](https://i.postimg.cc/HsNhTmGR/Screenshot-2024-07-30-at-21-51-32.png "Screenshot-2024-07-30-at-21-51-32.png")](https://postimg.cc/N514xWjk)

This means that the following check is going to pass when an “invalid round” occurs.

```solidity
uint256 timeSinceUp = block.timestamp - startedAt;
if (timeSinceUp <= Constants.SEQUENCER_GRACE_PERIOD_TIME) {
    revert Errors.GracePeriodNotOver();
}

```

`timeSinceUp` will always be more than the hardcoded grace period (3600) since `block.timestamp - 0 == block.timestamp`

When a round starts, at the beginning `startedAt` is recorded to be 0, and `answer`, the initial status is set to be 0. Note that docs say that if `answer` = 0, sequencer is up, if equals to 1, sequencer is down. But in this case here, `answer` and `startedAt` can be 0 initially, till after all data is gotten from oracles and update is confirmed then the values are reset to the correct values that show the correct status of the sequencer.

The checks in [ChainlinkUtil.getPrice()](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/external/chainlink/ChainlinkUtil.sol#L26-L57) allow for successful calls in an invalid round because code will not revert if answer == 0 and startedAt == 0 thus defeating the purpose of having a sequencerFeed check to assert the status of the sequencerFeed on L2 i.e if it is up/down/active or if its status is actually confirmed to be either.

## Impact

The current [ChainlinkUtil.getPrice()](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/external/chainlink/ChainlinkUtil.sol#L26-L57) checks for the sequencer will cause `getPrice()` to not revert even when the sequencer uptime feed is not updated or is called in an invalid round.

## Tools Used

Manual Review

## Recommended Mitigation

Revert the transaction if `startedAt` is 0

# [M-04] priceFeedHeartBeat cannot be updated

## Summary

&#x20;

`GlobalConfigurationBranch.updatePerpMarketConfiguration()` function does not update `priceFeedHeartbeatSeconds`.

## Vulnerability Details

`updatePerpMarketConfiguration()` updates the configuration variables of the given perp market id. It does this by first validating all values that are not the default ones and then proceeds to write in storage.

Reference in code: [link](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/GlobalConfigurationBranch.sol#L475-L541)

```solidity
    function updatePerpMarketConfiguration(
        uint128 marketId,
        UpdatePerpMarketConfigurationParams calldata params
    )
        external
        onlyOwner
        onlyWhenPerpMarketIsInitialized(marketId)
    {
        PerpMarket.Data storage perpMarket = PerpMarket.load(marketId);
        MarketConfiguration.Data storage perpMarketConfiguration = perpMarket.configuration;

        if (abi.encodePacked(params.name).length == 0) {
            revert Errors.ZeroInput("name");
        }
        if (abi.encodePacked(params.symbol).length == 0) {
            revert Errors.ZeroInput("symbol");
        }
        if (params.priceAdapter == address(0)) {
            revert Errors.ZeroInput("priceAdapter");
        }
        if (params.maintenanceMarginRateX18 == 0) {
            revert Errors.ZeroInput("maintenanceMarginRateX18");
        }
        if (params.maxOpenInterest == 0) {
            revert Errors.ZeroInput("maxOpenInterest");
        }
        if (params.maxSkew == 0) {
            revert Errors.ZeroInput("maxSkew");
        }
        if (params.initialMarginRateX18 == 0) {
            revert Errors.ZeroInput("initialMarginRateX18");
        }
        if (params.initialMarginRateX18 <= params.maintenanceMarginRateX18) {
            revert Errors.InitialMarginRateLessOrEqualThanMaintenanceMarginRate();
        }
        if (params.skewScale == 0) {
            revert Errors.ZeroInput("skewScale");
        }
        if (params.minTradeSizeX18 == 0) {
            revert Errors.ZeroInput("minTradeSizeX18");
        }
        if (params.maxFundingVelocity == 0) {
            revert Errors.ZeroInput("maxFundingVelocity");
        }
        if (params.priceFeedHeartbeatSeconds == 0) {
            revert Errors.ZeroInput("priceFeedHeartbeatSeconds");
        }

        perpMarketConfiguration.update(
            MarketConfiguration.Data({
                name: params.name,
                symbol: params.symbol,
                priceAdapter: params.priceAdapter,
                initialMarginRateX18: params.initialMarginRateX18,
                maintenanceMarginRateX18: params.maintenanceMarginRateX18,
                maxOpenInterest: params.maxOpenInterest,
                maxSkew: params.maxSkew,
                maxFundingVelocity: params.maxFundingVelocity,
                minTradeSizeX18: params.minTradeSizeX18,
                skewScale: params.skewScale,
                orderFees: params.orderFees,
@>              priceFeedHeartbeatSeconds: params.priceFeedHeartbeatSeconds
            })
        );

        emit LogUpdatePerpMarketConfiguration(msg.sender, marketId);
    }

```
The issue lies inside the `perpMarketConfiguration.update()`- it does not use  `priceFeedHeartbeatSeconds`provided in the input `params`, preventing its value from getting updated

Reference in code: [link](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/leaves/MarketConfiguration.sol#L20C5-L49C6)

```solidity
	struct Data {
	    string name;
	    string symbol;
	    address priceAdapter;
	    uint128 initialMarginRateX18;
	    uint128 maintenanceMarginRateX18;
	    uint128 maxOpenInterest;
	    uint128 maxSkew;
	    uint128 maxFundingVelocity;
	    uint128 minTradeSizeX18;
	    uint256 skewScale;
	    OrderFees.Data orderFees;
@>    uint32 priceFeedHeartbeatSeconds;
	}
	
	/// @notice Updates the given market configuration.
	/// @dev See {MarketConfiguration.Data} for parameter details.
	function update(Data storage self, Data memory params) internal {
	    self.name = params.name;
	    self.symbol = params.symbol;
	    self.priceAdapter = params.priceAdapter;
	    self.initialMarginRateX18 = params.initialMarginRateX18;
	    self.maintenanceMarginRateX18 = params.maintenanceMarginRateX18;
	    self.maxOpenInterest = params.maxOpenInterest;
	    self.maxSkew = params.maxSkew;
	    self.maxFundingVelocity = params.maxFundingVelocity;
	    self.minTradeSizeX18 = params.minTradeSizeX18;
	    self.skewScale = params.skewScale;
	    self.orderFees = params.orderFees;
@>	  // @audit priceFeedHeartbeatSeconds is missing
	}

```

Chainlink heartbeats can change over time and in order to be sure that the protocol is not using any stale prices there should be a way to update the price feed heartbeat.

## Impact

Not being able to change the chainlink price feed heartbeat could lead to the usage of stale prices and loss of funds.

## Tools Used

Manual Review

## Recommended Mitigation

Make the following changes to the update function:
```solidity
	/// @notice Updates the given market configuration.
	/// @dev See {MarketConfiguration.Data} for parameter details.
	function update(Data storage self, Data memory params) internal {
	    self.name = params.name;
	    self.symbol = params.symbol;
	    self.priceAdapter = params.priceAdapter;
	    self.initialMarginRateX18 = params.initialMarginRateX18;
	    self.maintenanceMarginRateX18 = params.maintenanceMarginRateX18;
	    self.maxOpenInterest = params.maxOpenInterest;
	    self.maxSkew = params.maxSkew;
	    self.maxFundingVelocity = params.maxFundingVelocity;
	    self.minTradeSizeX18 = params.minTradeSizeX18;
	    self.skewScale = params.skewScale;
	    self.orderFees = params.orderFees;
+	    self.priceFeedHeartbeatSeconds = params.priceFeedHeartbeatSeconds;
	}

```

# [L-01] UpgradeBranch.sol does not use _disableInitializers()
## Summary

&#x20;

`UpgradeBranch.sol` does not disable initializers which allows a third party to become the owner of this implementation contract.

## Vulnerability Details

`UpgradeBranch.sol` is one of the implementation contracts that the `rootProxy` will point to once the `rootProxy` gets deployed.  Since `UpgradeBranch.sol` does not disable initializers in its constructor, this opens the possibility for a third party to make himself the owner of `UpgradeBranch.sol` by calling `initialize()`.
```solidity
contract UpgradeBranch is Initializable, OwnableUpgradeable {
    using RootUpgrade for RootUpgrade.Data;

@>  function initialize(address owner) external initializer {
        __Ownable_init(owner);
    }
}

```

The problem here is that the `UpgradeBranch` itself contains the [upgrade logic function](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/tree-proxy/branches/UpgradeBranch.sol#L19):

```solidity
 function upgrade(
        RootProxy.BranchUpgrade[] memory branchUpgrades,
        address[] memory initializables,
        bytes[] memory initializePayloads
    )
        external
    {
        _authorizeUpgrade(branchUpgrades);
        RootUpgrade.Data storage rootUpgrade = RootUpgrade.load();

        rootUpgrade.upgrade(branchUpgrades, initializables, initializePayloads);
    }
```

It executes a delegatecall to any address, which means that the account that takes over the implementation can provide a malicious contract with `selfdestruct()`and destroy the implementation. The problem here is that the implementation contract itself is used to execute the upgrade logic( UUPS) instead the logic living on the Proxy (Transperent Upgradeable Proxy).

Sad simply, In order for the proxy to upgrade to another contract it relies on the implementation of `UpgradeBranch`, but if the implementation gets destroyed, there would be no way (no logic) to update to another contract or simply replace it ( as would be possible if the upgrade logic lives in the proxy itself)

## Impact

Implementation can get destoyed, compromising future updates

## Tools Used

Manual Review

## Recommended Mitigation

Make the following changes to `UpgradeBranch.sol`:

```solidity
contract UpgradeBranch is Initializable, OwnableUpgradeable {
    using RootUpgrade for RootUpgrade.Data;
    
+   constructor() {
+		    _disableInitializers();
+		}

    function initialize(address owner) external initializer {
        __Ownable_init(owner);
    }

    function upgrade(
        RootProxy.BranchUpgrade[] memory branchUpgrades,
        address[] memory initializables,
        bytes[] memory initializePayloads
    )
        external
    {
        _authorizeUpgrade(branchUpgrades);
        RootUpgrade.Data storage rootUpgrade = RootUpgrade.load();

        rootUpgrade.upgrade(branchUpgrades, initializables, initializePayloads);
    }

    function _authorizeUpgrade(RootProxy.BranchUpgrade[] memory) internal onlyOwner { }
}

```

# [L-02] User can withdraw all of his funds while having an active market order thus DOSing keeper

## Summary

&#x20;

The user can withdraw all of his funds while having an active market order in place thus allowing him to DOS the keeper when it tries to fill his order.

## Vulnerability Details

Steps to reproduce the issue:

1. User deposits funds
2. Opens market order
3. Withdraws funds
4. The market order is still present
5. Keeper tries to [`fillMarketOrder()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/SettlementBranch.sol#L107)
6. Transaction reverts

### POC

Make the following changes to simulateTrade.t.sol and run the test with `forge test --mt test_dos_keeper_by_withdraw -vvvv`
```solidity
...
+ import { OrderBranch } from "@zaros/perpetuals/branches/OrderBranch.sol";

contract SimulateTradeIntegrationTest is Base_Test {
		...
+		function test_dos_keeper_by_withdraw() public {
+        // user deposit funds
+        uint256 marginValueUsd = 5000e18;
+        deal({ token: address(usdz), to: users.naruto.account, give: marginValueUsd });
+        uint128 tradingAccountId = createAccountAndDeposit(marginValueUsd, address(usdz));
+        uint128 marketId = 1;
+
+       // user make market order
+        perpsEngine.createMarketOrder(
+            OrderBranch.CreateMarketOrderParams({
+                tradingAccountId: tradingAccountId,
+                marketId: marketId,
+                sizeDelta: 1e18
+            })
+        );

+        // user withdraws the funds
+        perpsEngine.withdrawMargin(tradingAccountId, address(usdz), marginValueUsd);

+        // keeper fills order
+        MarketConfig memory fuzzMarketConfig = getFuzzMarketConfig(marketId);
+        bytes memory firstMockSignedReport = getMockedSignedReport(fuzzMarketConfig.streamId, fuzzMarketConfig.mockUsdPrice);
+        address keeper = marketOrderKeepers[marketId];
+        changePrank({ msgSender: keeper });

+        // fillMarketOrder call reverts because there isn't any margin in the account
+        vm.expectRevert();
+        perpsEngine.fillMarketOrder(tradingAccountId, marketId, firstMockSignedReport);
+    }
		...
}

```

## Impact

Market order keeper can be DOSed and user can withdraw his funds while having an active market order ready to be filled.

## Tools Used

Manual Review

## Recommended Mitigation

Upon withdraw check if the user has an existing market order for this market and check if the withdraw transaction should revert or clear the market order.

Be careful if you decide to clear the the market order in this situation because this should be under the condition that the minimum order lifetime has already passed.

# [L-03] Offchain keepers cannot provide native currency in a reasonable way to cover the Chainlink fee

## Summary

&#x20;

Off-chain order keepers do not have a way to supply native currency to fill successfully off-chain orders.

## Vulnerability Details

Let’s follow the internal calls when an off-chain orders keeper calls [`fillOffchainOrders()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/SettlementBranch.sol#L186).

[`fillOffchainOrders()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/SettlementBranch.sol#L186) -> [`verifyOffchainPrice()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/leaves/SettlementConfiguration.sol#L133) -> [`verifyDataStreamsReport()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/leaves/SettlementConfiguration.sol#L160) -> [`getEthVericationFee()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/external/chainlink/ChainlinkUtil.sol#L83) -> [`verifyReport()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/external/chainlink/ChainlinkUtil.sol#L95)

The code calls Chainlink’s [`getFeeAndReward()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/external/chainlink/ChainlinkUtil.sol#L92) function to know what fee we should give to Chainlink for the verification (the fee could be in the form of tokens or native currency).

[`verifyReport()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/external/chainlink/ChainlinkUtil.sol#L95) in our case calls the Chainlink verifier by providing the fee amount in the form of native currency.

Reference to code: [link](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/external/chainlink/ChainlinkUtil.sol#L95)

```solidity
function verifyReport(
    IVerifierProxy chainlinkVerifier,
    FeeAsset memory fee,
    bytes memory signedReport
)
    internal
    returns (bytes memory verifiedReportData)
{
    verifiedReportData = chainlinkVerifier.verify{ value: fee.amount }(signedReport, abi.encode(fee.assetAddress));
}

```

The issue is that keepers do not have a way to supply native currency in order for the `verifyReport()` internal call to be successful. Current implementation contracts do not have a `receive` function and the only `payable` function present is the [`createTradingAccountAndMulticall()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/TradingAccountBranch.sol#L285).

This means that in order for keepers to be able to do their job, they will have to periodically call [`createTradingAccountAndMulticall()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/TradingAccountBranch.sol#L285) in order to supply native currency to the contract and they will keep creating new trading accounts that they will never use. After supplying native currency they will be able to call [`fillOffchainOrders()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/SettlementBranch.sol#L186) successfully.

It seems that part of the logic that handles native currency transfers is incomplete.

## Impact

Keepers cannot function normally without calling separately unrelated to them functions in order to successfully call [`fillOffchainOrders()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/SettlementBranch.sol#L186).

## Tools Used

Manual Review

## Recommended Mitigation

This is a design decision that the team should take for themselves since there is no info regarding who has to pay the chainlink verification fee.

If keepers are the one that have to pay the verification fee then [`fillOffchainOrders()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/SettlementBranch.sol#L186) should be made payable and the excess funds that are not used by Chainlink to be returned to the keeper. If the funds that are left after the verification fee are not returned to the user then either the team has to decide that they will be left in the contract to accumulate and be used at some stage by another keeper or the team could decide to have a withdraw function to collect the remaining dust.

In case users are the ones who have to supply this native currency, logic that determines how much must be sent and verify if it is indeed sent has to be implemented.

# [L-04] weETH and WSTETH do not have usd price feeds but have ETH ones

## Summary

&#x20;

weETH and WSTETH do not have Chainlink USD price feeds on Arbitrum. Instead they have ETH ones. Their USD price can be calculated but such logic is missing and they will not be able to be supported despite being in scope.

## Vulnerability Details

Chainlink Price Feed for WSTETH / ETH - [link](https://data.chain.link/feeds/arbitrum/mainnet/wsteth-eth)

Chainlink Price Feed for weETH / ETH - [link](https://data.chain.link/feeds/arbitrum/mainnet/weeth-eth)

As we can see 1 WSTETH or 1 weETH do not equal 1 ETH so assuming that the ETH / USD price feed would do the job this is not the case.

At the time of writing this report **1 WSTETH** == **1.1736 ETH** and **1 weETH** == **1.0438 ETH.**

In order for the protocol to support these tokens they should implement additional logic that fetches the **ETH / USD** price and then fetches either the **WSTETH / ETH or weETH / ETH** price. Once the protocol knows how much ETH is 1 WSTETH and how much USD is 1 ETH it will be easy to determine the price of the asset in USD.

There could be a list of assets that have a ETH price feed instead of a USD one for which this logic would be applied.

## Impact

With the current implementation weETH and WSTETH are not supported despite being in scope.

## Tools Used

Manual Review

## Recommended Mitigation

Implement the logic that was described above.

# [L-05] createTradingAccountAndMulticall() donates the native currency that is sent to the contract

## Summary

&#x20;

[`createTradingAccountAndMulticall()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/TradingAccountBranch.sol#L285) is a payable function and if a user sends native currency together with their call they will essentially donate their funds to the contract.

## Vulnerability Details

[`createTradingAccountAndMulticall()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/TradingAccountBranch.sol#L285) is payable but currently, there isn’t any logic inside the protocol that handles `msg.value` in any way.

In other words, users who create a trading account and use the multi-call functionality do not have a reason to send native currency (if they do have it was not stated anywhere and there isn’t any logic that determines the amount that has to be sent, nor the sent amount is recorded in storage).

## Impact

Users who send native funds by mistake or intentionally when calling [`createTradingAccountAndMulticall()`](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/TradingAccountBranch.sol#L285) will experience a loss of funds and there won’t be any gain or change in state as a result.

## Tools Used

Manual Review

## Recommended Mitigation

Implement any missing logic that requires native currency transfer or simply remove the `payable` modifier from [`createTradingAccountAndMulticall()`.](https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/TradingAccountBranch.sol#L285)
