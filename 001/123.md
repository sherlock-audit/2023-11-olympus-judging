Boxy Watermelon Hawk

high

# Invalid price calculation for BunniTokens leads to price manipulation

## Summary
BunniToken price is calculated using price of reserved tokens in the pool, leads to easy price manipulation as Warp Finance has been attacked.
REF: [Pricing LP tokens](https://cmichel.io/pricing-lp-tokens/)

## Vulnerability Detail
`BunniPrice` submodule only works with BunniTokens with full-range positions.
It's not validated directly but it's guaranteed by checking deviations.
```Solidity
function _validateReserves(
    BunniKey memory key_,
    BunniLens lens_,
    uint16 twapMaxDeviationBps_,
    uint32 twapObservationWindow_
) internal view {
    uint256 reservesTokenRatio = BunniHelper.getReservesRatio(key_, lens_);
    uint256 twapTokenRatio = UniswapV3OracleHelper.getTWAPRatio(
        address(key_.pool),
        twapObservationWindow_
    );

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
```
For non full-range positions, `reservesTokenRatio` is not even similar to `twapTokenRatio`.
Thus, these full-range positions on UniswapV3 works as same as UniswapV2 pools.

However, in calculating BunniToken price(aka UniswapV3 LP price), it sums up the price of token reserves in the pool:
```Solidity
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
Calculating the BunniToken price using this formula includes the vulnerability which is described in the reference link above(Warp Finance hack with price manipulation).

## Impact
BunniToken price can be manipulated by an attacker to generate profits from the vulnerability.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L162-L165
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L215-L233

## Tool used
Manual Review

## Recommendation
As in the above reference link, fair LP price calculation is introduced as follows: $$ p(LP) = \frac{2 * \sqrt{p0 * p1 * k}}{L}, k=x * y $$
For UniswapV3, $L = \sqrt{x * y}$, so we can rewrite the above formula like $$ p(LP) = \frac{2 * \sqrt{p0 * p1 * L^2}}{L} = 2 * \sqrt{p0 * p1} $$