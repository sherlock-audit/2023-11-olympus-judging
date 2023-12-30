Generous Azure Swallow

medium

# Price can be miscalculated.

## Summary
In `SimplePriceFeedStrategy.sol#getMedianPrice` function, when the length of `nonZeroPrices` is 2 and they are deviated it returns first non-zero value, not median value.

## Vulnerability Detail
`SimplePriceFeedStrategy.sol#getMedianPriceIfDeviation` is as follows.
```solidity
    function getMedianPriceIfDeviation(
        uint256[] memory prices_,
        bytes memory params_
    ) public pure returns (uint256) {
        // Misconfiguration
        if (prices_.length < 3) revert SimpleStrategy_PriceCountInvalid(prices_.length, 3);

237     uint256[] memory nonZeroPrices = _getNonZeroArray(prices_);

        // Return 0 if all prices are 0
        if (nonZeroPrices.length == 0) return 0;

        // Cache first non-zero price since the array is sorted in place
        uint256 firstNonZeroPrice = nonZeroPrices[0];

        // If there are not enough non-zero prices to calculate a median, return the first non-zero price
246     if (nonZeroPrices.length < 3) return firstNonZeroPrice;

        uint256[] memory sortedPrices = nonZeroPrices.sort();

        // Get the average and median and abort if there's a problem
        // The following two values are guaranteed to not be 0 since sortedPrices only contains non-zero values and has a length of 3+
        uint256 averagePrice = _getAveragePrice(sortedPrices);
253     uint256 medianPrice = _getMedianPrice(sortedPrices);

        if (params_.length != DEVIATION_PARAMS_LENGTH) revert SimpleStrategy_ParamsInvalid(params_);
        uint256 deviationBps = abi.decode(params_, (uint256));
        if (deviationBps <= DEVIATION_MIN || deviationBps >= DEVIATION_MAX)
            revert SimpleStrategy_ParamsInvalid(params_);

        // Check the deviation of the minimum from the average
        uint256 minPrice = sortedPrices[0];
262     if (((averagePrice - minPrice) * 10000) / averagePrice > deviationBps) return medianPrice;

        // Check the deviation of the maximum from the average
        uint256 maxPrice = sortedPrices[sortedPrices.length - 1];
266     if (((maxPrice - averagePrice) * 10000) / averagePrice > deviationBps) return medianPrice;

        // Otherwise, return the first non-zero value
        return firstNonZeroPrice;
    }
```
As you can see above, on L237 it gets the list of non-zero prices. If the length of this list is smaller than 3, it assumes that a median price cannot be calculated and returns first non-zero price.
This is wrong.
If the number of non-zero prices is 2 and they are deviated, it has to return median value.
The `_getMedianPrice` function called on L253 is as follows.
```solidity
    function _getMedianPrice(uint256[] memory prices_) internal pure returns (uint256) {
        uint256 pricesLen = prices_.length;

        // If there are an even number of prices, return the average of the two middle prices
        if (pricesLen % 2 == 0) {
            uint256 middlePrice1 = prices_[pricesLen / 2 - 1];
            uint256 middlePrice2 = prices_[pricesLen / 2];
            return (middlePrice1 + middlePrice2) / 2;
        }

        // Otherwise return the median price
        // Don't need to subtract 1 from pricesLen to get midpoint index
        // since integer division will round down
        return prices_[pricesLen / 2];
    }
```
As you can see, the median value can be calculated from two values.
This problem exists at `getMedianPrice` function as well.
```solidity
    function getMedianPrice(uint256[] memory prices_, bytes memory) public pure returns (uint256) {
        // Misconfiguration
        if (prices_.length < 3) revert SimpleStrategy_PriceCountInvalid(prices_.length, 3);

        uint256[] memory nonZeroPrices = _getNonZeroArray(prices_);

        uint256 nonZeroPricesLen = nonZeroPrices.length;
        // Can only calculate a median if there are 3+ non-zero prices
        if (nonZeroPricesLen == 0) return 0;
        if (nonZeroPricesLen < 3) return nonZeroPrices[0];

        // Sort the prices
        uint256[] memory sortedPrices = nonZeroPrices.sort();

        return _getMedianPrice(sortedPrices);
    }
```

## Impact
When the length of `nonZeroPrices` is 2 and they are deviated, it returns first non-zero value, not median value. It causes wrong calculation error.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus-web3-master/blob/main/bophades/src/modules/PRICE/submodules/strategies/SimplePriceFeedStrategy.sol#L246

## Tool used

Manual Review

## Recommendation
First, `SimplePriceFeedStrategy.sol#getMedianPriceIfDeviation` function has to be rewritten as follows.
```solidity
    function getMedianPriceIfDeviation(
        uint256[] memory prices_,
        bytes memory params_
    ) public pure returns (uint256) {
        // Misconfiguration
        if (prices_.length < 3) revert SimpleStrategy_PriceCountInvalid(prices_.length, 3);

        uint256[] memory nonZeroPrices = _getNonZeroArray(prices_);

        // Return 0 if all prices are 0
        if (nonZeroPrices.length == 0) return 0;

        // Cache first non-zero price since the array is sorted in place
        uint256 firstNonZeroPrice = nonZeroPrices[0];

        // If there are not enough non-zero prices to calculate a median, return the first non-zero price
-       if (nonZeroPrices.length < 3) return firstNonZeroPrice;
+       if (nonZeroPrices.length < 2) return firstNonZeroPrice;

        ...
    }
```
Second, `SimplePriceFeedStrategy.sol#getMedianPrice` has to be modified as following.
```solidity
    function getMedianPrice(uint256[] memory prices_, bytes memory) public pure returns (uint256) {
        // Misconfiguration
        if (prices_.length < 3) revert SimpleStrategy_PriceCountInvalid(prices_.length, 3);

        uint256[] memory nonZeroPrices = _getNonZeroArray(prices_);

        uint256 nonZeroPricesLen = nonZeroPrices.length;
        // Can only calculate a median if there are 3+ non-zero prices
        if (nonZeroPricesLen == 0) return 0;
-       if (nonZeroPricesLen < 3) return nonZeroPrices[0];
+       if (nonZeroPricesLen < 2) return nonZeroPrices[0];

        // Sort the prices
        uint256[] memory sortedPrices = nonZeroPrices.sort();

        return _getMedianPrice(sortedPrices);
    }
```