Quiet Ceramic Oyster

medium

# Insolvency Risk: `VirtualStakingRewards` may not possess enough `rewardTokens` to satisfy obligations.

## Summary

[`VirtualStakingRewards`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol) allows an established [`operator`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L22) to define the [`reward`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L144) for the next cycle via a call to [`notifyRewardAmount(uint256)`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L144), however it does not verify that the contract has sufficient balance to pay these rewards.

## Vulnerability Detail

When making a call to [`notifyRewardAmount(uint256)`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L144), care is taken to ensure that the contract's current reward token balance is capable of making users whole:

```solidity
// Ensure the provided reward amount is not more than the balance in the contract.
// This keeps the reward rate in the right range, preventing overflows due to
// very high values of rewardRate in the earned and rewardsPerToken functions;
// Reward + leftover must be less than 2^256 / 10^18 to avoid overflow.
uint256 balance = IERC20(rewardsToken).balanceOf(address(this));
if (rewardRate > balance / rewardsDuration) {
    revert InsufficientRewards();
}
```

However, this does not take into account current reward token obligations.

When extending a reward period, any unclaimed reward token balance in the contract will be mistakenly interpreted as reward token collateral that can be allocated for a new round of issuance.

Below, we show that an actor `ALICE` has `1 ether` staked for two `7 days` [`rewardDuration`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L27), each valued at `1 ether` in [`rewardsToken`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L24). Since `ALICE` doesn't redeem their tokens, [`VirtualStakingRewards`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol) assumes it possesses sufficient [`rewardsToken`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L24) capital for the second round:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.19;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";

import {VirtualStakingRewards} from "../../src/staking/VirtualStakingRewards.sol";
import {TruflationToken} from "../../src/token/TruflationToken.sol";

contract VirtualStakingRewardsSherlock is Test {

    // Warps the vm to a realistic timestamp.
    function _warp() internal {
        vm.roll(18936275);
        vm.warp(1704399287);
    }

    // Create a simple VirtualStakingRewards configuration with
    // a single `admin` who acts as the owner, operator and
    // distributor.
    function _initializeVirtualStaking() internal returns (
        VirtualStakingRewards virtualStakingRewards,
        TruflationToken truflationToken,
        address admin
    ) {
        admin = address(0x01);
        vm.startPrank(admin);
            truflationToken = new TruflationToken();
            virtualStakingRewards = new VirtualStakingRewards(admin, address(truflationToken));
            virtualStakingRewards.setOperator(admin);
        vm.stopPrank();
    }

    function test_sherlock_Insolvency() public {

        _warp() /* realistic_timestamp */;

        (VirtualStakingRewards staking, TruflationToken token, address operator) = _initializeVirtualStaking() /* deployment */;

        // Configure the reward for this period.
        vm.startPrank(operator);
            token.transfer(address(staking), 1 ether);
            staking.notifyRewardAmount(1 ether);
        vm.stopPrank();

        address ALICE = address(0x420);

        vm.prank(operator);
          staking.stake(ALICE, 1 ether);

        vm.warp(block.timestamp + 7 days);

        vm.prank(operator);
          staking.notifyRewardAmount(1 ether);

        vm.warp(block.timestamp + 7 days);

        assertEq(staking.earned(ALICE), 1999999999999814400);

        vm.prank(operator);
@>          vm.expectRevert("ERC20: transfer amount exceeds balance");
            staking.exit(ALICE);
    }

}
```

As demonstrated, attempts to withdraw `ALICE`'s reward `revert` with `"ERC20: transfer amount exceeds balance"`.

## Impact

- The staking contract can become insolvent in reward token balance, leaving some users unable to claim their rewards.
  - In order to sustain any liveliness of reward redemption in an insolvent contract, reward balances of honest users may need to be artificially reduced via a call to [`withdraw`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L117), else more reward tokens may need to be manually minted to fulfill existing obligations.
  - If [`VirtualStakingRewards`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol) is not owned by an EOA, is renounced, or is owned by a smart contract lacking required functionality, manually minting new tokens to satisfy demand may not be possible.
- Users who leave a reward cycle early in an insolvent configuration may receive amplified rewards compared to those who continue to stake.
- DoS when claiming rewards.

## Code Snippet

```solidity
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

## Tool used

Visual Studio Code, Foundry

## Recommendation

When calling [`notifyRewardAmount(uint256)`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L144), verify the amount of [`rewardsToken`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol#L24) in [`VirtualStakingRewards`](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/staking/VirtualStakingRewards.sol) can fulfil both the new reward amount and the existing reward obligations.
