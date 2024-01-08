Clever Boysenberry Rabbit

high

# `cancelVesting` will potentially not give users unclaimed, vested funds, even if giveUnclaimed = true

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
