Massive Grey Barracuda

high

# Incorrect StablePool BPT price calculation

## Summary
Incorrect StablePool BPT price calculation as rate's are not considered 

## Vulnerability Detail
The price of a stable pool BPT is computed as:

> minimum price among the pool tokens obtained via feeds * return value of `getRate()`
 
This method is used referring to an old documentation of Balancer 

```solidity
    function getStablePoolTokenPrice(
        address,
        uint8 outputDecimals_,
        bytes calldata params_
    ) external view returns (uint256) {
        // Prevent overflow
        if (outputDecimals_ > BASE_10_MAX_EXPONENT)
            revert Balancer_OutputDecimalsOutOfBounds(outputDecimals_, BASE_10_MAX_EXPONENT);


        address[] memory tokens;
        uint256 poolRate; // pool decimals
        uint8 poolDecimals;
        bytes32 poolId;
        {

        ......

            // Get tokens in the pool from vault
            (address[] memory tokens_, , ) = balVault.getPoolTokens(poolId);
            tokens = tokens_;

            // Get rate
            try pool.getRate() returns (uint256 rate_) {
                if (rate_ == 0) {
                    revert Balancer_PoolStableRateInvalid(poolId, 0);
                }


                poolRate = rate_;

        ......

        uint256 minimumPrice; // outputDecimals_
        {
            /**
             * The Balancer docs do not currently state this, but a historical version noted
             * that getRate() should be multiplied by the minimum price of the tokens in the
             * pool in order to get a valuation. This is the same approach as used by Curve stable pools.
             */
            for (uint256 i; i < len; i++) {
                address token = tokens[i];
                if (token == address(0)) revert Balancer_PoolTokenInvalid(poolId, i, token);

                (uint256 price_, ) = _PRICE().getPrice(token, PRICEv2.Variant.CURRENT); // outputDecimals_


                if (minimumPrice == 0) {
                    minimumPrice = price_;
                } else if (price_ < minimumPrice) {
                    minimumPrice = price_;
                }
            }
        }

        uint256 poolValue = poolRate.mulDiv(minimumPrice, 10 ** poolDecimals); // outputDecimals_
```

The `getRate()` function returns the exchange rate of a BPT to the underlying base asset of the pool which can be different from the minimum market priced asset for pools with rateProviders. To consider this, the price obtained from feeds must be divided by the `rate` provided by `rateProviders` before choosing the minimum as mentioned in the previous version of Balancer's documentation.   

https://github.com/balancer/docs/blob/663e2f4f2c3eee6f85805e102434629633af92a2/docs/concepts/advanced/valuing-bpt/bpt-as-collateral.md#metastablepools-eg-wsteth-weth 


#### 1. Get market price for each constituent token

Get market price of wstETH and WETH in terms of USD, using chainlink oracles.

#### 2. Get RateProvider price for each constituent token

Since wstETH - WETH pool is a MetaStablePool and not a ComposableStablePool, it does not have `getTokenRate()` function.
Therefore, it`s needed to get the RateProvider price manually for wstETH, using the rate providers of the pool. The rate 
provider will return the wstETH token in terms of stETH.

Note that WETH does not have a rate provider for this pool. In that case, assume a value of `1e18` (it means,
market price of WETH won't be divided by any value, and it's used purely in the minPrice formula).

#### 3. Get minimum price

$$ minPrice = min({P_{M_{wstETH}} \over P_{RP_{wstETH}}}, P_{M_{WETH}}) $$

#### 4. Calculates the BPT price

$$ P_{BPT_{wstETH-WETH}} = minPrice * rate_{pool_{wstETH-WETH}} $$

where `rate_pool_wstETH-WETH` is `pool.getRate()` of wstETH-WETH pool.

### Example

The wstEth-cbEth pool is a MetaStablePool having rate providers for both tokens since neither of them is the base token
https://app.balancer.fi/#/ethereum/pool/0x9c6d47ff73e0f5e51be5fd53236e3f595c5793f200020000000000000000042c

At block 18821323:
cbeth : 2317.48812
wstEth : 2526.84
pool total supply : 0.273259897168240633
getRate() : 1.022627523581711856
wstRateprovider rate : 1.150725009180224306
cbEthRateProvider rate : 1.058783029570983377
wstEth balance : 0.133842314907166538
cbeth balance : 0.119822100236557012
tvl : (0.133842314907166538 * 2526.84 + 0.119822100236557012 * 2317.48812) == 615.884408812

according to current implementation:
bpt price = 2317.48812 * 1.022627523581711856 == 2369.927137086
calculated tvl = bpt price * total supply = 647.606045776

correct calculation:
rate_provided_adjusted_cbeth = (2317.48812 / 1.058783029570983377) == 2188.822502132
rate_provided_adjusted_wsteth = (2526.84 / 1.150725009180224306) == 2195.867804942
bpt price = 2188.822502132 * 1.022627523581711856 == 2238.350134915
calculated tvl = bpt price * total supply = (2238.350134915 * 0.273259897168240633) == 611.651327693

## Impact
Incorrect calculation of bpt price. Has possibility to be over and under valued.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L514-L539

## Tool used
Manual Review

## Recommendation
For pools having rate providers, divide prices by rate before choosing the minimum