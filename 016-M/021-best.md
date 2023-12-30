Shallow Canvas Tortoise

medium

# `OlympusPricev2::getPrice()` with Variant.Current might run out of gas on using UniV3 and BPT price feeds

## Summary
When `OlympusPricev2::getPrice()` is called with Variant.Current for a certain asset whose UniV3 or Balancer price feed was added with selector `UniswapV3Price.getTokenTWAP.selector` or  `BalancerPoolTokenPrice.getTokenPriceFromWeightedPool.selector` it recursively calls the getPrice() function resulting in the utilization of millions in gas.

## Vulnerability Detail
For this issue, the following scenario must apply:
- A UniV3 pool or Balancer pool for a pair of tokens should exist eg: Let's take OHM and WEHT for UniV3 and OHM and RSV for the Balancer pool.
- All the assets were added to the contract.
- The OHM and WETH should have UniV3 price feed (or) OHM and RSV should have Balancer price feed. This means that both the tokens should have a UniV3 pool and both should have a UniswapV3 price feed set. 
Same for Balancer, both the tokens should be in the balancer pool and have a balancer price feed set.

```solidity
//Uniswap V3
PRICEv2.Component(
                toSubKeycode("PRICE.UNIV3"), // SubKeycode target
                UniswapV3Price.getTokenTWAP.selector, // bytes4 selector
                abi.encode(ohmFeedThreeParams) // bytes memory params
            );

//Balancer
PRICEv2.Component(
                toSubKeycode("PRICE.BPT"), // SubKeycode subKeycode_
                BalancerPoolTokenPrice.getTokenPriceFromWeightedPool.selector, // bytes4 functionSelector_
                abi.encode(bptParams) // bytes memory params_
            );
```
**Uniswap V3 price feed**
When getPrice() is called for an asset OHM or WETH with variant as Current, the getPrice() will be called recursively from `UniswapV3Price::getTokenTWAP()` to search for a quote token.

Steps:
1. Me calling `OlympusPricev2::getPrice(address(ohm), Variant.Current)`
2. `OlympusPricev2` calls `UniswapV3Price::getTokenTWAP()` with OHM as a lookUpToken.
3. `UniswapV3Price::getTokenTWAP()` calls `OlympusPricev2::getPrice(address(weth), Variant. Current)` with WETH as the quote token.
4. Since the Uniswap V3 price feed for WETH is present `OlympusPricev2` calls `UniswapV3Price::getTokenTWAP()` with WETH as a lookUpToken.
5. Now the `UniswapV3Price::getTokenTWAP()` calls `OlympusPricev2::getPrice(address(ohm), Variant. Current)` with OHM as the quote token.
6. And, thus recursively calling each other.


**Balancer price feed**
When getPrice() is called for an asset OHM or RSV with a variant as Current, the getPrice() will be called recursively from `BalancerPoolTokenPrice::getTokenPriceFromWeightedPool()` to search for a destination token.
Steps:
1. Me calling `OlympusPricev2::getPrice(address(ohm), Variant.Current)`
2. `OlympusPricev2` calls `BalancerPoolTokenPrice::getTokenPriceFromWeightedPool()` with OHM as a lookUpToken.
3. `BalancerPoolTokenPrice::getTokenPriceFromWeightedPool()` calls `OlympusPricev2::getPrice(address(reserve), Variant. Current)` with RSV as the destination token.
4. Since the BPT price feed for RSV is present `OlympusPricev2` calls `BalancerPoolTokenPrice::getTokenPriceFromWeightedPool()` with RSV as a lookUpToken.
5. Now the `BalancerPoolTokenPrice::getTokenPriceFromWeightedPool()` calls `OlympusPricev2::getPrice(address(ohm), Variant. Current)` with OHM as the destination token.
6. And, thus recursively calling each other.

Surprisingly, for some reason this recursive call is finite and the final price is being returned, but still, from the POC below you can see it made around 120 calls to getPrice() utilizing a maximum of 6+ million in gas.

###POC for UniV3
The below function can be pasted inside `PRICE.v2.t.sol`

**Note:** Running the test with the following command where the gas limit is set to `50 Million`

```js
forge test --mt test_RecursiveUniV3 --gas-limit 50000000 -vvv --gas-report
```

```solidity
    function test_RecursiveUniV3() public {
        uint256 nonce_ = 10;

        vm.startPrank(writer);

        // WETH - One feed with no strategy
        {
            ChainlinkPriceFeeds.OneFeedParams memory ethParams = ChainlinkPriceFeeds.OneFeedParams(
                ethUsdPriceFeed,
                uint48(24 hours)
            );

            UniswapV3Price.UniswapV3Params memory uniswapParam = UniswapV3Price
                .UniswapV3Params(ohmEthUniV3Pool, uint32(60 seconds), 0);

            PRICEv2.Component[] memory feeds = new PRICEv2.Component[](2);
            feeds[0] = PRICEv2.Component(
                toSubKeycode("PRICE.CHAINLINK"), // SubKeycode subKeycode_
                ChainlinkPriceFeeds.getOneFeedPrice.selector, // bytes4 functionSelector_
                abi.encode(ethParams) // bytes memory params_
            );
            feeds[1] = PRICEv2.Component(
                toSubKeycode("PRICE.UNIV3"), // SubKeycode target
                UniswapV3Price.getTokenTWAP.selector, // bytes4 selector
                abi.encode(uniswapParam) // bytes memory params
            );

            price.addAsset(
                address(weth), // address asset_
                true, // bool storeMovingAverage_ // don't track WETH MA
                false, // bool useMovingAverage_
                // uint32(0), // uint32 movingAverageDuration_
                // uint48(0), // uint48 lastObservationTime_
                // new uint256[](0), // uint256[] memory observations_
                uint32(5 days), // uint32 movingAverageDuration_
                uint48(block.timestamp), // uint48 lastObservationTime_
                _makeRandomObservations(twoma, feeds[0], nonce_, uint256(15)), // uint256[] memory observations_
                PRICEv2.Component(
                    toSubKeycode("PRICE.SIMPLESTRATEGY"),
                    SimplePriceFeedStrategy.getAveragePriceIfDeviation.selector,
                    abi.encode(uint256(300)) // 3% deviation
                ), // Component memory strategy_
                feeds //
            );
        }

        //OHM - Three feeds using the getMedianPriceIfDeviation strategy
        {
            ChainlinkPriceFeeds.OneFeedParams memory ohmFeedOneParams = ChainlinkPriceFeeds
                .OneFeedParams(ohmUsdPriceFeed, uint48(24 hours));

            ChainlinkPriceFeeds.TwoFeedParams memory ohmFeedTwoParams = ChainlinkPriceFeeds
                .TwoFeedParams(
                    ohmEthPriceFeed,
                    uint48(24 hours),
                    ethUsdPriceFeed,
                    uint48(24 hours)
                );

            UniswapV3Price.UniswapV3Params memory ohmFeedThreeParams = UniswapV3Price
                .UniswapV3Params(ohmEthUniV3Pool, uint32(60 seconds), 0);

            PRICEv2.Component[] memory feeds = new PRICEv2.Component[](3);
            feeds[0] = PRICEv2.Component(
                toSubKeycode("PRICE.CHAINLINK"), // SubKeycode target
                ChainlinkPriceFeeds.getOneFeedPrice.selector, // bytes4 selector
                abi.encode(ohmFeedOneParams) // bytes memory params
            );
            feeds[1] = PRICEv2.Component(
                toSubKeycode("PRICE.CHAINLINK"), // SubKeycode target
                ChainlinkPriceFeeds.getTwoFeedPriceMul.selector, // bytes4 selector
                abi.encode(ohmFeedTwoParams) // bytes memory params
            );
            feeds[2] = PRICEv2.Component(
                toSubKeycode("PRICE.UNIV3"), // SubKeycode target
                UniswapV3Price.getTokenTWAP.selector, // bytes4 selector
                abi.encode(ohmFeedThreeParams) // bytes memory params
            );

            price.addAsset(
                address(ohm), // address asset_
                true, // bool storeMovingAverage_ // track OHM MA
                false, // bool useMovingAverage_ // do not use MA in strategy
                uint32(30 days), // uint32 movingAverageDuration_
                uint48(block.timestamp), // uint48 lastObservationTime_
                _makeRandomObservations(ohm, feeds[0], nonce_, uint256(90)), // uint256[] memory observations_
                PRICEv2.Component(
                    toSubKeycode("PRICE.SIMPLESTRATEGY"),
                    SimplePriceFeedStrategy.getMedianPriceIfDeviation.selector,
                    abi.encode(uint256(300)) // 3% deviation
                ), // Component memory strategy_
                feeds // Component[] feeds_
            );
        }

        vm.stopPrank();

        console2.log('start get price......', gasleft());
        (uint256 priceRec,) = price.getPrice(address(ohm), PRICEv2.Variant.CURRENT);
        console2.log('end get price....', gasleft());
        console2.log('price', priceRec);

    }
```

Logs from the above test
```js
Logs:
  start get price...... 45634130
  end get price.... 39629773
  price 10000000000000000000
```

From the log, the total gas used is 6004357 which is around `6 million` in gas

**Gas Report**
In this table, we can the `getPrice()` was called 118 times and the max gas utilized is `6 Million`
| src/modules/PRICE/OlympusPrice.v2.sol:OlympusPricev2 contract |                 |         |         |         |         |
|---------------------------------------------------------------|-----------------|---------|---------|---------|---------|
| Deployment Cost                                               | Deployment Size |         |         |         |         |
| 3582631                                                       | 18033           |         |         |         |         |
| Function Name                                                 | min             | avg     | median  | max     | # calls |
| INIT                                                          | 856             | 856     | 856     | 856     | 1       |
| KEYCODE                                                       | 237             | 237     | 237     | 237     | 8       |
| addAsset                                                      | 1091077         | 2070666 | 2070666 | 3050255 | 2       |
| decimals                                                      | 352             | 1352    | 1352    | 2352    | 2       |
| getPrice                                                      | 803             | 2903541 | 2897286 | 6000315 | 118     |
| getSubmoduleForKeycode                                        | 1019            | 2019    | 2019    | 3019    | 2       |
| installSubmodule                                              | 56619           | 62220   | 56809   | 78645   | 4       |
| kernel                                                        | 889             | 889     | 889     | 889     | 1       |

###POC for BPT
The below function can be pasted inside `PRICE.v2.t.sol`

**Note:** Running the test with the following command where the gas limit is set to `50 Million`

```js
forge test --mt test_RecursiveBPTCall --gas-limit 50000000 -vvv --gas-report
```

```solidity
    function test_RecursiveBPTCall() public {
        uint256 nonce_ = 10;

        BalancerPoolTokenPrice.BalancerWeightedPoolParams
                        memory bptParams = BalancerPoolTokenPrice.BalancerWeightedPoolParams(
                            IWeightedPool(address(bpt))
                        );

        vm.startPrank(writer);

        //OHM - Three feeds using the getMedianPriceIfDeviation strategy
        {
            ChainlinkPriceFeeds.OneFeedParams memory ohmFeedOneParams = ChainlinkPriceFeeds
                .OneFeedParams(ohmUsdPriceFeed, uint48(24 hours));

            ChainlinkPriceFeeds.TwoFeedParams memory ohmFeedTwoParams = ChainlinkPriceFeeds
                .TwoFeedParams(
                    ohmEthPriceFeed,
                    uint48(24 hours),
                    ethUsdPriceFeed,
                    uint48(24 hours)
                );

            UniswapV3Price.UniswapV3Params memory ohmFeedThreeParams = UniswapV3Price
                .UniswapV3Params(ohmEthUniV3Pool, uint32(60 seconds), 0);

            PRICEv2.Component[] memory feeds = new PRICEv2.Component[](3);
            feeds[0] = PRICEv2.Component(
                toSubKeycode("PRICE.CHAINLINK"), // SubKeycode target
                ChainlinkPriceFeeds.getOneFeedPrice.selector, // bytes4 selector
                abi.encode(ohmFeedOneParams) // bytes memory params
            );
            feeds[1] = PRICEv2.Component(
                toSubKeycode("PRICE.CHAINLINK"), // SubKeycode target
                ChainlinkPriceFeeds.getTwoFeedPriceMul.selector, // bytes4 selector
                abi.encode(ohmFeedTwoParams) // bytes memory params
            );
            // feeds[2] = PRICEv2.Component(
            //     toSubKeycode("PRICE.UNIV3"), // SubKeycode target
            //     UniswapV3Price.getTokenTWAP.selector, // bytes4 selector
            //     abi.encode(ohmFeedThreeParams) // bytes memory params
            // );
            feeds[2] = PRICEv2.Component(
                toSubKeycode("PRICE.BPT"), // SubKeycode subKeycode_
                BalancerPoolTokenPrice.getTokenPriceFromWeightedPool.selector, // bytes4 functionSelector_
                abi.encode(bptParams) // bytes memory params_
            );

            price.addAsset(
                address(ohm), // address asset_
                true, // bool storeMovingAverage_ // track OHM MA
                false, // bool useMovingAverage_ // do not use MA in strategy
                uint32(30 days), // uint32 movingAverageDuration_
                uint48(block.timestamp), // uint48 lastObservationTime_
                _makeRandomObservations(ohm, feeds[0], nonce_, uint256(90)), // uint256[] memory observations_
                PRICEv2.Component(
                    toSubKeycode("PRICE.SIMPLESTRATEGY"),
                    SimplePriceFeedStrategy.getMedianPriceIfDeviation.selector,
                    abi.encode(uint256(300)) // 3% deviation
                ), // Component memory strategy_
                feeds // Component[] feeds_
            );
        }

        // RSV - Two feeds using the getAveragePriceIfDeviation strategy
        {
            ChainlinkPriceFeeds.OneFeedParams memory reserveFeedOneParams = ChainlinkPriceFeeds
                .OneFeedParams(reserveUsdPriceFeed, uint48(24 hours));

            ChainlinkPriceFeeds.TwoFeedParams memory reserveFeedTwoParams = ChainlinkPriceFeeds
                .TwoFeedParams(
                    reserveEthPriceFeed,
                    uint48(24 hours),
                    ethUsdPriceFeed,
                    uint48(24 hours)
                );

            PRICEv2.Component[] memory feeds = new PRICEv2.Component[](2);
            feeds[0] = PRICEv2.Component(
                toSubKeycode("PRICE.CHAINLINK"),
                ChainlinkPriceFeeds.getOneFeedPrice.selector,
                abi.encode(reserveFeedOneParams)
            );
            // feeds[1] = PRICEv2.Component(
            //     toSubKeycode("PRICE.CHAINLINK"), // SubKeycode subKeycode_
            //     ChainlinkPriceFeeds.getTwoFeedPriceMul.selector, // bytes4 functionSelector_
            //     abi.encode(reserveFeedTwoParams) // bytes memory params_
            // );
            feeds[1] = PRICEv2.Component(
                toSubKeycode("PRICE.BPT"), // SubKeycode subKeycode_
                BalancerPoolTokenPrice.getTokenPriceFromWeightedPool.selector, // bytes4 functionSelector_
                abi.encode(bptParams) // bytes memory params_
            );

            price.addAsset(
                address(reserve), // address asset_
                true, // bool storeMovingAverage_ // track reserve MA
                false, // bool useMovingAverage_ // do not use MA in strategy
                uint32(30 days), // uint32 movingAverageDuration_
                uint48(block.timestamp), // uint48 lastObservationTime_
                _makeRandomObservations(reserve, feeds[0], nonce_, uint256(90)), // uint256[] memory observations_
                PRICEv2.Component(
                    toSubKeycode("PRICE.SIMPLESTRATEGY"),
                    SimplePriceFeedStrategy.getAveragePriceIfDeviation.selector,
                    abi.encode(uint256(300)) // 3% deviation
                ), // Component memory strategy_
                feeds // Component[] feeds_
            );
        }

        vm.stopPrank();

        console2.log('start get price......', gasleft());
        (uint256 priceRec,) = price.getPrice(address(ohm), PRICEv2.Variant.CURRENT);
        console2.log('end get price....', gasleft());
        console2.log('price', priceRec);

    }

```

Logs from the above test
```js
Logs:
  start get price...... 43824932
  end get price.... 37473045
  price 10000000000000000000
```

From the log, the total gas used is 6351887 which is around `6.3 million` in gas

**Gas Report**
In this table, we can the `getPrice()` was called 120 times and the max gas utilized is `6.3 Million`
| src/modules/PRICE/OlympusPrice.v2.sol:OlympusPricev2 contract |                 |         |         |         |         |
|---------------------------------------------------------------|-----------------|---------|---------|---------|---------|
| Deployment Cost                                               | Deployment Size |         |         |         |         |
| 3582631                                                       | 18033           |         |         |         |         |
| Function Name                                                 | min             | avg     | median  | max     | # calls |
| INIT                                                          | 856             | 856     | 856     | 856     | 1       |
| KEYCODE                                                       | 237             | 237     | 237     | 237     | 8       |
| addAsset                                                      | 2821740         | 2930029 | 2930029 | 3038318 | 2       |
| decimals                                                      | 352             | 1352    | 1352    | 2352    | 2       |
| getPrice                                                      | 803             | 3038831 | 3025634 | 6347845 | 120     |
| getSubmoduleForKeycode                                        | 1019            | 2019    | 2019    | 3019    | 2       |
| installSubmodule                                              | 56619           | 62220   | 56809   | 78645   | 4       |
| kernel                                                        | 889             | 889     | 889     | 889     | 1       |

The same issue will arise while adding the BPT token of the OHM/RSV pool from `OlympusPricev2::addAsset()` since this function calls the `_getCurrentPrice()` function. The impact is less here since it will only be called by the protocol and not public.

This issue will also occur if the price feed is added with the selector `BalancerPoolTokenPrice.getTokenPriceFromStablePool.selector`.

## Impact
Any internal or external contract using the `getPrice()` function has to send extremely high gas for the call to be successful. Any protocol's contract or external contract using this function will be affected.
If a default estimated gas were sent then most of the time the transaction would run out of gas. 
Under the scenario mentioned above, the getPrice() function could be unusable because of high gas utilization.

## Code Snippet
Call to submodules from Olympusv2
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/OlympusPrice.v2.sol#L143

Call to Olympusv2::getPrice() from `getTokenTWAP()`
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/submodules/feeds/UniswapV3Price.sol#L165

Call to Olympusv2::getPrice() from `getTokenPriceFromWeightedPool()`
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L658

## Tool used

Manual Review

## Recommendation
Mitigating this issue is a little tricky to do without state storage. 
One possible solution is to skip looking for a price feed if `msg. sender` is from the same Module.