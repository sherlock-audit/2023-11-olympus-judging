Smooth Tawny Shrimp

high

# ChainlinkPriceFeeds.sol :: getTwoFeedPriceDiv() can return incorrect prices due to not validate whether the denominators match.

## Summary
**`getTwoFeedPriceDiv()`** can retrieve an asset price by combining two Chainlink price feeds. However, a drawback of the function lies in its failure to validate whether the denominators match, potentially leading to incorrect calculated prices.
## Vulnerability Detail
**`getTwoFeedPriceDiv()`** obtains the price of an asset using two different Chainlink price feeds.
```Solidity
// Get prices from feeds
        uint256 numeratorPrice = _getFeedPrice(
            params.firstFeed,
            uint256(params.firstUpdateThreshold),
            firstFeedDecimals,
            outputDecimals_
        );
        uint256 denominatorPrice = _getFeedPrice(
            params.secondFeed,
            uint256(params.secondUpdateThreshold),
            secondFeedDecimals,
            outputDecimals_
        );
```
This is the calculation process for determining the final price:
```Solidity
// Convert to numerator/denominator price and return
        uint256 priceResult = numeratorPrice.mulDiv(10 ** outputDecimals_, denominatorPrice);
```
The issue in the functions lies in the absence of a check for matching denominators of the two Chainlink price feeds.

To illustrate the problem more clearly, I will provide examples for both scenarios: 
when the denominators match and when they do not.

-------------------------Denominators match example-------------------------

assetPrice1 = OHM/DAI
assetPrice2 = ETH/DAI

As evident in the **priceResult**, the calculation of the price follows the format outlined below (output decimals are omitted for simplicity):

**`priceResult = OHM/DAI / ETH/DAI = OHM * DAI / ETH * DAI = OHM/ DAI`**

In this case the price is calculated correctly.

-------------------------Denominators NOT match example-------------------------

assetPrice1 = OHM/ETH
assetPrice2 = ETH/DAI

**`priceResult = OHM/ETH / ETH/DAI = OHM * DAI / ETH * ETH = OHM / ETH * DAI/ETH`**

In this scenario, the denominators are not cancelled. 
This leads to an incorrect calculation of the price as it deviates from the desired format of a single asset price in the form A/B. 

As specified in the comments of the function.
```Solidity
/// @notice                 Returns the result of dividing the price from the first Chainlink feed by the price from the second.
/// @dev                    For example, passing in ETH-DAI and USD-DAI will return the ETH-USD price.
```

## Impact
Incorrect price calculations can lead to a wrong returned value, potentially causing a malfunction in the protocol.
## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/ChainlinkPriceFeeds.sol#L253-L305
## Tool used
Manual Review

## POC
Include the following test in **`ChainlinkPriceFeeds.t.sol`** (test/modules/submodules/feeds) to verify the presence of the issue.
```Solidity
 function test_denominatorsNotMatch() public {
        bytes memory params = encodeTwoFeedParams(
            ohmEthPriceFeed,
            UPDATE_THRESHOLD,
            ethDaiPriceFeed,
            UPDATE_THRESHOLD
        );
        uint256 priceInt = chainlinkSubmodule.getTwoFeedPriceDiv(
            address(0),
            PRICE_DECIMALS,
            params
        );

        assertEq(priceInt, (ohmEthPrice * daiEthPrice) / 1e18); // OHM-ETH * DAI-ETH (divided by 1e18 to adjust the decimals)

    }
```

Furthermore, I consulted the protocol team, and their response was as follows:
![Protocol team response](https://github.com/sherlock-audit/2023-11-olympus-IvanFitro/assets/106178180/86cc9a7e-a227-49bd-91f7-5763efd308df)

## Recommendation
To address this issue, you can store the combination of Chainlink price feeds in a mapping, enabling a check to ensure that the denominators of these two Chainlink price feeds match.
```Soldity
mapping(AggregatorV2V3Interface  => mapping(AggregatorV2V3Interface => bool)) feedsDenominatorsMatch;
```