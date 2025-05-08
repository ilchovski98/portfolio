# Introduction
A time-boxed security review of the **Paytr** protocol was done by **ilchovski**, with a focus on the security aspects of the application's implementation.

# About Ilchovski
Ilchovski is an independent security researcher specializing in smart contract security. He has demonstrated a track record of identifying and addressing vulnerabilities across various Defi protocols. He conducts security research and reviews while leveraging his prior experience as a blockchain developer.

# Disclaimer
Security reviews are a time-boxed service that has the aim of finding as many issues as possible. The absence of vulnerabilities cannot be guaranteed. It is recommended further security reviews to be made, to set up a bug bounty program and to have an emergency response team if possible.

# Risk Classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

# Security Assessment Summary

**Audit Duration:** 12.05.2024 - 13.05.2024<br>
**_review commit hash_ - [5fd4d6585ab511be0619d493291711b0a06c4a18](https://github.com/paytr-protocol/contracts/tree/5fd4d6585ab511be0619d493291711b0a06c4a18)**

## Scope
- `Paytr.sol`

# Findings Summary

| ID     | Title                                                               | Severity |
| ------ | ------------------------------------------------------------------- | -------- |
| [M-01] | An attacker can cause a denial of service to invoice creation         | Medium |
| [M-02] | Giving max allowance to multiple third-party contracts              | Medium |
| [M-03] | Incorrect usage of the payable modifier                             | Meidum |
| [I-01] | Code duplication                                                    | Informational |
| [I-02] | Redundant error definitions                                         | Informational |
| [I-03] | Redundant functions                                                 | Informational |
| [I-04] | Use the return value of the `supply()`                              | Informational |
| [I-05] | Unnecessary max payout array size check                              | Informational |
| [I-06] | Unnecesary block.timestamp conversion to uint40                     | Informational |
| [I-07] | State-changing methods are missing event emissions                  | Informational |
| [I-08] | Not following the solidity style guide                              | Informational |

# Detailed Findings

## [M-01] An attacker can cause a denial of service to invoice creation
### Context
- [`src/Paytr.sol#L126`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L126)
- [`src/Paytr.sol#L181`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L181)

### Description
When a user calls `payInvoiceERC20()` or `payInvoiceERC20Escrow()` he provides bytes `paymentReference` by which the system saves the payment information.

https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L148-L157

```solidity
paymentMapping[_paymentReference] = PaymentERC20({
    amount: _amount,
    feeAmount: _feeAmount,
    wrapperSharesReceived: wrappedShares,
    dueDate: _dueDate,
    payer: msg.sender,
    payee: _payee,
    feeAddress: _feeAddress,
    shouldPayoutViaRequestNetwork: _shouldPayoutViaRequestNetwork
});
```

An attacker can front-run the call of the user while using the same `paymentReference`, cover the minimal token amount requirements, and set the payee address to one of his EOAs. The user call will then revert due to this check:

https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L135

```javascript
if (paymentERC20.amount != 0) revert PaymentReferenceInUse();
```

The only downside for the attacker is that he needs to lock the minimum amount of requred USDC that Paytr accepts for the minimum due date period.

### Recommendations

Have an increasing nonce that is assigned to each new payment:

```diff
+   uint256 private invoiceNonce;

...
    function payInvoiceERC20(
        address _payee,
        address _feeAddress,
        uint40 _dueDate,
        uint256 _amount,
        uint256 _feeAmount,
        bytes calldata _paymentReference,
        uint8 _shouldPayoutViaRequestNetwork
    ) public nonReentrant whenNotPaused {
        ...
-       paymentMapping[_paymentReference] = PaymentERC20({
+       paymentMapping[++invoiceNonce] = PaymentERC20({
            amount: _amount,
            feeAmount: _feeAmount,
            wrapperSharesReceived: wrappedShares,
            dueDate: _dueDate,
            payer: msg.sender,
            payee: _payee,
            feeAddress: _feeAddress,
            shouldPayoutViaRequestNetwork: _shouldPayoutViaRequestNetwork
        });
    }
```

All invoice references throughout the contract should be changed from using bytes payment reference to the new uint invoice nonce. This way even if an attacker frontruns the transaction he will just create another payment record and it will not affect the initial user call.

## [M-02] Giving max allowance to multiple third-party contracts

### Context
- [`src/Paytr.sol#L90`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L90)
- [`src/Paytr.sol#L91`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L91)
- [`src/Paytr.sol#L335`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L335)


### Description
Giving max token allowance to an external contract without having a way to remove it is dangerous and an industry standard is to avoid such actions. Due to unexpected behaviour or a vulnerability in the external smart contract's code, this can result in loss of user and protocol funds.

### Recommendations

Increase and remove allowances before and after each transfer transaction.

For example, in `payInvoiceERC20()`:
```diff
+   IERC20(baseAsset).safeIncreaseAllowance(cometAddress, totalAmount);
    IComet(cometAddress).supply(baseAsset, totalAmount);
+   IERC20(baseAsset).forceApprove(cometAddress, 0);
```

and

```diff
+   IComet(cometAddress).allow(wrapperAddress, true);
    uint256 wrappedShares = IWrapper(wrapperAddress).deposit(cUsdcAmountToWrap, address(this));
+   IComet(cometAddress).allow(wrapperAddress, false);
```

All `max approval` and `allow()` calls should be removed. Such are present in the `constructor()` and `setERC20FeeProxy()`.

## [M-03] Incorrect usage of the payable modifier

### Context
- [`src/Paytr.sol#L81`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L81)
- [`src/Paytr.sol#L310`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L310)
- [`src/Paytr.sol#L314`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L314)
- [`src/Paytr.sol#L318`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L318)
- [`src/Paytr.sol#L333`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L333)
- [`src/Paytr.sol#L340`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L340)
- [`src/Paytr.sol#L344`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L344)

### Description

There are numerous methods using the payable modifier which means callers can provide ETH with their function call. However, there is no reason for these methods to have the payable modifier because any provided ETH is going to be a donation to the contract and it will not be given back to the user. 

There isn't a withdraw or a receive function either which means that this ETH will be locked in the contract forever.

### Recommendations

Remove the payable modifier from all methods listed in the context above.

## [I-01] Code duplication
### Context
- [`src/Paytr.sol#L120`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L120)
- [`src/Paytr.sol#L176`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L176)

### Description
`payInvoiceERC20()` and `payInvoiceERC20Escrow()` are the same with the exception of the due date parameter. Code duplication can introduce vulnerabilities when frequent updates are needed in multiple places throughout the codebase during the development process. A best practice is to avoid it whenever possible.

### Recommendations

Delete `payInvoiceERC20Escrow()` and make the following edits to `payInvoiceERC20()`:

```diff
    function payInvoiceERC20(
        address _payee,
        address _feeAddress,
        uint40 _dueDate,
        uint256 _amount,
        uint256 _feeAmount,
        bytes calldata _paymentReference,
        uint8 _shouldPayoutViaRequestNetwork
    ) public nonReentrant whenNotPaused {
        ...
+       if (_dueDate != 0) {
            if(_dueDate < uint40(block.timestamp + minDueDateParameter) || _dueDate > uint40(block.timestamp + maxDueDateParameter)) revert DueDateNotAllowed();
+       }
        ...
    }
```

## [I-02]  Redundant error definitions

### Context
- [`src/Paytr.sol#L40`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L40)
- [`src/Paytr.sol#L42`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L42)
- [`src/Paytr.sol#L47`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L47)
- [`src/Paytr.sol#L48`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L48)
- [`src/Paytr.sol#L52`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L52)

### Description
Multiple custom errors are defined but never used.

### Recommendations
Remove the custom errors listed in the context above.

## [I-03] Redundant functions

### Context
- [`src/Paytr.sol#L300`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L300)
- [`src/Paytr.sol#L305`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L305)

### Description
`getContractBaseAssetBalance()` is not used anywhere in the code. Instead of declaring `getContractBaseAssetBalance()` and `getContractCometBalance()` to get token balances, we can use the `balanceOf()` method directly where we need it in order to increase code readability.

### Recommendations
Remove both `getContractBaseAssetBalance()` `getContractCometBalance()` and refactor accordingly.

## [I-04] Use the return value of the `supply()`

### Context
- [`src/Paytr.sol#L141-L143`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L141-L143)

### Recommendations
Instead of making 2 `balanceOf` calls you can use the return value of the `supply()` function.

## [I-05] Unnecessary max payout array size check

### Context
- [`src/Paytr.sol#L241`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L241)

### Description
There is no need to check if the provided array size is more than the maxPayoutArraySize since if the block gas limit is reached the transaction will revert anyway.

```javascript
    if (payoutReferencesArrayLength == 0 || payoutReferencesArrayLength > maxPayoutArraySize) revert InvalidArrayLength();
                                            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

### Recommendations
Remove the maxPayoutArraySize setting implementation and if statement.

## [I-06] Unnecessary block.timestamp conversion to uint40

### Context
- [`src/Paytr.sol#L137`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L137)

### Description
There is no need to convert block.timestamp to uint40.

### Recommendations
Remove the conversion.

## [I-07] State-changing methods are missing event emissions

### Context
- [`src/Paytr.sol#L223`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L223)
- [`src/Paytr.sol#L310`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L310)
- [`src/Paytr.sol#L314`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L314)
- [`src/Paytr.sol#L340`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L340)
- [`src/Paytr.sol#L344`](https://github.com/paytr-protocol/contracts/blob/5fd4d6585ab511be0619d493291711b0a06c4a18/src/Paytr.sol#L344)


### Description
It is a best practice for all state-changing functions to emit events for web2 systems to function properly.

### Recommendations
Emit events in all state-changing functions.

## [I-08] Not following the solidity style guide

### Recommendations
To achieve better readability and to be easier for other developers or auditors to get familiar with the codebase it is recommended to follow the the official Solidity style guide: https://docs.soliditylang.org/en/latest/style-guide.html
