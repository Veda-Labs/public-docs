# BoringVault Reference

`BoringVault` — the core custody contract. Holds all protocol assets and issues ERC20 shares.

---

## Overview

The BoringVault is a minimal, auth-gated token vault. It holds every asset the protocol manages, mints shares when assets come in, burns shares when assets go out, and executes arbitrary calls against external protocols on behalf of the vault's strategy.

**Use it when you want to:**
- Read a user's share balance or the total share supply
- Verify the vault's address before setting approvals
- Transfer vault shares directly (e.g., as a permissioned operator)
- Understand how the protocol executes strategy calls

**Don't use it to:**
- Deposit or withdraw directly — use the [Teller](./Teller.md) instead
- Read the share price — query the [Accountant](./Accountant.md) instead
- Submit withdrawal requests — use [BoringQueue](./BoringQueue.md)

**Mental model:** The BoringVault is a locked safe with a very specific set of keys. The Teller holds the deposit/withdraw key. The Manager holds the strategy key. Nobody else can move assets.

---

## Integration Guide

### Where it fits

```
External users
      │
      ▼
   Teller ──────────────────────────► BoringVault.enter() → mint shares
   Teller ◄──────────────────────── BoringVault.exit()  → burn shares

   Manager ─────────────────────────► BoringVault.manage() → execute strategy

   BoringQueue / BoringSolver ──────► BoringVault ERC20 transfers (shares move)
```

As an integrator, you will almost never call BoringVault directly. You interact with the Teller for deposits and withdrawals. The vault is relevant to you primarily as the **approval target** for deposit tokens and the **ERC20 token address** for vault shares.

### Key addresses to know

| What | Where |
|---|---|
| Approve token for deposit | `vault.address` — the Teller's `vault()` immutable |
| Vault share ERC20 | `vault.address` — the vault itself is the share token |
| Approve queue for share withdrawal | `vault.address` |

### Reading vault state

```typescript
import { createPublicClient, http } from 'viem'
import { mainnet } from 'viem/chains'

const publicClient = createPublicClient({ chain: mainnet, transport: http() })
const VAULT_ADDRESS = '0x...' as const
const USER_ADDRESS  = '0x...' as const

const BORING_VAULT_ABI = [
  {
    name: 'balanceOf',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'account', type: 'address' }],
    outputs: [{ name: '', type: 'uint256' }],
  },
  {
    name: 'totalSupply',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint256' }],
  },
  {
    name: 'decimals',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint8' }],
  },
  {
    name: 'hook',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'address' }],
  },
] as const

// Share balance
const shares = await publicClient.readContract({
  address: VAULT_ADDRESS,
  abi: BORING_VAULT_ABI,
  functionName: 'balanceOf',
  args: [USER_ADDRESS],
})

// Total shares in circulation
const totalSupply = await publicClient.readContract({
  address: VAULT_ADDRESS,
  abi: BORING_VAULT_ABI,
  functionName: 'totalSupply',
})

// Vault share decimals (used to calculate ONE_SHARE = 10n ** decimals)
const decimals = await publicClient.readContract({
  address: VAULT_ADDRESS,
  abi: BORING_VAULT_ABI,
  functionName: 'decimals',
})

// Current before-transfer hook (usually the Teller)
const hook = await publicClient.readContract({
  address: VAULT_ADDRESS,
  abi: BORING_VAULT_ABI,
  functionName: 'hook',
})
```

---

## Full Reference

### `enter`

```solidity
function enter(
    address from,
    ERC20 asset,
    uint256 assetAmount,
    address to,
    uint256 shareAmount
) external
```

Pulls `assetAmount` of `asset` from `from` into the vault, then mints `shareAmount` shares to `to`. If `assetAmount == 0`, no token transfer happens — only shares are minted.

**Access:** MINTER_ROLE (held by the Teller)

**Parameters:**
- `from` — address tokens are pulled from; must have approved the vault for `assetAmount`
- `asset` — ERC20 token to accept
- `assetAmount` — amount of `asset` to transfer in
- `to` — address that receives the minted shares
- `shareAmount` — number of shares to mint

**Emits:** `Enter(from, asset, assetAmount, to, shareAmount)`

**Notes:**
- The Teller pre-calculates `shareAmount` using the Accountant's exchange rate before calling this
- `from` and `to` can differ — this is how the Teller deposits on behalf of a user
- The vault does **not** validate that `shareAmount > 0`. Minting zero shares while pulling real tokens is possible if the caller (Teller) supplies a zero value. This would burn user funds without issuing any shares. A correctly configured Teller prevents this via the `__ZeroAssets` / `__MinimumMintNotMet` checks before calling `enter`.

---

### `exit`

```solidity
function exit(
    address to,
    ERC20 asset,
    uint256 assetAmount,
    address from,
    uint256 shareAmount
) external
```

Burns `shareAmount` shares from `from`, then sends `assetAmount` of `asset` to `to`. If `assetAmount == 0`, no token transfer happens — only shares are burned.

**Access:** BURNER_ROLE (held by the Teller)

**Parameters:**
- `to` — address that receives the withdrawn assets
- `asset` — ERC20 token to send out
- `assetAmount` — amount to transfer out
- `from` — address whose shares are burned
- `shareAmount` — number of shares to burn

**Emits:** `Exit(to, asset, assetAmount, from, shareAmount)`

**Notes:**
- Unlike ERC20 transfers, burning shares from `from` does not require `from` to have approved the vault — the vault calls `_burn` directly on its own internal balances
- The Teller pre-calculates `assetAmount` using the Accountant before calling this

---

### `manage` (single call)

```solidity
function manage(
    address target,
    bytes calldata data,
    uint256 value
) external returns (bytes memory result)
```

Executes an arbitrary call from the vault to `target` with `data` and `value` ETH. Used by the Manager to interact with external DeFi protocols (e.g., supply assets to Aave, provide liquidity to Uniswap).

**Access:** MANAGER_ROLE

**Returns:** raw `bytes` return value from the call

---

### `manage` (batch)

```solidity
function manage(
    address[] calldata targets,
    bytes[] calldata data,
    uint256[] calldata values
) external returns (bytes[] memory results)
```

Executes multiple calls in a single transaction. All arrays must have the same length.

**Access:** MANAGER_ROLE

---

### `setBeforeTransferHook`

```solidity
function setBeforeTransferHook(address _hook) external
```

Sets the contract that receives a callback before every share transfer. The hook is called as:

```solidity
hook.beforeTransfer(from, to, msg.sender)
```

In a standard deployment, the Teller is set as the hook, enforcing share locks and deny lists on all share movements.

**Access:** OWNER_ROLE

**Notes:**
- Setting `_hook` to `address(0)` disables the hook entirely — shares become freely transferable with no restrictions
- The hook is called on `transfer()` and `transferFrom()` but not on `_mint` or `_burn`

---

### ERC20 functions

The vault is itself an ERC20. Standard functions — `transfer`, `transferFrom`, `approve`, `permit`, `balanceOf`, `totalSupply`, `allowance`, `decimals`, `name`, `symbol` — all work as expected.

Every `transfer` and `transferFrom` calls the `beforeTransfer` hook (if set) before executing.

```solidity
function transfer(address to, uint256 amount) public override returns (bool)
function transferFrom(address from, address to, uint256 amount) public override returns (bool)
```

Both internally call `_callBeforeTransfer(from, to)` which invokes `hook.beforeTransfer(from, to, msg.sender)`.

---

### Internal Behavior

**The vault does not track which assets it holds.** It has no internal registry of what tokens are deposited. The Accountant determines the exchange rate based on off-chain strategy reporting. The vault simply holds whatever tokens are sent to it.

**`manage()` is fully unconstrained at the vault level.** The vault will execute any call the Manager sends. Access control over which calls are allowed is enforced by the Decoder/Sanitizer layer in the Manager contract — not in the vault itself.

**`enter` and `exit` do not validate the share price or amounts.** The vault trusts the Teller to have already checked the exchange rate via the Accountant. A misconfigured or malicious Teller with MINTER_ROLE could call `enter` with any `shareAmount`, including zero — the vault would execute it without complaint. The security boundary is entirely in who holds MINTER_ROLE and BURNER_ROLE.

**The vault can receive ETH.** A `receive()` function is present, so ETH sent directly to the vault is accepted. This is used when strategy calls return ETH.

**ERC721 and ERC1155 support.** The vault implements `ERC721Holder` and `ERC1155Holder`, so it can receive NFTs and multi-tokens from strategy positions.

---

### Edge Cases

**Shares can be minted without transferring assets.** If `assetAmount == 0` in `enter()`, no tokens are pulled but shares are still minted. This is used for initialization or cross-chain bridging scenarios.

**Assets can be transferred out without burning shares.** If `assetAmount == 0` in `exit()`, no tokens leave but shares are still burned. Used for specific accounting scenarios.

**Hook set to address(0).** If `setBeforeTransferHook(address(0))` is called, all share transfers become unrestricted. The share lock, deny list, and permissioned transfer controls from the Teller are bypassed entirely.

**Reentrancy.** The vault does not have a reentrancy guard. The Teller and other callers are expected to guard their own flows. The `manage()` function can trigger callbacks into external contracts.

---

### Common Mistakes

1. **Approving the Teller instead of the Vault.** Deposit tokens must be approved to the **vault** address. The Teller calls `vault.enter()`, which calls `safeTransferFrom(user, vault, amount)`.

2. **Approving the wrong address for queue withdrawals.** Vault shares must be approved to the **BoringQueue** address, not the Teller.

3. **Treating vault shares as a wrapped asset with 1:1 redemption.** Share value grows over time as the exchange rate increases. `1 share ≠ 1 base token` after any yield has accrued.

4. **Calling `enter` or `exit` directly.** These are auth-gated to specific roles. Direct calls from user addresses will revert.

5. **Triggering `claimFees` directly.** `Accountant.claimFees()` enforces `msg.sender == vault`. It cannot be called by an EOA or admin directly. The operational path is: construct the `claimFees(feeAsset)` calldata, then submit it via `vault.manage(accountantAddress, calldata, 0)` from an account holding MANAGER_ROLE.

---

## Related Contracts

| Contract | Relationship |
|---|---|
| [Teller](./Teller.md) | Holds MINTER_ROLE and BURNER_ROLE; calls `enter()` and `exit()` |
| [Accountant](./Accountant.md) | Provides the exchange rate that the Teller uses to calculate share amounts before calling `enter`/`exit` |
| [BoringQueue](./BoringQueue.md) | Holds vault shares in custody for pending withdrawal requests; BoringSolver calls `teller.bulkWithdraw()` which calls `exit()` |
