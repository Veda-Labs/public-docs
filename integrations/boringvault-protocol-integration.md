# BoringVault Protocol Integration

In order for a [`ManagerWithMerkleVerification`](https://github.com/Se7en-Seas/boring-vault/blob/main/src/base/Roles/ManagerWithMerkleVerification.sol) to manage a BoringVault, it needs a [DecoderAndSanitizer](https://github.com/Se7en-Seas/boring-vault/tree/main/src/base/DecodersAndSanitizers) for every protocol the BoringVault will interact with. The job of the DecoderAndSanitizer is to implement the function selector for every function the BoringVault will need to call. When a function is called, the DecoderAndSanitizer decodes the arguments and possibly sanitize them to return `bytes` containing all the addresses found in the `msg.data` in an `abi.encodePacked` format. The `ManagerWithMerkleVerification`, then takes this `bytes` and uses it to verify a Merkle proof.

Below is a tutorial for how to create the DecoderAndSanitizer for a Uniswap V3 Integration. The fully implemented DecoderAndSanitizer can be found [here](https://github.com/Se7en-Seas/boring-vault/blob/main/src/base/DecodersAndSanitizers/Protocols/UniswapV3DecoderAndSanitizer.sol).

### Step 1: Determine what functions need to be callable by the BoringVault

For a Uniswap V3 Integration, the boring vault should be able to

* Swap
* Create new positions
* Add liquidity to existing positions
* Remove liquidity from positions
* Collect fees from positions

Thus the _UniswapV3DecoderAndSanitizer_ must implement the following functions.

| BoringVault Action                            | Function to implement                                                         |
| --------------------------------------------- | ----------------------------------------------------------------------------- |
| Swap with Uniswap V3                          | exactInput(DecoderCustomTypes.ExactInputParams calldata params)               |
| Create an Uniswap V3 liquidity position       | mint(DecoderCustomTypes.MintParams calldata params)                           |
| Add to an Uniswap V3 liquidity position       | increaseLiquidity(DecoderCustomTypes.IncreaseLiquidityParams calldata params) |
| Take from an Uniswap V3 liquidity position    | decreaseLiquidity(DecoderCustomTypes.DecreaseLiquidityParams calldata params) |
| Collect from an Uniswap V3 liquidity position | collect(DecoderCustomTypes.CollectParams calldata params)                     |

### Step 2: Implementing the functions

All DecoderAndSanitizer function implementations _will_ follow these specifications.

* The implemented function selector _must match_ the underlying protocols function selector.
*   The implemented function must follow the form.

    ```solidity
    function protocolFunctionName(/*PROTOCOL FUNCTION ARGUMENTS*/) 
    	external
      view /*pure is also acceptable*/
      virtual
      returns (bytes memory addressesFound);
    ```

All DecoderAndSanitizer function implementations _may_ follow these specifications.

* Can revert if a sanitation check fails.
* Can return an empty `bytes` if there are no address arguments.
* Can further decode non address arguments, if there is an address in them.
  * Example:
    * `bytes` arguments can have abi encoded addresses in them
    * `bytes32` arguments could be used to derive an address, as is the case with [balancer pool ids](https://etherscan.io/address/0x1e19cf2d73a72ef1332c882f20534b6519be0276#readContract#F15)

The function body itself should extract all addresses from the input arguments and sanitize _any_ arguments that need it.

💡 Sanitizing an argument means to revert if a specific argument input should not be allowed. For instance, the [_MorphoBlueDecoderAndSanitizer_](https://github.com/Se7en-Seas/boring-vault/blob/main/src/base/DecodersAndSanitizers/Protocols/MorphoBlueDecoderAndSanitizer.sol) will revert if the `bytes calldata data` argument has a non zero length, as BoringVaults do not implement the required MorphoBlue callback functions.

Implementing `exactInput` results in the following code.

```solidity
    function exactInput(DecoderCustomTypes.ExactInputParams calldata params)
        external
        pure
        virtual
        returns (bytes memory addressesFound)
    {
        // Nothing to sanitize
        // Return addresses found
        // Determine how many addresses are in params.path.
        uint256 chunkSize = 23; // 3 bytes for uint24 fee, and 20 bytes for address token
        uint256 pathLength = params.path.length;
        if (pathLength % chunkSize != 20) revert UniswapV3DecoderAndSanitizer__BadPathFormat();
        uint256 pathAddressLength = 1 + (pathLength / chunkSize);
        uint256 pathIndex;
        for (uint256 i; i < pathAddressLength; ++i) {
            addressesFound = abi.encodePacked(addressesFound, params.path[pathIndex:pathIndex + 20]);
            pathIndex += chunkSize;
        }
        addressesFound = abi.encodePacked(addressesFound, params.recipient);
    }
```

💡 The `params.path` argument contains an abi encode packed sequence of `(TOKEN_0_ADDRESS, FEE_0, TOKEN_1_ADRESS, FEE_1,…… TOKEN_N_ADDRESS)` the implementation must iterate through this data, and extract every token address in it.

Implementing `mint` results in the following code.

```solidity
    function mint(DecoderCustomTypes.MintParams calldata params)
        external
        pure
        virtual
        returns (bytes memory addressesFound)
    {
        // Nothing to sanitize
        // Return addresses found
        addressesFound = abi.encodePacked(params.token0, params.token1, params.recipient);
    }
```

Implementing `increaseLiquidity` results in the following code.

```solidity
    function increaseLiquidity(DecoderCustomTypes.IncreaseLiquidityParams calldata params)
        external
        view
        virtual
        returns (bytes memory addressesFound)
    {
        // Sanitize raw data
        if (uniswapV3NonFungiblePositionManager.ownerOf(params.tokenId) != boringVault) {
            revert UniswapV3DecoderAndSanitizer__BadTokenId();
        }
        // No addresses in data
        return addressesFound;
    }
```

💡 The `params.tokenId` argument must be _sanitized_ to check that the manager is not trying to add liquidity to a token id not owned by the BoringVault.

💡 We add `uniswapV3NonFungiblePositionManager` as an immutable constructor value for this DecoderAndSanitizer, so we can run `ownerOf` checks.

Implementing `decreaseLiquidity` results in the following code.

```solidity
    function decreaseLiquidity(DecoderCustomTypes.DecreaseLiquidityParams calldata params)
        external
        view
        virtual
        returns (bytes memory addressesFound)
    {
        // Sanitize raw data
        // NOTE ownerOf check is done in PositionManager contract as well, but it is added here
        // just for completeness.
        if (uniswapV3NonFungiblePositionManager.ownerOf(params.tokenId) != boringVault) {
            revert UniswapV3DecoderAndSanitizer__BadTokenId();
        }

        // No addresses in data
        return addressesFound;
    }
```

💡 The `params.tokenId` argument is _sanitized_ to check that the manager is not trying to remove liquidity from a token id not owned by the BoringVault. This not a strict security requirement, rather we do it just to close scope.

Implementing `collect` results in the following code.

```solidity
    function collect(DecoderCustomTypes.CollectParams calldata params)
        external
        view
        virtual
        returns (bytes memory addressesFound)
    {
        // Sanitize raw data
        // NOTE ownerOf check is done in PositionManager contract as well, but it is added here
        // just for completeness.
        if (uniswapV3NonFungiblePositionManager.ownerOf(params.tokenId) != boringVault) {
            revert UniswapV3DecoderAndSanitizer__BadTokenId();
        }

        // Return addresses found
        addressesFound = abi.encodePacked(params.recipient);
    }
```

💡 The `params.tokenId` argument is _sanitized_ to check that the manager is not trying to collect from a token id not owned by the BoringVault. This not a strict security requirement, rather we do it just to close scope.

### Step 3: Integrating UniswapV3DecoderAndSanitizer into a BoringVault specific DecoderAndSanitizer

💡 Combining the several DecodersAndSanitizers into a single DecodersAndSanitizer is _not_ required. Rather, it is a gas optimization so manage calls make less calls to cold addresses. It is perfectly acceptable to have multiple different DecodersAndSanitizers contracts.

It is possible for other protocol DecoderAndSanitizers to implement the exact same function selectors, which causes compiler errors. This should be addresses in the BoringVault specific DecoderAndSanitizer.

Example The BalancerV2, ERC4626, and Curve DecoderAndSanitizers all implement the `deposit(uint256,address)` function. Since they all use the exact same function body, it can be safely overridden.

```solidity
    /**
     * @notice BalancerV2, ERC4626, and Curve all specify a `deposit(uint256,address)`,
     *         all cases are handled the same way.
     */
    function deposit(uint256, address receiver)
        external
        pure
        override(BalancerV2DecoderAndSanitizer, ERC4626DecoderAndSanitizer, CurveDecoderAndSanitizer)
        returns (bytes memory addressesFound)
    {
        addressesFound = abi.encodePacked(receiver);
    }
```
