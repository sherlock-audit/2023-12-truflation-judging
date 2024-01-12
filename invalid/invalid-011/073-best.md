Custom Shadow Osprey

medium

# Minting wrong amount for initial supply in `TruflationToken.sol`

## Summary

Minting wrong amount.

## Vulnerability Detail

In the contract comments, it is stated that the total supply should be `total supply: 100,000,000 TRUF.` However, in the constructor of the contract, the mint function is coded with `1_000_000_000e18`.

```javascript
/**
 * @title TruflationToken smart contract
 * @author Ryuhei Matsuda
 * @notice ERC677 Token like LINK token
 *      name: Truflation
 *      symbol: TRUF
 *      total supply: 100,000,000 TRUF
 */
contract TruflationToken is ERC677Token {
    constructor() ERC20("Truflation", "TRUF") {
@>      _mint(msg.sender, 1_000_000_000e18);
    }
}
```

## Impact

Minting the wrong amount could cause problems. It might mess up calculations, mishandle the token system, and affect how the token contract works.

## Code Snippet

https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TruflationToken.sol#L16

## Tool used

Manual Review

## Recommendation
Fix the number in `mint` function.

```diff
/**
 * @title TruflationToken smart contract
 * @author Ryuhei Matsuda
 * @notice ERC677 Token like LINK token
 *      name: Truflation
 *      symbol: TRUF
 *      total supply: 100,000,000 TRUF
 */
contract TruflationToken is ERC677Token {
    constructor() ERC20("Truflation", "TRUF") {
-        _mint(msg.sender, 1_000_000_000e18);
+        _mint(msg.sender, 100_000_000e18);
    }
}
```