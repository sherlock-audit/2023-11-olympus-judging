Strong Hazelnut Cyborg

medium

# `addAsset` does not check for duplicates of `locations`, which may result in double counting of balances in `_getCurrentBalance` and break core logics of `OlympusTreasury.sol`

## Summary
When adding a new asset to the `OlympusTreasury.sol` contract using the `addAsset` method, locations of the asset isn't checked for duplicates. Having a duplicated location for an asset will lead to double counting of balances, which will break calculation of overall balance of an asset and hence lead to inaccurate balance being returned. While `addAsset` is permissioned, it is vulnerable to human error.

## Vulnerability Detail
When adding a new asset, the tx creator also needs to provide the locations of an asset, which is a list of addresses that may own the asset and that needs to be counted when getting the total balance for an asset. It is essential to ensure that each address in this array is unique to prevent double counting of the asset balance.

The need to make sure a location is unique for an asset can be confirmed in `addAssetLocation`, which makes sure that any location added must be unique:
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L507-L515

However, the `addAsset` call itself does not have the uniqueness check for each location. Although the function is permissioned, it does not prevent bug from human error / malicious permissioned user, which will lead to inaccurate balances.
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L415-L444

## Impact
`addAsset` is vulnerable to human error and can lead to inaccurate balance being returned. This will break any logics or external contracts that rely on the balance of assets tracked by the treasury.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L415-L444

## Tool used

Manual Review

## Recommendation
Add uniqueness check to each location when `addAsset` is called:

```solidity
    function addAsset(
        address asset_,
        address[] calldata locations_
    ) external override permissioned {
        Asset storage asset = assetData[asset_];

        // Ensure asset is not already added
        if (asset.approved) revert TRSRY_AssetAlreadyApproved(asset_);

        // Check that asset is a contract
        if (asset_.code.length == 0) revert TRSRY_AssetNotContract(asset_);

        // Set asset as approved and add to array
        asset.approved = true;
        assets.push(asset_);

        mapping(address => bool) memory locationsMap; // @audit <---------- creates a map to track each location

        uint256 len = locations_.length;
        for (uint256 i; i < len; ) {
            if(locationsMap[locations_[i]]) revert TRSRY_InvalidParams(1, abi.encode(locations_[i])); // @audit <-------- revert if any location already exists in the map
            locationsMap[locations_[i]] = true; // @audit <-------- add location to the map

            if (locations_[i] == address(0))
                revert TRSRY_InvalidParams(1, abi.encode(locations_[i]));
            asset.locations.push(locations_[i]);
            unchecked {
                ++i;
            }
        }

        // Initialize cache with current value
        (asset.lastBalance, asset.updatedAt) = _getCurrentBalance(asset_);
    }
```