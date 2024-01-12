Muscular Butter Stallion

medium

# Partial staking is not possible, updated vested users can't stake the new amount if they had previous stake

## Summary
When user stakes his vest, user must stakes the full amount to get the best outcome because partial staking is not allowed for vests. However, there are some scenarios where user can get more vests but not able to stake it in the voting escrow due to having  a previous stake in the escrow contract. 
## Vulnerability Detail
Assume that the Alice has a vest of 100 tokens and she stakes them in the voting escrow. After some time, owner decides to allocate 200 tokens more to Alice by calling the [setUserVesting](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L484C14-L487). Alice would naturally wants to increase her stake at escrow contract since she now has 300 tokens. However, Alice can't stake the new 200 tokens the owner granted to her due to [this](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L245-L247) check.

PoC:
```solidity
// forge test --fork-url "YOUR-FANCY-RPC-URL-HERE" --match-contract TrufVestingTest --match-test test_PartialStakingNotPossible -vv
    function test_PartialStakingNotPossible() external {
        console.log("Stake vesting to veTRUF");

        _setupVestingPlan();
        _setupExampleUserVestings();

        uint256 categoryId = 0;
        uint256 vestingId = 0;
        uint256 stakeAmount = 10e18;
        uint256 duration = 30 days;

        vm.startPrank(alice);
        vm.expectEmit(true, true, true, true, address(vesting));
        emit Staked(categoryId, vestingId, alice, stakeAmount / 2, duration, 1);

        // stake the first half, maybe the user staked all and the owner granted him a new stake
        // in that case the user naturally wants to increase his/her staking position so let's see if 
        // that is possible
        vesting.stake(categoryId, vestingId, stakeAmount / 2, duration);

        (uint128 lockupAmount,,,, bool lockupIsVesting) = veTRUF.lockups(alice, 0);

        // we staked the half
        assertTrue(lockupAmount > 0, "Something staked");

        // stake the other half, tx will revert because there is already a lock
        // if the owner grants the user more vests, user can't stake that which is not the ideal 
        // output for the user.
        vm.expectRevert(abi.encodeWithSignature("LockExist()"));
        vesting.stake(categoryId, vestingId, stakeAmount / 2, duration);
    } 
```

Test result and Logs:
<img width="484" alt="image" src="https://github.com/sherlock-audit/2023-12-truflation-mstpr/assets/120012681/c522bc96-84fe-498a-9499-8db1c95d79d0">

## Impact
Vest owners who partially staked their vest tokens can't stake the full amount later. Also, if the owner gives more tokens vested for a user who had stake previously, that same user can't stake the freshly given vest tokens in the escrow. Considering those two points, I'll label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L241-L262
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L484-L516
## Tool used

Manual Review

## Recommendation
Allow user to restake his/her position when a new vest is granted.