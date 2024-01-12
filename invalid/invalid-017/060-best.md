Stable Strawberry Tadpole

medium

# Contracts don't call Ownable/Ownable2Step constructor

## Summary

Several contracts utilize OpenZeppelin's Ownable contract. However, since OZ has upgraded to 0.5.0 their API has [changed](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/CHANGELOG.md#access). Now contracts that inherit from Ownable should call the Ownable constructor when the contract's constructor is called. In addition, this also extends to Ownable2Step. However this does not occur on the following contracts:


Inherits Ownable:
- VirtualStakingRewards
- TrufVesting
- TrufPartner (not in scope)

Inherits Ownable2Step:
- TrufMigrator
- TruflationTokenCCIP

## Vulnerability Detail

In order to initialize Ownable for a given contract, the Ownable constructor should be called in order to set the owner of the contract. If constructor is not called, no owner can be set for the contract.

## Impact

Contract will not have an owner set for the contract.

## Code Snippet

https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/staking/VirtualStakingRewards.sol#L65-L71

https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L141-L150

## Tool used

Manual Review

## Recommendation

Call Ownable constructor in the contract constructor. This is applicable for both Ownable and Ownable2Step inherited contracts. For example, VirtualStakingRewards constructor should look like:

```solidity
// AUDIT: See Ownable below
constructor(address _rewardsDistribution, address _rewardsToken) Ownable(msg.sender) {
    if (_rewardsToken == address(0) || _rewardsDistribution == address(0)) {
        revert ZeroAddress();
    }
    rewardsToken = _rewardsToken;
    rewardsDistribution = _rewardsDistribution;
}
```