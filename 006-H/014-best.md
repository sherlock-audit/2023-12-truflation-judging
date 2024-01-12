Fantastic Canvas Platypus

high

# VotingEscrowTruf.sol#migrateVestingLock() - migrating a lock does not reset delegate votes

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
