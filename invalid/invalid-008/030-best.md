Gigantic Clear Buffalo

medium

# The contract allows unchecked expansion of `maxAllocation`, potentially leading to misallocation and unexpected behavior.

## Summary
The contract lacks proper validation when increasing the `maxAllocation` for a vesting category, allowing the possibility of exceeding the maximum allocation limit even when nothing has been allocated.

## Vulnerability Detail
The vulnerability is present in the `setVestingCategory` function, specifically in the block that handles the case when id is not equal to `type(uint256).max`. In this case, the function checks whether the new `maxAllocation` is greater than the current allocation. If it is, the function reverts with the `MaxAllocationExceed()` error. However, this check does not consider the scenario where no allocation has occurred yet, potentially leading to an increase in the `maxAllocation` without proper validation.

## Impact
The impact of this vulnerability is that it allows an increase in the `maxAllocation` for a vesting category without considering whether any allocation has occurred. This could potentially lead to mismanagement of allocation limits and unexpected behavior in the contract.


## Code Snippet
[Link](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L398-L431)
## Tool used

Manual Review

## Recommendation
A proper validation check should be implemented to ensure that the new `maxAllocation` is greater than or equal to the total allocated amount for the category. This validation should consider the scenario where no allocation has occurred yet.