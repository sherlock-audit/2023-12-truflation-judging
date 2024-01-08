Noisy Mercurial Hornet

medium

# function `_contractFallback()` is always revert because IERC677Receiver has wrong function signature for onTokenTransfer()

## Summary
because IERC677Receiver has wrong function signature for `onTokenTransfer()` so the `transferAndCall()` will always revert if it calls `onTokenTransfer()` of the target contract(which correctly implement the ERC)
  
## Vulnerability Detail
This is `transferAndCall()` code:
```javascript
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
as you can see it uses `IERC677Receiver` to call `onTokenTransfer()` function of the `to` address.  This is `IERC677Receiver`:
```javascript
interface IERC677Receiver {
    function onTokenTransfer(address _sender, uint256 _value, bytes calldata _data) external;
}
```
the issue is that function signature of the `onTokenTransfer()` is not according to the ERC677 and it doesn't define the return value of the function and if a target contract implements ERC677 correctly then it's gonna return the `bool` variable and because function signature is wrong in ERC677Token so solidity would revert that call and the whole transaction would revert. as result ERC677Token won't gonna work according to the EIP. 

## Impact
token ERC677Token would always revert in `transferAndCall()` function if the `to` is contract and that contract implements EIP correctly.

## Code Snippet
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/token/ERC677Token.sol#L26-L29
https://github.com/sherlock-audit/2023-12-truflation/blob/37ddbb69e0c7fb6510f1ec99162fd9172ec44733/truflation-contracts/src/interfaces/IERC677Receiver.sol#L4-L6

## Tool used
Manual Review

## Recommendation
change IERC677Receiver to be like this:
```javascript
interface IERC677Receiver {
    function onTokenTransfer(address _sender, uint256 _value, bytes calldata _data) external returns(bool);
}
```
