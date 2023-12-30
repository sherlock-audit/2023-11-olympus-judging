Formal White Beaver

high

# getBunniTokenPrice wrongly returns the total price of all tokens

## Summary

The function getBunniTokenPrice() is supposed to return the price of 1 Bunni token (1 share) like all other feeds, but it doesn't.  It returns the total price of all minted tokens/shares for a specific pool (total value of position's reserves) which is wrong. 

## Vulnerability Detail

This happens because the totalValue on line 163 is not devided by the total tokens supply. 

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L110-L166

## Impact

The function getBunniTokenPrice always returns wrong price. This would impact the operation of the RBS module. For instance, using the wrong price during a swap may lead to financial losses for the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L110-L166

## Tool used

Manual Review

## Recommendation

Devide totalValue by the total tokens supply. 