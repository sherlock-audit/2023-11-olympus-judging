Flat Infrared Caterpillar

medium

# Solmate safetransfer and safetransferfrom doesnot check the codesize of the token address, which may lead to fund loss

## Summary
Solmate safetransfer and safetransferfrom doesnot check the codesize of the token address, which may lead to fund loss

## Vulnerability Detail

In functions like `repayDebt()`, `incurDebt()` and `withdrawReserves()` of OlympusTreasury.sol or TRSRY.v1.sol,  the safetransfer and safetransferfrom doesn't check the existence of code at the token address. **This is a known issue while using solmate's libraries.**

Solmate specifically mentions,

> Note that none of the functions in this library check that a token has code at all!

Hence this may lead to miscalculation of funds and may lead to loss of funds , because if safetransfer() and safetransferfrom() are called on a token address that doesn't have contract in it, it will always return success, bypassing the return value check. Due to this protocol will think that funds has been transferred and successful , and records will be accordingly calculated, but in reality funds were never transferred. So this will lead to miscalculation and possibly loss of funds

This issue was also submitted in previous code4rena audit which can be checked [here](https://github.com/code-423n4/2022-08-olympus-findings/issues/117) where the project team had confirmed the issue and recommendation were supposed to be implemented, however the issue still persists with latest contracts.

## Impact
Due to this protocol will think that funds have been transferred successfully, and records will be accordingly calculated, but in reality, funds were never transferred. So this will lead to miscalculation and possibly loss of funds.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L115

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L162

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L180

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L4-L7

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/TransferHelper.sol#L4-L7

## Tool used
Manual Review

## Recommendation
Use openzeppelin's safeERC20 or implement a code existence check. This was also confirmed to be implemented by project team in previous audit issue confirmation [here](https://github.com/code-423n4/2022-08-olympus-findings/issues/117#issuecomment-1240019949)