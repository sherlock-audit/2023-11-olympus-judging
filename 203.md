Flat Infrared Caterpillar

medium

# Inadequate Gas Limit in Staticcall while checking for balancer reentrancy would fail

## Summary
Inadequate Gas Limit in Staticcall while checking for balancer reentrancy would fail

## Vulnerability Detail

`VaultReentrancyLib.ensureNotInVaultContext()` is used to check if the balancer contract has been re-entered or not. It does this by doing a `staticcall` on the pool contract and checking the return value. 
According to the solidity docs, if a staticcall encounters a state change, it burns up all gas and returns. The checkReentrancy tries to call `manageUserBalance` on the vault contract, and returns if it finds a state change.

`ensureNotInVaultContext()` is given as below,

```solidity

    function ensureNotInVaultContext(IVault vault) internal view {

         . . . some comments

        // Reduced to 1k gas to avoid large gas cost predictions when using this function in a non-view function
>>      (, bytes memory revertData) = address(vault).staticcall{gas: 1_000}(
            abi.encodeWithSelector(vault.manageUserBalance.selector, 0)
        );

        _require(revertData.length == 0, Errors.REENTRANCY);
    }
```

The gas limit for the static call is set to 1_000, which is insufficient for the target function i.e `vault.manageUserBalance()` to execute successfully if it consumes more gas than the specified limit.

Balancer `VaultReentrancyLib.sol` has provided a gas limit of 10_000 for staticcall so that the function would not fail. This can be checked [here](https://github.com/balancer/balancer-v2-monorepo/blob/227683919a7031615c0bc7f144666cdf3883d212/pkg/pool-utils/contracts/lib/VaultReentrancyLib.sol#L78)

```solidity

    function ensureNotInVaultContext(IVault vault) internal view {
        . . . some comments

>>        (, bytes memory revertData) = address(vault).staticcall{ gas: 10_000 }(
            abi.encodeWithSelector(vault.manageUserBalance.selector, 0)
        );

        _require(revertData.length == 0, Errors.REENTRANCY);
    }
```

For successfully working of staticcall, a 10_000 gas limit is enough. 

Referencing [this](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#aa-call-operations) documentation, 

staticcall gas is calculated as,

> AA-4: STATICCALL
Gas Calculation:
base_gas = access_cost + mem_expansion_cost
Calculate the gas_sent_with_call [below](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#aa-f-gas-to-send-with-call-operations).

> And the final cost of the operation:
gas_cost = base_gas + gas_sent_with_call


> EIP-150 increased the base_cost of the CALL opcode from 40 to 700 gas, but most contracts in use at the time were sending available_gas - 40 with every call. So, when base_cost increased, these contracts were suddenly trying to send more gas than they had left (requested_gas > remaining_gas). To avoid breaking these contracts, if the requested_gas is more than remaining_gas, we send all_but_one_64th of remaining_gas instead of trying to send requested_gas, which would result in an OUT_OF_GAS_ERROR

Therefore, the base cost is itself is 700 gas + access cost(for not touched address will be 2600, Refer [this](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#aa-call-operations) for better understanding) so the `ensureNotInVaultContext()` wont function properly due to insufficient gas. 

The contracts using `ensureNotInVaultContext()` are `BalancerPoolTokenPrice.getWeightedPoolTokenPrice()`, `BalancerPoolTokenPrice.getStablePoolTokenPrice()`,  `BalancerPoolTokenPrice.getTokenPriceFromWeightedPool()`, `BalancerPoolTokenPrice.getTokenPriceFromStablePool()`, `AuraBalancerSupply.getProtocolOwnedLiquidityOhm()`, `AuraBalancerSupply.getProtocolOwnedLiquidityReserves()` and reentrancy check wont function properly in these functions.

## Impact

the gas limit is too low for staticcall, the static call will fail, leading to unexpected behavior or errors while executing `ensureNotInVaultContext()` in above listed functions.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/Balancer/contracts/VaultReentrancyLib.sol#L77


## Tool used
Manual Review

## Recommendation
According to the Balancer monorepo [here](https://github.com/balancer/balancer-v2-monorepo/blob/227683919a7031615c0bc7f144666cdf3883d212/pkg/pool-utils/contracts/lib/VaultReentrancyLib.sol#L78), the staticall must be allocated a 10_000 amount of gas. 

Change the reentrancy check to the following.

```diff

    function ensureNotInVaultContext(IVault vault) internal view {

         . . . some comments

-        // Reduced to 1k gas to avoid large gas cost predictions when using this function in a non-view function
+        // We set the gas limit to 10k for the staticcall to avoid wasting gas when it reverts due to storage modification
-      (, bytes memory revertData) = address(vault).staticcall{gas: 1_000}(
+      (, bytes memory revertData) = address(vault).staticcall{gas: 10_000}(
            abi.encodeWithSelector(vault.manageUserBalance.selector, 0)
        );

        _require(revertData.length == 0, Errors.REENTRANCY);
    }
```