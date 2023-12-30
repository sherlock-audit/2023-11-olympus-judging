Slow Pebble Stallion

medium

# Oracle#getTimeWeightedTick fails to provide an accurate TWAP due to the exponential nature of tick math

## Summary

Uniswap V3 ticks are exponential in nature but when calculating TWAP they are treated as if they are linear. This leads to an inaccurate average, especially when the TWAP is taken over longer periods of time. This leads to TWAPs that can be more easily manipulated downwards.

## Vulnerability Detail

[Oracle.sol#L84-L93](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/UniswapV3/Oracle.sol#L84-L93)

    try pool.observe(observationWindow) returns (
        int56[] memory tickCumulatives,
        uint160[] memory
    ) {
        timeWeightedTick = (tickCumulatives[1] - tickCumulatives[0]) / int32(period_);
    } catch (bytes memory) {
        // This function will revert if the observation window is longer than the oldest observation in the pool
        // https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Oracle.sol#L226C30-L226C30
        revert UniswapV3OracleHelper_InvalidObservation(pool_, period_);
    }

The above lines of code are used to determine the time weighted tick for determining the TWAP. It simply takes the average tick over the given window. This methodology fails to consider the exponential nature of the ticks which are valued as follows:

    1.0001^tick = price

We can examine the following scenario to see the discrepancy. Assume there are two prices total. The first price is 1 the second is 2. Assume each price is there for 10 seconds. This gives a cumulativeTicks of:

    1 = tick 0
    2 = tick 6931

    0 * 10 + 6931 * 10 = 69310

Next we divide by 20 seconds, our total period:

    69310 / 20 = 3465.5
    
    tick 3465.5 = 1.0001^3465.5 = 1.4146

In this example our true TWAP would be at:

    1 * 10 + 2 * 10 / 20 = 1.5

This causes incorrect valuation of protocol assets. This bias towards the center tick allows the TWAP to be more easily pulled in that direction by malicious parties which can be used to potentially artificially deflate the value of the protocol causing policies to deploy funds incorrectly.

## Impact

TWAPs values can be incorrect and easier to manipulate

## Code Snippet

[Oracle.sol#L67-L106](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/UniswapV3/Oracle.sol#L67-L106)

## Tool used

Manual Review

## Recommendation

For a much better approximation considering utilizing multiple spaced out observation windows. As an example instead of using a single window 0-30 consider using three windows 0-10, 10-20, 20-30 which can be obtained with only 2 window requests 0-10 and 20-30. This gives us a linear piecewise approximation which is much more accurate.