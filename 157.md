Skinny Fern Fly

high

# Miscalculation of `timeElapsed` in TrufVesting::claimable() will result in a higher `vestedAmount` resulting to substantial value of claims / rewards.

## Summary
In this code [line](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L184), the `timeElapsed` is miscalculated by dividing and multiplying the same variable `info.unit`.

## Vulnerability Detail
- The said [miscalculation]((https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L184)) (dividing and multiplying the same variable hence cancelling each other out) means that the `elapsedTime` is numerically larger than expected (because the value is actually in seconds). The `elapsedTime` value is used in `vestedAmount` [calculation](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L186), thus making the `vestedAmount` larger than expected also.

- Here's the [snippet](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L184-L186) for your reference. 
    ```solidity
    uint64 timeElapsed = ((uint64(block.timestamp) - startTime) / info.unit) * info.unit;

    uint256 vestedAmount = ((totalAmount - initialRelease) * timeElapsed) / info.period + initialRelease;
    ```
- Looking at the definitions of `info.unit`, it is pointing to `VestingInfo` [struct](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L94-L100) and the comment states that it is actually a conversion to monthly, 6-months, etc. 
    ```solidity
    struct VestingInfo {
        uint64 initialReleasePct; // Initial Release percentage
        uint64 initialReleasePeriod; // Initial release period after TGE
        uint64 cliff; // Cliff period
        uint64 period; // Total period
        uint64 unit; // The period to claim. ex. montlhy or 6 monthly
    }
    ```    

## Impact
Erroneous amount of tokens will be distributed as claims / rewards. 

## Code Snippet
- the [culprit](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L184-L186)
- the [struct definitions](https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/TrufVesting.sol#L94-L100)


## Tool used

Manual Review

## Recommendation
- I believe the intention of `info.unit` is to have a standard unit of time dynamically. If `timeElapsed` is per day, then the `info.unit` should have a value of 86400 (1 day = 86400 secs) and should be divided as stated below.

```diff
-- uint64 timeElapsed = ((uint64(block.timestamp) - startTime) / info.unit) * info.unit;
++ uint64 timeElapsed = (uint64(block.timestamp) - startTime) / info.unit;

uint256 vestedAmount = ((totalAmount - initialRelease) * timeElapsed) / info.period + initialRelease;
```

- Another option is to just remove the `info.unit` all together but just keep in mind that `timeElapsed` is by default is in seconds. Hence other variables meant to express time should also be adjusted in seconds. 
```diff
-- uint64 timeElapsed = ((uint64(block.timestamp) - startTime) / info.unit) * info.unit;
++ uint64 timeElapsed = uint64(block.timestamp) - startTime;

uint256 vestedAmount = ((totalAmount - initialRelease) * timeElapsed) / info.period + initialRelease;
``` 
