Huge Foggy Weasel

medium

# Reward distribution assumes funds are dispersed even if there are no recipients

## Summary

The `VirtualStakingRewards`'s accounting logic does not properly handle cases where time passes without any staked users.


## Vulnerability Detail

The contract keep track of the total time passed, as well as the cumulative rate of rewards that have been generated, so that for every user, it just has to keep track of the last timestamp and last rate at the time of the last withdrawal, in order to calculate the rewards owed since the user began staking. The code special-cases the scenario where there are no users, by not updating the cumulative rate when the `_totalSupply` is zero, but it does not include such a condition for the tracking of the timestamp. Because of this, even when there are no users staking, the accounting logic still thinks funds were being dispersed during that timeframe (because the starting timestamp is updated), which means the funds effectively are distributed to nobody, rather than being saved for when there is someone to receive them.


## Impact

If funds are added and `notifyRewardAmount()` is called prior to there being any users staking (as is done in the [deployment script](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/script/01_deployTrufAndVesting.sol#L35-L37)), the funds that should have gone to the first stakers will instead accrue to nobody.

The funds are not locked in the contract forever though, because the admin can call `notifyRewardAmount()` without transferring any extra funds, which will use the latent balance as the reward source. However, the set of people and their durations will not be the same as the set in existence when the first notification happened.


In this POC, Alice doesn't stake until three days after the first reward notification, and therefore the first three days of rewards get stuck in the contract. After the [default](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/staking/VirtualStakingRewards.sol#L27) 7-day duration, the admin calls `notifyRewardAmount()` again, and now the full amount gets sent:
```diff
diff --git a/truflation-contracts/test/staking/VirtualStakingRewards.t.sol b/truflation-contracts/test/staking/VirtualStakingRewards.t.sol
index b2bb197..14bf9de 100644
--- a/truflation-contracts/test/staking/VirtualStakingRewards.t.sol
+++ b/truflation-contracts/test/staking/VirtualStakingRewards.t.sol
@@ -182,6 +182,45 @@ contract VirtualStakingRewardsTest is Test {
         );
     }
 
+    function testNotifyNoStakers() external {
+        console.log("Test notify with no stakers");
+
+        uint256 rewardAmount = 100_000e18;
+        uint256 aliceAmount = rewardAmount;
+
+        _notifyReward(rewardAmount);
+        vm.warp(block.timestamp + 3 days);
+        _stake(alice, aliceAmount);
+        vm.warp(block.timestamp + 4 days + 1);
+
+        assertEq(
+            trufStakingRewards.earned(alice),
+            rewardAmount,
+            "Earned amount is invalid"
+        );
+
+        // admin rescues remaining funds
+        uint256 remaining = trufToken.balanceOf(address(trufStakingRewards)) -
+            trufStakingRewards.earned(alice);
+        vm.prank(rewardsDistribuion);
+        trufStakingRewards.notifyRewardAmount(remaining);
+        vm.warp(block.timestamp + 7 days);
+        assertEq(
+            trufStakingRewards.earned(alice),
+            rewardAmount,
+            "Earned amount is invalid"
+        );
+        /**
+         [FAIL. Reason: assertion failed] testNotifyNoStakers() (gas: 248615)
+            ...
+            Left: 57142857142857142500000
+           Right: 100000000000000000000000
+            ...
+            Left: 99999999999999999400000
+           Right: 100000000000000000000000
+         */
+    }
+
     function testGetRewardForDuration() external {
         console.log("Get reward for duration");
 
```

In reality, Alice would have to re-stake after 7 days.


## Code Snippet

When `notifyRewardAmount()` is called, the full duration of the emission is set, even if there are currently no stakers:
```solidity
// File: src/staking/VirtualStakingRewards.sol : VirtualStakingRewards.notifyRewardAmount()   #1

162 @>         lastUpdateTime = block.timestamp;
163:@>         periodFinish = block.timestamp + rewardsDuration;
```
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/staking/VirtualStakingRewards.sol#L162-L163


## Tool used

Manual Review


## Recommendation

As you can see in the POC's output, there is a rounding issue leading to not all rewards being dispersed (dust amounts). Rather than trying to fix the complicated distribution logic, allow the admin to withdraw any excess funds in the contract, so that they can manually distribute them to the right participants.

Another option is to make `notifyRewardAmount()` revert if there are currently no stakers.