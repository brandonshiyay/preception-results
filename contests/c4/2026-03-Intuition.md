# Intuition Protocol — Final Audit Report

This report summarizes the security findings identified during the review of the Intuition Protocol's in-scope smart contracts. All findings have been verified against the codebase using the Codex MCP server.

# Issues Submitted
Med-01, Low-02
---

## Severity Assessment Criteria

| Severity      | Likelihood × Impact                                                              |
| ------------- | -------------------------------------------------------------------------------- |
| High          | Exploitable by external actors with meaningful financial or safety impact        |
| Medium        | Exploitable under specific conditions or by semi-trusted actors; moderate impact |
| Low           | Self-inflicted, requires unusual conditions, or minor impact                     |
| Informational | Design observation; no direct exploit path                                       |

**Trust Boundaries:** Timelock (3/7-day) and DEFAULT_ADMIN_ROLE are treated as trusted. Findings requiring trusted-role collusion are downgraded accordingly.

---

## [HIGH-1] AtomWallet: `validUntil`/`validAfter` time window is unsigned and freely malleable

- **Severity**: High
- **Status**: Confirmed present in audit-scope codebase (V12 patch NOT applied)
- **Description**: AtomWallet supports an extended 77-byte signature format that appends `validUntil` and `validAfter` values to enforce a time window on a signed user operation. However, the wallet verifies the signature only against `userOpHash`, while the appended timing fields are parsed separately afterward. Because the EntryPoint-generated `userOpHash` does not include the signature payload, those final 12 bytes are never authenticated.
- **Description Continued**: As a result, anyone who can observe and relay the signed operation can alter the expiry window without invalidating the signature. In practice, a signer may believe they authorized execution only during a narrow interval, while a malicious relayer can extend, shift, or effectively remove that restriction before submission. The issue does not affect the default 65-byte signature format, but it completely defeats the purpose of the protocol's time-restricted signature mode.
- **Prior Audit Coverage**: ⚠️ **Already reported by V12** as "Unsigned validity window metadata" (Critical). V12 provided both description and patch. The patch was NOT applied to the in-scope code. This finding overlaps with V12's report — submitting it may be treated as a known issue.
- **Likelihood**: Medium — requires a malicious relayer or mempool observer targeting 77-byte time-restricted signatures
- **Constraints**:
  1. User must submit a 77-byte time-restricted signature (standard 65-byte sigs are unaffected)
  2. Attacker must intercept and modify the trailing 12 bytes before inclusion
- **File Location**: `src/protocol/wallet/AtomWallet.sol` — `_validateSignature` (line 285–311), `_extractValidUntilAndValidAfterFromSignature` (line 337–361)
- **Root Cause**: `_validateSignature` constructs the recovery hash exclusively from `userOpHash`, omitting the parsed `validUntil`/`validAfter` suffix. Since `userOpHash` is computed by the EntryPoint from the UserOp *excluding* the signature field, the 12-byte time metadata is cryptographically unbound.
- **Code Snippet**:
  ```solidity
  bytes32 hash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", userOpHash));
  // validUntil and validAfter are parsed separately but NOT included in hash
  ```
- **Impact**: An adversary can extend or remove the expiry from any signed user operation, enabling execution beyond the signer's intended timeframe. This breaks the fundamental time-bounding guarantee of 77-byte signatures.
- **Reassessment**: Confirmed High. While it requires 77-byte signature usage (not default), the impact on users who explicitly opt into time-bounded operations is severe — their temporal safety guarantee is fully bypassed.
- **Mitigation**: Bind `validUntil`/`validAfter` into the signed hash when `signature.length == 77`:
  ```solidity
  bytes32 signedHash = userOpHash;
  if (userOp.signature.length == 77) {
      signedHash = keccak256(abi.encodePacked(userOpHash, validUntil, validAfter));
  }
  bytes32 hash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", signedHash));
  ```

---

## [MED-1] TrustSwapAndBridgeRouter: Bridge fee quoted from `minTrustOut`, not actual swap output

- **Severity**: Medium
- **Status**: Confirmed — V12 patch NOT present in audit-scope codebase
- **Description**: The router's combined swap-and-bridge flow estimates the cross-chain bridge fee before the swap executes, using `minTrustOut` as the quoted transfer amount. `minTrustOut` is only a lower bound chosen for slippage protection; it is not guaranteed to equal the amount actually received from the swap. When execution produces positive slippage, the real output `amountOut` can exceed the amount used for the bridge quote.
- **Description Continued**: The function then attempts to bridge `amountOut` while still paying the previously quoted fee derived from `minTrustOut`. If the underlying bridge charges a fee that scales with the bridged amount, the router underpays the bridge whenever the swap outperforms the minimum expectation. This mismatch between the quoted fee and the actual downstream transfer request can cause the bridge step to revert even though the swap itself succeeded.
- **Prior Audit Coverage**: ✅ **NOT covered by V12**. V12's automated report for TrustSwapAndBridgeRouter (M-03) describes "Bridge Fee Quote May Become Stale During Transaction" — a *staleness* concern — but does NOT identify the specific root cause that `minTrustOut` (not `amountOut`) is used. This is a **novel finding**.
- **Likelihood**: Medium — occurs on every `swapAndBridgeWithERC20`/`swapAndBridgeWithETH` call where the swap has positive slippage
- **Constraints**: Requires `amountOut > minTrustOut` (normal positive slippage) AND `quoteTransferRemote` to be amount-sensitive
- **File Location**: `intuition-contracts-v2-periphery/contracts/TrustSwapAndBridgeRouter.sol` — `swapAndBridgeWithERC20` (line 150), `swapAndBridgeWithETH` (line 94)
- **Root Cause**: The router quotes `bridgeFee = quoteTransferRemote(..., minTrustOut)` *before* the swap, then calls `transferRemote{value: bridgeFee}(..., amountOut)` with the actual (potentially higher) output. If the bridge fee is amount-proportional, the quoted fee undercovers the actual transfer.
- **Code Snippet**:
  ```solidity
  uint256 bridgeFee = metaERC20Hub.quoteTransferRemote(recipientDomain, recipientAddress, minTrustOut);
  // ... swap happens, amountOut > minTrustOut possible ...
  transferId = _bridgeTrust(amountOut, recipientAddress, bridgeFee);
  ```
- **Impact**: Successful swaps with positive slippage can revert at the bridge step, causing a denial of service for the core onboarding UX. Users who get favorable swap prices are paradoxically *more* likely to fail.
- **Reassessment**: Medium. This directly impacts the protocol's primary cross-chain onboarding flow and can cause persistent failures under normal market conditions (positive slippage is common).
- **Mitigation**: Quote the bridge fee *after* the swap using the actual `amountOut`, or re-quote before bridging:
  ```solidity
  uint256 bridgeFee = metaERC20Hub.quoteTransferRemote(recipientDomain, recipientAddress, amountOut);
  ```

---

## [LOW-1] TrustBonding: Mutable global parameters retroactively alter historical epoch rewards

- **Severity**: Low
- **Likelihood**: Low — requires Timelock (trusted role) to adjust parameters between epoch close and user claim
- **Constraints**: Timelock is a trusted role with 3-day minimum delay; purely administrative action
- **File Location**: `src/protocol/emissions/TrustBonding.sol` — `_getPersonalUtilizationRatio`, `_getSystemUtilizationRatio`, `claimRewards`
- **Root Cause**: `claimRewards` computes utilization discounts using real-time state (current `personalUtilizationLowerBound`, current `multiVault` address) rather than snapshotted epoch-close values.
- **Impact**: Timelock parameter adjustments retroactively change the reward calculation for already-concluded epochs. A `personalUtilizationLowerBound` change from 2500→9000 bps would multiply unclaimed rewards by ~3.6x.
- **Reassessment**: Low. The Timelock is trusted and has a 3-day delay, providing community reaction time. However, it remains a structural design flaw — parameter adjustments (not misconfigs) should not retroactively alter finalized economics.
- **Mitigation**: Snapshot reward parameters at each epoch boundary. Claim logic should read from snapshots.

---

## [LOW-2] TrustBonding: Epoch boundary timestamp allows free-riding on prior epoch rewards

- **Severity**: Low
- **Finding Description**: TrustBonding derives reward eligibility from veTRUST balances at `epochTimestampEnd(epoch)`, but `epochTimestampEnd(N)` is the same instant as `epochTimestampStart(N+1)`. At that exact boundary, the epoch math already considers the system to be in `N+1`, while VotingEscrow's inclusive checkpoint lookup still counts balance updates written at that timestamp in epoch `N`'s snapshot. This allows a user to create a new lock, top up an existing lock, or extend lock duration at the boundary and still improve their share of epoch `N` rewards even though they did not participate during epoch `N`. In practice, epoch `N` is not truly finalized at its end boundary, which gives boundary actors a last-mover advantage and retroactively reallocates rewards away from users who were actually locked throughout the epoch. Total emissions remain capped, so this is redistribution rather than inflation or principal theft.
```solidity
// TrustBonding snapshots exactly at the epoch boundary
function totalBondedBalanceAtEpochEnd(uint256 epoch) public view returns (uint256) {
    return _totalSupply(_epochTimestampEnd(epoch));
}

function userBondedBalanceAtEpochEnd(address account, uint256 epoch) public view returns (uint256) {
    return _balanceOf(account, _epochTimestampEnd(epoch));
}

// CoreEmissionsController treats that same timestamp as the next epoch
function _calculateEpochTimestampEnd(uint256 epoch) internal view returns (uint256) {
    return _START_TIMESTAMP + (epoch * _EPOCH_LENGTH) + _EPOCH_LENGTH;
}

function _calculateTotalEpochsToTimestamp(uint256 timestamp) internal view returns (uint256) {
    return (timestamp - _START_TIMESTAMP) / _EPOCH_LENGTH;
}

// VotingEscrow includes checkpoints written exactly at _ts
if (user_point_history[addr][_mid].ts <= _ts) {
    _min = _mid;
}
```
- **Mitigation**: Use strict `<` comparison for epoch-end snapshots, or snapshot at `_epochTimestampEnd(epoch) - 1`.

---

## [LOW-3] AtomWallet: Self-ownership via `transferOwnership(address(this))` permanently bricks wallet

- **Severity**: Low
- **Likelihood**: Very Low — requires the owner to deliberately execute a two-step self-ownership sequence
- **Constraints**: Entirely self-inflicted; no external attacker can trigger this
- **File Location**: `src/protocol/wallet/AtomWallet.sol` — `transferOwnership`, `acceptOwnership`, `execute`
- **Root Cause**: No guard prevents `transferOwnership(address(this))`. The wallet can then call `acceptOwnership()` on itself via `execute(address(this), ...)`, setting `owner() = address(this)`. Since no private key corresponds to a contract address, all subsequent owner-gated and signature-validated operations fail permanently.
- **Impact**: Permanent wallet bricking with no recovery path. Any ETH, tokens, or vault deposit fees in the wallet are permanently locked.
- **Reassessment**: Low. Self-inflicted only, but the consequence (permanent fund loss) is severe if triggered accidentally by a smart contract owner or multisig bug.
- **Mitigation**: Add `require(newOwner != address(this))` in `transferOwnership`.

---

## Informational Findings

### [INFO-1] TrustBonding: Zero-amount claims lack boolean marking

- **File**: `src/protocol/emissions/TrustBonding.sol`
- **Issue**: `_hasClaimedRewardsForEpoch` uses `amount > 0` instead of a dedicated boolean. Under current constraints (min `personalUtilizationLowerBound` = 2500 bps, revert on `userRewards == 0`), this cannot be exploited.
- **Reassessment**: Informational. Current guards make exploitation infeasible. Architectural debt only.

### [INFO-2] TrustBonding: `getUnclaimedRewardsForEpoch` uses max emissions, not utilization-adjusted emissions

- **File**: `src/protocol/emissions/TrustBonding.sol` (lines 335–340)
- **Issue**: Reports `maxEmissions - actualClaimed` as unclaimed, which includes tokens that were never user-claimable due to utilization discounts. By design per in-code comment, but overestimates "unclaimed" value.

### [INFO-3] TrustBonding: Pausing during epoch transition permanently forfeits user rewards

- **File**: `src/protocol/emissions/TrustBonding.sol`
- **Issue**: `claimRewards` has `whenNotPaused`. Epochs advance via `block.timestamp` regardless of pause state. If paused for >1 epoch, users permanently lose rewards for the skipped epoch.
- **Confirmed**: Codex MCP verified no freeze mechanism or window-extension exists.

### [INFO-4] AtomWallet: Accumulated deposit fees go to AtomWarden while wallet is unclaimed

- **File**: `src/protocol/wallet/AtomWallet.sol`, `src/protocol/MultiVault.sol`
- **Issue**: `claimAtomWalletDepositFees` resolves `owner()` lazily at claim time. Before `acceptOwnership()`, `owner()` returns `AtomWarden`, who receives all accumulated fees. Not earmarked per owner.

### [INFO-5] TrustSwapAndBridgeRouter: `safeIncreaseAllowance` can accumulate residual token approvals

- **File**: `intuition-contracts-v2-periphery/contracts/TrustSwapAndBridgeRouter.sol`
- **Issue**: `safeIncreaseAllowance` adds to existing allowance. If `exactInput` ever under-spends, residual accumulates. Risk only materializes if the hardcoded Slipstream router is compromised.

### [INFO-6] TrustSwapAndBridgeRouter: `swapAndBridgeWithETH` has no ETH refund path

- **File**: `intuition-contracts-v2-periphery/contracts/TrustSwapAndBridgeRouter.sol`
- **Issue**: All `msg.value - bridgeFee` is wrapped into WETH for the swap. Unlike `swapAndBridgeWithERC20`, no refund mechanism exists. Combined with MED-1 (fee quoted from `minTrustOut`), the swap amount may exceed user intent.

---

## Invalid / False Positive Findings (Removed)

| Original ID       | Claim                                            | Codex MCP Verdict                                                |
| ----------------- | ------------------------------------------------ | ---------------------------------------------------------------- |
| CRIT-1            | AtomWallet ownership slot mismatch bricks wallet | **Invalid** — Custom slots exactly match OZ ERC-7201 constants   |
| LOW-4             | Router `bridgeTrust(0)` wastes ETH fees          | **Invalid** — Zero-amount guards `AmountInZero` verified present |
| HIGH-1 (original) | Zero-amount claims enable double claiming        | **Downgraded to INFO** — Revert-on-zero prevents the write       |

---

## Confirmed Safe Behaviors

| Area                   | Hypothesis Tested                                      | Result                                                                 |
| ---------------------- | ------------------------------------------------------ | ---------------------------------------------------------------------- |
| TrustBonding           | Division-by-zero when `totalBalance == 0`              | **Safe** — explicit guard returns 0                                    |
| TrustBonding           | Double-claim reentrancy                                | **Safe** — `nonReentrant` + check-before-write                         |
| TrustBonding           | Flash-loan `totalBondedBalanceAtEpochEnd` manipulation | **Safe** — checkpoint snapshots, not live supply                       |
| TrustBonding           | System utilization ratio overflow >10000 bps           | **Safe** — explicit cap at `BASIS_POINTS_DIVISOR`                      |
| ProgressiveCurve       | Small deposit yields 0 shares (fund loss)              | **Safe** — `MultiVault` enforces `minDeposit`                          |
| ProgressiveCurve       | `squareUp` rounding inflates costs                     | **Safe** — `fullMulDivUp`, max 1 wei bias                              |
| OffsetProgressiveCurve | Full redemption underflows                             | **Safe** — `sNext = OFFSET`                                            |
| OffsetProgressiveCurve | OFFSET handling in sqrt/subtraction                    | **Safe** — applied on both sides                                       |
| Router                 | Malformed path reads garbage memory                    | **Safe** — length/modulo checks revert first                           |
| Router                 | ETH dust accumulates from failed ops                   | **Safe** — refund uses `msg.value - bridgeFee`                         |
| Router                 | `tokenIn == TRUST` bypasses validation                 | **Safe** — explicit `InvalidToken` revert                              |
| AtomWallet             | `executeBatch` gas-griefs EntryPoint                   | **Safe** — bounded by `callGasLimit` and block gas                     |
| AtomWallet             | Permissionless `addDeposit()` griefing                 | **Safe** — functionally a donation                                     |
| VotingEscrow           | Checkpoint staleness after inactivity                  | **Safe** — `_supply_at` reconstructs from slope changes (255-week cap) |
| MultiVault             | Utilization inflation via tx spam                      | **Safe** — volume-based (net assets), not count-based                  |
