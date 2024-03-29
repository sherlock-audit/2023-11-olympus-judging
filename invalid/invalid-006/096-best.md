Dry Lemonade Hornet

medium

# getBunniTokenPrice output not in outputDecimals

## Summary
getBunniTokenPrice function return values does not follow the outputDecimals argument.

## Vulnerability Detail
Modify test_getBunniTokenPrice at BunniPrice.t.sol file as follows:
```solidity
function test_getBunniTokenPrice() public {
        // Calculate the expected price
        (uint256 ohmReserve_, uint256 usdcReserve_) = _getReserves(poolTokenKey, bunniLens);

        uint256 ohmReserve = ohmReserve_.mulDiv(10 ** PRICE_DECIMALS, 10 ** OHM_DECIMALS);
        uint256 usdcReserve = usdcReserve_.mulDiv(10 ** PRICE_DECIMALS, 10 ** USDC_DECIMALS);
        uint256 expectedPrice = ohmReserve.mulDiv(OHM_PRICE, 1e18) +
            usdcReserve.mulDiv(USDC_PRICE, 1e18);

        // Call
        bytes memory params = abi.encode(
            BunniPrice.BunniParams({
                bunniLens: bunniLensAddress,
                // @audit I've opted to multiply the MAX DEVIATION by a hundred to avoid deviation errors.
                twapMaxDeviationsBps: uint16(TWAP_MAX_DEVIATION_BPS * 100),
                twapObservationWindow: TWAP_OBSERVATION_WINDOW
            })
        );

        uint256 price = submoduleBunniPrice.getBunniTokenPrice(
            poolTokenAddress,
            PRICE_DECIMALS,
            params
        );

  

        // Check values
        assertTrue(price > 0, "should be non-zero");
        assertEq(price, expectedPrice);
        console2.log("18 output decimals price: ", price);

        uint256 price2 = submoduleBunniPrice.getBunniTokenPrice(
            poolTokenAddress,
            1,
            params
        );

        console2.log("1 output decimals price: ", price2);

        uint256 price3 = submoduleBunniPrice.getBunniTokenPrice(
            poolTokenAddress,
            6,
            params
        );
        console2.log("6 output decimals price: ", price3);

    }****
```

Call the test with the following command:
```shell
forge test --match-test test_getBunniTokenPrice -vv
```

Notice the logs:
18 output decimals price:  73333428237792109000000000
1 output decimals price:    73333427600000000000000000
6 output decimals price:    73333428237782000000000000

## Impact
The outputDecimals argument is not taken into account in the final output, as different calls return the same amount of decimals. 
Calling with smaller outputDecimals values as arguments rounds the final output down.

## Code Snippet
```solidity
function getBunniTokenPrice(
        address bunniToken_,
        uint8 outputDecimals_,
        bytes calldata params_
    ) external view returns (uint256) {
    ...
    }
```
[2023-11-olympus/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol at main · sherlock-audit/2023-11-olympus (github.com)](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L110)

## Tool used

Manual Review

## Recommendation

Ensure the final totalValue is cast to outputDecimals at the end of the function's logic.