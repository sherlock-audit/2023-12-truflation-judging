Huge Foggy Weasel

medium

# Vesters can sometimes unfairly claim tokens, DOS-ing other users

## Summary

Everyone in the same vesting [category](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L398) has the same vesting cap and [emission schedule](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L439), and are supposed to differ based only on their [start times and total amounts](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L484). If the rate of emission is slower than the rate of vesting, tokens within a category aren't fairly distributed.


## Vulnerability Detail

If the amount of monthly emission is smaller than the rate of all users vesting, or if the start times of users are significantly different (as would be expected for employees joining at different times), then users who claim their tokens first, can claim more of the tokens than other users in the same category, DOS-ing the other users.


## Impact

There will be a race to see who can claim their vested tokens first. If a user with a large amount vesting claims first, people with fewer tokens vesting will be unable to claim all of their tokens until they either waste funds calling claim faster and more repeatedly than everyone else doing the same thing, or until all tokens vest (which will likely be a >1 year DOS).

For example (just focusing on the [cliff amounts](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L178-L188)):

1. Users Alice,Bob,Carol,...,Zane each are vesting 1,000 tokens per month, all on the same day. Assume that the emission schedule is 1,000 tokens per month, on the date they all vest.
2. After 1 month, Alice vests and claims 1,000 tokens, and everyone else is stuck because only 1,000 have been emitted so far
3. At month 2, Bob manages to wake up early and claim 1,000 tokens
4. At month 3, Alice was the lucky one again, and everyone else got nothing
...
N. This goes on for a year before Zane is finally able to get his monthly transaction in first. Even after a year, he has 12,000 "vested", but has only been able to actually claim 1,000 of them

This is an exaggerated schedule to illustrate the issue, but the same sort of thing can happen with more reasonable schedules and amounts.


## Code Snippet

If a user has vested more tokens than have been emitted by the schedule, the user can claim more than their proportion of the total allocation:
```solidity
// File: src/token/TrufVesting.sol : TrufVesting.claimable()   #1

196 @>         claimableAmount = vestedAmount - userVesting.claimed;
197            uint256 emissionLeft = getEmission(categoryId) - categories[categoryId].totalClaimed;
198    
199            if (claimableAmount > emissionLeft) {
200 @>             claimableAmount = emissionLeft;
201:           }
```
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L196-L201


## Tool used

Manual Review


## Recommendation

Scale the `emissionLeft` by the proportion of the category's [`allocated`](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L88) amount that that user's [`amount`](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L104) represents.
