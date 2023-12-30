Raspy Hemp Fly

high

# UniswapV3OracleHelper.getTWAPRatio will show incorrect price for negative ticks

## Summary
UniswapV3OracleHelper.getTWAPRatio will show incorrect price for negative ticks, because getTimeWeightedTick function doesn't round up for negative ticks
## Vulnerability Detail
`UniswapV3OracleHelper.getTWAPRatio` function is used by protocol [to validate price deviation in the `BunniPrice`](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L249-L252) and also [by `UniswapV3Price.getTokenTWAP` function](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/modules/PRICE/submodules/feeds/UniswapV3Price.sol#L154-L160).

Function itself calls [`getTimeWeightedTick` function](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/libraries/UniswapV3/Oracle.sol#L147) to get twap price tick using uniswap oracle. `getTimeWeightedTick` [uses `pool.observe` to get `tickCumulatives` array](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/libraries/UniswapV3/Oracle.sol#L85C28-L85C43) which is then used to calculate `timeWeightedTick`. 

The problem is that in case if `tickCumulatives[1] - tickCumulatives[0]` is negative, then `timeWeightedTick` should be rounded down [as it's done in the uniswap library](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/OracleLibrary.sol#L36).

As result, in case if `tickCumulatives[1] - tickCumulatives[0]` is negative and `(tickCumulatives[1] - tickCumulatives[0]) % secondsAgo != 0`, then returned `timeWeightedTick` will be bigger then it should be, which opens possibility for some price manipulations and arbitrage opportunities.
## Impact
In case if `tickCumulatives[1] - tickCumulatives[0]` is negative and `((tickCumulatives[1] - tickCumulatives[0]) % secondsAgo != 0`, then returned `timeWeightedTick` will be bigger then it should be.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Add this line.
`if (tickCumulatives[1] - tickCumulatives[0] < 0 && (tickCumulatives[1] - tickCumulatives[0]) % secondsAgo != 0) timeWeightedTick --;`