# Issue H-1: Users can fully drain the `TrufVesting` contract 

Source: https://github.com/sherlock-audit/2023-12-truflation-judging/issues/1 

## Found by 
0xLogos, CL001, HonorLt, IvanFitro, KupiaSec, asauditor, bughuntoor, carrotsmuggler, cawfree, fnanni, mstpr-brainbot, s1ce, ubl4nk, unforgiven, ydlee, zzykxx
## Summary
Due to flaw in the logic in `claimable` any arbitrary user can drain all the funds within the contract.

## Vulnerability Detail
A user's claimable is calculated in the following way: 
1. Up until start time it is 0.
2. Between start time and cliff time it's equal to `initialRelease`.
3. After cliff time, it linearly increases until the full period ends.

However, if we look at the code, when we are at stage 2., it always returns `initialRelease`, even if we've already claimed it. This would allow for any arbitrary user to call claim as many times as they wish and every time they'd receive `initialRelease`. Given enough iterations, any user can drain the contract. 

```solidity
    function claimable(uint256 categoryId, uint256 vestingId, address user)
        public
        view
        returns (uint256 claimableAmount)
    {
        UserVesting memory userVesting = userVestings[categoryId][vestingId][user];

        VestingInfo memory info = vestingInfos[categoryId][vestingId];

        uint64 startTime = userVesting.startTime + info.initialReleasePeriod;

        if (startTime > block.timestamp) {
            return 0;
        }

        uint256 totalAmount = userVesting.amount;

        uint256 initialRelease = (totalAmount * info.initialReleasePct) / DENOMINATOR;

        startTime += info.cliff;

        if (startTime > block.timestamp) {
            return initialRelease;
        }
```

```solidity
    function claim(address user, uint256 categoryId, uint256 vestingId, uint256 claimAmount) public {
        if (user != msg.sender && (!categories[categoryId].adminClaimable || msg.sender != owner())) {
            revert Forbidden(msg.sender);
        }

        uint256 claimableAmount = claimable(categoryId, vestingId, user);
        if (claimAmount == type(uint256).max) {
            claimAmount = claimableAmount;
        } else if (claimAmount > claimableAmount) {
            revert ClaimAmountExceed();
        }
        if (claimAmount == 0) {
            revert ZeroAmount();
        }

        categories[categoryId].totalClaimed += claimAmount;
        userVestings[categoryId][vestingId][user].claimed += claimAmount;
        trufToken.safeTransfer(user, claimAmount);

        emit Claimed(categoryId, vestingId, user, claimAmount);
    }
```


## Impact
Any user can drain the contract 

## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L176C1-L182C10

## Tool used

Manual Review

## Recommendation
change the if check to the following 
```solidity
        if (startTime > block.timestamp) {
            if (initialRelease > userVesting.claimed) {
            return initialRelease - userVesting.claimed;
            }
            else { return 0; } 
        }
```

## PoC 

```solidity
    function test_cliffVestingDrain() public { 
        _setupVestingPlan();
        uint256 categoryId = 2;
        uint256 vestingId = 0;
        uint256 stakeAmount = 10e18;
        uint256 duration = 30 days;

        vm.startPrank(owner);
        
        vesting.setUserVesting(categoryId, vestingId, alice, 0, stakeAmount);

        vm.warp(block.timestamp + 11 days);     // warping 11 days, because initial release period is 10 days
                                                // and cliff is at 20 days. We need to be in the middle 
        vm.startPrank(alice);
        assertEq(trufToken.balanceOf(alice), 0);
        vesting.claim(alice, categoryId, vestingId, type(uint256).max);
        
        uint256 balance = trufToken.balanceOf(alice);
        assertEq(balance, stakeAmount * 5 / 100);  // Alice should be able to have claimed just 5% of the vesting 

        for (uint i; i < 39; i++ ){ 
            vesting.claim(alice, categoryId, vestingId, type(uint256).max);
        }
        uint256 newBalance = trufToken.balanceOf(alice);   // Alice has claimed 2x the amount she was supposed to be vested. 
        assertEq(newBalance, stakeAmount * 2);             // In fact she can keep on doing this to drain the whole contract
    }
```




## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**Shaheen** commented:
>  Valid High. Solid Catch



**ryuheimat**

https://github.com/truflation/truflation-contracts/pull/3

Fixed `claimable` function

# Issue H-2: VotingEscrowTruf.sol#migrateVestingLock() 

Source: https://github.com/sherlock-audit/2023-12-truflation-judging/issues/14 

## Found by 
carrotsmuggler, cawfree, p-tsanev, rvierdiiev, s1ce, unforgiven
## Summary
The ``migrateVestingLock()`` function is used to transfer locks between users, but transferring the amounts and points, but it does not account the votes.

## Vulnerability Detail
When invoking ``migrateVestingLock()`` the old user's points get burned and the new user gets minted those points and receives a new lock. The old lock then gets deleted, but the function does not take into account the votes that the old user got delegated when he first created his lock.
As it can be seen in ``_stake``:
```solidity
        if (delegates(to) == address(0)) {
            // Delegate voting power to the receiver, if unregistered
            _delegate(to, to);
        }
```
Upon his first staking a user gets delegated voting power equal to the vote tokens he got minted. When migrating the lockup, a user's vote tokens get burned, but his delegated amount amount remains unchanged, because OZ's ``_delegate`` method saves a checkpoint of the vote amount. Unless a new delegate is invoked during migration, the old delegate would remain.
Another part of the issue is that the new user does not get delegated the voting power during migration, even though it might be the user's first lock, as per the logic of ``_stake`` above. Thus, a user who has no stake but receives a lock and an x amount of vote tokens could delegate himself even more tokens by then staking a y amount, thus the ``_delegate`` function would checkpoint x+y vote tokens.

## Impact
Improper voting power distribution and potential double-voting.

## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/VotingEscrowTruf.sol#L224-L252

## Tool used
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/VotingEscrowTruf.sol#L154-L172

Manual Review

## Recommendation
Properly handle the delegations during migration to fully rid the old user of any voting prowess he had.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**Shaheen** commented:
>  Valid medium. Good Finding



**IllIllI000**

Burning of the tokens removes the old delegation, so what's missing is solely the assignment of the new delegation

**ryuheimat**

https://github.com/truflation/truflation-contracts/pull/5

Fixed

# Issue H-3: When migrating the owner users will lose their rewards 

Source: https://github.com/sherlock-audit/2023-12-truflation-judging/issues/80 

## Found by 
0xmystery, Bauer, IllIllI, Krace, VAD37, ZdravkoHr., dany.armstrong90, mstpr-brainbot, s1ce
## Summary
When a user migrates the owner due to a lost private key, the rewards belonging to the previous owner remain recorded in their account and cannot be claimed, resulting in the loss of user rewards.

## Vulnerability Detail
According to the documentation, `migrateUser()` is used when a user loses their private key to migrate the old vesting owner to a new owner.
```solidity
    /**
     * @notice Migrate owner of vesting. Used when user lost his private key
     * @dev Only admin can migrate users vesting
     * @param categoryId Category id
     * @param vestingId Vesting id
     * @param prevUser previous user address
     * @param newUser new user address
     */

```

 In this function, the protocol calls `migrateVestingLock()` to obtain a new ID. 
```solidity
    if (lockupId != 0) {
            newLockupId = veTRUF.migrateVestingLock(prevUser, newUser, lockupId - 1) + 1;
            lockupIds[categoryId][vestingId][newUser] = newLockupId;
            delete lockupIds[categoryId][vestingId][prevUser];

            newVesting.locked = prevVesting.locked;
        }

```

However, in the `migrateVestingLock()` function, the protocol calls `stakingRewards.withdraw()` to withdraw the user's stake, burning points. In the `withdraw()` function, the protocol first calls `updateReward()` to update the user's rewards and records them in the user's account. 
```solidity
    function withdraw(address user, uint256 amount) public updateReward(user) onlyOperator {
        if (amount == 0) {
            revert ZeroAmount();
        }
        _totalSupply -= amount;
        _balances[user] -= amount;
        emit Withdrawn(user, amount);
    }
```

However, `stakingRewards.withdraw()` is called with the old owner as a parameter, meaning that the rewards will be updated on the old account. 
```solidity
  uint256 points = oldLockup.points;
        stakingRewards.withdraw(oldUser, points);
        _burn(oldUser, points);
```

As mentioned earlier, the old owner has lost their private key and cannot claim the rewards, resulting in the loss of these rewards.

## Impact
The user's rewards are lost

## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L329

## Tool used

Manual Review

## Recommendation
When migrating the owner, the rewards belonging to the previous owner should be transferred to the new owner.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**0xLogos** commented:
> loss of private key is extreme case, so i think better do not add additional complexity to the code



**ryuheimat**

https://github.com/truflation/truflation-contracts/pull/10

Added `to` param to `getReward` function, and call it when migrating.

# Issue H-4: Ended locks can be extended 

Source: https://github.com/sherlock-audit/2023-12-truflation-judging/issues/82 

## Found by 
Kow, aslanbek, bughuntoor, mstpr-brainbot, unforgiven, zzykxx
## Summary
When a lock period ends, it can be extended. If the new extension 'end' is earlier than the current block.timestamp, the user will have a lock that can be unstaked at any time."
## Vulnerability Detail
When the lock period ends, the owner of the expired lock can extend it to set a new lock end that is earlier than the current block.timestamp. By doing so, the lock owner can create a lock that is unstakeable at any time.

This is doable because there are no checks in the [extendLock](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/VotingEscrowTruf.sol#L333-L366) function that checks whether the lock is already ended or not. 

PoC:
```solidity
function test_ExtendLock_AlreadyEnded() external {
        uint256 amount = 100e18;
        uint256 duration = 5 days;

        _stake(amount, duration, alice, alice);

        // 5 days later, lock is ended for Alice
        skip(5 days + 1);

        (,, uint128 _ends,,) = veTRUF.lockups(alice, 0);

        // Alice's lock is indeed ended
        assertTrue(_ends < block.timestamp, "lock is ended");

        // 36 days passed  
        skip(36 days);

        // Alice extends her already finished lock 30 more days
        vm.prank(alice);
        veTRUF.extendLock(0, 30 days);

        (,,_ends,,) = veTRUF.lockups(alice, 0);

        // Alice's lock can be easily unlocked right away
        assertTrue(_ends < block.timestamp, "lock is ended");

        // Alice unstakes her lock, basically alice can unstake her lock anytime she likes
        vm.prank(alice);
        veTRUF.unstake(0);
    }
```
## Impact
The owner of the lock will achieve points that he can unlock anytime. This is clearly a gaming of the system and shouldn't be acceptable behaviour. A locker having a "lock" that can be unstaked anytime will be unfair for the other lockers. Considering this, I'll label this as high.
## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/VotingEscrowTruf.sol#L333-L366
## Tool used

Manual Review

## Recommendation
Do not let extension of locks that are already ended.



## Discussion

**ryuheimat**

https://github.com/truflation/truflation-contracts/pull/7
Disable to extend lock if already expired.

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**Shaheen** commented:
>  Valid issue but Medium. Good Finding



# Issue H-5: `cancelVesting` will potentially not give users unclaimed, vested funds, even if giveUnclaimed = true 

Source: https://github.com/sherlock-audit/2023-12-truflation-judging/issues/192 

## Found by 
s1ce, zraxx, zzykxx
## Summary

The purpose of `cancelVesting` is to cancel a vesting grant and potentially give users unclaimed but vested funds in the event that `giveUnclaimed = true`. However, due to a bug, in the event that the user had staked / locked funds, they will potentially not received the unclaimed / vested funds even if `giveUnclaimed = true`. 

## Vulnerability Detail

Here's the cancelVesting function in TrufVesting:

```solidity
function cancelVesting(uint256 categoryId, uint256 vestingId, address user, bool giveUnclaimed)
        external
        onlyOwner
{
        UserVesting memory userVesting = userVestings[categoryId][vestingId][user];

        if (userVesting.amount == 0) {
            revert UserVestingDoesNotExists(categoryId, vestingId, user);
        }

        if (userVesting.startTime + vestingInfos[categoryId][vestingId].period <= block.timestamp) {
            revert AlreadyVested(categoryId, vestingId, user);
        }

        uint256 lockupId = lockupIds[categoryId][vestingId][user];

        if (lockupId != 0) {
            veTRUF.unstakeVesting(user, lockupId - 1, true);
            delete lockupIds[categoryId][vestingId][user];
            userVesting.locked = 0;
        }

        VestingCategory storage category = categories[categoryId];

        uint256 claimableAmount = claimable(categoryId, vestingId, user);
        if (giveUnclaimed && claimableAmount != 0) {
            trufToken.safeTransfer(user, claimableAmount);

            userVesting.claimed += claimableAmount;
            category.totalClaimed += claimableAmount;
            emit Claimed(categoryId, vestingId, user, claimableAmount);
        }

        uint256 unvested = userVesting.amount - userVesting.claimed;

        delete userVestings[categoryId][vestingId][user];

        category.allocated -= unvested;

        emit CancelVesting(categoryId, vestingId, user, giveUnclaimed);
}
```

First, consider the following code:

```solidity
uint256 lockupId = lockupIds[categoryId][vestingId][user];

if (lockupId != 0) {
            veTRUF.unstakeVesting(user, lockupId - 1, true);
            delete lockupIds[categoryId][vestingId][user];
            userVesting.locked = 0;
}
```

First the locked / staked funds will essentially be un-staked. The following line of code: `userVesting.locked = 0;` exists because there is a call to `uint256 claimableAmount = claimable(categoryId, vestingId, user);` afterwards, and in the event that there were locked funds that were unstaked, these funds should now potentially be claimable if they are vested (but if locked is not set to 0, then the vested funds will potentially not be deemed claimable by the `claimable` function). 

However, because `userVesting` is `memory` rather than `storage`, this doesn't end up happening (so `userVesting.locked = 0;` is actually a bug). This means that if a user is currently staking all their funds (so all their funds are locked), and `cancelVesting` is called, then they will not receive any funds back even if `giveUnclaimed = true`. This is because the `claimable` function (which will access the unaltered `userVestings[categoryId][vestingId][user]`) will still think that all the funds are currently locked, even though they are not as they have been forcibly unstaked. 

## Impact

When `cancelVesting` is called, a user may not receive their unclaimed, vested funds. 

## Code Snippet

https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L348-L388

## Tool used

Manual Review

## Recommendation
Change `userVesting.locked = 0;` to `userVestings[categoryId][vestingId][user].locked = 0;`



## Discussion

**ryuheimat**

Fixed.
Updated `userVesting` type to storage type to fix https://github.com/sherlock-audit/2023-12-truflation-judging/issues/192
Considered initial release period and cliff for vesting end validation to fix https://github.com/sherlock-audit/2023-12-truflation-judging/issues/11

Extra:
Added `admin` permission, and allow several admins to be able to set vesting info and user vestings.
https://github.com/truflation/truflation-contracts/pull/4

# Issue M-1: Insolvency Risk: `VirtualStakingRewards` may not possess enough `rewardTokens` to satisfy obligations. 

Source: https://github.com/sherlock-audit/2023-12-truflation-judging/issues/4 

## Found by 
0xhashiman, Bauer, Ward, ZanyBonzy, bigbick123456789000, cawfree, dany.armstrong90, ddimitrov22, novaman33, radin200
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**Shaheen** commented:
>  Invalid. Owner mistake 



**ryuheimat**

This is duplication of https://github.com/sherlock-audit/2023-12-truflation-judging/issues/105

# Issue M-2: TrufVesting.cancelVesting calculates end of vesting incorrectly 

Source: https://github.com/sherlock-audit/2023-12-truflation-judging/issues/11 

## Found by 
UbiquitousComputing, fnanni, nobody2018, rvierdiiev
## Summary
TrufVesting.cancelVesting calculates end of vesting incorrectly and because of that owner can't stop vesting for the account.
## Vulnerability Detail
`TrufVesting.cancelVesting` function allows owner to stop vesting for some account.

In case if vesting already finished, [then function reverts](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L358-L360).

The problem is that `vestingInfos[categoryId][vestingId].period` is just [period for token distribution after cliff](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L186).

As you can see in `claimable` function the vesting starts on `startTime`, then when [`initialReleasePeriod` has passed](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L168), then `initialRelease` is distributed. Then [after `cliff` has passed](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L178), only then [final distribution starts](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L184).

Thus check if vesting already finished is incorrect in the `TrufVesting.cancelVesting` and it doesn't allow owner to cancel vesting when it is still going.
## Impact
Ongoing vesting can't be canceled.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
This check should be correct.
```solidity
if (userVesting.startTime + vestingInfos[categoryId][vestingId].initialReleasePeriod + vestingInfos[categoryId][vestingId].cliff + vestingInfos[categoryId][vestingId].period <= block.timestamp) {
            revert AlreadyVested(categoryId, vestingId, user);
}
```



## Discussion

**ryuheimat**

https://github.com/truflation/truflation-contracts/pull/4

# Issue M-3: Partial staking is not possible, updated vested users can't stake the new amount if they had previous stake 

Source: https://github.com/sherlock-audit/2023-12-truflation-judging/issues/39 

## Found by 
eeshenggoh, mstpr-brainbot, rvierdiiev
## Summary
When user stakes his vest, user must stakes the full amount to get the best outcome because partial staking is not allowed for vests. However, there are some scenarios where user can get more vests but not able to stake it in the voting escrow due to having  a previous stake in the escrow contract. 
## Vulnerability Detail
Assume that the Alice has a vest of 100 tokens and she stakes them in the voting escrow. After some time, owner decides to allocate 200 tokens more to Alice by calling the [setUserVesting](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L484C14-L487). Alice would naturally wants to increase her stake at escrow contract since she now has 300 tokens. However, Alice can't stake the new 200 tokens the owner granted to her due to [this](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L245-L247) check.

PoC:
```solidity
// forge test --fork-url "YOUR-FANCY-RPC-URL-HERE" --match-contract TrufVestingTest --match-test test_PartialStakingNotPossible -vv
    function test_PartialStakingNotPossible() external {
        console.log("Stake vesting to veTRUF");

        _setupVestingPlan();
        _setupExampleUserVestings();

        uint256 categoryId = 0;
        uint256 vestingId = 0;
        uint256 stakeAmount = 10e18;
        uint256 duration = 30 days;

        vm.startPrank(alice);
        vm.expectEmit(true, true, true, true, address(vesting));
        emit Staked(categoryId, vestingId, alice, stakeAmount / 2, duration, 1);

        // stake the first half, maybe the user staked all and the owner granted him a new stake
        // in that case the user naturally wants to increase his/her staking position so let's see if 
        // that is possible
        vesting.stake(categoryId, vestingId, stakeAmount / 2, duration);

        (uint128 lockupAmount,,,, bool lockupIsVesting) = veTRUF.lockups(alice, 0);

        // we staked the half
        assertTrue(lockupAmount > 0, "Something staked");

        // stake the other half, tx will revert because there is already a lock
        // if the owner grants the user more vests, user can't stake that which is not the ideal 
        // output for the user.
        vm.expectRevert(abi.encodeWithSignature("LockExist()"));
        vesting.stake(categoryId, vestingId, stakeAmount / 2, duration);
    } 
```

Test result and Logs:
<img width="484" alt="image" src="https://github.com/sherlock-audit/2023-12-truflation-mstpr/assets/120012681/c522bc96-84fe-498a-9499-8db1c95d79d0">

## Impact
Vest owners who partially staked their vest tokens can't stake the full amount later. Also, if the owner gives more tokens vested for a user who had stake previously, that same user can't stake the freshly given vest tokens in the escrow. Considering those two points, I'll label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L241-L262
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L484-L516
## Tool used

Manual Review

## Recommendation
Allow user to restake his/her position when a new vest is granted.

# Issue M-4: Reward distribution assumes funds are dispersed even if there are no recipients 

Source: https://github.com/sherlock-audit/2023-12-truflation-judging/issues/126 

## Found by 
0xmystery, IllIllI, Kodyvim, ZanyBonzy, chaduke, p-tsanev, zzykxx
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

# Issue M-5: If a vest is cancelled or decreased when block.timestamp > tgeTime the tokens are stuck in the contract 

Source: https://github.com/sherlock-audit/2023-12-truflation-judging/issues/162 

## Found by 
cawfree, mstpr-brainbot
## Summary
When the owner of the vesting contract sets user vests, it's crucial for the owner to ensure they deposit enough TRUF tokens into the contract. This ensures that the vesters can stake these tokens in escrow and subsequently claim them from the vesting contract.

Additionally, there exists a 'cancelVest' function that doesn't check whether the 'tgeTime' is over or not, implying it can be called at any time. If this function is called after 'tgeTime,' it means the owner of the vesting contract cannot update the vesting category. Consequently, the TRUF amount associated with the canceled vest remains stuck in the contract.
## Vulnerability Detail
Let's consider a scenario where the contract has 10 vesters holding a total of 10K TRUF tokens, and block.timestamp >= tgeTime, indicating that [updating the vest info](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L398-L404) is not possible. Suppose one of the vesters, who holds 1k TRUF tokens, has their vesting canceled by the owner for some reason. As this vester has no claims, all 1k TRUF tokens are now free.

When the entire vesting period concludes, the other 9k TRUF tokens will be claimed by the remaining vesters. However, the 1k TRUF tokens from the canceled vester will remain idle in the contract.

Another issue which has the same root of this problem, when a users vest is updated to decrease there will be excess TRUF tokens in the vesting contract which again they will be stuck in the contract once everyone claims their TRUF.

Example: Alice has 1k TRUF tokens (means that Alice entitled to 1k TRUF and the vesting contract has 1k TRUF for Alice) and owner updates her vest and now she has 900 TRUF tokens. However, vesting contract still holds 100 TRUF tokens which is not entitled to anyone and they are stuck. 
## Impact
Actually, there's a way to save those tokens by making a vest to an address and then claiming them. But honestly, I don't think that's what anyone would really want to do intentionally to rescue those tokens. If the owner's plan is to claim those idle TRUF tokens by setting up fake vests and then claiming them, it might reduce the problem's impact or make it not a big deal. However, I don't think this is a good way to get hold of those unused tokens in the contract. Plus, figuring out the new vest gets really tricky with lots of vesting happening all at once. Considering all, I label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L398-L431
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L348-L388
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L484-L515
## Tool used

Manual Review

## Recommendation
Make a sweep function that only owner can call. This function can be called when all vests are over.



## Discussion

**ryuheimat**

https://github.com/truflation/truflation-contracts/pull/9

We decided to disable adding new category after tge, but allow to update max allocation after tge.
This way, we can move allocation to other categories.

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**Shaheen** commented:
>  Valid Medium. There is no way to rescue stuck funds, there is a way around but still good catch



