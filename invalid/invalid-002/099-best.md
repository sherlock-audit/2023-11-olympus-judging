Daring Malachite Lizard

high

# Revert when retrieving pool token price with low output decimals

## Summary
Revert occurs when retrieving pool token price from UniswapV2PoolTokenPrice contract with low output decimals

## Vulnerability Detail
UniswapV2 pool token always has 18 decimals, but output decimals can be less than pool token decimals.

<img width="608" alt="Screenshot 2023-12-24 at 2 59 24 PM" src="https://github.com/sherlock-audit/2023-11-olympus-Coinstein/assets/7101806/89064197-800a-4bc8-83cc-f9996cd27a73">

Issue arises when getPoolTokenPrice() is called with lower output decimals. This happens because scaling the pool supply to the lower output decimals can result in a value of 0, causing a revert in the price calculation.

For example,
1. Set output decimals = 6, smaller than pool token decimals which is 18.
2. Create a Uniswap V2 pair where both tokens have 6 decimals.
3. Add 88e6 for token A and 531441e6 for token B as liquidity to make the total supply = 6838626177.
4. Try to getPoolTokenPrice().

At UniswapV2PoolTokenPrice.sol#L262, pool supply = 0

```solidity
poolSupply = poolSupply_.mulDiv(10 ** outputDecimals_, 10 ** poolDecimals);``` 
```

And therefore at UniswapV2PoolTokenPrice.sol#L286,

```solidity
poolValue = two.mulDiv(priceMultiple, poolSupply);
```

revert happened because of division by 0 

Added a new test unit for the example,

```solidity
contract AuditUniswapV2PoolTokenPriceTest is Test {
    uint256 mainnetFork;
    string MAINNET_RPC_URL = vm.envString("MAINNET_RPC_URL");

    address user = makeAddr("user");

    MockERC20 tokenA;
    MockERC20 tokenB;

    MockPrice internal mockPrice;
    IUniswapV2Pair pair;

    uint8 internal PRICE_DECIMALS = 6; // @audit output decimals = 6
    UniswapV2PoolTokenPrice internal uniswapSubmodule;

    function setUp() public {
        mainnetFork = vm.createSelectFork(MAINNET_RPC_URL);

        uint8 decimalsA = 6; // @audit 6 decimals
        uint8 decimalsB = 6; // @audit 6 decimals

        tokenA = new MockERC20("Token A", "A", decimalsA);
        tokenB = new MockERC20("Token B", "B", decimalsB);

        deal(address(tokenA), user, 88e6 * 10 ** decimalsA);
        deal(address(tokenB), user, 531441e6 * 10 ** decimalsB);

        // Create pair
        pair = IUniswapV2Pair(IUniswapV2Factory(0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f).createPair(address(tokenA), address(tokenB)));
        IUniswapV2Router02 router = IUniswapV2Router02(0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D);

        // Add liquidity
        vm.startPrank(user);
        tokenA.approve(address(router), type(uint256).max);
        tokenB.approve(address(router), type(uint256).max);
        router.addLiquidity(address(tokenA), address(tokenB), 88e6, 531441e6, 0, 0, user, block.timestamp);  // @audit Add 88e6 for token A and 531441e6 for token B as liquidity
        vm.stopPrank();

        assertEq(pair.totalSupply(), 6838626177);  // @audit total supply = 6838626177

        // Set up the UniswapV2 submodule
        {
            mockPrice = new MockPrice(new Kernel(), PRICE_DECIMALS, uint32(8 hours));
            mockPrice.setTimestamp(uint48(block.timestamp));

            uniswapSubmodule = new UniswapV2PoolTokenPrice(mockPrice);

            mockPrice.setPrice(address(tokenA), 10 ** PRICE_DECIMALS);
            mockPrice.setPrice(address(tokenB), 10 ** PRICE_DECIMALS);
        }
    }

    function test_audit_uniswapV2_getPoolTokenPrice() public {
        vm.expectRevert();
        uniswapSubmodule.getPoolTokenPrice(address(0), PRICE_DECIMALS, abi.encode(pair)); // @audit Get pool token price
    }
}
```

See @audit 

## Impact
This issue prevents fetching a pool token price for valid output decimal values.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/UniswapV2PoolTokenPrice.sol#L194
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/UniswapV2PoolTokenPrice.sol#L262
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/UniswapV2PoolTokenPrice.sol#L286

## Tool used
Manual Review

## Recommendation
Change

```solidity
poolSupply = poolSupply_.mulDiv(10 ** outputDecimals_, 10 ** poolDecimals);
```
to

```solidity
poolSupply = FixedPointMathLib.sqrt(balance0) * FixedPointMathLib.sqrt(balance1);
```

Note: The recommended solution is the same as the one proposed in https://github.com/sherlock-audit/2023-11-olympus-Coinstein/issues/4. However, the underlying root cause leading to the issue is different.
