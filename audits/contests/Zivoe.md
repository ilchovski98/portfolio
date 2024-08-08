# Introduction
Public contest organised by Sherlock.

Participated as Team Maniacs. (All findings in this report were found by Ilchovski)

Find contest details: [`here`](https://audits.sherlock.xyz/contests/280).

# About Zivoe

Zivoe is a real-world asset credit protocol aiming to disrupt predatory high-interest consumer lending. Leveraging a B2B2C model, Zivoe offers on-chain loans to regulated consumer lending entities, who then use that capital to fund consumer credit products off-chain. These entities then utilize the yield from such products to fulfill their on-chain obligations to Zivoe.

# Risk Classification

The risk classification for this audit were according to the platform's rules at the time of the audit. Only **High** and **Medium** severity findings were in scope.

# Findings Summary

| ID     | Title                                                              | Severity |
| ------ | ------------------------------------------------------------------ | -------- |
| [H-01] | All staked users will not receive rewards if they are with low...  | High     |
| [H-02] | All staked users rewards can be slowed down by anybody             | High     |
| [M-01] | Allowances block the protocol from adding liquidity to uniswap...  | Medium   |
| [M-02] | Attacker can skip the distribution of yield from OCL locker for... | Medium   |

# Findings

# [H-01] All staked users will not receive rewards if they are with low token decimals

## Summary

All users who have staked in any of the `StakingReward` contracts can lose all token rewards of low decimal tokens (USDC for example) if the difference between ZVE/Junior/Senior staked tokens is large enough when compared to the offered low decimal tokens as a reward.

The bigger the difference between these 2 values, the more feasible is going to be for an attacker to keep updating `rewardPerTokenStored` and make it increase by 0 with each call.

## Vulnerability Detail

Let's say ZivoeRewards has 1_000_000e18 staked tokens (ZVE/Junior/Senior depending on the contract) and the ZVL adds the USDC token as a reward with [`addReward()`](https://github.com/sherlock-audit/2024-03-zivoe-Maniacs/blob/f9c99eb8842a753e07d664100cc06ce133338f20/zivoe-core-foundry/src/ZivoeRewards.sol#L208). Then 40_000e6 USDC are set to be distributed to stakers with [`depositReward()`](https://github.com/sherlock-audit/2024-03-zivoe-Maniacs/blob/f9c99eb8842a753e07d664100cc06ce133338f20/zivoe-core-foundry/src/ZivoeRewards.sol#L228).

All main functions like [`depositReward()`](https://github.com/sherlock-audit/2024-03-zivoe-Maniacs/blob/f9c99eb8842a753e07d664100cc06ce133338f20/zivoe-core-foundry/src/ZivoeRewards.sol#L228), [`stake()`](https://github.com/sherlock-audit/2024-03-zivoe-Maniacs/blob/f9c99eb8842a753e07d664100cc06ce133338f20/zivoe-core-foundry/src/ZivoeRewards.sol#L253), [`stakeFor()`](https://github.com/sherlock-audit/2024-03-zivoe-Maniacs/blob/f9c99eb8842a753e07d664100cc06ce133338f20/zivoe-core-foundry/src/ZivoeRewards.sol#L268), [`getRewards()`](https://github.com/sherlock-audit/2024-03-zivoe-Maniacs/blob/f9c99eb8842a753e07d664100cc06ce133338f20/zivoe-core-foundry/src/ZivoeRewards.sol#L281), [`withdraw()`](https://github.com/sherlock-audit/2024-03-zivoe-Maniacs/blob/f9c99eb8842a753e07d664100cc06ce133338f20/zivoe-core-foundry/src/ZivoeRewards.sol#L299) have the `updateReward` modifier that updates `rewardPerTokenStored` -> `rewardData[token].rewardPerTokenStored = rewardPerToken(token);` which is storing how much tokens 1 staked token have earned after some time.

The problem is that [`rewardPerToken()`](https://github.com/sherlock-audit/2024-03-zivoe-Maniacs/blob/f9c99eb8842a753e07d664100cc06ce133338f20/zivoe-core-foundry/src/ZivoeRewards.sol#L196) introduces a rounding issue when the _totalSupply (the staked e18 tokens) is large enough and the deposited reward tokens are relatively small (e6 tokens):

```solidity
function rewardPerToken(address _rewardsToken) public view returns (uint256 amount) {
    if (_totalSupply == 0) { return rewardData[_rewardsToken].rewardPerTokenStored; }
    return rewardData[_rewardsToken].rewardPerTokenStored.add(
        lastTimeRewardApplicable(_rewardsToken).sub(
            rewardData[_rewardsToken].lastUpdateTime
@>      ).mul(rewardData[_rewardsToken].rewardRate).mul(1e18).div(_totalSupply)
    );
}
```

The formula is the following:

```solidity
// 30 days are defined in the tests
rewardRate = rewardTokens / 30 days
newRewardPerToken = lastUpdateTimestamp - currentUpdateTimestamp * rewardRate * 1e18 / _totalSupply

rewardRate = 40_000e6 / 2592000 = 15432
newRewardPerToken = 60 * 15432 * 1e18 / 1_000_000e18
newRewardPerToken = 925_920e18 / 1_000_000e18
newRewardPerToken = 0

rewardData[_rewardsToken].rewardPerTokenStored += newRewardPerToken
```

Here we have 3 critical conditions for the rounding to happen.

1. The token must be 6 decimals. (it would be more feasible with even lower decimals but those are not popular).
2. Staked token to be 25 times bigger compared to the reward
3. The `updateReward` modifier is to be called every 60 seconds. This can happen during normal user interaction between the protocol but also an attacker could be making the cheapest calls to some of the main functions in order to trigger the update. **Please note** that the time between updates can be much larger if the difference between the staked and reward tokens is more than 25 times (I just selected these values for the example).
We could have different scenarios in which the rounding and loss of rewards works:

- 50_000_000 Senior staked, 2_000_000 USDC as reward, 60 second update period
- 10_000_000 Senior staked, 40_000 USDC as reward, 600 seconds update period (10 minutes)
...

## Impact

All staked users can lose their USDC or other low decimal token rewards due to rounding.

## Code Snippet

Place the POC at the bottom of Test_ZivoeRewards.sol

```solidity
function test_Rewards_Poc() public {
      // adds a new reward with 6 decimals
      assert(zvl.try_addReward(address(stZVE), USDC, 30 days));


      // 1 000 000 ZVE staked
      assert(god.try_push(address(DAO), address(ZVEClaimer), address(ZVE), 1000000 ether, ""));
      ZVEClaimer.forward(address(ZVE), 1000000 ether, address(sam));

      assert(sam.try_approveToken(address(ZVE), address(stZVE), 1_000_000e18));
      assert(sam.try_stake(address(stZVE), 1_000_000e18));

      // deposit 40 000 USDC reward
      mint("USDC", address(bob), 40_000e6);
      assert(bob.try_approveToken(USDC, address(stZVE), 40_000e6));
      assert(bob.try_depositReward(address(stZVE), USDC, 40_000e6));

      // update rewards every minute for a duration of 1 day
      for (uint i = 0; i < (60 * 24 * 1); i++) {
          hevm.warp(block.timestamp + 60);
          assert(sam.try_getRewards(address(stZVE)));
      }

      console2.log("Sam USDC balance after 1 days: ", IERC20(USDC).balanceOf(address(sam)));
  }
```

## Recommendation

Change the contract accounting to convert low-decimal tokens reward values and store them as 18 decimals.

# [H-02] All staked users rewards can be slowed down by anybody

## Summary

In `ZivoeRewards` contract users can stake ZVE/Senior/Junior tokens to get rewards. These rewards in the tests are set to be distributed for 30 days but by calling [`depositReward`](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L228) anybody can slow this distribution rate by 35%.

## Vulnerability Detail

I will start with a simple example.

We call [`depositReward`](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L228) by depositing 1000e18 DAI (we assume that the distribution period for DAI is set to 30 days).

The rate of distribution per second is going to be 1000e18 / 30 days = 3.858e14

After 15 days, anybody can use `depositReward` to deposit 0 wei of DAI and extend the distribution period from 15 days remaining to 30 days remaining.

The rate of distribution per second becomes 500e18 / 30 days = 1.929e14

The staked users expect to receive 1000e18 but in reality, they will receive less.

Now by somebody calling `depositReward` with 0 wei over some time, they can slow down rewards in the following way:
For 1000e18 DAI deposited for 30 days, the users will receive:

- 632 DAI in 30 days (depositReward called every hour)
- 635 DAI in 30 days (depositReward called twice a day)
- 638 DAI in 30 days (depositReward called every day)
- 644 DAI in 30 days (depositReward called each 2 days)
- 651 DAI in 30 days (depositReward called each 3 days)
- 665 DAI in 30 days (depositReward called each 5 days)
- 703 DAI in 30 days (depositReward called each 10 days)

In summary, in the first month, users will get on average 65% of the tokens, the following 2 months after that they will get the next 30% and the last 5% need exponentially more time.

## Impact

Any user can slow the rewards distribution to all staked users at no cost (only gas fee cost)

## Code Snippet

```solidity
function test_Rewards_1() public {
    // deposit 1000 DAI reward
    mint("DAI", address(bob), 1000e18);
    assert(bob.try_approveToken(DAI, address(stZVE), 1000e18));
    assert(bob.try_depositReward(address(stZVE), DAI, 1000e18));

    // stake 1000 ZVE tokens
    assert(sam.try_approveToken(address(ZVE), address(stZVE), 1000e18));
    assert(sam.try_stake(address(stZVE), 1000e18));

    // call depositReward with 0 DAI every day for 30 days
    for (uint i = 0; i < 30; i++) {
        hevm.warp(block.timestamp + (1 days));
        assert(bob.try_depositReward(address(stZVE), DAI, 0));
    }

    // get DAI rewards
    assert(sam.try_getRewards(address(stZVE)));
    console2.log("Sam DAI balance after 30 days", IERC20(DAI).balanceOf(address(sam)));
}
```

## Recommendation

Either when calling `depositReward` do not increase the duration but increase the reward rate or add a minimum deposit amount check

# [M-01] Allowances block the protocol from adding liquidity to uniswap pools

## Summary

OCL_ZVE locker contract expects allowances of both tokens to be equal to 0 after providing liquidity to Uniswap/Sushiswap pool. However this is not the case in the majority of the time due to how Uniswap V2 addLiquidity function works.

## Vulnerability Detail

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L172-L215

```solidity
function pushToLockerMulti(
    address[] calldata assets, uint256[] calldata amounts, bytes[] calldata data
) external override onlyOwner nonReentrant {
    ...

    uint balPairAsset = IERC20(pairAsset).balanceOf(address(this));
    uint balZVE = IERC20(ZVE).balanceOf(address(this));
    IERC20(pairAsset).safeIncreaseAllowance(router, balPairAsset);
    IERC20(ZVE).safeIncreaseAllowance(router, balZVE);

    (uint256 depositedPairAsset, uint256 depositedZVE, uint256 minted) = IRouter_OCL_ZVE(router).addLiquidity(
        pairAsset, 
        ZVE, 
        balPairAsset,
        balZVE, 
        (balPairAsset * 9) / 10,
        (balZVE * 9) / 10, 
        address(this), 
        block.timestamp + 14 days
    );
    emit LiquidityTokensMinted(minted, depositedZVE, depositedPairAsset);
@>  assert(IERC20(pairAsset).allowance(address(this), router) == 0);
@>  assert(IERC20(ZVE).allowance(address(this), router) == 0);

    ...
}
```

The allowance would be 0 if the pool with both assets does not exist. In such a case Uniswap's [`addLiquidity()`](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L198) function will create a new pool and it will use whatever ratio is provided. Thus the whole amount of amountADesired and amountBDesired will be transferred to the router and allowances will be 0.

The second case where allowances could be 0 is when the locker succeeds in providing liquidity in perfect ratio as it is in the pair pool (not a 1 wei difference). This is extremely unlikely if there is high trading activity of the pool and it is expected that the pair ratio is going to change constantly. Even if there aren't any trades an attacker could just send a tiny amount to the pool and make the transaction revert because the router will not take the whole amount for one of the tokens.

## Impact

Adding liquidity to Uniswap/Sushiswap becomes almost impossible in normal market conditions. (DOS)
The likelihood is High
The impact is that it breaks protocol functionality and the OCL_ZVE locker contract can become unusable.

## Code Snippet

You can add this test at the bottom of Test_OCL_ZVE.sol. The test will **revert** when trying to provide liquidity with the correct ratio with the difference of 1 wei in tokenB. Assertion inside `pushToLockerMulti()` fails because there is 1 wei allowance left after adding liquidity.

```solidity
function test_approvals_poc() public {
    uint256 amountA = 1000 * USD;
    uint256 amountB = 1000 * USD;

    address[] memory assets = new address[](2);
    uint256[] memory amounts = new uint256[](2);

    assets[0] = DAI;
    assets[1] = address(ZVE);

    amounts[0] = amountA;
    amounts[1] = amountB;
    
    address poolBeforeFirstAddLiquidity = IFactory_OCL_ZVE(OCL_ZVE_UNIV2_DAI.factory()).getPair(DAI, address(ZVE));
    assert(poolBeforeFirstAddLiquidity == address(0)); // pool does not exist

    assert(god.try_pushMulti(address(DAO), address(OCL_ZVE_UNIV2_DAI), assets, amounts, new bytes[](2)));

    address poolAfterFirstAddLiquidity = IFactory_OCL_ZVE(OCL_ZVE_UNIV2_DAI.factory()).getPair(DAI, address(ZVE));
    assert(poolAfterFirstAddLiquidity != address(0)); // pool was created

    uint256[] memory differentAmounts = new uint256[](2);
    differentAmounts[0] = amountA;
    differentAmounts[1] = amountB + 1; // adding liquidity with 50/50 ratio with the difference of 1 wei

    assert(god.try_pushMulti(address(DAO), address(OCL_ZVE_UNIV2_DAI), assets, differentAmounts, new bytes[](2)));
    // reverts because 1 wei allowance from B token was not used and after addLiquidity we have an assertion for 0 allowance
}
```

## Recommendation

Instead of using assert all over the codebase, check what is the remaining allowance and reduce it to 0.

# [M-02] Attacker can skip the distribution of yield from OCL locker for the month

## Summary

[`forwardYield()`](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L287) can be called by a keeper or a user. If the keeper does not call the function and is left to the user to call it then the yield distribution can be skipped and be able to call `forwardYield()` again after 30 more days.

## Vulnerability Detail

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L301-L302
amount relies on the token pair of ZVE UniswapV2 pool which can be manipulated by the attacker.
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L336-L342

The attacker can make a swap in the pool in order to lower the price of ZVE relative to the pair asset (by using his own funds or a flash loan), then call [`forwardYield()`](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L287) and then swap back the amount.

If the difference in the `amount` and `basis` is small enough then the attacker can avoid experiencing price impact making this attack feasible.

## Impact

Yield distribution can be delayed by 30 days every time the function is left to be called by a user instead of a keeper.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L287-L305

## Recommendation

Make sure that only the keeper can call or if this is not desired the solution is not obvious.
