Energetic Canvas Bat

high

# Staking Reward amount calculation is wrong it is always same doesn't matter how many tokens have been staked

## Summary

Staking Reward amount will always be same for different stake amounts like if I stake `1e18`, `100e18`,`10000e18` for `30 days` reward amount will be `reward: 39999999999999916799 [3.999e19])` for all amount.

## Vulnerability Detail

Wrong reward amount calculation

considering Test code

```solidity
      function testClaimedRewardsRemainsSameForDifferentInputs_Amount() external {
        uint256 amount = 1e18; ----- change amount here -----
        uint256 duration = 1 days; ----- keep it same -----
        _stake(amount, duration, alice, bob);
        vm.warp(block.timestamp + 1 days);
        vm.startPrank(bob);
        //vm.warp(block.timestamp + 1 days);
        veTRUF.claimReward();
        vm.stopPrank();
    }
```

Execute above code for below conditions :

`amount = 1e18`
`duration = 1 days`

Stake Event
`emit Stake(user: Bob: [0xDba2C8c2363677bA0514EF616592D81557e679B6], isVesting: false, lockupId: 0, amount: 1000000000000000000 [1e18], end: 1696903130 [1.696e9], points: 913242009132420 [9.132e14])`
RewardClaimed
`emit RewardPaid(user: Bob: [0xDba2C8c2363677bA0514EF616592D81557e679B6], reward: 39999999999999916799 [3.999e19])`

```log
 [395] VirtualStakingRewards::lastUpdateTime() [staticcall]
    │   └─ ← 1696816730 [1.696e9]
    ├─ [3351] VirtualStakingRewards::rewardPerToken() [staticcall]
    │   └─ ← 43799999999999913275999 [4.379e22]
    ├─ [433] VirtualStakingRewards::lastTimeRewardApplicable() [staticcall]
    │   └─ ← 1696903130 [1.696e9]
    ├─ [2353] VirtualStakingRewards::earned(Bob: [0xDba2C8c2363677bA0514EF616592D81557e679B6]) [staticcall]
    │   └─ ← 39999999999999916799 [3.999e19]
    ├─ [554] VirtualStakingRewards::rewards(Bob: [0xDba2C8c2363677bA0514EF616592D81557e679B6]) [staticcall]
    │   └─ ← 0
    ├─ [554] VirtualStakingRewards::userRewardPerTokenPaid(Bob: [0xDba2C8c2363677bA0514EF616592D81557e679B6]) [staticcall]
    │   └─ ← 0
    ├─ [571] VirtualStakingRewards::balanceOf(Bob: [0xDba2C8c2363677bA0514EF616592D81557e679B6]) [staticcall]
    │   └─ ← 913242009132420 [9.132e14]
    ├─ [349] VirtualStakingRewards::totalSupply() [staticcall]
    │   └─ ← 913242009132420 [9.132e14]
```

`amount = 100e18`
`duration = 1 days`

Stake Event
`emit Stake(user: Bob: [0xDba2C8c2363677bA0514EF616592D81557e679B6], isVesting: false, lockupId: 0, amount: 100000000000000000000 [1e20], end: 1696903130 [1.696e9], points: 91324200913242009 [9.132e16]`
RewardClaimed
`emit RewardPaid(user: Bob: [0xDba2C8c2363677bA0514EF616592D81557e679B6], reward: 39999999999999916799 [3.999e19])`

```log
[328] VirtualStakingRewards::rewardPerTokenStored() [staticcall]
    │   └─ ← 0
    ├─ [395] VirtualStakingRewards::lastUpdateTime() [staticcall]
    │   └─ ← 1696816730 [1.696e9]
    ├─ [3351] VirtualStakingRewards::rewardPerToken() [staticcall]
    │   └─ ← 437999999999999089595 [4.379e20]
    ├─ [433] VirtualStakingRewards::lastTimeRewardApplicable() [staticcall]
    │   └─ ← 1696903130 [1.696e9]
    ├─ [2353] VirtualStakingRewards::earned(Bob: [0xDba2C8c2363677bA0514EF616592D81557e679B6]) [staticcall]
    │   └─ ← 39999999999999916799 [3.999e19]
    ├─ [554] VirtualStakingRewards::rewards(Bob: [0xDba2C8c2363677bA0514EF616592D81557e679B6]) [staticcall]
    │   └─ ← 0
    ├─ [554] VirtualStakingRewards::userRewardPerTokenPaid(Bob: [0xDba2C8c2363677bA0514EF616592D81557e679B6]) [staticcall]
    │   └─ ← 0
    ├─ [571] VirtualStakingRewards::balanceOf(Bob: [0xDba2C8c2363677bA0514EF616592D81557e679B6]) [staticcall]
    │   └─ ← 91324200913242009 [9.132e16]
    ├─ [349] VirtualStakingRewards::totalSupply() [staticcall]
    │   └─ ← 91324200913242009 [9.132e16]
```

Reason :
` (_balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account]))` is giving same value for both amounts `39989028000000000000000000000000000000` divide by 1e18 `39989028000000000000` [3.999e19], where `userRewardPerTokenPaid[account]` is zero in both cases

## Impact

Reward is same for different amiunts for same duration, if it is giving same reward for `1e18 token` and `100e18 tokens` why would user stake more if user is getting same rewards for all amounts.

## Code Snippet

```solidity
function earned(address account) public view returns (uint256) {
        return
            (_balances[account] *
                (rewardPerToken() - userRewardPerTokenPaid[account])) /
            1e18 +
            rewards[account];
    }
```

https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L95C1-L97C6

## Tool used

Manual Review

## Recommendation

` (_balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account]))` is giving same value. `earned(address account)` needs to be updated so that such issues doe not occur