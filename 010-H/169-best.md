Noisy Mercurial Hornet

high

# users still receive rewards and have voting power even after their lockups expires in VotingEscrowTruf

## Summary
users can stake(lockup) their tokens in VotingEscrowTruf for some duration and receive voting power and rewards. the issue is that when the lockups expires users still receive rewards and have voting power for those expired lockups. code doesn't automatically handle the expired lockups and doesn't allow 3rd party to trigger handling the expired lockups. so users would receive rewards and voting power even after lockup expiration while they don't take risk of lockups.

## Vulnerability Detail
when users stake and lock their tokens code mint voting power points for them and also they become eligible for receiving rewards in `stakingRewards` contract:
```javascript
        stakingRewards.stake(to, points);
        _mint(to, points);
```
the issue is that voting power and also stakingRewards doesn't handle expire time and those minted and staked amounts for user would be always valid even after lockup expire time. so while user lockup their token for some duration and take risk for that duration he can receive rewards have voting power after expiration of that lockup and code doesn't handle expired lockups automatically.

also there is no logic that 3rd party could trigger it and code would remove expired lockups voting power and staked rewards so 3rd party can't help handling the expired lockups.

only the user himself can call unstake and code would remove his voting power and reward eligibility, but users won't be incentive to do this for themself because they can just keep receiving the reward and voting power without risk of lookups.

## Impact
users voting power and reward eligibility is still valid after lockup expiration. this way users would receive benefits of lockups while their lockups are expired and they don't take risk of lockups. 

## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/VotingEscrowTruf.sol#L183-L186
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/VotingEscrowTruf.sol#L165-L168

## Tool used
Manual Review

## Recommendation
because voting power and stakingRewards doesn't handle expire time, fixing this issue is hard. one way to fix it is to have `handleExpireLookup(lookUpID)` and allow anyone call it and receive a percentage of the staked amount(very low). this way 3rd party would  and user himself would have  incentive to call this function as lockups expires and code would remove voting power and staking amount of those expired locks.