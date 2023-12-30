Energetic Clay Lemur

medium

# Incompatibility of ERC4626 vault with different asset and underlying decimals

## Summary

The ERC4626 vault implementation incorrectly restricts compatibility to vaults where the asset and underlying token decimals are the same, potentially excluding many valid ERC4626 vaults.

## Vulnerability Detail

The `getPriceFromUnderlying` function in the ERC4626 vault implementation checks for the equality of `assetDecimals` and `underlyingDecimals`. If they are not equal, the function reverts:

        uint256 assetScale;
        {
            uint8 assetDecimals = asset.decimals();
            uint8 underlyingDecimals = ERC20(underlying).decimals();
            // This shouldn't be possible, but we check anyway
            if (assetDecimals != underlyingDecimals) {
                revert ERC4626_AssetDecimalsMismatch(assetDecimals, underlyingDecimals);
            }

            // Don't allow an unreasonably large number of decimals that would result in an overflow
            if (assetDecimals > BASE_10_MAX_EXPONENT) {
                revert ERC4626_AssetDecimalsOutOfBounds(assetDecimals, BASE_10_MAX_EXPONENT);
            }

            assetScale = 10 ** assetDecimals;
        }

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/ERC4626Price.sol#L106-L121

However, many vaults intentionally set `assetDecimals` differently from `underlyingDecimals`, often at a higher value, to mitigate specific risks like vault donation attacks. This approach is used in OpenZeppelin's ERC4626 implementation:

See the OpenZeppelin implentation:

        /**
        * @dev Decimals are computed by adding the decimal offset on top of the underlying asset's decimals. This
        * "original" value is cached during construction of the vault contract. If this read operation fails (e.g., the
        * asset has not been created yet), a default of 18 is used to represent the underlying asset's decimals.
        *
        * See {IERC20Metadata-decimals}.
        */
        function decimals() public view virtual override(IERC20Metadata, ERC20) returns (uint8) {
            return _underlyingDecimals + _decimalsOffset();
        }

https://github.com/OpenZeppelin/openzeppelin-contracts/blob/abcf9dd8b78ca81ac0c3571a6ce9831235ff1b4c/contracts/token/ERC20/extensions/ERC4626.sol#L99-L108

Furthermore, EIP-4626 does not mandate that vault's decimals must match the underlying token’s decimals

        Although the convertTo functions should eliminate the need for any use of an EIP-4626 Vault’s decimals variable, it is still strongly recommended to mirror the underlying token’s decimals if at all possible, to eliminate possible sources of confusion and simplify integration across front-ends and for other off-chain users.

https://eips.ethereum.org/EIPS/eip-4626

## Impact

This limitation in the ERC4626 vault implementation prevents it from being compatible with any ERC4626 Vaults that have different decimals from their underlying assets, thus significantly reducing its applicability.

## Code Snippet

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/ERC4626Price.sol#L106-L121

## Tool used

Manual Review

## Recommendation

Modify the implementation to handle the scenario where the vault's decimals differ from the underlying's decimals, instead of reverting