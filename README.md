# Issue H-1: Frequency-dependent `TaxCalculator.sol::calculateCompoundedFactor` leads to interest loss either for users or for protocol 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/117 

## Found by 
kuprum
### Summary

[TaxCalculator.sol::calculateCompoundedFactor](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/TaxCalculator.sol#L90) uses discrete formula for calculating the compounded factor. Combined with the wrong divisor (`1000` instead of `10_000`, as I outline in another finding), when collection's utilization factor is moderately high (e.g. 90%), and the operations with the collection happen relatively frequently (e.g. every day), this leads to charging users excessive interest rates: for this example, `3200%` more than expected. 

If the divisor is fixed to the correct value (`10_000`), then the effects from using the discrete formula are less severe, but still quite substantial: namely for a 100% collection utilization, depending of the frequency, either the user will be charged up to $e-2 \approx$ `71%`  more interest per year compared to the non-compound interest, or the protocol will receive up to $1- 1/(e-1) \approx$ `42%` less interest per year compared to the compound interest.

It's worth noting that `calculateCompoundedFactor` is called whenever a new checkpoint is created for the collection, which happens e.g. when _any user_ [creates a listing](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L162), [cancels a listing](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L467), or [fills a listing](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L603). **This is neither enough to reach the desired precision, nor is it gas efficient.**

Depending on the frequency of collection operations, either:

- If non-compound interest is expected, but the frequency is high, then the users will be charged up to `71%` more interest than they expect;
- If compound interest is expected, but the frequency is low, then the protocol will receive up to `42%` less interest than it expects.

### Root Cause

[TaxCalculator.sol::calculateCompoundedFactor](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/TaxCalculator.sol#L90) employs the following formula: 

```solidity
compoundedFactor_ = _previousCompoundedFactor * (1e18 + (perSecondRate / 1000 * _timePeriod)) / 1e18;
```

As I explain in another finding, the divisor `1000` is incorrect, and has to be fixed to `10_000`. Provided this is fixed, the resulting formula is the correct discrete formula for calculating the compounded interest. The problem is that the formula will give vastly different results depending on the frequency of operations which have nothing to do with the user who holds the protected listing.

### Internal pre-conditions

Varying frequency of collection operations.

### External pre-conditions

none

### Attack Path

No attack is necessary. The interest rates will be wrongly calculated in most cases.

### Impact

Either users are charged up to `71%` more interest than they expect, or the protocol receives up to `42%` less interest than it expects.

### PoC

Drop this test to [TaxCalculator.t.sol](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/test/TaxCalculator.t.sol#L86), and execute with `forge test --match-test test_WrongInterestCalculation`

```solidity
   // This test uses the unmodified source code, with the wrong divisor of 1000
   function test_WrongInterestCalculation() public view {
        // We fix collection utilization to 90%
        uint utilization = 0.9 ether;

        // New checkpoints with the updated compoundedFactor are created 
        // and stored whenever there is activity wrt. the collection

        // The expected interest multiplier after 1 year
        uint expectedFactor = 
            taxCalculator.calculateCompoundedFactor(1 ether, utilization, 365 days);

        // The resulting interest multiplier if some activity happens every day
        uint compoundedFactor = 1 ether;
        for (uint time = 0; time < 365 days; time += 1 days) {
            compoundedFactor = 
                taxCalculator.calculateCompoundedFactor(compoundedFactor, utilization, 1 days);
        }

        // The user loss due to the activity which doesn't concern them is 3200%
        assertApproxEqRel(
            33 ether * expectedFactor / 1 ether,
            compoundedFactor,
            0.01 ether);
   }
```

### Mitigation

**Variant 1**: If non-compound interest is desired, apply this diff:

```diff
diff --git a/flayer/src/contracts/TaxCalculator.sol b/flayer/src/contracts/TaxCalculator.sol
index 915c0ff..4031aba 100644
--- a/flayer/src/contracts/TaxCalculator.sol
+++ b/flayer/src/contracts/TaxCalculator.sol
@@ -87,7 +87,7 @@ contract TaxCalculator is ITaxCalculator {
         uint perSecondRate = (interestRate * 1e18) / (365 * 24 * 60 * 60);
 
         // Calculate new compounded factor
-        compoundedFactor_ = _previousCompoundedFactor * (1e18 + (perSecondRate / 1000 * _timePeriod)) / 1e18;
+        compoundedFactor_ = _previousCompoundedFactor + (perSecondRate * _timePeriod / 1000);
     }
 
     /**
```

_Note: in the above the divisor is still unfixed, as this belongs to a different finding._

**Variant 2**: If compound interest is desired, employ either periodic per-second compounding, or continuous compounding with exponentiation. Any of these approaches are precise enough and much more gas efficient than the current one, but require substantial refactoring.

# Issue H-2: ERC1155Bridgable.sol cannot receive ETH royalties 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/140 

## Found by 
Ironsidesec, MohammedRizwan, OpaBatyo, ZanyBonzy, heeze, merlin
## Summary

ERC1155Bridgable.sol cannot receive ETH, so any attempts for royalty sources to send ETH to the contract will fail, and as a result, users cannot claim their ERC1155 royalties.

## Vulnerability Detail
ERC1155Bridgable.sol holds the claimRoyalties function which allows users, through the  `INFERNAL_RIFT_BELOW` to claim their royalties, ETH token or otherwise. However, when dealing with ETH, the contract has no payable receive or fallback function, and as a result cannot receive ETH. Thus, users cannot claim their ETH royalties.

```solidity
    function claimRoyalties(address _recipient, address[] calldata _tokens) external {
        if (msg.sender != INFERNAL_RIFT_BELOW) {
            revert NotRiftBelow();
        }

        // We can iterate through the tokens that were requested and transfer them all
        // to the specified recipient.
        uint tokensLength = _tokens.length;
        for (uint i; i < tokensLength; ++i) {
            // Map our ERC20
            ERC20 token = ERC20(_tokens[i]);

            // If we have a zero-address token specified, then we treat this as native ETH
            if (address(token) == address(0)) {
                SafeTransferLib.safeTransferETH(_recipient, payable(address(this)).balance);
            } else {
                SafeTransferLib.safeTransfer(token, _recipient, token.balanceOf(address(this)));
            }
        }
    }
```
## Impact

Contracts cannot receive ETH, and as a result, users cannot claim their royalties, leading to loss of funds.

Add the test code below to RiftTest.t.sol, and run it with `forge test --mt test_bridged1155CannotReceiveETH -vvvv`

```solidity
    function test_bridged1155CannotReceiveETH() public {

        l1NFT1155.mint(address(this), 0, 1);
        l1NFT1155.setApprovalForAll(address(riftAbove), true);
        address[] memory collections = new address[](1);
        collections[0] = address(l1NFT1155);

        uint[][] memory idList = new uint[][](1);
        uint[] memory ids = new uint[](1);
        ids[0] = 0;
        idList[0] = ids;

        uint[][] memory amountList = new uint[][](1);
        uint[] memory amounts = new uint[](1);
        amounts[0] = 1;
        amountList[0] = amounts;

        mockPortalAndMessenger.setXDomainMessenger(address(riftAbove));
        riftAbove.crossTheThreshold1155(
            _buildCrossThreshold1155Params(collections, idList, amountList, address(this), 0)
        );

        Test1155 l2NFT1155 = Test1155(riftBelow.l2AddressForL1Collection(address(l1NFT1155), true));
        address RoyaltyProvider = makeAddr("RoyaltyProvider");
        vm.deal(RoyaltyProvider, 10 ether);
        vm.expectRevert();
        vm.prank(RoyaltyProvider);
        (bool success, ) = address(l2NFT1155).call{value: 10 ether}("");
        assert(success);
        vm.stopPrank();
    }
```
The test passes because we are expecting a reversion with EvmError as the contract cannot receive ETH. Hence there's no ETH for the users to claim.

```md
    ├─ [0] VM::expectRevert(custom error f4844814:)
    │   └─ ← [Return] 
    ├─ [0] VM::prank(RoyaltyProvider: [0x5D4FfD958F2bfe55BfC8B0602A8C066E2D7eeBa8])
    │   └─ ← [Return] 
    ├─ [201] 0x094bb35C5C8E23F2A873541aDb8c5e464C29c668::fallback{value: 10000000000000000000}()
    │   ├─ [45] ERC1155Bridgable::fallback{value: 10000000000000000000}() [delegatecall]
    │   │   └─ ← [Revert] EvmError: Revert
    │   └─ ← [Revert] EvmError: Revert
    ├─ [0] VM::stopPrank()
``` 
## Code Snippet

https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/moongate/src/libs/ERC1155Bridgable.sol#L116C1-L135C6

## Tool used

Manual Review

## Recommendation

Add a payable receive function to the contract.

# Issue H-3: Quorum overflow in `CollectionShutdown` leads to complete drain of contract's funds 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/146 

## Found by 
0xc0ffEE, Audinarey, Ragnarok, Ruhum, almantare, araj, asui, blockchain555, h2134, kuprum, zzykxx
### Summary

[CollectionShutdown.sol#150](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L150) and [CollectionShutdown.sol#L247](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L247) cast quorum votes to `uint88` as follows:

```solidity
uint totalSupply = params.collectionToken.totalSupply();
if (totalSupply > MAX_SHUTDOWN_TOKENS * 10 ** params.collectionToken.denomination()) revert TooManyItems();


// Set our quorum vote requirement
params.quorumVotes = uint88(totalSupply * SHUTDOWN_QUORUM_PERCENT / ONE_HUNDRED_PERCENT);
```

The problem with the above is that it may overflow:
- `collectionToken.denomination()` may max `9`
- `MAX_SHUTDOWN_TOKENS == 4`
- `SHUTDOWN_QUORUM_PERCENT / ONE_HUNDRED_PERCENT == 1/2`
- Collection tokens are minted in `Locker` [as follows](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Locker.sol#L163):
  - `token.mint(_recipient, tokenIdsLength * 1 ether * 10 ** token.denomination());`

E.g. with `totalSupply == 0.6190 ether * 10**9`, we have that the check still passes, but:
- `totalSupply * SHUTDOWN_QUORUM_PERCENT / ONE_HUNDRED_PERCENT = 0.3095 ether * 10**9`
- `type(uint88).max ~= 0.309485 ether * 10**9`
- `uint88(0.3095 ether * 10**9) ~= 0.000015 ether * 10**9`

Upon collection shutdown, tokens are sold on Sudoswap, and the funds thus obtained are distributed among the claimants. The claimed amount is then _divided by `quorumVotes`_ [as follows](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L310):

```solidity
uint amount = params.availableClaim * claimableVotes / (params.quorumVotes * ONE_HUNDRED_PERCENT / SHUTDOWN_QUORUM_PERCENT);
```

Thus, when dividing by a much smaller `quorumVotes`, the claimant receives much more than they are eligible for: in the PoC it's `20647 ether` though only `1 ether` has been received from sales.

### Root Cause

[CollectionShutdown.sol#150](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L150) and [CollectionShutdown.sol#L247](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L247) downcast the quorum votes to `uint88`, which may overflow.

### Internal pre-conditions

1. The collection token denomination needs to be sufficiently large to cause overflow
2. The amount of shutdown votes needs to be sufficiently large to cause overflow.

### External pre-conditions

none

### Attack Path

1. A user creates a collection with denomination `9`
2. The user holding `0.6190 ether * 10**9` of the collection token (i.e. less than 1 NFT) starts collection shutdown.
    - At that point the quorum of votes overflows, and becomes much smaller
4. Collection shutdown is executed normally.
5. Tokens are sold on Sudoswap. 
     - In the PoC they are sold for `1 ether`.
7. User claims the balance. Due to the overflow, they receive much more than what their NFTs were worth.
     - In the PoC user receives `20647 ether` though only `1 ether` has been received from sales.

### Impact

The protocol suffers unbounded losses (the whole balance of `CollectionShutdown` contract can be drained.

### PoC

Drop this test to [CollectionShutdown.t.sol](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/test/utils/CollectionShutdown.t.sol#L51) and execute with `forge test --match-test test_QuorumOverflow`:

```solidity
function test_QuorumOverflow() public {
    locker.createCollection(address(erc721c), 'Test Collection', 'TEST', 9);

    // Initialize our collection, without inflating `totalSupply` of the {CollectionToken}
    locker.setInitialized(address(erc721c), true);

    // Set our collection token for ease for reference in tests
    collectionToken = locker.collectionToken(address(erc721c));

    // Approve our shutdown contract to use test suite's tokens
    collectionToken.approve(address(collectionShutdown), type(uint).max);

    // Give some initial balance to CollectionShutdown contract
    vm.deal(address(collectionShutdown), 30000 ether);
    
    vm.startPrank(address(locker));
    // Suppose address(1) holds 0.6190 ether
    // in a collection token with denomination 9
    collectionToken.mint(address(1), 0.6190 ether * 10**9);
    vm.stopPrank();

    // Start collection shutdown from address(1)
    vm.startPrank(address(1));
    collectionToken.approve(address(collectionShutdown), 0.6190 ether * 10**9);
    collectionShutdown.start(address(erc721c));
    vm.stopPrank();

    // Mint NFTs into our collection {Locker} and process the execution
    uint[] memory tokenIds = _mintTokensIntoCollection(erc721c, 3);
    collectionShutdown.execute(address(erc721c), tokenIds);

    // Mock the process of the Sudoswap pool liquidating the NFTs for ETH.
    vm.startPrank(SUDOSWAP_POOL);
    // Transfer the specified tokens away from the Sudoswap position to simulate a purchase
    for (uint i; i < tokenIds.length; ++i) {
        erc721c.transferFrom(SUDOSWAP_POOL, address(5), i);
    }
    // Ensure the sudoswap pool has enough ETH to send
    deal(SUDOSWAP_POOL, 1 ether);
    // Send ETH from the Sudoswap Pool into the {CollectionShutdown} contract
    (bool sent,) = payable(address(collectionShutdown)).call{value: 1 ether}('');
    require(sent, 'Failed to send {CollectionShutdown} contract');
    vm.stopPrank();

    // Get our start balances so that we can compare to closing balances from claim
    uint startBalanceAddress = payable(address(1)).balance;

    // address(1) now can claim
    collectionShutdown.claim(address(erc721c), payable(address(1)));

    // Due to quorum overflow, address(1) now holds ~ 20647 ether
    assertApproxEqRel(payable(address(1)).balance - startBalanceAddress, 20647 ether, 0.01 ether);
}
```

### Mitigation

Employ the appropriate type and cast for `quorumVotes`, e.g. `uint92`.

# Issue H-4: There is a calculation error inside the calculateCompoundedFactor() function, causing users to overpay interest. 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/227 

## Found by 
0x37, Sentryx, Tendency, ZeroTrust, dany.armstrong90, dimulski, kuprum, merlinboii, robertodf, stuart\_the\_minion
## Summary
There is a calculation error inside the calculateCompoundedFactor() function, causing users to overpay interest.
## Vulnerability Detail
```javascript
    function calculateProtectedInterest(uint _utilizationRate) public pure returns (uint interestRate_) {
        // If we haven't reached our kink, then we can just return the base fee
        if (_utilizationRate <= UTILIZATION_KINK) {
            // Calculate percentage increase for input range 0 to 0.8 ether (2% to 8%)
@>>            interestRate_ = 200 + (_utilizationRate * 600) / UTILIZATION_KINK;
        }
        // If we have passed our kink value, then we need to calculate our additional fee
        else {
            // Convert value in the range 0.8 to 1 to the respective percentage between 8% and
            // 100% and make it accurate to 2 decimal places.
            interestRate_ = (((_utilizationRate - UTILIZATION_KINK) * (100 - 8)) / (1 ether - UTILIZATION_KINK) + 8) * 100;
        }
    }
```

```javascript
 function calculateCompoundedFactor(uint _previousCompoundedFactor, uint _utilizationRate, uint _timePeriod) public view returns (uint compoundedFactor_) {
        // Get our interest rate from our utilization rate
@>>        uint interestRate = this.calculateProtectedInterest(_utilizationRate);

        // Ensure we calculate the compounded factor with correct precision. `interestRate` is
        // in basis points per annum with 1e2 precision and we convert the annual rate to per
        // second rate.
       uint perSecondRate = (interestRate * 1e18) / (365 * 24 * 60 * 60);

        // Calculate new compounded factor
@>>         compoundedFactor_ = _previousCompoundedFactor * (1e18 + (perSecondRate / 1000 * _timePeriod)) / 1e18;
    }
```
Through the calculateProtectedInterest() function, which calculates the annual interest rate, we know that 200 represents 2% and 800 represents 8%, so the decimal precision is 4.
 When the principal is 100 and the annual interest rate is 2% (200), the yearly interest should be calculated as 100 * 200 / 10000 = 2. 

 However, in the calculateCompoundedFactor function, there is a clear error when calculating compound interest, as it only divides by 1000, leading to the interest being multiplied by a factor of 10.


## Impact
The user overpaid interest, resulting in financial loss.
## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/TaxCalculator.sol#L80

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/TaxCalculator.sol#L59C1-L71C6

## Tool used

Manual Review

## Recommendation
```diff
 function calculateCompoundedFactor(uint _previousCompoundedFactor, uint _utilizationRate, uint _timePeriod) public view returns (uint compoundedFactor_) {
        // Get our interest rate from our utilization rate
        uint interestRate = this.calculateProtectedInterest(_utilizationRate);

        // Ensure we calculate the compounded factor with correct precision. `interestRate` is
        // in basis points per annum with 1e2 precision and we convert the annual rate to per
        // second rate.
       uint perSecondRate = (interestRate * 1e18) / (365 * 24 * 60 * 60);

        // Calculate new compounded factor
-         compoundedFactor_ = _previousCompoundedFactor * (1e18 + (perSecondRate / 1000 * _timePeriod)) / 1e18;
+         compoundedFactor_ = _previousCompoundedFactor * (1e18 + (perSecondRate / 10000 * _timePeriod)) / 1e18;
    }
```

# Issue H-5: `_listing` mapping not deleted when calling `Listings::reserve` can lead to a token being sold when it shouldn't be for sale 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/252 

## Found by 
0x37, 0xAlix2, 0xHappy, 0xNirix, BugPull, Feder, McToady, Ruhum, almantare, araj, blockchain555, h2134, jecikpo, kuprum, merlinboii, stuart\_the\_minion, utsav, valuevalk, zzykxx
## Summary
When the `reserve` function is called in `Listings` a protected listing is created for the selected `_tokenId`, however while doing this the original `_listing` mapping for this token is not deleted in `Listings`. While the protected listing is active this is not a problem because `getListingPrice` will check the protected listing before reaching the old listing. However if the protected listing is removed this old listing will become active again. 

This becomes especially problemative given the `ProtectedListings::unlockProtectedListing` function allows the user to not withdraw the token immediately, meaning the token will be in the `Locker` and can now be sold to a third party calling `Listings::fillListings`.

## Vulnerability Detail
Consider the following steps:
1. User A lists token via `Listings::createListings`
2. User B creates a reserve on the token calling `Listings::reserve`
3. Later when User B is ready to fully pay off the token they call `ProtectedListings::unlockProtectedListing` however they set `_withdraw == false` as they will choose to withdraw the token later.
4. Now User C will be able to gain the token via the original (still active) listing by calling `Listings::fillListings`

Add the following test to `Listings.t.sol` to highlight this issue:
```solidity
    function test_Toad_abuseOldListing() public {
        // Get user A token
        address userA = makeAddr("userA");
        vm.deal(userA, 1 ether);
        uint256 _tokenId = 1199;
        erc721a.mint(userA, _tokenId);

        vm.startPrank(userA);
        erc721a.approve(address(listings), _tokenId);

        Listings.Listing memory listing = IListings.Listing({
            owner: payable(userA),
            created: uint40(block.timestamp),
            duration: VALID_LIQUID_DURATION,
            floorMultiple: 120
        });
        _createListing({
            _listing: IListings.CreateListing({
                collection: address(erc721a),
                tokenIds: _tokenIdToArray(_tokenId),
                listing: listing
            })
        });

        vm.stopPrank();

        // User B calls Listings::reserve
        address userB = makeAddr("userB");
        uint256 startBalance = 10 ether;
        ICollectionToken token = locker.collectionToken(address(erc721a));
        deal(address(token), userB, startBalance);

        vm.warp(block.timestamp + 10);
        vm.startPrank(userB);
        token.approve(address(listings), startBalance);
        token.approve(address(protectedListings), startBalance);

        uint256 listingCountStart = listings.listingCount(address(erc721a));
        console.log("Listing count start", listingCountStart);

        listings.reserve({
            _collection: address(erc721a),
            _tokenId: _tokenId,
            _collateral: 0.2 ether
        });

        uint256 listingCountAfterReserve = listings.listingCount(address(erc721a));
        console.log("Listing count after reserve", listingCountAfterReserve);

        // User B later calls ProtectedListings::unlockProtectedListing with _withdraw == false
        vm.warp(block.timestamp + 1 days);
        protectedListings.unlockProtectedListing(address(erc721a), _tokenId, false);
        vm.stopPrank();

        // Other user now calls Listings::fillListings using User As listing data and gets the token
        address userC = makeAddr("userC");
        deal(address(token), userC, startBalance);
        vm.startPrank(userC);
        token.approve(address(listings), startBalance);

        uint[][] memory tokenIdsOut = new uint[][](1);
        tokenIdsOut[0] = new uint[](1);
        tokenIdsOut[0][0] = _tokenId;


        Listings.FillListingsParams memory fillListings = IListings.FillListingsParams({
            collection: address(erc721a),
            tokenIdsOut: tokenIdsOut
        });
        listings.fillListings(fillListings);
        vm.stopPrank();

        // Confirm User C is now owner of the token
        assertEq(erc721a.ownerOf(_tokenId), userC);

        // Confirm User C spent 1.2 collection tokens to buy
        uint256 endBalanceUserC = token.balanceOf(userC);
        console.log("User C End Bal", endBalanceUserC);

        // Confirm User B has lost ~1.2 collection tokens
        uint256 endBalanceUserB = token.balanceOf(userB);
        console.log("User B End Bal", endBalanceUserB);

        // Confirm User A has been paid difference between listingPrice & floor twice (approx 1.4x floor despite listing a 1.2)
        uint256 endBalanceUserA = token.balanceOf(userA);
        uint256 escrowBalUserA = listings.balances(userA, address(token));
        console.log("User A End Bal", endBalanceUserA);
        console.log("EscBal User A ", escrowBalUserA);

        // Confirm listing count decremented twice causes underflow
        uint256 listingCountEnd = listings.listingCount(address(erc721a));
        console.log("Listing count end", listingCountEnd);
    }
```

## Impact
This series of actions has the following effects on the users involved:
- User A gets paid the difference between their listing price and floor price twice (during both User B and User C's purchases)
- User B pays the full price of the token from User A but does not get the NFT
- User C pays the full price of the token from User A and gets the NFT

Additionally during this process `listingCount[_collection]` gets decremented twice, potentially leading to an underflow as the value is changed in an `unchecked` block. This incorrect internal accounting can later cause issues if `CollectionShutdown` attempts to sunset a collection as it's `hasListings` check when sunsetting a collection will be incorrect, potentially sunsetting a collection while listings are still live.

## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L690

## Tool used

Manual Review

## Recommendation
Just as in `_fillListing` and `cancelListing` functions, when `reserve` is call the existing `_listing` mapping for the specified `_tokenId` should be deleted as so:
```diff
    function reserve(address _collection, uint _tokenId, uint _collateral) public nonReentrant lockerNotPaused {
        // -- Snip --
        
        // Check if the listing is a floor item and process additional logic if there
        // was an owner (meaning it was not floor, so liquid or dutch).
        if (oldListing.owner != address(0)) {
            // We can process a tax refund for the existing listing if it isn't a liquidation
            if (!_isLiquidation[_collection][_tokenId]) {
                (uint _fees,) = _resolveListingTax(oldListing, _collection, true);
                if (_fees != 0) {
                    emit ListingFeeCaptured(_collection, _tokenId, _fees);
                }
            }

            // If the floor multiple of the original listings is different, then this needs
            // to be paid to the original owner of the listing.
            uint listingFloorPrice = 1 ether * 10 ** collectionToken.denomination();
            if (listingPrice > listingFloorPrice) {
                unchecked {
                    collectionToken.transferFrom(msg.sender, oldListing.owner, listingPrice - listingFloorPrice);
                }
            }

+           delete _listings[_collection][_tokenId]

            // Reduce the amount of listings
            unchecked { listingCount[_collection] -= 1; }
        }
        
        // -- Snip --
    }
```

This will ensure that even if the token's new protected listing is removed the stale listing will not be accessible.



# Issue H-6: The Users who voted for collection shutdown will lose their collection tokens by cancelling the shutdown 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/261 

## Found by 
0xAlix2, 0xHappy, 0xc0ffEE, Audinarey, Aymen0909, BADROBINX, BugPull, Hearmen, Limbooo, McToady, Ragnarok, almantare, araj, asui, blockchain555, cawfree, ctf\_sec, dany.armstrong90, g, merlinboii, onthehunt, steadyman, stuart\_the\_minion, utsav, ydlee, zzykxx
## Summary

When a user votes for collection shutdown, the `CollectionShutdown` contract gathers the whole balance from the user. However, when cancelling the shutdown process, the contract doesn't refund the user's votes.

## Vulnerability Detail

The [`CollectionShutdown::_vote()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L191-L214) function gathers the whole balance of the collection token from a voter.

```solidity
    function _vote(address _collection, CollectionShutdownParams memory params) internal returns (CollectionShutdownParams memory) {
        uint userVotes = params.collectionToken.balanceOf(msg.sender);
        if (userVotes == 0) revert UserHoldsNoTokens();

        // Pull our tokens in from the user
        params.collectionToken.transferFrom(msg.sender, address(this), userVotes);
        ... ...
    }
```

But in the [`CollectionShutdown::cancel()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L390-L405) function, it does not refund the voters tokens.

```solidity
    function cancel(address _collection) public whenNotPaused {
        // Ensure that the vote count has reached quorum
        CollectionShutdownParams memory params = _collectionParams[_collection];
        if (!params.canExecute) revert ShutdownNotReachedQuorum();

        // Check if the total supply has surpassed an amount of the initial required
        // total supply. This would indicate that a collection has grown since the
        // initial shutdown was triggered and could result in an unsuspected liquidation.
        if (params.collectionToken.totalSupply() <= MAX_SHUTDOWN_TOKENS * 10 ** locker.collectionToken(_collection).denomination()) {
            revert InsufficientTotalSupplyToCancel();
        }

        // Remove our execution flag
        delete _collectionParams[_collection];
        emit CollectionShutdownCancelled(_collection);
    }
```

### Proof-Of-Concept

Here is the testcase of the POC:

To bypass the [total supply vs shutdown votes restriction](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L398-L400), added the following line to the test case:

```solidity
    collectionToken.mint(address(10), _additionalAmount);
```

The whole test case is:

```solidity
    function test_CancelShutdownNotRefund() public withQuorumCollection {
        uint256 _additionalAmount = 1 ether;
        // Confirm that we can execute with our quorum-ed collection
        assertCanExecute(address(erc721b), true);

        vm.prank(address(locker));
        collectionToken.mint(address(10), _additionalAmount); 

        // Cancel our shutdown
        collectionShutdown.cancel(address(erc721b));

        // Now that we have cancelled the shutdown process, we should no longer
        // be able to execute the shutdown.
        assertCanExecute(address(erc721b), false);

        console.log("Address 1 balance after:", collectionToken.balanceOf(address(1)));
        console.log("Address 2 balance after:", collectionToken.balanceOf(address(2)));
    }
```

Here are the logs after running the test:
```bash
$ forge test --match-test test_CancelShutdownNotRefund -vv
[⠒] Compiling...
[⠃] Compiling 1 files with Solc 0.8.26
[⠊] Solc 0.8.26 finished in 8.81s
Compiler run successful!

Ran 1 test for test/utils/CollectionShutdown.t.sol:CollectionShutdownTest
[PASS] test_CancelShutdownNotRefund() (gas: 390566)
Logs:
  Address 1 balance after: 0
  Address 2 balance after: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.29s (454.80µs CPU time)

Ran 1 test suite in 8.29s (8.29s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

As can be seen from the logs, the voters(1, 2) were not refunded their tokens.

## Impact

Shutdown Voters will be ended up losing their whole collection tokens by cancelling the shutdown.

## Code Snippet

[utils/CollectionShutdown.sol#L390-L405](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L390-L405)

## Tool used

Manual Review

## Recommendation

The problem can be fixed by implementing following:

1. Add new state variable to the contract that records all voters

```solidity
    address[] public votersList;
```

2. Update the `_vote()` function like below:
```diff
    function _vote(address _collection, CollectionShutdownParams memory params) internal returns (CollectionShutdownParams memory) {
        ... ...
        // Register the amount of votes sent as a whole, and store them against the user
        params.shutdownVotes += uint96(userVotes);

        // Register the amount of votes for the collection against the user
+       if (shutdownVoters[_collection][msg.sender] == 0)
+           votersList.push(msg.sender);
        unchecked { shutdownVoters[_collection][msg.sender] += userVotes; }
        ... ...
    }
```

3. Add the new code section to the `reclaimVote()` function, that removes the sender from the `votersList`.

4. Update the `cancel()` function like below:
```diff
    function cancel(address _collection) public whenNotPaused {
        ... ...
        if (params.collectionToken.totalSupply() <= MAX_SHUTDOWN_TOKENS * 10 ** locker.collectionToken(_collection).denomination()) {
            revert InsufficientTotalSupplyToCancel();
        }

+       uint256 i;
+       uint256 votersLength = votersList.length;
+       for (; i < votersLength; i ++) {
+           params.collectionToken.transfer(
+               votersList[i], 
+               shutdownVoters[_collection][votersList[i]]
+           );
+       }

        // Remove our execution flag
        delete _collectionParams[_collection];
+       delete votersList;
        emit CollectionShutdownCancelled(_collection);
    }
```

After running the testcase on the above update, the user voters are able to get their own votes:

```bash
$ forge test --match-test test_CancelShutdownNotRefund -vv
[⠒] Compiling...
[⠘] Compiling 3 files with Solc 0.8.26
[⠃] Solc 0.8.26 finished in 8.70s
Compiler run successful!

Ran 1 test for test/utils/CollectionShutdown.t.sol:CollectionShutdownTest
[PASS] test_CancelShutdownNotRefund() (gas: 486318)
Logs:
  Address 1 balance after: 1000000000000000000
  Address 2 balance after: 1000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.46s (526.80µs CPU time)

Ran 1 test suite in 3.46s (3.46s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```



# Issue H-7: User can unlock protected listing without paying any fee. 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/269 

## Found by 
0x37, OpaBatyo, dany.armstrong90, g, jecikpo, onthehunt, zzykxx
## Summary
`ProtectedListings.adjustPosition()` function adjust `listing.tokenTaken` without considering compounded factor. Exploiting this vulnerability, user can unlock protected listing without paying any fee.


## Vulnerability Detail
`ProtectedListings.adjustPosition()` function is following.
```solidity
    function adjustPosition(address _collection, uint _tokenId, int _amount) public lockerNotPaused {
        // Ensure we don't have a zero value amount
        if (_amount == 0) revert NoPositionAdjustment();

        // Load our protected listing
        ProtectedListing memory protectedListing = _protectedListings[_collection][_tokenId];

        // Make sure caller is owner
        if (protectedListing.owner != msg.sender) revert CallerIsNotOwner(protectedListing.owner);

        // Get the current debt of the position
        int debt = getProtectedListingHealth(_collection, _tokenId);

        // Calculate the absolute value of our amount
        uint absAmount = uint(_amount < 0 ? -_amount : _amount);

        // cache
        ICollectionToken collectionToken = locker.collectionToken(_collection);

        // Check if we are decreasing debt
        if (_amount < 0) {
            // The user should not be fully repaying the debt in this way. For this scenario,
            // the owner would instead use the `unlockProtectedListing` function.
            if (debt + int(absAmount) >= int(MAX_PROTECTED_TOKEN_AMOUNT)) revert IncorrectFunctionUse();

            // Take tokens from the caller
            collectionToken.transferFrom(
                msg.sender,
                address(this),
                absAmount * 10 ** collectionToken.denomination()
            );

            // Update the struct to reflect the new tokenTaken, protecting from overflow
399:        _protectedListings[_collection][_tokenId].tokenTaken -= uint96(absAmount);
        }
        // Otherwise, the user is increasing their debt to take more token
        else {
            // Ensure that the user is not claiming more than the remaining collateral
            if (_amount > debt) revert InsufficientCollateral();

            // Release the token to the caller
            collectionToken.transfer(
                msg.sender,
                absAmount * 10 ** collectionToken.denomination()
            );

            // Update the struct to reflect the new tokenTaken, protecting from overflow
413:        _protectedListings[_collection][_tokenId].tokenTaken += uint96(absAmount);
        }

        emit ListingDebtAdjusted(_collection, _tokenId, _amount);
    }
```
As can be seen in `L399` and `L413`, `_protectedListings[_collection][_tokenId].tokenTaken` is updated without considering compounded factor. Exploiting this vulnerability, user can unlock protected listing without paying any fee.

PoC:
Add the following test code into `ProtectedListings.t.sol`.
```solidity
    function test_adjustPositionError() public {
        erc721a.mint(address(this), 0);
        
        erc721a.setApprovalForAll(address(protectedListings), true);

        uint[] memory _tokenIds = new uint[](2); _tokenIds[0] = 0; _tokenIds[1] = 1;

        IProtectedListings.CreateListing[] memory _listings = new IProtectedListings.CreateListing[](1);
        _listings[0] = IProtectedListings.CreateListing({
            collection: address(erc721a),
            tokenIds: _tokenIdToArray(0),
            listing: IProtectedListings.ProtectedListing({
                owner: payable(address(this)),
                tokenTaken: 0.4 ether,
                checkpoint: 0
            })
        });
        protectedListings.createListings(_listings);

        vm.warp(block.timestamp + 7 days);

        // unlock protected listing for tokenId = 0
        assertEq(protectedListings.unlockPrice(address(erc721a), 0), 402055890410801920);
        locker.collectionToken(address(erc721a)).approve(address(protectedListings), 0.4 ether);
        protectedListings.adjustPosition(address(erc721a), 0, -0.4 ether);
        assertEq(protectedListings.unlockPrice(address(erc721a), 0), 0);
        protectedListings.unlockProtectedListing(address(erc721a), 0, true);
    }
```
In the above test code, we can see that `unlockPrice(address(erc721a), 0)` is `402055890410801920`, but after calling `adjustPosition(address(erc721a), 0, -0.4 ether)`, `unlockPrice(address(erc721a), 0)` decreases to `0`. So we unlocked protected listing paying only `0.4 ether` without paying any fee.

## Impact
User can unlock protected listing without paying any fee. It means loss of funds for the protocol.
On the other hand, if user increase `tokenTaken` in `adjustPosition()` function, increasement of fee will be inflated by compounded factor. it means loss of funds for the user.

## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L366-L417

## Tool used

Manual Review

## Recommendation
Adjust `tokenTaken` considering compounded factor in `ProtectedListings.adjustPosition()` function. That is, divide `absAmount` by compounded factor before updating `tokenTaken`.

# Issue H-8: InfernalRiftBelow.thresholdCross verify the wrong msg.sender 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/405 

## Found by 
Ali9955, Tendency, ZeroTrust, novaman33

## Summary

`InfernalRiftBelow.thresholdCross` verify the wrong `msg.sender`, `thresholdCross` will fail to be called, resulting in the loss of user assets.

## Vulnerability Detail

`thresholdCross` determines whether `msg.sender` is `expectedAliasedSender`:

```solidity
    address expectedAliasedSender = address(uint160(INFERNAL_RIFT_ABOVE) + uint160(0x1111000000000000000000000000000000001111));
    // Ensure the msg.sender is the aliased address of {InfernalRiftAbove}
    if (msg.sender != expectedAliasedSender) {
        revert CrossChainSenderIsNotRiftAbove();
    }
```

but in fact the function caller should be `RELAYER_ADDRESS`,
In `sudoswap`, `crossTheThreshold` check whether `msg.sender` is `RELAYER_ADDRESS`:
https://github.com/sudoswap/InfernalRift/blob/7696827b3221929b3fa563692bd4c5d73b20528e/src/InfernalRiftBelow.sol#L56


L1 across chain message through the `PORTAL.depositTransaction`, rather than `L1_CROSS_DOMAIN_MESSENGER`.

To avoid confusion, use in L1 should all `L1_CROSS_DOMAIN_MESSENGER.sendMessage` to send messages across the chain, avoid the use of low level `PORTAL. depositTransaction` function.

```solidity
 function crossTheThreshold(ThresholdCrossParams memory params) external payable {
        ......
        // Send package off to the portal
        PORTAL.depositTransaction{value: msg.value}(
            INFERNAL_RIFT_BELOW,
            0,
            params.gasLimit,
            false,
            abi.encodeCall(InfernalRiftBelow.thresholdCross, (package, params.recipient))
        );

        emit BridgeStarted(address(INFERNAL_RIFT_BELOW), package, params.recipient);
    }
```

## Impact
When transferring nft across chains,`thresholdCross` cannot be called in L2, resulting in loss of user assets.

## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/moongate/src/InfernalRiftBelow.sol#L135-L145

## Tool used

Manual Review

## Recommendation
```solidity
    // Validate caller is cross-chain
    if (msg.sender != RELAYER_ADDRESS) { //or L2_CROSS_DOMAIN_MESSENGER
        revert NotCrossDomainMessenger();
    }

    // Validate caller comes from {InfernalRiftBelow}
    if (ICrossDomainMessenger(msg.sender).xDomainMessageSender() != InfernalRiftAbove) {
        revert CrossChainSenderIsNotRiftBelow();
    }
```



## Discussion

**zhaojio**

We asked sponsor:
> InfernalRiftBelow.claimRoyalties function of the msg.sender, is RELAYER_ADDRESS? right??

Sponsor reply:
> good catch, it should be L2_CROSS_DOMAIN_MESSENGER instead

# Issue H-9: InfernalRiftBelow.claimRoyalties no verification msg.sender 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/406 

## Found by 
Ironsidesec, Thanos, X12, ZeroTrust, cnsdkc007, ctf\_sec, gr8tree, rndquu, shaflow01, snapishere, t.aksoy, zzykxx



## Summary

`claimRoyalties` used to accept the message on the L1, then execute `ERC721Bridgable.claimRoyalties`, but not the authentication callers, may result in the loss of the assets in the protocol.

## Vulnerability Detail

`claimRoyalties` are used to accept cross-chain calls, but `msg.sender` is not validated:

```solidity
    if (ICrossDomainMessenger(msg.sender).xDomainMessageSender() != INFERNAL_RIFT_ABOVE) {
        revert CrossChainSenderIsNotRiftAbove();
    }
```

If `msg.sender` is a contract account implementing the `ICrossDomainMessenger` interface, the `xDomainMessageSender` function returns an address of `INFERNAL_RIFT_ABOVE`, which means that the `claimRoyalties` function can be invoked.

So anyone can call this function as long as he deploys a contract.

`InfernalRiftBelow.claimRoyalties` function will be called `ERC721Bridgable.claimRoyalties`, transfer NFTs from `ERC721Bridgable` contract, so any can transfer NFTs from `ERC721Bridgable`. 

L1 Send message to L2:

```solidity
    function claimRoyalties(address _collectionAddress, address _recipient, address[] calldata _tokens, uint32 _gasLimit) external {
        .....
        ICrossDomainMessenger(L1_CROSS_DOMAIN_MESSENGER).sendMessage(
            INFERNAL_RIFT_BELOW,
            abi.encodeCall(
                IInfernalRiftBelow.claimRoyalties,
                (_collectionAddress, _recipient, _tokens)
            ),
            _gasLimit
        );

        emit RoyaltyClaimStarted(address(INFERNAL_RIFT_BELOW), _collectionAddress, _recipient, _tokens);
    }

```
## Impact
Anyone can steal NFT from the protocol.

## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/moongate/src/InfernalRiftBelow.sol#L220-L232

## Tool used

Manual Review

## Recommendation
```diff
+    if (msg.sender != L2_CROSS_DOMAIN_MESSENGER) {
+        revert NotCrossDomainMessenger();
+    }
```


# Issue H-10: ERC1155 cannot claim royalities on L2. 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/456 

## Found by 
0xdice91, BugPull, OpaBatyo, Ruhum, Thanos, h2134, heeze, novaman33, zzykxx
## Summary

https://github.com/sherlock-audit/2024-08-flayer/blob/main/moongate/src/InfernalRiftBelow.sol#L220-L232

The royalty claim function is designed to allow owners of collections deployed on L2 to claim their royalties on L1. 
However, this function only supports collections using the ERC721 standard. If the collection is an ERC1155, 
the function reverts due to a check in isDeployedOnL2, preventing the owner from claiming their royalties.

## Vulnerability Detail
The function claimRoyalties checks if a collection is deployed on L2 using the isDeployedOnL2 function. This check only passes if the collection is an ERC721 standard. When an ERC1155 collection is used as the _collectionAddress, the function reverts because the check fails. 
As a result, owners of ERC1155 collections are unable to claim their royalties.

NOTE: The ERC1155Bridgable contract implements claimRoyalty function: https://github.com/sherlock-audit/2024-08-flayer/blob/main/moongate/src/libs/ERC1155Bridgable.sol#L104-L135

POC:
```solidity 
// on L1
function claimRoyalties(address _collectionAddress, address _recipient, address[] calldata _tokens, uint32 _gasLimit) external {
      //...
@>>>  ICrossDomainMessenger(L1_CROSS_DOMAIN_MESSENGER).sendMessage(
          INFERNAL_RIFT_BELOW,
          abi.encodeCall(IInfernalRiftBelow.claimRoyalties, (_collectionAddress, _recipient, _tokens)),
          _gasLimit
      );
      //...
}

// on L2
function claimRoyalties(address _collectionAddress, address _recipient, address[] calldata _tokens) public {
        // Ensure that our message is sent from the L1 domain messenger
        if (ICrossDomainMessenger(msg.sender).xDomainMessageSender() != INFERNAL_RIFT_ABOVE) {
            revert CrossChainSenderIsNotRiftAbove();
        }

        // Get our L2 address from the L1
@>>>    // revert will happen here because passing ERC1155 _collectionAddress with false to isDeployedOnL2
        // will check if ERC721Bridgable is deployed not ERC1155Bridgable.
        // so calling claimRoyalties will cause revert.
@>>>    if (!isDeployedOnL2(_collectionAddress, false)) revert L1CollectionDoesNotExist();

        // Call our ERC721Bridgable contract as the owner to claim royalties to the recipient
        ERC721Bridgable(l2AddressForL1Collection(_collectionAddress, false)).claimRoyalties(_recipient, _tokens);
        emit RoyaltyClaimFinalized(_collectionAddress, _recipient, _tokens);
}
```

## Impact
the inability of ERC1155 owners to claim royalties results in a loss of expected income for the collection owners. 

## Recommendation

add logic to handle ERC1155 _collectionAddress.

```diff
function claimRoyalties(address _collectionAddress, address _recipient, address[] calldata _tokens) public {
    // Ensure that our message is sent from the L1 domain messenger
    if (ICrossDomainMessenger(msg.sender).xDomainMessageSender() != INFERNAL_RIFT_ABOVE) {
        revert CrossChainSenderIsNotRiftAbove();
    }

    // Get our L2 address from the L1
+   if (isDeployedOnL2(_collectionAddress, false)) {
+       // Call our ERC721Bridgable contract as the owner to claim royalties to the recipient
+       ERC721Bridgable(l2AddressForL1Collection(_collectionAddress, false)).claimRoyalties(_recipient, _tokens);
+       emit RoyaltyClaimFinalized(_collectionAddress, _recipient, _tokens);
+   } 

+   else if (isDeployedOnL2(_collectionAddress, true)) {
+       // Call our ERC1155Bridgable contract as the owner to claim royalties to the recipient
+       ERC1155Bridgable(l2AddressForL1Collection(_collectionAddress, true)).claimRoyalties(_recipient, _tokens);
+       emit RoyaltyClaimFinalized(_collectionAddress, _recipient, _tokens);
+   }

+   else {
+       revert L1CollectionDoesNotExist();
+   }
}
```


# Issue H-11: Creation of listings does not accurately reflect the utilization rate, which could lead to loss of interest 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/530 

## Found by 
Tendency, blockchain555, valuevalk
## Summary
The `createListings()` function in both `Listings.sol` and `ProtectedListing.sol` has a flaw that prevents accurate checkpoint creation.

## Vulnerability Detail
The main vulnerability is in `ProtectedListing.sol`, where we can call `createListings()` and specify an array of `CreateListing` that we want to create:

```solidity
  function createListings(CreateListing[] calldata _createListings) public nonReentrant lockerNotPaused {
```

The issue lies in the logic that ensures `_createCheckpoint` is only called once per collection. While this is intended, the problem occurs because this action happens **before** all the changes that may affect the **utilization rate**.

```solidity
  function createListings(CreateListing[] calldata _createListings) public nonReentrant lockerNotPaused {
  ....Skip Code....
        for (uint i; i < _createListings.length; ++i) {
            CreateListing calldata listing = _createListings[i];
            _validateCreateListing(listing);

@>>         checkpointKey = keccak256(abi.encodePacked('checkpointIndex', listing.collection));
            assembly { checkpointIndex := tload(checkpointKey) }
@>>      if (checkpointIndex == 0) {
@>>             checkpointIndex = _createCheckpoint(listing.collection);
                assembly { tstore(checkpointKey, checkpointIndex) }
            }

            tokensIdsLength = listing.tokenIds.length;
            tokensReceived = _mapListings(listing, tokensIdsLength, checkpointIndex) * 10 ** 
locker.collectionToken(listing.collection).denomination();

            unchecked {
@>>             listingCount[listing.collection] += tokensIdsLength;
            }

@>>         _depositNftsAndReceiveTokens(listing, tokensReceived);
        }
```            

The `_createCheckpoint` **_must be called after the listing is added to the `listingCount[]` and after the changes to the total supply of CT_**, which in this case occurs when `_depositNftsAndReceiveTokens` is called. This function mints new `CT`, impacting the total supply. The total supply directly affects the **utilization rate**, which is used to calculate the compound interest rate.

However, in the current logic, `_createCheckpoint` is called **before** changes that affect the **utilization rate**, the `listingCount[]`, and the `totalSupply`:

```solidity
    function utilizationRate(address _collection)  {
@>>     listingsOfType_ = listingCount[_collection];
        if (listingsOfType_ != 0) {
            ICollectionToken collectionToken = locker.collectionToken(_collection);
            uint totalSupply = collectionToken.totalSupply();
            if (totalSupply != 0) {
@>>             utilizationRate_ = (listingsOfType_ * 1e36 * 10 ** collectionToken.denomination()) / totalSupply;
            }
        }
    }
```

## Impact
This behavior contradicts the required functionality as stated in the comments in both [Listings.cancelListings()](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/Listings.sol#L466-L467) and [Listings.fillListings()](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/Listings.sol#L602-L603):

```solidity
@>> // Create our checkpoint as utilization rates will change
@>> protectedListings.createCheckpoint(listing.collection);
```
And in [ProtectedListings.sol](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L480-L481)
```solidity
@>>  // Update our checkpoint to reflect that listings have been removed
 @>> _createCheckpoint(_collection);
 ```
 
**Root Causes:**
- **`createCheckpoint` is only called for the first created listing** of a specific collection, not the last one. This means the calculation of the new **checkpoint compoundingFactor** is incorrect, as **utilization rate changes** occur after the checkpoint is created. The increase in **listing count** is not reflected in the checkpoint.
- **`createCheckpoint` is executed before the minting of `CT`**, which means the total supply and utilization rate are not correctly updated at the time of checkpoint creation.

As a result, this can also lead to **lower interest rates**, which may accrue over time, reducing the overall accuracy of the protocol's interest calculation.

## Tool Used
**Manual Review**

## Recommendation
Since a user may list multiple collections via `ProtectedListing.sol` or `Listings.sol` through `createListings()`, but for different collections, the best approach is to maintain an array `address[] collections;` to track the **unique collections**. After the loops are executed, perform another loop to call `_createCheckpoint(listing.collection)` for each unique collection.

# Issue H-12: The `relist` function does not check whether the listing is a liquidation listing causing users to pay taxes and refunds being paid to the listing owner who did not pay taxes 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/547 

## Found by 
0x37, 0xAlix2, 0xNirix, BADROBINX, BugPull, Ollam, Sentryx, Tendency, almantare, almurhasan, araj, blockchain555, h2134, kuprum, merlinboii, t.aksoy, utsav, valuevalk, zzykxx
## Summary
Since liquidated listings are created without the original owner paying taxes, the lack of a check for the liquidation status means the system might incorrectly process a tax refund for a listing that was liquidated, allowing the original owner to receive funds they should not be entitled to
## Vulnerability Detail
Liquidated listings are created without the original owner paying taxes. If the system lacks a check for liquidation status, it might process a tax refund incorrectly, The absence of a liquidation status check can lead to the original owner receiving tax refunds for liquidated listings, which they are not entitled to. This creates an opportunity for the owner to receive funds improperly

The problem arises because the `relist` function does not check if the Listing being relisted was from a liquidation or not, therefore causing the "relister" to pay taxes and the contract depositing refunds to the "liquidated user" who did not pay taxes.
## Impact
Without checking liquidation status, the system may wrongly process tax refunds for liquidated listings, allowing the original owner to receive undeserved funds. This error could result in a loss of funds within the protocol.

## Code Snippet
- https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L644
<details>
<summary>Whole Function</summary>

```solidity
    function relist(CreateListing calldata _listing, bool _payTaxWithEscrow) public nonReentrant lockerNotPaused {
  // Load our tokenId
  address _collection = _listing.collection;
  uint _tokenId = _listing.tokenIds[0];

  // Read the existing listing in a single read
  Listing memory oldListing = _listings[_collection][_tokenId];

  // Ensure the caller is not the owner of the listing
  if (oldListing.owner == msg.sender) revert CallerIsAlreadyOwner();

  // Load our new Listing into memory
  Listing memory listing = _listing.listing;

  // Ensure that the existing listing is available
  (bool isAvailable, uint listingPrice) = getListingPrice(_collection, _tokenId);
  if (!isAvailable) revert ListingNotAvailable();

  // We can process a tax refund for the existing listing
@>(uint _fees,) = _resolveListingTax(oldListing, _collection, true);
  if (_fees != 0) {
    emit ListingFeeCaptured(_collection, _tokenId, _fees);
  }

  // Find the underlying {CollectionToken} attached to our collection
  ICollectionToken collectionToken = locker.collectionToken(_collection);

  // If the floor multiple of the original listings is different, then this needs
  // to be paid to the original owner of the listing.
  uint listingFloorPrice = 1 ether * 10 ** collectionToken.denomination();
  if (listingPrice > listingFloorPrice) {
    unchecked {
      collectionToken.transferFrom(msg.sender, oldListing.owner, listingPrice - listingFloorPrice);
    }
  }

  // Validate our new listing
  _validateCreateListing(_listing);

  // Store our listing into our Listing mappings
  _listings[_collection][_tokenId] = listing;

  // Pay our required taxes
  payTaxWithEscrow(address(collectionToken), getListingTaxRequired(listing, _collection), _payTaxWithEscrow);

  // Emit events
  emit ListingRelisted(_collection, _tokenId, listing);
}
```

</details>

## Tool used

Manual Review

## Recommendation
Add a check inside the `relist` Function to prevent taxes being paid for listings that were liquidated
```diff
  // We can process a tax refund for the existing listing
+  if (!_isLiquidation[_collection][_tokenId]) {
  (uint _fees,) = _resolveListingTax(oldListing, _collection, true);
  if (_fees != 0) {
    emit ListingFeeCaptured(_collection, _tokenId, _fees);
      }
+  } else {
+      delete _isLiquidation[_collection][_tokenId];
+     }
```

# Issue H-13: When relisting a floor item listing, listingCount is not increased, causing listingCount can be underflowed. 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/574 

## Found by 
araj, g, utsav, zraxx
### Summary

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L625-L672
The `relist` function is used to refill the parameters of any token listings, including floor item listing. For a loor item listing, users can change its owner from address(0) to their own address without paying any fees, and set its type to DUTCH or LIQUID. However, this operation does not change `listingCount`. So, when all the listing is filled, the `listingCount` will be underflowed to a large num(close to uint256.max) instead of 0.  Later, in the function `CollectionShutdown.sol#execute`, it will be reverted as `listings.listingCount(_collection) != 0`.

### Root Cause

In `Listings.sol#relist`, when relist for a floor item listing, listingCount is not incresed, causing listingCount can be underflowed.

### Internal pre-conditions

1. The `_collection` is initialized.
2. There are floor item listings in the `_collection`.

### External pre-conditions

_No response_

### Attack Path

1. The attacker call `relist` for a floor item listing.
2. The attacker immediately calls cancelListings to cancel it, in order to refund the tax. At the same time, listingCount is decreased. As a result, the listingCount cannot correctly reflect the number of listings contained in _collection.

### Impact

listingCount cannot correctly reflect the number of listings contained in _collection.
`CollectionShutdown.sol#execute` will be DOSed.

### PoC

_No response_

### Mitigation

Avoid relisting floor item listings 
OR
Increase listingCount by one.

# Issue H-14: Owner Can Lose The Token After Being Unlocked but Not Withdrawn 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/601 

## Found by 
0x37, 0xAlix2, BADROBINX, Greese, Ironsidesec, adamn, almantare, araj, cnsdkc007, dany.armstrong90, h2134, merlinboii, t.aksoy, utsav, zarkk01, zzykxx
## Summary

The `unlockProtectedListing` function allows a user to unlock an NFT and make it withdrawable for themselves. However, if the user does not immediately withdraw the NFT, another user can front-run the process by swapping, redeeming, or buying the NFT, as there is no protection mechanism preventing this. The `Locker::isListing` function fails to correctly identify the unlocked NFT as being in a protected state, allowing it to be redeemed.

## Vulnerability Detail

When the unlockProtectedListing function is called, the NFT owner can either withdraw the NFT or leave it in the contract for later withdrawal. However If the NFT is not withdrawn immediately, a malicious user can swap, redeem, or buy the unlocked NFT. 

Inside the unlockProtectedListing function, if the owner opts not to withdraw the NFT, the listing owner is set to zero and the NFT becomes withdrawable later by the rightful owner. 
```solidity
        // Delete the listing objects
        delete _protectedListings[_collection][_tokenId];

        // Transfer the listing ERC721 back to the user
        if (_withdraw) {
            locker.withdrawToken(_collection, _tokenId, msg.sender);
            emit ListingAssetWithdraw(_collection, _tokenId);
        } else {
            canWithdrawAsset[_collection][_tokenId] = msg.sender;
        }
```
However, due to this owner field being set to zero, the `Locker::isListing function` incorrectly interprets the NFT as no longer being protected, allowing other users to redeem or buy the unlocked NFT before the rightful owner can withdraw it.

```solidity
    function isListing(address _collection, uint _tokenId) public view returns (bool) {
        IListings _listings = listings;

        // Check if we have a liquid or dutch listing
        if (_listings.listings(_collection, _tokenId).owner != address(0)) {
            return true;
        }

        // Check if we have a protected listing
        if (_listings.protectedListings().listings(_collection, _tokenId).owner != address(0)) {
            return true;
        }

        return false;
    }
```


Test:
```solidity
    function test_redeemBeforeOner(uint _tokenId, uint96 _tokensTaken) public {

        _assumeValidTokenId(_tokenId);
        vm.assume(_tokensTaken >= 0.1 ether);
        vm.assume(_tokensTaken <= 1 ether - protectedListings.KEEPER_REWARD());

        // Set the owner to one of our test users (Alice)
        address payable _owner = users[0];

        // Mint our token to the _owner and approve the {Listings} contract to use it
        erc721a.mint(_owner, _tokenId);

        // Create our listing
        vm.startPrank(_owner);
        erc721a.approve(address(protectedListings), _tokenId);
        _createProtectedListing({
            _listing: IProtectedListings.CreateListing({
                collection: address(erc721a),
                tokenIds: _tokenIdToArray(_tokenId),
                listing: IProtectedListings.ProtectedListing({
                    owner: _owner,
                    tokenTaken: _tokensTaken,
                    checkpoint: 0
                })
            })
        });

        // Approve the ERC20 token to be used by the listings contract to unlock the listings
        locker.collectionToken(address(erc721a)).approve(
            address(protectedListings),
            _tokensTaken
        );

        protectedListings.unlockProtectedListing(
            address(erc721a),
            _tokenId,
            false
        );

        vm.stopPrank();

        // Approve the ERC20 token to be used by the listings contract to unlock the listings
        locker.collectionToken(address(erc721a)).approve(
            address(locker),
            1000000000000 ether
        );
        //@audit another user can redeem before owner.
        uint[] memory redeemTokenIds = new uint[](1);
        redeemTokenIds[0] = _tokenId;
        locker.redeem(address(erc721a), redeemTokenIds);

        vm.prank(_owner);
        // Owner cant withdraw anymore since they have been redeemed
        protectedListings.withdrawProtectedListing(address(erc721a), _tokenId);
    }
```

## Impact

original owner might lose the NFT to a malicious actor who front-runs the withdrawal process

## Code Snippet

https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L314
https://github.com/sherlock-audit/2024-08-
flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/Locker.sol#L438

## Tool used

Manual Review

## Recommendation

Modify isListing function: Ensure that tokens marked as canWithdrawAsset are still considered active listings until they are fully withdrawn by their rightful owner.

# Issue H-15: Donation fees are sandwichable in one transaction 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/615 

## Found by 
0x37, BugPull, Ironsidesec, araj
## Summary
Doesn't matters if MEV is openly possible in the chain, when eevr a user does actions like `liquidateProtectedListing`, `cancelListing`, `modifyListings`, `fillListings`, `Reserve` and `Relist`. They can sandwich the fees donation to make profit or recover the tax they paid. Or, the liquidaton is open access, so you can sandwich that in same tc itself.

Root casue : allowing more than max donation limit per transaction.

Even if you don't allow to donate more than max, the user will just loop the donation and still extract the $, so its better to have a max donation per block state that tracks this. So `donateThresholdMax` should be implemented in per block limit way.

## Vulnerability Detail

Uniswap openly advises to design the donation mechanism to not allow MEV extraction

https://github.com/Uniswap/v4-core/blob/18b223cab19dc778d9d287a82d29fee3e99162b0/src/interfaces/IPoolManager.sol#L167-L172

```SOLIDITY
 IPoolManager.sol
    /// @notice Donate the given currency amounts to the in-range liquidity providers of a pool
    /// @dev Calls to donate can be frontrun adding just-in-time liquidity, with the aim of receiving a portion donated funds.
    /// Donors should keep this in mind when designing donation mechanisms.
    /// @dev This function donates to in-range LPs at slot0.tick. In certain edge-cases of the swap algorithm, the `sqrtPrice` of
    /// a pool can be at the lower boundary of tick `n`, but the `slot0.tick` of the pool is already `n - 1`. In this case a call to
    /// `donate` would donate to tick `n - 1` (slot0.tick) not tick `n` (getTickAtSqrtPrice(slot0.sqrtPriceX96)).
```

Fees are donated to uniV4 pool in several listing actions
- `liquidateProtectedListing` : Amount worth `1 ether - listing.tokenTaken - KEEPER_REWARD` is donated. This amount will be huge in cases where, someone listed a protected listing and took 0.5 ether as token taken, but didnot unlock the listing. So since utilization rate became high, the listing heath gone negative and was put to liquidation during this time an amount f (1 ether - 0.5 ether taker - 0.05 ether keeper reward) = 0.40 ether is donated to pool. Thats a 1000$ direct donation
- Other 5 flows of Listings contract, such as `cancelListing`, `modifyListings`, `fillListings`, `Reserve` and `Relist` donate the tax fees to the pool. The likelihood is above medium to have huge amount when most users do multiple arrays of tokens of multiple collections done in one transaction.


Issue flow :
1. Figure out the tick where the donated fee will go in according to the @notice comment above on `IPoolManager.donate`.
2. And provide the in-range liquidity so heavy ( like 100x than available in the whole range of that pool).
3. Then do the `liquidateProtectedListing`, `cancelListing`, `modifyListings`, `fillListings`, `Reserve` and `Relist` actions that donate fees worth doing this sandwich
4. Then trigger a fee donation happening on `before swap` hook and then remove the provided liquidity in the first step.

You don't need to frontrun other user's actions, Just do this sandwiching whenever a liquidation of someone's listing happens. And you can also recover the paid tax/fees of your listings by this MEV. Or eevn better if chain allows to have public mempool, then every user's action is sandwichable.


https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/implementation/UniswapImplementation.sol#L335-L345

https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/implementation/BaseImplementation.sol#L58-L62

```solidity
UniswapImplementation.sol

32:     /// Prevents fee distribution to Uniswap V4 pools below a certain threshold:
33:     /// - Saves wasted calls that would distribute less ETH than gas spent
34:     /// - Prevents targetted distribution to sandwich rewards
35:     uint public donateThresholdMin = 0.001 ether;
36:  >> uint public donateThresholdMax = 0.1 ether; 


336:     function _distributeFees(PoolKey memory _poolKey) internal {
 ---- SNIP ----
365:         if (poolFee > 0) {
366:             // Determine whether the currency is flipped to determine which is the donation side
367:             (uint amount0, uint amount1) = poolParams.currencyFlipped ? (uint(0), poolFee) : (poolFee, uint(0));
368:   >>>       BalanceDelta delta = poolManager.donate(_poolKey, amount0, amount1, '');
369: 
370:             // Check the native delta amounts that we need to transfer from the contract
371:             if (delta.amount0() < 0) {
372:                 _pushTokens(_poolKey.currency0, uint128(-delta.amount0()));
373:             }
374: 
375:             if (delta.amount1() < 0) {
376:                 _pushTokens(_poolKey.currency1, uint128(-delta.amount1()));
377:             }
378: 
379:             emit PoolFeesDistributed(poolParams.collection, poolFee, 0);
380:         }
 ---- SNIP ----

402:     }
403: 

```


## Impact
Loss of funds to the LPs. The fees they about to get due to LP, can be sandwiched and extracted. So high severity, and above medium likelihood even in the chans that doesn't have mempool. So, high severity.


## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/implementation/UniswapImplementation.sol#L335-L345

https://github.com/Uniswap/v4-core/blob/18b223cab19dc778d9d287a82d29fee3e99162b0/src/interfaces/IPoolManager.sol#L167-L172

https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/implementation/BaseImplementation.sol#L58-L62

## Tool used

Manual Review

## Recommendation
Introduce a new way to track how much is donated on this block and limit it on evrery `_donate` call. example, allow only 0.1 ether per block

# Issue H-16: The health of a ```ProtectedListing``` is incorrectly calculated if the ```tokenTaken``` has be changed through ```ProtectedListings::adjustPosition()```. 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/689 

## Found by 
0x73696d616f, 0xNirix, ZeroTrust, dimulski, t.aksoy, zarkk01
### Summary

```ProtectedListings::adjustPosition()``` will change the ```tokenTaken``` but after this, ```ProtectedListings::getProtectedListingHealth()``` will assume that these ```tokenTaken``` has been the same as the starting ones and will incorrectly compound them.

### Root Cause

A user can create a ```ProtectedListing``` by calling ```ProtectedListings::createPosition()```, deposit his token and take back an amount ```tokenTake``` as debt. This amount is supposed to be compounded through the time depending on the newest compound factor and the compound factor when the position was created. However, when the user calls ```ProtectedListings::adjustPosition()``` and take some more debt, this new debt will be assume it will be there from the start and it will be compounded as such in ```ProtectedListings::getProtectedListingHealth()``` while this is not the case and it will be compounded from the moment it got taken. Let's have a look on ```ProtectedListings::adjustPosition()``` :
```solidity
    function adjustPosition(address _collection, uint _tokenId, int _amount) public lockerNotPaused {
        // ...

        // Get the current debt of the position
@>        int debt = getProtectedListingHealth(_collection, _tokenId);

        // ...

        // Check if we are decreasing debt
        if (_amount < 0) {
            // ...

            // Update the struct to reflect the new tokenTaken, protecting from overflow
@>            _protectedListings[_collection][_tokenId].tokenTaken -= uint96(absAmount);
        }
        // Otherwise, the user is increasing their debt to take more token
        else {
           // ...

            // Update the struct to reflect the new tokenTaken, protecting from overflow
@>            _protectedListings[_collection][_tokenId].tokenTaken += uint96(absAmount);
        }

        // ...
    }
```
[Link to code](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L366C1-L417C6)

As we can see, this function just increases or decreases the ```tokenTaken``` of this ```ProtectedListings``` meaning the debt that the owner should repay. Now, if we see the ```ProtectedListings::getProtectedListingHealth()```, we will understand that function just takes the ```tokenTaken``` and compounds it without caring **when** this ```tokenTaken``` increased or decreased :
```solidity
    function getProtectedListingHealth(address _collection, uint _tokenId) public view listingExists(_collection, _tokenId) returns (int) {
        // So we start at a whole token, minus: the keeper fee, the amount of tokens borrowed
        // and the amount of collateral based on the protected tax.
@>        return int(MAX_PROTECTED_TOKEN_AMOUNT) - int(unlockPrice(_collection, _tokenId));
    }

    function unlockPrice(address _collection, uint _tokenId) public view returns (uint unlockPrice_) {
        // Get the information relating to the protected listing
        ProtectedListing memory listing = _protectedListings[_collection][_tokenId];

        // Calculate the final amount using the compounded factors and principle amount
        unlockPrice_ = locker.taxCalculator().compound({
@>            _principle: listing.tokenTaken,
            _initialCheckpoint: collectionCheckpoints[_collection][listing.checkpoint],
            _currentCheckpoint: _currentCheckpoint(_collection)
        });
    }
```
[Link to code](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L497)

So, it will take this ```tokenTaken``` and it will compound it from the start of the ```ProtectedListing``` until now, while it shouldn't be the case since some debt may have been added later (through ```ProtectedListings::adjustPosition()```) and in this way this amount must be compounded from the moment it got taken until now, not from the start.

### Internal pre-conditions

1. User creates a ```ProtectedListing``` through ```ProtectedListings::createListings()```.

### External pre-conditions

1. User wants take some debt on his ```ProtectedListing```.

### Attack Path

1. Firstly, user calls ```ProtectedListings::createListings()``` and creates a ```ProtectedListing``` with ```tokenTaken``` and expecting them to compound.
2. After some time and the initial ```tokenTaken``` have been compounded a bit, user decides to take some more debt and increase ```tokenTaken``` and calls ```ProtectedListings::adjustListing()``` to take ```x``` more debt.
3. Now, ```ProtectedListings::getProtectedListingHealth()``` shows that the ```x``` more debt is compounded like it was taken from the very start of the ```ProtectedListing``` creation and in this way his debt is unfairly more inflated.

### Impact

The impact of this serious vulnerability is that the user is being in more debt than what he should have been, since he accrues interest for a period that he had not actually taken that debt. In this way, while he expects his ```tokenTaken``` to be increased by ```x``` amount (as it would be fairly happen), he sees his debt to be inflated by ```x compounded```. This can cause instant and unfair liquidations and **loss of funds** for users unfairly taken into more debt.

### PoC

No PoC needed.

### Mitigation

To mitigate this vulnerability successfully, consider updating the checkpoints of the ```ProtectedListing``` whenever an adjustment is happening in the ```position```, so the debt to be compounded correctly.

# Issue H-17: Incorrect index handling in checkpoint creation leads to incorrect initial checkpoint retrieval and potential DoS 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/732 

## Found by 
0x37, ComposableSecurity, Ironsidesec, KungFuPanda, McToady, Ollam, Tendency, blockchain555, dany.armstrong90, g, h2134, heeze, merlinboii, stuart\_the\_minion, zzykxx
## Summary

In the current implementation of [`ProtectedListings::_createCheckpoint()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L530-L571), when multiple listings are created for the same collection at the same timestamp, the existing checkpoint is updated, and no new checkpoint is pushed. 

However, the function incorrectly returns the wrong index for this case leads to incorrect index referencing during subsequent listing creations.

## Vulnerability Detail

When a checkpoint is created at the same timestamp, the existing checkpoint is updated, and no new checkpoint is pushed.

[ProtectedListings::_createCheckpoint()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L530-L571)
```solidity
File: ProtectedListings.sol
530:     function _createCheckpoint(address _collection) internal returns (uint index_) {
531:         // Determine the index that will be created
532:         index_ = collectionCheckpoints[_collection].length;
---
559:         // Get our new (current) checkpoint
560:         Checkpoint memory checkpoint = _currentCheckpoint(_collection);
561: 
562:         // If no time has passed in our new checkpoint, then we just need to update the
563:         // utilization rate of the existing checkpoint.
564:@>         if (checkpoint.timestamp == collectionCheckpoints[_collection][index_ - 1].timestamp) {
565:@>             collectionCheckpoints[_collection][index_ - 1].compoundedFactor = checkpoint.compoundedFactor;
566:@>             return index_;
567:         }
---
571:     }
```

However, the current implementation returns the wrong index for this case, causing incorrect checkpoint handling for new listing creations, especially when creating multiple listings for the same collection with different variations.

[ProtectedListings::createListings()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L117-L156)
```solidity
File: ProtectedListings.sol
116:      */
117:     function createListings(CreateListing[] calldata _createListings) public nonReentrant lockerNotPaused {
---
134:             checkpointKey = keccak256(abi.encodePacked('checkpointIndex', listing.collection));
135:             assembly { checkpointIndex := tload(checkpointKey) }
136:             if (checkpointIndex == 0) {
137:                 checkpointIndex = _createCheckpoint(listing.collection);
138:                 assembly { tstore(checkpointKey, checkpointIndex) }
139:             }
---
143:             tokensReceived = _mapListings(listing, tokensIdsLength, checkpointIndex) * 10 ** locker.collectionToken(listing.collection).denomination();
---
156:     }
```
An edge case arises when a new listing is created for a collection that has no checkpoints (`collectionCheckpoints[_collection].length == 0`). 

Assuming `erc721b` has no existing checkpoints (length = 0):
* Creating 2 `CreateListing`s for the same collection (`erc721b`) with different variants should result in only one checkpoint being created.
* In the first iteration, the [`_createCheckpoint()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L530-L571) returns `0` as the index, stores it in `checkpointIndex`, and updates the transient storage at the `checkpointKey` slot. The listing is then stored with the current checkpoint.

[ProtectedListings::createListings()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L117-L156)
```solidity
File: ProtectedListings.sol
116:      */
117:     function createListings(CreateListing[] calldata _createListings) public nonReentrant lockerNotPaused {
---
134:             checkpointKey = keccak256(abi.encodePacked('checkpointIndex', listing.collection));
135:             assembly { checkpointIndex := tload(checkpointKey) }
136:@>           if (checkpointIndex == 0) {
137:@>               checkpointIndex = _createCheckpoint(listing.collection);
138:@>               assembly { tstore(checkpointKey, checkpointIndex) }
139:             }
---
143:             tokensReceived = _mapListings(listing, tokensIdsLength, checkpointIndex) * 10 ** locker.collectionToken(listing.collection).denomination();
---
156:     }
```
* In the second iteration, since `checkpointKey` stores `0`, [`_createCheckpoint()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L530-L571) is triggered again and returns `1` (the length of checkpoints) even though no new checkpoint was pushed.

As a result, the second iteration incorrectly references index `1`, even though the checkpoint only exists at index `0` (with a length of 1). This causes incorrect indexing for the listings.

## Impact

Incorrect index returns lead to the wrong initial checkpoint index for new listings, causing incorrect checkpoint retrieval and utilization. This can result in inaccurate data and potential out-of-bound array access, leading to a Denial of Service (DoS) in [`ProtectedListings.unlockPrice()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L607-L617)

## Code Snippet

[ProtectedListings::_createCheckpoint()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L530-L571)
```solidity
File: ProtectedListings.sol
530:     function _createCheckpoint(address _collection) internal returns (uint index_) {
531:         // Determine the index that will be created
532:         index_ = collectionCheckpoints[_collection].length;
---
559:         // Get our new (current) checkpoint
560:         Checkpoint memory checkpoint = _currentCheckpoint(_collection);
561: 
562:         // If no time has passed in our new checkpoint, then we just need to update the
563:         // utilization rate of the existing checkpoint.
564:@>         if (checkpoint.timestamp == collectionCheckpoints[_collection][index_ - 1].timestamp) {
565:@>             collectionCheckpoints[_collection][index_ - 1].compoundedFactor = checkpoint.compoundedFactor;
566:@>             return index_;
567:         }
---
571:     }
```

[ProtectedListings::createListings()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L117-L156)
```solidity
File: ProtectedListings.sol
116:      */
117:     function createListings(CreateListing[] calldata _createListings) public nonReentrant lockerNotPaused {
---
134:             checkpointKey = keccak256(abi.encodePacked('checkpointIndex', listing.collection));
135:             assembly { checkpointIndex := tload(checkpointKey) }
136:@>           if (checkpointIndex == 0) {
137:@>               checkpointIndex = _createCheckpoint(listing.collection);
138:@>               assembly { tstore(checkpointKey, checkpointIndex) }
139:             }
---
143:             tokensReceived = _mapListings(listing, tokensIdsLength, checkpointIndex) * 10 ** locker.collectionToken(listing.collection).denomination();
---
156:     }
```

[ProtectedListings::unlockPrice()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L607-L617)
```solidity
File: ProtectedListings.sol
607:     function unlockPrice(address _collection, uint _tokenId) public view returns (uint unlockPrice_) {
608:         // Get the information relating to the protected listing
609:         ProtectedListing memory listing = _protectedListings[_collection][_tokenId];
610: 
611:         // Calculate the final amount using the compounded factors and principle amount
612:         unlockPrice_ = locker.taxCalculator().compound({
613:             _principle: listing.tokenTaken,
614:             _initialCheckpoint: collectionCheckpoints[_collection][listing.checkpoint],
615:             _currentCheckpoint: _currentCheckpoint(_collection)
616:         });
617:     }
```

## Tool used

Manual Review

## Recommendation

Update the return value of the `ProtectedListings::_createCheckpoint()` to return `index_ - 1` when the checkpoint is updated at the same timestamp to ensure that subsequent listings reference the correct index.

```diff
function _createCheckpoint(address _collection) internal returns (uint index_) {
    // Determine the index that will be created
    index_ = collectionCheckpoints[_collection].length;
---
    // If no time has passed in our new checkpoint, then we just need to update the
    // utilization rate of the existing checkpoint.
    if (checkpoint.timestamp == collectionCheckpoints[_collection][index_ - 1].timestamp) {
        collectionCheckpoints[_collection][index_ - 1].compoundedFactor = checkpoint.compoundedFactor;
-        return index_;
+        return (index_ - 1);
    }
---
}
```

# Issue H-18: The attacker will prevent eligible users from claiming the liquidated balance 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/742 

## Found by 
0x37, 0xc0ffEE, BADROBINX, OpaBatyo, Ruhum, ZeroTrust, almantare, araj, asui, dany.armstrong90, merlinboii, utsav, zzykxx
### Summary

The `CollectionShutdown` contract has vulnerabilities allowing a malicious actor to prevent eligible users from claiming the liquidated balance after liquidation by `SudoSwap`.

### Root Cause

* The [`CollectionShutdown::vote()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L175-L181) does not prevent voting after the collection shutdown is executed and/or during the claim state, allowing malicious actors to trigger `canExecute` to `TRUE` after execution.
* The [`CollectionShutdown::cancel()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L390-L405) does not use `params.collectionToken` to retrieve the `denomination()` for validating the total supply during cancellation, which opens the door to manipulations that can bypass the checks.

### Internal pre-conditions

1. The collection token total supply must be within a valid limit for the shutdown condition (e.g., less than or equal to `MAX_SHUTDOWN_TOKENS`).
2. The `denomination` of the collection token for the shutdown collection is greater than 0.

### External pre-conditions

1. The attacker holds some portion of the collection token supply for the shutdown collection.

### Attack Path

#### Pre-condition: 
1. Assume the collection token (CT) total supply is 4 CTs (`4 * 1e18 * 10 ** denom`).
2. There are 2 holders of this supply: **Lewis (2 CTs)** and **Max (2 CTs)**.

#### Attack:
1. Lewis notices that the collection can be shutdown and calls `CollectionShutdown::start()`.
    * `totalSupply` meets the condition `<= MAX_SHUTDOWN_TOKENS`.
    * `params.quorumVotes` = 50% of totalSupply = `2 * 1e18 * 1eDenom` (2 CTs).
    * Vote for Lewis is recorded.
    * The contract transfer 2 CTs of Lewis balances, and `params.shutdownVotes += 2 CTs`.
    * Now `params.canExecute` is flagged to be `TRUE` since `params.shutdownVotes (2CTs) >= params.quorumVotes (2 CTs)`.

2. Time passes, no cancellation occurs, and the owner executes the pending shutdown.
    * The NFTs are liquidated on SudoSwap.
    * `params.quorumVotes` remains the same as there is no change in supply.
    * The collection is sunset in the `Locker`, deleting `_collectionToken[_collection]` and `collectionInitialized[_collection]`.
    * `params.canExecute` is flagged back to `FALSE`.

**After some or all NFTs are sold on SudoSwap:**

3. Max monitors the NFT sales and prepares for the attack.
4. Max splits their balance of CTs to his another wallet and remains holding a small amount to perform the attack.
5. Max, who never voted, calls `CollectionShutdown::vote()` to **flag `params.canExecute` back to `TRUE`**.
    * The contract transfer small amount of CTs of Max balances.
    * Since `params.shutdownVotes >= params.quorumVotes` (due to Lewis' shutdown), `params.canExecute` is set back to `TRUE`.

6. Max registers the target collection again, manipulating the token's `denomination` via the `Locker::createCollection()`.
    * Max specifies a `denomination` lower than the previous one (e.g., previously 4, now 0).
    
7. Max invokes `CollectionShutdown::cancel()` to remove all properties of `_collectionParams[_collection]`, including `_collectionParams[].availableClaim`.
    * The following check passes:
    ```solidity
    File: CollectionShutdown.sol
    398:         if (params.collectionToken.totalSupply() <= MAX_SHUTDOWN_TOKENS * 10 ** locker.collectionToken(_collection).denomination()) {
    399:             revert InsufficientTotalSupplyToCancel();
    400:         }
    ```
    * Since the new denomination is 0, the check becomes:
    ```solidity
    (4 * 1e18 * 10 ** 4) <= (4 * 1e18 * 10 ** 0): FALSE
    ```
**Result**: This check passes, allowing Max to cancel and prevent Lewis from claiming their eligible ETH from SudoSwap.

### Impact

The attack allows a malicious actor to prevent legitimate token holders from claiming their eligible NFT sale proceeds from SudoSwap. This could lead to significant financial losses for affected users.

### PoC

#### Setup
* Update the [`CollectionShutdown.t::constructor()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/test/utils/CollectionShutdown.t.sol#L35) to mint CTs token with denominator more that 0
```diff
File: CollectionShutdown.t.sol
29:     constructor () forkBlock(19_425_694) {
30:         // Deploy our platform contracts
31:         _deployPlatform();
---
-35:         locker.createCollection(address(erc721b), 'Test Collection', 'TEST', 0);
+35:         locker.createCollection(address(erc721b), 'Test Collection', 'TEST', 4);
36: 
```
* Put the snippet below into the protocol test suite: `flayer/test/utils/CollectionShutdown.t.sol` 
* Run test: 
```bash
forge test --mt test_CanBlockEligibleUsersToClaim -vvv
```

#### Coded PoC
<details>
  <summary>Show Coded PoC</summary>

```solidity
        function test_CanBlockEligibleUsersToClaim() public {
        address Lewis = makeAddr("Lewis");
        address Max = makeAddr("Max");
        address MaxRecovery = makeAddr("MaxRecovery");

        // -- Before Attack --
        
        // Mint some tokens to our test users -> totalSupply: 4 ethers (can shutdown)
        vm.startPrank(address(locker));
        collectionToken.mint(Lewis, 2 ether * 10 ** collectionToken.denomination());
        collectionToken.mint(Max, 2 ether * 10 ** collectionToken.denomination());
        vm.stopPrank();

        // Start shutdown with their vore that has passed the threshold quorum
        vm.startPrank(Lewis);
        uint256 lewisVoteBalance = 2 ether * 10 ** collectionToken.denomination();
        collectionToken.approve(address(collectionShutdown), type(uint256).max);
        collectionShutdown.start(address(erc721b));
        assertEq(collectionShutdown.shutdownVoters(address(erc721b), address(Lewis)), lewisVoteBalance);
        vm.stopPrank();

        // Confirm that we can now execute
        assertCanExecute(address(erc721b), true);

        // Mint NFTs into our collection {Locker} and process the execution
        uint[] memory tokenIds = _mintTokensIntoCollection(erc721b, 3);
        collectionShutdown.execute(address(erc721b), tokenIds);

        // Confirm that the {CollectionToken} has been sunset from our {Locker}
        assertEq(address(locker.collectionToken(address(erc721b))), address(0));

        // After we have executed, we should no longer have an execute flag
        assertCanExecute(address(erc721b), false);

        // Mock the process of the Sudoswap pool liquidating the NFTs for ETH. This will
        // provide 0.5 ETH <-> 1 {CollectionToken}.
        _mockSudoswapLiquidation(SUDOSWAP_POOL, tokenIds, 2 ether);

        // Ensure that all state are SET
        ICollectionShutdown.CollectionShutdownParams memory shutdownParamsBefore = collectionShutdown.collectionParams(address(erc721b));
        assertEq(shutdownParamsBefore.shutdownVotes, lewisVoteBalance);
        assertEq(shutdownParamsBefore.sweeperPool, SUDOSWAP_POOL);
        assertEq(shutdownParamsBefore.quorumVotes, lewisVoteBalance);
        assertEq(shutdownParamsBefore.canExecute, false);
        assertEq(address(shutdownParamsBefore.collectionToken), address(collectionToken));
        assertEq(shutdownParamsBefore.availableClaim, 2 ether);

        // -- Attack --
        uint256 balanceOfMaxBefore = collectionToken.balanceOf(address(Max));
        uint256 amountSpendForAttack = 1;

        // Transfer almost full funds to their second account and perform with small amount
        vm.prank(Max);
        collectionToken.transfer(address(MaxRecovery), balanceOfMaxBefore - amountSpendForAttack);
        uint256 balanceOfMaxAfter = collectionToken.balanceOf(address(Max));
        assertEq(balanceOfMaxAfter, amountSpendForAttack);

        // Max votes even it is in the claim state to flag the `canExecute` back to Trrue
        vm.startPrank(Max);
        collectionToken.approve(address(collectionShutdown), type(uint256).max);
        collectionShutdown.vote(address(erc721b));
        assertEq(collectionShutdown.shutdownVoters(address(erc721b), address(Max)), amountSpendForAttack);
        vm.stopPrank();

        // Confirm that Max can now flag `canExecute` back to `TRUE`
        assertCanExecute(address(erc721b), true);

        // Attack to delete all varaibles track, resulting others cannot claim thier eligible ethers
        vm.startPrank(Max);
        locker.createCollection(address(erc721b), 'Test Collection', 'TEST', 0);
        collectionShutdown.cancel(address(erc721b));
        vm.stopPrank();

        // Ensure that all state are DELETE
        ICollectionShutdown.CollectionShutdownParams memory shutdownParamsAfter = collectionShutdown.collectionParams(address(erc721b));
        assertEq(shutdownParamsAfter.shutdownVotes, 0);
        assertEq(shutdownParamsAfter.sweeperPool, address(0));
        assertEq(shutdownParamsAfter.quorumVotes, 0);
        assertEq(shutdownParamsAfter.canExecute, false);
        assertEq(address(shutdownParamsAfter.collectionToken), address(0));
        assertEq(shutdownParamsAfter.availableClaim, 0);

        // -- After Attack --
        vm.expectRevert();
        vm.prank(Lewis);
        collectionShutdown.claim(address(erc721b), payable(Lewis));
    }
```
</details>

#### Result
Results of running the test:
```bash
Ran 1 test for test/utils/CollectionShutdown.t.sol:CollectionShutdownTest
[PASS] test_CanBlockEligibleUsersToClaim() (gas: 1491640)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.96s (3.48ms CPU time)

Ran 1 test suite in 11.17s (10.96s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Mitigation

* Add validations to prevent manipulation of the CT denomination, and restrict voting during the claim state to prevent re-triggering of `params.canExecute`.
```diff
function vote(address _collection) public nonReentrant whenNotPaused {
    // Ensure that we are within the shutdown window
    CollectionShutdownParams memory params = _collectionParams[_collection];
    if (params.quorumVotes == 0) revert ShutdownProccessNotStarted();
+   if (params.sweeperPool != address(0)) revert ShutdownExecuted();
    _collectionParams[_collection] = _vote(_collection, params);
}
```

* Update the usage of token denomination to use the token depens on the tracked token to inconsisten value.
```diff
function cancel(address _collection) public whenNotPaused {
    // Ensure that the vote count has reached quorum
    CollectionShutdownParams memory params = _collectionParams[_collection];
    if (!params.canExecute) revert ShutdownNotReachedQuorum();

    // Check if the total supply has surpassed an amount of the initial required
    // total supply. This would indicate that a collection has grown since the
    // initial shutdown was triggered and could result in an unsuspected liquidation.
-    if (params.collectionToken.totalSupply() <= MAX_SHUTDOWN_TOKENS * 10 ** locker.collectionToken(_collection).denomination()) {
+    if (params.collectionToken.totalSupply() <= MAX_SHUTDOWN_TOKENS * 10 ** params.collectionToken.denomination()) {
        revert InsufficientTotalSupplyToCancel();
    }

    // Remove our execution flag
    delete _collectionParams[_collection];
    emit CollectionShutdownCancelled(_collection);
}
```

# Issue H-19: User can withdraw all N free NFT from Locker for 1 token + txes instead of N token just in one tx. 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/756 

## Found by 
almantare
## Summary

The Locker stores all NFTs that interact with the protocol. NFTs can enter the locker in the following ways:

- The user lists their NFT for sale in `Listings / ProtectedListings` (these NFTs are locked until the end of the listing). It's worth noting that this method immediately returns `1 Collection Token` to the user.
- The user exchanges an NFT for `1 Collection Token` using the `Locker::deposit` functions (NFTs that enter the contract this way are available for withdrawal from the Locker using `Locker::redeem`, which in turn burns `1 Collection Token` from the user, returning them an NFT (any free NFT that is on the contract))

Thus, the normal behavior of the protocol assumes the following invariants:

- Free NFTs should be in the Locker so that the user can use the redeem function
- 1 FLOOR PRICE NFT = 1 Collection Token (this is especially clear from the logic of the listing, where the user specifies floorMultiple - how many tokens above the floor they should be paid for their NFT, however, they are guaranteed to receive `1 collection token` when sold)

The attack, the essence of which will be described below, allows an attacker to take all free NFTs from the locker for just 1 NFT and a relatively small number of tokens to cover the commission, putting them up for sale. Thus, as a result of this attack, the following will happen.

n - number of free NFTs on the `Locker` contract

- The attacker, using 1 + n * listing_tax collection tokens, can earn n + 1 tokens in one transaction
- All free NFTs of the `Locker` contract will be withdrawn and put up for sale, which means the `redeem` functionality allowing to quickly exchange their NFT for `1 Collection Token` will be unavailable.

## Vulnerability Detail

So, let's describe the attack scenario.

Let's say there are `n` free NFTs currently on the `Locker` contract.

Let's say the attacker has one NFT and `n * listing_tax collection tokens`.

Then the attacker only needs to do the following:

1. List their NFT for sale in `Listings` by calling `CreateListings`, paying `listing_tax` (user NFTs = 0, free NFTs = n)
2. As mentioned above, `CreateListings` returns `1 Collection Token` to the user in the same transaction
3. In the same transaction, the user withdraws 1 free NFT from `Locker`, exchanging it for the token received from `CreateListings` (user NFTs = 1, free NFTs = n - 1)
4. Go back to step 1.

At the end, the state will be as follows. (user NFTs on sell = n + 1, free NFTs = 0)

If in

If in the normal behavior of the protocol, `1 free NFT` would be exchanged for `1 Collection token` which would be burned, then in the case of an attack, this is simply an easy way for the attacker to enrich themselves (tokens that in a normal scenario would simply be burned, reducing totalSupply and driving up the price of tokens, as a result of the attack are simply redistributed to the attacker's wallet from other users). Moreover, this also breaks one of the protocol's functionalities - `Locker::redeem`

## Impact

The possibility of this attack in the Flayer protocol is in many ways similar to the possibility of a Sandwich attack in Uniswap. (It also harms the protocol's economy, moreover, it blocks the functionality of quickly exchanging NFT for 1 token).

However, the Sandwich attack is possible due to the structure of the EVM, while this attack is due to shortcomings in the current economic model/implementation.

Severity: High

## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L130

## Tool used

Manual Review

## Recommendation

Perhaps the simplest way to avoid this problem is to not return 1 token to the user during createListings, but only when the listing is realized.

# Issue M-1: Previous `beneficiary` will not be able to claim `beneficiaryFees` if current beneficiary is a pool 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/101 

## Found by 
Ragnarok, araj, cawfree, dany.armstrong90, g, h2134, utsav
## Summary
Previous `beneficiary` will not be able to claim `beneficiaryFees` if current beneficiary is a `pool`

## Vulnerability Detail
Beneficiary can claim their fees using `claim()`, which checks if the `beneficiaryIsPool` and if its true, it reverts
```solidity
function claim(address _beneficiary) public nonReentrant {
        // Ensure that the beneficiary has an amount available to claim. We don't revert
        // at this point as it could open an external protocol to DoS.
        uint amount = beneficiaryFees[_beneficiary];
        if (amount == 0) return;

        // We cannot make a direct claim if the beneficiary is a pool
@>      if (beneficiaryIsPool) revert BeneficiaryPoolCannotClaim();
...
    }
```
Above pointed check is a problem because it checks `beneficiaryIsPool` regardless of `_beneficiary` is current beneficiary or previous beneficiary. 

As result, if current beneficiary is a pool but previous beneficiary was not, then previous beneficiary will not be able to withdraw the fees as above check will revert because `beneficiaryIsPool` represents the status of current beneficiary.

//How this works
1. Suppose the current beneficiaryA is not a pool ie beneficiaryIsPool = false & receives a fees of 100e18
2. Owner changed the beneficiary using `setBeneficiary()` to beneficiaryB, which is a pool ie beneficiaryIsPool = true
3. Natspecs of the `setBeneficiary()` clearly says previous beneficiary should claim their fees, but they will not be able to claim because now beneficiaryIsPool = true, which will revert the transaction
```solidity
    /**
@>   * Allows our beneficiary address to be updated, changing the address that will
     * be allocated fees moving forward. The old beneficiary will still have access
     * to `claim` any fees that were generated whilst they were set.
     *
     * @param _beneficiary The new fee beneficiary
     * @param _isPool If the beneficiary is a Flayer pool
     */
    function setBeneficiary(address _beneficiary, bool _isPool) public onlyOwner {
        beneficiary = _beneficiary;
        beneficiaryIsPool = _isPool;

        // If we are setting the beneficiary to be a Flayer pool, then we want to
        // run some additional logic to confirm that this is a valid pool by checking
        // if we can match it to a corresponding {CollectionToken}.
        if (_isPool && address(locker.collectionToken(_beneficiary)) == address(0)) {
            revert BeneficiaryIsNotPool();
        }

        emit BeneficiaryUpdated(_beneficiary, _isPool);
    }
```

This issue is arising because beneficiaryIsPool is the status of the current beneficiary, but claim() can be used to claim fee by previous beneficiary also

## Impact
Previous beneficiary will not be able to claim their fees

## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/implementation/BaseImplementation.sol#L171
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/implementation/BaseImplementation.sol#L203C4-L223C6

## Tool used
Manual Review

## Recommendation
Remove the above check because if the beneficiary is a `pool` then their fees are store in a different mapping `_poolFee` not `beneficiaryFees`, which means any beneficiary which is a pool, will try to claim then it will revert as there `beneficiaryFee` will be 0(zero)

# Issue M-2: Malicious user can bypass execution of  `CollectionShutdown` function 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/113 

## Found by 
BugPull
### Summary

Malicious user bypasses `CollectionShutdown::preventShutdown` by calling `CollectionShutdown::start` then `CollectionShutdown::reclaimVote` making the use of checks in `preventShutdown` useless.

### Root Cause

In `preventShutdown` function there a check to make sure there isn't currently a shutdown in progress.

[CollectionShutdown.sol#L420-L421](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/utils/CollectionShutdown.sol#L420-L421)

```solidity
File: CollectionShutdown.sol
415:     function preventShutdown(address _collection, bool _prevent) public {
416:         // Make sure our user is a locker manager
417:         if (!locker.lockerManager().isManager(msg.sender)) revert ILocker.CallerIsNotManager();
418: 
419:         // Make sure that there isn't currently a shutdown in progress
420:@>       if (_collectionParams[_collection].shutdownVotes != 0) revert ShutdownProcessAlreadyStarted();
421: 
422:         // Update the shutdown to be prevented
423:         shutdownPrevented[_collection] = _prevent;
424:         emit CollectionShutdownPrevention(_collection, _prevent);
425:     }
```

This check doesn't confirm that the shutdown is in progress or not user can call `CollectionShutdown::start` to start a shutdown.  
Then `CollectionShutdown::reclaimVote` to set shutdownVotes back to 0.

Calling `start` function indeed increas the votes by calling `_vote`  
[CollectionShutdown.sol#L156-L157](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/utils/CollectionShutdown.sol#L156-L157)

```solidity
File: CollectionShutdown.sol
135:     function start(address _collection) public whenNotPaused {
//code
155:         // Cast our vote from the user
156:@>       _collectionParams[_collection] = _vote(_collection, params);
```

In `_votes` the count of shotdowVotes increase which is normal  
[CollectionShutdown.sol#L200-L201](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/utils/CollectionShutdown.sol#L200-L201)

```solidity
File: CollectionShutdown.sol
191:     function _vote(address _collection, CollectionShutdownParams memory params) internal returns (CollectionShutdownParams memory) {
//code
199:         // Register the amount of votes sent as a whole, and store them against the user
200:@>       params.shutdownVotes += uint96(userVotes);
```

There is no prevention for the use initiated the shutdown from calling `reclaimVote` .
[CollectionShutdown.sol#L369-L370](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/utils/CollectionShutdown.sol#L369-L370)

```solidity
File: CollectionShutdown.sol
356:     function reclaimVote(address _collection) public whenNotPaused {
//code
368:         // We delete the votes that the user has attributed to the collection
369:@>       params.shutdownVotes -= uint96(userVotes);
```

This line resets back the shutdownVotes to 0  
Making `CollectionShutdown::preventShutdown` checks useless.

### Internal pre-conditions

- `lockerManager` calling `CollectionShutdown::preventShutdown` .

### Attack Path 1

1. `lockerManager` calling CollectionShutdown::preventShutdown.
2. Malicious user front run the `CollectionShutdown::preventShutdown` by calling `CollectionShutdown::start` and `CollectionShutdown::reclaimVote` .
3. `lockerManger` believe this collection is prevented from shutdown but its not.

### Attack Path 2

1. Malicious user calling `CollectionShutdown::start` and `CollectionShutdown::reclaimVote` .
2. `lockerManager` calling CollectionShutdown::preventShutdown.
3. `lockerManger` believe this collection is prevented from shutdown but its not.

### Impact

Bypass `preventShutdown` function making it useless.

### Mitigation

- When doing a shutdown check for `quorumVotes` or check during `vote()` that the collection is prevented from shutdown.

Change the check in `preventShutdown`.

```diff
-         if (_collectionParams[_collection].shutdownVotes != 0) revert ShutdownProcessAlreadyStarted();
+         if (_collectionParams[_collection].quorumVotes != 0) revert ShutdownProcessAlreadyStarted();
```

# Issue M-3: ERC721 Airdrop item can be redeemed/swapped out by user who is not an authorised claimant 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/124 

## Found by 
h2134
## Summary
ERC721 airdrop  item can be redeemed/swapped out by user who is not an authorised claimant.

## Vulnerability Detail
Locker contract owner calls [requestAirdrop()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/AirdropRecipient.sol#L85) to claim airdrop from external contracts. The airdropped items can be ERC20, ERC721, ERC1155 or Native ETH, and these items are only supposed to be claimed by authorised claimants.

Unfortunately, if the airdropped item is ERC721, a malicious user can bypass the restriction and claim the item by calling [redeem()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Locker.sol#L198) / [swap()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Locker.sol#L241), despite they are not the authorised claimant. 

Consider the following scenario:
1. A highly valuable ERC721 collection item is claimed by Locker contract in an airdrop;
2. Bob creates a collection in Locker contract against the this ERC721 collection;
3. Bob swaps the airdropped item by using a floor collection item;
6. By doing that, bob is able to claim the airdropped even if he is not the authorised claimant.

Please run the PoC in Locker.t.sol to verify:
```solidity
    function testAudit_RedeemAirdroppedItem() public {
        // Airdrop ERC721
        ERC721WithAirdrop erc721d = new ERC721WithAirdrop();

        // Owner requests to claim a highly valuable airdropped item
        locker.requestAirdrop(address(erc721d), abi.encodeWithSignature("claimAirdrop()"));
        assertEq(erc721d.ownerOf(888), address(locker));

        address bob = makeAddr("Bob");
        erc721d.mint(bob, 1);

        // Bob creates a Locker collection against the airdrop collection
        vm.prank(bob);
        address collectionToken = locker.createCollection(address(erc721d), "erc721d", "erc721d", 0);

        // Bob swaps out the airdropped item
        vm.startPrank(bob);
        erc721d.approve(address(locker), 1);
        locker.swap(address(erc721d), 1, 888);
        vm.stopPrank();

        // Bob owns the airdropped item despite he is not the authorised claimant
        assertEq(erc721d.ownerOf(888), address(bob));
    }
```

## Impact

An airdropped ERC721 item can be stolen by malicious user, the authorised claimant won't be able make a claim.

## Code Snippet

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Locker.sol#L198

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Locker.sol#L241

## Tool used

Manual Review

## Recommendation

Should not allow arbitrary user to redeem/swap the airdropped item.

# Issue M-4: In the `Listings.sol#relist()` function, `listing.created` is not set to `block.timestamp`. 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/164 

## Found by 
0xAlix2, Ollam, ZeroTrust, araj, blockchain555, cawfree, cnsdkc007, dany.armstrong90, h2134, jecikpo, ydlee, zraxx, zzykxx
### Summary

The core functionality of the protocol can be blocked by malicious users by not setting `listing.created` to `block.timestamp` in the [`Listings.sol#relist()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L625-L672) function.


### Root Cause

In the `Listings.sol#relist()` function, `listing.created` is not set to `block.timestamp` and it is determined based on the input parameter.


### Internal pre-conditions

_No response_

### External pre-conditions

- When a collection is illiquid and we have a disperate number of tokens spread across multiple  users, a pool has to become unusable.


### Attack Path

- In [`Listings.sol#L665`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L665), a malicious user sets `listing.created` to `type(uint).max` instead of `block.timestamp` in the `Listings.sol#relist()` function.
- Next, When a collection is illiquid and we have a disperate number of tokens spread across multiple  users, a pool has to become unusable.
- In [`CollectionShutdown.sol#L241`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L241), [`_hasListings(_collection)`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L497-L514 ) is always `true`, so the `CollectionShutdown.sol#execute()` function is always reverted.


### Impact

The CollectionShutdown function that is core function of the protocol is damaged.

### PoC

```solidity
function test_CanRelistFloorItemAsLiquidListing(address _lister, address payable _relister, uint _tokenId, uint16 _floorMultiple) public {
        // Ensure that we don't get a token ID conflict
        _assumeValidTokenId(_tokenId);

        // Ensure that we don't set a zero address for our lister and filler, and that they
        // aren't the same address
        _assumeValidAddress(_lister);
        _assumeValidAddress(_relister);
        vm.assume(_lister != _relister);

        // Ensure that our listing multiplier is above 1.00
        _assumeRealisticFloorMultiple(_floorMultiple);

        // Provide a token into the core Locker to create a Floor item
        erc721a.mint(_lister, _tokenId);

        vm.startPrank(_lister);
        erc721a.approve(address(locker), _tokenId);

        uint[] memory tokenIds = new uint[](1);
        tokenIds[0] = _tokenId;

        // Rather than creating a listing, we will deposit it as a floor token
        locker.deposit(address(erc721a), tokenIds);
        vm.stopPrank();

        // Confirm that our listing user has received the underlying ERC20. From the deposit this will be
        // a straight 1:1 swap.
        ICollectionToken token = locker.collectionToken(address(erc721a));
        assertEq(token.balanceOf(_lister), 1 ether);

        vm.startPrank(_relister);

        // Provide our filler with sufficient, approved ERC20 tokens to make the relist
        uint startBalance = 0.5 ether;
        deal(address(token), _relister, startBalance);
        token.approve(address(listings), startBalance);

        // Relist our floor item into one of various collections
        listings.relist({
            _listing: IListings.CreateListing({
                collection: address(erc721a),
                tokenIds: _tokenIdToArray(_tokenId),
                listing: IListings.Listing({
                    owner: _relister,
-->                 created: uint40(type(uint32).max),
                    duration: listings.MIN_LIQUID_DURATION(),
                    floorMultiple: _floorMultiple
                })
            }),
            _payTaxWithEscrow: false
        });

        vm.stopPrank();

        // Confirm that the listing has been created with the expected details
        IListings.Listing memory _listing = listings.listings(address(erc721a), _tokenId);

        assertEq(_listing.created, block.timestamp);
    }
```

Result:
```solidity
Ran 1 test suite in 17.20ms (15.77ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/Listings.t.sol:ListingsTest
[FAIL. Reason: assertion failed: 4294967295 != 3601; counterexample: calldata=0x102a3f2c0000000000000000000000007b71078b91e0cdf997ea0019ceaaec1e461a64ca0000000000000000000000000a255597a7458c26b0d008204a1336eb2fd6aa090000000000000000000000000000000000000000000000000005c3b7d197caff000000000000000000000000000000000000000000000000000000000000006d args=[0x7b71078b91E0CdF997EA0019cEaAeC1E461A64cA, 0x0A255597a7458C26B0D008204A1336EB2fD6AA09, 1622569146370815 [1.622e15], 109]] test_CanRelistFloorItemAsLiquidListing(address,address,uint256,uint16) (runs: 0, μ: 0, ~: 0)

Encountered a total of 1 failing tests, 0 tests succeeded
```


### Mitigation

Add the follow lines to the `Listings.sol#relist()` function:
```solidity
function relist(CreateListing calldata _listing, bool _payTaxWithEscrow) public nonReentrant lockerNotPaused {
        // Load our tokenId
        address _collection = _listing.collection;
        uint _tokenId = _listing.tokenIds[0];

        // Read the existing listing in a single read
        Listing memory oldListing = _listings[_collection][_tokenId];

        // Ensure the caller is not the owner of the listing
        if (oldListing.owner == msg.sender) revert CallerIsAlreadyOwner();

        // Load our new Listing into memory
        Listing memory listing = _listing.listing;

        // Ensure that the existing listing is available
        (bool isAvailable, uint listingPrice) = getListingPrice(_collection, _tokenId);
        if (!isAvailable) revert ListingNotAvailable();

        // We can process a tax refund for the existing listing
        (uint _fees,) = _resolveListingTax(oldListing, _collection, true);
        if (_fees != 0) {
            emit ListingFeeCaptured(_collection, _tokenId, _fees);
        }

        // Find the underlying {CollectionToken} attached to our collection
        ICollectionToken collectionToken = locker.collectionToken(_collection);

        // If the floor multiple of the original listings is different, then this needs
        // to be paid to the original owner of the listing.
        uint listingFloorPrice = 1 ether * 10 ** collectionToken.denomination();
        if (listingPrice > listingFloorPrice) {
            unchecked {
                collectionToken.transferFrom(msg.sender, oldListing.owner, listingPrice - listingFloorPrice);
            }
        }

        // Validate our new listing
        _validateCreateListing(_listing);

        // Store our listing into our Listing mappings
        _listings[_collection][_tokenId] = listing;

+++     _listings[_collection][_tokenId].created = uint40(block.timestamp);        

        // Pay our required taxes
        payTaxWithEscrow(address(collectionToken), getListingTaxRequired(listing, _collection), _payTaxWithEscrow);

        // Emit events
        emit ListingRelisted(_collection, _tokenId, listing);
    }
```

# Issue M-5: Relisting Then Cancelling A Liquidation Auction Results In Losses On Subsequent Deposits 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/185 

## Found by 
cawfree, kuprum
## Summary

Relisting a token undergoing dutch auction risks erroneously persisting an orphaned `isLiquidation` state flags on the token, resulting in loss of due harberger taxes and refunds when the token is relisted.

## Vulnerability Detail

In the following sequence, we protect, liquidate, relist and finally cancel a listing for an underlying token.

Once the sequence is finished, all value is conserved and the underlying token is back in the hands of the original owner - however we have managed to erroneously persist an `isLiquidation` flag against the underlying token, even though it is no longer undergoing liquidation.

### ProtectedListings.t.sol

First, add the following test sequence to `ProtectedListings.t.sol`:

```solidity
function testSherlock_CreateOrphanedFlag() external returns (uint256[] memory tokenIds) {

    // 0. Assume a flashloan address:
    address flashloan = address(0xf1a510a9);
    uint256 numberOfFlashloanTokens = 10;
    vm.startPrank(flashloan);
    {
        tokenIds = new uint256[](numberOfFlashloanTokens);
        for (uint256 i = 0; i < numberOfFlashloanTokens; i++) erc721a.mint(flashloan, tokenIds[i] = 10_000 + i);
        erc721a.setApprovalForAll(address(locker), true);
        locker.deposit(address(erc721a), tokenIds);
    }
    vm.stopPrank();

    assertEq(locker.collectionToken(address(erc721a)).balanceOf(flashloan), 10000000000000000000) /* Flashloan Initial Balance */;

    // 1. `deadbeef` mints and protects their token:
    address deadbeef = address(0xdeadbeef);
    uint256 deadbeefTokenId = 100;
    vm.startPrank(deadbeef);
        erc721a.setApprovalForAll(address(protectedListings), true);
        erc721a.mint(deadbeef, deadbeefTokenId);
        tokenIds = new uint256[](1);
        tokenIds[0] = deadbeefTokenId;
        IProtectedListings.ProtectedListing memory listing = IProtectedListings.ProtectedListing({
            owner: payable(deadbeef),
            tokenTaken: 0.95 ether,
            checkpoint: 0
        });
        _createProtectedListing({
            _listing: IProtectedListings.CreateListing({
                collection: address(erc721a),
                tokenIds: tokenIds,
                listing: listing
            })
        });
    vm.stopPrank();
    assertEq(locker.collectionToken(address(erc721a)).balanceOf(deadbeef), 950000000000000000) /* Deadbeef Balance After Deposit */;

    // 2. Block is mined:
    vm.roll(block.number + 1);
    vm.warp(block.timestamp + 2);

    // 3. Asset is now unhealthy.
    assert(protectedListings.getProtectedListingHealth(address(erc721a), deadbeefTokenId) < 0) /* unhealthy */;

    // 4. Start a liquidation:
    vm.startPrank(deadbeef);
        protectedListings.liquidateProtectedListing(address(erc721a), deadbeefTokenId);
    vm.stopPrank();
    assertEq(locker.collectionToken(address(erc721a)).balanceOf(deadbeef), 1000000000000000000) /* Deadbeef Liquidates Own Listing */;

    // 5. `deadbeefOtherAccount` flash loans the required capital:
    address deadbeefOtherAccount = address(0xdeadbeef + 1);
    uint256 flashLoanAmount = 3000000000000000000 + 90000000000000000;
    vm.startPrank(flashloan);
        locker.collectionToken(address(erc721a)).transfer(deadbeefOtherAccount, flashLoanAmount);
    vm.stopPrank();

    assertEq(locker.collectionToken(address(erc721a)).balanceOf(flashloan), 6910000000000000000) /* Flash Loan Balance Reduced */;

    // 6. Relist, masquerading as a different account:
    vm.startPrank(deadbeefOtherAccount);
        locker.collectionToken(address(erc721a)).approve(address(listings), type(uint256).max) /* Max Approve Listings */;

        tokenIds = new uint256[](1);
        tokenIds[0] = deadbeefTokenId;
        listings.relist(
            IListings.CreateListing({
                collection: address(erc721a),
                tokenIds: tokenIds,
                listing: IListings.Listing({
                    owner: payable(deadbeef),
                    created: uint40(block.timestamp),
                    duration: 7 days /* convert back into a liquid listing */,
                    floorMultiple: 400
                })
            }),
            false
        );
    vm.stopPrank();

    // 7. Now the listing is no longer in dutch auction, cancel it:
    vm.startPrank(deadbeef);
        locker.collectionToken(address(erc721a)).approve(address(listings), type(uint256).max);
        listings.cancelListings(address(erc721a), tokenIds, false);

        // Repay the flash loan for deadbeef:
        locker.collectionToken(address(erc721a)).transfer(flashloan, flashLoanAmount);

        // Assert that value has been conserved (nothing has been lost):
        assertEq(locker.collectionToken(address(erc721a)).balanceOf(deadbeef), 0);
        assertEq(locker.collectionToken(address(erc721a)).balanceOf(deadbeefOtherAccount), 0);
        assertEq(locker.collectionToken(address(erc721a)).balanceOf(flashloan), numberOfFlashloanTokens * 1 ether);

        // Deadbeef is back in possession of the token:
        assertEq(erc721a.ownerOf(deadbeefTokenId), deadbeef);

    vm.stopPrank();

}
```

Then add the following `console.log`s to `Listings.sol` to the end of [`cancelListings`](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/Listings.sol#L414C14-L414C28):

```diff
// Create our checkpoint as utilisation rates will change
protectedListings.createCheckpoint(_collection); /// @custom:hunter now we can call this arbitrarily since tehre are no throws
+
+ for (uint256 i = 0; i < _tokenIds.length; i++) {
+    console.log("After cancellation, is still a liquidation?", _isLiquidation[_collection][_tokenIds[i]]);
+ }
+
emit ListingsCancelled(_collection, _tokenIds);
```

And finally run using:

```shell
forge test --match-test "testSherlock_CreateOrphanedFlag" -vv
```

```shell
Ran 1 test for test/ProtectedListings.t.sol:ProtectedListingsTest
[PASS] testSherlock_CreateOrphanedFlag() (gas: 2248654)
Logs:
  After cancellation, is still a liquidation? true

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.49ms (2.00ms CPU time)
```

This confirms that it is possible to persist an `_isLiquidation` flag against a token which is no longer undergoing liquidation.

## Impact

An orphaned `_isLiquidation` flag persisted against a token can result in losses for a users and the protocol if it were to be deposited back into the system.

This is because refunds for harberger taxes are only paid for the case where a token is specifically not marked as `_isLiquidation`:

```solidity
// Check if there is collateral on the listing, as this we bypass fees and refunds
if (!_isLiquidation[_collection][_tokenId]) { /// @audit only pay refunds for tokens which aren't marked as liquidations
    // Find the amount of prepaid tax from current timestamp to prepaid timestamp
    // and refund unused gas to the user.
    (uint fee, uint refund) = _resolveListingTax(_listings[_collection][_tokenId], _collection, false);
    emit ListingFeeCaptured(_collection, _tokenId, fee);

    assembly {
        tstore(FILL_FEE, add(tload(FILL_FEE), fee))
        tstore(FILL_REFUND, add(tload(FILL_REFUND), refund))
    }
} else {
    delete _isLiquidation[_collection][_tokenId];
}
```

```solidity
// We can process a tax refund for the existing listing if it isn't a liquidation
if (!_isLiquidation[_collection][_tokenId]) {
    (uint _fees,) = _resolveListingTax(oldListing, _collection, true); /// @audit only realize refunds and fees if not a liquidation
    if (_fees != 0) {
        emit ListingFeeCaptured(_collection, _tokenId, _fees);
    }
}
```

This materializes as losses for both the user and the protocol.

## Code Snippet

## Tool used

[**Shaheen Vision**](https://x.com/0x_Shaheen/status/1722664258142650806)

## Recommendation

In our proof of concept, we break the protocol invariant that a [dutch auction cannot be modified](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/Listings.sol#L311C13-L312C98) by first converting it back into a liquid listing via a call to [`relist`](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/Listings.sol#L625C14-L625C20).

Since dutch auctions should not be modifiable, we recommend **rejecting attempts to relist them**, as this is a form of modification.

In addition, ensure that the `_isLiquidation` flag is cleared when cancelling a listing.


# Issue M-6: Admin can not set the pool fee since it is only set in memory 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/188 

## Found by 
Ironsidesec, ZeroTrust, alexzoid, g, h2134, tvdung94, zarkk01
### Summary

The pool fee is only set in memory and not in storage so specific pool fees will not apply.

### Root Cause

The pool fee is only stored in memory.

ref: [UniswapImplementation:setFee()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/implementation/UniswapImplementation.sol#L788-L789)

```solidity
    // @audit poolParams is only stored in memory so the new fee is not set in permanent storage
    PoolParams memory poolParams = _poolParams[_poolId];
    poolParams.poolFee = _fee;
```

### Internal pre-conditions

1. Admin calls [`setFee()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/implementation/UniswapImplementation.sol#L783-L793) with any fee value.

### External pre-conditions

None

### Attack Path

None

### Impact

Specific pool fees will not apply. Only the default fee will apply to swaps.

ref: [UniswapImplementation::getFee()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/implementation/UniswapImplementation.sol#L703-L706)
```solidity
    // @audit poolFee will always be 0
    uint24 poolFee = _poolParams[_poolId].poolFee;
    if (poolFee != 0) {
        fee_ = poolFee;
    }
```

### PoC

_No response_

### Mitigation

Use storage instead of memory for `poolParams` in [`setFee()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/implementation/UniswapImplementation.sol#L783-L793).

# Issue M-7: `collectionLiquidationComplete()` can be DoS by directly transferring a collection NFT to the `sweeperPool` 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/209 

## Found by 
Audinarey, OpaBatyo, Sentryx, araj, cawfree, utsav
## Summary
`collectionLiquidationComplete()` can be DoS by directly transferring a collection NFT to the `sweeperPool`

## Vulnerability Detail
When a user claim their ETH using claim(), it calls `collectionLiquidationComplete()` to check if all the NFT are liquidated in sudoswap pool or not.
```solidity
function claim(address _collection, address payable _claimant) public nonReentrant whenNotPaused {
...
        // Ensure that all NFTs have sold from our Sudoswap pool
@>      if (!collectionLiquidationComplete(_collection)) revert NotAllTokensSold();
...
    }
```
```solidity
function collectionLiquidationComplete(address _collection) public view returns (bool) {
...
        // Check that all token IDs have been bought from the pool
        for (uint i; i < sweeperPoolTokenIdsLength; ++i) {
            // If the pool still owns the NFT, then we have to revert as not all tokens have been sold
@>         if (collection.ownerOf(params.sweeperPoolTokenIds[i]) == sweeperPool) {
                return false;
            }
        }

        return true;
    }
```
To check all NFTs are liquidated or not, it calls `ownerOf()`. Now the problem is a malicious user can buy a NFT of that collection and directly `transfer` it to the `Pool` address. As result, the above check will return `false`, which will revert the `claim()` ie permanently DoS the claim()

## Impact
Claim() can be permanently DoS ie users will not be able to claim their ETH

## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L295
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L461C1-L464C1

## Tool used
Manual Review

## Recommendation
Don't use ownerOf() as checking if all NFTs are liquidated or not, instead use any internal mechanism/ counting

# Issue M-8: There is a logical error in the removeFeeExemption() function. 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/219 

## Found by 
0xc0ffEE, BugPull, Ironsidesec, NoOne, Tendency, ZeroTrust, alexzoid, cawfree, g, h2134, kuprum, stuart\_the\_minion, valuevalk, xKeywordx, zzykxx

## Summary
There is a logical error in the removeFeeExemption() function, causing feeOverrides[_beneficiary] to never be removed.
## Vulnerability Detail
```javascript
   function removeFeeExemption(address _beneficiary) public onlyOwner {
        // Check that a beneficiary is currently enabled
@>>        uint24 hasExemption = uint24(feeOverrides[_beneficiary] & 0xFFFFFF);
@>>        if (hasExemption != 1) {
            revert NoBeneficiaryExemption(_beneficiary);
        }

        delete feeOverrides[_beneficiary];
        emit BeneficiaryFeeRemoved(_beneficiary);
    }
```
Through setFeeExemption(), we know that the lower 24 bits of feeOverrides[_beneficiary] are set to 0xFFFFFF. Therefore, the expression uint24 hasExemption = uint24(feeOverrides[_beneficiary] & 0xFFFFFF) results in 0xFFFFFF & 0xFFFFFF = 0xFFFFFF. As a result, hasExemption will never be 1, causing removeFeeExemption() to always revert. Consequently, feeOverrides[_beneficiary] will never be removed.
```javascript
function setFeeExemption(address _beneficiary, uint24 _flatFee) public onlyOwner {
        // Ensure that our custom fee conforms to Uniswap V4 requirements
        if (!_flatFee.isValid()) {
            revert FeeExemptionInvalid(_flatFee, LPFeeLibrary.MAX_LP_FEE);
        }

        // We need to be able to detect if the zero value is a flat fee being applied to
        // the user, or it just hasn't been set. By packing the `1` in the latter `uint24`
        // we essentially get a boolean flag to show this.
@>>        feeOverrides[_beneficiary] = uint48(_flatFee) << 24 | 0xFFFFFF;
        emit BeneficiaryFeeSet(_beneficiary, _flatFee);
    }
```

## Impact
Since feeOverrides[_beneficiary] cannot be removed, the user continues to receive reduced fee benefits, leading to partial fee losses for LPs and the protocol.

## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/implementation/UniswapImplementation.sol#L749

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/implementation/UniswapImplementation.sol#L729
## Tool used

Manual Review

## Recommendation
```diff
   function removeFeeExemption(address _beneficiary) public onlyOwner {
        // Check that a beneficiary is currently enabled
        uint24 hasExemption = uint24(feeOverrides[_beneficiary] & 0xFFFFFF);
-        if (hasExemption != 1) {
+        if (hasExemption != 0xFFFFFF) {
            revert NoBeneficiaryExemption(_beneficiary);
        }

        delete feeOverrides[_beneficiary];
        emit BeneficiaryFeeRemoved(_beneficiary);
    }
```

# Issue M-9: It is possible to prevent the execution of the `execute()` function, listing only one NFT. 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/244 

## Found by 
Aymen0909, BugPull, BugsFinders0x, Tendency, ZeroTrust
### Summary

It Is possible for everyone to front-run the calls to the `execute()` function, by listing an NFT in the `Listings.sol` contract, right after the quorum is reached. 


### Root Cause

In the `CollectionShutdown.sol`, there is the possibility to start a voting system to delete a specific collection from the platform.
The logic of the voting system is simple: 
There is a quorum to reach.
After that, it is possible to call the `execute()` function that will delete the collection.

The function that will initialize this voting system is the `start()` function that is possible to check here [`CollectionShutdown.sol:135`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L135).

Supposing that the Voting system for a collection starts, and the quorum is reached.

At this point, the Owner of the `CollectionShutdown.sol` contract, could call the `execute()` function to delete the collection from the platform, [`CollectionShutdown.sol:231`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L231).

In the meantime, a malicious actor who doesn't want the collection will be deleted lists an NFT of the same collection, [`Listings.sol:130`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L130).

At this point, when the Owner of the `CollectionShutdown.sol` tries to call the `execute()` function, there is a call on the `_hasListing()` function to check if the collection is listed [`CollectionShutdown.sol:241`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L241).
The collection is listed, so the `execute` function will go in revert due to the previous check.

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

There are 2 different users:
User2, Hacker

1) Hacker creates and initializes Collection with 10 NFTs.
2) User2 owns the same NFTs (5 in total) and after some time, wants to start a voting system to delete the collection from the platform.
3) User2 calls `start()` in `CollectionShutdown.sol` and reaches the quorum to delete the collection.
4) Hacker has one more NFT, and lists it in `Listings.sol`
5) Owner of `CollectionShutdown.sol` calls the `execute()`
6) Reverts, due to the `_hasListing()` check, that attests that there is one token listed already.

### Impact

Everyone can prevent the execution of the `execute()` function.
This will make the voting system useless

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;

import {stdStorage, StdStorage, Test} from 'forge-std/Test.sol';
import {ProxyAdmin, ITransparentUpgradeableProxy} from "@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol";
import {TransparentUpgradeableProxy} from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CollectionToken} from "@flayer/CollectionToken.sol";
import {FlayerTest} from "../lib/FlayerTest.sol";
import {LockerManager} from "../../src/contracts/LockerManager.sol";
import {Locker} from "../../src/contracts/Locker.sol";
import {Listings} from "../../src/contracts/Listings.sol";
import {ProtectedListings} from "../../src/contracts/ProtectedListings.sol";
import {TaxCalculator} from "../../src/contracts/TaxCalculator.sol";
import {WETH9} from '../lib/WETH.sol';
import {ERC721Mock} from '../mocks/ERC721Mock.sol';
import {ERC721MyMock} from '../mocks/ERC721MyMock.sol';
import {UniswapImplementation} from '@flayer/implementation/UniswapImplementation.sol';
import {PoolManager} from "@uniswap/v4-core/src/PoolManager.sol";
import {ICollectionToken} from "../../src/interfaces/ICollectionToken.sol";
import {Clones} from '@openzeppelin/contracts/proxy/Clones.sol';
import {PoolKey} from '@uniswap/v4-core/src/types/PoolKey.sol';
import {IPoolManager, PoolManager, Pool} from '@uniswap/v4-core/src/PoolManager.sol';
import {PoolIdLibrary, PoolId} from '@uniswap/v4-core/src/types/PoolId.sol';
import {CurrencyLibrary} from "../../lib/v4-core/src/types/Currency.sol";
import {LinearRangeCurve} from '@flayer/lib/LinearRangeCurve.sol';
import {CollectionShutdown} from '@flayer/utils/CollectionShutdown.sol';
import {ICollectionShutdown} from "../../src/interfaces/utils/ICollectionShutdown.sol";
import {IListings} from "../../src/interfaces/IListings.sol";
import "forge-std/console.sol";


contract CollectionTokenProxyTest is Test {

    using PoolIdLibrary for PoolKey;

    address RANGE_CURVE;
    address payable PAIR_FACTORY = payable(0xA020d57aB0448Ef74115c112D18a9C231CC86000);
    address payable internal _proxy;
    ProxyAdmin internal _proxyAdmin;

    CollectionToken internal _collectionTokenV1Impl;
    CollectionToken internal _collectionTokenV1;
    TransparentUpgradeableProxy _collectionTokenV1Proxy;

    LockerManager lockerManager;
    Locker locker;
    Listings listings;
    ProtectedListings protListings;
    TaxCalculator calculator;
    WETH9 WETH;
    ERC721Mock erc721a;
    ERC721MyMock erc721MyA;
    PoolManager poolManager;
    UniswapImplementation uniswapImplementation;
    CollectionShutdown collectionShutdown;
    address payable public constant VALID_LOCKER_ADDRESS = payable(0x57D88D547641a626eC40242196F69754b25D2FCC);



    // Address that interacts
    address owner;
    address user1;
    address user2;
    address hacker;


    function setUp() public {
        owner = vm.addr(4);
        user1 = vm.addr(1);
        user2 = vm.addr(2);
        hacker = vm.addr(4);

        vm.deal(owner, 1000 ether);
        vm.deal(user1, 1000 ether);
        vm.deal(user2, 1000 ether);
        vm.deal(hacker, 1000 ether);


        //Deploy Contracts
        deployAllContracts();

        vm.prank(user1);
        WETH.deposit{value: 100 ether}();

        vm.prank(user2);
        WETH.deposit{value: 100 ether}();

        vm.prank(hacker);
        WETH.deposit{value: 100 ether}();

    }
    // ================= Listings.sol =====================


    function test_ListBeforeExecute()public{

        // Hacker creates a collection.
        vm.startPrank(hacker);

        address newCollectionToken = locker.createCollection(address(erc721a), "Mock", "MOCK", 0);

        uint256[] memory path = new uint256[](10);

        // Hacker approves 10 tokens to the locker contract to initialize the collection.
        for(uint i = 0; i < 10; i++){
            path[i] = i;
            erc721a.approve(address(locker), i);
        }

        WETH.approve(address(locker), 10 ether);


        // Hacker initialize the collection and send to locker 10 ERC721a. Hacker owns only one more out of the platform.
        locker.initializeCollection(address(erc721a), 1 ether, path, path.length * 1 ether, 158456325028528675187087900672);
        vm.stopPrank();





        // To call the start() function, user2 must deposit first to get collectionTokens to vote.
        vm.startPrank(user2);

        // Approve TOKEN IDS To locker contract
        uint[] memory path2 = new uint[](10);

        uint iterations = 0;

        for(uint i = 11; i < 21; i++){
            erc721a.approve(address(locker), i);
            path2[iterations] = i;
            iterations++;
        }


        // User2 deposit and get collectionTokens in ratio 1:1
        locker.deposit(address(erc721a), path2);

        console.log("User2 CollectionToken Balance, BEFORE start a votation", CollectionToken(newCollectionToken).balanceOf(user2));



        // User2 approves CollectionTokens to the collectionShutdown contract to start a vote.
        CollectionToken(newCollectionToken).approve(address(collectionShutdown), CollectionToken(newCollectionToken).balanceOf(user2));


        // User2 calls the start() function
        collectionShutdown.start(address(erc721a));


        console.log("User2 CollectionToken Balance, AFTER start a votation", CollectionToken(newCollectionToken).balanceOf(user2));
        console.log("");



        // Check Params to calls execute() function.
        ICollectionShutdown.CollectionShutdownParams memory params = collectionShutdown.collectionParams(address(erc721a));

        console.log("shutdownVotes", params.shutdownVotes);
        console.log("quorumVotes", params.quorumVotes);
        console.log("canExecute", params.canExecute);
        console.log("availableClaim", params.availableClaim);

        vm.stopPrank();




        // Hacker sees that the quorum is reached, and decide to deposit the other token that he owns, and list it.
        vm.startPrank(hacker);

        // Approves token ID 10, to listing contract
        erc721a.approve(address(listings), 10);
        uint[] memory path3 = new uint[](1);
        path3[0] = 10;



        // Listings Paramenter
        IListings.Listing memory newListingsBaseParameter = listings.listings(address(erc721a), 10);
        newListingsBaseParameter.owner = payable(hacker);
        newListingsBaseParameter.duration = uint32(5 days);
        newListingsBaseParameter.created = uint40(block.timestamp);
        newListingsBaseParameter.floorMultiple = 10_00;


        // Listings Paramenter
        IListings.CreateListing memory listingsParams;
        listingsParams.collection = address(erc721a);
        listingsParams.tokenIds = path3;
        listingsParams.listing = newListingsBaseParameter;

        // Listings Paramenter
        IListings.CreateListing[] memory listingsParamsCorrect = new IListings.CreateListing[](1);
        listingsParamsCorrect[0] = listingsParams;


        // Hacker creates a Listing for the collection erc721a
        listings.createListings(listingsParamsCorrect);
        vm.stopPrank();




        //Delete collection is impossible, due to hasListing error.
        vm.prank(owner);
        vm.expectRevert();
        collectionShutdown.execute(address(erc721a), path);
    }


    function deployAllContracts()internal{


        // Deploy Collection Token
        vm.startPrank(owner);

        _proxyAdmin = new ProxyAdmin();

        bytes memory data = abi.encodeWithSignature("initialize(string,string,uint256)", "TokenName", "TKN", 10);

        _collectionTokenV1Impl = new CollectionToken();
        _collectionTokenV1Proxy = new TransparentUpgradeableProxy(
            address(_collectionTokenV1Impl),
            address(_proxyAdmin),
            data
        );

        _collectionTokenV1 = CollectionToken(address(_collectionTokenV1Proxy));
        assertEq(_collectionTokenV1.name(), "TokenName");


        // Deploy LockerManager
        lockerManager = new LockerManager();

        // Deploy Locker
        locker = new Locker(address(_collectionTokenV1Impl), address(lockerManager));

        // Deploy Listings
        listings = new Listings(locker);

        // Deploy ProtectedListings
        protListings = new ProtectedListings(locker, address(listings));


        // Deploy RANGE_CURVE
        RANGE_CURVE = address(new LinearRangeCurve());


        // Deploy TaxCalculator
        calculator = new TaxCalculator();


        // Deploy CollectionShutdown
        collectionShutdown = new CollectionShutdown(locker, PAIR_FACTORY, RANGE_CURVE);


        // Deploy PoolManager
        poolManager = new PoolManager(500000);


        // Deploy WETH
        WETH = new WETH9();


        // Deploy ERC721 Contract
        erc721a = new ERC721Mock();


        //// Deploy ERC721 Malicious Contract
        //erc721MyA = new ERC721MyMock(address(locker));


        // Uniswap Implementation
        deployCodeTo('UniswapImplementation.sol', abi.encode(address(poolManager), address(locker), address(WETH)), VALID_LOCKER_ADDRESS);
        uniswapImplementation = UniswapImplementation(VALID_LOCKER_ADDRESS);
        uniswapImplementation.initialize(locker.owner());


        // First settings for the contracts
        listings.setProtectedListings(address(protListings));

        locker.setListingsContract(payable(address(listings)));
        locker.setTaxCalculator(address(calculator));
        locker.setCollectionShutdownContract(payable(address(collectionShutdown)));
        locker.setImplementation(address(uniswapImplementation));

        lockerManager.setManager(address(listings), true);
        lockerManager.setManager(address(protListings), true);
        lockerManager.setManager(address(collectionShutdown), true);


        // Mint some tokens ERC721 for user1 and hacker
        for(uint i = 0; i < 11; i++){
            erc721a.mint(hacker, i);
        }


        for(uint i = 11; i < 21; i++){
            erc721a.mint(user2, i);
        }


        // erc721a id 10 For the hacker
        erc721a.mint(user1, 21);


        vm.stopPrank();
    }
}

```

### Mitigation

Perform a check to prevent a collection that is up for deletion from being listed

# Issue M-10: If a collection has been shutdown but later re-initialized, it cannot be shutdown again 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/318 

## Found by 
0x37, 0xAlix2, Audinarey, BugPull, Hearmen, OpaBatyo, dany.armstrong90, kuprum, merlinboii, novaman33, robertodf
## Summary
If a collection has been shutdown, it can later be re-initialized for use in the protocol again. But any attempts to shut it down again will fail due to a varaible not being reset when the first shutdown is made.
## Vulnerability Detail
When a collection's shutdown has collected `>= quorum` votes, the admin can initiate the [execution of shutdown](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/utils/CollectionShutdown.sol#L231-L275) which essentially sunsets the collection from the locker and lists all the NFTs in a sudoswap pool so owners of the collection token can later claim their share of the sale proceeds.

When the call to [`sunsetCollection()`](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/Locker.sol#L409-L427) in `Locker.sol` is made, it deletes both the `_collectionToken` and `collectionInitialized` mappings' entries so that the collection can later be re-registered and initialized if there is interest:

```solidity
    // Delete our underlying token, then no deposits or actions can be made
    delete _collectionToken[_collection];

    // Remove our `collectionInitialized` flag
    delete collectionInitialized[_collection];
```

The issue is that nowhere during the shutdown process is the `params.shutdownVotes` variable reset back to 0. The only way for it to decrease is if a user reclaims their vote, but then the shutdown won't finalize at all. In normal circumstances, the variable is [checked against]((https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/utils/CollectionShutdown.sol#L141)) to ensure it is 0 before starting a collection shutdown, in order to prevent starting 2 simulteaneous shutdowns of the same collection.

```solidity
    if (params.shutdownVotes != 0) revert ShutdownProcessAlreadyStarted();
```

Thus, since it is never reset back to 0 when the shutdown is executed, if the collection is later re-initialized into the protocol and an attempt to shutdown again is made, the call to `start()` will revert on this line.
## Impact
A collection that has been shutdown and later re-initialized can never be shutdown again.
## Code Snippet
```solidity
    if (params.shutdownVotes != 0) revert ShutdownProcessAlreadyStarted();
```
## Tool used
Manual Review

## Recommendation
Reset variable back to 0 when all users have claimed their tokens and the process of shutting down a collection is completely finished.

# Issue M-11: A user loses funds when he modifies only price of listings. 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/340 

## Found by 
0xHappy, Ironsidesec, Ruhum, ZeroTrust, blockchain555, dany.armstrong90, dimulski, super\_jack, t.aksoy, ydlee, zarkk01
## Summary
When user modifies only price of listings, the protocol applies more taxes than normal.

## Vulnerability Detail
`Listings.sol#modifiyListings()` function is as follows.
```solidity
    function modifyListings(address _collection, ModifyListing[] calldata _modifyListings, bool _payTaxWithEscrow) public nonReentrant lockerNotPaused returns (uint taxRequired_, uint refund_) {
        uint fees;

        for (uint i; i < _modifyListings.length; ++i) {
            // Store the listing
            ModifyListing memory params = _modifyListings[i];
            Listing storage listing = _listings[_collection][params.tokenId];

            // We can only modify liquid listings
            if (getListingType(listing) != Enums.ListingType.LIQUID) revert InvalidListingType();

            // Ensure the caller is the owner of the listing
            if (listing.owner != msg.sender) revert CallerIsNotOwner(listing.owner);

            // Check if we have no changes, as we can continue our loop early
            if (params.duration == 0 && params.floorMultiple == listing.floorMultiple) {
                continue;
            }

            // Collect tax on the existing listing
323         (uint _fees, uint _refund) = _resolveListingTax(listing, _collection, false);
            emit ListingFeeCaptured(_collection, params.tokenId, _fees);

            fees += _fees;
            refund_ += _refund;

            // Check if we are altering the duration of the listing
330         if (params.duration != 0) {
                // Ensure that the requested duration falls within our listing range
                if (params.duration < MIN_LIQUID_DURATION) revert ListingDurationBelowMin(params.duration, MIN_LIQUID_DURATION);
                if (params.duration > MAX_LIQUID_DURATION) revert ListingDurationExceedsMax(params.duration, MAX_LIQUID_DURATION);

                emit ListingExtended(_collection, params.tokenId, listing.duration, params.duration);

                listing.created = uint40(block.timestamp);
                listing.duration = params.duration;
            }

            // Check if the floor multiple price has been updated
342         if (params.floorMultiple != listing.floorMultiple) {
                // If we are creating a listing, and not performing an instant liquidation (which
                // would be done via `deposit`), then we need to ensure that the `floorMultiple` is
                // greater than 1.
                if (params.floorMultiple <= MIN_FLOOR_MULTIPLE) revert FloorMultipleMustBeAbove100(params.floorMultiple);
                if (params.floorMultiple > MAX_FLOOR_MULTIPLE) revert FloorMultipleExceedsMax(params.floorMultiple, MAX_FLOOR_MULTIPLE);

                emit ListingFloorMultipleUpdated(_collection, params.tokenId, listing.floorMultiple, params.floorMultiple);

                listing.floorMultiple = params.floorMultiple;
            }

            // Get the amount of tax required for the newly extended listing
355         taxRequired_ += getListingTaxRequired(listing, _collection);
        }

        // cache
        ICollectionToken collectionToken = locker.collectionToken(_collection);

        // If our tax refund does not cover the full amount of tax required, then we will need to make an
        // additional tax payment.
        if (taxRequired_ > refund_) {
            unchecked {
                payTaxWithEscrow(address(collectionToken), taxRequired_ - refund_, _payTaxWithEscrow);
            }
            refund_ = 0;
        } else {
            unchecked {
                refund_ -= taxRequired_;
            }
        }

        // Check if we have fees to be paid from the listings
        if (fees != 0) {
            collectionToken.approve(address(locker.implementation()), fees);
            locker.implementation().depositFees(_collection, 0, fees);
        }

        // If there is tax to refund after paying the new tax, then allocate it to the user via escrow
        if (refund_ != 0) {
            _deposit(msg.sender, address(collectionToken), refund_);
        }
    }
```
It calculates refund amount on L323 through `_resolveListingTax()`.
```solidity
    function _resolveListingTax(Listing memory _listing, address _collection, bool _action) private returns (uint fees_, uint refund_) {
        // If we have been passed a Floor item as the listing, then no tax should be handled
        if (_listing.owner == address(0)) {
            return (fees_, refund_);
        }

        // Get the amount of tax in total that will have been paid for this listing
        uint taxPaid = getListingTaxRequired(_listing, _collection);
        if (taxPaid == 0) {
            return (fees_, refund_);
        }

        // Get the amount of tax to be refunded. If the listing has already ended
        // then no refund will be offered.
        if (block.timestamp < _listing.created + _listing.duration) {
933         refund_ = (_listing.duration - (block.timestamp - _listing.created)) * taxPaid / _listing.duration;
        }

        ...
    }
```
As we can see on L933, refund amount is calculated according to remained time.   
If a user modifies with `params.duration == 0`, `listing.created` is not updated.   
But on L355, the protocol applies full tax for whole duration.   
This is wrong.

## Impact
The protocol applies more tax than normal.

## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L303-L384

## Tool used

Manual Review

## Recommendation
`Listings.sol#modifiyListings()` function has to be modified as follows.
```solidity
    function modifyListings(address _collection, ModifyListing[] calldata _modifyListings, bool _payTaxWithEscrow) public nonReentrant lockerNotPaused returns (uint taxRequired_, uint refund_) {
        uint fees;

        for (uint i; i < _modifyListings.length; ++i) {
            // Store the listing
            ModifyListing memory params = _modifyListings[i];
            Listing storage listing = _listings[_collection][params.tokenId];

            ...
   
            // Check if we are altering the duration of the listing
            if (params.duration != 0) {
                // Ensure that the requested duration falls within our listing range
                if (params.duration < MIN_LIQUID_DURATION) revert ListingDurationBelowMin(params.duration, MIN_LIQUID_DURATION);
                if (params.duration > MAX_LIQUID_DURATION) revert ListingDurationExceedsMax(params.duration, MAX_LIQUID_DURATION);
   
                emit ListingExtended(_collection, params.tokenId, listing.duration, params.duration);
   
-               listing.created = uint40(block.timestamp);
                listing.duration = params.duration;
            }
+           listing.created = uint40(block.timestamp);
   
            // Check if the floor multiple price has been updated
            if (params.floorMultiple != listing.floorMultiple) {
                // If we are creating a listing, and not performing an instant liquidation (which
                // would be done via `deposit`), then we need to ensure that the `floorMultiple` is
                // greater than 1.
                if (params.floorMultiple <= MIN_FLOOR_MULTIPLE) revert FloorMultipleMustBeAbove100(params.floorMultiple);
                if (params.floorMultiple > MAX_FLOOR_MULTIPLE) revert FloorMultipleExceedsMax(params.floorMultiple, MAX_FLOOR_MULTIPLE);
   
                emit ListingFloorMultipleUpdated(_collection, params.tokenId, listing.floorMultiple, params.floorMultiple);
   
                listing.floorMultiple = params.floorMultiple;
            }
   
            // Get the amount of tax required for the newly extended listing
            taxRequired_ += getListingTaxRequired(listing, _collection);
        }

        ...
    }
```

# Issue M-12: In the unlockProtectedListing() function, the interest that was supposed to be distributed to LP holders was instead burned. 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/431 

## Found by 
0x73696d616f, Audinarey, Ironsidesec, Ollam, ZeroTrust
## Summary
In the unlockProtectedListing() function, the interest that was supposed to be distributed to LP holders was instead burned.
## Vulnerability Detail
```javascript
function unlockProtectedListing(address _collection, uint _tokenId, bool _withdraw) public lockerNotPaused {
        // Ensure this is a protected listing
        ProtectedListing memory listing = _protectedListings[_collection][_tokenId];

        // Ensure the caller owns the listing
        if (listing.owner != msg.sender) revert CallerIsNotOwner(listing.owner);

        // Ensure that the protected listing has run out of collateral
        int collateral = getProtectedListingHealth(_collection, _tokenId);
        if (collateral < 0) revert InsufficientCollateral();

        // cache
        ICollectionToken collectionToken = locker.collectionToken(_collection);
        uint denomination = collectionToken.denomination();
        uint96 tokenTaken = _protectedListings[_collection][_tokenId].tokenTaken;

        // Repay the loaned amount, plus a fee from lock duration
@>>        uint fee = unlockPrice(_collection, _tokenId) * 10 ** denomination;
@>>        collectionToken.burnFrom(msg.sender, fee);

        // We need to burn the amount that was paid into the Listings contract
@>>        collectionToken.burn((1 ether - tokenTaken) * 10 ** denomination);

        // Remove our listing type
        unchecked { --listingCount[_collection]; }

        // Delete the listing objects
        delete _protectedListings[_collection][_tokenId];

        // Transfer the listing ERC721 back to the user
        if (_withdraw) {
            locker.withdrawToken(_collection, _tokenId, msg.sender);
            emit ListingAssetWithdraw(_collection, _tokenId);
        } else {
            canWithdrawAsset[_collection][_tokenId] = msg.sender;
        }

        // Update our checkpoint to reflect that listings have been removed
        _createCheckpoint(_collection);

        // Emit an event
        emit ListingUnlocked(_collection, _tokenId, fee);
    }
```
从unlockProtectedListing()调用的unlockPrice()可以知道，feeb包含了本金和利息。
```javascript
 function unlockPrice(address _collection, uint _tokenId) public view returns (uint unlockPrice_) {
        // Get the information relating to the protected listing
        ProtectedListing memory listing = _protectedListings[_collection][_tokenId];

        // Calculate the final amount using the compounded factors and principle amount
@>>        unlockPrice_ = locker.taxCalculator().compound({
            _principle: listing.tokenTaken,
            _initialCheckpoint: collectionCheckpoints[_collection][listing.checkpoint],
            _currentCheckpoint: _currentCheckpoint(_collection)
        });
    }
```
Therefore, after burning the fee, the interest paid by the user was also burned. This portion of the interest should have been distributed to the LP holders. Evidence for this can be found in the liquidateProtectedListing() function, where the interest generated by the ProtectedListing NFT is distributed to the LP holders.
```javascript
  function liquidateProtectedListing(address _collection, uint _tokenId) public lockerNotPaused listingExists(_collection, _tokenId) {
        //-------skip-----------

        // Send the remaining tokens to {Locker} implementation as fees
        uint remainingCollateral = (1 ether - listing.tokenTaken - KEEPER_REWARD) * 10 ** denomination;
        if (remainingCollateral > 0) {
            IBaseImplementation implementation = locker.implementation();
            collectionToken.approve(address(implementation), remainingCollateral);
@>>            implementation.depositFees(_collection, 0, remainingCollateral);
        }
 //-------skip-----------
    }
```
After the interest is burned, it causes deflation in the total amount of collectionToken, which leads to serious problems:

	1.	The total number of collectionTokens no longer matches the number of NFTs (it becomes less than the number of NFTs in the Locker contract), making it impossible to redeem some NFTs.
	2.	The utilizationRate() calculation results in a utilization rate greater than 100%, leading to an excessively high interestRate_, which in turn makes it impossible for users to create listings on ProtectedListings.

```javascript
    /**
     * Determines the usage rate of a listing type.
     *
     * @param _collection The collection to calculate the utilization rate of
     *
     * @return listingsOfType_ The number of listings that match the type passed
     * @return utilizationRate_ The utilization rate percentage of the listing type (80% = 0.8 ether)
     */
    function utilizationRate(address _collection) public view virtual returns (uint listingsOfType_, uint utilizationRate_) {
        // Get the count of active listings of the specified listing type
        listingsOfType_ = listingCount[_collection];

        // If we have listings of this type then we need to calculate the percentage, otherwise
        // we will just return a zero percent value.
        if (listingsOfType_ != 0) {
            ICollectionToken collectionToken = locker.collectionToken(_collection);

            // If we have no totalSupply, then we have a zero percent utilization
            uint totalSupply = collectionToken.totalSupply();
            if (totalSupply != 0) {
                utilizationRate_ = (listingsOfType_ * 1e36 * 10 ** collectionToken.denomination()) / totalSupply;
            }
        }
    }
```
## Impact
LP holders suffer losses, and users may be unable to use ProtectedListings normally.
## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L287

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L607

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L429

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L261
## Tool used

Manual Review

## Recommendation
Distribute the interest to the LP holders.

# Issue M-13: Users can dodge `createListing` fees 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/440 

## Found by 
OpaBatyo, valuevalk
## Summary
Users can abuse a loophole to create a listing for 5% less tax than intended, hijack others' tokens at floor value and perpetually relist for free.
## Vulnerability Detail  

For the sake of simplicity, we assume that collection token denomination = 4 and floor price = 1e22  

Users who intend to create a `dutch` listing for any duration <= 4 days can do the following:  
1) Create a normal listing at min floor multiple = 101 and MIN_DUTCH_DURATION = 1 days, tax = 0.1457% (`tokensReceived` = 0.99855e22)
2) [Reserve](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/Listings.sol#L690) their own listing from a different account for tokenTaken 0.95e18, collateral 0.05e18, collateral is burnt, protected listing is at 0 health and will be liquidatable in a few moments (`remainingTokens` = 0.94855e22)
3) Liquidate their own listing through `liquidateProtectedListing`, receive KEEPER_REWARD = 0.05e22 (`remainingTokens` = 0.99855e22)
4) `createLiquidationListing` is invoked with us as owner and hardcoded values `floorMultiple` = 400 and `duration` = 4 days

User has paid 0.14% in tax for a listing that would've normally cost them 5.14% in tax.
This process can be repeated any number of times even after the `liquidationListing` expires to constantly reserve-liquidate it instead of calling `relist` and paying further tax.  

There are numerous other ways in which this loophole of reserve-liquidate can be abused:  
1) Users can create listings for free out of any expired listing at floor value, they only burn 0.05e18 collateral which is then received back as `KEEPER_REWARD`
2) Users can constantly cycle NFTs at floor value (since it is free) and make `liquidationListings`, either making profit if the token sells or making the token unpurchasable at floor since it is in the loop
3) Any user-owned expired listing can be relisted for free through this method instead of paying tax by invoking `relist` 

## Impact
Tax evasion
## Code Snippet
```solidity
        _listings.createLiquidationListing(
            IListings.CreateListing({
                collection: _collection,
                tokenIds: tokenIds,
                listing: IListings.Listing({
                    owner: listing.owner,
                    created: uint40(block.timestamp),
                    duration: 4 days,
                    floorMultiple: 400
                })
            })
        );
```
                
## Tool used

Manual Review

## Recommendation
Impose higher minimum collateral and lower tokenTaken (e.g 0.2e18 and 0.8e18) so the `KEEPER_REWARD` would not cover the cost of burning collateral during reservation, making this exploit unprofitable.

# Issue M-14: Manipulating collection token's total supply to manipulate `utilizationRate` 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/448 

## Found by 
0x73696d616f, BugPull, ComposableSecurity, Ironsidesec, OpaBatyo, jo13
## Summary
You can liquidate the genuine and healthy protected listings by manipulating the utilization rate. You can also unlock your protected listing by paying minimal tax by again manipulating the utlization rate. Same with adjusting the listing to add/remove the token taken amounts, do add/remove more/less than allowed due to the listing health/unlock price manipulation that directly depends on the utilization rate. 

Root cause: allowing to mint/redeem collection tokens in a single transaction/block.
Impact: ultra-high
Likelikood: high

## Vulnerability Detail
`utilizationRate` which is used in accounting the tax and debt, fee calculation can be manipulated which in turn manipulates the protected listing health and listing's unlock price. This is possible because, the `utilizationRate` depends on the collection token's total supply which is again manipulatable by depositing multiple tokens inside the locker.
The flow will be like this, deposit token ids into locker and mint the collection token of a collection, so that total supply increases. So now utilization rate decreases if total supply increases. Now interact with PROTECTED LISTING to adjust, unlock, or liquidate. All these actions depend on the listing health, listing's unlock price which depends on the difference between the compounding factor between different timestamps. These compound factors rely on utilization rate, which is manipulatable at will.

Once protected listing actions are done, you can redeem the collection tokens that were minted and you will get back the token ids. This is pure manipulation, all done in one transaction. Attackers can even increase the utilization rate by first redeeming their collateral token that they bought with WETH from uniV4 pool, and decrease the total supply to minimum due to multiple redemption, and then interact with protected listing actions (adjust, liquidate, unlock) and then due to total supply reduction, the utilization rate is pumped to max and now you can liquidate at high debts and negative listing healths which are actually health when not manipulated. After all actions, you will deposit those token ids back and make the system back to normal. But this second strategy won't have high likelihood, because the tokens redeemable will be low because the deposited ones mostly is a listing, and you cannot redeem a listed token id.

**One of the attack flow :**
1. User creates a protected listing of BAYC 25, which has current collection total supply of 20e18 tokens (, also 10 listings count, 10 locked when pool initialization), so current utilization is 0.5e18 (50%).
2. And user listed his at 2e18 compound factor at checkpoint 500, and user takes 0.7e18 tokens when listing
3. So after 3 days, the checkpoint is 515 and compound factor is at 2.5e18 and the unlock price of the listing is 0.78e18 (0.08 ether tax) at current utilization rate
4. And again 1 day passes, and the there isn't been much activity for a day, so no checkpoint is updated. In these times, the lister can deposit some BAYC into locker and pump the collection token supply and reduce the utilization rate by 1/3 or even half, and then now unlock the listing.
5. Now the lister is supposed to pay 0.8 as repayment, but he will only pay 0.78 because the utilization rate is heavily down and the new compound factor at the current timestamp is literally close to the previous checkpoint. Hence the fee owed to repay will be 0.79 + some 0.002 depending on how well the utilization rate is dumped down.  Check [here](https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L304-L305).

So, this is how the fee manipulation will happen. There are other flows too, but you get it. Manipulate the utilization rate, then adjust/liquidate/unlock your /other listing to abuse the manipulated unlock price/listing healths.


https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L273

https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L582-L591

https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L497-L500

https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L607-L617

```solidity
ProtectedListings.sol

240:     function utilizationRate(address _collection) public view virtual returns (uint listingsOfType_, uint utilizationRate_) {
242:         listingsOfType_ = listingCount[_collection];
243: 
246:         if (listingsOfType_ != 0) {
247:             ICollectionToken collectionToken = locker.collectionToken(_collection);
248: 
249:             // If we have no totalSupply, then we have a zero percent utilization
254:             uint totalSupply = collectionToken.totalSupply();
255:             if (totalSupply != 0) {
256:   >>>           utilizationRate_ = (listingsOfType_ * 1e36 * 10 ** collectionToken.denomination()) / totalSupply;
257:             }
258:         }
259:     }


585:     function _currentCheckpoint(address _collection) internal view returns (Checkpoint memory checkpoint_) {
587:   >>>   (, uint _utilizationRate) = utilizationRate(_collection);
588: 
590:         Checkpoint memory previousCheckpoint = collectionCheckpoints[_collection][collectionCheckpoints[_collection].length - 1];
591: 
593:         checkpoint_ = Checkpoint({
594:             compoundedFactor: locker.taxCalculator().calculateCompoundedFactor({
595:                 _previousCompoundedFactor: previousCheckpoint.compoundedFactor,
596:   >>>           _utilizationRate: _utilizationRate,
597:                 _timePeriod: block.timestamp - previousCheckpoint.timestamp
598:             }),
599:             timestamp: block.timestamp
600:         });
601:     }


499:     function getProtectedListingHealth(address _collection, uint _tokenId) public view listingExists(_collection, _tokenId) returns (int) {
502:         return int(MAX_PROTECTED_TOKEN_AMOUNT) - int(unlockPrice(_collection, _tokenId));
503:     }


612:     function unlockPrice(address _collection, uint _tokenId) public view returns (uint unlockPrice_) {
614:         ProtectedListing memory listing = _protectedListings[_collection][_tokenId];
615: 
617:         unlockPrice_ = locker.taxCalculator().compound({
618:             _principle: listing.tokenTaken,
619:   >>>       _initialCheckpoint: collectionCheckpoints[_collection][listing.checkpoint],
620:   >>>       _currentCheckpoint: _currentCheckpoint(_collection)
621:         });
622:     }

```
https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/TaxCalculator.sol#L69-L90

```solidity
TaxCalculator.sol

110:     function compound(
111:         uint _principle,
112:         IProtectedListings.Checkpoint memory _initialCheckpoint,
113:         IProtectedListings.Checkpoint memory _currentCheckpoint
114:     ) public pure returns (uint compoundAmount_) {
117:         if (_initialCheckpoint.timestamp >= _currentCheckpoint.timestamp) return _principle;
120: 
123:         uint compoundedFactor = _currentCheckpoint.compoundedFactor * 1e18 / _initialCheckpoint.compoundedFactor;
124:         compoundAmount_ = _principle * compoundedFactor / 1e18;
125:     }

```

## Impact
You can liquidate the genuine and healthy protected listings by manipulating the utilization rate. You can also unlock your protected listing by paying minimal tax by again manipulating utlization rate. Same with adjusting the listing to add/remove the token taken amounts, do add/remove more/less than allowed due to the listng health/unlock price manipulation that directly depends on the utilization rate. 

Likelikood : high to medium.
Loss of funds, so High severity.

## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/TaxCalculator.sol#L69-L90

https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L273

https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L582-L591

https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L497-L500

https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/flayer/src/contracts/ProtectedListings.sol#L607-L617

## Tool used

Manual Review

## Recommendation

Block mint/redeem within a single transaction or within 1 block. In that case, sudden deposit and redeem in 1 tx is not possible, so manipulating utilization rate is also not possible.

Do the changes in Locker.sol, by introducing a new state, that `mininumDelay = 1 minute`, so that when a deposit is made, any new deposit/redeem can be done only after a minute. Or better track in number of blocks, like `minBlocksDelay`. Most dex protocols or LRT ones have this mechanism.

# Issue M-15: `UniswapImplementation::beforeSwap()` might revert when swapping native tokens to collection tokens 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/517 

## Found by 
0xAlix2, 0xc0ffEE, ComposableSecurity, JokerStudio, alexzoid, g, h2134, zarkk01, zzykxx
### Summary

[UniswapImplementation::beforeSwap()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/implementation/UniswapImplementation.sol#L547) performs a wrong check which can lead to swaps from native tokens to collection tokens reverting.

### Root Cause

[UniswapImplementation::beforeSwap()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/implementation/UniswapImplementation.sol#L547) swaps the internally accumulated collection token fees into native tokens when:

1. The hook accumulated at least `1` wei of fees in collection tokens
2. An user is swapping native tokens for collection tokens

When the swap is performed by specifing the exact amount of native tokens to pay (ie. `amountSpecified < 0`) the protocol should allow the internal swap only when the amount of native tokens being paid is enough to convert all of the accumulated collection token fees. The protocol however does this incorrectly, as the code checks the `amountSpecified` against `tokenOut` instead of `ethIn`:

```solidity
if (params.amountSpecified >= 0) {
    ...
else {
    (, ethIn, tokenOut, ) = SwapMath.computeSwapStep({
        sqrtPriceCurrentX96: sqrtPriceX96,
        sqrtPriceTargetX96: params.sqrtPriceLimitX96,
        liquidity: poolManager.getLiquidity(poolId),
        amountRemaining: int(pendingPoolFees.amount1),
        feePips: 0
    });

@>  if (tokenOut <= uint(-params.amountSpecified)) {
        // Update our hook delta to reduce the upcoming swap amount to show that we have
        // already spent some of the ETH and received some of the underlying ERC20.
        // Specified = exact input (ETH)
        // Unspecified = token1
        beforeSwapDelta_ = toBeforeSwapDelta(ethIn.toInt128(), -tokenOut.toInt128());
    } else {
        ethIn = tokenOut = 0;
    }
}
```

This results in the [UniswapImplementation::beforeSwap()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/implementation/UniswapImplementation.sol#L490) reverting in the situations explained in internal pre-conditions below.

### Internal pre-conditions

1. User is swapping native tokens for collection tokens
2. The hook accumulated at least `1` wei of collection tokens in fees
3. `tokenOut` is lower than `uint(-params.amountSpecified)` and `ethIn` is bigger than `uint(-params.amountSpecified)`

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

All swaps that follow the internal pre-conditions will revert.

### PoC

To copy-paste in `UniswapImplementation.t.sol`:
```solidity
function test_swapFails() public {
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    ERC721Mock erc721 = new ERC721Mock();
    CollectionToken ctoken = CollectionToken(locker.createCollection(address(erc721), 'ERC721', 'ERC', 0));




    //### APPROVALS
    //-> Alice approvals
    vm.startPrank(alice);
    erc721.setApprovalForAll(address(locker), true);
    ctoken.approve(address(poolSwap), type(uint256).max);
    ctoken.approve(address(uniswapImplementation), type(uint256).max);
    vm.stopPrank();
    _approveNativeToken(alice, address(locker), type(uint).max);
    _approveNativeToken(alice, address(poolManager), type(uint).max);
    _approveNativeToken(alice, address(poolSwap), type(uint).max);

    //-> Bob approvals
    vm.startPrank(bob);
    erc721.setApprovalForAll(address(locker), true);
    ctoken.approve(address(uniswapImplementation), type(uint256).max);
    ctoken.approve(address(poolSwap), type(uint256).max);
    vm.stopPrank();
    _approveNativeToken(bob, address(locker), type(uint).max);
    _approveNativeToken(bob, address(poolManager), type(uint).max);
    _approveNativeToken(bob, address(poolSwap), type(uint).max);

    // uniswapImplementation.setAmmFee(1000);
    // uniswapImplementation.setAmmBeneficiary(BENEFICIARY);



    //### MINT NFTs
    //-> Mint 10 tokens to Alice
    uint[] memory _tokenIds = new uint[](10);
    for (uint i; i < 10; ++i) {
        erc721.mint(alice, i);
        _tokenIds[i] = i;
    }

    //-> Mint an extra token to Alice
    uint[] memory _tokenIdToDepositAlice = new uint[](1);
    erc721.mint(alice, 10);
    _tokenIdToDepositAlice[0] = 10;


    //### [ALICE] COLLECTION INITIALIZATION + LIQUIDITY PROVISION
    //-> alice initializes a collection and adds liquidity: 1e19 NATIVE + 1e19 CTOKEN
    uint256 initialNativeLiquidity = 1e19;
    _dealNativeToken(alice, initialNativeLiquidity);
    vm.startPrank(alice);
    locker.initializeCollection(address(erc721), initialNativeLiquidity, _tokenIds, 0, SQRT_PRICE_1_1);
    vm.stopPrank();



    //### [ALICE] ADDING CTOKEN FEES TO HOOK
    //-> alice deposits an NFT to get 1e18 CTOKEN and then deposits 1e18 CTOKENS as fees in the UniswapImplementation hook
    vm.startPrank(alice);
    locker.deposit(address(erc721), _tokenIdToDepositAlice, alice);
    uniswapImplementation.depositFees(address(erc721), 0, 1e18);
    vm.stopPrank();



    //### [BOB] SWAP FAILS 
    _dealNativeToken(bob, 1e18);

    //-> bob swaps `1e18` NATIVE tokens for CTOKENS but the swap fails
    vm.startPrank(bob);
    poolSwap.swap(
        PoolKey({
            currency0: Currency.wrap(address(ctoken)),
            currency1: Currency.wrap(address(WETH)),
            fee: LPFeeLibrary.DYNAMIC_FEE_FLAG,
            tickSpacing: 60,
            hooks: IHooks(address(uniswapImplementation))
        }),
        IPoolManager.SwapParams({
            zeroForOne: false,
            amountSpecified: -int(1e18),
            sqrtPriceLimitX96: (TickMath.MAX_SQRT_PRICE - 1)
        }),
        PoolSwapTest.TestSettings({
            takeClaims: false,
            settleUsingBurn: false
        }),
        ''
    );
}
```


### Mitigation

In [UniswapImplementation::beforeSwap()](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/implementation/UniswapImplementation.sol#L547) check against `ethIn` instead of `tokenOut`:

```solidity
@>  if (ethIn <= uint(-params.amountSpecified)) {
        // Update our hook delta to reduce the upcoming swap amount to show that we have
        // already spent some of the ETH and received some of the underlying ERC20.
        // Specified = exact input (ETH)
        // Unspecified = token1
        beforeSwapDelta_ = toBeforeSwapDelta(ethIn.toInt128(), -tokenOut.toInt128());
    } else {
        ethIn = tokenOut = 0;
    }
```

# Issue M-16: User can cancel or modify Dutch auctions, compromising market integrity and user trust 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/520 

## Found by 
0xNirix
### Summary

A lack of listing type preservation during relisting can be exploited by by users to modify or cancel Dutch auctions, violating core protocol principles.


### Root Cause

In Listings.sol, there's a critical oversight in the relist function that allows bypassing restrictions on Dutch auctions and Liquidation listings. Here's a detailed walkthrough:

The modifyListings and cancelListings functions have checks to prevent modification of Dutch auctions:
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L312 and https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L430

```solidity
function modifyListings(address _collection, ModifyListing[] calldata _modifyListings, bool _payTaxWithEscrow) public nonReentrant lockerNotPaused returns (uint taxRequired_, uint refund_) {
    // ...
    if (getListingType(listing) != Enums.ListingType.LIQUID) revert InvalidListingType();
    // ...
}

function cancelListings(address _collection, uint[] memory _tokenIds, bool _payTaxWithEscrow) public lockerNotPaused {
    // ...
    if (getListingType(listing) != Enums.ListingType.LIQUID) revert CannotCancelListingType();
    // ...
}
```

However, the relist function lacks these checks:

```solidity
function relist(CreateListing calldata _listing, bool _payTaxWithEscrow) public nonReentrant lockerNotPaused {
    // Read the existing listing in a single read
    Listing memory oldListing = _listings[_collection][_tokenId];

    // Ensure the caller is not the owner of the listing
    if (oldListing.owner == msg.sender) revert CallerIsAlreadyOwner();

    // ... price difference payment logic ...

    // We can process a tax refund for the existing listing
    (uint _fees,) = _resolveListingTax(oldListing, _collection, true);

    // Validate our new listing
    _validateCreateListing(_listing);

    // Store our listing into our Listing mappings
    _listings[_collection][_tokenId] = listing;

    // Pay our required taxes
    payTaxWithEscrow(address(collectionToken), getListingTaxRequired(listing, _collection), _payTaxWithEscrow);

    // ... events ...
}
```
This function allows any non-owner to relist an item, potentially changing its type from Dutch or liquidation to liquid. The attacker suffers minimal loss due to:

a) Tax refund for the old listing:
```solidity
(uint _fees,) = _resolveListingTax(oldListing, _collection, true);
```
b) Immediate cancellation after relisting, which refunds most of the new listing's tax:
```solidity
function _resolveListingTax(Listing memory _listing, address _collection, bool _action) private returns (uint fees_, uint refund_) {
    // ...
    if (block.timestamp < _listing.created + _listing.duration) {
        refund_ = (_listing.duration - (block.timestamp - _listing.created)) * taxPaid / _listing.duration;
    }
    // ...
}
```
This oversight allows attackers to bypass the intended restrictions on Dutch auctions and Liquidation listings, violating the core principle of auction immutability with minimal financial loss.

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

1. Attacker creates a Dutch auction or liquidation listing from Wallet A.
2. Attacker uses Wallet B to call relist, converting the Dutch auction to a Liquid listing by modifying the duration.
3. Attacker immediately cancels the new liquid listing using cancelListings at minimal cost.

### Impact

The NFT holders and potential bidders suffer a loss of trust and market efficiency. The attacker gains the ability to manipulate auctions, potentially extracting value by gaming the system. This violates the core principle stated in the whitepaper like around  expiry of a liquid auction - `the item is immediately locked into a dutch auction where the price falls from its current floor multiple down to the floor (1x) over a period of 4 days.` However, as seen in this issue, the lock can be broken by just relisting and then cancelling. The issue affects not only Dutch auctions but also liquidation listings which can be similarly canceled or modified and is not properly handled.  For example, in case a liquidation listing is cancelled, the `_isLiquidation[_collection][_tokenId]` flag is not cleared causing further issues.


### PoC

_No response_

### Mitigation

_No response_

# Issue M-17: Reserving a listing checkpoints the collection's `compoundFactor` at an intermediary higher compound factor 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/533 

## Found by 
Sentryx
### Summary

When a listing is reserved (**Listings**#`reserve()`) there are multiple CollectionToken operations that affect its `totalSupply` that take place in the following order: transfer → transfer → burn → mint → transfer → burn. After the function ends execution the `totalSupply` of the CollectionToken itself remains unchanged compared to before the call to the function, but in the middle of its execution a protected listing is created and its compound factor is checkpointed at an intermediary state of the CollectionToken's total supply (between the first burn and the mint) that will later affect the rate of interest accrual on the loan itself in harm to all borrowers of NFTs in that collection causing them to actually accrue more interest on the loan.

### Root Cause

To be able to understand the issue, we must inspect what CollectionToken operations are performed throughout the execution of the `reserve()` function and at which point exactly the protected listing's `compoundFactor` is checkpointed.

(Will comment out the irrelevant parts of the function for brevity)
https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/Listings.sol#L690-L759
```solidity
    function reserve(address _collection, uint _tokenId, uint _collateral) public nonReentrant lockerNotPaused {
        // ...
        
        if (oldListing.owner != address(0)) {
            // We can process a tax refund for the existing listing if it isn't a liquidation
            if (!_isLiquidation[_collection][_tokenId]) {
                // 1st transfer
→               (uint _fees,) = _resolveListingTax(oldListing, _collection, true);
                if (_fees != 0) {
                    emit ListingFeeCaptured(_collection, _tokenId, _fees);
                }
            }
            
            // ...
            
            if (listingPrice > listingFloorPrice) {
                unchecked {
                    // 2nd transfer
→                   collectionToken.transferFrom(msg.sender, oldListing.owner, listingPrice - listingFloorPrice);
                }
            }
            
            // ...
        }

        // 1st burn
→       collectionToken.burnFrom(msg.sender, _collateral * 10 ** collectionToken.denomination());

        // ...

        // the protected listing is recorded in storage with the just-checkpointed compoundFactor
        // then: mint + transfer
→       protectedListings.createListings(createProtectedListing);

        // 2nd burn
→       collectionToken.burn((1 ether - _collateral) * 10 ** collectionToken.denomination());

        // ...
    }
```

Due to the loan's `compoundFactor` being checkpointed before the second burn of `1 ether - _collateral` CollectionTokens (and before `listingCount[listing.collection]` is incremented) , the `totalSupply` will be temporarily decreased which will make the collection's utilization ratio go up a notch due to the way it's derived and this will eventually be reflected in the checkpointed `compoundFactor` for the current block and respectively for the loan as well.

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L117-L156
```solidity
    function createListings(CreateListing[] calldata _createListings) public nonReentrant lockerNotPaused {
        // ...
        
        for (uint i; i < _createListings.length; ++i) {
            // ...
            
            if (checkpointIndex == 0) {
                // @audit Checkpoint the temporarily altered `compoundFactor` due to the temporary
                // change in the CollectionToken's `totalSupply`.
→               checkpointIndex = _createCheckpoint(listing.collection);
                assembly { tstore(checkpointKey, checkpointIndex) }
            }

            // ...
            
            // @audit Store the listing with a pointer to the index of the inacurate checkpoint above
→           tokensReceived = _mapListings(listing, tokensIdsLength, checkpointIndex) * 10 ** locker.collectionToken(listing.collection).denomination();

            // Register our listing type
            unchecked {
                listingCount[listing.collection] += tokensIdsLength;
            }

            // ...
        }
    }
```

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L530-L571
```solidity
    function _createCheckpoint(address _collection) internal returns (uint index_) {
→       Checkpoint memory checkpoint = _currentCheckpoint(_collection);

        // ...
        
        collectionCheckpoints[_collection].push(checkpoint);
    }
```

`_currentCheckpoint()` will fetch the current utilization ratio which is temporarily higher and will calculate the current checkpoint's `compoundedFactor` with it (which the newly created loan will reference thereafter).

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L580-L596
```solidity
    function _currentCheckpoint(address _collection) internal view returns (Checkpoint memory checkpoint_) {
        // ...
→       (, uint _utilizationRate) = utilizationRate(_collection);

        // ...
        
        checkpoint_ = Checkpoint({
→           compoundedFactor: locker.taxCalculator().calculateCompoundedFactor({
                _previousCompoundedFactor: previousCheckpoint.compoundedFactor,
                _utilizationRate: _utilizationRate,
                _timePeriod: block.timestamp - previousCheckpoint.timestamp
            }),
            timestamp: block.timestamp
        });
    }
```

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L261-L276
```solidity
    function utilizationRate(address _collection) public view virtual returns (uint listingsOfType_, uint utilizationRate_) {
        listingsOfType_ = listingCount[_collection];
        // ...
        if (listingsOfType_ != 0) {
            // ...
→           uint totalSupply = collectionToken.totalSupply();
            if (totalSupply != 0) {
→               utilizationRate_ = (listingsOfType_ * 1e36 * 10 ** collectionToken.denomination()) / totalSupply;
            }
        }
    }
```


### Internal pre-conditions

None

### External pre-conditions

None

### Attack Path

No attack required.


### Impact

Knowing how a collection's utilization rate is calculated we can clearly see the impact it'll have on the checkpointed compounded factor for a block:

$utilizationRate = \dfrac{collection\ protected\ listings\ count\ *\ 1e36\ *\ 10^{denomination}}{CT\ total\ supply}$

The less CollectionToken (CT) total supply, the higher the utilization rate for a constant collection's protected listings count. The higher the utilization rate, the higher the `compoundedFactor` will be for the current checkpoint and for the protected position created (the loan). 

$`compoundedFactor = \dfrac{previousCompoundFactor\ *\ (1e18 + (perSecondRate\ / 1000 * \_timePeriod))}{1e18}`$
Where:
$perSecondRate = \dfrac{interestRate * 1e18}{365 * 24 * 60 * 60}$

$interestRate = 200 + \dfrac{utilizationRate * 600}{0.8e18}$ – When `utilizationRate` ≤ 0.8e18 (`UTILIZATION_KINK`)
OR
$interestRate = (\dfrac{(utilizationRate - 200) * (100 - 8)}{1e18 - 200} + 8) * 100$ – When `utilizationRate` > 0.8e18 (`UTILIZATION_KINK`)

As a result (and with the help of another issue that has a different root cause and a fix which is submitted separately) the loan will end up checkpointing a temporarily higher `compoundedFactor`  and thus will compound more interest in the future than it's correct to. It's important to know that no matter how many times `createCheckpoint()` is called after the call to `reserve()`, the `compoundFactor` for the current block's checkpoint will remain as. But even without that, there is **no guarantee** that even if it worked correctly, there'd by any calls that'd record a new checkpoint for that collection.


### PoC

1. Bob lists an NFT for sale. The `duration` and the `floorMultiple` of the listing are irrelevant in this case.
2. John sees the NFT and wants to reserve it, putting up $0.9e18$ amount of CollectionTokens as `_collateral`.
3. The `_collateral` is burned.
4. The collection's `compoundFactor` for the current block is checkpointed.

Let's say there is only one protected listing prior to John's call to `reserve()` and its owner has put up $0.5e18$ CollectionTokens as collateral.

$old\ collection\ token\ total\ supply = 5e18$
$collection\ protected\ listings\ count = 1$

We can now calculate the utilization rate the way it's calculated right now:

$utilizationRate = \dfrac{collection\ protected\ listings\ count\ * 1e36 * 10^{denomination}}{CT\ total\ supply}$

$`utilizationRate = \dfrac{1*1e36*10^0}{5e18 - 0.9e18}`$ (assuming `denomination` is 0)

$utilizationRate = \dfrac{1e36}{4.1e18} = 243902439024390243$

We can now proceed to calculate the wrong compounded factor:

$`compoundedFactor = \dfrac{previousCompoundFactor\ *\ (1e18 + (perSecondRate\ / 1000 * \_timePeriod))}{1e18}`$

**Where**:
$previousCompoundFactor = 1e18$
$interestRate = 200 + \dfrac{utilizationRate * 600}{0.8e18} = 200 + \dfrac{243902439024390243 * 600}{0.8e18} = 382$ (3.82 %)

$perSecondRate = \dfrac{interestRate * 1e18}{365 * 24 * 60 * 60} = \dfrac{382 * 1e18}{31 536 000} = 12113140537798$
$timePeriod = 432000\ (5\ days)$ (last checkpoint was made 5 days ago)

$compoundedFactor = \dfrac{1e18 * (1e18 + (12113140537798 / 1000 * 432000))}{1e18}$

$compoundedFactor = \dfrac{1e18 * 1005232876711984000}{1e18} = 1005232876711984000$ (This will be the final compound factor for the checkpoint for the current block)

The correct utilization rate however, should be calculated with a current collection token total supply of $5e18$ at the time when `reserve()` is called, which will result in:
$`utilizationRate = \dfrac{collection\ protected\ listings\ count\ *\ 1e36\ *\ 10^{denomination}}{CT\ total\ supply} = \dfrac{1 * 1e36}{5e18} = 200000000000000000`$

The difference with the wrong utilization rate is $`43902439024390243`$ or ~$`0.439e18`$ which is ~18% smaller than the wrongly computed utilization rate).

From then on the interest rate will be lower and thus the final and correct compounded factor comes out at $1004794520547808000$ (will not repeat the formulas for brevity) which is around 0.05% smaller than the wrongly recorded compounded factor. The % might not be big but remember that this error will be bigger with the longer time period between the two checkpoints and will be compounding with every call to `reserve()`.

5. A protected listing is created for the reserved NFT, referencing the current checkpoint.
6. When the collection's `compoundFactor` is checkpointed the next time, the final `compoundFactor` product will be times greater due to the now incremented collection's protected listings count and the increased (back to the value before the reserve was made) total supply of CollectionTokens.

Lets say after another 5 days the `createCheckpoint()` method is called for that collection without any changes in the CollectionToken total supply or the collection's protected listings count. The math will remain mostly the same with little updates and we will first run the math with the wrongly computed $previousCompoundedFactor$ and then will compare it to the correct one.

$collection\ token\ total\ supply = 5e18$ (because the burned `_collateral` amount of CollectionTokens has been essentially minted to the **ProtectedListings** contract hence as we said `reserve()` does **not** affect the total supply after the function is executed).
$collection\ protected\ listings\ count = 2$ (now 1 more due to the created protected listing)
$previousCompoundedFactor = 1005232876711984000$ (the wrong one, as we derived it a bit earlier)
$utilizationRate = \dfrac{collection\ protected\ listings\ count\ *\ 1e36\ *\ 10^{denomination}}{CT\ total\ supply}$

$utilizationRate = \dfrac{2 * 1e36 * 10^0}{5e18} = \dfrac{2e36}{5e18} = 0.4e18$

$interestRate = 200 + \dfrac{utilizationRate * 600}{0.8e18} = 200 + \dfrac{0.4e18 * 600}{0.8e18} = 500$ (5 %)
$perSecondRate = \dfrac{interestRate * 1e18}{365 * 24 * 60 * 60} = \dfrac{500 * 1e18}{31 536 000} = 15854895991882$
$timePeriod = 432000\ (5\ days)$ (the previous checkpoint was made 5 days ago)

$compoundedFactor = \dfrac{1005232876711984000 * (1e18 + (15854895991882 / 1000 * 432000)}{1e18}$

$compoundedFactor = \dfrac{1005232876711984000 * 1006849315068112000}{1e18} = 1012118033401408964$ or 0.68% accrued interest for that collection for the past 5 days.

Now let's run the math but compounding on top of the correct compound factor:

$compoundedFactor = \dfrac{1004794520547808000 * 1006849315068112000}{1e18} = 1011676674797752473$ or 0.21% of interest should've been accrued for that collection for the past 5 days, instead of 0.68% which in this case is 3 times bigger.


### Mitigation

Just burn the `_collateral` amount after the protected listing is created. This way the `compoundedFactor` will be calculated and checkpointed properly.

```diff
diff --git a/flayer/src/contracts/Listings.sol b/flayer/src/contracts/Listings.sol
index eb39e7a..c8eac4d 100644
--- a/flayer/src/contracts/Listings.sol
+++ b/flayer/src/contracts/Listings.sol
@@ -725,10 +725,6 @@ contract Listings is IListings, Ownable, ReentrancyGuard, TokenEscrow {
             unchecked { listingCount[_collection] -= 1; }
         }
 
-        // Burn the tokens that the user provided as collateral, as we will have it minted
-        // from {ProtectedListings}.
-        collectionToken.burnFrom(msg.sender, _collateral * 10 ** collectionToken.denomination());
-
         // We can now pull in the tokens from the Locker
         locker.withdrawToken(_collection, _tokenId, address(this));
         IERC721(_collection).approve(address(protectedListings), _tokenId);
@@ -750,6 +746,10 @@ contract Listings is IListings, Ownable, ReentrancyGuard, TokenEscrow {
         // Create our listing, receiving the ERC20 into this contract
         protectedListings.createListings(createProtectedListing);
 
+        // Burn the tokens that the user provided as collateral, as we will have it minted
+        // from {ProtectedListings}.
+        collectionToken.burnFrom(msg.sender, _collateral * 10 ** collectionToken.denomination());
+
         // We should now have received the non-collateral assets, which we will burn in
         // addition to the amount that the user sent us.
         collectionToken.burn((1 ether - _collateral) * 10 ** collectionToken.denomination());

```


# Issue M-18: Compounded factor for the current block's checkpoint will not be updated and allows for locking in manipulated compounded factor for any new checkpoint 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/540 

## Found by 
Sentryx
### Summary

Due to the way the **ProtectedListings**#`_currentCheckpoint()` calculates the `_timePeriod` for which to calculate the compounded factor, the `compoundFactor` for the current block's checkpoint will actually **not** be updated in consecutive calls to `_createCheckpoint()` within the same block and will remain as recorded initially.


### Root Cause

As `_createCheckpoint()` is called for the first time in a block, a new **Checkpoint** struct will be pushed to the `collectionCheckpoints[_collection]` array of checkpoints. The next time `_createCheckpoint()` is called in the same block, `previousCheckpoint` in the `_currentCheckpoint()` function will now be the last checkpoint we've just created. And it will have `timestamp` = `block.timestamp`.

When calculating the current checkpoint, the `_currentCheckpoint()` function will calculate the accrued interest for the `_timePeriod` between the `block.timestamp` and `previousCheckpoint.timestamp`, which will work for the first time the checkpoint is recorded for the current block, but on consecutive calls the `_timePeriod` will evaluate to 0 due to `block.timestamp = previousCheckpoint.timestamp` and thus their difference will be 0. And when `_timePeriod` is 0 no interest will be compounded resulting in the checkpoint's compound factor staying the same.

The code obviously intends to update the compound factor of the current block's checkpoint if one exists but will fail to do so.

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L530-L571
```solidity
    function _createCheckpoint(address _collection) internal returns (uint index_) {
        // Determine the index that will be created
        index_ = collectionCheckpoints[_collection].length;

        // ...

        // Get our new (current) checkpoint
→       Checkpoint memory checkpoint = _currentCheckpoint(_collection);

→       // If no time has passed in our new checkpoint, then we just need to update the
→       // utilization rate of the existing checkpoint.
        if (checkpoint.timestamp == collectionCheckpoints[_collection][index_ - 1].timestamp) {
            // @audit Will NOT change the compoundedFactor's value.
→           collectionCheckpoints[_collection][index_ - 1].compoundedFactor = checkpoint.compoundedFactor;
            return index_;
        }

        // Store the new (current) checkpoint
        collectionCheckpoints[_collection].push(checkpoint);
    }
```

https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/ProtectedListings.sol#L580-L596
```solidity
    function _currentCheckpoint(address _collection) internal view returns (Checkpoint memory checkpoint_) {
        // Calculate the current interest rate based on utilization
        (, uint _utilizationRate) = utilizationRate(_collection);

        // Update the compounded factor with the new interest rate and time period
        Checkpoint memory previousCheckpoint = collectionCheckpoints[_collection][collectionCheckpoints[_collection].length - 1];

        // Save the new checkpoint
        checkpoint_ = Checkpoint({
            compoundedFactor: locker.taxCalculator().calculateCompoundedFactor({
                _previousCompoundedFactor: previousCheckpoint.compoundedFactor,
                _utilizationRate: _utilizationRate,
                // @audit Evaluates to 0 from the second time on this is being called within the same block and thus compounding
                // the {_previousCompoundFactor} by 1 – essentially not changing it.
→               _timePeriod: block.timestamp - previousCheckpoint.timestamp
            }),
            timestamp: block.timestamp
        });
    }
```


### Internal pre-conditions

None

### External pre-conditions

None

### Attack Path

The error in compounded factor calculation will happen by itself alone. However, this can also be exploited intentionally. 

$utilizationRate = \dfrac{collection\ protected\ listings\ count\ *\ 1e36\ *\ 10^{denomination}}{CT\ total\ supply}$

So as evident from the formula above – for a constant collection protected listings count, the higher the CollectionToken (CT) total supply for a collection, the lower the utilization rate. And the lower the CT total supply – the higher the utilization rate.

A few scenarios we came up with:
1. A malicious actor can use it to lock in a higher `compoundedFactor` for the current block's checkpoint forcing users that have protected listings in that collection to accrue more interest over the same period of time as they can control the `utilizationRate` of a collection. The attacker can deposit NFTs directly to the **Locker** in one block then in the next block redeem them to bring up the `utilizationRate` and then call **ProtectedListings**#`createCheckpoint()` to lock in a higher compounded factor for that block. After that they can deposit back the NFTs in the same block so they are able to repeat this attack for as long as they wish.
2. A malicious actor can deposit NFTs at the beginning of a block to bring down the `utilizationRate` of a collection and then call **ProtectedListings**#`createCheckpoint()` to lock in a lower `compoundedFactor` for the collection's current block checkpoint. Thus the attacker and other users will accrue less interest on their protected listings (loans). After this they can redeem the listings in the **Locker** and get their NFTs back.


### Impact

Due to some actions in the **Locker** contract **NOT** checkpointing a collection's compounded factor – `deposit()`, `redeem()` and `unbackedDeposit()` (which is an issue of its own with a different root cause), an attacker or an unsuspecting user can lock in the current block's checkpoint for a collection at a better or worse `compoundedFactor` than what the actual compounded factor for that collection will be at the end of the block when more actions that affect its utilization rate are performed.

$compoundedFactor = \dfrac{previousCompoundFactor\ *\ (1e18 + (perSecondRate\ / 1000 * timePeriod))}{1e18}$

Where:
$perSecondRate = \dfrac{interestRate * 1e18}{365 * 24 * 60 * 60}$

$interestRate = 200 + \dfrac{utilizationRate * 600}{0.8e18}$ – When `utilizationRate` ≤ 0.8e18 (`UTILIZATION_KINK`)
OR
$interestRate = (\dfrac{(utilizationRate - 200) * (100 - 8)}{1e18 - 200} + 8) * 100$ – When `utilizationRate` > 0.8e18 (`UTILIZATION_KINK`)

Which shows the `compoundedFactor` is a function of the `utilizationRate` (and the `timePeriod` and the `previousCompoundFactor`) which is controllable by anyone due to another issue with a different root cause, and as outlined in the **Attack Path** section, for a constant collection protected listings count, the higher the CollectionToken (CT) total supply for a collection, the lower the utilization rate. And the lower the CT total supply – the higher the utilization rate.

As a result owners of protected listings in a given collection can be made to pay more interest on their loans or the borrowers themselves can lower the interest they are supposed to pay on their loans.


### PoC

See **Attack Path** and **Impact**.


### Mitigation

```diff
diff --git a/flayer/src/contracts/ProtectedListings.sol b/flayer/src/contracts/ProtectedListings.sol
index 92ac03a..575e3ff 100644
--- a/flayer/src/contracts/ProtectedListings.sol
+++ b/flayer/src/contracts/ProtectedListings.sol
@@ -584,6 +584,13 @@ contract ProtectedListings is IProtectedListings, ReentrancyGuard {
         // Update the compounded factor with the new interest rate and time period
         Checkpoint memory previousCheckpoint = collectionCheckpoints[_collection][collectionCheckpoints[_collection].length - 1];
 
+        // If there is already a checkpoint for this block, actually get the truly previous
+        // checkpoint which is second to last as in the beginning of a block we've pushed the
+        // current new checkpoint to the array.
+        if (previousCheckpoint.timestamp == block.timestamp) {
+            previousCheckpoint = collectionCheckpoints[_collection][collectionCheckpoints[_collection].length - 2];
+        }
+
         // Save the new checkpoint
         checkpoint_ = Checkpoint({
             compoundedFactor: locker.taxCalculator().calculateCompoundedFactor({

```


# Issue M-19: FTokens are burned after `quorumVotes` are recorded making a portion of the shares unclaimable 

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/610 

## Found by 
g, zarkk01
### Summary

`params.quorumVotes` is larger than it should because Locker's collection tokens are [burned](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L245-L251) after `quorumVotes` is recorded.

```solidity
    uint newQuorum = params.collectionToken.totalSupply() * SHUTDOWN_QUORUM_PERCENT / ONE_HUNDRED_PERCENT;
    if (params.quorumVotes != newQuorum) {
        params.quorumVotes = uint88(newQuorum);
    }

    // Lockdown the collection to prevent any new interaction
    // @audit sunsetCollection() burns the Locker's tokens decreasing the `totalSupply`. If this was called before
    // `params.quorumVotes` was set, the totalSupply() would be smaller and each vote would get more share of the ETH profits.
    locker.sunsetCollection(_collection);
```

When a collection is shut down, its NFTs are sold off via Dutch auction in a SudoSwap pool. This sale's ETH profits are distributed to the holders of the collection token (fToken).

The [claim amount](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L310) is calculated with `availableClaim * claimableVotes / params.quorumVotes * 2`.  The higher `params.quorumVotes`, the lower the claim amount for each vote. 

### Root Cause

`params.quorumVotes` is larger than it should because Locker's collection tokens are [burned](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L245-L251) after `quorumVotes` is recorded.

### Internal pre-conditions

1. At least 50% of Holders have [voted](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L208-L209) for a collection to shut down. 
2. No [listings](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L241) for the target collection exist. 

### External pre-conditions

None

### Attack Path

1. Admin calls [`CollectionShutdown::execute()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L231). The vulnerability always happens when this is called.

### Impact

A portion of the claimable ETH can never be claimed and all voters can claim less ETH.

### PoC

_No response_

### Mitigation

Consider calling `Locker::sunsetCollection()` before setting the `params.quorumVotes` in [`CollectionShutdown::execute()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L245-L251).

