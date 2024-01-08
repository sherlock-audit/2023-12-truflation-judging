Glamorous Sable Nightingale

medium

# VotingEscrowTruf::`stake()` do not hold the returned `lockupId`

## Summary
When the [`stake()`](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/VotingEscrowTruf.sol#L96) is called by one user to stake for another user this function do not holds the returned value which is `lockupId` of the position.

## Vulnerability Detail
The internal [`_stake()`](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/VotingEscrowTruf.sol#L137) returns `lockupId` for a staked position, this function is called by `stake()` but this `stake()` does not hold the `lockupId`.

## Impact
Calling this function will not return `lockupId` of the position.

## Code Snippet
```solidity
function stake(uint256 amount, uint256 duration, address to) external {
    _stake(amount, duration, to, false); // @audit-issue _stake() returns a lockUp Id, but here is nothing to hold it
  }
```

## Tool used

Manual Review

## Recommendation
Replace this function with this:
```solidity
function stake(uint256 amount, uint256 duration, address to) external  returns(uint256 lockupId) {
   lockupId =  _stake(amount, duration, to, false); 
  }
```