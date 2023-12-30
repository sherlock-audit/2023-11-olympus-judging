Innocent Concrete Panther

medium

# Deprecated Usage of `answeredInRound` in ChainlinkPriceFeeds

## Summary
The ChainlinkPriceFeeds contract exhibits a moderate-level issue related to the usage of the deprecated variable `answeredInRound`. This variable is a legacy artifact from the time when Chainlink price feeds were based on the Flux Aggregator model instead of OCR (Off-Chain Reporting). It is no longer necessary to check the relationship between `answeredInRound` and `roundId`.

## Vulnerability Detail
The `answeredInRound` variable was historically used to check if the answer was in a different round. In the current design, `roundId` represents the round in which the answer was created, and every time a Chainlink network updates a price feed, they increment the roundId by `1`.

**answeredInRound** was a legacy variable from the Flux Aggregator model, where it was possible for a price to be updated slowly and leak into the next "round." However, with the transition to OCR (Off-Chain Reporting), this scenario is no longer applicable.

## Impact
The impact of this issue is moderate, as it involves a deprecated check that is no longer relevant in the current Chainlink design. However, failing to update the code accordingly may result in redundant and potentially confusing logic.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus-iftikharuddin/blob/79926c74b9eb249eb9f0ed9ea25ce8d516446dd7/bophades/src/modules/PRICE/submodules/feeds/ChainlinkPriceFeeds.sol#L158-L163

https://github.com/sherlock-audit/2023-11-olympus-iftikharuddin/blob/79926c74b9eb249eb9f0ed9ea25ce8d516446dd7/bophades/src/modules/PRICE/submodules/feeds/ChainlinkPriceFeeds.sol#L174-L199

```solidity
function _getFeedPrice(
    AggregatorV2V3Interface feed_,
    uint256 updateThreshold_,
    uint8 feedDecimals_,
    uint8 outputDecimals_
)
    internal
    view
    returns (uint256)
{
    FeedRoundData memory roundData;
    {
        try feed_.latestRoundData() returns (
            uint80 roundId, int256 priceInt, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound
        ) {
            roundData = FeedRoundData(roundId, priceInt, startedAt, updatedAt, answeredInRound);
        } catch (bytes memory) {
            revert Chainlink_FeedInvalid(address(feed_));
        }
    }
    _validatePriceFeedResult(feed_, roundData, block.timestamp, uint256(updateThreshold_));

    uint256 price = uint256(roundData.priceInt);

    return price.mulDiv(10 ** outputDecimals_, 10 ** feedDecimals_);
}
```

## Tool used
Manual Review

## Recommendation
It is recommended to update the codebase specially in `ChainlinkPriceFeeds` to align with the latest Chainlink practices and remove any checks or logic related to the deprecated variable `answeredInRound`. Developers should follow the new OCR model and ensure compatibility with the current Chainlink implementation.

Ref:
- https://docs.chain.link/data-feeds/api-reference#latestrounddata
- https://ethereum.stackexchange.com/questions/133896/whats-the-difference-between-answeredinround-and-roundid-in-a-chainlink-pri
