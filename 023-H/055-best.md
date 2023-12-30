Generous Azure Swallow

high

# OlympusPrice.v2.sol#storePrice: The moving average prices are used recursively for the calculation of the moving average price.

## Summary
The moving average prices should be calculated by only oracle feed prices.
But now, they are calculated by not only oracle feed prices but also moving average price recursively.

That is, the `storePrice` function uses the current price obtained from the `_getCurrentPrice` function to update the moving average price.
However, in the case of `asset.useMovingAverage = true`, the `_getCurrentPrice` function computes the current price using the moving average price.

Thus, the moving average prices are used recursively to calculate moving average price, so the current prices will be obtained incorrectly.

## Vulnerability Detail
`OlympusPrice.v2.sol#storePrice` function is the following.
```solidity
    function storePrice(address asset_) public override permissioned {
        Asset storage asset = _assetData[asset_];

        // Check if asset is approved
        if (!asset.approved) revert PRICE_AssetNotApproved(asset_);

        // Get the current price for the asset
319:    (uint256 price, uint48 currentTime) = _getCurrentPrice(asset_);

        // Store the data in the obs index
        uint256 oldestPrice = asset.obs[asset.nextObsIndex];
        asset.obs[asset.nextObsIndex] = price;

        // Update the last observation time and increment the next index
        asset.lastObservationTime = currentTime;
        asset.nextObsIndex = (asset.nextObsIndex + 1) % asset.numObservations;

        // Update the cumulative observation, if storing the moving average
        if (asset.storeMovingAverage)
331:        asset.cumulativeObs = asset.cumulativeObs + price - oldestPrice;

        // Emit event
        emit PriceStored(asset_, price, currentTime);
    }
```
`L319` obtain the current price for the asset by calling the `_getCurrentPrice` function and use it to update `asset.cumulativeObs` in `L331`.
The `_getCurrentPrice` function is the following.
```solidity
    function _getCurrentPrice(address asset_) internal view returns (uint256, uint48) {
        Asset storage asset = _assetData[asset_];

        // Iterate through feeds to get prices to aggregate with strategy
        Component[] memory feeds = abi.decode(asset.feeds, (Component[]));
        uint256 numFeeds = feeds.length;
138:    uint256[] memory prices = asset.useMovingAverage
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
            if (success_) prices[i] = abi.decode(data_, (uint256));

            unchecked {
                ++i;
            }
        }

        // If moving average is used in strategy, add to end of prices array
160:    if (asset.useMovingAverage) prices[numFeeds] = asset.cumulativeObs / asset.numObservations;

        // If there is only one price, ensure it is not zero and return
        // Otherwise, send to strategy to aggregate
        if (prices.length == 1) {
            if (prices[0] == 0) revert PRICE_PriceZero(asset_);
            return (prices[0], uint48(block.timestamp));
        } else {
            // Get price from strategy
            Component memory strategy = abi.decode(asset.strategy, (Component));
            (bool success, bytes memory data) = address(_getSubmoduleIfInstalled(strategy.target))
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
As can be seen, when `asset.useMovingAverage = true`, the `_getCurrentPrice` calculates the current price `price` using the moving average price obtained by `asset.cumulativeObs / asset.numObservations` in `L160`.

So the `price` value in `L331` is obtained from not only oracle feed prices but also moving average price. 
Then, `storePrice` calculates the cumulative observations `asset.cumulativeObs = asset.cumulativeObs + price - oldestPrice` using the `price` which is obtained incorrectly above.

Thus, the moving average prices are used recursively for the calculation of the moving average price.

## Impact
Now the moving average prices are used recursively for the calculation of the moving average price.
Then, the moving average prices become more smoothed than the intention of the administrator.
That is, even when the actual price fluctuations are large, the price fluctuations of `_getCurrentPrice` function will become too small.

Moreover, even though all of the oracle price feeds fails, the moving averge prices will be calculated only by moving average prices.

Thus the current prices will become incorrect.
If `_getCurrentPrice` function value is miscalculated, it will cause fatal damage to the protocol.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus-web3-master/blob/main/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L312-L335
https://github.com/sherlock-audit/2023-11-olympus-web3-master/blob/main/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L132-L184

## Tool used
Manual Review

## Recommendation
When updating the current price and cumulative observations in the `storePrice` function, it should use the oracle price feeds and not include the moving average prices.
So, instead of using the `asset.useMovingAverage` state variable  in the `_getCurrentPrice` function, we can add a `useMovingAverage` parameter as the following.
```solidity
>>  function _getCurrentPrice(address asset_, bool useMovingAverage) internal view returns (uint256, uint48) {
        Asset storage asset = _assetData[asset_];

        // Iterate through feeds to get prices to aggregate with strategy
        Component[] memory feeds = abi.decode(asset.feeds, (Component[]));
        uint256 numFeeds = feeds.length;
>>      uint256[] memory prices = useMovingAverage
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
            if (success_) prices[i] = abi.decode(data_, (uint256));

            unchecked {
                ++i;
            }
        }

        // If moving average is used in strategy, add to end of prices array
>>      if (useMovingAverage) prices[numFeeds] = asset.cumulativeObs / asset.numObservations;

        // If there is only one price, ensure it is not zero and return
        // Otherwise, send to strategy to aggregate
        if (prices.length == 1) {
            if (prices[0] == 0) revert PRICE_PriceZero(asset_);
            return (prices[0], uint48(block.timestamp));
        } else {
            // Get price from strategy
            Component memory strategy = abi.decode(asset.strategy, (Component));
            (bool success, bytes memory data) = address(_getSubmoduleIfInstalled(strategy.target))
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
Then we should set `useMovingAverage = false` to call `_getCurrentPrice` function only in the `storePrice` function.
In other cases, we should set `useMovingAverage = asset.useMovingAverage` to call `_getCurrentPrice` function.