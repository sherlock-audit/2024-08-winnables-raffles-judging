Shaggy Ultraviolet Parakeet

Medium

# Incorrect casting in `BaseCCIPContract::_packCCIPContract()`

### Summary

The style of casting the `chainSelector` is incorrect hence it will evaluate to zero and not form part of the bytes returned.

### Root Cause

In [`BaseCCIPContract::_packCCIPContract()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/BaseCCIPContract.sol#L40), the [`chainSelector()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/BaseCCIPContract.sol#L43) was casted to uint256 after the bitwise operation which is not supposed to be so

### Internal pre-conditions

whenever the function params are set it will occur

### External pre-conditions

There are no preconditions, anytime it is called by another function it will occur.

### Attack Path

This is an internal casting error so whenever another function calls `BaseCCIPContract::_packCCIPContract()` it will occur.

### Impact

The `_packCCIPContract()` returns an incorrect value since the `chainSelector` will not be part of the encoded bytes. 

### PoC

Code with the casting error
```solidity
    function _packCCIPContract(address contractAddress, uint64 chainSelector) internal pure returns(bytes32) { return bytes32(
            uint256(uint160(contractAddress)) | uint256(chainSelector << 160));
    }
```

these are the test results after supplying it with random values
```solidity
[PASS] test() (gas: 5678)
Traces:
  [5678] returnTest::test_packCCIPContract()
    ├─ [427] BitwiseOperations::_packCCIPContract(returnTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 43114 [4.311e4]) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000007fa9385be102ac3eac297483dd6233d62b3e1496
    └─ ← [Stop] 

Suite result: ok. 1 passed;
```
we can observe that the test passed but only the address has been encoded

code with the right casting style
```solidity
    function CCIP(address contractAddress, uint64 chainSelector) public pure returns(bytes32) {
        return bytes32(
            uint256(uint160(contractAddress)) |
            (uint256(chainSelector) << 160)
        );
    }
```

these are the test results
```solidity
[PASS] testCCIP() (gas: 5705)
Traces:
  [5705] returnTest::test_packCCIPContract()
    ├─ [454] BitwiseOperations::_packCCIPContract(returnTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 43114 [4.311e4]) [staticcall]
    │   └─ ← [Return] 0x00000000000000000000a86a7fa9385be102ac3eac297483dd6233d62b3e1496
    └─ ← [Stop] 

Suite result: ok. 1 passed;
```
from this test result we can observe that the chainSelector has been encoded to the bytes32 returned and not only the address

### Mitigation

Instead of casting after the bitwise operation, cast before the operation
```diff
    function _packCCIPContract(address contractAddress, uint64 chainSelector) internal pure returns(bytes32) {
        return bytes32(
-            uint256(uint160(contractAddress)) | uint256(chainSelector << 160));
+            uint256(uint160(contractAddress)) | uint256(chainSelector) << 160);
    }

```