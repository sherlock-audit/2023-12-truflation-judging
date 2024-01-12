Loud Cotton Mallard

medium

# Possible Rounding Errors

## Summary

The `notifyRewardAmount` function at [here](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L144) may lead to rounding errors to 

## Vulnerability Detail

The `notifyRewardAmount` calculates the `rewardRate` that is used in other functions to calculated the `earned` amount and other computations. The   problem arises when the provided `reward` (parameter) is used to calculate the `rewardRate` by diving it to the `rewardsDuration`, in a situation where the `reward` is lower than the `rewardsDuration`, the value will round down  to zero(0) potentially causing a non-intended  calculations of functions like `rewardPerToken` and so, this is unlikely to happen, but as Auditors , our primary goal is to ensure we tackle any possible issues that will lead to user loosing funds and the protocol itself, taking account of the fact that the `rewardsDuration` can be adjusted to a desirable value via the `setRewardsDuration` function, the possibility of occurrence  is to be considered 

## Impact
The issue can lead to wrong accounting as the value will round down when the one with privilege role put the  `reward` parameter that is < the `rewardsDuration`   either intentional or by mistake 

## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L146

## Tool used

Manual Review

## Recommendation

Add a `require` check to ensure that the code reverts whenever the value = 0 (round down), this will be something like :

```solidity
function notifyRewardAmount(uint256 reward) external onlyRewardsDistribution updateReward(address(0)) {
    if (block.timestamp >= periodFinish) {
        // @audit-info periodFinish ->  reward period.
        rewardRate = reward / rewardsDuration; // @audit rounding errors?
    } else {
        uint256 remaining = periodFinish - block.timestamp;
        uint256 leftover = remaining * rewardRate;
        rewardRate = (reward + leftover) / rewardsDuration;
    }

    // Ensure the provided reward amount is not more than the balance in the contract.
    // This keeps the reward rate in the right range, preventing overflows due to
    // very high values of rewardRate in the earned and rewardsPerToken functions;
    // Reward + leftover must be less than 2^256 / 10^18 to avoid overflow.
    uint256 balance = IERC20(rewardsToken).balanceOf(address(this));
    
    // Require that the rewardRate is non-zero to avoid rounding error.
+    require(rewardRate > 0, "Reward rate must be greater than zero");

    if (rewardRate > balance / rewardsDuration) {
        revert InsufficientRewards();
    }

    lastUpdateTime = block.timestamp;
    periodFinish = block.timestamp + rewardsDuration;
    emit RewardAdded(reward);
}
``` 

`or`

```solidity
  require(rewardRate != 0, "Reward rate must not be zero");
```
