Plain Chiffon Peacock

medium

# Analysis of Potential Vulnerability in Uniswap V3 Pool Reserve Ratio Calculation

## Summary

A potential vulnerability was identified in the code snippet responsible for calculating the reserves ratio in a Uniswap V3 pool. This issue arises due to the incorrect handling of differences in token decimals, which can lead to inaccurate ratio calculations.

## Vulnerability Detail

The original code snippet assumes that both tokens in the pool (`token0` and `token1`) have the same number of decimal places. ERC20 tokens, however, can have varying decimal places, and this discrepancy can result in an incorrect calculation of the reserves ratio. This vulnerability can potentially affect valuations or decisions based on this inaccurate ratio.

## Impact

If not addressed, this vulnerability could lead to significant financial inaccuracies or manipulations, impacting users' trust and the integrity of transactions within the Uniswap V3 pool.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus-BradMoonUESTC/blob/main/bophades/src/libraries/UniswapV3/BunniHelper.sol#L47-L55
Original Code:
```solidity
function getReservesRatio(BunniKey memory key_, BunniLens lens_) public view returns (uint256) {
 IUniswapV3Pool pool = key_.pool;
 uint8 token0Decimals = ERC20(pool.token0()).decimals();

 (uint112 reserve0, uint112 reserve1) = lens_.getReserves(key_);
 (uint256 fee0, uint256 fee1) = lens_.getUncollectedFees(key_);

 return (reserve1 + fee1).mulDiv(10 ** token0Decimals, reserve0 + fee0);
}
```

## Tool used

Metatrust's Metascan Experimental Engine For Logic bugs Automated detection (metatrust.io). This issue is a test for proving our capability in identifying and addressing logical vulnerabilities in smart contract code.

## Recommendation

To mitigate this vulnerability, the code should be revised to correctly account for the differences in decimals between `token0` and `token1`. This adjustment ensures accurate calculation of the reserves ratio, irrespective of the decimal places of the tokens involved. Further testing and auditing are recommended to ensure the vulnerability is fully addressed and to maintain the integrity of financial transactions within the Uniswap V3 pool.