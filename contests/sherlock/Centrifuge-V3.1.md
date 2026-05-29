### [H-01] Pool manager can fabricate batch deposit approvals to drain shared async escrow

**Severity**: High  
**Likelihood**: Medium  
**Impact**: Cross-pool fund loss  
**Location**: protocol/src/vaults/BatchRequestManager.sol:102, protocol/src/core/spoke/Spoke.sol:321, protocol/src/vaults/AsyncRequestManager.sol:254

**Summary/Description**  
A pool manager can temporarily set its spoke request manager to a malicious contract, submit fake Request messages that BatchRequestManager accepts as pending deposits, then switch the spoke request manager back to AsyncRequestManager before approving the fake deposits. The ApprovedDeposits callback is routed to AsyncRequestManager, which moves assetAmount from the shared globalEscrow into the attacker's pool escrow without checking any matching per-vault pending deposit state.

**Root Cause**  
BatchRequestManager trusts authenticated hub requests but does not bind pending requests to the spoke request manager or vault state that originated them. Spoke.request lets the current requestManager for a pool send arbitrary request payloads, while Spoke.requestCallback later routes callbacks to whatever requestManager is current at callback time. AsyncRequestManager.approvedDeposits transfers from globalEscrow before any per-user pending request validation.

**Pre_conditions**  
The attacker controls a pool manager for pool A; an asset has funds sitting in AsyncRequestManager.globalEscrow from any pool's pending async deposits; the attacker can update pool A's request manager and balance sheet manager through normal pool-manager functions.

**Impact**  
The attacker can move assets from the shared globalEscrow into pool A's pool escrow and then withdraw them as pool A balance-sheet manager. This can steal pending async deposit assets belonging to other pools/users for the same asset.

**Mitigation**  
Bind request callbacks to the exact request manager/vault state that created the pending request, or make AsyncRequestManager.approvedDeposits verify corresponding pending deposits before moving global escrow funds. Avoid using a mutable per-pool requestManager lookup for callbacks whose pending state was created under a different manager.

### [H-02] Factory-returned existing async vault can be remapped to drain shared escrow assets

**Severity**: High  
**Likelihood**: Medium  
**Impact**: Cross-pool fund loss from the shared async escrow  
**Location**: protocol/src/core/spoke/VaultRegistry.sol:75, protocol/src/core/spoke/VaultRegistry.sol:99, protocol/src/vaults/AsyncVaultFactory.sol:43, protocol/src/vaults/AsyncRequestManager.sol:254

**Summary/Description**  
VaultRegistry.deployVault trusts the vault address returned by the caller-supplied factory and registerVault overwrites _vaultDetails for that address before linkVault checks uniqueness. A pool manager can deploy a legitimate AsyncVault through the official factory for one asset, making it a ward on AsyncRequestManager, then call DeployAndLink for a different asset with a malicious factory that returns the same already-relied vault. The vault keeps transferring its immutable original asset from users, while AsyncRequestManager now reads the overwritten registry assetId/details and treats the request as a deposit of the second asset.

**Root Cause**  
The registry does not require factory.newVault to return a fresh vault, does not reject an already-registered vault, and does not validate the returned vault's immutable asset/share/manager fields against the assetId and share token being registered. Because official factories rely newly created vaults on AsyncRequestManager, reusing one of those vault addresses bypasses the intended fix that an arbitrary malicious factory cannot rely its own malicious vault.

**Pre_conditions**  
The attacker controls a pool manager for pool A and can execute normal vault updates for that pool. A legitimate async vault for pool A/share class S has already been created through AsyncVaultFactory for asset X. The target asset Y has funds sitting in AsyncRequestManager.globalEscrow from pending async deposits or cancellations for any pool.

**Impact**  
After remapping the already-authorized vault from asset X to asset Y, the attacker submits a deposit through that vault. The vault transfers asset X into globalEscrow, but AsyncRequestManager sends and processes the request under asset Y because vaultRegistry.vaultDetails(vault) was overwritten. When the attacker approves the deposit, AsyncRequestManager.approvedDeposits transfers asset Y from the shared globalEscrow into the attacker's pool escrow, stealing pending escrowed asset Y that can belong to other pools/users.

**Mitigation**  
Make deployVault/registerVault reject already-registered vault addresses and validate the returned vault's poolId, scId, asset, share token, and request-manager configuration before any registry write. Avoid granting manager auth through a factory-wide ward until after the registry has verified the exact fresh vault being registered, or replace vault wards with a msg.sender == vault plus registry-linked validation model.

### [M-01] Frozen controller can redirect claimable async shares

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: protocol/src/vaults/AsyncRequestManager.sol:405

**Summary/Description**  
Async deposit claims and cancelled redeem claims transfer shares from globalEscrow directly to the receiver. The transfer hook only evaluates globalEscrow -> receiver, so it never checks the controller whose claimable shares are being released. If the controller is frozen after the request becomes claimable, the controller can still call the claim function and send the shares to another receiver that satisfies the receiver-side hook checks.

**Root Cause**  
_processDeposit() and claimCancelRedeemRequest() release controller-owned claimable shares through globalEscrow.authTransferTo() without first checking the controller-to-receiver restriction. The actual ERC20 hook callback observes only globalEscrow -> receiver, and globalEscrow is the depositTarget/endorsed system account, so controller freeze state is not evaluated.

**Pre_conditions**  
Controller has claimable deposit shares or claimable cancelled redeem shares in globalEscrow; before claiming, the controller is frozen; the chosen receiver is not frozen and satisfies the receiver-side share hook checks.

**Impact**  
A frozen controller can move claimable shares to another address and continue controlling transferable share tokens through that address. This evades the intended freeze enforcement for asynchronous share-claim paths. Per contest guidance, hook-check circumvention impact is Medium at most.

**Mitigation**  
Before releasing shares in _processDeposit() and claimCancelRedeemRequest(), enforce the same controller-to-receiver restriction used by redeem claims and cancelled deposit claims. When controller != receiver, require _canTransfer(vault, controller, receiver, shares), and ensure frozen-controller cases cannot redirect claimable shares.

### [M-02] Asset-only queue syncs are not rate-limited after the first allowed sync

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Repeatable missing throttling for permissionless asset syncs and downstream snapshot updates  
**Location**: protocol/src/managers/spoke/QueueManager.sol:66, protocol/src/managers/spoke/QueueManager.sol:80, protocol/src/managers/spoke/QueueManager.sol:85

**Summary/Description**  
QueueManager enforces minDelay against sc.lastSync, but sc.lastSync is updated only when queued shares are submitted. Asset-only queue syncs can still be complete snapshots through BalanceSheet.submitQueuedAssets when no share delta exists. After the first allowed asset-only sync, lastSync remains at its older value, so subsequent permissionless asset-only updates for the same pool/share class can be synced immediately instead of being spaced by minDelay.

**Root Cause**  
syncCallback writes sc.lastSync only inside the submitShares branch. The branch requires delta > 0 and queuedAssetCounter == 0, so successful asset-only syncs, including snapshot=true asset submissions, never refresh the delay anchor.

**Pre_conditions**  
The pool enables QueueManager with a non-zero minDelay and has a permissionless or externally frequent asset producer, such as an enabled OnOfframpManager onramp, while no queued share delta exists for the target share class.

**Impact**  
A caller can repeatedly create small asset-only queue updates and sync them without the configured delay once lastSync is old enough. Each complete asset-only sync can set the hub snapshot state and invoke snapshot hooks; with SimplePriceManager configured, this can repeatedly update share price state and enqueue downstream price notifications, weakening the intended anti-spam control for permissionless balance-sheet changes.

**Mitigation**  
Refresh lastSync whenever QueueManager submits any queued update, or at least whenever the post-sync state is a complete snapshot. If partial asset submissions should not consume the delay, track a separate lastAssetSync/lastSnapshotSync and enforce the configured delay for asset-only snapshots as well as share submissions.

### [M-03] QueueManager sync can be forced into unpaid execution during inbound callbacks

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Repeatable sync liveness disruption and unpaid batch creation for other pools  
**Location**: protocol/src/core/messaging/MessageProcessor.sol:83, protocol/src/core/messaging/Gateway.sol:183, protocol/src/managers/spoke/QueueManager.sol:56, protocol/src/core/spoke/BalanceSheet.sol:213

**Summary/Description**  
MessageProcessor enables Gateway.unpaidMode for the full duration of inbound message handling. During that window, caller-controlled target code reached through trusted/untrusted contract update paths can call permissionless QueueManager.sync for another pool. QueueManager then submits queued BalanceSheet updates with zero payment; Gateway._send records the outbound batch as underpaid instead of reverting, while the BalanceSheet queues have already been cleared and QueueManager may refresh lastSync.

**Root Cause**  
Gateway.unpaidMode is a global transient switch for the entire inbound handler call stack. QueueManager.sync has no guard preventing execution while unpaidMode is active, and BalanceSheet.submitQueuedAssets/submitQueuedShares clear queued state before the eventual Gateway._send decides whether the batch is actually funded or only stored as underpaid.

**Pre_conditions**  
A target pool uses QueueManager on a spoke chain and has queued asset or share updates. A separate caller can cause arbitrary target code to execute on that spoke during MessageProcessor.handle, for example through a caller-controlled pool manager sending a TrustedContractUpdate to a target contract on the affected spoke chain.

**Impact**  
The caller can move another pool's queued sync into Gateway.underpaid without funding the outbound message. The hub does not receive the update until someone repays the exact batch, but the local queue has been cleared; normal keepers cannot simply call QueueManager.sync again for the same update. This can repeatedly delay balance-sheet synchronization, snapshots, NAV/share-price updates, and downstream operations that depend on timely synced state.

**Mitigation**  
Scope unpaidMode so only the intended internal forwarding paths can use it, or make QueueManager reject sync while Gateway.unpaidMode is active. Alternatively, defer clearing BalanceSheet queue state until Gateway.send is funded/sent, or store enough local state to let a funded retry resubmit without relying solely on Gateway.repay.

### [M-04] Manual OracleValuation price updates can be sandwiched before the matching asset-price notification

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Stale asset-price accounting can transfer value from existing share holders to a sync depositor  
**Location**: protocol/src/valuations/OracleValuation.sol:51, protocol/src/core/hub/Hub.sol:175, protocol/src/core/spoke/BalanceSheet.sol:201, protocol/src/vaults/SyncManager.sol:157

**Summary/Description**  
OracleValuation.setPrice updates the Hub valuation and can trigger SimplePriceManager share-price notifications, but it does not send the corresponding asset-price notification to the Spoke. Under the intended setup the feeder only has OracleValuation feeder permission while notifyAssetPrice is a Hub manager call, so a user can sandwich a manual price decrease by depositing through a sync vault before setPrice and syncing the queued deposit before the stale Spoke asset price is replaced.

**Root Cause**  
The asset price update and the Spoke asset-price notification are split across separate permission domains. setPrice writes the new OracleValuation price and calls hub.updateHoldingValue, while Hub.notifyAssetPrice must be called separately by a pool manager. BalanceSheet.submitQueuedAssets then prices queued deposits with the current Spoke price, which can still be the old price.

**Pre_conditions**  
A pool uses OracleValuation with NAVManager/SimplePriceManager, sync deposits are enabled with capacity, QueueManager sync is callable when the price update is observed, and the manual price change decreases the deposit asset price.

**Impact**  
A user can mint sync-deposit shares at the old share/asset ratio, let setPrice lower existing NAV/share price, then sync the queued asset increase using the old Spoke asset price. The Hub records too much pool value for the new deposit, inflating the post-sync share price and allowing the user to redeem or otherwise hold shares worth more than the correctly valued deposit.

**Mitigation**  
Make OracleValuation.setPrice atomically send or require the matching asset-price notification before queued asset syncs can use the old price. Alternatively gate sync deposits/QueueManager sync during manual price changes, or use an atomic manager-controlled price-update wrapper that updates OracleValuation and Spoke asset prices together.

### [M-05] Split asset-claim syncs can leave rounded value in NAV accounting

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Repeatable NAV/accounting overstatement from split asset withdrawals  
**Location**: protocol/src/misc/types/D18.sol:52, protocol/src/core/libraries/PricingLib.sol:187, protocol/src/core/hub/Holdings.sol:151, protocol/src/core/hub/Holdings.sol:169, protocol/src/core/hub/HubHandler.sol:77, protocol/src/core/spoke/BalanceSheet.sol:191, protocol/src/managers/spoke/QueueManager.sol:56, protocol/src/vaults/AsyncRequestManager.sol:438, protocol/src/managers/hub/NAVManager.sol:125, protocol/src/managers/hub/SimplePriceManager.sol:110

**Summary/Description**  
Holdings stores pool-currency value by adding or subtracting each queued asset delta after PricingLib conversion rounds down. A fulfilled async redemption can be claimed through multiple smaller withdraw calls, and QueueManager can sync each asset-only withdrawal separately. NAVManager then consumes the path-dependent rounded accounting value during snapshot sync and SimplePriceManager publishes a share price from that value, even though the user received the same aggregate asset payout.

**Root Cause**  
Holding value is maintained as path-dependent rounded deltas. increase()/decrease() apply convertWithPrice(amount) to each submitted net amount, while BalanceSheet and QueueManager allow one fulfilled payout to be split into independently synced withdrawal deltas. Hub calls the snapshot hook without recomputing holding value from total asset amount first, so NAVManager forwards the rounded residual into automated NAV pricing.

**Pre_conditions**  
A pool uses a non-integer asset price in pool minor units, async redemption payouts can be claimed in smaller asset chunks, and a QueueManager or balance-sheet manager syncs those asset-only withdrawals separately while no share delta is queued.

**Impact**  
The ledger can retain up to almost one pool minor unit per separately synced withdrawal. Repeating the sequence overstates NAV and can feed SimplePriceManager share-price updates, allowing remaining shares to be priced against accounting value that is no longer backed by escrowed assets. The issue is repeatable but bounded by per-sync dust and operational cost, so Medium severity is appropriate.

**Mitigation**  
For holding amount updates, compute the post-update value from the resulting total asset amount and journal the difference from the previous stored value, or carry explicit rounding residuals per holding. Alternatively force asset-claim withdrawals for a fulfilled payout to sync as one aggregate delta, and refresh holding valuation before snapshot hooks update share price.

### [M-06] MerkleProofManager policies cannot constrain ERC6909 token IDs

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: protocol/src/managers/spoke/decoders/BaseDecoder.sol:20, protocol/src/managers/spoke/decoders/BaseDecoder.sol:34, protocol/src/managers/spoke/MerkleProofManager.sol:67, protocol/src/core/spoke/BalanceSheet.sol:103, protocol/src/core/spoke/BalanceSheet.sol:121, protocol/src/core/spoke/PoolEscrow.sol:19

**Summary/Description**  
MerkleProofManager validates BalanceSheet calls from the bytes returned by the configured decoder. The bundled BaseDecoder for BalanceSheet deposit/withdraw returns poolId, scId, asset, and receiver, but omits tokenId. For ERC6909 holdings the protocol treats asset plus tokenId as the asset identity, so a strategist with a valid proof for one ERC6909 token ID can submit the same target and addresses with a different tokenId and still satisfy the proof.

**Root Cause**  
The policy leaf does not include all security-relevant BalanceSheet arguments. BaseDecoder ignores the tokenId argument even though BalanceSheet maps asset plus tokenId to AssetId and updates/withdraws that specific holding.

**Pre_conditions**  
A pool uses MerkleProofManager as a BalanceSheet manager; the strategist has a policy leaf allowing BalanceSheet deposit or withdraw for an ERC6909 token contract; the same ERC6909 contract has another registered/held tokenId for the pool/share class.

**Impact**  
A restricted strategist can move or account for an ERC6909 token ID outside the allowed policy scope. On withdraw, the call can decrement and transfer an unauthorized tokenId from the pool escrow to the policy-approved receiver; on deposit, it can queue/account a different assetId than the policy intended.

**Mitigation**  
Include tokenId, or the resolved AssetId, in the decoder output for BalanceSheet deposit/withdraw and any other asset-scoped calls. Policies should bind the exact asset identity used by BalanceSheet accounting, not only the token contract address.

### [M-07] Public NAV refresh can keep stale share prices valid

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Expired NAV-derived share prices can be refreshed without new NAV data and used for sync deposit pricing  
**Location**: protocol/src/managers/hub/NAVManager.sol:155, protocol/src/core/hub/Hub.sol:407, protocol/src/core/hub/Holdings.sol:127, protocol/src/managers/hub/SimplePriceManager.sol:111, protocol/src/core/spoke/Spoke.sol:271, protocol/src/vaults/SyncManager.sol:157

**Summary/Description**  
NAVManager.updateHoldingValue has no NAV-manager check. When a pool grants NAVManager Hub-manager rights, any caller can route through it and Hub sees NAVManager as the manager in Hub.updateHoldingValue. The call recomputes the holding value and, if the network snapshot flag is true, Hub calls Holdings.callOnSyncSnapshot even when the value diff is zero. NAVManager then forwards the existing NAV to SimplePriceManager, which writes a fresh block.timestamp share price and sends configured notifications. A local notified network is refreshed immediately; remote impact depends on the resulting Gateway notification being delivered.

**Root Cause**  
The public NAVManager.updateHoldingValue wrapper acts as a confused deputy for Hub.updateHoldingValue, Hub propagates snapshot hooks unconditionally after a holding valuation update, and SimplePriceManager timestamps the recomputed share price with block.timestamp instead of source NAV or valuation freshness.

**Pre_conditions**  
A pool uses NAVManager as a Hub manager with SimplePriceManager as the NAV hook, at least one initialized holding exists on a network whose Holdings snapshot flag is true, finite max share price age is relied on by Spoke or vault logic, and the affected Spoke price notification is local or otherwise delivered. The stale share price is economically favorable to the caller.

**Impact**  
An attacker can repeatedly call NAVManager.updateHoldingValue for an initialized holding after the legitimate share price should have expired. If holdings.update returns zero diff, accounting and NAV stay unchanged, but SimplePriceManager updates computedAt to the current block and the affected Spoke accepts the old NAV-derived price as valid. Sync deposits then convert assets to shares through SyncManager.convertToShares using the refreshed stale pool-per-share price, allowing excess share minting when the stale price is below the true NAV per share and transferring value from existing holders.

**Mitigation**  
Restrict NAVManager.updateHoldingValue to authorized pool/NAV managers, or only propagate the snapshot hook when NAV-relevant accounting or issuance actually changed. Prefer carrying a source NAV/valuation timestamp into SimplePriceManager and never refreshing computedAt from unchanged or stale inputs.

### [M-08] Unsynced sync deposits can be redeemed at prices computed without their shares

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Sync depositors can extract interim NAV gains by redeeming shares before their deposit is synced to the Hub  
**Location**: protocol/src/vaults/SyncManager.sol:198, protocol/src/core/spoke/BalanceSheet.sol:191, protocol/src/core/hub/Holdings.sol:111, protocol/src/managers/hub/NAVManager.sol:125, protocol/src/managers/hub/SimplePriceManager.sol:86, protocol/src/vaults/AsyncRequestManager.sol:124, protocol/src/vaults/BatchRequestManager.sol:331

**Summary/Description**  
SyncDepositVault deposits mint shares and add assets to the local BalanceSheet immediately, but the Hub snapshot flag is not marked stale until the later queued asset/share submissions arrive. While that local queue is still pending, Hub-side NAV updates can still treat the network as a synced snapshot and SimplePriceManager can publish a share price from Hub NAV/issuance that excludes the new deposit assets and shares. The depositor can then redeem the freshly minted local shares through the async redeem path at that overstated price before the deposit queue is synced.

**Root Cause**  
Local sync deposits only update BalanceSheet queuedAssets/queuedShares. The Hub-side Holdings snapshot state remains whatever it was before the deposit until submitQueuedAssets or submitQueuedShares is delivered. NAVManager and SimplePriceManager trust that stale snapshot flag, and the async redeem path does not require the just-minted shares to have been synced to ShareClassManager before a redeem request can be approved and revoked.

**Pre_conditions**  
A pool uses SyncDepositVault with async redemptions, NAVManager/SimplePriceManager pricing, and sufficient redemption liquidity. A sync deposit is made while the previous Hub snapshot flag is true, but QueueManager or the balance sheet manager has not yet submitted the queued deposit asset/share updates. A NAV/share-price update that is favorable to the depositor is published before the queued deposit is synced, and the redeem request is processed at that price.

**Impact**  
The user mints shares at the old price, waits for a price update computed from Hub state that excludes the user's unsynced assets and shares, then submits and claims an async redemption of those shares. When the redemption burns the shares locally it can net against the queued issuance, while the asset withdrawal nets against or exceeds the queued deposit. The excess payout is taken from pool escrow and later appears as a net holding decrease, diluting remaining holders. The value captured can scale with the unsynced deposit size and the interim NAV gain, bounded by maxReserve and available liquidity.

**Mitigation**  
Invalidate or consume the Hub snapshot before any NAV/share-price update can rely on a network with local queued BalanceSheet changes. Practical fixes include forcing sync-deposit asset/share submissions before newly minted shares can be redeemed, blocking async redeem requests while the share class has unsynced BalanceSheet queues, or making price/NAV update paths require a freshness proof that the relevant spoke queue is empty. Operational sequencing alone should not be the only protection.

### [M-09] Rootless refund escrows make subsidy recovery depend on old controller

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Native gas subsidies can become unusable by a replacement async manager; recovery depends on retaining old-controller access  
**Location**: protocol/script/FullDeployer.s.sol:215, protocol/src/vaults/factories/RefundEscrowFactory.sol:28, protocol/src/vaults/RefundEscrow.sol:21, protocol/src/vaults/AsyncRequestManager.sol:78

**Summary/Description**  
RefundEscrowFactory is designed to grant both root and controller permissions to each per-pool RefundEscrow, but FullDeployer only files the controller and never files root. Escrows created from this deployed factory therefore rely address(0) plus the current AsyncRequestManager, then remove the factory. In the normal state the current controller can still withdraw subsidies, so refunded gas is not automatically stuck. The gap is that root has no direct escrow backstop and cannot authorize a replacement controller for an already deployed deterministic escrow.

**Root Cause**  
FullDeployer omits refundEscrowFactory.file('root', root), while RefundEscrowFactory.newEscrow does not reject root == address(0). Permissions are copied only at escrow creation, so later factory controller/root changes do not update already deployed escrows.

**Pre_conditions**  
A pool has a deployed RefundEscrow with native subsidy balance; that escrow was created before a valid root was filed on the factory; the protocol migrates away from the original AsyncRequestManager/controller or otherwise loses authorized access through it before withdrawing or moving the balance.

**Impact**  
A replacement controller cannot call depositFunds or withdrawFunds on the existing deterministic refund escrow, so it cannot use that escrow for subsidized async requests. Existing native balance is recoverable through the old controller while an authorized actor can still call that old AsyncRequestManager, but if that path is revoked or unavailable the intended root recovery path is absent and the balance can be stranded. Per contest guidance, RefundEscrow native-token loss is Medium at most.

**Mitigation**  
Set refundEscrowFactory.root during deployment before any escrow can be created, require non-zero root and controller in newEscrow, and provide a root-authorized migration path for already deployed refund escrows to add or replace the controller. During upgrades, withdraw or migrate existing subsidy balances before revoking access to the old controller.

### [M-10] Pool-scoped adapter can reserve another pool's share-token salt

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Cross-pool share-class deployment denial on a spoke  
**Location**: protocol/src/core/spoke/factories/TokenFactory.sol:36, protocol/src/core/spoke/Spoke.sol:203, protocol/src/core/messaging/MultiAdapter.sol:103, protocol/src/core/messaging/libraries/MessageLib.sol:121, protocol/src/core/hub/ShareClassManager.sol:149

**Summary/Description**  
ShareClassManager makes salts globally unique on the hub path, but TokenFactory itself deploys by CREATE2 using only the caller-supplied salt and decimals. A pool-specific adapter message is authorized by the message pool id, yet Spoke.addShareClass lets that message consume any salt in the factory namespace for that spoke. A pool A adapter can therefore deploy a pool A share token with pool B's public salt before pool B is notified on the spoke.

**Root Cause**  
The token factory's CREATE2 namespace is global to the factory, while the spoke registry is only keyed by poolId and shareClassId. Spoke.addShareClass does not verify that the salt belongs to the hub-registered share class or that it has not already been consumed globally on that spoke. The pool-scoped adapter check only constrains the message pool id, not the global salt side effect.

**Pre_conditions**  
The actor controls a manager/adapter for pool A, pool A exists on the target spoke, pool B has a share class salt visible on the hub and has not yet deployed that share class on the target spoke, and pool A uses the same share-token decimals.

**Impact**  
Pool A can predeploy a token at the CREATE2 address that pool B later needs. Pool B's later NotifyShareClass reaches TokenFactory.newToken and reverts on CREATE2 address collision, preventing that share class from being deployed on the affected spoke with its immutable salt. This is cross-pool availability impact; recovery requires privileged migration or changing factory/wiring rather than normal pool-manager action.

**Mitigation**  
Namespace the CREATE2 salt with poolId and shareClassId inside TokenFactory or have Spoke track and reject globally used salts before deployment. Alternatively require AddShareClass messages to carry a hub-authenticated salt commitment for the exact poolId/shareClassId instead of allowing a pool-scoped message to consume an arbitrary factory salt.

### [M-11] Share-only redemption sync prices remaining shares against reserved payout assets

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Redeemers can extract value from remaining holders by using a NAV/share price that excludes burned shares but still includes their reserved payout assets  
**Location**: protocol/src/vaults/AsyncRequestManager.sol:280, protocol/src/core/spoke/BalanceSheet.sol:172, protocol/src/core/spoke/BalanceSheet.sol:225, protocol/src/core/hub/HubHandler.sol:100, protocol/src/core/hub/ShareClassManager.sol:78, protocol/src/core/hub/Holdings.sol:111, protocol/src/managers/hub/NAVManager.sol:125, protocol/src/managers/hub/SimplePriceManager.sol:148, protocol/src/core/spoke/PoolEscrow.sol:46

**Summary/Description**  
When an async redeem epoch is revoked, the spoke reserves payout assets and burns shares, but reservation does not queue an asset decrease. A public queue sync can submit the share burn with isSnapshot true because no asset delta is queued. The hub then reduces ShareClassManager issuance, leaves accounting NAV unchanged because the reserved assets remain in Holdings, and SimplePriceManager publishes NAV divided by reduced issuance. The next redemption can be processed at this inflated price.

**Root Cause**  
Reservation is treated only as a local liquidity lock, not as a NAV liability or decrease, while share revocation is synchronized as a complete snapshot. SimplePriceManager divides gross NAV including reserved redemption assets by active post-burn issuance.

**Pre_conditions**  
A pool uses async redemptions with NAVManager and SimplePriceManager, a redeem epoch is revoked and its share burn can be synced before the redeemed assets are withdrawn and synced, and a later redeem is processed using the current automated share price.

**Impact**  
Example with asset price 1: a pool has 100 assets and 100 shares. The attacker redeems 50 shares, the revoke callback reserves 50 assets and burns 50 shares, then the attacker or keeper syncs the share burn before any asset withdrawal. The hub now prices 100 NAV over 50 active shares, so the share price becomes 2 even though 50 assets are already owed to the first redemption. The attacker redeems 25 more shares at price 2, reserving another 50 assets. Total reserved assets equal the full pool balance, so the attacker can withdraw 100 assets for 75 shares and leave the remaining 25 shares with no backing.

**Mitigation**  
Do not mark share-only redemption revocation as a complete pricing snapshot while reserved payout assets remain in NAV. Queue a matching asset decrease or liability at revoke time, exclude reserved redemption assets from NAV, or keep claimable redeemed shares in the price denominator until the payout asset decrease is synced. Also make PoolEscrow.reserve enforce total >= reserved to prevent over-reservation.

### [M-12] Adapter reentrancy can drop queued sync messages from gateway batches

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Dropped QueueManager sync messages and hub/spoke accounting desync  
**Location**: protocol/src/core/messaging/Gateway.sol:230, protocol/src/core/messaging/Gateway.sol:258, protocol/src/core/messaging/Gateway.sol:262, protocol/src/core/messaging/Gateway.sol:270, protocol/src/core/messaging/Gateway.sol:276, protocol/src/core/messaging/MultiAdapter.sol:172, protocol/src/managers/spoke/QueueManager.sol:56, protocol/src/core/spoke/BalanceSheet.sol:213, protocol/src/core/hub/Holdings.sol:116

**Summary/Description**  
Gateway.withBatch intentionally keeps isBatching true until the outer batch is drained. During _endBatching, Gateway snapshots and clears the locator array, then performs external adapter sends while isBatching is still true. A pool-specific custom adapter can reenter during its send and call permissionless QueueManager.sync for another pool. The nested sync clears BalanceSheet queue state and calls Gateway.send, but the new outbound message is appended after the locator snapshot and is never processed by the current _endBatching call.

**Root Cause**  
Gateway's batching state is global and remains active while _endBatching performs external adapter calls. Reentrant nested batches are treated as part of the active batch even though the set of locators to process was already snapshotted, so messages appended during the drain window can be silently discarded instead of sent or stored as underpaid.

**Pre_conditions**  
A victim pool uses QueueManager and has queued asset/share updates ready to sync. The actor controls a custom adapter configured for another pool and triggers an outgoing message for that pool. The custom adapter reenters during MultiAdapter.send and calls QueueManager.sync for the victim pool.

**Impact**  
The victim pool's BalanceSheet clears queued asset/share deltas and increments its sync nonce, but the corresponding hub update message is not dispatched and is not recorded as underpaid. The hub snapshot nonce remains behind, so later legitimate sync messages for that share class can revert with InvalidNonce, leaving hub/spoke accounting desynchronized until privileged recovery or manual correction. The behavior is repeatable against pools with pending QueueManager work.

**Mitigation**  
Do not leave Gateway accepting batched sends while an outer batch is draining. Add a drain-phase reentrancy guard that rejects withBatch/Gateway.send batching during _endBatching, or scope batching to the current batch owner and process/clear all reentrant locators deterministically before isBatching is disabled. Also consider moving external adapter calls after internal batch state is finalized or recording reentrant sends as normal standalone/underpaid sends instead of transient batch entries.

### [M-13] Gateway batches let one message consume gas reserved for later messages

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Repeatable failure or delay of later same-pool batched messages, including QueueManager sync updates  
**Location**: protocol/src/core/messaging/Gateway.sol:102, protocol/src/core/messaging/Gateway.sol:116, protocol/src/core/messaging/Gateway.sol:152, protocol/src/core/messaging/Gateway.sol:159, protocol/src/core/messaging/Gateway.sol:166, protocol/src/core/spoke/Spoke.sol:159, protocol/src/managers/spoke/QueueManager.sol:56, protocol/src/core/utils/ContractUpdater.sol:33

**Summary/Description**  
Gateway.send sums the gas limits of messages appended to a batch, but the batch payload only contains concatenated messages. On the destination chain, Gateway.handle iterates the concatenated messages and forwards nearly all remaining gas to each processor.handle call. A caller can place a gas-heavy same-pool UntrustedContractUpdate first, then append a permissionless QueueManager.sync for that pool in the same gateway batch. The first handler can consume the gas that was paid for the later sync update messages.

**Root Cause**  
The batching format preserves only total batch gas, not each message's gas reservation. Gateway.handle therefore cannot cap a message to the gas paid for that message and instead calls processor.handle with gasleft() - GAS_FAIL_MESSAGE_STORAGE for every item.

**Pre_conditions**  
A pool has queued BalanceSheet updates that QueueManager.sync can submit. The attacker deploys a target whose untrustedCall burns most forwarded gas and returns. The attacker uses Gateway.withBatch to call Spoke.updateContract for that pool first and QueueManager.sync for the same pool second.

**Impact**  
The source-chain sync clears BalanceSheet queued asset/share deltas and may advance QueueManager state, but the corresponding hub update messages can fail or be left for bridge/protocol retry because the first message consumed their gas. Until the missing batch is retried, hub/spoke accounting is stale and later nonce-ordered syncs can be blocked. The attack is repeatable against queued sync work for pools that expose QueueManager syncing.

**Mitigation**  
Preserve and enforce per-message gas limits in batched payloads. During Gateway.handle, reserve enough gas for the remaining messages and call each processor.handle with only that message's paid gas allowance. Alternatively, avoid concatenating messages into a single unmetered execution context or prevent arbitrary permissionless messages from sharing a batch with QueueManager sync updates.

