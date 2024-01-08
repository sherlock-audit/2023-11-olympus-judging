# Issue H-1: OlympusPrice.v2.sol#storePrice: The moving average prices are used recursively for the calculation of the moving average price. 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/55 

## Found by 
dany.armstrong90, nirohgo
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

# Issue H-2: Incorrect ProtocolOwnedLiquidityOhm calculation due to inclusion of other user's reserves 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/172 

## Found by 
hash, tvdung94
## Summary
ProtocolOwnedLiquidityOhm for Bunni can include the liquidity deposited by other users which is not protocol owned

## Vulnerability Detail
The protocol owned liquidity in Bunni is calculated as the sum of reserves of all the BunniTokens
```solidity
    function getProtocolOwnedLiquidityOhm() external view override returns (uint256) {

        uint256 len = bunniTokens.length;
        uint256 total;
        for (uint256 i; i < len; ) {
            TokenData storage tokenData = bunniTokens[i];
            BunniLens lens = tokenData.lens;
            BunniKey memory key = _getBunniKey(tokenData.token);

        .........

            total += _getOhmReserves(key, lens);
            unchecked {
                ++i;
            }
        }


        return total;
    }
```

The deposit function of Bunni allows any user to add liquidity to a token. Hence the returned reserve will contain amounts other than the reserves that actually belong to the protocol
```solidity

    // @audit callable by any user
    function deposit(
        DepositParams calldata params
    )
        external
        payable
        virtual
        override
        checkDeadline(params.deadline)
        returns (uint256 shares, uint128 addedLiquidity, uint256 amount0, uint256 amount1)
    {
    }
```  
## Impact
Incorrect assumption of the protocol owned liquidity and hence the supply. An attacker can inflate the liquidity reserves
The wider system relies on the supply calculation to be correct in order to perform actions of economical impact
```text
https://discord.com/channels/812037309376495636/1184355501258047488/1184397904551628831
it will be determined to get backing
so it will have an economical impact, as we could be exchanging ohm for treasury assets at a wrong price
```

## Code Snippet
POL liquidity is calculated as the sum of bunni token reserves
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/SPPLY/submodules/BunniSupply.sol#L171-L191

BunniHub allows any user to deposit
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/external/bunni/BunniHub.sol#L71-L106

## Tool used
Manual Review

## Recommendation
Guard the deposit function in BunniHub or compute the liquidity using shares belonging to the protocol



## Discussion

**0xJem**

This is a good catch, and the high level is justified

# Issue H-3: Incorrect StablePool BPT price calculation 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/176 

## Found by 
Bauer, ast3ros, ge6a, hash, jasonxiale, tvdung94
## Summary
Incorrect StablePool BPT price calculation as rate's are not considered 

## Vulnerability Detail
The price of a stable pool BPT is computed as:

> minimum price among the pool tokens obtained via feeds * return value of `getRate()`
 
This method is used referring to an old documentation of Balancer 

```solidity
    function getStablePoolTokenPrice(
        address,
        uint8 outputDecimals_,
        bytes calldata params_
    ) external view returns (uint256) {
        // Prevent overflow
        if (outputDecimals_ > BASE_10_MAX_EXPONENT)
            revert Balancer_OutputDecimalsOutOfBounds(outputDecimals_, BASE_10_MAX_EXPONENT);


        address[] memory tokens;
        uint256 poolRate; // pool decimals
        uint8 poolDecimals;
        bytes32 poolId;
        {

        ......

            // Get tokens in the pool from vault
            (address[] memory tokens_, , ) = balVault.getPoolTokens(poolId);
            tokens = tokens_;

            // Get rate
            try pool.getRate() returns (uint256 rate_) {
                if (rate_ == 0) {
                    revert Balancer_PoolStableRateInvalid(poolId, 0);
                }


                poolRate = rate_;

        ......

        uint256 minimumPrice; // outputDecimals_
        {
            /**
             * The Balancer docs do not currently state this, but a historical version noted
             * that getRate() should be multiplied by the minimum price of the tokens in the
             * pool in order to get a valuation. This is the same approach as used by Curve stable pools.
             */
            for (uint256 i; i < len; i++) {
                address token = tokens[i];
                if (token == address(0)) revert Balancer_PoolTokenInvalid(poolId, i, token);

                (uint256 price_, ) = _PRICE().getPrice(token, PRICEv2.Variant.CURRENT); // outputDecimals_


                if (minimumPrice == 0) {
                    minimumPrice = price_;
                } else if (price_ < minimumPrice) {
                    minimumPrice = price_;
                }
            }
        }

        uint256 poolValue = poolRate.mulDiv(minimumPrice, 10 ** poolDecimals); // outputDecimals_
```

The `getRate()` function returns the exchange rate of a BPT to the underlying base asset of the pool which can be different from the minimum market priced asset for pools with rateProviders. To consider this, the price obtained from feeds must be divided by the `rate` provided by `rateProviders` before choosing the minimum as mentioned in the previous version of Balancer's documentation.   

https://github.com/balancer/docs/blob/663e2f4f2c3eee6f85805e102434629633af92a2/docs/concepts/advanced/valuing-bpt/bpt-as-collateral.md#metastablepools-eg-wsteth-weth 


#### 1. Get market price for each constituent token

Get market price of wstETH and WETH in terms of USD, using chainlink oracles.

#### 2. Get RateProvider price for each constituent token

Since wstETH - WETH pool is a MetaStablePool and not a ComposableStablePool, it does not have `getTokenRate()` function.
Therefore, it`s needed to get the RateProvider price manually for wstETH, using the rate providers of the pool. The rate 
provider will return the wstETH token in terms of stETH.

Note that WETH does not have a rate provider for this pool. In that case, assume a value of `1e18` (it means,
market price of WETH won't be divided by any value, and it's used purely in the minPrice formula).

#### 3. Get minimum price

$$ minPrice = min({P_{M_{wstETH}} \over P_{RP_{wstETH}}}, P_{M_{WETH}}) $$

#### 4. Calculates the BPT price

$$ P_{BPT_{wstETH-WETH}} = minPrice * rate_{pool_{wstETH-WETH}} $$

where `rate_pool_wstETH-WETH` is `pool.getRate()` of wstETH-WETH pool.

### Example

The wstEth-cbEth pool is a MetaStablePool having rate providers for both tokens since neither of them is the base token
https://app.balancer.fi/#/ethereum/pool/0x9c6d47ff73e0f5e51be5fd53236e3f595c5793f200020000000000000000042c

At block 18821323:
cbeth : 2317.48812
wstEth : 2526.84
pool total supply : 0.273259897168240633
getRate() : 1.022627523581711856
wstRateprovider rate : 1.150725009180224306
cbEthRateProvider rate : 1.058783029570983377
wstEth balance : 0.133842314907166538
cbeth balance : 0.119822100236557012
tvl : (0.133842314907166538 * 2526.84 + 0.119822100236557012 * 2317.48812) == 615.884408812

according to current implementation:
bpt price = 2317.48812 * 1.022627523581711856 == 2369.927137086
calculated tvl = bpt price * total supply = 647.606045776

correct calculation:
rate_provided_adjusted_cbeth = (2317.48812 / 1.058783029570983377) == 2188.822502132
rate_provided_adjusted_wsteth = (2526.84 / 1.150725009180224306) == 2195.867804942
bpt price = 2188.822502132 * 1.022627523581711856 == 2238.350134915
calculated tvl = bpt price * total supply = (2238.350134915 * 0.273259897168240633) == 611.651327693

## Impact
Incorrect calculation of bpt price. Has possibility to be over and under valued.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L514-L539

## Tool used
Manual Review

## Recommendation
For pools having rate providers, divide prices by rate before choosing the minimum



## Discussion

**0xJem**

This is a valid issue and highlights problems with Balancer's documentation.

We are likely to drop both the Balancer submodules from the final version, since we no longer have any Balancer pools used for POL and don't have any assets that require price resolution via Balancer pools.

# Issue H-4: Incorrect price for tokens of Balancer stable pools due to fixed 1e18 input amount 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/179 

## Found by 
hash
## Summary
Stable token rate is not considered for Balancer MetaStablePool tokens when computing the price

## Vulnerability Detail
The amountIn is always provided as 1e18 for the `StableMath._calcOutGivenIn` function when calculating the price of stable pool token.

```solidity
    function getTokenPriceFromStablePool(
        address lookupToken_,
        uint8 outputDecimals_,
        bytes calldata params_
    ) external view returns (uint256) {

        ......
                try pool.getLastInvariant() returns (uint256, uint256 ampFactor) {
                    // Upscale balances by the scaling factors
                    uint256[] memory scalingFactors = pool.getScalingFactors();
                    uint256 len = scalingFactors.length;
                    for (uint256 i; i < len; ++i) {
                        balances_[i] = FixedPoint.mulDown(balances_[i], scalingFactors[i]);
                    }


                    // Calculate the quantity of lookupTokens returned by swapping 1 destinationToken
                    lookupTokensPerDestinationToken = StableMath._calcOutGivenIn(
                        ampFactor,
                        balances_,
                        destinationTokenIndex,
                        lookupTokenIndex,
                        1e18,
                        StableMath._calculateInvariant(ampFactor, balances_) // Sometimes the fetched invariant value does not work, so calculate it
                    );
        .......
```

The `lookupTokensPerDestinationToken` is used as-is without any further division with the inputted token amount as the input amount is assumed to be 1 unit.

Although in normal stable pools 1 unit of token is upscaled to 1e18 for calculations, pools having rate proivders (MetaStablePool) upscale by also factoring in the token rate.

```solidity
    function _scalingFactors() internal view virtual override returns (uint256[] memory scalingFactors) {
        
        .......

        scalingFactors = super._scalingFactors();
        scalingFactors[0] = scalingFactors[0].mulDown(_priceRate(_token0));
        scalingFactors[1] = scalingFactors[1].mulDown(_priceRate(_token1));
    }
```

Hence the input token amount will deviate from 1 unit of token and will not return the price of the token.

### Example Scenario
In wstEth-cbEth pool wstEth has rate of 1.151313786310212348
Actual input to the `calcAmountIn` should be 1.151313786310212348e18 and not 1e18. This will give an inflated price for cbEth.

## Impact
Incorrect token price calculation for MetaStablePool tokens in which lookup tokens have rates.

## Code Snippet
1e18 is used as input amount always
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L811-L827


## Tool used
Manual Review

## Recommendation
Input the upscaled amount instead of 1e18
```solidity
                    lookupTokensPerDestinationToken = StableMath._calcOutGivenIn(
                        ampFactor,
                        balances_,
                        destinationTokenIndex,
                        lookupTokenIndex,
=>                      FixedPoint.mulDown(1e18,scalingFactors[destinationTokenIndex]),
                        StableMath._calculateInvariant(ampFactor, balances_) // Sometimes the fetched invariant value does not work, so calculate it
                    );
```



## Discussion

**0xJem**

Please provide links to the documentation/code to prove this:
> Although in normal stable pools 1 unit of token is upscaled to 1e18 for calculations, pools having rate proivders (MetaStablePool) upscale by also factoring in the token rate.

We are likely to drop both the Balancer submodules from the final version, since we no longer have any Balancer pools used for POL and don't have any assets that require price resolution via Balancer pools.

**nevillehuang**

Hi @0xJem here is additional information provided by the watson:

I will give the etherscan link of a metastablepool

https://vscode.blockscan.com/ethereum/0x1e19cf2d73a72ef1332c882f20534b6519be0276

on searching _scalingFactors() it can be seen that the token balances are multiplied with priceRate apart from the normal scalingFactor (look for the scalingFacotrs in MetaStablePool.sol)

for the swap flow, the general flow is onSwap -> _swapGivenIn() -> update balances with scaling factors -> _onSwapGivenIn() which actually does the StableMath._calcOutGivenIn calculation.
While updating balances with scaling factors in _swapGivenIn() for metastable pools, 1 unit of token will be converted to 1 * normalScalingFactor * priceRate

# Issue H-5: Incorrect deviation calculation in isDeviatingWithBpsCheck function 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/193 

## Found by 
ast3ros, coffiasd, dany.armstrong90, evilakela, hash
## Summary

The current implementation of the `isDeviatingWithBpsCheck` function in the codebase leads to inaccurate deviation calculations, potentially allowing deviations beyond the specified limits.

## Vulnerability Detail

The function `isDeviatingWithBpsCheck` checks if the deviation between two values exceeds a defined threshold. This function incorrectly calculates the deviation, considering only the deviation from the larger value to the smaller one, instead of the deviation from the mean (or TWAP).

        function isDeviatingWithBpsCheck(
            uint256 value0_,
            uint256 value1_,
            uint256 deviationBps_,
            uint256 deviationMax_
        ) internal pure returns (bool) {
            if (deviationBps_ > deviationMax_)
                revert Deviation_InvalidDeviationBps(deviationBps_, deviationMax_);

            return isDeviating(value0_, value1_, deviationBps_, deviationMax_);
        }

        function isDeviating(
            uint256 value0_,
            uint256 value1_,
            uint256 deviationBps_,
            uint256 deviationMax_
        ) internal pure returns (bool) {
            return
                (value0_ < value1_)
                    ? _isDeviating(value1_, value0_, deviationBps_, deviationMax_)
                    : _isDeviating(value0_, value1_, deviationBps_, deviationMax_);
        }

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/Deviation.sol#L23-L52

The function then call `_isDeviating` to calculate how much the smaller value is deviated from the bigger value.

        function _isDeviating(
            uint256 value0_,
            uint256 value1_,
            uint256 deviationBps_,
            uint256 deviationMax_
        ) internal pure returns (bool) {
            return ((value0_ - value1_) * deviationMax_) / value0_ > deviationBps_;
        }

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/Deviation.sol#L63-L70

The function `isDeviatingWithBpsCheck` is usually used to check how much the current value is deviated from the TWAP value to make sure that the value is not manipulated. Such as spot price and twap price in UniswapV3.

        if (
            // `isDeviatingWithBpsCheck()` will revert if `deviationBps` is invalid.
            Deviation.isDeviatingWithBpsCheck(
                baseInQuotePrice,
                baseInQuoteTWAP,
                params.maxDeviationBps,
                DEVIATION_BASE
            )
        ) {
            revert UniswapV3_PriceMismatch(address(params.pool), baseInQuoteTWAP, baseInQuotePrice);
        }

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/UniswapV3Price.sol#L225-L235

The issue is isDeviatingWithBpsCheck is not check the deviation of current value to the TWAP but deviation from the bigger value to the smaller value. This leads to an incorrect allowance range for the price, permitting deviations that exceed the acceptable threshold.

Example:

TWAP price: 1000
Allow deviation: 10%.

The correct deviation calculation will use deviation from the mean. The allow price will be from 900 to 1100 since:

-   |1100 - 1000| / 1000 = 10%
-   |900 - 1000| / 1000 = 10%

However the current calculation will allow the price from 900 to 1111

-   (1111 - 1000) / 1111 = 10%
-   (1000 - 900) / 1000 = 10%

Even though the actual deviation of 1111 to 1000 is |1111 - 1000| / 1000 = 11.11% > 10%

## Impact

This miscalculation allows for greater deviations than intended, increasing the vulnerability to price manipulation and inaccuracies in Oracle price reporting.

## Code Snippet

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/Deviation.sol#L63-L70

## Tool used

Manual review

## Recommendation

To accurately measure deviation, the isDeviating function should be revised to calculate the deviation based on the mean value: `| spot value - twap value | / twap value`.

# Issue H-6: getBunniTokenPrice wrongly returns the total price of all tokens 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/198 

## Found by 
Arabadzhiev, Drynooo, ge6a, hash, jasonxiale, tvdung94
## Summary

The function getBunniTokenPrice() is supposed to return the price of 1 Bunni token (1 share) like all other feeds, but it doesn't.  It returns the total price of all minted tokens/shares for a specific pool (total value of position's reserves) which is wrong. 

## Vulnerability Detail

This happens because the totalValue on line 163 is not devided by the total tokens supply. 

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L110-L166

## Impact

The function getBunniTokenPrice always returns wrong price. This would impact the operation of the RBS module. For instance, using the wrong price during a swap may lead to financial losses for the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L110-L166

## Tool used

Manual Review

## Recommendation

Devide totalValue by the total tokens supply. 

# Issue M-1: BunniPrice.getBunniTokenPrice doesn't include fees 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/37 

## Found by 
dany.armstrong90, rvierdiiev, tvdung94
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**nirohgo** commented:
> duplicate of 137



**0xJem**

IMO this is medium or low severity - the fees will be compounded regularly, and so aren't going to increasing to a significant amount.

**nevillehuang**

Hi @0xJem, since this could occur naturally without any external factors, I think it could constitute high severity given important price values could be directly affected. 

Does the following mean uncollected fees are going to be collected by olympus regularly?

> the fees will be compounded regularly

**0xJem**

> Hi @0xJem, since this could occur naturally without any external factors, I think it could constitute high severity given important price values could be directly affected.
> 
> Does the following mean uncollected fees are going to be collected by olympus regularly?
> 
> > the fees will be compounded regularly

Yes we will have a keeper function calling the harvest() function on BunniManager. It can be called maximum once in 24 hours.


# Issue M-2: UniswapV3OracleHelper.getTWAPRatio will show incorrect price for negative ticks 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/38 

## Found by 
rvierdiiev
## Summary
UniswapV3OracleHelper.getTWAPRatio will show incorrect price for negative ticks, because getTimeWeightedTick function doesn't round up for negative ticks
## Vulnerability Detail
`UniswapV3OracleHelper.getTWAPRatio` function is used by protocol [to validate price deviation in the `BunniPrice`](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L249-L252) and also [by `UniswapV3Price.getTokenTWAP` function](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/modules/PRICE/submodules/feeds/UniswapV3Price.sol#L154-L160).

Function itself calls [`getTimeWeightedTick` function](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/libraries/UniswapV3/Oracle.sol#L147) to get twap price tick using uniswap oracle. `getTimeWeightedTick` [uses `pool.observe` to get `tickCumulatives` array](https://github.com/sherlock-audit/2023-11-olympus-rvierdiyev/blob/main/bophades/src/libraries/UniswapV3/Oracle.sol#L85C28-L85C43) which is then used to calculate `timeWeightedTick`. 

The problem is that in case if `tickCumulatives[1] - tickCumulatives[0]` is negative, then `timeWeightedTick` should be rounded down [as it's done in the uniswap library](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/OracleLibrary.sol#L36).

As result, in case if `tickCumulatives[1] - tickCumulatives[0]` is negative and `(tickCumulatives[1] - tickCumulatives[0]) % secondsAgo != 0`, then returned `timeWeightedTick` will be bigger then it should be, which opens possibility for some price manipulations and arbitrage opportunities.
## Impact
In case if `tickCumulatives[1] - tickCumulatives[0]` is negative and `((tickCumulatives[1] - tickCumulatives[0]) % secondsAgo != 0`, then returned `timeWeightedTick` will be bigger then it should be.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Add this line.
`if (tickCumulatives[1] - tickCumulatives[0] < 0 && (tickCumulatives[1] - tickCumulatives[0]) % secondsAgo != 0) timeWeightedTick --;`



## Discussion

**0xJem**

Disagree with the severity. Should be low, as it would be a minor difference in the value.

**nevillehuang**

Hi @0xJem, Could you elaborate more on how minor is this difference? Since this can directly impact price, I think this could at least be medium severity.

**0xJem**

> Hi @0xJem, Could you elaborate more on how minor is this difference? Since this can directly impact price, I think this could at least be medium severity.

1 tick is 1 basis point from the next tick:
> Prices from 0 to ∞ can be expressed in sharp enough granularity using a large-enough (int24) signed integer power of 1.0001, called a tick, such that each tick’s price is exactly .01% (1 basis point) away from its neighbor.

https://ryanjameskim.medium.com/uniswap-v3-part-2-ticks-and-fee-acounting-explainer-with-toy-example-e9bf4d706884

**nevillehuang**

> > Hi @0xJem, Could you elaborate more on how minor is this difference? Since this can directly impact price, I think this could at least be medium severity.
> 
> 1 tick is 1 basis point from the next tick:
> 
> > Prices from 0 to ∞ can be expressed in sharp enough granularity using a large-enough (int24) signed integer power of 1.0001, called a tick, such that each tick’s price is exactly .01% (1 basis point) away from its neighbor.
> 
> https://ryanjameskim.medium.com/uniswap-v3-part-2-ticks-and-fee-acounting-explainer-with-toy-example-e9bf4d706884

Might maintain medium severity given although you need huge amount of funds to gain small profits, it is still profits so it would fall under this sherlock [rule for medium severity](https://docs.sherlock.xyz/audits/judging/judging#v.-how-to-identify-a-medium-issue)


> 3. A material loss of funds, no/minimal profit for the attacker at a considerable cost

# Issue M-3: Allowing inconsistent time slices, the moving average can misrepresent the mean price 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/39 

## Found by 
coffiasd, squeaky\_cactus
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



## Discussion

**0xJem**

Acknowledgement of the issue. However, disagree with the severity. `storePrice` is permissioned, so only a policy installed by the owner, the DAO MS, can call it, and the policy can only be called by a whitelisted address.

Low-medium severity

# Issue M-4: Inconsistency in BunniToken Price Calculation 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/49 

## Found by 
Arabadzhiev, KupiaSec, dany.armstrong90, hash, lil.eth, rvierdiiev
## Summary
The deviation check (`_validateReserves()`) from BunniPrice.sol considers both position reserves and uncollected fees when validating the deviation with TWAP, while the final price calculation (`_getTotalValue()`) only accounts for position reserves, excluding uncollected fees. 

The same is applied to BunniSupply.sol where `getProtocolOwnedLiquidityOhm()` validates reserves + fee deviation from TWAP and then returns only Ohm reserves using `lens_.getReserves(key_)` 

Note that `BunniSupply.sol#getProtocolOwnedLiquidityReserves()` validates deviation using reserves+fees with TWAP and then return reserves+fees in a good way without discrepancy.

But this could lead to a misalignment between the deviation check and actual price computation.

## Vulnerability Detail

1. Deviation Check : `_validateReserves` Function:
```solidity
### BunniPrice.sol and BunniSupply.sol : 
    function _validateReserves( BunniKey memory key_,BunniLens lens_,uint16 twapMaxDeviationBps_,uint32 twapObservationWindow_) internal view 
        {
        uint256 reservesTokenRatio = BunniHelper.getReservesRatio(key_, lens_);
        uint256 twapTokenRatio = UniswapV3OracleHelper.getTWAPRatio(address(key_.pool),twapObservationWindow_);

        // Revert if the relative deviation is greater than the maximum.
        if (
            // `isDeviatingWithBpsCheck()` will revert if `deviationBps` is invalid.
            Deviation.isDeviatingWithBpsCheck(
                reservesTokenRatio,
                twapTokenRatio,
                twapMaxDeviationBps_,
                TWAP_MAX_DEVIATION_BASE
            )
        ) {
            revert BunniPrice_PriceMismatch(address(key_.pool), twapTokenRatio, reservesTokenRatio);
        }
    }

### BunniHelper.sol : 
    function getReservesRatio(BunniKey memory key_, BunniLens lens_) public view returns (uint256) {
        IUniswapV3Pool pool = key_.pool;
        uint8 token0Decimals = ERC20(pool.token0()).decimals();

        (uint112 reserve0, uint112 reserve1) = lens_.getReserves(key_);
        
        //E compute fees and return values 
        (uint256 fee0, uint256 fee1) = lens_.getUncollectedFees(key_);
        
        //E calculates ratio of token1 in token0
        return (reserve1 + fee1).mulDiv(10 ** token0Decimals, reserve0 + fee0);
    }

### UniswapV3OracleHelper.sol : 
    //E Returns the ratio of token1 to token0 in token1 decimals based on the TWAP
        //E used in bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol, and SPPLY/submodules/BunniSupply.sol
    function getTWAPRatio(
        address pool_, 
        uint32 period_ //E period of the TWAP in seconds 
    ) public view returns (uint256) 
    {
        //E return the time-weighted tick from period_ to now
        int56 timeWeightedTick = getTimeWeightedTick(pool_, period_);

        IUniswapV3Pool pool = IUniswapV3Pool(pool_);
        ERC20 token0 = ERC20(pool.token0());
        ERC20 token1 = ERC20(pool.token1());

        // Quantity of token1 for 1 unit of token0 at the time-weighted tick
        // Scale: token1 decimals
        uint256 baseInQuote = OracleLibrary.getQuoteAtTick(
            int24(timeWeightedTick),
            uint128(10 ** token0.decimals()), // 1 unit of token0 => baseAmount
            address(token0),
            address(token1)
        );
        return baseInQuote;
    }
```
You can see that the deviation check includes uncollected fees in the `reservesTokenRatio`, potentially leading to a higher or more volatile ratio compared to the historical `twapTokenRatio`.

2. Final Price Calculation in `BunniPrice.sol#_getTotalValue()` : 
```solidity
    function _getTotalValue(
        BunniToken token_,
        BunniLens lens_,
        uint8 outputDecimals_
    ) internal view returns (uint256) {
        (address token0, uint256 reserve0, address token1, uint256 reserve1) = _getBunniReserves(
            token_,
            lens_,
            outputDecimals_
        );
        uint256 outputScale = 10 ** outputDecimals_;

        // Determine the value of each reserve token in USD
        uint256 totalValue;
        totalValue += _PRICE().getPrice(token0).mulDiv(reserve0, outputScale);
        totalValue += _PRICE().getPrice(token1).mulDiv(reserve1, outputScale);

        return totalValue;
    }
```
You can see that this function (`_getTotalValue()`) excludes uncollected fees in the final valuation, potentially overestimating the total value within deviation check process, meaning the check could pass in certain conditions whereas it could have not pass if fees where not accounted on the deviation check.
Moreover the below formula used : 

$$
price_{LP} = {reserve_0 \times price_0 + reserve_1 \times price_1}
$$

where $reserve_i$ is token $i$ reserve amount, $price_i$ is the price of token $i$ 

In short, it is calculated by getting all underlying balances, multiplying those by their market prices

However, this approach of directly computing the price of LP tokens via spot reserves is well-known to be vulnerable to manipulation, even if TWAP Deviation is checked, the above summary proved that this method is not 100% bullet proof as there are discrepancy on what is mesured.
Taken into the fact that the process to check deviation is not that good plus the fact that methodology used to compute price is bad, the impact of this is high

4. The same can be found in BunnySupply.sol `getProtocolOwnedLiquidityReserves()` : 
```solidity
    function getProtocolOwnedLiquidityReserves()
        external
        view
        override
        returns (SPPLYv1.Reserves[] memory)
    {
        // Iterate through tokens and total up the reserves of each pool
        uint256 len = bunniTokens.length;
        SPPLYv1.Reserves[] memory reserves = new SPPLYv1.Reserves[](len);
        for (uint256 i; i < len; ) {
            TokenData storage tokenData = bunniTokens[i];
            BunniToken token = tokenData.token;
            BunniLens lens = tokenData.lens;
            BunniKey memory key = _getBunniKey(token);
            (
                address token0,
                address token1,
                uint256 reserve0,
                uint256 reserve1
            ) = _getReservesWithFees(key, lens);

            // Validate reserves
            _validateReserves(
                key,
                lens,
                tokenData.twapMaxDeviationBps,
                tokenData.twapObservationWindow
            );

            address[] memory underlyingTokens = new address[](2);
            underlyingTokens[0] = token0;
            underlyingTokens[1] = token1;
            uint256[] memory underlyingReserves = new uint256[](2);
            underlyingReserves[0] = reserve0;
            underlyingReserves[1] = reserve1;

            reserves[i] = SPPLYv1.Reserves({
                source: address(token),
                tokens: underlyingTokens,
                balances: underlyingReserves
            });

            unchecked {
                ++i;
            }
        }

        return reserves;
    }
```
Where returned value does not account for uncollected fees whereas deviation check was accounting for it

## Impact

`_getTotalValue()` from BunniPrice.sol and `getProtocolOwnedLiquidityReserves()` from BunniSupply.sol have both ratio computation that includes uncollected fees to compare with TWAP ratio, potentially overestimating the total value compared to what these functions are aim to, which is returning only the reserves or LP Prices by only taking into account the reserves of the pool.
Meaning the check could pass in certain conditions where fees are included in the ratio computation and the deviation check process whereas the deviation check should not have pass without the fees accounted.

## Code Snippet

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/submodules/BunniSupply.sol#L212-L260
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L110

## Tool used

Manual Review

## Recommendation
Align the methodology used in both the deviation check and the final price computation. This could involve either including the uncollected fees in both calculations or excluding them in both.

It's ok for BunniSupply as there are 2 functions handling both reserves and reserves+fees but change deviation check process on the second one to include only reserves when checking deviation twap ratio



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**nirohgo** commented:
> True observation but the effect on deviation is miniscule and no viable scenario has been shown that leads to a loss of material amounts.



**0xJem**

Accurate that uncollected fees are excluded from the TWAP check but included in the reserves check, which could lead to inconsistencies. This has been made consistent now.

> this approach of directly computing the price of LP tokens via spot reserves is well-known to be vulnerable to manipulation

We are aware, hence the reserves & TWAP check, plus re-entrancy check.

# Issue M-5: Price can be miscalculated. 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/56 

## Found by 
dany.armstrong90
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



## Discussion

**0xJem**

Agree with the highlighted issue, disagree with the proposed solution.

# Issue M-6: The calculation of current price of asset can be wrong. 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/57 

## Found by 
dany.armstrong90, nobody2018
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



## Discussion

**0xJem**

The onus would be on the system operator(s) to ensure that prices are updated in a timely fashion using `storePrice`. The key asset prices, for OHM and DAI, are stored with every heartbeat (8 hours) in Heart.sol, so this is unlikely to be an issue.

**0xJem**

We reconsidered this and feel that a check is safer, rather than relying upon the system operator

# Issue M-7: getReservesByCategory() when useSubmodules =true and submoduleReservesSelector=bytes4(0) will revert 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/149 

## Found by 
bin2chen, dany.armstrong90, lemonmon, rvierdiiev
## Summary
in `getReservesByCategory() `
Lack of check  `data.submoduleReservesSelector!=""`
when call `submodule.staticcall(abi.encodeWithSelector(data.submoduleReservesSelector));` will revert

## Vulnerability Detail

when `_addCategory()`
if `useSubmodules==true`, `submoduleMetricSelector` must not empty
and `submoduleReservesSelector` can empty (bytes4(0))

like "protocol-owned-treasury"
```solidity
        _addCategory(toCategory("protocol-owned-treasury"), true, 0xb600c5e2, 0x00000000); // getProtocolOwnedTreasuryOhm()`
```
but when call `getReservesByCategory()` , don't check `submoduleReservesSelector!=bytes4(0)` and direct call `submoduleReservesSelector`
```solidity
    function getReservesByCategory(
        Category category_
    ) external view override returns (Reserves[] memory) {
...
        // If category requires data from submodules, count all submodules and their sources.
        len = (data.useSubmodules) ? submodules.length : 0;

...

        for (uint256 i; i < len; ) {
            address submodule = address(_getSubmoduleIfInstalled(submodules[i]));
            (bool success, bytes memory returnData) = submodule.staticcall(
                abi.encodeWithSelector(data.submoduleReservesSelector)
            );
```

this way , when call like `getReservesByCategory(toCategory("protocol-owned-treasury")` will revert

## POC
add to `SUPPLY.v1.t.sol`
```solidity
    function test_getReservesByCategory_includesSubmodules_treasury() public {
        _setUpSubmodules();

        // Add OHM/gOHM in the treasury (which will not be included)
        ohm.mint(address(treasuryAddress), 100e9);
        gohm.mint(address(treasuryAddress), 1e18); // 1 gOHM

        // Categories already defined

        uint256 expectedBptDai = BPT_BALANCE.mulDiv(
            BALANCER_POOL_DAI_BALANCE,
            BALANCER_POOL_TOTAL_SUPPLY
        );
        uint256 expectedBptOhm = BPT_BALANCE.mulDiv(
            BALANCER_POOL_OHM_BALANCE,
            BALANCER_POOL_TOTAL_SUPPLY
        );

        // Check reserves
        SPPLYv1.Reserves[] memory reserves = moduleSupply.getReservesByCategory(
            toCategory("protocol-owned-treasury")
        );
    }
```

```console
 forge test -vv --match-test test_getReservesByCategory_includesSubmodules_treasury

Running 1 test for src/test/modules/SPPLY/SPPLY.v1.t.sol:SupplyTest
[FAIL. Reason: SPPLY_SubmoduleFailed(0xeb502B1d35e975321B21cCE0E8890d20a7Eb289d, 0x0000000000000000000000000000000000000000000000000000000000000000)] test_getReservesByCategory_includesSubmodules_treasury() (gas: 4774197
```

## Impact
some category can't get `Reserves`

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/OlympusSupply.sol#L541

## Tool used

Manual Review

## Recommendation
```diff
    function getReservesByCategory(
        Category category_
    ) external view override returns (Reserves[] memory) {
...


        CategoryData memory data = categoryData[category_];
        uint256 categorySubmodSources;
        // If category requires data from submodules, count all submodules and their sources.
-       len = (data.useSubmodules) ? submodules.length : 0;
+       len = (data.useSubmodules && data.submoduleReservesSelector!=bytes4(0)) ? submodules.length : 0;
```



## Discussion

**0xJem**

Good catch! Thank you for the clear explanation and test case, too.

# Issue M-8: removeAsset() when locations.length>1  will revert 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/151 

## Found by 
Arabadzhiev, AuditorPraise, KupiaSec, bin2chen, dany.armstrong90, evilakela, ge6a, hash, jah, jasonxiale, jovi, pontifex, rvierdiiev, squeaky\_cactus, tvdung94, zach030
## Summary
In `removeAsset()`, the implementation of deleting `locations` is incorrect.
 If `locations.length > 1`, it will revert `out-of-bounds`.

## Vulnerability Detail
In `removeAsset()`, deleting `asset` will clear `locations`. The code is as follows:
```solidity
    function removeAsset(address asset_) external override permissioned {
...

        len = asset.locations.length;
        for (uint256 i; i < len; ) {
            asset.locations[i] = asset.locations[len - 1];
            asset.locations.pop();
            unchecked {
                ++i;
            }
        }
```
The above code loops `pop()`, the size of the array will become smaller and smaller
but it always uses `asset.locations[len - 1]`, which will cause `out-of-bounds`.

## POC

add to `TRSRY.v1_1.t.sol`

```solidity
    function testCorrectness_removeAsset_two_locations() public {
        address[] memory locations = new address[](2);
        locations[0] = address(1);
        locations[1] = address(2);

        // Add an asset
        vm.prank(godmode);
        TRSRY.addAsset(address(reserve), locations);

        // Remove the asset
        vm.prank(godmode);
        TRSRY.removeAsset(address(reserve));

        // Verify asset data
        TRSRYv1_1.Asset memory asset = TRSRY.getAssetData(address(reserve));
        assertEq(asset.approved, false);
        assertEq(asset.lastBalance, 0);
        assertEq(asset.locations.length, 0);

        // Verify asset list
        address[] memory assets = TRSRY.getAssets();
        assertEq(assets.length, 0);
    } 
```

```console
forge test -vv --match-test testCorrectness_removeAsset_two_locations

Running 1 test for src/test/modules/TRSRY.v1_1.t.sol:TRSRYv1_1Test
[FAIL. Reason: panic: array out-of-bounds access (0x32)] testCorrectness_removeAsset_two_locations() (gas: 204563)
```

## Impact

when `locations.length>1`  unable to properly delete `asset`.

## Code Snippet

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L470-L477

## Tool used

Manual Review

## Recommendation

```diff
    function removeAsset(address asset_) external override permissioned {
...

-        len = asset.locations.length;
-        for (uint256 i; i < len; ) {
-            asset.locations[i] = asset.locations[len - 1];
-            asset.locations.pop();
-            unchecked {
-                ++i;
-            }
-        }

```



## Discussion

**0xJem**

Technically correct.

Low severity. The issue would not affect the treasury valuation (which is the purpose of all of this), and `removeAsset()` is a permissioned function called by a whitelisted admin (via the policy).

**nevillehuang**

Hi @0xJem, to me the watsons highlighted a valid point that can break core functionality of `removeAsset` as long as locations.length is greater than 1, which would constitute medium severity. I think it is intended that there will be more than one location for an asset, unless I am missing something. Open to hearing your opinion.

# Issue M-9: Balancer LP valuation methodologies use the incorrect supply metric 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/155 

## Found by 
0x52, 0xMR0, Arabadzhiev, AuditorPraise, Bauer, CL001, Drynooo, ZanyBonzy, ast3ros, bin2chen, coffiasd, cu5t0mPe0, ge6a, hash, jovi, shealtielanz, tvdung94
## Summary

In various Balancer LP valuations, totalSupply() is used to determine the total LP supply. However this is not the appropriate method for determining the supply. Instead getActualSupply should be used instead. Depending on the which pool implementation and how much LP is deployed, the valuation can be much too high or too low. Since the RBS pricing is dependent on this metric. It could lead to RBS being deployed at incorrect prices.

## Vulnerability Detail

[AuraBalancerSupply.sol#L345-L362](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/submodules/AuraBalancerSupply.sol#L345-L362)

    uint256 balTotalSupply = pool.balancerPool.totalSupply();
    uint256[] memory balances = new uint256[](_vaultTokens.length);
    // Calculate the proportion of the pool balances owned by the polManager
    if (balTotalSupply != 0) {
        // Calculate the amount of OHM in the pool owned by the polManager
        // We have to iterate through the tokens array to find the index of OHM
        uint256 tokenLen = _vaultTokens.length;
        for (uint256 i; i < tokenLen; ) {
            uint256 balance = _vaultBalances[i];
            uint256 polBalance = (balance * balBalance) / balTotalSupply;


            balances[i] = polBalance;


            unchecked {
                ++i;
            }
        }
    }

To value each LP token the contract divides the valuation of the pool by the total supply of LP. This in itself is correct, however the totalSupply method for a variety of Balancer pools doesn't accurately reflect the true LP supply. If we take a look at a few Balancer pools we can quickly see the issue:

This [pool](https://etherscan.io/token/0x42ed016f826165c2e5976fe5bc3df540c5ad0af7) shows a max supply of 2,596,148,429,273,858 whereas the actual supply is 6454.48. In this case the LP token would be significantly undervalued. If a sizable portion of the reserves are deployed in an affected pool the backing per OHM would appear to the RBS system to be much lower than it really is. As a result it can cause the RBS to deploy its funding incorrectly, potentially selling/buying at a large loss to the protocol.

## Impact

Pool LP can be grossly under/over valued

## Code Snippet

[AuraBalancerSupply.sol#L332-L369](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/submodules/AuraBalancerSupply.sol#L332-L369)

## Tool used

Manual Review

## Recommendation

Use a try-catch block to always query getActualSupply on each pool to make sure supported pools use the correct metric.



## Discussion

**0xJem**

This is a valid issue and highlights problems with Balancer's documentation.

We are likely to drop both the Balancer submodules from the final version, since we no longer have any Balancer pools used for POL and don't have any assets that require price resolution via Balancer pools.

# Issue M-10: OlympusTreasury#removeCategoryGroup fails to properly clear the categorization which can lead to incorrect asset categorization 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/158 

## Found by 
0x52, Arabadzhiev, Drynooo, dany.armstrong90, ggg\_ttt\_hhh, tvdung94
## Summary

The categorization mapping in OlympusTreasury is used to track which assets are associated with which category groups. When removing a categoryGroup, the categorization array is never properly cleared. This leaves a tangled and potentially hazardous mapping. In the even the category is added again, it could lead to incorrect valuation when deploying RBS.

## Vulnerability Detail

[OlympusTreasury.sol#L657-L667](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L657-L667)

    function categorize(address asset_, Category category_) external override permissioned {
        // Check that asset is initialized
        if (!assetData[asset_].approved) revert TRSRY_InvalidParams(0, abi.encode(asset_));


        // Check if the category exists by seeing if it has a non-zero category group
        CategoryGroup group = categoryToGroup[category_];
        if (fromCategoryGroup(group) == bytes32(0)) revert TRSRY_CategoryDoesNotExist(category_);


        // Store category data for address
        categorization[asset_][group] = category_;
    }

Above we see that when an asset is categorized, it is added to the categorization mapping. This mapping is essential for determining which assets are in which category.

[OlympusTreasury.sol#L567-L583](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L567-L583)

    function removeCategoryGroup(CategoryGroup group_) external override permissioned {
        // Check if the category group exists
        if (!_categoryGroupExists(group_)) revert TRSRY_CategoryGroupDoesNotExist(group_);


        // Remove category group
        uint256 len = categoryGroups.length;
        for (uint256 i; i < len; ) {
            if (fromCategoryGroup(categoryGroups[i]) == fromCategoryGroup(group_)) {
                categoryGroups[i] = categoryGroups[len - 1];
                categoryGroups.pop();
                break;
            }
            unchecked {
                ++i;
            }
        }

Inside OlympusTreasury#removeCategoryGroup we can see that even though the categoryGroup is removed, the assets are never cleared from it. This can lead to issues down the line. As an example, if a the category is re-added at a later date, it may contain assets that are no longer supported or have since been re-categorized. This could lead to invalid protocol metrics that cause RBS (or other future modules) funds to be deployed incorrectly  

## Impact

Incorrect protocol metrics leading polices to function incorrectly

## Code Snippet

[OlympusTreasury.sol#L567-L583](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L567-L583)

## Tool used

Manual Review

## Recommendation

Clear the categorization of every asset associated with that categoryGroup



## Discussion

**0xJem**

Low severity - does not put funds at risk.

**nevillehuang**

Hi @0xJem, I think the watson highlighted a valid scenario that could impact core functionality even though no funds are at loss.

>  As an example, if a the category is re-added at a later date, it may contain assets that are no longer supported or have since been re-categorized. This could lead to invalid protocol metrics that cause RBS (or other future modules) funds to be deployed incorrectly


# Issue M-11: Possible outdated price for tokens in Balancer stable pools due to cached rate 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/177 

## Found by 
hash
## Summary
While computing the token price from a balancer stable pool, cached rates may be used leading to incorrect prices.

## Vulnerability Detail
When computing the token price inside `getTokenPriceFromStablePool` the current scalingFactors are fetched from the Balancer pool in order to scale the token amounts by calling the `getScalingFactors` function
```solidity
    function getTokenPriceFromStablePool(
        address lookupToken_,
        uint8 outputDecimals_,
        bytes calldata params_
    ) external view returns (uint256) {

        ......

                try pool.getLastInvariant() returns (uint256, uint256 ampFactor) {
                    // Upscale balances by the scaling factors
                    uint256[] memory scalingFactors = pool.getScalingFactors();
                    uint256 len = scalingFactors.length;
                    for (uint256 i; i < len; ++i) {
                        balances_[i] = FixedPoint.mulDown(balances_[i], scalingFactors[i]);
                    }

        .......
```

For stable pools with rate providers, the scaling factors also include the rate of the token which is provided by the rateProvider periodically.

https://vscode.blockscan.com/ethereum/0x1e19cf2d73a72ef1332c882f20534b6519be0276
```solidity
    function _scalingFactors() internal view virtual override returns (uint256[] memory scalingFactors) {
        
        .....
        
        scalingFactors = super._scalingFactors();
        scalingFactors[0] = scalingFactors[0].mulDown(_priceRate(_token0));
        scalingFactors[1] = scalingFactors[1].mulDown(_priceRate(_token1));
    }
```

If the time gap b/w two swaps in the pool is large, the scalingFactors can be using the older rate. In Balancer this is mitigated by checking for the cached duration before any swap is made

https://vscode.blockscan.com/ethereum/0x1e19cf2d73a72ef1332c882f20534b6519be0276
```solidity
    function onSwap(
        SwapRequest memory request,
        uint256 balanceTokenIn,
        uint256 balanceTokenOut
    ) public virtual override onlyVault(request.poolId) returns (uint256) {
        _cachePriceRatesIfNecessary();
        return super.onSwap(request, balanceTokenIn, balanceTokenOut);
    }
```


## Impact
Incorrect calculation of token prices

## Code Snippet
using possible outdated token rates
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L811-L827

## Tool used
Manual Review

## Recommendation
To obtain the latest rates for metastable pools, check whether the current rates are expired by calling the `getPriceRateCache` function. If expired, call the `updatePriceRateCache()` function before fetching the scaling factors.

https://vscode.blockscan.com/ethereum/0x1e19cf2d73a72ef1332c882f20534b6519be0276
MetaStable.sol
```solidity
    function getPriceRateCache(IERC20 token)
        external
        view
        returns (
            uint256 rate,
            uint256 duration,
            uint256 expires
        )
    {
        if (_isToken0(token)) return _getPriceRateCache(_getPriceRateCache0());
        if (_isToken1(token)) return _getPriceRateCache(_getPriceRateCache1());
        _revert(Errors.INVALID_TOKEN);
    }
```
```solidity
    function updatePriceRateCache(IERC20 token) external {
        if (_isToken0WithRateProvider(token)) {
            _updateToken0PriceRateCache();
        } else if (_isToken1WithRateProvider(token)) {
            _updateToken1PriceRateCache();
        } else {
            _revert(Errors.INVALID_TOKEN);
        }
    }
```



## Discussion

**0xJem**

This is a valid issue and highlights problems with Balancer's documentation.

We are likely to drop both the Balancer submodules from the final version, since we no longer have any Balancer pools used for POL and don't have any assets that require price resolution via Balancer pools.

# Issue M-12: Possible incorrect price for tokens in Balancer stable pool due to amplification parameter update 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/178 

## Found by 
hash
## Summary
Incorrect price calculation of tokens in StablePools if amplification factor is being updated

## Vulnerability Detail
The amplification parameter used to calculate the invariant can be in a state of update. In such a case, the current amplification parameter can differ from the amplificaiton parameter at the time of the last invariant calculation.
The current implementaiton of `getTokenPriceFromStablePool` doesn't consider this and always uses the amplification factor obtained by calling `getLastInvariant` 

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L811-L827
```solidity
            function getTokenPriceFromStablePool(
        address lookupToken_,
        uint8 outputDecimals_,
        bytes calldata params_
    ) external view returns (uint256) {

                .....

                try pool.getLastInvariant() returns (uint256, uint256 ampFactor) {
                   
                   // @audit the amplification factor as of the last invariant calculation is used
                    lookupTokensPerDestinationToken = StableMath._calcOutGivenIn(
                        ampFactor,
                        balances_,
                        destinationTokenIndex,
                        lookupTokenIndex,
                        1e18,
                        StableMath._calculateInvariant(ampFactor, balances_) // Sometimes the fetched invariant value does not work, so calculate it
                    );
```

https://vscode.blockscan.com/ethereum/0x1e19cf2d73a72ef1332c882f20534b6519be0276
StablePool.sol
```solidity

        // @audit the amplification parameter can be updated
        function startAmplificationParameterUpdate(uint256 rawEndValue, uint256 endTime) external authenticate {

        // @audit for calculating the invariant the current amplification factor is obtained by calling _getAmplificationParameter()
        function _onSwapGivenIn(
        SwapRequest memory swapRequest,
        uint256[] memory balances,
        uint256 indexIn,
        uint256 indexOut
    ) internal virtual override whenNotPaused returns (uint256) {
        (uint256 currentAmp, ) = _getAmplificationParameter();
        uint256 amountOut = StableMath._calcOutGivenIn(currentAmp, balances, indexIn, indexOut, swapRequest.amount);
        return amountOut;
    }
```


## Impact
In case the amplification parameter of a pool is being updated by the admin, wrong price will be calculated.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L811-L827

## Tool used
Manual Review

## Recommendation
Use the latest amplification factor by callling the `getAmplificationParameter` function



## Discussion

**0xJem**

This doesn't seem valid - if the amplification factor is changed since the invariant was last calculated, wouldn't the value of the invariant also be invalid?

**nevillehuang**

Hi @0xJem here is additional information provided by watson:

The invariant used for calculating swap amounts in Balancer is always based on the latest amplification factor hence their calculation would be latest. If there are no join actions, the cached amplification factor which is used by Olympus will not reflect the new one and will result in a different invariant and different token price.

i am attaching a poc if required:
https://gist.github.com/10xhash/8e24d0765ee98def8c6409c71a7d2b17

# Issue M-13: Inadequate Gas Limit in Staticcall while checking for balancer reentrancy would fail 

Source: https://github.com/sherlock-audit/2023-11-olympus-judging/issues/203 

## Found by 
0xMR0
## Summary
Inadequate Gas Limit in Staticcall while checking for balancer reentrancy would fail

## Vulnerability Detail

`VaultReentrancyLib.ensureNotInVaultContext()` is used to check if the balancer contract has been re-entered or not. It does this by doing a `staticcall` on the pool contract and checking the return value. 
According to the solidity docs, if a staticcall encounters a state change, it burns up all gas and returns. The checkReentrancy tries to call `manageUserBalance` on the vault contract, and returns if it finds a state change.

`ensureNotInVaultContext()` is given as below,

```solidity

    function ensureNotInVaultContext(IVault vault) internal view {

         . . . some comments

        // Reduced to 1k gas to avoid large gas cost predictions when using this function in a non-view function
>>      (, bytes memory revertData) = address(vault).staticcall{gas: 1_000}(
            abi.encodeWithSelector(vault.manageUserBalance.selector, 0)
        );

        _require(revertData.length == 0, Errors.REENTRANCY);
    }
```

The gas limit for the static call is set to 1_000, which is insufficient for the target function i.e `vault.manageUserBalance()` to execute successfully if it consumes more gas than the specified limit.

Balancer `VaultReentrancyLib.sol` has provided a gas limit of 10_000 for staticcall so that the function would not fail. This can be checked [here](https://github.com/balancer/balancer-v2-monorepo/blob/227683919a7031615c0bc7f144666cdf3883d212/pkg/pool-utils/contracts/lib/VaultReentrancyLib.sol#L78)

```solidity

    function ensureNotInVaultContext(IVault vault) internal view {
        . . . some comments

>>        (, bytes memory revertData) = address(vault).staticcall{ gas: 10_000 }(
            abi.encodeWithSelector(vault.manageUserBalance.selector, 0)
        );

        _require(revertData.length == 0, Errors.REENTRANCY);
    }
```

For successfully working of staticcall, a 10_000 gas limit is enough. 

Referencing [this](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#aa-call-operations) documentation, 

staticcall gas is calculated as,

> AA-4: STATICCALL
Gas Calculation:
base_gas = access_cost + mem_expansion_cost
Calculate the gas_sent_with_call [below](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#aa-f-gas-to-send-with-call-operations).

> And the final cost of the operation:
gas_cost = base_gas + gas_sent_with_call


> EIP-150 increased the base_cost of the CALL opcode from 40 to 700 gas, but most contracts in use at the time were sending available_gas - 40 with every call. So, when base_cost increased, these contracts were suddenly trying to send more gas than they had left (requested_gas > remaining_gas). To avoid breaking these contracts, if the requested_gas is more than remaining_gas, we send all_but_one_64th of remaining_gas instead of trying to send requested_gas, which would result in an OUT_OF_GAS_ERROR

Therefore, the base cost is itself is 700 gas + access cost(for not touched address will be 2600, Refer [this](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#aa-call-operations) for better understanding) so the `ensureNotInVaultContext()` wont function properly due to insufficient gas. 

The contracts using `ensureNotInVaultContext()` are `BalancerPoolTokenPrice.getWeightedPoolTokenPrice()`, `BalancerPoolTokenPrice.getStablePoolTokenPrice()`,  `BalancerPoolTokenPrice.getTokenPriceFromWeightedPool()`, `BalancerPoolTokenPrice.getTokenPriceFromStablePool()`, `AuraBalancerSupply.getProtocolOwnedLiquidityOhm()`, `AuraBalancerSupply.getProtocolOwnedLiquidityReserves()` and reentrancy check wont function properly in these functions.

## Impact

the gas limit is too low for staticcall, the static call will fail, leading to unexpected behavior or errors while executing `ensureNotInVaultContext()` in above listed functions.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/Balancer/contracts/VaultReentrancyLib.sol#L77


## Tool used
Manual Review

## Recommendation
According to the Balancer monorepo [here](https://github.com/balancer/balancer-v2-monorepo/blob/227683919a7031615c0bc7f144666cdf3883d212/pkg/pool-utils/contracts/lib/VaultReentrancyLib.sol#L78), the staticall must be allocated a 10_000 amount of gas. 

Change the reentrancy check to the following.

```diff

    function ensureNotInVaultContext(IVault vault) internal view {

         . . . some comments

-        // Reduced to 1k gas to avoid large gas cost predictions when using this function in a non-view function
+        // We set the gas limit to 10k for the staticcall to avoid wasting gas when it reverts due to storage modification
-      (, bytes memory revertData) = address(vault).staticcall{gas: 1_000}(
+      (, bytes memory revertData) = address(vault).staticcall{gas: 10_000}(
            abi.encodeWithSelector(vault.manageUserBalance.selector, 0)
        );

        _require(revertData.length == 0, Errors.REENTRANCY);
    }
```

