Scruffy Quartz Cod

medium

# After addAssetLocation，there is not initialize cache with current value

## Summary
Due to there is not initialize cache with current value after` addAssetLocation`,the last cached `balance `of 'asset'  will be inaccurate.
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/TRSRY/OlympusTreasury.sol#L498

`removeAssetLocation()`
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/TRSRY/OlympusTreasury.sol#L525

## Vulnerability Detail
1.`addAssetLocation` method  is used to adds an additional external address that holds an `asset`,and current `balance `of 'asset'  are accumulated based on all `locations`.
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/TRSRY/OlympusTreasury.sol#L324

2.the last cached `balance`  obtained  through `_getLastBalance` method,
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/TRSRY/OlympusTreasury.sol#L335

as  code comments, 
> `_getLastBalance` method returns the last cached balance of `asset_` (including debt) across all `locations`

,so if there is no initialize cache with current value after `addAssetLocation`,the balance of `asset_`  will be inaccurate


## Impact
Get current balance of 'asset' is the core function of the OlympusTreasury.sol,if there is no initialize cache with current value after `addAssetLocation`, the balance of `asset_`  will be inaccurate，which would hurt the protocol .

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/TRSRY/OlympusTreasury.sol#L498
## Tool used

Manual Review

## Recommendation
Initialize cache with current value

```solidity
    asset.locations.push(location_);
+ (asset.lastBalance, asset.updatedAt) = _getCurrentBalance(asset_);
```
