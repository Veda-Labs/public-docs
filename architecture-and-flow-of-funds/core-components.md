# Core Components

#### BoringVault

The central contract where users deposit assets and where all vault funds are custodied. BoringVault minimizes internal logic by delegating tasks to external modules. It holds the assets, mints and burns share tokens, and executes management calls — but the rules governing _how_ and _when_ those actions happen are enforced by the surrounding modules.

#### Teller

The Teller handles user-facing interactions for depositing into and withdrawing from the vault. When a user deposits, the Teller retrieves the current exchange rate from the Accountant, calculates the shares to mint, and coordinates the transfer of assets into the BoringVault. On withdrawal, it burns shares and returns the corresponding assets.

The Teller enforces share lock periods after deposit to protect against arbitrage and MEV exploitation. During the lock period, shares cannot be transferred or redeemed. Certain vault configurations also allow the Teller to refund deposits if needed, providing an additional safety mechanism.

Vault deployments may use specialized Teller variants depending on the vault's configuration — for example, `TellerWithYieldStreaming` for vaults that stream yield gradually over time, or `TellerWithBuffer` for vaults with instant withdrawal capabilities.

#### Accountant

The Accountant calculates and publishes the exchange rate between vault shares and the underlying assets. Exchange rates are updated offchain and pushed onchain, with built-in safety checks: rate limiting restricts how frequently updates can occur, and deviation bounds constrain how much the rate can move relative to the previous value. If an update falls outside acceptable bounds, the Accountant can pause rate updates to safeguard against volatile market conditions or oracle manipulation.

For vaults with yield streaming enabled, the `AccountantWithYieldStreaming` variant smooths rate changes over time rather than applying them in discrete jumps, providing a more predictable return profile for depositors.

#### Manager

The Manager controls which strategies the vault can execute during rebalancing. It stores all permitted operations in a Merkle tree, where each leaf encodes a specific allowed action: the target contract address, the function to call, and the acceptable parameter values. When the vault executes a strategy, the Manager verifies the operation against its Merkle proof, ensuring that only pre-authorized actions can be performed.

The system supports per-strategist Merkle trees, enabling granular access control. A main strategist may have broad permissions to rebalance across many protocols, while specialized or automated accounts can operate under tighter constraints.

#### Hook

An optional module that triggers custom logic before share transfers occur. Hooks can enforce transfer restrictions, implement whitelisting requirements, lock shares to prevent transfers, or execute other compliance-related checks at the smart contract level.

Common use cases include restricting deposits to approved addresses, preventing share transfers during certain conditions, and implementing regulatory compliance logic directly onchain.

#### DecoderAndSanitizer

When the vault interacts with external DeFi protocols, the DecoderAndSanitizer decodes and validates the calldata for each operation. It ensures that only safe and intended interactions are performed — verifying target addresses, function selectors, and parameter values before the vault executes any external call. Each integrated protocol has its own DecoderAndSanitizer implementation tailored to that protocol's interface.

#### BoringQueue

The BoringQueue (`BoringOnChainQueue`) implements time-delayed, solver-based withdrawals. When a user submits a withdrawal request, their shares are transferred to the queue and the request enters a maturity period. After maturity, third-party solvers can fulfill the request by providing the underlying assets to the user in exchange for the vault shares.

The BoringQueue serves as the standard withdrawal method for vaults without instant withdrawals enabled, and as a fallback when instant withdrawal buffers have insufficient liquidity. Key parameters — including the maturity period, solver deadline window, and discount range — are configurable per asset. Users can cancel unfulfilled requests at any time to reclaim their shares.
