Flat Infrared Caterpillar

medium

# `getReserveBalance()` does not return actual tokens held by treasury and it breaks other contract functionality like swaps

## Summary
`getReserveBalance()` does not return actual tokens held by treasury and it breaks other contract functionality like swaps

## Vulnerability Detail

Referencing [this](https://github.com/code-423n4/2022-08-olympus-findings/issues/422) issue which was submitted in code4rena audit and the issue is still there in `OlympusTreasury.sol or TRSRY.v1.sol` as it is not fixed in latest contracts. The issue was acknowledged with reasoning `reserve checks will be added in treasury debt functions`, however, it is not added in debt functions and the issue is still feasible here. Consider below additional context for same,

Some contracts of `olympus` have swaps functionality which will break and the functions calling from `TRSRY.v1.sol` will always not work if the large loan is taken from `OlympusTreasury.sol or TRSRY.v1.sol` contract. To be noted, no contracts is the current audit repo has used a loan.

For example:
`Operator.sol` is out of scope contract for current audit but the inscope contract(`OlympusTreasury.sol or TRSRY.v1.sol`) function is responsible for this issue so the issue should be considered and fix in inscope contract (`OlympusTreasury.sol or TRSRY.v1.sol`) function.

`Operator.sol` has `swap()` function which the swaps the tokenIn and in this function, it calculates the amount of `wrappedReserve` equivalent to `amountOut` and withdraw wrapped reserves from `TRSRY`.

```solidity
            // Calculate amount of wrappedReserve equivalent to amountOut
            // and withdraw wrapped reserves from TRSRY
            TRSRY.withdrawReserves(
                address(this),
                _wrappedReserve,
                _wrappedReserve.previewWithdraw(amountOut)
            );
```
`Operator.swap()` will always fail if the treasury does not have enough tokens to transfer to `address(this)` i.e in this case `Operator.sol`. 

The issue lies here with `getReserveBalance()` in `OlympusTreasury.sol or TRSRY.v1.sol` as it returns the treasury contract has enough tokens which is wrong since capacity is determined based on non-existing tokens while calculating the `getReserveBalance()`

Similar issue is also applicable if the getReserveBalance is referred in case of low market bonds since they are created based on the same capacity.

## Impact
Issue breaks treasury contracts functionality as the actual token held by contract are not taken into consideration which is incorrect as non-existing tokens is being considered here and it also eventaully break treasury functions used in other olympus contracts.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L232

## Tool used
Manual review

## Recommendation
determine capacity from actual tokens held by `OlympusTreasury.sol or TRSRY.v1.sol` instead of non-existing tokens or add a reserve requirement check inside the `OlympusTreasury.sol or TRSRY.v1.sol` debt functions.