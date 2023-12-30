Shallow Cherry Alpaca

medium

# getReservesByCategory() when useSubmodules =true and submoduleReservesSelector=bytes4(0) will revert

## Summary
in `getReservesByCategory() `
Lack of check  `data.submoduleReservesSelector!=""`
when call `submodule.staticcall(abi.encodeWithSelector(data.submoduleReservesSelector));` will revert

## Vulnerability Detail

when `_addCategory()`
if `useSubmodules==true`, `submoduleMetricSelector` must not empty
and `submoduleReservesSelector` can empty (bytes4(0))

like "protocol-owned-treasury"
```solidity
        _addCategory(toCategory("protocol-owned-treasury"), true, 0xb600c5e2, 0x00000000); // getProtocolOwnedTreasuryOhm()`
```
but when call `getReservesByCategory()` , don't check `submoduleReservesSelector!=bytes4(0)` and direct call `submoduleReservesSelector`
```solidity
    function getReservesByCategory(
        Category category_
    ) external view override returns (Reserves[] memory) {
...
        // If category requires data from submodules, count all submodules and their sources.
        len = (data.useSubmodules) ? submodules.length : 0;

...

        for (uint256 i; i < len; ) {
            address submodule = address(_getSubmoduleIfInstalled(submodules[i]));
            (bool success, bytes memory returnData) = submodule.staticcall(
                abi.encodeWithSelector(data.submoduleReservesSelector)
            );
```

this way , when call like `getReservesByCategory(toCategory("protocol-owned-treasury")` will revert

## POC
add to `SUPPLY.v1.t.sol`
```solidity
    function test_getReservesByCategory_includesSubmodules_treasury() public {
        _setUpSubmodules();

        // Add OHM/gOHM in the treasury (which will not be included)
        ohm.mint(address(treasuryAddress), 100e9);
        gohm.mint(address(treasuryAddress), 1e18); // 1 gOHM

        // Categories already defined

        uint256 expectedBptDai = BPT_BALANCE.mulDiv(
            BALANCER_POOL_DAI_BALANCE,
            BALANCER_POOL_TOTAL_SUPPLY
        );
        uint256 expectedBptOhm = BPT_BALANCE.mulDiv(
            BALANCER_POOL_OHM_BALANCE,
            BALANCER_POOL_TOTAL_SUPPLY
        );

        // Check reserves
        SPPLYv1.Reserves[] memory reserves = moduleSupply.getReservesByCategory(
            toCategory("protocol-owned-treasury")
        );
    }
```

```console
 forge test -vv --match-test test_getReservesByCategory_includesSubmodules_treasury

Running 1 test for src/test/modules/SPPLY/SPPLY.v1.t.sol:SupplyTest
[FAIL. Reason: SPPLY_SubmoduleFailed(0xeb502B1d35e975321B21cCE0E8890d20a7Eb289d, 0x0000000000000000000000000000000000000000000000000000000000000000)] test_getReservesByCategory_includesSubmodules_treasury() (gas: 4774197
```

## Impact
some category can't get `Reserves`

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/OlympusSupply.sol#L541

## Tool used

Manual Review

## Recommendation
```diff
    function getReservesByCategory(
        Category category_
    ) external view override returns (Reserves[] memory) {
...


        CategoryData memory data = categoryData[category_];
        uint256 categorySubmodSources;
        // If category requires data from submodules, count all submodules and their sources.
-       len = (data.useSubmodules) ? submodules.length : 0;
+       len = (data.useSubmodules && data.submoduleReservesSelector!=bytes4(0)) ? submodules.length : 0;
```