Faint Violet Shrimp

medium

# Missing minAnswer & maxAnswers checks in ChainlinkPriceFeeds

## Summary
It's possible that a Chainlink feed will report wrong price in some cases. Chainlink feeds have in-built `minAnswer` and `maxAnswer`. If the price to return is outside the range `[minAnswer; maxAnswer]`, the corresponding cap values will be returned. The [`ChainlinkPriceFeeds:_validatePriceFeedResult()`](https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/submodules/feeds/ChainlinkPriceFeeds.sol#L142-L164) does not validate that the returned value is in the above range. 

## Vulnerability Detail
The price of this token can either fall below the minAnswer or go beyond the maxAnswer. Imagine in the scenario of a crash of one of the assets that back OHM and its price drops close to 0. This would cause the price of OHM to spike beyond the RBS cushion. A market with treasury funds will be allocated to try to normalize the price, even though the asset is now not reliable. This will allow users to buy OHM at a very cheap price, messing up the whole purpose of the RBS.

 
## Impact
RBS not working correctly, can lead to buying cheaper OHM.

## Code Snippet
```solidity
    function _validatePriceFeedResult(
        AggregatorV2V3Interface feed_,
        FeedRoundData memory roundData,
        uint256 blockTimestamp,
        uint256 paramsUpdateThreshold
    ) internal pure {
        if (roundData.priceInt <= 0)
            revert Chainlink_FeedPriceInvalid(address(feed_), roundData.priceInt);

        if (roundData.updatedAt < blockTimestamp - paramsUpdateThreshold)
            revert Chainlink_FeedRoundStale(
                address(feed_),
                roundData.updatedAt,
                blockTimestamp - paramsUpdateThreshold
            );

        if (roundData.answeredInRound != roundData.roundId)
            revert Chainlink_FeedRoundMismatch(
                address(feed_),
                roundData.roundId,
                roundData.answeredInRound
            );
    }
```

## Tool used

Manual Review

## Recommendation
Revert if the price is outside the safe range.
```diff
    function _validatePriceFeedResult(
        AggregatorV2V3Interface feed_,
        FeedRoundData memory roundData,
        uint256 blockTimestamp,
        uint256 paramsUpdateThreshold
    ) internal pure {
        if (roundData.priceInt <= 0)
            revert Chainlink_FeedPriceInvalid(address(feed_), roundData.priceInt);

        if (roundData.updatedAt < blockTimestamp - paramsUpdateThreshold)
            revert Chainlink_FeedRoundStale(
                address(feed_),
                roundData.updatedAt,
                blockTimestamp - paramsUpdateThreshold
            );

        if (roundData.answeredInRound != roundData.roundId)
            revert Chainlink_FeedRoundMismatch(
                address(feed_),
                roundData.roundId,
                roundData.answeredInRound
            );

+     if (roundData.priceInt < feed.minAnswer() || roundData.priceInt > feed.maxAnswer()) 
+         revert(); // Revert with the desired custom error
    }
```