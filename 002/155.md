Slow Pebble Stallion

medium

# Balancer LP valuation methodologies use the incorrect supply metric

## Summary

In various Balancer LP valuations, totalSupply() is used to determine the total LP supply. However this is not the appropriate method for determining the supply. Instead getActualSupply should be used instead. Depending on the which pool implementation and how much LP is deployed, the valuation can be much too high or too low. Since the RBS pricing is dependent on this metric. It could lead to RBS being deployed at incorrect prices.

## Vulnerability Detail

[AuraBalancerSupply.sol#L345-L362](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/submodules/AuraBalancerSupply.sol#L345-L362)

    uint256 balTotalSupply = pool.balancerPool.totalSupply();
    uint256[] memory balances = new uint256[](_vaultTokens.length);
    // Calculate the proportion of the pool balances owned by the polManager
    if (balTotalSupply != 0) {
        // Calculate the amount of OHM in the pool owned by the polManager
        // We have to iterate through the tokens array to find the index of OHM
        uint256 tokenLen = _vaultTokens.length;
        for (uint256 i; i < tokenLen; ) {
            uint256 balance = _vaultBalances[i];
            uint256 polBalance = (balance * balBalance) / balTotalSupply;


            balances[i] = polBalance;


            unchecked {
                ++i;
            }
        }
    }

To value each LP token the contract divides the valuation of the pool by the total supply of LP. This in itself is correct, however the totalSupply method for a variety of Balancer pools doesn't accurately reflect the true LP supply. If we take a look at a few Balancer pools we can quickly see the issue:

This [pool](https://etherscan.io/token/0x42ed016f826165c2e5976fe5bc3df540c5ad0af7) shows a max supply of 2,596,148,429,273,858 whereas the actual supply is 6454.48. In this case the LP token would be significantly undervalued. If a sizable portion of the reserves are deployed in an affected pool the backing per OHM would appear to the RBS system to be much lower than it really is. As a result it can cause the RBS to deploy its funding incorrectly, potentially selling/buying at a large loss to the protocol.

## Impact

Pool LP can be grossly under/over valued

## Code Snippet

[AuraBalancerSupply.sol#L332-L369](https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/SPPLY/submodules/AuraBalancerSupply.sol#L332-L369)

## Tool used

Manual Review

## Recommendation

Use a try-catch block to always query getActualSupply on each pool to make sure supported pools use the correct metric.