Modern Linen Fly

high

# `VirtualStakingRewards::notifyRewardAmount` could reduce users rewards

## Summary
Adjusting a user's rewards not working correctly 

## Vulnerability Detail
Taking down `VirtualStakingRewards::rewardRate` without updating `VirtualStakingRewards::rewards`, forces user who used to stake some tokens to lose rewards.

Scenario:
1. User stake funds with `VotingEscrowTruf::stake`
2. Operator takes down `VirtualStakingRewards::rewardRate` by calling 'notifyRewardAmount'
3. User calls `VotingEscrowTruf::claimRewards` after the change and get less tokens

At first random user is about to stake some funds by calling `VotingEscrowTruf::stake`:
```javascript
function stake(uint256 amount, uint256 duration) external returns (uint256 lockupId) {
        lockupId = _stake(amount, duration, msg.sender, false);
    }
```

Then after some time protocol owners decides to take reward down by calling `VirtualStakingRewards::notifyRewardAmount`, which is going to take `VirtualStakingRewards::rewardRate` down:
```javascript
function notifyRewardAmount(uint256 reward) external onlyRewardsDistribution updateReward(address(0)) {
        if (block.timestamp >= periodFinish) {
            rewardRate = reward / rewardsDuration;
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
        if (rewardRate > balance / rewardsDuration) {
            revert InsufficientRewards();
        }

        lastUpdateTime = block.timestamp;
        periodFinish = block.timestamp + rewardsDuration;
        emit RewardAdded(reward);
    }
```

Then user calls `VotingEscrowTruf::claimReward` which calls `VirtualStakingRewards::getReward` and takes less token than it has to get:

```javascript
function claimReward() external {
        stakingRewards.getReward(msg.sender);
    }

function getReward(address user) public updateReward(user) returns (uint256 reward) {
        reward = rewards[user];
        if (reward != 0) {
            rewards[user] = 0;
            IERC20(rewardsToken).safeTransfer(user, reward);
            emit RewardPaid(user, reward);
        }
    }
```
## Impact
Loss of funds for the user

## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/staking/VirtualStakingRewards.sol#L144-L151

## Tool used

Manual Review

## Recommendation
Consider using `VirtualStakingRewards::_updateRewardForAll` function or modifier before changing 'rewardRate'

## Proof of Concept

Put the code below in test folder and run both tests to see the difference in user's balance:

```javascript
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.19;

import {Test, console} from "forge-std/Test.sol";
import {VirtualStakingRewards} from "../src/staking/VirtualStakingRewards.sol";
import {TruflationToken} from "../src/token/TruflationToken.sol";
import {VotingEscrowTruf} from "../src/token/VotingEscrowTruf.sol";
import {TrufVesting} from "../src/token/TrufVesting.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract VulnTest is Test {
    VirtualStakingRewards stakingRewards;
    TruflationToken tfi;
    TrufVesting trufVesting;
    VotingEscrowTruf votingEscrowTruf;

    address rewardsDistribution = makeAddr("distributor");
    address victim = makeAddr("victim");

    uint256 MIN_STAKE_DURATION = 1 days;

    function setUp() public {
        tfi = new TruflationToken();
        stakingRewards = new VirtualStakingRewards(
            rewardsDistribution,
            address(tfi)
        );
        trufVesting = new TrufVesting(IERC20(address(tfi)), 7 days);
        votingEscrowTruf = new VotingEscrowTruf(
            address(tfi),
            address(trufVesting),
            MIN_STAKE_DURATION,
            address(stakingRewards)
        );
        stakingRewards.setOperator(address(votingEscrowTruf));

        deal(address(tfi), victim, 10e18);
        deal(address(tfi), address(stakingRewards), 100e18);
    }

    function testStakerUsualReward() public {
        console.log("User's assets before stake: ", tfi.balanceOf(victim));
        vm.prank(rewardsDistribution);
        stakingRewards.notifyRewardAmount(2e18);

        skip(1 days);

        uint256 lockupId;
        vm.startPrank(victim);
        tfi.approve(address(votingEscrowTruf), 10e18);
        lockupId = votingEscrowTruf.stake(10e18, MIN_STAKE_DURATION);
        vm.stopPrank();

        console.log("User's assets after stake: ", tfi.balanceOf(victim));

        vm.prank(rewardsDistribution);
        stakingRewards.notifyRewardAmount(2e18);

        skip(2 days);

        vm.startPrank(victim);
        votingEscrowTruf.unstake(lockupId);
        votingEscrowTruf.claimReward();
        console.log(
            "User's assets after unstake and getting rewards: ",
            tfi.balanceOf(victim)
        );
        vm.stopPrank();
    }

    function testStakerLosingReward() public {
        console.log("User's assets before stake: ", tfi.balanceOf(victim));
        vm.prank(rewardsDistribution);
        stakingRewards.notifyRewardAmount(2e18);

        skip(1 days);

        uint256 lockupId;
        vm.startPrank(victim);
        tfi.approve(address(votingEscrowTruf), 10e18);
        console.log(tfi.balanceOf(victim));
        lockupId = votingEscrowTruf.stake(10e18, MIN_STAKE_DURATION);
        vm.stopPrank();

        console.log("User's assets after stake: ", tfi.balanceOf(victim));

        vm.prank(rewardsDistribution);
        stakingRewards.notifyRewardAmount(1e18);

        skip(2 days);

        vm.startPrank(victim);
        votingEscrowTruf.unstake(lockupId);
        votingEscrowTruf.claimReward();
        console.log(
            "User's assets after unstake and getting rewards: ",
            tfi.balanceOf(victim)
        );
        vm.stopPrank();
    }
}
```