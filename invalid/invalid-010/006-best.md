Atomic Porcelain Bat

high

# Users will never be able to fully migrate tokens from Ethereum to Base

## Summary
The protocol wants to move towards Base network leaving Ethereum. The purpose of the `TrufMigator::migrate()` function is to enable users to claim new TRUF tokens within the Base L2 network. However, the logic implementation is wrong.

## Vulnerability Detail
The issue with the `migrate()` function lies in the default initialization of `_migratedAmount` to 0. The conditional statement verifies if the user's token amount is greater than the previously migrated amount. However, this setup poses a problem as users may choose to migrate only a specific token quantity. Any prior transfers of tokens lower than the current value restrict the user from transferring tokens in subsequent attempts.

### First Scenario
1) Let say that Alice has 100 tokens in Ethereum and 0 in Base. 
2) Alice migrates 50 tokens to Base. 
3) `uint256 _migratedAmount` will be 50.
4) The next day, Alice decided to migrate another 50 tokens. 
5) She is not able to do so due to the if statement, hence, the TFI token will be stuck in Ethereum.

### Second Scenario
1) Let say that Alice has 200 tokens in Ethereum and 0 in Base. 
2) Alice migrates 50 tokens to Base. `uint256 _migratedAmount` will be 50.
3) The next day, Alice decided to migrate another 100 tokens. 
4) Alice will bypass the if statement as `amount > _migratedAmount`, the main issue is accounting for the user's migrated balance. 
5) The `migratedAmount[msg.sender] = amount;` will become 100 instead of 150. Thus only 50 tokens will be sent, which isn't what the user wants.
This means that the next time Alice wants to empty and move to Base with the final 100 tokens. She will not be able to do so.

## Impact
Due to this issue, user tokens will become stranded on the tp-be depreciated Ethereum network, leading to a denial of service with funds locked as they won't be able to migrate to the Base network.

## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufMigrator.sol#L55C1-L64C59
```solidity
function migrate(uint256 index, uint256 amount, bytes32[] calldata proof) external {
    bytes32 leaf = keccak256(abi.encode(msg.sender, index, amount));
    if (MerkleProof.verify(proof, merkleRoot, leaf) == false) {
        revert InvalidProof();
    }
    //@note Initialized with the previously migrated amount for the user, defaults to 0 if not migrated before.
    uint256 _migratedAmount = migratedAmount[msg.sender];
    // Thus always revert
    if (amount <= _migratedAmount) { //@audit Denial of Service
        revert AlreadyMigrated();
    }
    migratedAmount[msg.sender] = amount;
    uint256 migrateAmount = amount - _migratedAmount;
    trufToken.safeTransfer(msg.sender, migrateAmount);
    emit Migrated(msg.sender, migrateAmount);
}
```
## Tool used

Manual Review

## Recommendation
Determine and track the permissible token quantities for migration by considering both the total amount migrated and the intended migration amount. Ensure that the logic does not exceed the remaining actual token balance in the user's Ethereum network account.