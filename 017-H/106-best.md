Muscular Butter Stallion

high

# ERC20Votes CLOCK_MODE() must need to take block.timestamp as time reference

## Summary
In the ERC20Votes token type, there exists a `CLOCK_MODE` function that defines the units of time in voting operations. The default value for this is 'block.number.' However, in certain chains like OP and Base chains, using 'block.number' may not be advisable. This is due to the potential for 'block.number' to be manipulated significantly, leading to premature vote endings and creating timing discrepancies. The current code does not override this behavior, deploying the voting escrow contract to Base mainnet posing a serious threats for the voting procedures. 
## Vulnerability Detail
First, let's check the ERC20Votes `CLOCK_MODE` function:
```solidity
/**
     * @dev Description of the clock
     */
    // solhint-disable-next-line func-name-mixedcase
    function CLOCK_MODE() public view virtual override returns (string memory) {
        // Check that the clock was not modified
        require(clock() == block.number, "ERC20Votes: broken clock mode");
        return "mode=blocknumber&from=default";
    }
```
As seen, the default for the function is "block.number." Therefore, all voting logic relies on "block.number" as time references. However, since the Base mainnet operates on the OP stack, it shares the same "block.number" property as the OP mainnet, where "block.number" returns the L2 block number and operates at a considerably fast pace. Nearly every 2 seconds, a block can be generated, and it depends on network congestion, allowing users to submit dummy transactions to expedite the process even further.

For further insights on OP blocks, refer to: [OP Blocks Documentation](https://docs.optimism.io/stack/protocol/overview#block-production)

In such circumstances, using the ERC20Votes clock unit as "block.number" could result in excessive governance manipulation.

## Impact
High, since voting actions are not reliable and easily manipulatable. Governor can't pick a decent number for block.number in future. I don't think there is any way to have a healthy governance with "block.number" in Base so this made me feel about this issue as high rather than a medium. 
## Code Snippet
https://github.com/OpenZeppelin/openzeppelin-contracts/blob/ef68ac3ed83f85e4ce078b26170280c468ffeabb/contracts/governance/utils/Votes.sol#L58-L72
## Tool used

Manual Review

## Recommendation
Override the clock mode and set it to block.timestamp