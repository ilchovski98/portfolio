# Introduction
Public contest organised by Code4rena.

[`Ilchovski`](https://x.com/ilchovski98) participated as a solo auditor for this contest (was not part of a team).

Find contest details: [`here`](https://code4rena.com/audits/2024-05-munchables#top).

# About Munchables

A web3 point farming game in which Keepers nurture creatures to help them evolve, deploying strategies to earn them rewards in competition with other players.

# Risk Classification

The risk classification for this audit were according to the platform's rules at the time of the audit. Only **High** and **Medium** severity findings were in scope.

# Findings Summary

| ID     | Title                                                              | Severity |
| ------ | ------------------------------------------------------------------ | -------- |
| [H-01] | Attacker can extend user lock indefinitely and DOS setLockDuration()      | High   |

# Findings

# [HIGH-01] Attacker can extend user lock indefinitely and DOS setLockDuration()

# Lines of code
https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L275

## Impact
Users lock can be extended by a malicious user with just 1 or 0 wei deposit and this would also cause [`setLockDuration()`](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L245) to always revert because the unlock time will be constantly increasing. As a result user funds will not be retrievable until the attacker stops making deposits on the behalf of the user.

## Proof of Concept
A user can lock tokens by using [`lock()`](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L297) or somebody else can lock tokens on behalf of the user with [`lockOnBehalf()`](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L275).

When a lock is made the unlockTime gets updated with the user configured lock duration:

Reference: [`link`](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L379-L384)

```solidity
    ...
    lockedToken.remainder = remainder;
    lockedToken.quantity += _quantity;
    lockedToken.lastLockTime = uint32(block.timestamp);
@>  lockedToken.unlockTime =
        uint32(block.timestamp) +
        uint32(_lockDuration);
    ...
```

Both [`unlock()`](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L401) and [`setLockDuration()`](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L245) will be affected and will revert each time the attacker extends the lock:

Reference: [`link`](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L410-L411)

```solidity
    if (lockedToken.unlockTime > uint32(block.timestamp))
        revert TokenStillLockedError();
```

Reference: [`link`](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L256-L261)

```solidity
    if (
        uint32(block.timestamp) + uint32(_duration) <
        lockedTokens[msg.sender][tokenContract].unlockTime
    ) {
        revert LockDurationReducedError();
    }
```

So with just 1 or 0 wei an attacker can extend the lock of a user and can continue to do this as much as he wants. The only expense he has are the gas fee costs.

## Recommended Mitigation Steps
Introduce mechanisms that will make it harder for a malicious user to extend the lock duration of a user or remove this opportunity entirely.

Such mechanisms could include adding a minimum lock token amount check or allow the user to create a whitelist of addresses that can make locks on his behalf.
