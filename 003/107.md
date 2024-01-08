Shambolic Fossilized Pelican

medium

# The implementation of _extendLock() is incorrect.

## Summary
The implementation of _extendLock() is incorrect. The NatSpec says ``The stake end time is computed from the current time + duration, just   like it is for new stakes. So a new stake for seven days duration and an old stake extended with a seven days duration would have the same end". However, the implementation does not calculate the new end time as specified. Both newEnd and the total duration of staking is calculated wrongly. 

## Vulnerability Detail
Based on the NatSpec:
1. The newEnd should be block.timestamp + duration
2. The total duration of staking should be: 
     existing duration: block.timestamp - (oldEnd - oldDuration) 
     so total duration should be block.timestamp - (oldEnd - oldDuration)  + duration

Unfortunately, the current implementation _extendLock() has two mistakes: 
1) It does not extend from "now" but extend from existing ``oldEnd``
2) It calculates the total duration as oldDuration + duration.

This is different from extending the lock with ``duration`` seconds from now on. 

## Impact
The implementation of _extendLock() is incorrect. The behavior of extension will be different from what users expected. 

## Code Snippet
[https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/VotingEscrowTruf.sol#L333C5-L366C6](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/VotingEscrowTruf.sol#L333C5-L366C6)

## Tool used
VSCode

Manual Review

## Recommendation
The correct implementation is as follows. Note how the newEnd and newDuration are calculated. 

```javascript
function _extendLock(address user, uint256 lockupId, uint256 duration, bool isVesting) internal {
        // duration checked inside previewPoints
        Lockup memory lockup = lockups[user][lockupId];
        if (lockup.isVesting != isVesting) {
            revert NoAccess();
        }

        uint256 amount = lockup.amount;
        uint256 oldEnd = lockup.end;
        uint256 oldPoints = lockup.points;
        uint256 newDuration = block.timestamp - (oldEnd - lockup.duration) + duration; // @audit

        (uint256 newPoints,) = previewPoints(amount, newDuration);

        if (newPoints <= oldPoints) {
            revert NotIncrease();
        }

        uint256 newEnd = block.timestamp + duration;  // @audit

        uint256 mintAmount = newPoints - oldPoints;

        lockup.end = uint128(newEnd);
        lockup.duration = uint128(newDuration);
        lockup.points = newPoints;

        lockups[user][lockupId] = lockup;

        stakingRewards.stake(user, mintAmount);
        _mint(user, mintAmount);

        emit Unstake(user, isVesting, lockupId, amount, oldEnd, oldPoints);
        emit Stake(user, isVesting, lockupId, amount, newEnd, newPoints);
    }
```
