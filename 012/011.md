Massive Umber Spider

medium

# TrufVesting.cancelVesting calculates end of vesting incorrectly

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