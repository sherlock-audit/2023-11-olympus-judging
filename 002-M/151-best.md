Shallow Cherry Alpaca

medium

# removeAsset() when locations.length>1  will revert

## Summary
In `removeAsset()`, the implementation of deleting `locations` is incorrect.
 If `locations.length > 1`, it will revert `out-of-bounds`.

## Vulnerability Detail
In `removeAsset()`, deleting `asset` will clear `locations`. The code is as follows:
```solidity
    function removeAsset(address asset_) external override permissioned {
...

        len = asset.locations.length;
        for (uint256 i; i < len; ) {
            asset.locations[i] = asset.locations[len - 1];
            asset.locations.pop();
            unchecked {
                ++i;
            }
        }
```
The above code loops `pop()`, the size of the array will become smaller and smaller
but it always uses `asset.locations[len - 1]`, which will cause `out-of-bounds`.

## POC

add to `TRSRY.v1_1.t.sol`

```solidity
    function testCorrectness_removeAsset_two_locations() public {
        address[] memory locations = new address[](2);
        locations[0] = address(1);
        locations[1] = address(2);

        // Add an asset
        vm.prank(godmode);
        TRSRY.addAsset(address(reserve), locations);

        // Remove the asset
        vm.prank(godmode);
        TRSRY.removeAsset(address(reserve));

        // Verify asset data
        TRSRYv1_1.Asset memory asset = TRSRY.getAssetData(address(reserve));
        assertEq(asset.approved, false);
        assertEq(asset.lastBalance, 0);
        assertEq(asset.locations.length, 0);

        // Verify asset list
        address[] memory assets = TRSRY.getAssets();
        assertEq(assets.length, 0);
    } 
```

```console
forge test -vv --match-test testCorrectness_removeAsset_two_locations

Running 1 test for src/test/modules/TRSRY.v1_1.t.sol:TRSRYv1_1Test
[FAIL. Reason: panic: array out-of-bounds access (0x32)] testCorrectness_removeAsset_two_locations() (gas: 204563)
```

## Impact

when `locations.length>1`  unable to properly delete `asset`.

## Code Snippet

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L470-L477

## Tool used

Manual Review

## Recommendation

```diff
    function removeAsset(address asset_) external override permissioned {
...

-        len = asset.locations.length;
-        for (uint256 i; i < len; ) {
-            asset.locations[i] = asset.locations[len - 1];
-            asset.locations.pop();
-            unchecked {
-                ++i;
-            }
-        }

```