# Accountant Reference

`AccountantWithRateProviders` — tracks the vault's exchange rate, prices deposits and withdrawals across multiple assets, and accumulates protocol fees.

---

## Overview

The Accountant is the vault's pricing oracle. It holds the authoritative exchange rate — how many base-asset tokens one vault share is worth — and knows how to convert that rate into any supported quote token using per-asset rate providers. Every deposit and withdrawal routes through this contract to determine how many shares to mint or how many tokens to return.

**Use it when you want to:**
- Read the current share price in the base asset or any supported quote token
- Check whether the Accountant is paused before attempting a deposit or withdrawal
- Preview how a rate update would affect fees
- Build off-chain tooling that shows share NAV or pending fees

**Don't use it to:**
- Deposit or withdraw — use the [Teller](./Teller.md) instead
- Trigger exchange rate updates — that's the UPDATE_EXCHANGE_RATE_ROLE
- Claim fees — `claimFees()` must be called by the vault itself

**Mental model:** The Accountant is a pricing engine and fee ledger. It answers one question: "what is one share worth right now, in asset X?" — and tracks what the protocol is owed as a result of that value growing.

---

## Integration Guide

### Where it fits

```
Accountant.updateExchangeRate()  ← called periodically by keeper
         │
         │  stores: exchangeRate, feesOwedInBase, highwaterMark
         │
         ▼
Accountant.getRateInQuoteSafe(asset)  ← called by Teller and BoringQueue
         │
         │  returns: how many `asset` tokens = 1 share
         │
         ▼
Teller uses rate to calculate shares on deposit:
    shares = depositAmount * ONE_SHARE / rateInQuote

Teller uses rate to calculate assets on withdrawal:
    assetsOut = shareAmount * rateInQuote / ONE_SHARE
```

### Key read functions

| Function | What it returns |
|---|---|
| `getRate()` | Share price in the base asset (raw, never reverts) |
| `getRateSafe()` | Same, but reverts if paused |
| `getRateInQuote(quote)` | Share price expressed in `quote` token units |
| `getRateInQuoteSafe(quote)` | Same, reverts if paused |
| `previewUpdateExchangeRate(newRate)` | Whether a rate update would pause the contract and what fees it would generate |

### Checking if the Accountant is paused

Before any deposit or withdrawal, the Teller calls `getRateInQuoteSafe()`. If the Accountant is paused, that call reverts, and the deposit or withdrawal reverts too. You can check this directly:

```typescript
const state = await publicClient.readContract({
  address: ACCOUNTANT_ADDRESS,
  abi: ACCOUNTANT_ABI,
  functionName: 'accountantState',
})
// state.isPaused === true means all deposits/withdrawals are currently blocked
```

### Reading share price in a specific asset

```typescript
import { createPublicClient, http } from 'viem'
import { mainnet } from 'viem/chains'

const publicClient = createPublicClient({ chain: mainnet, transport: http() })
const ACCOUNTANT_ADDRESS = '0xAccountantAddress'
const USDC_ADDRESS       = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48'

// Share price in USDC (6 decimals)
// Returns: how many USDC wei = 1 vault share
const rateInUSDC = await publicClient.readContract({
  address: ACCOUNTANT_ADDRESS,
  abi: ACCOUNTANT_ABI,
  functionName: 'getRateInQuoteSafe',
  args: [USDC_ADDRESS],
})

// Share price in the base asset
const rateInBase = await publicClient.readContract({
  address: ACCOUNTANT_ADDRESS,
  abi: ACCOUNTANT_ABI,
  functionName: 'getRateSafe',
})

// Full accountant state
const {
  exchangeRate,
  highwaterMark,
  feesOwedInBase,
  isPaused,
  platformFee,
  performanceFee,
  lastUpdateTimestamp,
} = await publicClient.readContract({
  address: ACCOUNTANT_ADDRESS,
  abi: ACCOUNTANT_ABI,
  functionName: 'accountantState',
})
```

---

## Full Reference

### `getRate`

```solidity
function getRate() public view returns (uint256 rate)
```

Returns the current exchange rate: how many base-asset tokens (in base-asset decimals) equal one vault share. Never reverts regardless of pause state.

---

### `getRateSafe`

```solidity
function getRateSafe() external view returns (uint256 rate)
```

Same as `getRate()` but reverts if the Accountant is paused.

**Reverts:** `__Paused`

---

### `getRateInQuote`

```solidity
function getRateInQuote(ERC20 quote) public view returns (uint256 rateInQuote)
```

Returns the exchange rate denominated in `quote` tokens.

**If `quote == base`:** returns `exchangeRate` directly.

**If `quote` is pegged to base (e.g., USDC and USDT both pegged to USD):** returns `exchangeRate` adjusted for `quote`'s decimals.

**Otherwise:** uses the per-asset `rateProvider` to convert:
```
rateInQuote = (1 quote token in quote decimals) * exchangeRate_in_quote_decimals / quoteRate
```
where `quoteRate` is the rate provider's output (how many base tokens = 1 quote token, in quote token decimals).

**Reverts:** if `quote` has no `RateProviderData` configured and is not the base asset.

---

### `getRateInQuoteSafe`

```solidity
function getRateInQuoteSafe(ERC20 quote) external view returns (uint256 rateInQuote)
```

Same as `getRateInQuote()` but reverts if paused. This is what the Teller and BoringQueue call on every deposit and withdrawal.

**Reverts:** `__Paused`

---

### `updateExchangeRate`

```solidity
function updateExchangeRate(uint96 newExchangeRate) external
```

Updates the stored exchange rate. Called periodically by a keeper with UPDATE_EXCHANGE_RATE_ROLE.

**Behavior:**
- If the new rate is within the allowed bounds and enough time has passed since the last update: updates the rate and calculates fees owed
- If the new rate is outside bounds or the minimum delay hasn't elapsed: pauses the Accountant and stores the rate anyway (without calculating fees)

**Pause conditions — any one triggers a pause:**
- `newExchangeRate > currentRate * allowedExchangeRateChangeUpper / 1e4`
- `newExchangeRate < currentRate * allowedExchangeRateChangeLower / 1e4`
- `block.timestamp < lastUpdateTimestamp + minimumUpdateDelayInSeconds`

**Emits:** `ExchangeRateUpdated(oldRate, newRate, timestamp)`

**Notes:**
- A paused Accountant blocks all deposits and withdrawals via `getRateInQuoteSafe`
- Admin must call `unpause()` to resume after investigating the anomalous rate
- Fee calculation happens only on non-pausing updates
- `newExchangeRate == 0` is **not validated**. A zero rate would pass the bounds check if `allowedExchangeRateChangeLower` is 0, and would price all deposits at infinite shares. Keepers must validate the rate value off-chain before submitting.

---

### `claimFees`

```solidity
function claimFees(ERC20 feeAsset) external
```

Transfers accumulated fees from the vault to `accountantState.payoutAddress`, denominated in `feeAsset`.

**Access:** Only callable by the BoringVault itself (not directly by users or admins)

**Parameters:**
- `feeAsset` — the token to claim fees in. Must have `RateProviderData` configured or be the base asset.

**Reverts:**
- `__OnlyCallableByBoringVault` — caller is not the vault
- `__Paused`
- `__ZeroFeesOwed` — no fees have accrued yet

**Emits:** `FeesClaimed(feeAsset, amount)`

**Notes:**
- Fees are stored internally in base-asset units (`feesOwedInBase`)
- The conversion to `feeAsset` units is done at claim time using the current rate
- `feesOwedInBase` is zeroed after claiming

---

### `previewUpdateExchangeRate`

```solidity
function previewUpdateExchangeRate(uint96 newExchangeRate)
    external
    view
    returns (
        bool updateWillPause,
        uint256 newFeesOwedInBase,
        uint256 totalFeesOwedInBase
    )
```

Previews the effect of updating the exchange rate without writing any state. Useful for off-chain keepers before submitting an update.

**Returns:**
- `updateWillPause` — true if this update would trigger a pause
- `newFeesOwedInBase` — additional fees this update would generate (0 if pausing)
- `totalFeesOwedInBase` — total accumulated fees after this update

---

### `setRateProviderData`

```solidity
function setRateProviderData(ERC20 asset, bool isPeggedToBase, address rateProvider) external
```

Configures how the Accountant prices a given asset relative to the base.

**Access:** OWNER_ROLE

**Parameters:**
- `asset` — the ERC20 to configure
- `isPeggedToBase` — if true, the asset is treated as 1:1 with base (only decimal-adjusted); `rateProvider` is ignored
- `rateProvider` — contract implementing `IRateProvider.getRate()` that returns how many base tokens = 1 `asset` token

**Emits:** `RateProviderUpdated(asset, isPegged, rateProvider)`

**Notes:**
- An asset must have `RateProviderData` set before the Teller can use it for deposits/withdrawals
- Rate providers must return rates in the same decimals as `asset`
- The contract does **not** validate that `rateProvider.getRate()` returns a nonzero value. A broken or freshly deployed rate provider that returns 0 would cause division-by-zero in `getRateInQuote`, making that asset's deposits and withdrawals revert. Verify the rate provider returns a live, nonzero value before calling `setRateProviderData`.

---

### `AccountantState` fields

```solidity
struct AccountantState {
    address payoutAddress;               // where claimFees sends fees
    uint96  highwaterMark;               // highest exchange rate ever recorded
    uint128 feesOwedInBase;              // accumulated unpaid fees in base token units
    uint128 totalSharesLastUpdate;       // vault.totalSupply() at last rate update
    uint96  exchangeRate;                // current share price in base token (96-bit)
    uint16  allowedExchangeRateChangeUpper; // max upward change per update, in bps (e.g. 10100 = +1%)
    uint16  allowedExchangeRateChangeLower; // min downward change per update, in bps (e.g. 9900 = -1%)
    uint64  lastUpdateTimestamp;         // block.timestamp of last update
    bool    isPaused;
    uint24  minimumUpdateDelayInSeconds; // min seconds between non-pausing updates (max 14 days)
    uint16  platformFee;                 // annual fee in bps (max 2000 = 20%)
    uint16  performanceFee;              // fee on yield above highwaterMark in bps (max 5000 = 50%)
}
```

Read in full with `accountantState()`.

---

### Fee mechanics

**Platform fee** — a time-weighted annual fee charged on AUM regardless of performance.

```
shareSupply    = min(totalSharesLastUpdate, currentTotalShares)
minimumAssets  = shareSupply * min(newRate, currentRate) / ONE_SHARE
annualFee      = minimumAssets * platformFee / 1e4
platformFee    = annualFee * timeDelta / 365 days
```

**Performance fee** — charged only when the exchange rate exceeds the `highwaterMark`.

```
yieldEarned        = (newExchangeRate - highwaterMark) * shareSupply / ONE_SHARE
performanceFee     = yieldEarned * performanceFee / 1e4
```

When performance fees accrue, `highwaterMark` advances to `newExchangeRate` so the same yield is never double-charged.

Both fees are denominated in base-asset units and accumulated in `feesOwedInBase`.

---

### Internal Behavior

**Pause is self-triggering.** The Accountant pauses itself when it detects a rate anomaly — it doesn't just revert. This means the rate is still updated to the new value (even an anomalous one), but fee calculation is skipped and all `Safe` view functions start reverting. A human must investigate and call `unpause()`.

**Decimal normalization.** The `exchangeRate` is stored in base-asset decimals. When converting to a quote with different decimals, the Accountant adjusts using `_changeDecimals(amount, fromDecimals, toDecimals)`. Precision loss occurs if the base has more decimals than the quote.

**`highwaterMark` never decreases automatically.** It only moves up when a new ATH exchange rate is recorded. Admins can call `resetHighwaterMark()` to reset it to the current rate — but only if the current rate is below the mark (e.g., after a drawdown).

**`totalSharesLastUpdate` is used as a fee base.** On each update, the minimum of current and last share supply is used to prevent gaming by minting/burning shares between updates.

---

### Edge Cases

**Rate update too soon.** If `block.timestamp < lastUpdateTimestamp + minimumUpdateDelayInSeconds`, the update goes through but triggers a pause. The keeper should wait the minimum delay before updating again.

**Platform fee eats all yield.** For `AccountantWithFixedRate`, if platform + performance fees exceed the yield earned above the fixed rate, the platform fee is forfeited and only the performance fee is charged.

**Quote asset with lower decimals than base.** Precision loss occurs in the decimal normalization step. For example, if base is an 18-decimal token and quote is a 6-decimal token, any rate precision below `1e-6` is lost. For USDC or USDT as the deposit asset, this means share calculations in the Teller are slightly truncated. The effect is small per transaction but compounds at scale. Off-chain previews should use the same `_changeDecimals` logic to match on-chain behavior.

**`claimFees` requires a vault-initiated call.** Admins cannot directly call `claimFees`. The vault must call it via `manage()`, which is gated to the Manager role. The operational path is:
```solidity
bytes memory data = abi.encodeWithSelector(Accountant.claimFees.selector, feeAssetAddress);
vault.manage(accountantAddress, data, 0);
```
This must come from an account with MANAGER_ROLE on the vault.

**`resetHighwaterMark` blocked when rate is above the mark.** It reverts with `__ExchangeRateAboveHighwaterMark`. This prevents resetting the mark to avoid performance fees that would otherwise be owed.

---

### Common Mistakes

1. **Calling `getRateInQuote` on an asset with no rate provider configured.** This reverts. Call `rateProviderData(asset)` first to confirm it's been set up.

2. **Comparing `getRateInQuote(USDC)` with `getRate()` and expecting the same number.** The base rate is in base-asset decimals; the USDC rate is in 6 decimals. They are numerically different representations of the same price.

3. **Assuming a paused Accountant means the vault is broken.** A pause is a safety circuit, not an error. The vault resumes normally once the Accountant is unpaused.

4. **Using `getRate()` instead of `getRateInQuoteSafe(asset)` to calculate deposit amounts.** The Teller uses `getRateInQuoteSafe(depositAsset)` — use the same call to predict share amounts off-chain. For non-base assets, the rate from `getRate()` gives you the wrong number.

5. **Not accounting for fees when calculating share NAV.** `feesOwedInBase` represents value that will leave the vault when fees are claimed. A fully accurate NAV calculation must account for this pending outflow.

---

## Related Contracts

| Contract | Relationship |
|---|---|
| [Teller](./Teller.md) | Calls `getRateInQuoteSafe(depositAsset)` and `getRateInQuoteSafe(withdrawAsset)` on every deposit and withdrawal to determine share counts |
| [BoringQueue](./BoringQueue.md) | Calls `getRateInQuoteSafe(assetOut)` at withdrawal request time to lock in the `amountOfAssets` |
| [BoringVault](./BoringVault.md) | `claimFees()` must be initiated via `vault.manage()` — the vault is the only authorized caller |
