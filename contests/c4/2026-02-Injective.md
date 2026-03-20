# Injective Core - Peggy/Peggo Security Findings

## Scope Note
This report keeps only medium-severity findings with a concrete, code-backed break path in the current codebase. It excludes low/info issues, centralization/dependency notes, findings that depended on pre-broken invariants, unspecified state corruption, broad speculative assumptions, or issues already covered in [v12_injective.md]().

## Severity Definitions
- `Medium`: concrete runtime correctness, liveness, accounting, or enforcement failure in the bridge path that does not clearly self-heal.



## V12 Crosscheck


| Report Finding | V12 Status | Notes |
| -------------- | ---------- | ----- |
| `H-01` | `Removed as overlapping` | V12's `Fee-on-transfer tokens inflate deposits` covers the same core bridge-soundness bug: `Peggy.sol` records nominal deposit amounts without measuring actual token balance deltas, leading to undercollateralization for non-standard tokens. |
| `All retained findings below` | `No material overlap found` | V12 contains one additional high finding about permissionless nonce-spam on `sendToCosmos`, but that issue is not present in this report. The retained findings concern different Peggy/Peggo behaviors. |

Practical reading:
- `H-01` is intentionally excluded from the retained set because it is already covered by V12.
- No other retained finding appears duplicative of the current V12 report.

---

## Known-Issue Crosscheck


| Finding | Status                                      | Notes                                                                                                                                                                                                                                                                                                                                    |
| ------- | ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `M-01`  | `Not covered`                               | README does not mention deposit-claim `Data` hashing drift or attestation-split deadlock.                                                                                                                                                                                                                                                |
| `M-02`  | `Not covered`                               | README mentions mint-limit fund loss and a historical pre-existing mint-amount issue on rate-limit creation, but not negative `MintAmountERC20` enlarging mint headroom through `TrackTokenOutflow`, `UpdateRateLimit`, or exported-genesis restore.                                                                                     |
| `M-03`  | `Not covered`                               | No README known issue covers dropped iterator errors in peggo Ethereum log readers.                                                                                                                                                                                                                                                      |
| `M-04`  | `Not covered`                               | No README known issue covers the batch-slashing gap for unbonding validators.                                                                                                                                                                                                                                                            |
| `M-05`  | `Not covered`                               | No README known issue covers same-block batch index overwrite.                                                                                                                                                                                                                                                                           |
| `M-06`  | `Not covered`                               | No README known issue covers peggo retrying unresolved Ethereum submissions without a true replacement/gas-bump path.                                                                                                                                                                                                                    |
| `M-07`  | `Not covered`                               | No README known issue covers the `known transaction` retry path reusing a stale local nonce inside the serialized Ethereum submit path.                                                                                                                                                                                                   |

Practical reading:
- The retained medium findings are not described in the README's publicly known issue list.

---

## Findings Summary

| ID   | Prev ID | Severity | Title                                                                                           |
| ---- | ------- | -------- | ----------------------------------------------------------------------------------------------- |
| M-01 | `H6`    | Medium   | Deposit claim `Data` drift can deadlock event nonce progression                                 |
| M-02 | `H9`    | Medium   | MintAmountERC20 can go negative and enlarge absolute mint headroom                              |
| M-03 | `H10`   | Medium   | Ethereum log readers silently drop iterator errors                                              |
| M-04 | `H12`   | Medium   | Batch-confirm slashing skips unbonding validators                                               |
| M-05 | `H13`   | Medium   | Same-block batch secondary index overwrite hides batches from slashing/bookkeeping              |
| M-06 | `N/A`   | Medium   | Peggo does not replace stuck Ethereum txs and can re-submit the same operation at higher nonces |
| M-07 | `N/A`   | Medium   | `known transaction` retry loop can hot-loop on a stale nonce and block the sender path          |

---

## M-01 - Deposit claim `Data` drift can deadlock event nonce progression

**Severity:** `Medium`

### Finding Description
The deposit event schema, claim hashing logic, and current peggo broadcast path do not agree on whether deposit `Data` is part of the canonical bridge event. The Ethereum event and Injective claim hash both treat `Data` as meaningful, but peggo currently submits deposit claims with `Data: ""` unconditionally.

That mismatch means a non-empty `_data` field can produce two valid-looking but different attestation payloads for the same event nonce. Once validator/orchestrator behavior diverges on that field, Peggy cannot observe the disputed nonce and later event nonces remain blocked by strict in-order processing. The chain itself keeps producing blocks, but the bridge event pipeline can stall behind the first split nonce.

### Relevant Code Snippet
```go
msg := &peggytypes.MsgDepositClaim{
    ...
    Data: "",
}
```
Source: `peggo/orchestrator/cosmos/peggy/broadcast.go:245-254`

```go
func (msg *MsgDepositClaim) ClaimHash() []byte {
    path := fmt.Sprintf("%d/%d/%s/%s/%s/%s/%s", ..., msg.Data)
    return tmhash.Sum([]byte(path))
}
```
Source: `injective-chain/modules/peggy/types/msgs.go:366-368`

### Mitigation
- Make deposit `Data` canonical end-to-end.
- Either always populate `MsgDepositClaim.Data` from the emitted Ethereum event or remove `Data` from the event/claim-hash surface if it is intentionally unused.
- Add compatibility checks so mixed orchestrator versions cannot disagree on the canonical deposit payload during rollout.

---

## M-02 - MintAmountERC20 can go negative and enlarge absolute mint headroom

**Severity:** `Medium`

### Finding Description
The Peggy rate-limit subsystem uses `MintAmountERC20` as the running measure of outstanding bridged mint exposure. That tracker is decremented on outflow, and deposit enforcement computes remaining headroom as `AbsoluteMintLimit - currentMintAmount`.

The issue is that `MintAmountERC20` is allowed to go negative. If the tracker is ever initialized below the true outstanding minted amount, later outflows continue subtracting from that already-wrong baseline and can drive it below zero. Once negative, the remaining-headroom calculation becomes overly permissive because subtracting a negative number increases the reported allowance. The result is not free minting from thin air, but it does weaken the configured absolute cap that is supposed to constrain bridged ERC20 exposure.

### Relevant Code Snippet
```go
currentAmount := k.GetMintAmountERC20(ctx, tokenAddress)
k.SetMintAmountERC20(ctx, tokenAddress, currentAmount.Sub(out))
```
Source: `injective-chain/modules/peggy/keeper/rate_limit.go:126-127`

```go
absoluteLimit := rateLimit.AbsoluteMintLimit
currentAmount := k.GetMintAmountERC20(ctx, tokenAddress)
if remaining := absoluteLimit.Sub(currentAmount); remaining.LT(amount) {
    return ErrAbsoluteMintLimitOverflow
}
```
Source: `injective-chain/modules/peggy/keeper/rate_limit.go:231-234`

```go
func (k msgServer) UpdateRateLimit(...)
```
Source: `injective-chain/modules/peggy/keeper/msg_server.go:681-707` (no re-seeding of `MintAmountERC20`)

### Mitigation
- Enforce a non-negative invariant on `MintAmountERC20`.
- Clamp underflowing outflow updates at zero or reject inconsistent state before applying the subtraction.
- When creating or updating a rate limit, seed the tracker from the actual live outstanding minted supply instead of relying on an implicit zero baseline.
- Add repair logic for restore/migration paths so the tracker cannot restart from an incorrect value.

---

## M-03 - Ethereum log readers silently drop iterator errors

**Severity:** `Medium`

### Finding Description
Peggo’s Ethereum event readers iterate over go-ethereum log iterators with `for iter.Next()` but do not check `iter.Error()` after iteration finishes. That makes a partially failed log stream look identical to a fully successful one: the helper returns a truncated event list with `nil` error.

This becomes a bridge liveness issue because the oracle loop treats the returned slice as authoritative and can advance its Ethereum scan cursor past blocks whose full event set was never processed. On a busy bridge, later nonce gaps may eventually force a replay. On a quiet bridge, missed events can remain unclaimed until manual intervention.

### Relevant Code Snippet
```go
for iter.Next() {
    sendToInjectiveEvents = append(sendToInjectiveEvents, iter.Event)
}

return sendToInjectiveEvents, nil
```
Source: `peggo/orchestrator/ethereum/network.go:194-199`

```go
if len(newEvents) == 0 {
    ...
    l.lastRecordedEthEventHeight = latestHeight
    l.resetQueryRange()
    return nil
}
```
Source: `peggo/orchestrator/oracle.go:124-132`

### Mitigation
- Check `iter.Error()` after every log-iterator loop and propagate the error upward.
- Do not advance the local scanned-height cursor unless the full block-range query completed successfully.
- Add retry or replay logic that preserves the previous cursor on partial failures.

---

## M-04 - Batch-confirm slashing skips unbonding validators

**Severity:** `Medium`

### Finding Description
Peggy’s valset slashing flow explicitly considers unbonding validators, but the batch-confirm slashing flow does not. `batchSlashing()` iterates only the currently bonded validator set when determining who failed to submit required batch confirmations.

That creates an enforcement gap between the validator set that was actually responsible for signing a batch on Ethereum and the validator set that is still punishable when the slashing window is evaluated. A validator can therefore avoid the intended batch-confirm penalty by entering unbonding before the check runs, weakening bridge-signing accountability and the incentive structure that supports relaying.

### Relevant Code Snippet
```go
unslashedBatches := h.k.GetUnslashedBatches(ctx, maxHeight)

for _, batch := range unslashedBatches {
    currentBondedSet, _ := h.k.StakingKeeper.GetBondedValidatorsByPower(ctx)
    confirms := h.k.GetBatchConfirmByNonceAndTokenContract(ctx, batch.BatchNonce, common.HexToAddress(batch.TokenContract))
    for i := range currentBondedSet {
        ...
    }
}
```
Source: `injective-chain/modules/peggy/abci.go:349-379`

### Mitigation
- Apply the same unbonding-validator coverage in `batchSlashing()` that already exists in the valset-slashing path.
- Evaluate batch-confirm responsibility against the validator set active at the relevant batch nonce/window, not only the currently bonded set.
- Add regression tests covering validators who start unbonding between missed confirmation and slashing evaluation.

---

## M-05 - Same-block batch secondary index overwrite hides batches from slashing and bookkeeping

**Severity:** `Medium`

### Finding Description
Peggy stores each batch correctly under its primary key `(tokenContract, batchNonce)`, but it also maintains a secondary index keyed only by block height. When multiple batches are created in the same Cosmos block, each batch write targets that same secondary key and overwrites the previous one.

The primary batch objects remain in state, so this is not direct data loss. The problem is that any code path enumerating batches through the block-height secondary index can now see only the last batch written for that block. That incomplete view affects unslashed-batch discovery and related bookkeeping, which means some batches can effectively disappear from enforcement and maintenance flows despite still existing under their primary key.

### Relevant Code Snippet
```go
func GetOutgoingTxBatchBlockKey(block uint64) []byte {
    return append(OutgoingTXBatchBlockKey, UInt64Bytes(block)...)
}
```
Source: `injective-chain/modules/peggy/types/key.go:220-222`

```go
blockKey := types.GetOutgoingTxBatchBlockKey(batch.Block)
store.Set(blockKey, k.cdc.MustMarshal(batch))
```
Source: `injective-chain/modules/peggy/keeper/batch.go:149-150`

```go
k.IterateBatchBySlashedBatchBlock(ctx, lastSlashedBatchBlock, maxHeight, func(_ []byte, batch *types.OutgoingTxBatch) bool {
    if batch.Block > lastSlashedBatchBlock {
        out = append(out, batch)
    }
    return false
})
```
Source: `injective-chain/modules/peggy/keeper/batch.go:339-345`

### Mitigation
- Make the secondary index key unique per batch by including token contract and/or batch nonce alongside block height.
- If the intent is to enumerate all batches created at a height, store a list/prefix index instead of a single batch object at that key.
- Add tests covering same-block multi-batch creation and slashing enumeration through the secondary index.

---

## M-06 - Peggo does not replace stuck Ethereum txs and can re-submit the same operation at higher nonces

**Severity:** `Medium`

### Finding Description
Peggo increments its local Ethereum nonce cache immediately after `eth_sendRawTransaction` succeeds. It does not wait for confirmation, and it does not implement a true same-nonce replacement or gas-bump strategy for transactions that remain pending or disappear from the txpool.

There is a partial duplicate-submission guard based on pending transaction input data, but that protection is optional and temporary because it depends on an Alchemy websocket subscription and only suppresses repeats for `pendingTxWaitDuration`. If the original submission remains unresolved beyond that window, peggo can resubmit the same bridge operation at a higher nonce instead of replacing the stuck transaction. The result is a relayer-local liveness failure that often requires operator cleanup or nonce resynchronization.

### Relevant Code Snippet
```go
if err == nil {
    txHash = txHashRet
    e.nonceCache.Incr(e.fromAddress)
    return nil
}
```
Source: `peggo/orchestrator/ethereum/committer/eth_committer.go:161-165`

```go
if s.pendingTxInputList.IsPendingTxInput(txData, s.pendingTxWaitDuration) {
    return nil, errors.New("Transaction with same batch input data is already present in mempool")
}
```
Source: `peggo/orchestrator/ethereum/peggy/submit_batch.go:85-87`

```go
if cfg.EthNodeAlchemyWS != "" {
    ...
    go peggyContract.SubscribeToPendingTxs(cfg.EthNodeAlchemyWS)
}
```
Source: `peggo/orchestrator/ethereum/network.go:113-118`

### Mitigation
- Track Ethereum submission state by nonce until confirmation or explicit replacement, instead of advancing the local cache on raw submission alone.
- Implement a same-nonce replacement path with gas bumping for unresolved transactions.
- Treat websocket-based pending-tx dedupe as an optimization only, not as the primary correctness guard.
- Add recovery logic to reconcile local nonce state against the Ethereum account nonce after txpool churn or restart.

---

## M-07 - `known transaction` retry loop can hot-loop on a stale nonce and block the sender path

**Severity:** `Medium`

### Finding Description
Peggo's Ethereum submit path handles `"known transaction"` as a retryable condition, but the retry logic mutates a shadowed `nonce` variable instead of the outer loop nonce. On the next iteration, `SendTx` recreates and resubmits the same stale-nonce transaction again, while the shared nonce cache has already been incremented.

Because this happens inside `nonceCache.Serialize`, repeated `"known transaction"` responses can trap the sender in an unbounded hot loop under the per-address serialization lock. In practice that can stall later Ethereum submissions for the same signer address, including batch relays, valset updates, and `sendToCosmos` operations. This is not a chain-wide halt, but it is a concrete relayer-side liveness failure with no automatic recovery.

### Relevant Code Snippet
```go
if err := e.nonceCache.Serialize(e.fromAddress, func() (err error) {
    nonce, _ := e.nonceCache.Get(e.fromAddress)
    ...
    for {
        opts.Nonce = big.NewInt(nonce)
        ...
    }
}); err != nil {
```
Source: `peggo/orchestrator/ethereum/committer/eth_committer.go:132-146`

```go
if strings.Contains(err.Error(), "known transaction") {
    // skip one nonce step, try to send again
    nonce := e.nonceCache.Incr(e.fromAddress)
    opts.Nonce = big.NewInt(nonce)
    continue
}
```
Source: `peggo/orchestrator/ethereum/committer/eth_committer.go:199-204`

```go
func (n nonceCache) Serialize(account common.Address, fn func() error) error {
    mux, _ := n.guard.LoadOrStore(account, new(sync.Mutex))
    mux.(*sync.Mutex).Lock()
    defer mux.(*sync.Mutex).Unlock()
    return fn()
}
```
Source: `peggo/orchestrator/ethereum/util/noncecache.go:36-41`

### Mitigation
- Remove the shadowed `nonce :=` assignment and update the outer loop nonce before retrying.
- Bound the retry loop and force a nonce resync from the provider before retrying `"known transaction"` responses.
- Do not increment the local nonce cache on `"known transaction"` unless the provider/account nonce confirms that the transaction was actually accepted at that nonce.
- Add regression tests for repeated `"known transaction"` responses to ensure the serialized sender path always exits or resynchronizes.

---
