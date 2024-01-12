Perfect Lemonade Bison

medium

# exit() in VirtualStakingRewards contract will always fail

## Summary
In VirtualStakingRewards, there's a function that consistently reverts, rendering it unusable for regular users.

Under normal protocol circumstances, the operator will be the VotingEscrowTruf contract. The owner retains the ability to designate the operator through this function:

```solidity
  function setOperator(address _operator) external onlyOwner {
        if (_operator == address(0)) {
            revert ZeroAddress();
        }
        operator = _operator;

        emit OperatorUpdated(_operator);
    }
```

The operator contract is responsible for executing the stake and unstake (withdraw) functions, as they are safeguarded by the onlyOperator modifier:

```solidity
function stake(address user, uint256 amount) external updateReward(user) onlyOperator {}
function withdraw(address user, uint256 amount) public updateReward(user) onlyOperator {}
```

However in this case users who will utilize the exit() in VirtualStakingRewards in order to withdraw wont succeed and the transaction will fail.

## Vulnerability Detail

The exit() function serves as a user-oriented feature designed to enable fund withdrawal. Within the execution of this function, a call is made to withdraw() as follows:

```solidity
 withdraw(user, _balances[user]);
```
However, this call will consistently encounters failure because the msg.sender equals the user, not the operator.

For a practical demonstration illustrating this issue, here is a POC showcasing this behavior you can add it in VotingEscrowTrufTest.

```solidity
    function testExitWillFail() external {
        console.log("Exit function in VirtualStakingRewards will always fail");
        vm.startPrank(alice);
        uint256 amount = 100e18;
        uint256 duration = 30 days;
        trufToken.approve(address(veTRUF), amount);
        veTRUF.stake(amount, duration, alice);

        vm.warp(block.timestamp + 10 days);
        vm.expectRevert(); 
        trufStakingRewards.exit(alice);
        vm.stopPrank();
    }
```
You can run it with 
```bash
forge test --match-contract VotingEscrowTrufTest --match-test testExitWillFail -vvvvv
```


## Impact

The user will lose the gas fees for this transaction and the transaction will revert.


## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/staking/VirtualStakingRewards.sol#L135-L140

## Tool used

Manual Review

## Recommendation

Given that both staking and unstaking operations are managed within the operator contract, namely VotingEscrowTruf, it is better to remove the exit() functions from VirtualStakingRewards contract. In case we need a user function that let him both unstake and claimrewards we can indeed do that in VotingEscrowTruf.


