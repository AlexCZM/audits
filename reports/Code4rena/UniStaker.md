# L-01 Users who claim entire protocol fee accrued in the pool will have their tx reverted

For gas savings, UniswapPool does [not clear](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L857) (and [here](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L862)) the accumulated protocolFees. 
Users who calls `claimFees` to claim the entire pool's protocol fee will have their tx reverted due to [this](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/V3FactoryOwner.sol#L193) check.
```solidity
    // Protect the caller from receiving less than requested. See `collectProtocol` for context.
    if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
```
Consider ignoring the 1 wei substraction made in UniswapPool and either
- request 1 wei less when calling `_pool.collectProtocol`
```solidity 
_pool.collectProtocol(_recipient, _amount0Requested -1, _amount1Requested -1);
```
- or subtract 1 wei when checking the requested amount and received amount
```solidity
    if (_amount0 < _amount0Requested - 1 || _amount1 < _amount1Requested - 1) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
```