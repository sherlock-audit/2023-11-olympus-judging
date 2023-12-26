Trendy Cobalt Monkey

medium

# OlympusTreasury.getCategoryBalance may return incorrect balance

## Summary

Different assets can be classified into the same category via [[categorize](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L657-L667)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L657-L667). But different assets have different decimals or different values. The current implementation of [[getCategoryBalance](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L389-L390)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L389-L390) returns the sum of the balances of all assets in the same category. This is not correct.

## Vulnerability Detail

```solidity
File: bophades\src\modules\TRSRY\OlympusTreasury.sol
372:     function getCategoryBalance(
373:         Category category_,
374:         Variant variant_
375:     ) external view override returns (uint256, uint48) {
376:         // Get category group and check that it is valid
377:         CategoryGroup group = categoryToGroup[category_];
378:         if (fromCategoryGroup(group) == bytes32(0))
379:             revert TRSRY_InvalidParams(0, abi.encode(category_));
380: 
381:         // Get category assets
382:->       address[] memory categoryAssets = getAssetsByCategory(category_);
383: 
384:         // Get balance for each asset in the category and add to total
385:         uint256 len = categoryAssets.length;
386:         uint256 balance;
387:         uint48 time;
388:         for (uint256 i; i < len; ) {
389:->           (uint256 assetBalance, uint48 assetTime) = getAssetBalance(categoryAssets[i], variant_);
390:->           balance += assetBalance;
391: 
392:             // Get the most outdated time
393:             if (i == 0) {
394:                 time = assetTime;
395:             } else if (assetTime < time) {
396:                 time = assetTime;
397:             }
398: 
399:             unchecked {
400:                 ++i;
401:             }
402:         }
403: 
404:->       return (balance, time);
405:     }
```

L382, getAssetsByCategory(category_) will iterate the `assets` array, return all assets of the same category_, and assign the results to `categoryAssets`.

L388-402, the `for` loop traverses `categoryAssets` and accumulates the balance of each asset. If `categoryAssets` has only one asset, there is no problem. However, **there may be multiple assets in the same category. Different assets have different values and cannot be simply added**.

L404, returns the accumulated result. This causes the caller to get incorrect results, leading to incorrect behavior.

## Impact

Contracts that rely on the return value of `getCategoryBalance` will get incorrect values, leading to undesirable consequences.

## Code Snippet

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L389-L390

## Tool used

Manual Review

## Recommendation

Oracle should be used to convert other assets into a specified asset(quote token).