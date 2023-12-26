Raspy Hemp Fly

high

# BunniPrice.getBunniTokenPrice doesn't include fees

## Summary
BunniPrice.getBunniTokenPrice doesn't include fees into calculation, which means that price will be smaller.
## Vulnerability Detail
`BunniPrice.getBunniTokenPrice` function should return usd cost of `bunniToken_`.
After initial checks this function actually do 2 things:
- validate deviation
- calculates total cost of token

Let's check how it's done to calculate total cost of token. It's done [using `_getTotalValue` function](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L163).

So we get token reserves [using `_getBunniReserves` function](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L220-L224) and the convert them to usd value and return the sum. `_getBunniReserves` in it's turn just [fetches amount of token0 and token1](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L198) that can be received in case if you swap all bunni token liquidity right now in the pool. 

The problem is that this function doesn't include fees that are earned by the position. As result total cost of bunni token is smaller than in reality. The difference depends on the amount of fees there were earned and not returned back.

Also when you do burn in the uniswap v3, then amount of token0/token1 is not send back to the caller, but they are added to the position and can be collected later. So in case if `BunniPrice.getBunniTokenPrice` will be called for the token, where some liquidity is burnt but not collected, then difference in real token price can be huge.
## Impact
Price for the bunni token can be calculated in wrong way.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Include collected amounts and additionally earned fees to the position reserves.