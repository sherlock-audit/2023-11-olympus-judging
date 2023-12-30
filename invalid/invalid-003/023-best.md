Generous Azure Swallow

high

# OlympusTreasury.setDebt function is not safe against the front-run attack.

## Summary
A `debtor_` is not an administrator and cannot be trusted entirely.
If not, we will not need to set the debt limit for debtors using the `debtApproval` state variable.
If administrator wants to change the debt of `debtor_` by calling `OlympusTreasury.setDebt` function, the `debtor_` can front-run the admin's transaction and can borrow all possible tokens by calling the `OlympusTreasury.incurDebt` function, so that `debtor_` can take out more tokens than the manager's intention.

## Vulnerability Detail
`OlympusTreasury.setDebt` function changes the `reserveDebt[token_][debtor_]` value to parameter value `amount_`.
```solidity
File: OlympusTreasury.sol
197:     function setDebt(
198:         address debtor_,
199:         ERC20 token_,
200:         uint256 amount_
201:     ) external override permissioned {
202:         uint256 oldDebt = reserveDebt[token_][debtor_];
203: 
204:         reserveDebt[token_][debtor_] = amount_;
205: 
206:         if (oldDebt < amount_) totalDebt[token_] += amount_ - oldDebt;
207:         else totalDebt[token_] -= oldDebt - amount_;
208: 
209:         emit DebtSet(token_, debtor_, amount_);
210:     }
```
The `debtor_` monitors mempool and front-run the admin's `OlympusTreasury.setDebt` function call by calling `OlympusTreasury.incurDebt` function with high gas and taking out all tokens avaiable.

Example:
1. Administrator approve 500 tokens to `debtor_` by calling `increaseDebtorApproval` function.
2. `debtor_` take out 100 tokens by calling `incurDebt` function. Now, the debt of `debtor_` is 100 and approval is 400.
3. Admin calls `setDebt` function to decrease the debt of `debtor_` from 100 to 0.
4. `debtor_` monitors mempool and front-run the admin's tx by calling `incurDebt` function to take out 400 tokens.
5. At first, the debt of `debtor_` will become 500 by debtor's transaction and next it will become 0 by admin's transaction.
5. Admin's intention is to decrease the debt of `debtor_` by 100, but in fact, the debt of `debtor_` has been decreased by 500 and the protocol lost tokens by 400.

## Impact
When the administrator calls `OlympusTreasury.setDebt` function, it may lose tokens by front-run attack.
This can be applied if the admin wants to lower the debt of `debtor_` or if admin wants to raise it.
This error corresponds to loss-of-fund.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus-web3-master/blob/main/bophades/src/modules/TRSRY/OlympusTreasury.sol#L197-L210

## Tool used
Manual Review

## Recommendation
`OlympusTreasury.sol` uses `increaseDebtorApproval` and `decreaseDebtorApproval` functions instead of `setDebtorApproval` function when change the `debtApproval` state variables.
Similarly, when we change the `reserveDebt` variables, we should create and use `increaseDebt` and `decreaseDebt` functions instead of `setDebt`.