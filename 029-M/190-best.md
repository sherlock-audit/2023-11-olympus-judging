Upbeat Coffee Robin

high

# Price Module reports wrong MA/Current Price for asset using MA when feeds are down after storePrice has been called

## Summary
Price Module reports wrong MA (and current price) for an asset that uses MA, when all feeds are down after storePrice has been called. This is due to the fact that [storePrice](https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L312) uses _getCurrentPrice which by-design returns the MA when all feeds are down. It then stores the MA as a new observation inspite of the fact that no new information was added.

## Vulnerability Detail
If all feeds are down, an asset using MA reports the MA as current price (by design). However if while feeds are down storePrice is called, the MA will be stored as a new observation (replacing a real observation) creating a new, distorted MA value. This create two problems: 1. the MA reported is wrong (depending on the samples variance and order it can deviate 10% or more as the POC shows). 2. The last observation timestamp is set to the last call to storePrice, causing a failure to report the true freshness of the price. An example of how this affects the system is the _onlyWhileActive function of Operator:
```solidity
function _onlyWhileActive() internal view {
        (, uint48 lastObservation) = PRICE.getPriceIn(
            address(_ohm),
            address(_reserve),
            PRICEv2.Variant.LAST
        );
        if (!active || uint48(block.timestamp) > lastObservation + 3 * PRICE.observationFrequency())
            revert Operator_Inactive();
    }
```
this function is meant to prevent market operations happening based on stale MA (older than three observationFrequency) however in the situation described above it fails to detect staleness because of the stored MA observation.

## Impact
For the Ohm asset, this can cause market operations to start (or fail to stop) based on distorted MA and stale information (see _onlyWhileActive example above). For other assets, distorted current prices will cause Appraiser to report inaccurate metrics (i.e. [backing](https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/policies/OCA/Appraiser.sol#L327)) which in turn affect [Operator targetPrice()](https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/policies/RBS/Operator.sol#L858) causing potential wrongfull activation/deactivation of market operations. Note that a single corrupt asset price can affect multiple assets due to recursive PRICE calls in various PRICE submodules.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L312

### POC:
to run, add the code below to the PRICE.v2.t.sol test file, then run `forge test --match-test 'test_getPrice_feedsdown_MA_failure' -vvv`
```solidity
function test_getPrice_feedsdown_MA_failure() public {
        _addBaseAssets(6);
        uint256[] memory obs_ = new uint256[](5);
        obs_[0] = 32000000000000000000;
        obs_[1] = 18000000000000000000;
        obs_[2] = 19000000000000000000;
        obs_[3] = 20000000000000000000;
        obs_[4] = 16000000000000000000;
        vm.prank(writer);
        price.updateAssetMovingAverage(address(twoma),true,5 * OBSERVATION_FREQUENCY,uint48(block.timestamp),obs_);

        // Set price feeds to zero
        twomaUsdPriceFeed.setLatestAnswer(int256(0));
        twomaEthPriceFeed.setLatestAnswer(int256(0));
        ethUsdPriceFeed.setLatestAnswer(int256(0));

        (uint256 realMA, ) = price.getPrice(address(twoma), PRICEv2.Variant.MOVINGAVERAGE);
        console2.log("Real MA: %s\n",realMA);
        for (uint i;i<6;i++) {
            rollPrice(realMA,i);
        }
    }

    function rollPrice(uint256 realMA, uint256 index) internal {
        //fast forward and call storePrice to reproduce the error
        vm.warp(block.timestamp+OBSERVATION_FREQUENCY);
        vm.prank(writer);
        price.storePrice(address(twoma));
        (uint256 returnedPrice, ) = price.getPrice(address(twoma), PRICEv2.Variant.CURRENT);
        (uint256 movingAverage, ) = price.getPrice(address(twoma), PRICEv2.Variant.MOVINGAVERAGE);
        (uint256 Last, uint48 lupdate) = price.getPrice(address(twoma), PRICEv2.Variant.LAST);
        console2.log("After calling storePrice %s times",index+1);
        console2.log("current price: %s, curre MA: %s, Last Price: %s",returnedPrice,movingAverage,Last);
        uint256 deviation = returnedPrice > realMA ? returnedPrice - realMA : realMA- returnedPrice;
        console2.log("Deviation from real MA: %s%\n",(deviation * 100) / realMA);
        
    }
```

## Tool used

Manual Review

## Recommendation
When storePrice is called, identify the case that MA is returned as current ptice (due to all feeds being down) and avoid storing the price as a new observation in this case.