Massive Grey Barracuda

medium

# Possible outdated price for tokens in Balancer stable pools due to cached rate

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