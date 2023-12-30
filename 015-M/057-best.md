Generous Azure Swallow

medium

# The calculation of current price of asset can be wrong.

## Summary
In `OlympusPrice.v2.sol#_getCurrentPrice` function, when `useMovingAverage == true` it uses `movingAveragePrice` for calculation of the current price.
Here, unlike prices from `feeds`, `movingAveragePrice` was determined before, possibly long before.
When `movingAveragePrice` is determined long before the current time, it can be much different from real price. Then, it can confuse the strategy so it can cause wrong calculation of the current price.

## Vulnerability Detail
`OlympusPrice.v2.sol#_getCurrentPrice` function is as follows.
```solidity
    function _getCurrentPrice(address asset_) internal view returns (uint256, uint48) {
        Asset storage asset = _assetData[asset_];

        // Iterate through feeds to get prices to aggregate with strategy
        Component[] memory feeds = abi.decode(asset.feeds, (Component[]));
        uint256 numFeeds = feeds.length;
        uint256[] memory prices = asset.useMovingAverage
            ? new uint256[](numFeeds + 1)
            : new uint256[](numFeeds);
        uint8 _decimals = decimals; // cache in memory to save gas
        for (uint256 i; i < numFeeds; ) {
            (bool success_, bytes memory data_) = address(_getSubmoduleIfInstalled(feeds[i].target))
                .staticcall(
                    abi.encodeWithSelector(feeds[i].selector, asset_, _decimals, feeds[i].params)
                );

            // Store price if successful, otherwise leave as zero
            // Idea is that if you have several price calls and just
            // one fails, it'll DOS the contract with this revert.
            // We handle faulty feeds in the strategy contract.
152         if (success_) prices[i] = abi.decode(data_, (uint256));

            unchecked {
                ++i;
            }
        }

        // If moving average is used in strategy, add to end of prices array
160     if (asset.useMovingAverage) prices[numFeeds] = asset.cumulativeObs / asset.numObservations;

        // If there is only one price, ensure it is not zero and return
        // Otherwise, send to strategy to aggregate
        if (prices.length == 1) {
            if (prices[0] == 0) revert PRICE_PriceZero(asset_);
            return (prices[0], uint48(block.timestamp));
        } else {
            // Get price from strategy
            Component memory strategy = abi.decode(asset.strategy, (Component));
170         (bool success, bytes memory data) = address(_getSubmoduleIfInstalled(strategy.target))
                .staticcall(abi.encodeWithSelector(strategy.selector, prices, strategy.params));

            // Ensure call was successful
            if (!success) revert PRICE_StrategyFailed(asset_, data);

            // Decode asset price
            uint256 price = abi.decode(data, (uint256));

            // Ensure value is not zero
            if (price == 0) revert PRICE_PriceZero(asset_);

            return (price, uint48(block.timestamp));
        }
    }
```
As we can see above, `L152` gets price from `feed` and on L160 when `asset.useMovingAverage == true` it sets `movingAveragePrice`.
And then on `L170` it calculates the current price through `strategy`.
But `movingAveragePrice` is calculated from previously stored values, so it can be much different from real price.
It means that this wrong value can confuse strategy and cause wrong calculation of the current price.
i.e. `SimplePriceFeedStrategy.sol#getAveragePrice` calculates the average as the price and much different value from real one can cause wrong calculation.

## Impact
When `movingAveragePrice` is much different from real price, it can confuse strategy and cause wrong calculation of the current price.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus-web3-master/blob/main/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L160

## Tool used

Manual Review

## Recommendation
`OlympusPrice.v2.sol#_getCurrentPrice` has to be rewritten as follows.
```solidity
    function _getCurrentPrice(address asset_) internal view returns (uint256, uint48) {
        Asset storage asset = _assetData[asset_];

        ...

        // If moving average is used in strategy, add to end of prices array
-       if (asset.useMovingAverage) prices[numFeeds] = asset.cumulativeObs / asset.numObservations;
+       if (asset.useMovingAverage) {
+           if(block.timestamp - asset.lastObservationTime > MAX_MOVING_AVERAGE_DELAY){
+               revert(...);
+           }
+           prices[numFeeds] = asset.cumulativeObs / asset.numObservations;
+       }

        ...
    }
```
Here, `MAX_MOVING_AVERAGE_DELAY` is a `threshold` which the system set.