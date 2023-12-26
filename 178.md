Massive Grey Barracuda

medium

# Possible incorrect price for tokens in Balancer stable pool due to amplification parameter update

## Summary
Incorrect price calculation of tokens in StablePools if amplification factor is being updated

## Vulnerability Detail
The amplification parameter used to calculate the invariant can be in a state of update. In such a case, the current amplification parameter can differ from the amplificaiton parameter at the time of the last invariant calculation.
The current implementaiton of `getTokenPriceFromStablePool` doesn't consider this and always uses the amplification factor obtained by calling `getLastInvariant` 

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L811-L827
```solidity
            function getTokenPriceFromStablePool(
        address lookupToken_,
        uint8 outputDecimals_,
        bytes calldata params_
    ) external view returns (uint256) {

                .....

                try pool.getLastInvariant() returns (uint256, uint256 ampFactor) {
                   
                   // @audit the amplification factor as of the last invariant calculation is used
                    lookupTokensPerDestinationToken = StableMath._calcOutGivenIn(
                        ampFactor,
                        balances_,
                        destinationTokenIndex,
                        lookupTokenIndex,
                        1e18,
                        StableMath._calculateInvariant(ampFactor, balances_) // Sometimes the fetched invariant value does not work, so calculate it
                    );
```

https://vscode.blockscan.com/ethereum/0x1e19cf2d73a72ef1332c882f20534b6519be0276
StablePool.sol
```solidity

        // @audit the amplification parameter can be updated
        function startAmplificationParameterUpdate(uint256 rawEndValue, uint256 endTime) external authenticate {

        // @audit for calculating the invariant the current amplification factor is obtained by calling _getAmplificationParameter()
        function _onSwapGivenIn(
        SwapRequest memory swapRequest,
        uint256[] memory balances,
        uint256 indexIn,
        uint256 indexOut
    ) internal virtual override whenNotPaused returns (uint256) {
        (uint256 currentAmp, ) = _getAmplificationParameter();
        uint256 amountOut = StableMath._calcOutGivenIn(currentAmp, balances, indexIn, indexOut, swapRequest.amount);
        return amountOut;
    }
```


## Impact
In case the amplification parameter of a pool is being updated by the admin, wrong price will be calculated.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L811-L827

## Tool used
Manual Review

## Recommendation
Use the latest amplification factor by callling the `getAmplificationParameter` function