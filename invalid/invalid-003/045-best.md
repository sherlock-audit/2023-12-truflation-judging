Fun Opal Squid

medium

# zero address verification

## Summary
withdraw function should check  for the zero  address .
## Vulnerability Detail
 function withdraw(address user, uint256 amount) public updateReward(user) onlyOperator {
        if (amount == 0) {
            revert ZeroAmount();
        }
        _totalSupply -= amount;
        _balances[user] -= amount;
        emit Withdrawn(user, amount);
    }
## Impact
 _totalSupply can be changed by calling withdraw function.
## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/staking/VirtualStakingRewards.sol#L117
## Tool used

Manual Review

## Recommendation
 if (user == address(0)) {
            revert ZeroAddress();
        }