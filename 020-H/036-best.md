Raspy Hemp Fly

high

# BunniPrice.getBunniTokenPrice deviation check is not correct

## Summary
BunniPrice.getBunniTokenPrice deviation check is not correct, because reserves ratio from BunniHelper.getReservesRatio doesn't show token0 price, but shows ratio of the position.
## Vulnerability Detail
`BunniPrice.getBunniTokenPrice` function should return usd cost of `bunniToken_`.
After initial checks this function actually do 2 things:
- validate deviation
- calculates total cost of token

Let's check how it's done to check deviation.
To validate deviation function calls [`_validateReserves` function](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L155-L160). This function fetches reserves ratio using `BunniHelper.getReservesRatio` and `UniswapV3OracleHelper.getTWAPRatio` and [then compare them to check deviation](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L255-L265).

`UniswapV3OracleHelper.getTWAPRatio` function [will get twap tick for the observation window](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/libraries/UniswapV3/Oracle.sol#L147) and then[ will return price of unit of token0 in term of token1](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/libraries/UniswapV3/Oracle.sol#L151-L156). So it actually return twap price of token0.

`BunniHelper.getReservesRatio` function will get current reserves of bunni token in uniswap v3 pool + earned fees and [then will calculate price of token0](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/libraries/UniswapV3/BunniHelper.sol#L51-L54).

Now we need to understand what is current reserve of bunni token. It is fetched using `lens_.getReserves` function.
So we get position from uniswapv3 and [receive liquidity amount that bunni token holds](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/external/bunni/BunniLens.sol#L61-L63). And after that we will calculate how many amount of token0 and token1 we can get in the current moment with current price inside v3 pool. This is done using `_getReserves` function. As you can see it [fetches current price from the pool](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/external/bunni/BunniLens.sol#L116) and then calculates amount that user can get if he will burn his liquidity.

So what we have here. 
- `BunniHelper.getReservesRatio` can simply revert in case if position is fully in token1 and didn't earn any token0 fees. It will revert with division by 0 then.
- `BunniHelper.getReservesRatio` doesn't provide current price of base token.
- It is not correct to compare `BunniHelper.getReservesRatio` with `UniswapV3OracleHelper.getTWAPRatio` because of reason above.
## Impact
Deviation check is not correct for the bunni tokens, which can lead to unexpected behaviour.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
You can't check deviation in such way.