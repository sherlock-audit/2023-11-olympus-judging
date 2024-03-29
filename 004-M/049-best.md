Mammoth Latte Vulture

high

# Inconsistency in BunniToken Price Calculation

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
