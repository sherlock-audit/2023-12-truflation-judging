Huge Foggy Weasel

medium

# First staker gets full rewards with only one wei

## Summary

Rewards are paid based on the percentage total supply of the veTRUF one holds, rather than of the percentage of the total supply of the TRUF token one holds, which allows one to get full rewards with at little as one wei.


## Vulnerability Detail

If someone is the only one to stake, and they have staked only one wei, they'll be given the full rewards distributable for that period, which will be significantly more than one wei. This sort of setup works fine when anyone is able to get tokens for staking, but in the TRUF case, most TRUF tokens are sequestered in the vesting contract, and presumably the intention is to not allow staking until at least each user's start time has been reached.

It's possible that a single user figures out the other bug I submitted (`Users can immediately stake their full vesting amounts`) and ends up claiming all rewards using only one wei.

If everyone withdraws their stakes, the one-wei-scenario becomes possible again.


## Impact

Rewards will be wasted without actually locking useful amounts of the TRUF token. The purpose of voting escrows is to incentivize people to lock up as many of their coins for as long as possible, so that they can't sell and push down the price of the token. If someone is able to get all of the available incentives while only locking up one wei, they're still free to to sell all of their other coins, tanking the price when the tokens are more liquid and appear on a DEX/CEX. If the votes associated with the veTRUF token control some pool of funds, after getting all of the rewards, the attacker can stake the rewarded tokens, and get their voting power to control the pool of funds.

In this POC, Alice gets all of the rewards by staking only one wei:
```diff
diff --git a/truflation-contracts/test/staking/VirtualStakingRewards.t.sol b/truflation-contracts/test/staking/VirtualStakingRewards.t.sol
index b2bb197..32cefa9 100644
--- a/truflation-contracts/test/staking/VirtualStakingRewards.t.sol
+++ b/truflation-contracts/test/staking/VirtualStakingRewards.t.sol
@@ -182,6 +182,29 @@ contract VirtualStakingRewardsTest is Test {
         );
     }
 
+    function testOneWei() external {
+        console.log("Test that you can only earn what you put in");
+
+        uint256 rewardAmount = 100_000e18;
+        uint256 aliceAmount = 1;
+
+        _stake(alice, aliceAmount);
+        _notifyReward(rewardAmount);
+        vm.warp(block.timestamp + 8 days);
+
+        assertEq(
+            trufStakingRewards.earned(alice),
+            aliceAmount,
+            "Earned amount is invalid"
+        );
+        /* Output:
+        [FAIL. Reason: assertion failed] testOneWei() (gas: 205040)
+        ...
+                Left: 99999999999999999446400
+               Right: 1
+        */
+    }
+
     function testGetRewardForDuration() external {
         console.log("Get reward for duration");
 
```


## Code Snippet

If you ignore the time-related parts of the highlighted lines, the equation boils down to `earned = rewardRate * (_balances[account] / _totalSupply)`:
```solidity
// File: src/staking/VirtualStakingRewards.sol : VirtualStakingRewards.earned()   #1

87        function rewardPerToken() public view returns (uint256) {
88            if (_totalSupply == 0) {
89                return rewardPerTokenStored;
90            }
91            return
92 @>             rewardPerTokenStored + (((lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18) / _totalSupply);
93        }
94    
95        function earned(address account) public view returns (uint256) {
96 @>         return (_balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account])) / 1e18 + rewards[account];
97:       }
```
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/staking/VirtualStakingRewards.sol#L87-L97

where `_totalSupply` and `_balances[account]` refer to the veTRUF token, and `rewardRate` is the total reward amount [divided by](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/staking/VirtualStakingRewards.sol#L146) the duration of the reward. When `_balances[account] == _totalSupply`, the user gets the full `rewardRate` per second.


## Tool used

Manual Review


## Recommendation

Cap the total amount earnable to `_balances[account]` (this is not a simple change to implement)

