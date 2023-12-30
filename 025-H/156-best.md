Slow Pebble Stallion

high

# TWAP manipulation of held Bunni tokens can be used to DOS entire RBS system

## Summary

When valuing Bunni tokens, the function will revert if the price is currently outside an acceptable variation. If the valuation of even one of these tokens reverts it will DOS the entire RBS system. A malicious user could manipulate the TWAP of the lowest liquidity token to cause a revert and DOS the RBS at keys times to exploit pricing differences.

## Vulnerability Detail

[BunniSupply.sol#L458-L473](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/submodules/BunniSupply.sol#L458-L473)

    uint256 reservesTokenRatio = BunniHelper.getReservesRatio(key_, lens_);
    uint256 twapTokenRatio = UniswapV3OracleHelper.getTWAPRatio(
        address(key_.pool),
        twapObservationWindow_
    );

    // Revert if the relative deviation is greater than the maximum
    if (
        // Not necessary to use `isDeviatingWithBpsCheck()` as the checked is already performed in `addBunniToken`
        Deviation.isDeviating(
            reservesTokenRatio,
            twapTokenRatio,
            twapMaxDeviationBps_,
            TWAP_MAX_DEVIATION_BASE
        )
    ) {

_validateReserves will revert if the price of the token is too far away from the TWAP. This is generally a good thing but it becomes weak point to the RBS system. RBS depends on this function via the following (long) chain of calls:

Operator#operate => Appraiser#getMetric(liquid_backing_per_backed_OHM) => Appraiser#_liquidBackingPerBackedOhm => Appraiser#_liquidBacking => Appraiser#_backing => OlympusSupply#getReservesByCategory("protocol-owned-liquidity") => BunniSupply#getProtocolOwnedLiquidityReserves => BunniSupply#_validateReserves

Operator#operate is ESSENTIAL for the RBS system to function/update correctly. By purposely timing the DOS a malicious party could prevent the RBS system from adjusting properly in volatile conditions. This can lead to abuse of RBS pricing that is outdated.

To accomplish this, a malicious party could target a low liquidity pool and beginning trading it a highly depressed values to artificially lower the TWAP. With the TWAP maliciously lowered, all calls to operate the RBS will revert.

## Impact

RBS can be DOS'd at opportune times for attackers

## Code Snippet

[BunniSupply.sol#L452-L480](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/submodules/BunniSupply.sol#L452-L480)

## Tool used

Manual Review

## Recommendation

Cache previous reserve rates for Bunni tokens. If the token value using the cached value is larger than a set threshold and the token is deviating from it's TWAP then the RBS system should gracefully shut down or fallback to a simpler methodology.