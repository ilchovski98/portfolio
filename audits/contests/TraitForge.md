# Introduction
Public contest organised by Code4rena.

[`Ilchovski`](https://x.com/ilchovski98) participated as a solo auditor for this contest (was not part of a team).

Find contest details: [`here`](https://code4rena.com/audits/2024-07-traitforge).

# About TraitForge

The ultimate NFT game where your digital entities evolve, breed, and nuke for big ETH rewards. Mint, forge, and compete in this unique game. Play smart and reshape the future of NFTs!

# Risk Classification

The risk classification for this audit were according to the platform's rules at the time of the audit. Only **High** and **Medium** severity findings were in scope.

# Findings Summary

| ID     | Title                                                              | Severity |
| ------ | ------------------------------------------------------------------ | -------- |
| [H-01] |   Users can mint an unlimited number of NFTs and generations    | High   |
| [H-02] |   mintWithBudget() will become unusable after the first generation    | High   |
| [H-03] |   Users can mint more NFTs per generation because generationMintCounts gets reset    | High   |
| [H-04] |   initializeAlphaIndices uses an incorrect modifier    | High   |
| [M-01] |   Nuke can burn user NFT without giving any reward    | Medium   |
| [M-02] |   Contracts do not implement pause/unpause()    | Medium   |
| [M-03] |   User about to nuke can be front-runned and burn their NFT for far less ETH than expected    | Medium   |

# Findings

# [HIGH-01] Users can mint an unlimited number of NFTs and generations

# Lines of code
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L280 https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L311

# Vulnerability details
## Impact
Users can create an unlimited number of generations and can mint an unlimited number of NFTs.

## Proof of Concept
The vulnerability is present in both [`_mintInternal()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L280) and [`_mintNewEntity()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L311) and we will explore both scenarios.

### _mintInternal()
This function is used in both [`mintToken()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L181) and [`mintWithBudget()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L202) .

Reference to code: [link](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L280-L309)

```solidity
  function _mintInternal(address to, uint256 mintPrice) internal {
    if (generationMintCounts[currentGeneration] >= maxTokensPerGen) {
@>    _incrementGeneration();
    }

    _tokenIds++;
    uint256 newItemId = _tokenIds;
    _mint(to, newItemId);
    uint256 entropyValue = entropyGenerator.getNextEntropy();

    tokenCreationTimestamps[newItemId] = block.timestamp;
    tokenEntropy[newItemId] = entropyValue;
    tokenGenerations[newItemId] = currentGeneration;
    generationMintCounts[currentGeneration]++;
    initialOwners[newItemId] = to;

    if (!airdropContract.airdropStarted()) {
      airdropContract.addUserAmount(to, entropyValue);
    }

    emit Minted(
      msg.sender,
      newItemId,
      currentGeneration,
      entropyValue,
      mintPrice
    );

    _distributeFunds(mintPrice);
  }
```

As we can see from the code above, once the max amount of tokens per generation is reached [`_incrementGeneration()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355) gets called.

Reference to code: [link](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355)

```solidity
  function _incrementGeneration() private {
    require(
@>    generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    currentGeneration++;
    generationMintCounts[currentGeneration] = 0;
    priceIncrement = priceIncrement + priceIncrementByGen;
    entropyGenerator.initializeAlphaIndices();
    emit GenerationIncremented(currentGeneration);
  }
```

Just like [`_mintInternal()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L280), [`_incrementGeneration()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355) checks if the max amount of tokens per generation is reached but there isn’t a check to verify that `currentGeneration` will become above `maxGeneration` once increased. This allows any user to mint as much tokens he wants and the generation count will increase indefinitely every time `maxTokensPerGen` is reached.

### **_mintNewEntity()**
This function is used in [`forge()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L153-L179).

Reference to code: [link](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L153-L179)

```solidity
  function forge(
    address newOwner,
    uint256 parent1Id,
    uint256 parent2Id,
    string memory
  ) external whenNotPaused nonReentrant returns (uint256) {
    require(
      msg.sender == address(entityForgingContract),
      'unauthorized caller'
    );
    uint256 newGeneration = getTokenGeneration(parent1Id) + 1;

    /// Check new generation is not over maxGeneration
@>  require(newGeneration <= maxGeneration, "can't be over max generation");

    // Calculate the new entity's entropy
    (uint256 forgerEntropy, uint256 mergerEntropy) = getEntropiesForTokens(
      parent1Id,
      parent2Id
    );
    uint256 newEntropy = (forgerEntropy + mergerEntropy) / 2;

    // Mint the new entity
@>  uint256 newTokenId = _mintNewEntity(newOwner, newEntropy, newGeneration);

    return newTokenId;
  }
```

The impact that can be done here is limited by the `maxGeneration` check that is implemented above. However, we can still increase the current generation above the max generation just once (if max is 10 we increase it to be 11).

Here is how:

1. We forge 2 parents that are both the 9th generation.
2. newGeneration is going to be 9 + 1 = 10
3. Max generation check passes since 10 ≤ 10 is true.
4. We go inside _mintNewEntity(x, y, 10);

Reference to code: [link](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L311-L343)

```solidity
  function _mintNewEntity(
    address newOwner,
    uint256 entropy,
    uint256 gen
  ) private returns (uint256) {
    require(
@>    generationMintCounts[gen] < maxTokensPerGen,
      'Exceeds maxTokensPerGen'
    );

    _tokenIds++;
    uint256 newTokenId = _tokenIds;
    _mint(newOwner, newTokenId);

    tokenCreationTimestamps[newTokenId] = block.timestamp;
    tokenEntropy[newTokenId] = entropy;
    tokenGenerations[newTokenId] = gen;
    generationMintCounts[gen]++;
    initialOwners[newTokenId] = newOwner;

    if (
@>    generationMintCounts[gen] >= maxTokensPerGen && gen == currentGeneration
    ) {
      _incrementGeneration();
    }

    if (!airdropContract.airdropStarted()) {
      airdropContract.addUserAmount(newOwner, entropy);
    }

    emit NewEntityMinted(newOwner, newTokenId, gen, entropy);
    return newTokenId;
  }
```

1. The current generation is 10 and generationMintCounts is `maxTokensPerGen` - 1 so the first check passes.
2. generationMintCounts increases to `maxTokensPerGen` and the generation of the forged NFT is 10 (like the current generation) so [`_incrementGeneration()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355) gets called and increases the current generation to 11 (above the max that is 10)

We will not be able to increase the generations above that since [`forge()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L153-L179) implements:

```solidity
require(newGeneration <= maxGeneration, "can't be over max generation");
```

### Final notes
Both functions [`_mintInternal()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L280) and [`_mintNewEntity()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L311) use [`_incrementGeneration()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355) under slightly different circumstances. [`_mintInternal()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L280) can be used to increase the number of generations without a limit and [`_mintNewEntity()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L311) can increase the generations from gen10 to gen11, limited in comparison to [`_mintInternal()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L280) due to the validation of the [`forge()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L153-L179) function.

I decided to put them both in one report since we can mitigate the issue by putting additional validation straight into [`_incrementGeneration()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355) itself (as you will see in the Mitigation section below) instead of adding separate additional checks in both [`_mintInternal()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L280) and [`_mintNewEntity()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L311).

## Tools Used
Manual Review

## Recommended Mitigation Steps
Make the following changes to [`_incrementGeneration()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355):

```diff
  function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    currentGeneration++;
+   require(currentGeneration <= maxGeneration, "can't be over max generation");
    generationMintCounts[currentGeneration] = 0;
    priceIncrement = priceIncrement + priceIncrementByGen;
    entropyGenerator.initializeAlphaIndices();
    emit GenerationIncremented(currentGeneration);
  }
```

## Assessed type
Invalid Validation

# [HIGH-02] mintWithBudget() will become unusable after the first generation

# Lines of code
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L202

# Vulnerability details
## Impact
[`mintWithBudget()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L202-L225) will no longer mint new NFTs once the first generation is minted thus breaking protocol functionality.

## Proof of Concept
Reference in code: [link](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L202-L225)

```solidity
  function mintWithBudget(
    bytes32[] calldata proof
  )
    public
    payable
    whenNotPaused
    nonReentrant
    onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
  {
    uint256 mintPrice = calculateMintPrice();
    uint256 amountMinted = 0;
    uint256 budgetLeft = msg.value;

@>  while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) {
      _mintInternal(msg.sender, mintPrice);
      amountMinted++;
      budgetLeft -= mintPrice;
      mintPrice = calculateMintPrice();
    }
    if (budgetLeft > 0) {
      (bool refundSuccess, ) = msg.sender.call{ value: budgetLeft }('');
      require(refundSuccess, 'Refund failed.');
    }
  }
```

Technically, [`mintWithBudget()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L202-L225) can stop serving its purpose sooner than the whole first generation gets minted because `_tokenIds` get incremented also when forging is done but for the sake of giving a simple example we will focus only on the minting process.

Lets say the first generation can mint `10000` NFTs and after that the generation should increase to 2 where the following `10000` could be minted. Each mint increments `_tokenIds` as it can be seen in [`_mintInternal()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L285).

This means that if we are at generation 2 or 3 for example, [`mintWithBudget()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L202-L225) will never go into the loop since 30000 already minted NFTs (`_tokenIds`) are not less than 10000 (`maxTokensPerGen`).

The function will simply start returning the sent native currency and will no longer mint any NFTs.

## Tools Used
Manual Review

## Recommended Mitigation Steps
If we want to stop the while loop once the `maxTokensPerGen` of the current generation is reached the implementation should be changed to the following:

```diff
function mintWithBudget(
    bytes32[] calldata proof
  )
    public
    payable
    whenNotPaused
    nonReentrant
    onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
  {
    uint256 mintPrice = calculateMintPrice();
    uint256 amountMinted = 0;
    uint256 budgetLeft = msg.value;

-    while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) {
+    while (budgetLeft >= mintPrice && generationMintCounts[currentGeneration] < maxTokensPerGen) {
      _mintInternal(msg.sender, mintPrice);
      amountMinted++;
      budgetLeft -= mintPrice;
      mintPrice = calculateMintPrice();
    }
    if (budgetLeft > 0) {
      (bool refundSuccess, ) = msg.sender.call{ value: budgetLeft }('');
      require(refundSuccess, 'Refund failed.');
    }
  }
```

## Assessed type
Invalid Validation

# [HIGH-03] Users can mint more NFTs per generation because generationMintCounts gets reset

# Lines of code
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345

# Vulnerability details
## Impact
Users can mint more NFTs per generation because `generationMintCounts[currentGeneration]` value gets reset once the current generation gets incremented.

## Proof of Concept
`generationMintCounts` is a mapping that holds the number of NFTs that were minted in a specific generation. This number can be increased during normal mints or via forging.

By using forging users can mint NFTs that are 1+ the current generation and if we are currently in generation 3 by forging two gen3 NFTs we can create new NFT that is gen4. This will increase `generationMintCounts[4]` and will become a non-zero value.

Once the `maxTokensPerGen` for generation 3 is reached [`_incrementGeneration()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355) will get called and it will reset `generationMintCounts[4]` back to 0:

Reference in code: link

```solidity
  function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    currentGeneration++;
@>  generationMintCounts[currentGeneration] = 0;
    priceIncrement = priceIncrement + priceIncrementByGen;
    entropyGenerator.initializeAlphaIndices();
    emit GenerationIncremented(currentGeneration);
  }
```

By resetting the count of the already minted tokens for a specific generation this will allow users to mint 10000 (`maxTokensPerGen`) more NFTs in this exact generation. By the time the protocol reaches the maximum number of generations the minted NFTs could be far more (depending on how many NFTs were forged in the following generations) than the limit. Assuming that the parameters do not change this limit is `maxTokensPerGen` * `maxGeneration`.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Simply remove the following line:

```solidity
  function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    currentGeneration++;
@>  generationMintCounts[currentGeneration] = 0;
    priceIncrement = priceIncrement + priceIncrementByGen;
    entropyGenerator.initializeAlphaIndices();
    emit GenerationIncremented(currentGeneration);
  }
```

# [HIGH-04] initializeAlphaIndices uses an incorrect modifier

# Lines of code
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L206

# Vulnerability details
## Impact
The usage of the `onlyOwner` modifier suggests that `TraitForgeNFT` must be the owner of `EntropyGenerator` to work as intended. If this is true this would block the usage of multiple `EntropyGenerator` functions forever.

## Proof of Concept
[`initializeAlphaIndices()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L206) is called in `TraitForgeNFT` when a generation gets incremented:

Reference in code: [link](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355)

```solidity
  function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    currentGeneration++;
    generationMintCounts[currentGeneration] = 0;
    priceIncrement = priceIncrement + priceIncrementByGen;
@>  entropyGenerator.initializeAlphaIndices();
    emit GenerationIncremented(currentGeneration);
  }
```

This is a core protocol function and for this call to be successful this would mean `TraitForgeNFT` must be the owner of `EntropyGenerator` but this is not feasible since `TraitForgeNFT` does not implement functions that would allow it to pause/unpause, `EntropyGenerator.transferOwnership()`, `EntropyGenerator.renounceOwnership()` or call [`EntropyGenerator.setAllowedCaller()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L36C12-L36C28) if it is indeed the owner of `EntropyGenerator`.

For this reason, I believe that the correct modifier is `onlyAllowedCaller`.

## Tools Used
Manual Review

## Recommended Mitigation Steps
```diff
-  function initializeAlphaIndices() public whenNotPaused onlyOwner {
+  function initializeAlphaIndices() public whenNotPaused onlyAllowedCaller {
    uint256 hashValue = uint256(
      keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp))
    );

    uint256 slotIndexSelection = (hashValue % 258) + 512;
    uint256 numberIndexSelection = hashValue % 13;

    slotIndexSelectionPoint = slotIndexSelection;
    numberIndexSelectionPoint = numberIndexSelection;
  }
```

## Assessed type
Access Control

# [MEDIUM-01] Nuke can burn user NFT without giving any reward

# Lines of code
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153

# Vulnerability details
## Impact
[`nuke()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153) function can burn user NFTs without giving any reward to the user under certain circumstances.

## Proof of Concept
There are certain cases where `claimAmount` could be 0 which would mean that the caller will not get anything in return for calling [`nuke()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153) and his NFT will get burned for nothing.

The most common reason for this could be an NFT having **an** **entropy of 0**. This would make [`calculateAge()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L118) return 0:

Reference in code: [link](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L118)

```solidity
  function calculateAge(uint256 tokenId) public view returns (uint256) {
    require(nftContract.ownerOf(tokenId) != address(0), 'Token does not exist');

    uint256 daysOld = (block.timestamp -
      nftContract.getTokenCreationTimestamp(tokenId)) /
      60 /
      60 /
      24;
@>  uint256 perfomanceFactor = nftContract.getTokenEntropy(tokenId) % 10;

@>  uint256 age = (daysOld * perfomanceFactor * MAX_DENOMINATOR * ageMultiplier) / 365; // add 5 digits for decimals
    return age;
  }
```

Which in turn would make `finalNukeFactor` 0 as well:

Reference in code: [link](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L136)

```solidity
  function calculateNukeFactor(uint256 tokenId) public view returns (uint256) {
    require(
      nftContract.ownerOf(tokenId) != address(0),
      'ERC721: operator query for nonexistent token'
    );

@>  uint256 entropy = nftContract.getTokenEntropy(tokenId); // 0
@>  uint256 adjustedAge = calculateAge(tokenId); // 0

@>  uint256 initialNukeFactor = entropy / 40; // calcualte initalNukeFactor based on entropy, 5 digits
		// 0

@>  uint256 finalNukeFactor = ((adjustedAge * defaultNukeFactorIncrease) /
      MAX_DENOMINATOR) + initialNukeFactor; // 0

    return finalNukeFactor;
  }
```

Finally this would make `claimAmount` to be 0 and since there isn’t any validation that would revert the transaction the NFT will get burned.

Reference in code: [link](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153)

```solidity
  function nuke(uint256 tokenId) public whenNotPaused nonReentrant {
    require(
      nftContract.isApprovedOrOwner(msg.sender, tokenId),
      'ERC721: caller is not token owner or approved'
    );
    require(
      nftContract.getApproved(tokenId) == address(this) ||
        nftContract.isApprovedForAll(msg.sender, address(this)),
      'Contract must be approved to transfer the NFT.'
    );
    require(canTokenBeNuked(tokenId), 'Token is not mature yet');

@>  uint256 finalNukeFactor = calculateNukeFactor(tokenId); // finalNukeFactor has 5 digits
@>  uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR; // Calculate the potential claim amount based on the finalNukeFactor
    uint256 maxAllowedClaimAmount = fund / maxAllowedClaimDivisor; // Define a maximum allowed claim amount as 50% of the current fund size

    // Directly assign the value to claimAmount based on the condition, removing the redeclaration
@>  uint256 claimAmount = finalNukeFactor > nukeFactorMaxParam
      ? maxAllowedClaimAmount
      : potentialClaimAmount;

    fund -= claimAmount; // Deduct the claim amount from the fund

    nftContract.burn(tokenId); // Burn the token
    (bool success, ) = payable(msg.sender).call{ value: claimAmount }('');
    require(success, 'Failed to send Ether');

    emit Nuked(msg.sender, tokenId, claimAmount); // Emit the event with the actual claim amount
    emit FundBalanceUpdated(fund); // Update the fund balance
  }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Make the following changes to [`nuke()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153) in order to prevent cases where user could burn their NFT without getting anything in return.

```diff
  function nuke(uint256 tokenId) public whenNotPaused nonReentrant {
    require(
      nftContract.isApprovedOrOwner(msg.sender, tokenId),
      'ERC721: caller is not token owner or approved'
    );
    require(
      nftContract.getApproved(tokenId) == address(this) ||
        nftContract.isApprovedForAll(msg.sender, address(this)),
      'Contract must be approved to transfer the NFT.'
    );
    require(canTokenBeNuked(tokenId), 'Token is not mature yet');

    uint256 finalNukeFactor = calculateNukeFactor(tokenId); // finalNukeFactor has 5 digits
    uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR; // Calculate the potential claim amount based on the finalNukeFactor
    uint256 maxAllowedClaimAmount = fund / maxAllowedClaimDivisor; // Define a maximum allowed claim amount as 50% of the current fund size

    // Directly assign the value to claimAmount based on the condition, removing the redeclaration
    uint256 claimAmount = finalNukeFactor > nukeFactorMaxParam
      ? maxAllowedClaimAmount
      : potentialClaimAmount;

+   require(claimAmount > 0, "claimAmount must be more than 0");

    fund -= claimAmount; // Deduct the claim amount from the fund

    nftContract.burn(tokenId); // Burn the token
    (bool success, ) = payable(msg.sender).call{ value: claimAmount }('');
    require(success, 'Failed to send Ether');

    emit Nuked(msg.sender, tokenId, claimAmount); // Emit the event with the actual claim amount
    emit FundBalanceUpdated(fund); // Update the fund balance
  }
```

## Assessed type
Invalid Validation

# [MEDIUM-02] Contracts do not implement pause/unpause()

## Impact
All contracts cannot be paused or unpaused in case of emergencies since this implementation is missing.

## Proof of Concept
All contracts inherit Pausable.sol (Openzeppelin contract) and have functions that use the `whenNotPaused` modifier but there isn’t a way for them to be paused because Pausable.sol does not implement `pause()` or `unpause()` external functions since this is left to be implemented by the contracts that inherit it.

Because this implementation is missing in the following contracts the owner will not be able to pause any of the contracts or the transferring of the NFTs in case of emergencies:

* TraitForgeNft
* NukeFund
* EntropyGenerator
* EntityTrading
* EntityForging
* DevFund

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider implementing external `pause()` and `unpause()` functions in each contract that inherits Pausable.sol.

## Assessed type
Other

# [MEDIUM-03] User about to nuke can be front-runned and burn their NFT for far less ETH than expected

# Lines of code
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153-L182

# Vulnerability details
## Impact
Callers of [`nuke()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153-L182) can get front-runned and lose big portion of their rewards and still lose their NFT.

## Proof of Concept
[`nuke()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153-L182) function burns the NFT of the caller and gives him part of the funds the contract is holding based on certain parameters.

Users call [`nuke()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153-L182) when they think the amount of funds they will get are worth burning their NFT.

```solidity
  function nuke(uint256 tokenId) public whenNotPaused nonReentrant {
    require(
      nftContract.isApprovedOrOwner(msg.sender, tokenId),
      'ERC721: caller is not token owner or approved'
    );
    require(
      nftContract.getApproved(tokenId) == address(this) ||
        nftContract.isApprovedForAll(msg.sender, address(this)),
      'Contract must be approved to transfer the NFT.'
    );
    require(canTokenBeNuked(tokenId), 'Token is not mature yet');

    uint256 finalNukeFactor = calculateNukeFactor(tokenId); // finalNukeFactor has 5 digits
    uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR; // Calculate the potential claim amount based on the finalNukeFactor
    uint256 maxAllowedClaimAmount = fund / maxAllowedClaimDivisor; // Define a maximum allowed claim amount as 50% of the current fund size

    // Directly assign the value to claimAmount based on the condition, removing the redeclaration
    uint256 claimAmount = finalNukeFactor > nukeFactorMaxParam
      ? maxAllowedClaimAmount
      : potentialClaimAmount;

    fund -= claimAmount; // Deduct the claim amount from the fund

    nftContract.burn(tokenId); // Burn the token
    (bool success, ) = payable(msg.sender).call{ value: claimAmount }('');
    require(success, 'Failed to send Ether');

    emit Nuked(msg.sender, tokenId, claimAmount); // Emit the event with the actual claim amount
    emit FundBalanceUpdated(fund); // Update the fund balance
  }
```

As it can be seen from the code above, nuke does not have arguments that Alice could use to define how much reward she expects to receive and there isn’t a cooldown period between the [`nuke()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153-L182) calls. This means that Alice could expect to get 1 ETH for example but her call gets front-runned 1 or more times and this would result in her getting significantly less like 0.5 ETH or 0.1 ETH etc.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider implementing slippage arguments or a cooldown period for the [`nuke()`](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153-L182) function in order to prevent users from receiving far less than they expected and still burn their NFT.

## Assessed type
MEV

