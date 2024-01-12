Funny Crepe Mustang

medium

# Frontrunning of ERC20Permit's permit function may lead to loss of user funds.

## Summary

VotingEscrowTruf.sol contract uses ERC20permit from OZ to allow users delegate staking rights to another address. The protocol does not use security considerations suggested by OZ on using permit and makes users vulnerable to frontrunning in transferring staking rights to another user. 

## Vulnerability Detail
VotingEscrowTruf.sol contract uses ERC20permit for the function stake to transfer staking rights to another address. 
Per Openzeppelin documentation: "There are two important considerations concerning the use of permit. The first is that a valid permit signature expresses an allowance, and it should not be assumed to convey additional meaning. In particular, it should not be considered as an intention to spend the allowance in any specific way. The second is that because permits have built-in replay protection and can be submitted by anyone, they can be frontrun. A protocol that uses permits should take this into consideration and allow a permit call to fail."

The current protocol fails to follow that security advice and does not use 2 separate functions before transferring the tokens to the staking contract. 

This issue can be observed with Certora formal verification. The following rule fails with the contract VotingEscrowTruf.sol.

rule permitFrontRun(){
    env e1;
    env e2;

    address clientHolder;
    address clientSpender;
    uint256 clientAmount;
    uint256 clientDeadline;
    uint8 clientV;
    bytes32 clientR;
    bytes32 clientS;

    address attackerHolder;
    address attackerSpender;
    uint256 attackerAmount;
    uint256 attackerDeadline;
    uint8 attackerV;
    bytes32 attackerR;
    bytes32 attackerS;

    require e1.msg.sender != e2.msg.sender;

    storage init = lastStorage;

    permit(e1, clientHolder, clientSpender, clientAmount, clientDeadline, clientV, clientR, clientS); // if pass not reverted
    
    permit(e2, attackerHolder, attackerSpender, attackerAmount, attackerDeadline, attackerV, attackerR, attackerS) at init; // attacker attack

    permit@withrevert(e1, clientHolder, clientSpender, clientAmount, clientDeadline, clientV, clientR, clientS);

    assert !lastReverted, "Cannot sign permit with same signature";
}

The run where the rule fails can be found at:

https://prover.certora.com/output/548871/46f2cd0998464754821e94db9f4d4b9d/?anonymousKey=d7f20c1ce3b90e04153eb2eac9cdeb3e79d9770b



## Impact
Loss of user staking rights or depending on the staking period and the time to discover the hack may lead to total loss of the user funds.
## Code Snippet

    function stake(uint256 amount, uint256 duration, address to) external {
        _stake(amount, duration, to, false);
    }

https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/VotingEscrowTruf.sol#L96-L98


## Tool used

Manual Review and Certora formal verification.


## Recommendation

Follow the OZ instructions, allow permit in one function and do the token transfer in another function.
