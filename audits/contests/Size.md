# Introduction
Public contest organised by Code4rena.

[`Ilchovski`](https://x.com/ilchovski98) participated as a solo auditor for this contest (was not part of a team).

Find contest details: [`here`](https://code4rena.com/audits/2024-06-size#top).

# About Size

A credit marketplace with unified liquidity across maturities.

# Risk Classification

The risk classification for this audit were according to the platform's rules at the time of the audit. Only **High** and **Medium** severity findings were in scope.

# Findings Summary

| ID     | Title                                                              | Severity |
| ------ | ------------------------------------------------------------------ | -------- |
| [H-01] | liquidatorReward is calculated with the wrong token leading to incorrect amount of funds being sent to the liquidator      | High   |
| [M-01] | Borrower is not able to compensate his lenders if he is underwater  | Medium   |
| [M-02] | Protocol is not usable due to incorrect aaveV3 liquidity check | Medium   |
| [M-03] | BuyCreditMarket and SellCreditMarket can revert despite using correct amount of credit that is more than minimumCreditBorrowAToken | Medium   |

# Findings

# [HIGH-01] liquidatorReward is calculated with the wrong token leading to incorrect amount of funds being sent to the liquidator

## Impact
LiquidatorReward is calculated by using borrow tokens instead of collateral ones, thus sending incorrect amount of collateral tokens to the liquidator as a reward for his service.

## Proof of Concept
During a liquidation ([`executeLiquidate`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Liquidate.sol#L75C14-L75C30)) the `liquidatorReward` is used to tell how much collateral tokens should be transferred to the liquidator as a reward. However, part of the calculation uses directly debtPosition.futureValue without converting it to its equivalent in collateral tokens or directly using `assignedCollateral`.

Link to the full code snippet: [`link`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Liquidate.sol#L75-L126)

```solidity
    function executeLiquidate(State storage state, LiquidateParams calldata params)
        external
        returns (uint256 liquidatorProfitCollateralToken)
    {
        DebtPosition storage debtPosition = state.getDebtPosition(params.debtPositionId);

        ...

 @>      uint256 assignedCollateral = state.getDebtPositionAssignedCollateral(debtPosition);
         uint256 debtInCollateralToken = state.debtTokenAmountToCollateralTokenAmount(debtPosition.futureValue);
         uint256 protocolProfitCollateralToken = 0;
 
 
         // profitable liquidation
         if (assignedCollateral > debtInCollateralToken) {
             uint256 liquidatorReward = Math.min(
                 assignedCollateral - debtInCollateralToken,
 @>              Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)
             );
             liquidatorProfitCollateralToken = debtInCollateralToken + liquidatorReward;
 
             ...
         } else {
             ...
         }
 
         ...
 
 @>      state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, liquidatorProfitCollateralToken);
 
         ...
 
     }
 ```
 
 ## Recommended Mitigation Steps
 The correct calculation should use `assignedCollateral` which is the converted into collateral tokens version of `debtPosition.futureValue`:
 
 ```diff
     function executeLiquidate(State storage state, LiquidateParams calldata params)
         external
         returns (uint256 liquidatorProfitCollateralToken)
     {
         ...
         uint256 assignedCollateral = state.getDebtPositionAssignedCollateral(debtPosition);
         uint256 debtInCollateralToken = state.debtTokenAmountToCollateralTokenAmount(debtPosition.futureValue);
         uint256 protocolProfitCollateralToken = 0;
 
         // profitable liquidation
         if (assignedCollateral > debtInCollateralToken) {
             uint256 liquidatorReward = Math.min(
                 assignedCollateral - debtInCollateralToken,
 -                Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)
 +                Math.mulDivUp(assignedCollateral, state.feeConfig.liquidationRewardPercent, PERCENT)
             );   
             ...
         } else {
             ...
         }
         ...
     }
 ```

# [MEDIUM-01] Borrower is not able to compensate his lenders if he is underwater

## Impact
A borrower of a loan could have other credit positions with borrowers that have healthy CRs which he could use to compensate his lenders to avoid liquidations and improve his CR. However he is not able to do this via [`compensate()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L247) and he is forced to sell his credit positions via [`sellCreditMarket()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L188) or [`sellCreditLimit()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L172). The complications of this are described in the example below.

## Proof of Concept
Before we start just to have context, [`repay()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L198) repays the whole debt amount of the loan in full and can be called even when the borrower is underwater or past due date.

[`compensate()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L247) is used to either split a loan in smaller parts in order to be more easily repaid with [`repay()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L198) or to repay certain lenders and reduce the debt of the loan by the borrower giving specific lenders some of his own healthy credit positions that he owns.

Here is an example of the issue: Bob has collateral X in place and has 10 different debt positions (he borrowed). With the newly acquired borrow tokens he buys 10 different credit positions.

Some time passes and his collateral value drops significantly (volatile market), his CR is below the healthy threshold. At the moment Bob does not have any more borrow tokens available but he has 10 healthy credit positions that he can use to improve his CR by compensating some of his lenders.

Here comes the important part.

He can successfully compensate 1 of his lenders if at the end of the [`compensate()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L247) tx his CR becomes healthy. The problem appears when he must compensate for 3 out of all 10 debt positions (more than 1 position) in order to become with healthy CR.

He will call compensate for the first time and the end of this tx he will face a revert due to this check because he needs to compensate 2 more times to become healthy.

Link to code: [`link`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L247-L251)

```solidity
    function compensate(CompensateParams calldata params) external payable override(ISize) whenNotPaused {
        state.validateCompensate(params);
        state.executeCompensate(params);
@>      state.validateUserIsNotUnderwater(msg.sender);
     }
 ```
 
 Link to code: [`link`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/RiskLibrary.sol#L117-L133)
 
 ```solidity
     function isUserUnderwater(State storage state, address account) public view returns (bool) {
 @>      return collateralRatio(state, account) < state.riskConfig.crLiquidation;
     }
 
     function validateUserIsNotUnderwater(State storage state, address account) external view {
 @>      if (isUserUnderwater(state, account)) {
             revert Errors.USER_IS_UNDERWATER(account, collateralRatio(state, account));
         }
     }
 ```
 
 The only option Bob has currently is to sell some of his credit positions via `sellCreditMarket` or `sellCreditLimit`. There might not be any buyers at the moment or he might be forced to take a very bad deal for his credit positions because he will be in a rush in order to repay part of his loans on time to not get liquidated.
 
 ## Tools Used
 Manual Review
 
 ## Recommended Mitigation Steps
 The CR check at the end of compensate is important because fragmentation fees could occur and lower the CR and make it under the healthy threshold in some specific situations.
 
 What I propose as a solution is measuring the CR before and after the [`compensate()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L247) logic and it should revert if the CR is becoming worse.
 
 This way Bob will be able to improve his CR by using his credit positions even if he has to compensate multiple times before becoming healthy again.

# [MEDIUM-02] Protocol is not usable due to incorrect aaveV3 liquidity check

## Impact
Protocol checks the current liquidity of the underlying borrow token in aaveV3 pool in [`buyCreditMarket()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L178), [`sellCreditMarket()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L188) and [`liquidateWithReplacement()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L229). The check gets the balance of an incorrect address (the variable pool) which does not hold any funds. This causes a revert every time any of these 3 functions are called.

## Proof of Concept
Inside variablePool.supply() function inside AaveV3 code (that can be inspected via etherscan [`here`](https://etherscan.deth.net/address/0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2#readProxyContract)) we can see that the funds are actually transferred immediately to the aTokenAddress:

```solidity
function executeSupply(
     mapping(address => DataTypes.ReserveData) storage reservesData,
     mapping(uint256 => address) storage reservesList,
     DataTypes.UserConfigurationMap storage userConfig,
     DataTypes.ExecuteSupplyParams memory params
   ) external {
     DataTypes.ReserveData storage reserve = reservesData[params.asset];
     DataTypes.ReserveCache memory reserveCache = reserve.cache();
 
     reserve.updateState(reserveCache);
 
     ValidationLogic.validateSupply(reserveCache, reserve, params.amount);
 
     reserve.updateInterestRates(reserveCache, params.asset, params.amount, 0);
 
 @>  IERC20(params.asset).safeTransferFrom(msg.sender, reserveCache.aTokenAddress, params.amount);
 
     bool isFirstSupply = IAToken(reserveCache.aTokenAddress).mint(
       msg.sender,
       params.onBehalfOf,
       params.amount,
       reserveCache.nextLiquidityIndex
     );
 
     if (isFirstSupply) {
       if (
         ValidationLogic.validateAutomaticUseAsCollateral(
           reservesData,
           reservesList,
           userConfig,
           reserveCache.reserveConfiguration,
           reserveCache.aTokenAddress
         )
       ) {
         userConfig.setUsingAsCollateral(reserve.id, true);
         emit ReserveUsedAsCollateralEnabled(params.asset, params.onBehalfOf);
       }
     }
 
     emit Supply(params.asset, msg.sender, params.onBehalfOf, params.amount, params.referralCode);
   }
 ```
 
 The check will revert every time since the balance of variable pool is going to be 0.
 
 Link to code: [`link`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/CapsLibrary.sol#L67-L72)
 
 ```solidity
     function validateVariablePoolHasEnoughLiquidity(State storage state, uint256 amount) public view {
 @>        uint256 liquidity = state.data.underlyingBorrowToken.balanceOf(address(state.data.variablePool));
 @>      if (liquidity < amount) {
             revert Errors.NOT_ENOUGH_BORROW_ATOKEN_LIQUIDITY(liquidity, amount);
         }
     }
 ```
 
 ## Recommended Mitigation Steps
 Use the aToken address to check what is the current liquidity:
 
 ```diff
     function validateVariablePoolHasEnoughLiquidity(State storage state, uint256 amount) public view {
 -         uint256 liquidity = state.data.underlyingBorrowToken.balanceOf(address(state.data.variablePool));
 +         address aToken = state.data.variablePool.getReserveData(address(state.data.underlyingBorrowToken)).aTokenAddress;
 +         uint256 liquidity = state.data.underlyingBorrowToken.balanceOf(aToken);
 
         if (liquidity < amount) {
             revert Errors.NOT_ENOUGH_BORROW_ATOKEN_LIQUIDITY(liquidity, amount);
         }
     }
 ```
 
# [MEDIUM-03] BuyCreditMarket and SellCreditMarket can revert despite using correct amount of credit that is more than minimumCreditBorrowAToken
 
## Impact
User could try to create correct amount of credit (above `minimumCreditBorrowAToken`) but fail to do so because under some circumstances `minimumCreditBorrowAToken` will be compared to the incoming cash (instead of the end credit value).

## Proof of Concept
`minimumCreditBorrowAToken` is used to ensure "the minimum credit value of loans" as it is stated in a code comment [`here`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/SizeStorage.sol#L47).

Let's use [`buyCreditMarket`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L178) for the example.

We call it with exactAmountIn == true and params.creditPositionId == RESERVED_ID. `params.amount` in this case will represent the amount of cash that will be is sent to the borrower minus the fees. The credit amount of the loan is going to be `params.amount * PERCENT + ratePerTenor / PERCENT`.

```solidity
    function executeBuyCreditMarket(State storage state, BuyCreditMarketParams memory params)
        external
        returns (uint256 cashAmountIn)
    {
        ...
        if (params.exactAmountIn) {
            cashAmountIn = params.amount;
            (creditAmountOut, fees) = state.getCreditAmountOut({
                cashAmountIn: cashAmountIn,
                maxCashAmountIn: params.creditPositionId == RESERVED_ID
@>                  ? cashAmountIn
                    : Math.mulDivUp(creditPosition.credit, PERCENT, PERCENT + ratePerTenor),
                maxCredit: params.creditPositionId == RESERVED_ID
@>                  ? Math.mulDivDown(cashAmountIn, PERCENT + ratePerTenor, PERCENT)
                    : creditPosition.credit,
                ratePerTenor: ratePerTenor,
                tenor: tenor
            });
        } else {
            ...
        }


        if (params.creditPositionId == RESERVED_ID) {
            // slither-disable-next-line unused-return
            state.createDebtAndCreditPositions({
                lender: msg.sender,
                borrower: borrower,
                futureValue: creditAmountOut,
                dueDate: block.timestamp + tenor
            });
        } else {
            ...
        }


@>      state.data.borrowAToken.transferFrom(msg.sender, borrower, cashAmountIn - fees);
        state.data.borrowAToken.transferFrom(msg.sender, state.feeConfig.feeRecipient, fees);
    }
```

This means that under these circumstances `params.amount` could be below `minimumCreditBorrowAToken` but the credit amount of the loan could be above this threshold when we add the interest so the transaction should not revert.

`minimumCreditBorrowAToken` is used to ensure that the credit of the loans are going to be more than this value but this requirement should not be present for the cash that is coming in.

## Recommended Mitigation Steps
Ensure that a revert occurs only when `params.amount` is representing credit that is below `minimumCreditBorrowAToken` or when the end loan credit amount is below this threshold too.
