Harsh Ginger Mallard

medium

# `VotingEscrowTruf.extendLock()` forget `points` overflow validation like in other place in the contract.


## Summary

`VotingEscrowTruf.stake()` revert when there is too many `points` or Voting token in circulation.

```solidity
File: VotingEscrowTruf.sol
150:         if (points + totalSupply() > type(uint192).max) {
151:             revert MaxPointsExceeded();//@ points always smaller amount of token transfered in.
152:         }//@audit L no revert on zero points.

```

`VotingEscrowTruf._extendLock()` add new points when extend lock but does not check for overflow like in staking function.
<https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/VotingEscrowTruf.sol#L345-L349>

## Vulnerability Detail

Realistically, this is not a vulnerability. Because `points` ,aka minted `veTRUF`, is always smaller than TRUF token totalSupply.
Because you can only swap TRUF for same amount of veTRUF when stake for whole 4 years. And you cannot stake longer than that. Any less will give you portion of veTRUF based on time linearly.

<https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/VotingEscrowTruf.sol#L270-L280>

```solidity
    function previewPoints(uint256 amount, uint256 duration) public view returns (uint256 points, uint256 end) {
        if (duration < minStakeDuration) {
            revert TooShort();
        }
        if (duration > MAX_DURATION) {
            revert TooLong();
        }
        //@points = amount * duration(3600:) / 94608000;
        points = amount * duration / MAX_DURATION;//@points is veTRUF about being minted
        end = block.timestamp + duration;
    }
```

It is impossible to veTRUF token overflow `type(uint192).max`. The limit is 1B token `1_000_000_000e18`.

## Impact

Missing validation but impossible overflow scenario.

## Code Snippet

<https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/VotingEscrowTruf.sol#L150-L152>
<https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/VotingEscrowTruf.sol#L345-L349>

## Tool used

Manual Review

## Recommendation

Safely remove `revert MaxPointsExceeded();` validation
