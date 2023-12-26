Slow Pebble Stallion

medium

# OlympusTreasury#removeCategoryGroup fails to properly clear the categorization which can lead to incorrect asset categorization

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