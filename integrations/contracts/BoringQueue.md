# BoringQueue Reference

`BoringOnChainQueue` — an async withdrawal queue where users submit share-redemption requests that solvers fulfill in batches.

---

## Overview

The BoringQueue handles withdrawals that cannot be processed instantly — either because the vault's assets are illiquid, or because instant withdrawals are disabled on the Teller. Users lock their shares in the queue, specify the asset they want out, accept a discount, and wait for a solver to fill the request within a defined time window.

**Use it when you want to:**
- Withdraw shares when instant withdrawals via the Teller are unavailable
- Submit a withdraw request with a custom discount and deadline
- Cancel or replace a pending request
- Build a solver that fulfills user withdrawal requests

**Don't use it to:**
- Make instant deposits — use the [Teller](./Teller.md) instead
- Withdraw synchronously — if the Teller has `allowWithdraws == true` for your asset, that's faster
- Read the vault's exchange rate — query the [Accountant](./Accountant.md) directly

**Mental model:** The BoringQueue is an order book for share redemptions. Users post sell orders (with a discount to attract solvers), solvers fill those orders by providing the required assets, and users receive their tokens.

---

## Integration Guide

### Where it fits

```
User
  │
  ▼
BoringQueue.requestOnChainWithdraw()
  │  user approves queue to spend their shares
  │  shares transferred to queue
  │  request stored with: assetOut, amountOfShares, amountOfAssets, maturity window
  │
  │  (wait for secondsToMaturity to pass)
  │
  ▼
Solver calls BoringQueue.solveOnChainWithdraws()
  │  validates: all requests mature, none expired, all same asset
  │  transfers shares to solver
  │  calls solver's boringSolve() callback (if solveData provided)
  │  solver provides required assets
  │  queue distributes assets to each user
  │
  ▼
User receives assetOut
```

### Key Functions

| Function | Who calls it | What it does |
|---|---|---|
| `requestOnChainWithdraw` | End user | Submit a withdrawal request, locks shares in queue |
| `requestOnChainWithdrawWithPermit` | End user | Same but uses ERC-2612 permit for share approval |
| `cancelOnChainWithdraw` | Request owner | Cancel own request and retrieve shares |
| `replaceOnChainWithdraw` | Request owner | Cancel and resubmit with new discount/deadline |
| `solveOnChainWithdraws` | SOLVER_ROLE | Fulfill a batch of requests |
| `previewAssetsOut` | Anyone | Preview asset output for given shares + discount |
| `getRequestIds` | Anyone | List all pending request IDs |

### Asset amount calculation

When a request is created, the asset amount is fixed at submission time:

```
amountOfAssets = amountOfShares * (exchangeRate * (10000 - discount) / 10000) / ONE_SHARE
```

The discount is specified in basis points (e.g., `100` = 1% discount). The solver receives shares and provides `amountOfAssets` in return.

### Request lifecycle

```
creationTime
     │
     │← secondsToMaturity →│← secondsToDeadline →│
     │                      │                      │
  submitted              mature                 expired
  (not fillable)        (fillable)            (not fillable)
```

- Before maturity: request exists but solver cannot fill it
- After maturity, before deadline: solver can fill
- After deadline: request is expired and cannot be filled (user must cancel and resubmit)

### Submit a request (step-by-step)

1. Call `boringQueue.withdrawAssets(assetAddress)` — verify `allowWithdraws == true`
2. Note the `secondsToMaturity`, `minDiscount`, `maxDiscount`, `minimumShares`, and `minimumSecondsToDeadline` for the asset
3. Approve the **BoringQueue** to spend your vault shares (or use the permit variant)
4. Call `requestOnChainWithdraw(assetOut, amountOfShares, discount, secondsToDeadline)`
   - `discount` must be between `minDiscount` and `maxDiscount`
   - `secondsToDeadline` must be ≥ `minimumSecondsToDeadline`
   - `amountOfShares` must be ≥ `minimumShares`
5. Save the returned `requestId` and the full `OnChainWithdraw` struct emitted in the event — you need both to cancel or reference the request later

### Cancel a request (step-by-step)

1. Reconstruct the `OnChainWithdraw` struct from the original `OnChainWithdrawRequested` event
2. Call `cancelOnChainWithdraw(request)` — only callable by `request.user`
3. Shares are returned to your address

### Example (viem)

```typescript
import { createPublicClient, createWalletClient, http, parseUnits, decodeEventLog } from 'viem'
import { mainnet } from 'viem/chains'
import { privateKeyToAccount } from 'viem/accounts'

const QUEUE_ADDRESS = '0x...' as const
const VAULT_ADDRESS = '0x...' as const
const USDC_ADDRESS  = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48' as const

const account = privateKeyToAccount('0x...')

const publicClient = createPublicClient({ chain: mainnet, transport: http() })
const walletClient  = createWalletClient({ account, chain: mainnet, transport: http() })

const QUEUE_ABI = [
  {
    name: 'withdrawAssets',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'asset', type: 'address' }],
    outputs: [
      { name: 'allowWithdraws', type: 'bool' },
      { name: 'secondsToMaturity', type: 'uint24' },
      { name: 'minimumSecondsToDeadline', type: 'uint24' },
      { name: 'minDiscount', type: 'uint16' },
      { name: 'maxDiscount', type: 'uint16' },
      { name: 'minimumShares', type: 'uint96' },
      { name: 'withdrawCapacity', type: 'uint256' },
    ],
  },
  {
    name: 'previewAssetsOut',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'assetOut', type: 'address' },
      { name: 'amountOfShares', type: 'uint128' },
      { name: 'discount', type: 'uint16' },
    ],
    outputs: [{ name: 'amountOfAssets', type: 'uint128' }],
  },
  {
    name: 'requestOnChainWithdraw',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'assetOut', type: 'address' },
      { name: 'amountOfShares', type: 'uint128' },
      { name: 'discount', type: 'uint16' },
      { name: 'secondsToDeadline', type: 'uint24' },
    ],
    outputs: [{ name: 'requestId', type: 'bytes32' }],
  },
  {
    name: 'cancelOnChainWithdraw',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      {
        name: 'request',
        type: 'tuple',
        components: [
          { name: 'nonce', type: 'uint96' },
          { name: 'user', type: 'address' },
          { name: 'assetOut', type: 'address' },
          { name: 'amountOfShares', type: 'uint128' },
          { name: 'amountOfAssets', type: 'uint128' },
          { name: 'creationTime', type: 'uint40' },
          { name: 'secondsToMaturity', type: 'uint24' },
          { name: 'secondsToDeadline', type: 'uint24' },
        ],
      },
    ],
    outputs: [{ name: 'requestId', type: 'bytes32' }],
  },
  {
    name: 'OnChainWithdrawRequested',
    type: 'event',
    inputs: [
      { name: 'requestId', type: 'bytes32', indexed: true },
      { name: 'user', type: 'address', indexed: true },
      { name: 'assetOut', type: 'address', indexed: true },
      { name: 'nonce', type: 'uint96', indexed: false },
      { name: 'amountOfShares', type: 'uint128', indexed: false },
      { name: 'amountOfAssets', type: 'uint128', indexed: false },
      { name: 'creationTime', type: 'uint40', indexed: false },
      { name: 'secondsToMaturity', type: 'uint24', indexed: false },
      { name: 'secondsToDeadline', type: 'uint24', indexed: false },
    ],
  },
] as const

const ERC20_ABI = [
  {
    name: 'approve',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'spender', type: 'address' },
      { name: 'amount', type: 'uint256' },
    ],
    outputs: [{ name: '', type: 'bool' }],
  },
] as const

// 1. Check asset config
const withdrawAsset = await publicClient.readContract({
  address: QUEUE_ADDRESS,
  abi: QUEUE_ABI,
  functionName: 'withdrawAssets',
  args: [USDC_ADDRESS],
})
// withdrawAsset.allowWithdraws must be true before proceeding

// 2. Preview how many USDC you'd receive
const shareAmount       = parseUnits('100', 18)  // 100 vault shares (uint128 input)
const discount          = 50n                     // 0.5% in bps — bigint required for uint16
const secondsToDeadline = 604800n                 // 7 days — bigint required for uint24

const assetsOut = await publicClient.readContract({
  address: QUEUE_ADDRESS,
  abi: QUEUE_ABI,
  functionName: 'previewAssetsOut',
  args: [USDC_ADDRESS, shareAmount, discount],
})

// 3. Approve the Queue to spend your vault shares
await walletClient.writeContract({
  address: VAULT_ADDRESS,
  abi: ERC20_ABI,
  functionName: 'approve',
  args: [QUEUE_ADDRESS, shareAmount],
})

// 4. Submit request
const txHash = await walletClient.writeContract({
  address: QUEUE_ADDRESS,
  abi: QUEUE_ABI,
  functionName: 'requestOnChainWithdraw',
  args: [USDC_ADDRESS, shareAmount, discount, secondsToDeadline],
})

// 5. Parse the event to recover the full OnChainWithdraw struct
//    You must store this — the contract only stores the hash, not the struct itself
const receipt = await publicClient.waitForTransactionReceipt({ hash: txHash })
const log = receipt.logs.find(
  l => l.address.toLowerCase() === QUEUE_ADDRESS.toLowerCase()
)
const { args: eventArgs } = decodeEventLog({
  abi: QUEUE_ABI,
  eventName: 'OnChainWithdrawRequested',
  data: log!.data,
  topics: log!.topics,
})
// Store eventArgs — you need nonce, creationTime, secondsToMaturity to reconstruct
// the struct later if you want to cancel

// 6. Cancel if needed (before a solver fills it)
// const request = {
//   nonce:             eventArgs.nonce,
//   user:              account.address,
//   assetOut:          USDC_ADDRESS,
//   amountOfShares:    shareAmount,
//   amountOfAssets:    eventArgs.amountOfAssets,
//   creationTime:      eventArgs.creationTime,
//   secondsToMaturity: eventArgs.secondsToMaturity,
//   secondsToDeadline: secondsToDeadline,
// }
// await walletClient.writeContract({
//   address: QUEUE_ADDRESS,
//   abi: QUEUE_ABI,
//   functionName: 'cancelOnChainWithdraw',
//   args: [request],
// })
```

---

## Full Reference

### `requestOnChainWithdraw`

```solidity
function requestOnChainWithdraw(
    address assetOut,
    uint128 amountOfShares,
    uint16 discount,
    uint24 secondsToDeadline
) external returns (bytes32 requestId)
```

Locks `amountOfShares` in the queue and creates a withdrawal request. The amount of assets the user will receive (`amountOfAssets`) is calculated and locked in at this point.

**Parameters:**
- `assetOut` — the token the user wants to receive
- `amountOfShares` — shares to redeem (must be ≥ `withdrawAssets[assetOut].minimumShares`)
- `discount` — discount applied to the exchange rate in bps (must be between `minDiscount` and `maxDiscount`)
- `secondsToDeadline` — how long after maturity the request remains fillable (must be ≥ `minimumSecondsToDeadline`)

**Returns:** `requestId` — `keccak256(abi.encode(OnChainWithdraw))` — the unique ID for this request

**Emits:** `OnChainWithdrawRequested(requestId, user, assetOut, nonce, amountOfShares, amountOfAssets, creationTime, secondsToMaturity, secondsToDeadline)`

**Reverts:**
- `__Paused` — queue is paused
- `__WithdrawsNotAllowedForAsset` — asset not enabled
- `__BadDiscount` — discount outside `[minDiscount, maxDiscount]`
- `__BadShareAmount` — below `minimumShares`
- `__BadDeadline` — below `minimumSecondsToDeadline`
- `__NotEnoughWithdrawCapacity` — asset's `withdrawCapacity` would be exceeded

**Notes:**
- Requires prior `approve(QUEUE_ADDRESS, amountOfShares)` on the vault's ERC20
- `amountOfAssets` is fixed at request time using the current exchange rate and discount
- `withdrawCapacity` for the asset is decremented by `amountOfShares`

---

### `requestOnChainWithdrawWithPermit`

```solidity
function requestOnChainWithdrawWithPermit(
    address assetOut,
    uint128 amountOfShares,
    uint16 discount,
    uint24 secondsToDeadline,
    uint256 permitDeadline,
    uint8 v,
    bytes32 r,
    bytes32 s
) external returns (bytes32 requestId)
```

Same as `requestOnChainWithdraw` but uses an ERC-2612 permit to approve the queue to spend vault shares in the same transaction.

**Reverts:**
- `__PermitFailedAndAllowanceTooLow` — permit failed and existing allowance is insufficient
- All `requestOnChainWithdraw` reverts apply

---

### `cancelOnChainWithdraw`

```solidity
function cancelOnChainWithdraw(OnChainWithdraw memory request) external returns (bytes32 requestId)
```

Cancels the request and returns shares to `request.user`. Only callable by `request.user`.

**Parameters:**
- `request` — the full `OnChainWithdraw` struct, which must exactly match what was stored (reconstructed from the `OnChainWithdrawRequested` event)

**Reverts:**
- `__BadUser` — `msg.sender != request.user`
- `__RequestNotFound` — request not found in queue (wrong struct or already solved/cancelled)

**Emits:** `OnChainWithdrawCancelled(requestId, user, timestamp)`

**Notes:**
- `withdrawCapacity` is incremented by `request.amountOfShares` on cancellation
- Can be cancelled at any point (before or after maturity) as long as it hasn't been solved

---

### `replaceOnChainWithdraw`

```solidity
function replaceOnChainWithdraw(
    OnChainWithdraw memory oldRequest,
    uint16 discount,
    uint24 secondsToDeadline
) external returns (bytes32 oldRequestId, bytes32 newRequestId)
```

Atomically cancels `oldRequest` and creates a new request with the same shares but updated `discount` and `secondsToDeadline`. Does not consume additional `withdrawCapacity`.

**Reverts:**
- `__BadUser` — caller is not the request owner
- All `requestOnChainWithdraw` validation reverts (except capacity check) apply to the new request

**Emits:** `OnChainWithdrawCancelled(oldRequestId, ...)` and `OnChainWithdrawRequested(newRequestId, ...)`

---

### `solveOnChainWithdraws`

```solidity
function solveOnChainWithdraws(
    OnChainWithdraw[] calldata requests,
    bytes calldata solveData,
    address solver
) external
```

Fulfills a batch of withdrawal requests. All requests must be for the same `assetOut`. Called by SOLVER_ROLE.

**Parameters:**
- `requests` — array of `OnChainWithdraw` structs to fill; must be mature and not expired
- `solveData` — arbitrary bytes passed to `solver.boringSolve()`; pass empty bytes to skip the callback
- `solver` — address receiving the shares and making the callback

**Solve flow:**
1. Validates each request: same asset, mature, not expired
2. Dequeues all requests
3. Transfers total shares to `solver`
4. If `solveData.length > 0`, calls `solver.boringSolve(initiator, boringVault, solveAsset, totalShares, requiredAssets, solveData)` — solver must provide `requiredAssets` of `solveAsset` back to the queue before this call returns
5. Transfers `request.amountOfAssets` of `solveAsset` from `solver` to each `request.user`

**Reverts:**
- `__Paused` — queue is paused
- `__SolveAssetMismatch` — not all requests have the same `assetOut`
- `__NotMatured` — a request hasn't reached maturity yet
- `__DeadlinePassed` — a request has expired
- `__RequestNotFound` — a request isn't in the queue

**Emits:** `OnChainWithdrawSolved(requestId, user, timestamp)` for each request

---

### `previewAssetsOut`

```solidity
function previewAssetsOut(
    address assetOut,
    uint128 amountOfShares,
    uint16 discount
) public view returns (uint128 amountOfAssets)
```

Calculates the asset amount a user would receive for the given shares and discount at the current exchange rate.

```
price       = accountant.getRateInQuoteSafe(assetOut) * (10000 - discount) / 10000
amountOfAssets = amountOfShares * price / ONE_SHARE
```

**Reverts:** `__Overflow` — result exceeds `uint128`

---

### `getRequestIds`

```solidity
function getRequestIds() public view returns (bytes32[] memory)
```

Returns all active request IDs currently in the queue. Includes requests that are pending, mature, and expired — but not ones that have been solved or cancelled.

---

### `getRequestId`

```solidity
function getRequestId(OnChainWithdraw calldata request) external pure returns (bytes32 requestId)
```

Computes the request ID for a given struct. Equivalent to `keccak256(abi.encode(request))`.

---

### `WithdrawAsset` configuration

```solidity
struct WithdrawAsset {
    bool allowWithdraws;
    uint24 secondsToMaturity;       // max 30 days; 0 is valid — requests are fillable immediately
    uint24 minimumSecondsToDeadline; // max 30 days
    uint16 minDiscount;             // bps, e.g. 0 = no minimum
    uint16 maxDiscount;             // bps, max 3000 (30%)
    uint96 minimumShares;
    uint256 withdrawCapacity;       // rolling cap on outstanding shares; type(uint256).max = unlimited
}
```

Read per asset with `withdrawAssets(address assetOut)`.

**`secondsToMaturity` can be zero.** The contract does not enforce a minimum. If it is 0, submitted requests are immediately in the fillable window — there is no waiting period. Check this field before building any UX that shows users a "wait time" before their request can be filled.

---

### `OnChainWithdraw` struct

```solidity
struct OnChainWithdraw {
    uint96  nonce;             // auto-assigned, ensures unique request IDs
    address user;              // msg.sender at request time
    address assetOut;          // token to receive
    uint128 amountOfShares;    // shares locked in queue
    uint128 amountOfAssets;    // assets solver must provide (fixed at request time)
    uint40  creationTime;      // block.timestamp at request time
    uint24  secondsToMaturity; // from withdrawAssets config at request time
    uint24  secondsToDeadline; // user-specified, >= minimumSecondsToDeadline
}
```

The request ID is `keccak256(abi.encode(OnChainWithdraw))`. You must store the entire struct to cancel, replace, or reference a request — the contract does not store the struct, only its hash.

---

### Internal Behavior

**`withdrawCapacity`:** A per-asset rolling cap on the total shares outstanding in the queue. Decremented on new requests, incremented on cancellations. **Solver fills do not restore capacity** — it is one-way consumption. After successful solves, the admin must manually call `setWithdrawCapacity` to replenish it, or new requests will revert with `__NotEnoughWithdrawCapacity`. Plan for this in operational runbooks.

**`amountOfAssets` is fixed at request time:** The solver knows exactly how many assets to provide before filling. Exchange rate changes between submission and solve do not affect the payout to users.

**Shares leave the user's wallet at request time:** The queue holds shares in custody. Users cannot transfer or use those shares while the request is open.

**Solver callback is optional:** If `solveData` is empty, the callback is skipped and the solver must have pre-approved the queue to spend `requiredAssets` before calling `solveOnChainWithdraws`.

**Request ID uniqueness:** Request IDs are computed as `keccak256(abi.encode(OnChainWithdraw))`. The `nonce` field (auto-incremented from contract state) makes it practically impossible to generate two requests with the same ID.

---

### Edge Cases

**Expired requests can't be filled or auto-cancelled.** If a request passes its deadline, the solver cannot fill it. The user must call `cancelOnChainWithdraw` to get their shares back.

**Exchange rate movement between request and solve.** `amountOfAssets` is locked at submission time using the current exchange rate. If the exchange rate rises after submission, users receive fewer assets than a fresh withdrawal would give — the locked amount does not update. If the rate drops, users are protected (they get the higher locked-in value, solver absorbs the difference). Encourage users to cancel and resubmit if the rate has moved materially in their favour before a solver fills the request.

**Discount range is validated at request time.** If the admin changes `minDiscount`/`maxDiscount` after a request is submitted, the existing request is unaffected — it remains in the queue at its original discount.

**Multiple requests with the same parameters.** If two requests are submitted with identical parameters in the same block, they would have different nonces and thus different request IDs. Keccak256 collision is theoretically impossible.

**`withdrawCapacity == 0`.** Admin can set this to stop new requests for an asset without disabling existing ones. Existing requests remain fillable.

---

### Common Mistakes

1. **Storing only `requestId` and not the full struct.** You need the complete `OnChainWithdraw` struct to cancel or replace a request. Parse and store it from the `OnChainWithdrawRequested` event.

2. **Approving the wrong address for shares.** Approve the **BoringQueue**, not the Teller or vault, to spend vault shares.

3. **Submitting a discount outside the asset's allowed range.** Call `withdrawAssets(assetOut)` first and respect `minDiscount`/`maxDiscount`.

4. **Calling `cancelOnChainWithdraw` from a different address than the request's `user`.** The queue enforces `msg.sender == request.user`.

5. **Assuming a request will be filled before the deadline.** Solvers are not guaranteed to fill any request. If liquidity is low or the discount is unattractive, the request may expire. Build UI flows that let users monitor and replace expired requests.

6. **Not accounting for `withdrawCapacity`**. If capacity is 0 for an asset, `requestOnChainWithdraw` will revert with `__NotEnoughWithdrawCapacity`.

---

## Related Contracts

| Contract | Relationship |
|---|---|
| [BoringVault](./BoringVault.md) | Queue holds vault shares in custody; solver receives shares from vault's ERC20 |
| [Accountant](./Accountant.md) | Queue calls `accountant.getRateInQuoteSafe(assetOut)` to calculate `amountOfAssets` at request time |
| [Teller](./Teller.md) | The default solver (`BoringSolver`) calls `teller.bulkWithdraw()` to convert shares to assets for fulfillment |
