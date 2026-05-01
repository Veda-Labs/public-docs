# Teller Reference

`TellerWithMultiAssetSupport` — the single entry point for all deposits and withdrawals into a BoringVault.

---

## Overview

The Teller accepts ERC20 tokens (or native ETH) from users, calculates the correct share amount using the Accountant's exchange rate, and instructs the vault to mint shares. On the way out, it burns shares and releases assets.

**Use it when you want to:**
- Deposit a supported asset and receive vault shares
- Withdraw shares for a supported asset immediately (if enabled)
- Integrate referral tracking into deposits
- Check whether an asset is supported for deposit or withdrawal

**Don't use it to:**
- Withdraw asynchronously from an illiquid vault — use [BoringQueue](./BoringQueue.md) instead
- Read the current exchange rate — query the [Accountant](./Accountant.md) directly
- Execute vault strategies — that's the Manager role

**Mental model:** The Teller is the vault's front desk. It validates your entry, sets a temporary share lock on the way in, and stamps your exit on the way out.

---

## Integration Guide

### Where it fits

```
User
  │
  ▼
Teller.deposit()          ← you call this
  │  checks: not paused, asset allowed, cap not hit
  │  calculates shares via Accountant.getRateInQuoteSafe()
  │
  ▼
BoringVault.enter()       ← Teller calls this
  │  pulls asset from user → vault
  │  mints shares to receiver
  │
  ▼
User receives vault shares (locked for shareLockPeriod)
```

For withdrawals:

```
User
  │
  ▼
Teller.withdraw()
  │  checks: not paused, asset allowed for withdraws
  │  calculates assets out via Accountant.getRateInQuoteSafe()
  │
  ▼
BoringVault.exit()
  │  burns shares from user
  │  transfers asset to destination
  │
  ▼
User receives asset
```

### Key Functions

| Function | Who calls it | What it does |
|---|---|---|
| `deposit` | End user | Deposit ERC20 or ETH, receive shares |
| `depositWithPermit` | End user | Deposit using ERC-2612 permit instead of prior approval |
| `bulkDeposit` | SOLVER_ROLE | Batch deposit on behalf of users |
| `bulkWithdraw` | SOLVER_ROLE | Batch withdraw on behalf of users |
| `withdraw` | End user | Withdraw shares for asset immediately |
| `refundDeposit` | DEPOSIT_REFUNDER_ROLE | Cancel a deposit that is still within its lock period |

**Share calculation:**
```
shares = depositAmount * ONE_SHARE / getRateInQuoteSafe(depositAsset)
```
If the asset has a `sharePremium` configured (in basis points):
```
shares = shares * (10000 - sharePremium) / 10000
```

**Asset calculation on withdrawal:**
```
assetsOut = shareAmount * getRateInQuoteSafe(withdrawAsset) / ONE_SHARE
```

### Auth requirement

`deposit`, `depositWithPermit`, and `withdraw` all carry `requiresAuth`. The constructor sets no default Authority (`Authority(address(0))`), which means only the owner can call these functions out of the box. A production deployment must configure an Authority contract (or set the owner appropriately) that grants access to all intended callers — including regular users. If this is misconfigured, all user-facing calls silently revert with no meaningful error. Confirm the Auth setup before integrating.

### Deposit step-by-step

1. Call `teller.assetData(tokenAddress)` — verify `allowDeposits == true`
2. Approve the **vault** (not the Teller) to spend your deposit token
3. Call `teller.deposit(tokenAddress, amount, minimumSharesOut, referralAddress)`
4. Shares are minted to your address and locked for `shareLockPeriod` seconds
5. After the lock expires, shares are freely transferable

For native ETH: skip the approval, send ETH as `msg.value`, pass `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` as `depositAsset`.

### Withdraw step-by-step

1. Call `teller.assetData(tokenAddress)` — verify `allowWithdraws == true`
2. Confirm your share lock period has expired (`beforeTransferData[yourAddress].shareUnlockTime < block.timestamp`)
3. Call `teller.withdraw(tokenAddress, shareAmount, minimumAssetsOut, recipient)`
4. Shares are burned, assets are sent to `recipient`

### Example (viem)

```typescript
import {
  createPublicClient,
  createWalletClient,
  http,
  parseUnits,
  maxUint256,
} from 'viem'
import { mainnet } from 'viem/chains'

const VAULT_ADDRESS   = '0xVaultAddress'
const TELLER_ADDRESS  = '0xTellerAddress'
const USDC_ADDRESS    = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48'
const USDC_DECIMALS   = 6

const publicClient = createPublicClient({ chain: mainnet, transport: http() })
const walletClient = createWalletClient({ chain: mainnet, transport: http() })

// 1. Check asset is supported
const [assetData] = await publicClient.readContract({
  address: TELLER_ADDRESS,
  abi: TELLER_ABI,
  functionName: 'assetData',
  args: [USDC_ADDRESS],
})
// assetData.allowDeposits must be true

// 2. Approve the VAULT (not the Teller) to spend USDC
await walletClient.writeContract({
  address: USDC_ADDRESS,
  abi: ERC20_ABI,
  functionName: 'approve',
  args: [VAULT_ADDRESS, parseUnits('1000', USDC_DECIMALS)],
})

// 3. Deposit
const depositAmount = parseUnits('1000', USDC_DECIMALS)
// WARNING: never use 0n for minimumShares in production — this exposes the tx to sandwich attacks
// Calculate expected shares off-chain: depositAmount * ONE_SHARE / getRateInQuoteSafe(asset)
// Then apply a slippage tolerance, e.g. 0.1% = multiply by 0.999
const minimumShares = 0n  // replace with a real slippage-protected value

const txHash = await walletClient.writeContract({
  address: TELLER_ADDRESS,
  abi: TELLER_ABI,
  functionName: 'deposit',
  args: [USDC_ADDRESS, depositAmount, minimumShares, '0x0000000000000000000000000000000000000000'],
})

// 4. Withdraw later (after share lock expires)
const shareAmount = parseUnits('990', 18)  // vault share decimals
const minimumAssets = parseUnits('985', USDC_DECIMALS)

await walletClient.writeContract({
  address: TELLER_ADDRESS,
  abi: TELLER_ABI,
  functionName: 'withdraw',
  args: [USDC_ADDRESS, shareAmount, minimumAssets, walletClient.account.address],
})
```

---

## Full Reference

### `deposit`

```solidity
function deposit(
    ERC20 depositAsset,
    uint256 depositAmount,
    uint256 minimumMint,
    address referralAddress
) external payable returns (uint256 shares)
```

Deposits an ERC20 token or native ETH into the vault and mints shares to `msg.sender`.

**Parameters:**
- `depositAsset` — ERC20 token address, or `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` for native ETH
- `depositAmount` — amount of `depositAsset` to deposit (ignored for native ETH; `msg.value` is used instead)
- `minimumMint` — minimum shares to receive; reverts if shares calculated fall below this
- `referralAddress` — tracked in the `Deposit` event; pass `address(0)` if unused

**Returns:** `shares` — shares minted to `msg.sender`

**Emits:** `Deposit(nonce, receiver, depositAsset, depositAmount, shareAmount, depositTimestamp, shareLockPeriod, referralAddress)`

**Reverts:**
- `__Paused` — contract is paused
- `__AssetNotSupported` — `assetData[depositAsset].allowDeposits == false`
- `__ZeroAssets` — `depositAmount == 0` (or `msg.value == 0` for ETH)
- `__MinimumMintNotMet` — calculated shares < `minimumMint`
- `__DepositExceedsCap` — `shares + vault.totalSupply() > depositCap`
- `__DualDeposit` — ERC20 deposit sent with nonzero `msg.value`
- `__TransferDenied` — `msg.sender` or receiver is on a deny list

**Notes:**
- Requires prior `approve(vault, depositAmount)` for ERC20 deposits
- For native ETH: send ETH as `msg.value`, no approval required
- Shares are locked to `msg.sender` for `shareLockPeriod` seconds after minting
- The `requiresAuth` modifier means the contract's Auth configuration must permit `msg.sender` to call this function

---

### `depositWithPermit`

```solidity
function depositWithPermit(
    ERC20 depositAsset,
    uint256 depositAmount,
    uint256 minimumMint,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s,
    address referralAddress
) external returns (uint256 shares)
```

Same as `deposit` but uses an ERC-2612 permit signature to set the vault's allowance in the same transaction. The permit approves the **vault** to spend `depositAsset`.

**Reverts:**
- `__PermitFailedAndAllowanceTooLow` — permit call failed and existing allowance is insufficient
- `__CannotDepositNative` — native ETH is not supported via this function
- All `deposit` reverts apply

---

### `bulkDeposit`

```solidity
function bulkDeposit(
    ERC20 depositAsset,
    uint256 depositAmount,
    uint256 minimumMint,
    address to
) external returns (uint256 shares)
```

Deposits on behalf of a recipient without emitting the public deposit history or setting a share lock. Called by SOLVER_ROLE.

**Parameters:**
- `to` — address that receives the minted shares

**Emits:** `BulkDeposit(asset, depositAmount)`

**Notes:**
- Shares minted via `bulkDeposit` are **immediately transferable** — no share lock is applied
- These deposits are **non-refundable** — no entry is written to `publicDepositHistory`, so `refundDeposit` cannot be used

---

### `bulkWithdraw`

```solidity
function bulkWithdraw(
    ERC20 withdrawAsset,
    uint256 shareAmount,
    uint256 minimumAssets,
    address to
) external returns (uint256 assetsOut)
```

Burns shares from `msg.sender` (the solver) and sends assets to `to`. Called by SOLVER_ROLE as part of the BoringQueue solve flow.

**Emits:** `BulkWithdraw(asset, shareAmount)`

---

### `withdraw`

```solidity
function withdraw(
    ERC20 withdrawAsset,
    uint256 shareAmount,
    uint256 minimumAssets,
    address to
) external returns (uint256 assetsOut)
```

Burns `shareAmount` from `msg.sender` and sends `withdrawAsset` to `to`.

**Parameters:**
- `withdrawAsset` — token to receive
- `shareAmount` — shares to burn
- `minimumAssets` — minimum assets to receive; reverts if below
- `to` — recipient of the withdrawn assets

**Reverts:**
- `__Paused` — contract is paused
- `__AssetNotSupported` — `assetData[withdrawAsset].allowWithdraws == false`
- `__ZeroShares` — `shareAmount == 0`
- `__MinimumAssetsNotMet` — calculated assets < `minimumAssets`
- `__SharesAreLocked` — caller's share lock has not expired
- `__TransferDenied` — caller is on the deny list

**Emits:** `Withdraw(asset, shareAmount)`

---

### `refundDeposit`

```solidity
function refundDeposit(
    uint256 nonce,
    address receiver,
    address depositAsset,
    uint256 depositAmount,
    uint256 shareAmount,
    uint256 depositTimestamp,
    uint256 shareLockUpPeriodAtTimeOfDeposit,
    address referralAddress
) external
```

Cancels a pending deposit and returns assets to the receiver. All parameters must exactly match those recorded at deposit time (they reconstruct the stored `publicDepositHistory` hash).

**Reverts:**
- `__SharesAreUnLocked` — lock period has already elapsed; too late to refund
- `__BadDepositHash` — provided parameters don't match the stored deposit record

**Emits:** `DepositRefunded(nonce, depositHash, user)`

**Notes:**
- The deposit history hash is deleted after refund to prevent replay
- If the original deposit used native ETH, the refund asset is the wrapped native token (WETH)
- Callable by DEPOSIT_REFUNDER_ROLE / STRATEGIST_MULTISIG_ROLE

---

### `beforeTransfer` (hook)

```solidity
function beforeTransfer(address from, address to, address operator) public view
```

Called by BoringVault on every share transfer. Reverts if any of the following are true:
- `from` is on the denyFrom list
- `to` is on the denyTo list
- `operator` is on the denyOperator list
- `permissionedTransfers == true` and `operator` is not a permissioned operator
- `from`'s `shareUnlockTime > block.timestamp`

---

### View functions and state

| Query | What it returns |
|---|---|
| `assetData(address)` | `Asset { allowDeposits, allowWithdraws, sharePremium }` |
| `depositCap()` | Max total shares (uint112). `type(uint112).max` = unlimited |
| `shareLockPeriod()` | Seconds shares are locked after deposit |
| `isPaused()` | Whether the contract is paused. Pausing blocks **both** deposits and withdrawals — despite the contract comments describing it as a deposit-only pause, `_withdraw()` also checks `isPaused` |
| `beforeTransferData(address)` | Deny flags and `shareUnlockTime` for an address |
| `publicDepositHistory(nonce)` | keccak256 hash of deposit parameters for refund validation |
| `depositNonce()` | Current deposit counter |

---

### Internal Behavior

**Approval target:** The vault (not the Teller) pulls the deposit token using `safeTransferFrom(user, vault, amount)`. Users must approve the **vault address**, not the Teller.

**Share lock:** After every public deposit, `beforeTransferData[receiver].shareUnlockTime` is set to `block.timestamp + shareLockPeriod`. All share transfers from that address revert until the lock expires. The lock only applies if `shareLockPeriod > 0`.

**Deposit cap:** Enforced as `shares + vault.totalSupply() > depositCap`. No partial fills — the entire deposit reverts if it would exceed the cap.

**Share premium:** A per-asset haircut on shares minted, expressed in basis points. A premium of 40 means the user receives 0.4% fewer shares. This is used to account for deposit/withdrawal spread on illiquid assets.

**Native ETH flow:** The Teller wraps ETH via WETH, approves the vault, and calls `vault.enter(teller, WETH, amount, receiver, shares)`. The deposit history records the original asset as `NATIVE` but any refund returns WETH.

---

### Edge Cases

**Double-deposit with shorter lock:** If the `shareLockPeriod` is decreased, a user who already has a pending lock can make a new deposit (even 1 wei) and their unlock time resets to the shorter period. Their original deposit becomes refundable via `refundDeposit` as long as the original lock period hasn't expired.

**Permit reverting silently:** `depositWithPermit` catches permit failures and checks the existing allowance. If the allowance is sufficient, the deposit proceeds without a permit. Only reverts with `__PermitFailedAndAllowanceTooLow` if the allowance is also insufficient.

**Zero-value share lock:** If `shareLockPeriod == 0`, deposits do not record to `publicDepositHistory` and shares are immediately transferable. Deposits are also non-refundable.

**Permissioned transfers:** If `permissionedTransfers == true`, only addresses explicitly added as permissioned operators can transfer shares. This affects all transfers, not just deposits.

---

### Common Mistakes

1. **Approving the Teller instead of the Vault.** The Teller never pulls tokens directly. The Vault does. Approve `vault.address`.

2. **Trying to transfer or withdraw shares before the lock expires.** Check `beforeTransferData[yourAddress].shareUnlockTime` before initiating any share movement.

3. **Passing `depositAmount = 0` for native ETH deposits.** The `depositAmount` parameter is ignored for ETH — only `msg.value` matters.

4. **Setting `minimumMint = 0` in production.** This exposes deposits to sandwich attacks. Calculate the expected shares off-chain and apply a slippage tolerance.

5. **Calling `withdraw` when the asset is not enabled for withdrawals.** Check `assetData[asset].allowWithdraws` first. Some assets are deposit-only and require going through the BoringQueue.

---

## Related Contracts

| Contract | Relationship |
|---|---|
| [BoringVault](./BoringVault.md) | Teller calls `vault.enter()` and `vault.exit()` to mint/burn shares |
| [Accountant](./Accountant.md) | Teller calls `accountant.getRateInQuoteSafe(asset)` to price every deposit and withdrawal |
| [BoringQueue](./BoringQueue.md) | Alternative withdrawal path for async exits; calls `teller.bulkWithdraw()` via the solver |
