Keen Amethyst Moth

high

# When migrating the owner users will lose their rewards

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
