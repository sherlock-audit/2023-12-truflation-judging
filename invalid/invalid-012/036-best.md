Bouncy Chiffon Aardvark

medium

# Potential  Second-Preimage Attack

## Summary
The function `migrate` in the contract `TrufMigrator.sol` is related to the risk of a [second-preimage attack](https://flawed.net.nz/2018/02/21/attacking-merkle-trees-with-a-second-preimage-attack/)

## Vulnerability Detail
The vulnerability lies in the generation of the leaf variable using `keccak256` to hash the concatenation of the `msg.sender`, `index`, and `amount` parameters. This approach is susceptible to a [second-preimage attack](https://flawed.net.nz/2018/02/21/attacking-merkle-trees-with-a-second-preimage-attack/) where an attacker could potentially find a different input that hashes to the same value as the original leaf without possessing the original input.

## Impact
If a successful "second-preimage attack" occurs, an attacker could present a different set of inputs that would generate the same leaf value. Consequently, the `MerkleProof.verify` function would incorrectly validate the proof

## Code Snippet

https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufMigrator.sol#L49

```solidity
function migrate(uint256 index, uint256 amount, bytes32[] calldata proof) external {
        bytes32 leaf = keccak256(abi.encode(msg.sender, index, amount));

        if (MerkleProof.verify(proof, merkleRoot, leaf) == false) {
            revert InvalidProof();
        }
```

## Tool used
Manual Review

## Recommendation
One potential way:
```solidity
bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(addr, amount))));
```
For more details: https://github.com/OpenZeppelin/merkle-tree?tab=readme-ov-file#standard-merkle-trees