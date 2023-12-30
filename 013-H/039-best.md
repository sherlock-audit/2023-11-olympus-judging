Polite Scarlet Antelope

high

# Allowing inconsistent time slices, the moving average can misrepresent the mean price

## Summary
On creation `OlympusPricev2` is provided an `observationFrequency`, the expected frequency at which prices are stored for moving average (MA).

`OlympusPricev2.storePrice()` calculates the latest asset price and updates the moving average, but lacks any logic to ensure the timeliness of updates, or consistency in the time between updates.

The MA in `OlympusPricev2` is a simple moving average (SMA), where the points of time-series data taken for the SMA can have inconsistent time spacing,  meaning the SMA will contain a degree of error in the following of the line graph of all the points, due to even weighting of points irrespective of intervals.
(The average produced may not be accurately following movements in the underlying data).


## Vulnerability Detail
The expectation of how much time will be between moving average updates if given as `observationFrequency_` in the [OlympusPricev2 constructor](https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L26-L27)
```solidity
    /// @param observationFrequency_    Frequency at which prices are stored for moving average
    constructor(Kernel kernel_, uint8 decimals_, uint32 observationFrequency_) Module(kernel_) {
```

### Updating the MA
When the update of the MA happens, there are no guards or checks enforcing the `currentTime` is `observationFrequency` away from the `lastObservationTime` in [OlympusPricev2.storePrice](https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L312-L335)
```solidity
    function storePrice(address asset_) public override permissioned {
    Asset storage asset = _assetData[asset_];

    // Check if asset is approved
    if (!asset.approved) revert PRICE_AssetNotApproved(asset_);

    // Get the current price for the asset
    (uint256 price, uint48 currentTime) = _getCurrentPrice(asset_);

    // Store the data in the obs index
    uint256 oldestPrice = asset.obs[asset.nextObsIndex];
    asset.obs[asset.nextObsIndex] = price;

    // Update the last observation time and increment the next index
    asset.lastObservationTime = currentTime;
    asset.nextObsIndex = (asset.nextObsIndex + 1) % asset.numObservations;

    // Update the cumulative observation, if storing the moving average
    if (asset.storeMovingAverage)
        asset.cumulativeObs = asset.cumulativeObs + price - oldestPrice;

    // Emit event
    emit PriceStored(asset_, price, currentTime);
}
```
The `currentTime` is a return value, but that is simply returning the current `block.timestamp` in [__getCurrentPrice](https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L182-L182)
```solidity
            return (price, uint48(block.timestamp));
```
With `storePrice` having no restriction on how short or long after the `currentTime` could be away from the `lastObservationTime`, the time between the data points in the SMA are not guaranteed to be consistent.

When the MA is calculated, each data point is given equal weighting by [OlympusPricev2._getMovingAveragePrice]()
```solidity
        // Calculate moving average
        uint256 movingAverage = asset.cumulativeObs / asset.numObservations;
```

Providing an equal weighting to each value, irrespective of the time interval between them, introduces the potential for distortion in the MA.

As `OlympusPricev2.storePrice()` exists as a part of a larger system, lets looks how further into upstream interactions.


#### Heartbeat
The call to `OlympusPricev2.storePrice()` occurs in the RBS Policy `Heart.beat()` (out of scope for the audit, but included to provide context), that itself is called by an incentivized keeper, with a target interval of 8 hours.

```solidity
    function beat() external nonReentrant {
        if (!active) revert Heart_BeatStopped();
        uint48 currentTime = uint48(block.timestamp);
        if (currentTime < lastBeat + frequency()) revert Heart_OutOfCycle();

        // Update the OHM/RESERVE moving average by store each of their prices on the PRICE module
        PRICE.storePrice(address(operator.ohm()));
        PRICE.storePrice(address(operator.reserve()));
```
`(currentTime < lastBeat + frequency())` checks that at least `observationFrequency` time has passed since the last `beat`.
`OlympusPricev2.storePrice()` will not be called at intervals of less than observationFrequency`, but there is nothing preventing longer time intervals.

#### Incentivized keeper
An external actor who performs operations usually driven by financial incentives, where timeliness of actions rely on the incentives providing a return on performing the action.
Incentivizes are subject to free market conditions, for `Heart.beat()` the variables being the valuation of the incentives and the cost of gas.


### The effect of time intervals
When selecting data points from a time-series data set and no accounting for different time intervals between them is made, there can be unintended effects.

For a quick example to shows a small average during a change from a time of sidewards moving price into a strong upward trending price, lets assume the following properties:
- SMA is initialized flat (all points are equal, the starting price)
- SMA of 5 data points
- Starting price is one full unit of the asset
- Price has linear increase of 10% every 24 hours
- The test will run for 5 updates to the SMA

Two tests, each with two 10 hour intervals and three 8 hour intervals, the different being one has the longer intervals in the beginning, the other at the end.

A quick test logging the MA after each update produces these results:
```text
[PASS] test_linear_price_ma_closer_early_points() (gas: 793044)
Logs:
  10066666666000000000
  10199999998000000000
  10399999998000000000
  10683333330000000000
  11049999996000000000

[PASS] test_linear_price_ma_closer_later_points() (gas: 792999)
Logs:
  10083333332000000000
  10249999998000000000
  10483333330000000000
  10783333330000000000
  11149999996000000000
```
With a result that is likely to surprise nobody, the test with the two 10 hours intervals at the beginning score a larger MA (~0.9% larger MA, over an underlying price increase of ~18%).

Irregular time intervals can have a real impact during a strong, consistent trend.


#### Proof of Concept - Sample Data
To generate the sample data, add the below Solidity to `PRICE.v2.t.sol` and run `forge test --match-test test_linear_price_ma -vv`

```solidity
    uint private constant START_PRICE = 10e8;       // Alpha USD feed is 8 decimals
    uint private constant PARSED_PRICE = 10e18;     // Oracle after parsed by Olympus code is 18 decimals

    /// MA with longer time between the two earliest data points
    function test_linear_price_ma_closer_later_points() external {
        _addAlphaSingleFeedStoreMA();
        uint startTime = block.timestamp;
        uint80 roundId = 1;

        storeLinearPriceIncrease( startTime, startTime + 10 hours, roundId++);
        storeLinearPriceIncrease( startTime, startTime + 20 hours, roundId++);
        storeLinearPriceIncrease( startTime, startTime + 28 hours, roundId++);
        storeLinearPriceIncrease( startTime, startTime + 36 hours, roundId++);
        storeLinearPriceIncrease( startTime, startTime + 44 hours, roundId++);
    }

    /// MA with longer time between the two latest data points
    function test_linear_price_ma_closer_early_points() external {
        _addAlphaSingleFeedStoreMA();
        uint startTime = block.timestamp;
        uint80 roundId = 1;

        storeLinearPriceIncrease( startTime, startTime + 8 hours, roundId++);
        storeLinearPriceIncrease( startTime, startTime + 16 hours, roundId++);
        storeLinearPriceIncrease( startTime, startTime + 24 hours, roundId++);
        storeLinearPriceIncrease( startTime, startTime + 34 hours, roundId++);
        storeLinearPriceIncrease( startTime, startTime + 44 hours, roundId++);
    }

    /// Linear increase of price by 10% every 24 hours
    function storeLinearPriceIncrease( uint startTime, uint timeNow, uint80 roundId) private {
        vm.warp(timeNow);

        uint tenPercentBPS = 1_000;
        uint oneHundredPercentBPS = 10_000;
        uint timePassed = timeNow-startTime;
        uint priceIncrease = (START_PRICE * timePassed * tenPercentBPS) / (24 hours * oneHundredPercentBPS);
        uint priceNow = START_PRICE + priceIncrease;

        alphaUsdPriceFeed.setRoundId(roundId);
        alphaUsdPriceFeed.setAnsweredInRound(roundId);
        alphaUsdPriceFeed.setTimestamp(timeNow);
        alphaUsdPriceFeed.setLatestAnswer(int(priceNow));

        vm.prank(writer);
        price.storePrice(address(alpha));

        log_ma();
    }

    function log_ma() private {
        (uint256 ma,) = price.getPrice(address(alpha), PRICEv2.Variant.MOVINGAVERAGE);
        emit log_uint(ma);
    }

    // Add the Alpha with a single feed and moving average stored, but not used in price calculations
    function _addAlphaSingleFeedStoreMA() private {
        ChainlinkPriceFeeds.OneFeedParams memory alphaParams = ChainlinkPriceFeeds.OneFeedParams(alphaUsdPriceFeed, uint48(24 hours));
        PRICEv2.Component[] memory feeds = new PRICEv2.Component[](1);
        feeds[0] = PRICEv2.Component(toSubKeycode("PRICE.CHAINLINK"), ChainlinkPriceFeeds.getOneFeedPrice.selector, abi.encode(alphaParams));
        vm.prank(writer);
        price.addAsset(address(alpha), true, false, uint32(40 hours), uint48(block.timestamp), _makeStaticObservations(uint256(5)), PRICEv2.Component(toSubKeycode(bytes20(0)), bytes4(0), abi.encode(0)), feeds);
    }

    function _makeStaticObservations(uint observationCount) private returns (uint256[] memory){
        uint[] memory observations = new uint[](observationCount);
        for (uint i = observationCount; i > 0; --i) {
            observations[i - 1] = PARSED_PRICE;
        }
        return observations;
    }
```

## Impact
These three areas impacted by an inaccurate MA: price retrieval, MA timespan and MA retrieval.

### Price retrieval
The MA can be included in the current price calculation by the strategies in [SimplePriceFeedStrategy](https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/submodules/strategies/SimplePriceFeedStrategy.sol#L123), 
with the degree of impact varying by strategy and whether the feeds are live (e.g. Tests have [OHM and RSV with multiple feeds](https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/test/modules/PRICE.v2/PRICE.v2.t.sol#L151-L152)), 
which seriously reduce the likelihood of every encountering them i.e. as multiple simultaneous feed failures are required for the MA to constitute the price or a large component of it. 

### Moving average timespan
When adding an asset the param `movingAverageDuration_` is used in combination with `observationFrequency` to calculate the number of data points to use in the SMA.
Due to the incentivized keeper update mechanism, the time between observations is likely to differ from `observationFrequency`, where the timespan of the entire MA could be materially different to the duration given when adding the asset.
If the timespan was chosen due to risk modelling (rather than an implicit calculation for the number of SMA points), this would be unfortunate, as the models would not be match the actual behaviour. 

### Moving average retrieval
The more impactful use of the `OlympusPricev2` MA is with the RBS policy, accessed in [Operator.targetPrice()](https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/policies/RBS/Operator.sol#L856) as part of updating the target price for the `RANGE` module.
[Range Bound Stability](https://docs.olympusdao.finance/main/overview/range-bound) being the mechanism that stabilizes the price of OHM, by using the MA as the anchor point to decide upper and lower bounds, and consequent actions to stabilize the OHM price. 

An inaccurate MA would lead to an incorrect target price, then the wrong amount of corrective action when attempting the price stabilization would follow, ultimately costing the protocol (by selling less OHM during up-trends, deploying less of the Treasury during down-trends, while aso providing less stability to OHM then intended). 
 


## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L312-L335
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L160-L160
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L227-L227
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L696-L705


## Tool used
Manual Review, Forge test


## Recommendation
There are two parts, the price weighting and timespan drift.

### Price weighting
Switching from a SMA to a time-weight average price (TWAP).

### Timespan drift
Keeping with the idea of a protocol that does require manual intervention, a feedback loop to trigger amending the incentives for calling `Heat.beat()`,
that would be the nice approach of steering the timespan of the moving average to move towards the expected `movingAverageDuration_`.
It could either be included as part of the heartbeat or a separate monitor.
