Energetic Canvas Bat

medium

# Lack of Explicit Check for Sufficient Balance in `TrufMigrator.sol`

## Summary

The `migrate()` function in the `TrufMigrator.sol` lacks an explicit check to ensure that the contract has a sufficient token balance before attempting to transfer tokens to `msg.sender`. 

## Vulnerability Detail

Token balance check is missing before token transfer call

```solidity

    function migrate(
        uint256 index,
        uint256 amount,
        bytes32[] calldata proof
    ) external {
        .
        .
        .
        trufToken.safeTransfer(msg.sender, migrateAmount);
        .
        .
    }
```

## Impact

While the Ethereum network will automatically revert a transaction if the sender does not have a sufficient token balance, explicitly checking the balance before initiating the transfer provides an additional layer of validation and can help prevent unnecessary gas costs associated with failed transactions.

## Code Snippet

```solidity

    function migrate(
        uint256 index,
        uint256 amount,
        bytes32[] calldata proof
    ) external {
        .
        .
        .
        trufToken.safeTransfer(msg.sender, migrateAmount);
        .
        .
    }
```

https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufMigrator.sol#L64C1-L65C1

## Tool used

Manual Review

## Recommendation

```solidity
  require(
     trufToken.balanceOf(address(this)) >= migrateAmount,
     "Not Enough Tokens"
);
```
