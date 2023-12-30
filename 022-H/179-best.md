Massive Grey Barracuda

high

# Incorrect price for tokens of Balancer stable pools due to fixed 1e18 input amount

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