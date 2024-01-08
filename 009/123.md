Big Carbon Fish

medium

# Potential reentrancy risk in ERC677 Token

## Summary
The `TruflationToken` is inherited from `ERC677Token` , and also have `transferAndCall` function .
If token are transfered to a malicious contract address, and this contract rewrite a malicious `onTokenTransfer` callback can cause a reentrancy attack .

## Vulnerability Detail

After The receiver received the Turf token transfer , and then will execute receiver's `onTokenTransfer` method .

```solidity
  function _contractFallback(address _to, uint256 _value, bytes calldata _data) private {
        IERC677Receiver receiver = IERC677Receiver(_to);
        receiver.onTokenTransfer(msg.sender, _value, _data);
    }
```

But if I rewrite receiver's `onTokenTransfer` function and the call to token transfer's sender, the Turf token call be transfered to malicious contract address again .


## Impact
Potential reentrancy attack will cause the sender lost his all Turf token .

## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/main/truflation-contracts/src/token/ERC677Token.sol#L16-L29

```solidity
 function transferAndCall(address _to, uint256 _value, bytes calldata _data) public returns (bool success) {
        super.transfer(_to, _value);
        emit Transfer(msg.sender, _to, _value, _data);
        if (Address.isContract(_to)) {
            _contractFallback(_to, _value, _data);
        }
        return true;
    }

    // PRIVATE
    function _contractFallback(address _to, uint256 _value, bytes calldata _data) private {
        IERC677Receiver receiver = IERC677Receiver(_to);
        receiver.onTokenTransfer(msg.sender, _value, _data);
    }
```

## Tool used

Manual Review

## Recommendation
I check the codebase , and there is no use of TruflationToken:transferAndCall
So cancel the inherition from `ERC677Token`, instead of using a standard ERC20 token .
