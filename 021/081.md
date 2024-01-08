Energetic Canvas Bat

medium

# Explicit check to ensure `id` passed to `setVestingCategory( uint256 id, string calldata category,uint256 maxAllocation,bool adminClaimable)` exists or not

## Summary

The `setVestingCategory( uint256 id, string calldata category,uint256 maxAllocation,bool adminClaimable)` function in the `TrufVesting.sol` lacks an explicit check to ensure that the `id` passed to `setVestingCategory( uint256 id, string calldata category,uint256 maxAllocation,bool adminClaimable)` exists or not

## Vulnerability Detail

Array length check is missing 

```solidity
    function setVestingCategory(
        uint256 id,
        string calldata category,
        uint256 maxAllocation,
        bool adminClaimable
    ) public onlyOwner {
        ...

        int256 tokenMove;
        if (id == type(uint256).max) {
            ...
        } else {
            ----HERE----
            if (category.allocated > category.maxAllocation) {
                revert MaxAllocationExceed();
            }
            ...
        }

       ...

        emit VestingCategorySet(id, category, maxAllocation, adminClaimable);
    }
```

## Impact

Prevent unnecessary gas costs associated with failed transactions

## Code Snippet

```solidity
    function setVestingCategory(
        uint256 id,
        string calldata category,
        uint256 maxAllocation,
        bool adminClaimable
    ) public onlyOwner {
        ...

        int256 tokenMove;
        if (id == type(uint256).max) {
            ...
        } else {
            ----HERE----
            if (category.allocated > category.maxAllocation) {
                revert MaxAllocationExceed();
            }
            ...
        }

       ...

        emit VestingCategorySet(id, category, maxAllocation, adminClaimable);
    }
```

https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L398C1-L431C6

## Tool used

Manual Review

## Recommendation

```solidity

  if (id>= categories.length) {
            revert InvalidVestingCategory(id);
   }

```
