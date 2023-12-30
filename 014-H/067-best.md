Trendy Cobalt Monkey

high

# BunniHelper.getReservesRatio does not consider the situation that the current tick is out of [tickLower, tickUpper]

## Summary

If `sqrtRatioX96` (corresponding to the current tick) is outside the range of `sqrtRatioAX96` (corresponding to `tickLower`) and `sqrtRatioBX96` (corresponding to `tickUpper`), either `amount0` or `amount1` returned by [[LiquidityAmounts.getAmountsForLiquidity](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L125-L130)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L125-L130) will be 0, depending on which side `sqrtRatioX96` is on. In this case, [[BunniHelper.getReservesRatio](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/UniswapV3/BunniHelper.sol#L47-L55)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/UniswapV3/BunniHelper.sol#L47-L55) will return the wrong radio or generate a divide-by-zero error.

## Vulnerability Detail

```solidity
File: bophades\src\libraries\UniswapV3\BunniHelper.sol
47:     function getReservesRatio(BunniKey memory key_, BunniLens lens_) public view returns (uint256) {
48:         IUniswapV3Pool pool = key_.pool;
49:         uint8 token0Decimals = ERC20(pool.token0()).decimals();
50: 
51:         (uint112 reserve0, uint112 reserve1) = lens_.getReserves(key_);
52:         (uint256 fee0, uint256 fee1) = lens_.getUncollectedFees(key_);
53: 
54:         return (reserve1 + fee1).mulDiv(10 ** token0Decimals, reserve0 + fee0);
55:     }
```

At L51, the internal logic of [[lens_.getReserves(key_)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L58-L65)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L58-L65) is roughly as follows:

1.  get existing liquidity via [[key.pool.positions](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L61-L63)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L61-L63).
2.  get `sqrtRatioX96` via [[key.pool.slot0()](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L116)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L116).
3.  get `sqrtRatioAX96` via [[TickMath.getSqrtRatioAtTick(key.tickLower)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L123)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L123).
4.  get `sqrtRatioBX96` via [[TickMath.getSqrtRatioAtTick(key.tickUpper)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L124)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L124).
5.  get `amount0` and `amount1` via [[LiquidityAmounts.getAmountsForLiquidity](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L125-L130)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniLens.sol#L125-L130). When [[sqrtRatioX96 <= sqrtRatioAX96, amout0 is greater than 0 and amount1 is equal to 0](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/LiquidityAmounts.sol#L129)](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/LiquidityAmounts.sol#L129); and when [[sqrtRatioX96 >= sqrtRatioBX96, amout0 is equal to 0 and amount1 is greater than 0](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/LiquidityAmounts.sol#L134)](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/LiquidityAmounts.sol#L134).

At L52, `lens_.getUncollectedFees` returns uncollected fees. We can collect these fees via [[BunniHub.compound(key)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniHub.sol#L178-L180)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/external/bunni/BunniHub.sol#L178-L180) since anyone can call it. In this way, they are all 0. As the price fluctuates, when `sqrtRatioX96` leaves `[sqrtRatioAX96, sqrtRatioBX96]`, the position will no longer gain fee until the price goes back the range.

This issue is described below in two situations:

### Case 1: reserve0 = 0, reserve1 > 0

1.  If `BunniHub.compound(key)` is not called, then `fee0 > 0, fee1 > 0`. Since the fee is very small relative to the number of tokens provided by liquidity, L54 `(reserve1 + fee1).mulDiv(10 ** token0Decimals, fee0)` will return a very large value.
2.  If `BunniHub.compound(key)` is called, then `fee0 = 0, fee1 = 0`. Then a division-by-zero error will occur at L54 `(reserve1 + fee1).mulDiv(10 ** token0Decimals, 0)`.

### Case 2: reserve0 > 0, reserve1 = 0

1.  If `fee0 > 0, fee1 > 0`, then L54 `(fee1).mulDiv(10 ** token0Decimals, reserve0 + fee0)` will get a very small value.
2.  If `fee0 = 0, fee1 = 0`, then L54 will return 0.

`BunniHelper.getReservesRatio` is used in [[BunniSupply.sol](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/submodules/BunniSupply.sol#L458)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/submodules/BunniSupply.sol#L458) and [[BunniPrice.sol](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L248)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L248). The returned ratio is used to check whether the relative deviation is greater than the maximum. Through the above analysis, tx will revert until `sqrtRatioX96` returns to the original range.

## Impact

If `BunniHelper.getReservesRatio` revert, external callers will revert, including: `OlympusSupply.getMetric`/`getReservesByCategory`/`getSupplyByCategory` and `OlympusPrice.v2._getCurrentPrice`.

- Three functions related to OlympusSupply are frequently used by Appraiser.sol, causing the core function to revert.
- The impact on `OlympusPrice.v2._getCurrentPrice` is smaller, and [[it will only revert when the number of feeds is 1](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L165)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L165). Otherwise, there is just a missing price for an asset with multiple feeds.

## Code Snippet

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/UniswapV3/BunniHelper.sol#L51-L54

## Tool used

Manual Review

## Recommendation

Because there is no deep dive into the entire code base of the protocol, we can only point out problems but cannot provide recommendation.