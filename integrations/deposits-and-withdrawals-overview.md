# Deposits & Withdrawals Overview

This page explains how funds flow into and out of Veda vaults at a high level. All deposits and withdrawals are executed onchain through Veda's smart contracts — your assets are always held in the BoringVault contract or in the DeFi positions it manages, never in an offchain system.

### Deposits

#### How Deposits Work

When you deposit into a Veda vault, you interact with two contracts: the **BoringVault** (which will hold your assets) and the **Teller** (which processes the deposit and mints your shares).

The flow is:

1. **Approve** — You approve the BoringVault contract to spend your tokens. (Note: the approval target is the BoringVault address, not the Teller.)
2. **Deposit** — You call `teller.deposit()` with your chosen asset and amount. The Teller retrieves the current exchange rate from the Accountant, calculates how many vault shares to mint, and coordinates the transfer.
3. **Receive shares** — Your tokens move into the BoringVault, and vault shares are minted to your wallet. These shares represent your proportional claim on the vault's assets plus any accrued yield.

For tokens that support EIP-2612 permits, you can combine the approval and deposit into a single transaction using `teller.depositWithPermit()`, saving gas and simplifying the user experience.

#### After You Deposit

**Share lock period** — After depositing, your shares may be locked for a configurable period (depending on the vault). During this time, shares cannot be transferred or redeemed. This protects the vault and other depositors against arbitrage and MEV exploitation.

**Automatic buffer routing** — In vaults with instant withdrawals enabled, your deposited assets may be automatically routed into yield-generating positions (such as Aave V3 lending pools) within the same transaction. This is handled by onchain BufferHelper contracts that generate the necessary instructions — importantly, BufferHelpers never hold funds themselves.

**Slippage protection** — The `minimumMint` parameter on deposits protects you from receiving fewer shares than expected if the exchange rate changes between when you prepare your transaction and when it executes. A 0.1% tolerance is recommended for most use cases.

#### Checking Deposit Eligibility

Before depositing, you can verify that an asset is supported by checking `teller.assetData(tokenAddress)` to confirm `allowDeposits` is true, or by calling `lens.checkUserDeposit()` for a comprehensive eligibility check.

***

### Withdrawals

Veda vaults support two withdrawal methods. Which method to use depends on the vault's configuration and current liquidity conditions.

| Condition                                                             | Recommended Method         |
| --------------------------------------------------------------------- | -------------------------- |
| Vault has instant withdrawals enabled AND sufficient buffer liquidity | **Instant Withdrawal**     |
| Vault uses standard configuration OR buffer liquidity is insufficient | **BoringQueue Withdrawal** |

#### Instant Withdrawals

For vaults with instant withdrawals enabled, you can withdraw funds immediately in a single atomic transaction — no waiting for solvers or settlement periods.

When you call `teller.withdraw()`, the vault checks the available buffer liquidity for your requested asset. If sufficient liquidity exists, the transaction completes atomically: your shares are burned, the buffer position is exited if needed, and the underlying assets arrive in your wallet within the same transaction.

You can check available instant liquidity before attempting a withdrawal using the **BufferLens** contract, which reports the withdrawable amount for each asset based on the current buffer configuration.

If your withdrawal amount exceeds available buffer liquidity, use the BoringQueue method instead.

#### BoringQueue Withdrawals

The BoringQueue is the standard withdrawal method for vaults without instant withdrawals, and serves as the fallback when instant withdrawal buffers don't have enough liquidity.

BoringQueue withdrawals are a two-step process:

1. **Request** — You approve the BoringQueue contract to spend your vault shares, then call `queue.requestOnChainWithdraw()` specifying the asset you want to receive, the number of shares to redeem, a discount (in basis points), and a deadline window. Your shares transfer to the queue.
2. **Fulfillment** — After a maturity period (typically \~1 hour, configurable per asset), third-party solvers can fulfill your request. The solver provides the underlying assets directly to your wallet in exchange for the vault shares held in the queue.

You don't need to take any action after submitting your request — once a solver fulfills it, assets are sent directly to you. If the request is not fulfilled within the deadline window, it expires and you can cancel to reclaim your shares.

**Key parameters** — You can query `queue.withdrawAssets(assetAddress)` to get the recommended values for each asset, including the maturity period, minimum deadline, allowed discount range, and minimum share amount.

**Cancellation** — You can cancel a pending withdrawal request at any time before it's fulfilled by calling `queue.cancelOnChainWithdraw()`, which returns your shares to your wallet.



## Compatibility

**Wallet & Provider Compatibility** — Veda's deposit and withdrawal contracts use standard ERC-20 approve/transfer patterns and are compatible with any wallet or web3 provider that supports EVM transactions, including Privy, MetaMask, WalletConnect, Coinbase Wallet, Safe multisig, RainbowKit, and libraries like wagmi and viem.
