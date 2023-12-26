Daring Malachite Lizard

medium

# BalancerPoolTokenPrice should support Composable Stable Pools

## Summary
To maintain compatibility with new Balancer stable pools, Olympus needs to update BalancerPoolTokenPrice to support composable stable pools.

## Vulnerability Detail
Composable stable pools deprecate and replace previous stable pool types. See https://docs.balancer.fi/concepts/pools/composable-stable.html, **Composable Stable Pools are a superset of all previous Stable-type pools (Stable Pools, MetaStable Pools, StablePhantom Pools, and StablePool v2) and therefore deprecate all previous pools.**

According to Balancer, going forward any new Balancer stable pools created must use the composable stable pool type.

<img width="1048" alt="Screenshot 2023-12-23 at 4 54 38 PM" src="https://github.com/sherlock-audit/2023-11-olympus-Coinstein/assets/7101806/c28d9cb3-bf5c-47d7-a747-d7043a0a1ebb">

Also, many older stable pools are created with disabled factory, so there are security issues and vulnerabilities

<img width="1138" alt="Screenshot 2023-12-24 at 10 54 10 PM" src="https://github.com/sherlock-audit/2023-11-olympus-Coinstein/assets/7101806/c3249eb5-15fa-40ac-a5f9-a5cd0cb575f0">

However, BalancerPoolTokenPrice#getStablePoolTokenPrice and BalancerPoolTokenPrice#getTokenPriceFromStablePool functions rely on the getLastInvariant() function to retrieve the price for stable pool

```solidity
try pool.getLastInvariant() ... {
// other logic
} catch (bytes memory) {
revert Balancer_PoolTypeNotStable(poolId);
}
```

This getLastInvariant function does not exist in the composable stable pool contracts. For example, the WETH / osETH pool (address 0xDACf5Fa19b1f720111609043ac67A9818262850c) is a composable stable pool that does not have the getLastInvariant() function.

From the following conversation with the sponsor,
<img width="517" alt="Screenshot 2023-12-24 at 10 22 12 PM" src="https://github.com/sherlock-audit/2023-11-olympus-Coinstein/assets/7101806/05a4d982-9824-44ed-a7a2-e19d01bb2fb6">

he insists that Olympus has no plan to support Composable stable pool. 

From now, if Olympus wants to add a new stable pool, the type must be a composable stable pool. And with the current code, it is unable to track pricing for it.

## Impact
Without updates, BalancerPoolTokenPrice will fail to get prices if Olympus creates a stable pool because it must be a Composable stable pool. 

The current design of BalancerPoolTokenPrice only supports getting pricing data from older Balancer stable pool types. With Balancer's shift towards making composable stable pools the sole option for new stable pools going forward, we should improve BalancerPoolTokenPrice contract to handle composable stable pools.

**This doesn't serve as a viable long-term solution, potentially hindering overall usability. If Olympus plans to participate in the new stable Balancer pool, the current code falls short as it cannot accommodate a composable stable pool.**

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L1-L858
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L477

## Tool used
Manual Review

## Recommendation
Update BalancerPoolTokenPrice to add support for interacting with and getting prices from composable stable pools. This is needed for the contract to work with new Balancer stable pools going forward.