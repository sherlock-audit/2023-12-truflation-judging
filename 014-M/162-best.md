Muscular Butter Stallion

medium

# If a vest is cancelled or decreased when block.timestamp > tgeTime the tokens are stuck in the contract

## Summary
When the owner of the vesting contract sets user vests, it's crucial for the owner to ensure they deposit enough TRUF tokens into the contract. This ensures that the vesters can stake these tokens in escrow and subsequently claim them from the vesting contract.

Additionally, there exists a 'cancelVest' function that doesn't check whether the 'tgeTime' is over or not, implying it can be called at any time. If this function is called after 'tgeTime,' it means the owner of the vesting contract cannot update the vesting category. Consequently, the TRUF amount associated with the canceled vest remains stuck in the contract.
## Vulnerability Detail
Let's consider a scenario where the contract has 10 vesters holding a total of 10K TRUF tokens, and block.timestamp >= tgeTime, indicating that [updating the vest info](https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L398-L404) is not possible. Suppose one of the vesters, who holds 1k TRUF tokens, has their vesting canceled by the owner for some reason. As this vester has no claims, all 1k TRUF tokens are now free.

When the entire vesting period concludes, the other 9k TRUF tokens will be claimed by the remaining vesters. However, the 1k TRUF tokens from the canceled vester will remain idle in the contract.

Another issue which has the same root of this problem, when a users vest is updated to decrease there will be excess TRUF tokens in the vesting contract which again they will be stuck in the contract once everyone claims their TRUF.

Example: Alice has 1k TRUF tokens (means that Alice entitled to 1k TRUF and the vesting contract has 1k TRUF for Alice) and owner updates her vest and now she has 900 TRUF tokens. However, vesting contract still holds 100 TRUF tokens which is not entitled to anyone and they are stuck. 
## Impact
Actually, there's a way to save those tokens by making a vest to an address and then claiming them. But honestly, I don't think that's what anyone would really want to do intentionally to rescue those tokens. If the owner's plan is to claim those idle TRUF tokens by setting up fake vests and then claiming them, it might reduce the problem's impact or make it not a big deal. However, I don't think this is a good way to get hold of those unused tokens in the contract. Plus, figuring out the new vest gets really tricky with lots of vesting happening all at once. Considering all, I label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L398-L431
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L348-L388
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/TrufVesting.sol#L484-L515
## Tool used

Manual Review

## Recommendation
Make a sweep function that only owner can call. This function can be called when all vests are over.