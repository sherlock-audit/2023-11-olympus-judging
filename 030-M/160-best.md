Flat Infrared Caterpillar

high

# TWAP observation window period is very low allowing the TWAP price to be easily manipulated

## Summary
TWAP observation window period is very low allowing the TWAP price to be easily manipulated

## Vulnerability Detail

In `Oracle.sol`, the contract has used TWAP observation window period as 19 seconds.

```solidity
    uint32 internal constant TWAP_MINIMUM_OBSERVATION_SECONDS = 19;
```
This period size means that the oracle will take a new observation after every single block, which would allow an attacker to easily flood the TWAP oracle and manipulate the price. The contracts will only be deployed on Ethereum mainet which produces blocks for every 12 seconds post Ethereum merge i.e after Proof of Stake(PoS). With the adoption of PoS, oracles are theoretically less secure because a malicious validator knows whether they control the next block.

`TWAP_MINIMUM_OBSERVATION_SECONDS` is used in `getTimeWeightedTick()` to check the provided argument period i.e The period (in seconds) over which to calculate the time-weighted tick. `getTimeWeightedTick()` is further used in `getTWAPRatio()` which returns the ratio of token1 to token0 based on the TWAP.

`getTWAPRatio()` is further extensively used in `BunniPrice.sol`, `UniswapV3Price.sol` and `BunniSupply.sol` various functions which uses TWAP. All these contracts functionality is at risk due to low TWAP period which can be easily manipulated and the functions would return incorrect data.

For more information on TWAP price manipulation, check [this](https://blog.uniswap.org/uniswap-v3-oracles) uniswap-v3 article.

Also, check [this](https://chaoslabs.xyz/posts/chaos-labs-uniswap-v3-twap-market-risk) uniswap-v3 TWAP market risk article.

## Impact
TWAP oracle easily manipulated leading to price manipulation. The functions discussed above are at risk to price manipulation which would also brick the functionality of contracts.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/UniswapV3/Oracle.sol#L20

## Tool used
Manual Review

## Recommendation
Increase the `TWAP_MINIMUM_OBSERVATION_SECONDS` to 1800(30 minutes) which is standard TWAP window period usually considered in DeFi. A higher TWAP period may also be considered.