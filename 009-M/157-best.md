Slow Pebble Stallion

medium

# CustomSupply contains no methodology to set _protocolOwnedTreasuryOhm after it has been set

## Summary

CustomSupply contains the storage variable _protocolOwnedTreasuryOhm which is used by OlympusSupply when determining keep protocol metrics. It can be set in the constructor but there is no way to change the supply after. In the event that there is a large shift in Ohm allocation to or from _protocolOwnedTreasuryOhm, key supply metrics will return incorrectly which can lead to the RBS deploying incorrectly.

## Vulnerability Detail

[CustomSupply.sol#L46-L64](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/submodules/CustomSupply.sol#L46-L64)

    constructor(
        Module parent_,
        uint256 collateralizedOhm_,
        uint256 protocolOwnedBorrowableOhm_,
        uint256 protocolOwnedLiquidityOhm_,
        uint256 protocolOwnedTreasuryOhm_,
        address source_
    ) Submodule(parent_) {
        _collateralizedOhm = collateralizedOhm_;
        _protocolOwnedBorrowableOhm = protocolOwnedBorrowableOhm_;
        _protocolOwnedLiquidityOhm = protocolOwnedLiquidityOhm_;
        _protocolOwnedTreasuryOhm = protocolOwnedTreasuryOhm_;
        _source = source_;


        emit CollateralizedValueUpdated(collateralizedOhm_);
        emit ProtocolOwnedBorrowableValueUpdated(protocolOwnedBorrowableOhm_);
        emit ProtocolOwnedLiquidityValueUpdated(protocolOwnedLiquidityOhm_);
        emit SourceValueUpdated(source_);
    }

Here we see _protocolOwnedTreasuryOhm set in the constructor. However no function exists to change it after it is set in the constructor, like exists for other key metrics.

## Impact

The _protocolOwnedTreasuryOhm metric can become heavily skewed if tokens are moved to or from a CustomSupply submodule to a location that is automatically accounted.

## Code Snippet

[CustomSupply.sol#L111-L135](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/submodules/CustomSupply.sol#L111-L135)

## Tool used

Manual Review

## Recommendation

Add a setProtocolOwnedBorrowableOhm method to CustomSupply