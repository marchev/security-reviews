# [H-1] `ZivoeRewardsVesting` Miscalculation when revoking vesting schedule incorrectly grants users the entire vesting schedule voting power

## Summary

Due to an incorrect calculation, revoking a vesting schedule grants the user full voting power associated with the entire vesting schedule instead of the amount that has vested up to that point.

## Vulnerability Detail

The Zivoe protocol allows vesting schedules to be revoked if `revokable` is set to `true`:

https://github.com/sherlock-audit/2024-03-zivoe-marchev/blob/main/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L65

When a vesting schedule is revoked, the user must only be granted the tokens and voting power that has vested up to that point. Although the amount of tokens to be granted is calculated correctly, there is a bug in the calculation of the voting power, resulting in the user receiving the full voting power associated with the vesting schedule.

The following scenario illustrates the issue:

1. Alice, an early contributor, is granted a vesting schedule with the following terms:
	- Amount to vest: **1,000,000 $ZVE**
	- Days to vest: **100 days**
	- Days to cliff: **0 days**
	- Revokable: **Yes**
2. On the 10th day the Zivoe protocol decides to revoke Alice's vesting schedule due to misconduct.

**Expected outcome:** Alice should receive 100,000 ZVE and thus a voting power of 100,000.

**Actual outcome:** Alice receives the correct amount of tokens (100,000 ZVE) but also retains the full voting power of **1,000,000** tokens.

Let us see why this happens. The problem lies in `revokeVestingSchedule()` and in particular the logic that updates the account voting power checkpoint:

```solidity
    function revokeVestingSchedule(address account) external updateReward(account) onlyZVLOrITO nonReentrant {
        // ...
        uint256 amount = amountWithdrawable(account);
        uint256 vestingAmount = vestingScheduleOf[account].totalVesting;
        // ...
        _writeCheckpoint(_checkpoints[account], _subtract, amount); //@audit Incorrect voting power calculation
		// ...
}
```

The implementation here incorrectly subtracts the account's withdrawable amount. In Alice's case, the calculation looks like this:

$$
\_checkpoints[alice] = 1,000,000 - 100,000 = 900,000
$$

This causes Alice to incorrectly end up with a voting power of 900,000 in the `ZivoeRewardsVesting` contract.

The flaw manifests in the `ZivoeGovernorV2` contract where the voting power is calculated as the sum of ZVE, vestZVE and stZVE voting power:

https://github.com/sherlock-audit/2024-03-zivoe-marchev/blob/e95cf46deabb1c019da47e774684164dc3e0b01f/zivoe-core-foundry/src/ZivoeGovernorV2.sol#L149-L151

In our example scenario, Alice has 100,000 ZVE plus 900,000 vestZVE:

$$
aliceVotingPower = 100,000\ ZVE + 900,000\ vestZVE = 1,000,000
$$

Note: it's only the vestZVE voting power that is calculated incorrectly, the actual token tokens are calculated and granted as expected.

### PoC

The following coded PoC demonstrates the issue. Add the following test to `Test_ZivoeRewardsVesting.sol`

```solidity
    function test_ZivoeRewardsVesting_revokeVestingScheduleDoesNotRevokeRemainingVotingPower() public {
        uint256 daysToVest = 100;
        uint256 amountToVest = 1_000_000 ether;

        assert(zvl.try_createVestingSchedule(
            address(vestZVE), 
            address(moe), 
            0, // Days to cliff 
            daysToVest,
            amountToVest,
            true // Revokable
        ));

        hevm.roll(block.number + 100);

        // Pre-state
        assertEq(ZVE.balanceOf(address(moe)), 0);
        assertEq(vestZVE.balanceOf(address(moe)), 1_000_000 ether);
        assertEq(vestZVE.getVotes(address(moe)), 1_000_000 ether);
        assertEq(GOV.getVotes(address(moe), block.number - 1), 1_000_000 ether);

        // warp time to 10 days after the vesting started
        hevm.warp(block.timestamp + 10 days); // At this point only 10% of the tokens would be vested

        // Revoke vesting
        assert(zvl.try_revokeVestingSchedule(address(vestZVE), address(moe)));

        hevm.prank(address(moe));
        ZVE.delegate(address(moe)); // Delegate to oneself to enable voting power tracking
        
        hevm.roll(block.number + 100);

        // Post-state
        //@audit We use approx. equality due to precision loss in vestingPerSecond, delta % = 0.01%
        assertApproxEqRel(ZVE.balanceOf(address(moe)), 100_000 ether, 0.0001e18); //@audit Amount of vested ZVE tokens is correct
        assertEq(vestZVE.balanceOf(address(moe)), 0); //@audit Amount of vestZVE token is correct => 0 since vesting is revoked
        assertApproxEqRel(vestZVE.getVotes(address(moe)), 900_000 ether, 0.0001e18); //@audit Amount of vestZVE votes is NOT correct, should be 0
        assertEq(GOV.getVotes(address(moe), block.number - 1), 1_000_000 ether); //@audit Amount of governance votes is NOT correct, should be 100_000 but is 1_000_000
    }
```

Use the following command to run the PoC:

```sh
forge test --mt test_ZivoeRewardsVesting_revokeVestingScheduleDoesNotRevokeRemainingVotingPower --fork-url $MAINNET_RPC_URL -vvvvv
```

## Impact

This vulnerability allows users whose vesting schedules have been revoked to retain full voting power, disproportionately influencing governance decisions and potentially leading to governance attacks or manipulation.

The integrity of the governance system is compromised, as the vulnerability in voting power calculation could skew voting outcomes, undermining the governance system’s credibility and the protocol’s intended fairness.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe-marchev/blob/e95cf46deabb1c019da47e774684164dc3e0b01f/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L453

https://github.com/sherlock-audit/2024-03-zivoe-marchev/blob/e95cf46deabb1c019da47e774684164dc3e0b01f/zivoe-core-foundry/src/ZivoeGovernorV2.sol#L149-L151

## Tool used

Manual Review

## Recommendation

The `account` voting power in `createVestingSchedule()` must be set to `0`. This could be implemented like that:

```diff
diff --git a/zivoe-core-foundry/src/ZivoeRewardsVesting.sol b/zivoe-core-foundry/src/ZivoeRewardsVesting.sol
index 865fdd1..2a280ec 100644
--- a/zivoe-core-foundry/src/ZivoeRewardsVesting.sol
+++ b/zivoe-core-foundry/src/ZivoeRewardsVesting.sol
@@ -450,7 +450,7 @@ contract ZivoeRewardsVesting is ReentrancyGuard, Context, ZivoeVotes {

         _totalSupply = _totalSupply.sub(vestingAmount);
         _writeCheckpoint(_totalSupplyCheckpoints, _subtract, vestingAmount);
-        _writeCheckpoint(_checkpoints[account], _subtract, amount);
+        _writeCheckpoint(_checkpoints[account], _subtract, getVotes(account));
         _balances[account] = 0;
         stakingToken.safeTransfer(account, amount);
```

# [H-2] `ZivoeRewardsVesting` Flawed total supply accounting may lead to failed or exploitable vesting schedule revocations

## Summary

An error in the accounting of the vestZVE `_totalSupply` lead to a failure in revoking the last vesting schedule if any tokens have already been withdrawn from it.

## Vulnerability Detail

The `ZivoeRewardsVesting` contract, which manages the vestZVE tokens, uses `_totalSupply` to track the total amount of non-transferrable vestZVE tokens. The `_totalSupply` is updated during:

- `createVestingSchedule()`
- `withdraw()`
- `revokeVestingSchedule()`

While `createVestingSchedule()` and `withdraw()` handle updates correctly, `revokeVestingSchedule()` has flawed accounting. The method subtracts the entire original vesting amount from `_totalSupply`, not taking into consideration any vested tokens that have already been withdrawn:

```solidity
    function revokeVestingSchedule(address account) external updateReward(account) onlyZVLOrITO nonReentrant {
        // ...
        uint256 vestingAmount = vestingScheduleOf[account].totalVesting;
        // ...
        _totalSupply = _totalSupply.sub(vestingAmount);
        // ...
    }
```

This oversight can lead to an integer underflow if the vesting schedule is the last remaining in the contract and any tokens have been withdrawn. Such an underflow causes the transaction to revert, potentially allowing a user to exploit this vulnerability to prevent the revocation of their schedule.

#### Example scenario

**Pre-requisite:** Only one vesting schedule remains in the contract.

1. Alice is granted a vesting schedule with the following terms:
	- Amount to vest: **1,000,000 ZVE**
	- Days to vest: **100 days**
	- Days to cliff: **0 days**
	- Revokable: **Yes**
2. On day 10, Alice withdraws 100,000 ZVE.
3. The Zivoe protocol attempts to revoke Alice's vesting schedule.

**Expected outcome:** The vesting schedule is successfully revoked.

**Actual outcome:** The revocation fails due to integer underflow. This could occur even if only 1 wei has been previously withdrawn.

#### PoC

Here is a coded PoC that demonstrates the issue. Simply add the following test case to `Test_ZivoeRewardsVesting.sol`:

```solidity
    function test_ZivoeRewardsVesting_revokeVestingScheduleCanBeGriefed() public {
        uint256 daysToVest = 100;
        uint256 amountToVest = 1_000_000 ether;

        assert(zvl.try_createVestingSchedule(
            address(vestZVE), 
            address(moe), 
            0,
            daysToVest,
            amountToVest,
            true
        ));

        hevm.warp(block.timestamp + 10 days); // At this point only 10% of the tokens would be vested

        // Frontrunning revoke() or unsuspectingly withdrawing vested tokens
        hevm.prank(address(moe));
        vestZVE.withdraw();

        // Revoke vesting
        assert(zvl.try_revokeVestingSchedule(address(vestZVE), address(moe)));
    }
```

Run it with:

```sh
forge test --mt test_ZivoeRewardsVesting_revokeVestingScheduleCanBeGriefed --fork-url $MAINNET_RPC_URL -vvvvv
```

## Impact

This issue can lead to the failure of revoking the last vesting schedule if any tokens have been withdrawn, preventing the protocol from correctly managing vesting terminations.

Users might exploit this vulnerability to block the revocation of their vesting schedules, undermining the protocol's control over its token distribution.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L451

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L440

## Tool used

Manual Review

## Recommendation

Take into consideration any previous withdrawals when updating `_totalSupply` in `revokeVestingSchedule()`. The logic can be fixed like that:

```diff
diff --git a/zivoe-core-foundry/src/ZivoeRewardsVesting.sol b/zivoe-core-foundry/src/ZivoeRewardsVesting.sol
index 865fdd1..d735041 100644
--- a/zivoe-core-foundry/src/ZivoeRewardsVesting.sol
+++ b/zivoe-core-foundry/src/ZivoeRewardsVesting.sol
@@ -448,7 +448,7 @@ contract ZivoeRewardsVesting is ReentrancyGuard, Context, ZivoeVotes {

         vestingTokenAllocated -= (vestingAmount - vestingScheduleOf[account].totalWithdrawn);

-        _totalSupply = _totalSupply.sub(vestingAmount);
+        _totalSupply = _totalSupply.sub(vestingAmount - vestingScheduleOf[account].totalWithdrawn);
         _writeCheckpoint(_totalSupplyCheckpoints, _subtract, vestingAmount);
         _writeCheckpoint(_checkpoints[account], _subtract, amount);
         _balances[account] = 0;
```

# [H-3] `ZivoeRewards` Anyone can prolong the reward distribution time

## Summary

Anyone can prolong the time for reward distribution with as little as 1 wei.

## Vulnerability Detail

In the `ZivoeRewards` and `ZivoeRewardsVesting` contracts, rewards are added via the `depositReward()` function and are then distributed to all stakers over a set duration. The distribution rate, or `rewardRate`, is recalculated every time rewards are deposited according to this code:

```solidity
// Update vesting accounting for reward (if existing rewards being distributed, increase proportionally).
if (block.timestamp >= rewardData[_rewardsToken].periodFinish) {
	rewardData[_rewardsToken].rewardRate = reward.div(rewardData[_rewardsToken].rewardsDuration);
} else {
	uint256 remaining = rewardData[_rewardsToken].periodFinish.sub(block.timestamp);
	uint256 leftover = remaining.mul(rewardData[_rewardsToken].rewardRate);
	rewardData[_rewardsToken].rewardRate = reward.add(leftover).div(rewardData[_rewardsToken].rewardsDuration);
}
```

The code shows that if any rewards of `_rewardsToken` are currently being distributed, the `rewardRate` is calculated by interpolating the current `rewardRate` and the reward rate of the new reward being deposited.

We should also note that `depositReward()` has no access control or minimum deposit requirements, which allows anyone to deposit even 1 wei.

Here is how this can be exploited:

**Example scenario:**

- Suppose USDC is configured as a reward token with `rewardsDuration` of 30 days.
- An initial legitimate reward of 100,000 USDC sets the `rewardRate` to:

$$
rewardRate = \frac{100,000e6}{30\ days} = \frac{100,000e6}{2,592,000\ sec.} = 38,580
$$

- 15 days later, a malicious actor deposists just 1 wei of USDC. The new `rewardRate` becomes:

$$
rewardRate = \frac{1\ wei + (15\ days \times 38,580)}{30\ days} = \frac{1 + 49,999,680,000}{2,592,000 sec.} = 19,291
$$

This recalculation effectively halves the speed of the reward distribution. This results in the distribution of only 25,001 USDC over the next days instead of the expected 50,000 USDC:

$$
distributedTokens = 15\ days \times 19,291 = 25,001e6\ USDC 
$$

With a configured `rewardsDuration` of 30 days, a malicious actor can reduce the speed of reward distribution by up to 36% by depositing just 1 wei of rewards daily. The following coded PoC will demonstrate 
this exploit.
### Coded PoC

Add this test to `Test_ZivoeRewards.sol` and add the following import: `import "forge-std/console2.sol";`

Run the PoC with `forge test --mt testRewardsDistributionCanBeSlowedDownByAnyone --fork-url $MAINNET_RPC_URL -vvv`

```solidity
    function testRewardsDistributionCanBeSlowedDownByAnyone() public {
        uint256 deposit = 100_000 ether;

        // Stake
        assert(sam.try_approveToken(address(ZVE), address(stZVE), IERC20(address(ZVE)).balanceOf(address(sam))));
        sam.try_stake(address(stZVE), IERC20(address(ZVE)).balanceOf(address(sam)));

        // Deposit rewards (rewardsDuration = 30 days for DAI)
        depositReward_DAI(address(stZVE), deposit);

        // Malicious actor deposits 1 wei every day
        for (uint i; i < 30; i++) {
            hevm.warp(block.timestamp + 1 days);
            depositReward_DAI(address(stZVE), 1 wei);
        }

        // Get rewards
        assert(sam.try_getRewards(address(stZVE)));

        uint256 expected = deposit;
        uint256 actual = IERC20(DAI).balanceOf(address(sam));

        console2.log("Distributed rewards after 30 days");
        console2.log("=================================");
        console2.log("Actual distributed rewards  : %s", actual);
        console2.log("Expected distributed rewards: %s", expected);
    }
```

## Impact

This vulnerability allows a malicious actor to significantly slow down the distribution of rewards by depositing a trivial amount of tokens, potentially disrupting the intended payout schedule. This can lead to stakers receiving their earned rewards much later than expected.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewards.sol#L235-L237

## Tool used

Manual Review

## Recommendation

Depending on the project's needs, different solutions can help to solve this vulnerability. Some possible options:

- Introduce a minimum threshold for deposit amounts to prevent negligible amounts from impacting the reward distribution process.
- Restrict the `depositRewards()` function so that only authorized users or contracts can add rewards.
- Adopt a weighted distribution algorithm that allows reward rate calculation to be proportional to the size of the deposit. This means that smaller deposits will have less impact on the overall reward rate, ensuring that significant changes to the distribution pace require correspondingly large deposits.

# [M-1] `ZivoeRewards` Rewards are irretrievably lost if there are no stakers

## Summary

Rewards are irretrievably lost in the `ZivoeRewards` contract if there are no active stakers, with no current method to recover these funds.

## Vulnerability Detail

The `ZivoeRewards` contract is designed to distribute rewards to stakers continuously. However, if the staker count drops to zero, the system continues to stream rewards, which then become trapped within the contract. This occurs because there is no mechanism in place to halt reward distribution or recover funds when there are no participants to claim them.

### Coded PoC

Add this test to `Test_ZivoeRewards.sol` and add the following import: `import "forge-std/console2.sol";`

Run the PoC with `forge test --mt testRewardsGetIrretrievablyLostIfThereAreNoStakers --fork-url $MAINNET_RPC_URL -vvv`

```solidity
    function testRewardsGetIrretrievablyLostIfThereAreNoStakers() public {
        uint256 deposit = 100_000 ether;

        // Deposit rewards (rewardsDuration = 30 days for DAI)
        // NOTE: There are no stakers at this point
        depositReward_DAI(address(stZVE), deposit);

        // 15 days pass
        vm.warp(block.timestamp + 15 days);

        // Sam stakes
        assert(sam.try_approveToken(address(ZVE), address(stZVE), 1000e18));
        sam.try_stake(address(stZVE), 1000e18);

        // Another 30 days pass.
        // At this point the reward distribution have already finished 15 days ago
        vm.warp(block.timestamp + 30 days);

        // Sam claims his awards
        assert(sam.try_getRewards(address(stZVE)));

        // Sam (rightlfully) gets 50% of the rewards since he staked for 15 out of 30 days
        assertApproxEqRel(50_000 ether, IERC20(DAI).balanceOf(address(sam)), 0.01e18);
        // The remaining 50% of the rewards get stuck in the contract with no option to be receovered
        assertApproxEqRel(50_000 ether, IERC20(DAI).balanceOf(address(stZVE)), 0.01e18);

        // Another reward gets deposited, at this point there is 1 staker (sam)
        depositReward_DAI(address(stZVE), deposit);

        // Another 30 days pass
        vm.warp(block.timestamp + 30 days);

        // Sam claims his awards
        assert(sam.try_getRewards(address(stZVE)));
        // Sam has 50K (from the 1st awards distribution) + 100K (from the 2nd awards distributoin)
        assertApproxEqRel(150_000 ether, IERC20(DAI).balanceOf(address(sam)), 0.01e18);
        // Proof that the undistributed rewards from the first reward distribution remain stuck in stZVE
        assertApproxEqRel(50_000 ether, IERC20(DAI).balanceOf(address(stZVE)), 0.01e18);
    }
```

## Impact

This issue results in a direct loss of funds for the protocol in the described scenario. While the likelihood of this issue occurring is low, its impact is significant due to the irreversible loss of funds.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewards.sol#L26

## Tool used

Manual Review

## Recommendation

To mitigate this risk, it is recommended to implement a safety mechanism within the `ZivoeRewards` contract that allows the recovery of funds by `ZivoeDAO` when there are no active stakers. This could be achieved through:

- **Pause Functionality:** Introduce logic to pause reward distribution if the staker count is zero (`_totalSupply == 0`).
- **Recovery Mechanism:** Provide a method for `ZivoeDAO` to reclaim unclaimed rewards, ensuring these funds can be reallocated or returned to the treasury.
