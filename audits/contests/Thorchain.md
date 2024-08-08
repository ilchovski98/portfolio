# [HIGH-01] Rebasing tokens are not handled correctly

## Submission Link
https://github.com/code-423n4/2024-06-thorchain-findings/issues/53

# Lines of code
https://github.com/code-423n4/2024-06-thorchain/blob/e3fd3c75ff994dce50d6eb66eb290d467bd494f5/chain/ethereum/contracts/THORChain_Router.sol#L54

# Vulnerability details
## Impact
Not handling rebasing tokens properly can result some vaults not being able to use their allowances due to lack of funds in the router or amount of rebasing tokens to get locked forever.

## Proof of Concept
Rebasing tokens like stETH for example can change the users underlying balance in both directions due to adding rewards or slashing.

In such case it is not correct to store the allowances for vaults in a storage variable while using the raw version of stETH because the underlying balance of the Router contract will change over time due to Lido's protocol logic.

So if stETH contract balance increases over time even if all `_vaultAllowance[vault][token]` go to 0 (due to transfer out functions usage), there will be remaining stETH amounts that will get locked in the contract forever.

On the other hand if a slashing event occurs and all vaults try to reduce their `_vaultAllowance[vault][token]` to 0, not all will be able to do it. The last couple of vaults will not be able to do this because there won't be enough stETH left in the router.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Rebasing tokens should be handled differently from normal ERC20 tokens. One solution I think of is when accepting such tokens they should be wrapped to their non rebasing version. This way _vaultAllowance logic will not break. In the case of stETH, the tokens should be converted to wstETH.

## Assessed type
ERC20

# [HIGH-02] Refund functionality breaks protocol logic

## Submission Link
https://github.com/code-423n4/2024-06-thorchain-findings/issues/70

# Lines of code
https://github.com/code-423n4/2024-06-thorchain/blob/e3fd3c75ff994dce50d6eb66eb290d467bd494f5/chain/ethereum/contracts/THORChain_Router.sol#L196 https://github.com/code-423n4/2024-06-thorchain/blob/e3fd3c75ff994dce50d6eb66eb290d467bd494f5/chain/ethereum/contracts/THORChain_Router.sol#L213 https://github.com/code-423n4/2024-06-thorchain/blob/e3fd3c75ff994dce50d6eb66eb290d467bd494f5/chain/ethereum/contracts/THORChain_Router.sol#L280 https://github.com/code-423n4/2024-06-thorchain/blob/e3fd3c75ff994dce50d6eb66eb290d467bd494f5/chain/ethereum/contracts/THORChain_Router.sol#L326

# Vulnerability details
## Impact
In cases where the recipient fails to receives native funds being transferred to him via any of the transfer out functions, the system does not have a way of knowing that the transfer have indeed failed. This causes the funds to stay in the vault and the recipient loses the funds.

## Proof of Concept
Whenever a vault carries out a transfer out transaction that sends native asset to an address and this operation fails the native asset is returned to the caller but the event is emitted like the transaction is successful.

Example:

1. User calls [`depositWithExpiry()`](https://github.com/code-423n4/2024-06-thorchain/blob/e3fd3c75ff994dce50d6eb66eb290d467bd494f5/chain/ethereum/contracts/THORChain_Router.sol#L131) on Ethereum to transfer funds.
2. The system picks up the deposit event and processes the tx.
3. The user must receive native funds on another chain but the native currency transfer that uses internal send call fails due to OOG error. Then the code sends the funds to the msg.sender which is the vault.

```solidity
  function transferOut(
    address payable to,
    address asset,
    uint amount,
    string memory memo
  ) public payable nonReentrant {
    uint safeAmount;
    if (asset == address(0)) {
      safeAmount = msg.value;
      bool success = to.send(safeAmount); // Send ETH.

      if (!success) {
@>      payable(address(msg.sender)).transfer(safeAmount); // For failure, bounce back to vault & continue.
      }
    } else {
       ...
    }
@>  emit TransferOut(msg.sender, to, asset, safeAmount, memo);
  }


function _transferOutV5(TransferOutData memory transferOutPayload) private {
    if (transferOutPayload.asset == address(0)) {
      bool success = transferOutPayload.to.send(transferOutPayload.amount); // Send ETH.

      if (!success) {
@>      payable(address(msg.sender)).transfer(transferOutPayload.amount); // For failure, bounce back to vault & continue.
      }
    } else {
      ...
    }


@>  emit TransferOut(
      msg.sender,
      transferOutPayload.to,
      transferOutPayload.asset,
      transferOutPayload.amount,
      transferOutPayload.memo
    );
  }
```

4. The TransferOut event is emitted which will indicate to the system that the transaction is completed but in fact the user never received the funds.
5. This results in a loss for the user.

## Tools Used
Manual review

## Recommended Mitigation Steps
Instead of refunding the funds to the vault and emitting an event either revert when the native assets cannot be transferred to the recipient or change the event parameters to indicate for this failure and occurred refund.

## Assessed type
Other

# [MEDIUM-01] Using msg.value inside a loop breaks function functionality

## Submission Link
https://github.com/code-423n4/2024-06-thorchain-findings/issues/56

# Lines of code
https://github.com/code-423n4/2024-06-thorchain/blob/e3fd3c75ff994dce50d6eb66eb290d467bd494f5/chain/ethereum/contracts/THORChain_Router.sol#L397

# Vulnerability details
## Impact
When [`batchTransferOutAndCallV5()`](https://github.com/code-423n4/2024-06-thorchain/blob/e3fd3c75ff994dce50d6eb66eb290d467bd494f5/chain/ethereum/contracts/THORChain_Router.sol#L397) is called with 2 native currency transfers either contract native assets will get stolen (if there are any inside the contract) or the function will always revert.

## Proof of Concept
The root of the problem is that msg.value is used inside a loop. When this is done the contract receives for example 1 ETH but on each iteration of the loop msg.value is 1 ETH.

This will cause the contract to try sending on each iteration 1 ETH by spending its own funds and not only the funds it had received. Since the contract is not expected to hold ETH the function will revert on the second loop iteration since there won't be enough ETH to be sent.

Example:

1. We call [`batchTransferOutAndCallV5()`](https://github.com/code-423n4/2024-06-thorchain/blob/e3fd3c75ff994dce50d6eb66eb290d467bd494f5/chain/ethereum/contracts/THORChain_Router.sol#L397) with [{fromAsset: address(0), fromAmount: 5e17...}, {fromAsset: address(0), fromAmount: 5e17...}] and we send 1e18 ETH with the function call.
2. On the first iteration the function will make a swap with the whole msg.value amount - thus using for the first recipient the whole 1 ETH despite the defined fromAmount value:
   Snippet: [`link`](https://github.com/code-423n4/2024-06-thorchain/blob/e3fd3c75ff994dce50d6eb66eb290d467bd494f5/chain/ethereum/contracts/THORChain_Router.sol#L304-L389)

```solidity
  function _transferOutAndCallV5(
    TransferOutAndCallData calldata aggregationPayload
  ) private {
    if (aggregationPayload.fromAsset == address(0)) {
      // call swapOutV5 with ether
      (bool swapOutSuccess, ) = aggregationPayload.target.call{
@>      value: msg.value // @audit the value here is 1 ETH
      }(
        abi.encodeWithSignature(
          "swapOutV5(address,uint256,address,address,uint256,bytes,string)",
          aggregationPayload.fromAsset,
          aggregationPayload.fromAmount, // @audit this value gets ignored when fromAsset == address(0)
          aggregationPayload.toAsset,
          aggregationPayload.recipient,
          aggregationPayload.amountOutMin,
          aggregationPayload.payload,
          aggregationPayload.originAddress
        )
      );

      if (!swapOutSuccess) {
        bool sendSuccess = payable(aggregationPayload.target).send(msg.value); // If can't swap, just send the recipient the gas asset
        if (!sendSuccess) {
          payable(address(msg.sender)).transfer(msg.value); // For failure, bounce back to vault & continue.
        }
      }


      emit TransferOutAndCallV5(
        msg.sender,
        aggregationPayload.target,
        msg.value,
        aggregationPayload.toAsset,
        aggregationPayload.recipient,
        aggregationPayload.amountOutMin,
        aggregationPayload.memo,
        aggregationPayload.payload,
        aggregationPayload.originAddress
      );
    } else {
      ...
    }
  }
```

We can see the swapOutV5 is using only msg.value and ignores fromAmount value:

```solidity
  function swapOutV5(
    address fromAsset,
    uint256 fromAmount,
    address toAsset,
    address recipient,
    uint256 amountOutMin,
    bytes memory payload,
    string memory originAddress
  ) public payable nonReentrant {
    address[] memory path = new address[](2);


    // you can do something with originAddress: https://gitlab.com/thorchain/thornode/-/merge_requests/3339#note_1884273837


    if (fromAsset == address(0)) {
      // received ETH, swap to toToken or do as you please
      path[0] = WETH;
      path[1] = toAsset;


      if (payload.length == 0) {
        // no payload, process without wasting gas
@>      swapRouter.swapExactETHForTokens{value: msg.value}(
@>        amountOutMin,
          path,
          recipient,
          type(uint).max
        );
      } else {
        // do something with your payload like parse to an aggregator specific structure
        swapRouter.swapExactETHForTokens{value: msg.value}(
          amountOutMin,
          path,
          recipient,
          type(uint).max
        );
      }
    } else {
      ...
    }
  }
```

4. The second loop iteration will revert if there isn't 1 ETH in the contract or if there is it will make the swap by using contract's funds.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Use fromAmount instead of msg.value and inside [`batchTransferOutAndCallV5()`](https://github.com/code-423n4/2024-06-thorchain/blob/e3fd3c75ff994dce50d6eb66eb290d467bd494f5/chain/ethereum/contracts/THORChain_Router.sol#L397) place a check that verifies that the sum of all native fromAmount values in the function parameters is equal to msg.value. By using fromAmount each loop will use the correct amount for the swap and we will make sure that `_transferOutAndCallV5()` will only use the native funds that were sent with the same transaction.

## Assessed type
ETH-Transfer

