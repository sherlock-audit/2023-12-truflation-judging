Young Gauze Dachshund

high

# As anyone can call the function exit which leads to user getting less rewards which is unfair.

## Summary
User may not want to exit from virtualstakingrewards contract but anyone can call the exit function on behalf of any user.


## Vulnerability Detail
1. Let assume , Alice stakes 100e18 TRUF token with duration 1 year by calling  function stake(VotingEscrowTruf contract).
2. Now alice’s point = (100e18*1 year)/3 year = 33.33e18.
3. See the function _stake(VotingEscrowTruf contract). This function calls the VirtualStakingRewards contract’s stake function i.e  stakingRewards.stake(alice, 33.33e18);
4. Now if you see VirtualStakingRewards contract’s stake function , 33.33e18 amounts are staked in the VirtualStakingRewards contract for alice i.e  _balances[alice] = 33.33e18;
5. As anyone can call the function exit(VirtualStakingRewards contract) for anyone, so bob calls the exit function for alice , see the exit function, as  _balances[alice] = 33.33e18, so withdraw function is called i.e withdraw(alice, 33.33e18). After that  _balances[alice] = 0; this is unfair as alice may not want to exit.

## Impact
alice will get less reward. 

## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/staking/VirtualStakingRewards.sol#L135
## Tool used

Manual Review

## Recommendation

Make sure that caller and  usersare same when  caling the function exit 