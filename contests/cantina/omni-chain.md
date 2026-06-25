### [H-01] OmniBridgeNative l1BridgeBalance absolute sync can erase pending L1 withdrawal reservations

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni/contracts/core/src/token/OmniBridgeNative.sol:95; omni/contracts/core/src/token/OmniBridgeNative.sol:127; omni/contracts/core/src/token/OmniBridgeL1.sol:87; omni/contracts/core/src/xchain/OmniPortal.sol:256

**Summary/Description**  
OmniBridgeL1.bridge encodes token.balanceOf(L1Bridge)+amount as an absolute l1Balance for OmniBridgeNative.withdraw. On execution, OmniBridgeNative.withdraw overwrites l1BridgeBalance with that snapshot, while OmniBridgeNative.bridge decrements l1BridgeBalance immediately for native-to-L1 reservations. Because L1-to-Omni and Omni-to-L1 streams are independently ordered, and because a destination callback can create a native-to-L1 reservation before later L1-to-Omni messages in the same batch execute, a later absolute L1 snapshot can restore already-reserved liquidity. The snapshot can be stale whether the corresponding L1 withdrawal is still pending or has already executed after the snapshot was taken.

**Root Cause**  
The native bridge stores only an absolute synchronized L1 escrow balance and does not track pending native-to-L1 reservations separately. A later L1 snapshot assignment at OmniBridgeNative.withdraw line 105 can overwrite prior decrements made by OmniBridgeNative.bridge line 130.

**Pre_conditions**  
At least one native-to-L1 bridge reservation has been accepted on Omni, either through a normal bridge call or a callback during L1-to-Omni execution; then an L1-to-Omni bridge message carrying an absolute L1 escrow snapshot that does not account for that reservation executes on Omni. This can happen before or after the corresponding L1 withdrawal executes, since the snapshot is captured when the L1 bridge transaction is created and the opposite-direction streams have no shared ordering.

**Impact**  
After the overwrite, OmniBridgeNative.bridge checks amount <= l1BridgeBalance against overstated liquidity and accepts additional native OMNI. When L1 later executes native-to-L1 withdrawals in source-stream order, earlier withdrawals can drain the L1 bridge escrow and later accepted withdrawals revert. OmniPortal records ordinary destination reverts as failed XReceipts after incrementing the inbound offset, so the user whose native OMNI was accepted on Omni receives no L1 OMNI and has no bridge refund path.

**Proof of Concept**  
Reasoning sequence: start with L1 escrow and native l1BridgeBalance at 100. User A bridges 100 native to L1, so native l1BridgeBalance becomes 0 while the L1 withdrawal xmsg is pending and the L1 snapshot basis still includes those tokens. User B bridges 1 OMNI from L1 to native; the L1 message carries l1Balance=101. If B's message is delivered to Omni after A's reservation, OmniBridgeNative.withdraw sets l1BridgeBalance=101, even if A's L1 withdrawal is still pending or has already executed after B's snapshot was taken. User C can now bridge 101 native to L1. L1 source-stream order makes A's 100 execute before C's 101; after A drains the escrow down to 1, C's transfer reverts and OmniPortal consumes the failed receipt. The same overwrite can occur inside one Omni xsubmit if an L1-to-Omni receiver callback bridges native to L1 before a later L1-to-Omni message overwrites l1BridgeBalance with its absolute snapshot.

**Mitigation**  
Do not overwrite available liquidity with raw L1 escrow balance while outbound native-to-L1 messages may be pending. Track pending native-to-L1 reservations separately and compute available liquidity as synchronized L1 escrow minus pending reservations, or change L1-to-native sync to apply deltas that cannot erase native-side reservations. Consider adding an explicit refund/claim path for failed L1 withdrawals.

### [H-02] Malformed staking pubkeys can reach validator-set processing

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni/contracts/core/src/octane/Staking.sol:89; omni/halo/evmstaking/evmstaking.go:149; omni/lib/k1util/k1util.go:115; omni/halo/valsync/keeper/keeper.go:231

**Summary/Description**  
Staking.createValidator only enforces a 33-byte pubkey and evmstaking wraps the bytes as a Cosmos secp256k1 key without decompressing or checking the point. Cosmos SDK staking accepts the key type/length and can admit the validator; later valsync decompresses validator pubkeys while storing validator-set updates and returns an error for malformed compressed points.

**Root Cause**  
EVM admission and evmstaking conversion perform length-only validation, so on-curve validation is deferred until downstream validator-set processing.

**Pre_conditions**  
Validator creation is open, or the caller is on the staking allowlist; caller sends at least MinDeposit with a 33-byte malformed compressed secp256k1 value.

**Impact**  
A malformed validator pubkey can create a non-signing/invalid validator update path. In this codebase valsync returns an error while processing the validator update, which can block finalization for the block containing the delivered create-validator event. If the allowlist is later disabled, any funded caller can trigger the path.

**Mitigation**  
Validate compressed secp256k1 pubkeys on the EVM side or in evmstaking before emitting/delivering CreateValidator. Decompress the key and reject malformed or off-curve points before any validator admission or deposit finalization.

### [H-03] Late precommit vote extensions bypass app validation and can poison the next proposal

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: github.com/cometbft/cometbft@v0.38.12/consensus/state.go:2137; omni/halo/attest/keeper/cpayload.go:33; omni/halo/attest/keeper/cpayload.go:82; omni/halo/attest/keeper/keeper.go:745; omni/halo/attest/keeper/keeper.go:891

**Summary/Description**  
CometBFT verifies Omni vote extensions through VerifyVoteExtension for current-height non-nil precommits before adding them to cs.Votes. However, previous-height precommits received while the node is already at the next height are added to cs.LastCommit directly and return before the VerifyVoteExtension branch. When that node later proposes, createProposalBlock builds LocalLastCommit from cs.LastCommit, and Omni PrepareVotes only calls baseapp.ValidateVoteExtensions, which checks consensus extension signatures but not Omni's vote-extension contents. The unverified extension is then decoded and aggregated into MsgAddVotes.

**Root Cause**  
Omni treats LocalLastCommit as already app-validated, but CometBFT's previous-height LastCommit path only validates consensus vote and extension signatures. votesFromLastCommit does not reapply VerifyVoteExtension's content checks before constructing the proposed MsgAddVotes.

**Pre_conditions**  
A validator withholds a precommit for block H until after other validators have committed H and moved to height H+1. The late precommit is for the committed block and has valid consensus vote and extension signatures, but its Omni vote-extension payload would be rejected by VerifyVoteExtension, for example duplicate votes for the same attestation header/root or more than vote_extension_limit votes.

**Impact**  
If an honest H+1 proposer has the late vote in cs.LastCommit, PrepareVotes can include an invalid MsgAddVotes. Other validators then reject the proposal in ProcessProposal because verifyAggVotes sees duplicate validator signatures or per-validator count overflow. If the bad late precommit is propagated to all validators' LastCommit sets, every proposer for that height can keep producing rejectable proposals, causing a repeatable consensus liveness failure by a single Byzantine validator. Variants with duplicate same-header but different roots can also pass ProcessProposal and create useless pending attestation roots before signature insertion drops the duplicate signer.

**Proof of Concept**  
Code review only. The path is: CometBFT addVote previous-height branch adds vote.Height+1 == cs.Height precommits to cs.LastCommit and returns before VerifyVoteExtension; createProposalBlock uses cs.LastCommit.MakeExtendedCommit for the next proposal; PrepareVotes validates only extension signatures via baseapp.ValidateVoteExtensions; votesFromLastCommit decodes/aggregates without checking duplicate AttestHeader, per-extension limit, or that the vote signature address matches the commit validator; proposalServer.AddVotes later rejects duplicate same-root signatures or count overflow.

**Mitigation**  
Refactor VerifyVoteExtension's payload checks into a shared validator and run it from PrepareVotes/votesFromLastCommit for every LocalLastCommit extension before aggregation, using the commit validator address to enforce signer identity. Invalid late extensions should be skipped or otherwise prevented from producing an invalid proposal. Also reject same-validator duplicate attestation headers across aggregate roots before storage.

### [H-04] ProcessProposal retries deterministic Engine API V3 parameter errors forever

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni/octane/evmengine/keeper/abci.go:98; omni/octane/evmengine/keeper/abci.go:116; omni/octane/evmengine/keeper/keeper.go:113; omni/octane/evmengine/keeper/proposal_server.go:27; omni/octane/evmengine/keeper/msg_server.go:37; omni/octane/evmengine/keeper/msg_server.go:172; omni/halo/app/prouter.go:78; omni/halo/genutil/evm/evm.go:68

**Summary/Description**  
MsgExecutionPayload carries raw JSON for geth engine.ExecutableData. parseAndVerifyProposedPayload checks JSON-required fields via unmarshal plus withdrawals length, fee recipient, parent/height, timestamp, and prevRandao, but it does not require the Engine API V3 fields that geth requires on Cancun payloads: withdrawals must be non-nil (and empty for Omni), blobGasUsed must be non-nil, and excessBlobGas must be non-nil. Missing or null optional V3 fields therefore pass local checks and reach engine_newPayloadV3. In go-ethereum v1.14.11, those cases return a JSON-RPC InvalidParams error; Omni's Engine client exposes only the error and a zero status, and both proposal and finalization handlers classify every pushPayload error as retryable.

**Root Cause**  
The evmengine handlers conflate deterministic Engine API JSON-RPC validation errors with transient transport errors. Local payload validation uses len(withdrawals)>0, which allows nil withdrawals, omits blobGasUsed/excessBlobGas pointer checks, and has no non-retryable classifier for Engine API errors such as InvalidParams or UnsupportedFork. Because go-ethereum's JSON-RPC handler returns an error response without the method result when the method returns an error, the INVALID status returned by the local API method is not available to Omni's RPC client.

**Pre_conditions**  
A proposer includes a MsgExecutionPayload whose JSON has all required ExecutableData fields and otherwise matches the current execution head, next block number, allowed fee recipient, timestamp bounds, and prevRandao rule, but omits or nulls withdrawals, blobGasUsed, or excessBlobGas. The proposer does not need the payload to reconstruct to a valid block hash, because geth rejects these V3 parameter shapes before the later ExecutableDataToBlock validation path.

**Impact**  
Honest validators processing the proposal enter retryForever around engine_newPayloadV3 instead of returning REJECT. CometBFT v0.38.12 calls ProcessProposal with context.TODO(), so this deterministic validation error has no normal round-time cancellation at the application boundary and can halt progress at the height. The same error classification exists in msgServer.ExecutionPayload during FinalizeBlock, so an accepted malformed payload or equivalent integration mismatch can also stall execution payload finalization.

**Proof of Concept**  
PoC not run per instruction. Code path: parseAndVerifyProposedPayload accepts omitted/null withdrawals because len(nil)==0 and does not check blobGasUsed or excessBlobGas. pushPayload calls NewPayloadV3 with the parsed payload, empty non-nil versionedHashes, and non-nil app-hash beacon root. go-ethereum v1.14.11 NewPayloadV3 returns InvalidParams for nil withdrawals, nil blobGasUsed, or nil excessBlobGas. engineclient.NewPayloadV3 wraps that RPC error, and proposalServer.ExecutionPayload/msgServer.ExecutionPayload treat err != nil as retryable by returning false,nil to retryForever. By contrast, malformed executable-data cases that fail ExecutableDataToBlock, such as bad transaction bytes, logsBloom length, extraData length, blockhash mismatch, or blob versioned-hash mismatch, return PayloadStatus INVALID with nil RPC error in go-ethereum v1.14.11 and are rejected by Omni's isInvalid branch in ProcessProposal.

**Mitigation**  
Prevalidate all Engine API V3 mandatory fields before pushPayload: require withdrawals to be non-nil and empty, require blobGasUsed and excessBlobGas to be non-nil and consistent with Omni's supported blob policy, and reject unsupported future payload fields before calling the execution client. Add explicit Engine API error classification so deterministic JSON-RPC errors such as InvalidParams, UnsupportedFork, invalid payload attributes, and invalid forkchoice state reject proposals or fail finalization as non-retryable bugs; reserve retryForever for transport/timeouts and syncing/unknown-status cases.

### [H-05] Rejected create-validator events permanently lock accepted EVM deposits

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni/contracts/core/src/octane/Staking.sol:89; omni/halo/evmstaking/evmstaking.go:149; omni/halo/evmstaking/evmstaking.go:160; omni/halo/evmstaking/evmstaking.go:189; omni/octane/evmengine/keeper/evmmsgs.go:17; omni/octane/evmengine/keeper/msg_server.go:111; omni/octane/evmengine/keeper/msg_server.go:142

**Summary/Description**  
Staking.createValidator accepts value and emits CreateValidator before checking whether Cosmos staking will accept the validator. During delivery, duplicate operator-address events are skipped before minting, while duplicate consensus-pubkey events can fail inside SDK MsgCreateValidator after evmstaking has minted and sent stake coins in the cached event branch. Same-payload duplicate-pubkey collisions are resolved by evmEvents' lexicographic sort rather than EVM log order, so a later copied-pubkey event can be delivered first and win validator admission. deliverEvents discards failed branches and continues, while the EVM deposit is already final and has no refund/recovery path.

**Root Cause**  
The EVM staking contract records only an event and has no pending/existing validator accounting or refund path. The event delivery layer treats downstream failures as non-fatal skipped events, so consensus rejection does not unwind the EVM value transfer.

**Pre_conditions**  
Caller passes Staking.createValidator checks with at least MinDeposit. Concrete paths include an existing validator address calling createValidator again, a different allowed/open validator address submitting a pubkey whose consensus address is already registered, or two same-payload createValidator events using the same consensus pubkey where the lexicographically earlier sorted event is delivered first.

**Impact**  
Each rejected accepted event locks 100% of the affected validator deposit in the staking predeploy without minting a corresponding successful Cosmos validator position. The minimum affected amount is 100 OMNI per event, and copied-pubkey or collision paths can strand any larger supplied validator deposit. Under the severity-assessment fund-impact rule, the affected asset base is the depositor's validator deposit rather than global TVL; because the affected depositor can lose the full deposit and the issue hits the create-validator deposit flow, the impact is High.

**Mitigation**  
Mirror all deterministic consensus-side admission checks before accepting value, or add a recoverable pending-deposit/refund mechanism keyed by event result. At minimum, reject existing validator addresses and already-used consensus pubkeys before accepting deposits, and make delivery failures produce an explicit recovery state instead of silently skipping.

### [H-06] Same-block validator creation can cause subsequent self-delegation to be skipped

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/core/src/octane/Staking.sol:103; halo/evmstaking/evmstaking.go:203; octane/evmengine/keeper/evmmsgs.go:44; octane/evmengine/keeper/msg_server.go:141

**Summary/Description**  
A validator can have a Delegate event and its first successful CreateValidator event included in the same previous EVM payload. The EVM side accepts both transactions and emits both events, but Halo sorts collected EVM events by address/topics/data rather than preserving log order. Because Delegate event topic 0x510b... sorts before CreateValidator topic 0xc7ab..., deliverDelegate can run before the validator exists in Cosmos staking, return an error, and be permanently skipped by deliverEvents. The later CreateValidator event can then succeed, leaving the delegate value locked on the EVM predeploy without minted/delegated stake.

**Root Cause**  
Cross-layer staking delivery does not preserve EVM log order and treats failed event delivery as non-retriable. Staking.delegate also accepts self-delegation based only on EVM-side checks even when the corresponding Cosmos validator is not yet present.

**Pre_conditions**  
A whitelisted or allowlist-disabled validator has a Delegate event included in the same previous EVM payload as its first successful CreateValidator event before the create has been delivered to Cosmos. This includes the user-visible createValidator-then-delegate EVM order because evmEvents reorders same-contract logs by topic bytes.

**Impact**  
The affected validator loses 100% of the additional delegation amount from that Delegate event: native OMNI is transferred to the predeploy, but no Cosmos stake is minted or delegated because the failed event branch is discarded and never retried. The affected asset base is the validator's delegated amount, not protocol-wide TVL; any meaningful delegation loses more than 1% of the affected assets, and the path can repeat for validators that reasonably create and self-delegate in the same payload. Final impact is High.

**Proof of Concept**  
Not run; code review only. The ordering follows from Delegate event topic 0x510b... sorting before CreateValidator topic 0xc7ab..., deliverDelegate requiring GetValidator success, and deliverEvents continuing without writing or retrying on delivery error.

**Mitigation**  
Preserve canonical EVM log order when collecting/delivering events, or make dependent staking events retryable/idempotent. Alternatively reject or queue Delegate events until the validator exists, and expose a recovery path for accepted EVM value whose Halo delivery fails.

### [H-07] Malleated duplicate attestation signatures can make accepted consensus payload transactions fail

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni/lib/k1util/k1util.go:51; omni/halo/attest/keeper/keeper.go:1077; omni/halo/attest/keeper/keeper.go:899; omni/octane/evmengine/keeper/abci.go:122

**Summary/Description**  
Attestation signature verification accepts a 65-byte ECDSA signature if it recovers the declared validator address and has V set to 27 or 28. It does not canonicalize the signature or treat a same-root/same-validator repeat as an idempotent duplicate. A validator can therefore submit a byte-different valid signature for an attestation root that is already stored. VerifyVoteExtension accepts the vote because it is individually valid and in-window, and ProcessProposal accepts the aggregate because verifyAggVotes checks only the current aggregate, not existing signature state. During FinalizeBlock, Keeper.addOne hits the unique att_id,validator_address key, isDoubleSign finds the same attestation with different signature bytes, and returns "different signature for identical vote [BUG]". Since MsgAddVotes is the first message in the unsigned consensus payload transaction, that error makes the transaction fail and prevents the execution payload message in the same transaction from being delivered.

**Root Cause**  
The code treats signature bytes as identity for an already-stored logical vote while accepting non-canonical ECDSA encodings. Duplicate insertion handling only ignores byte-identical duplicates and classifies byte-different duplicates for the same attestation ID as a BUG error.

**Pre_conditions**  
A signature for a pending or recently approved attestation is already stored, and the same validator later includes a byte-different valid signature for that same attestation root while the offset remains inside the vote window. The repeated signature can arrive through an otherwise accepted vote extension and does not require the validator to be the block proposer.

**Impact**  
A single validator can make an otherwise accepted consensus payload transaction fail at FinalizeBlock for any proposer that aggregates the repeated vote. Because MsgAddVotes is bundled before MsgExecutionPayload in the unsigned consensus payload transaction, the failed first message prevents the EVM execution payload message from being delivered for that block. Repeating this within the vote window can keep execution payload finalization and dependent protocol progress from advancing even while consensus blocks finalize, affecting shared execution and settlement state. The strongest realistic outcome is repeatable disruption of a core protocol flow, so final impact is High.

**Mitigation**  
Require canonical low-S signatures in SigTuple.Verify or k1util.Verify before recovery, normalize signatures before storage/comparison, and make addOne ignore any valid duplicate for the same attestation root and validator address based on logical identity rather than byte equality.

### [M-01] ProcessProposal accepts zero-message consensus transactions

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: omni/halo/app/prouter.go:56; omni/halo/app/prouter.go:62; omni/halo/app/prouter.go:85; omni/octane/evmengine/keeper/abci.go:140; omni/octane/evmengine/keeper/keeper.go:140; omni/octane/evmengine/keeper/db.go:58; omni/halo/genutil/defaults.go:10

**Summary/Description**  
The custom ProcessProposal handler decodes each proposed transaction and validates only the messages returned by tx.GetMsgs. A proposal with no transactions, or a decoded transaction with zero messages, performs no message checks and can still reach ACCEPT. The same max-count map caps MsgExecutionPayload and MsgAddVotes but never requires them after the loop, so a proposer can omit required consensus lifecycle messages or add zero-message filler transactions alongside an otherwise valid payload.

**Root Cause**  
Proposal validation enforces only per-message maximum counts while iterating messages that exist. It does not reject len(req.Txs)==0 after the initial block, does not reject decoded transactions with len(tx.GetMsgs())==0, does not enforce required final counts for MsgExecutionPayload/MsgAddVotes, and does not apply proposal byte/resource accounting to useless decoded transactions.

**Pre_conditions**  
A proposer crafts the consensus block bytes instead of using the local PrepareProposal output. The included zero-message transactions decode under the configured Cosmos tx decoder and the proposal otherwise satisfies the last-commit vote-extension quorum check.

**Impact**  
Validators can accept proposals that skip the EVM execution payload lifecycle for a consensus height, or proposals padded with useless transactions. A no-payload block finalizes without calling evmengine.ExecutionPayload, leaving the stored execution head unchanged while consensus height advances; the next valid payload is only checked against the stale execution head. Zero-message filler transactions fail later in FinalizeBlock with 'must contain at least one message', but the block still finalizes and the wrapper only logs the failed tx result. This is a repeatable proposer-controlled degradation of core execution and attestation progress, but it does not by itself prove direct fund loss or persistent system-wide halt, so final impact is Medium.

**Proof of Concept**  
Code review only. PrepareProposal normally returns one unsigned transaction containing MsgAddVotes and MsgExecutionPayload, but ProcessProposal independently loops over req.Txs, decodes each raw transaction, and only checks messages inside tx.GetMsgs. If req.Txs is empty, or if a decoded transaction has zero messages, there are no message type checks, no count decrement, and no final required-count check before ACCEPT. The existing TestProcessProposalRouter includes a 'no messages' case that expects ACCEPT.

**Mitigation**  
Reject empty proposals after the intentionally empty initial block, reject decoded transactions with zero messages, and enforce required proposal contents after processing all transactions. At minimum require exactly one MsgExecutionPayload for non-initial blocks and apply the intended MsgAddVotes policy. Consider requiring the consensus messages to appear in the canonical single transaction produced by PrepareProposal, and reject or cap unrelated/filler transaction bytes before returning ACCEPT.

### [M-02] PostFinalize validator RPC failures can fail finalized blocks

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: omni/halo/app/abci.go:101; omni/halo/app/abci.go:111; omni/octane/evmengine/keeper/abci.go:169; omni/octane/evmengine/keeper/abci.go:183; omni/octane/evmengine/keeper/keeper.go:177; omni/halo/comet/comet.go:73; omni/halo/config/config.go:43

**Summary/Description**  
The ABCI wrapper calls EVMEngKeeper.PostFinalize after BaseApp FinalizeBlock succeeds and returns any PostFinalize error to CometBFT. Optimistic EVM building is enabled by default, so PostFinalize queries the local Comet RPC validator set at the just-finalized height to guess whether this node is the next proposer. Errors, unavailable validator data, or inconsistent validator responses are propagated even though the optimistic build itself is only a performance optimization.

**Root Cause**  
A best-effort optimistic-build callback is placed on the critical FinalizeBlock return path and is allowed to fail the ABCI call. The callback depends on nondeterministic external Comet RPC Status/Validators responses instead of treating validator lookup failures as skip-optimistic-build conditions.

**Pre_conditions**  
evm-build-optimistic is enabled, as in the default config; the app has a Comet API client set; during PostFinalize, the local Comet RPC Status or Validators query returns an error, reports the current height unavailable, returns an empty or inconsistent validator set, or cannot map the next proposer key.

**Impact**  
A validator whose local callback query fails returns a FinalizeBlock error after the block's BaseApp execution already succeeded, which can stop that node from progressing. If the same local RPC failure mode affects a large validator set, finalization liveness can halt. The successful branch only mutates the in-memory optimistic payload cache and the execution client build pipeline, so this issue is a liveness failure rather than a consensus-store divergence. Under severity-assessment, impact is High but likelihood remains Low because it depends on local RPC failure conditions affecting enough voting power; final severity is Medium.

**Proof of Concept**  
Code review only. abciWrapper.FinalizeBlock returns resp, err when l.postFinalize(sdkCtx) fails. PostFinalize calls isNextProposer when buildOptimistic is true. isNextProposer calls k.cmtAPI.Validators(ctx, currentHeight), and errors or !ok become returned errors. The production config defaults EVMBuildOptimistic to true and app startup wires a local RPC-backed comet.API.

**Mitigation**  
Make PostFinalize strictly best-effort: log validator lookup, proposer prediction, and startBuild failures but return nil to FinalizeBlock. If next-proposer prediction is needed, derive it from deterministic consensus data already available to the app or keep failures local to the optimistic payload cache. FinalizeBlock should not fail because an optional optimistic build cannot be started.

### [M-03] Aggregate vote proposal verification does expensive signature recovery before cheap duplicate and count bounds

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: omni/halo/attest/keeper/keeper.go:891

**Summary/Description**  
ProcessProposal routes MsgAddVotes to verifyAggVotes, which calls AggVote.Verify for every aggregate before checking duplicate attestation roots, validator membership, or per-validator vote_extension_limit counts. AggVote.Verify performs ECDSA public-key recovery for each signature, so a byzantine proposer can submit a single allowed MsgAddVotes with many duplicate or otherwise invalid roots/signatures and force every validator to do excessive verification work before the proposal is rejected.

**Root Cause**  
verifyAggVotes orders expensive cryptographic verification before cheap structural bounds: no early len(aggs) or total signature cap, duplicate-root screening is after AggVote.Verify, and vote_extension_limit enforcement is after signature recovery.

**Pre_conditions**  
A validator proposer crafts the unsigned consensus payload MsgAddVotes instead of using the local PrepareVotes output. The payload fits the block byte limit and contains signatures that pass or at least reach ECDSA recovery.

**Impact**  
Repeated proposer slots can waste validator CPU during ProcessProposal and degrade consensus liveness. Normal honest PrepareVotes output is bounded by VerifyVoteExtension and vote_extension_limit, but proposed bytes are independently verified and do not receive those cheap checks first.

**Mitigation**  
In verifyAggVotes, pre-scan aggregate headers and signature metadata before ECDSA recovery: cap total aggregates/signatures to active validators * voteExtLimit, reject duplicate attestation roots before verifying signatures, reject duplicate validator addresses per aggregate by address before recovery, and enforce countsPerVal/validator membership as early as possible.

### [M-04] Aggregate vote validation allows repeated pending fake roots for one attestation header

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: omni/halo/attest/keeper/keeper.go:891; omni/halo/attest/keeper/keeper.go:924; omni/halo/attest/keeper/keeper.go:181; omni/halo/attest/keeper/keeper.go:234; omni/halo/attest/keeper/attestation.proto:44; omni/halo/attest/keeper/attestation.proto:45

**Summary/Description**  
Proposal-time aggregate vote validation de-duplicates only by full attestation root and enforces vote_extension_limit only as a per-validator signature count. It does not reject the same validator signing many different roots for the same attestation header. During finalization, Keeper.addOne inserts the attestation row before inserting the signature; when the signature insert later hits the unique chain/conf/offset/validator index, the duplicate signature is skipped but the newly inserted attestation row remains pending with no useful signature.

**Root Cause**  
The validator limit is modeled as a raw count of aggregate signatures, not as unique attestation headers per validator matching the vote-extension semantics. The storage path is not atomic at the logical vote level: attestation rows are created before the signature uniqueness check proves whether the aggregate contributes a usable vote.

**Pre_conditions**  
A validator proposer, or a proposer with signatures from one or more validators, includes a MsgAddVotes containing multiple AggVote entries with distinct block header or message root values but the same source chain, confirmation level, attestation offset, and signer. Each aggregate is individually signed and within the vote window.

**Impact**  
A single validator can cause up to vote_extension_limit pending fake attestation roots per accepted proposal for the same logical attestation slot; with the configured limit of 256 and the network MaxValidators setting of 30, the accepted shape scales up to 7,680 aggregate roots per block if multiple validators participate. These rows cannot reach quorum unless a quorum signs the same fake root, but they consume proposal verification CPU, finalize-time storage writes, pending-attestation iteration, and later pruning work until they leave the vote window. This is a repeatable shared-state resource pressure path from sub-quorum validator behavior.

**Proof of Concept**  
Code review only. verifyAggVotes tracks duplicate aggregate roots but not duplicate attestation headers per validator. It increments countsPerVal for every signature and allows counts up to k.voteExtLimit. addOne inserts a new attestation by unique attestation_root before inserting signatures. For a second root with the same chain/conf/offset/signer, the signature insert hits the unique chain_id,conf_level,attest_offset,validator_address index and is ignored as a duplicate slashable vote, leaving the new attestation row pending without that signature.

**Mitigation**  
In verifyAggVotes, track (validator, consensus chain, source chain, conf level, attest offset) and reject duplicates across aggregate roots before signature recovery or storage. Alternatively normalize aggregate votes by validator/header and drop duplicate-root variants deterministically before constructing or accepting MsgAddVotes. In addOne, avoid persisting a new attestation until at least one signature has been proven insertable, or delete the row if all signatures are duplicates.

### [M-05] ProcessProposal accepts syncing NewPayload without forkchoice sync and can retry event-log reads forever

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: omni/octane/evmengine/keeper/proposal_server.go:27; omni/octane/evmengine/keeper/proposal_server.go:37; omni/octane/evmengine/keeper/proposal_server.go:49; omni/octane/evmengine/keeper/evmmsgs.go:21; omni/halo/evmstaking/evmstaking.go:75; /Users/y4y/go/pkg/mod/github.com/ethereum/go-ethereum@v1.14.13/eth/catalyst/api.go:887; /Users/y4y/go/pkg/mod/github.com/ethereum/go-ethereum@v1.14.13/eth/catalyst/api.go:1017; /Users/y4y/go/pkg/mod/github.com/ethereum/go-ethereum@v1.14.13/eth/catalyst/api.go:1031; /Users/y4y/go/pkg/mod/github.com/cometbft/cometbft@v0.38.12/state/execution.go:166

**Summary/Description**  
During ProcessProposal, proposalServer.ExecutionPayload pushes the proposed execution payload with engine_newPayloadV3. If the execution client returns SYNCING or ACCEPTED, the handler logs the status and treats the push as complete. It then immediately reads previous-payload EVM events through evmEvents(payload.ParentHash). In geth, NewPayload with a missing parent or unavailable state only stashes the block and explicitly relies on a later ForkchoiceUpdated call to force sync to that head; ProcessProposal does not make that ForkchoiceUpdated call. If the local execution client also cannot serve logs for the parent hash, eth_getLogs returns a deterministic unknown-block style error and evmEvents retries forever.

**Root Cause**  
The ProcessProposal NewPayload status machine treats SYNCING/ACCEPTED as forward progress but does not perform the forkchoice update that geth requires to convert a stashed payload into a sync target. The next step reads block-hash-scoped logs from the same execution client under retryForever, and deterministic unavailable-block errors are classified as transient network failures.

**Pre_conditions**  
A validator processes an otherwise valid MsgExecutionPayload while its local execution client is behind, still syncing, missing the proposed payload parent, or missing state/log data for the parent hash. This can occur during startup/catch-up or an execution-client sync race; no malformed payload is required.

**Impact**  
The validator can block inside ProcessProposal because CometBFT v0.38.12 calls the ABCI method with context.TODO(). A single affected validator loses liveness until restarted or manually resynced; if the same execution-client lag affects enough voting power, consensus can halt at the height. The finalized handler has the missing sync trigger: after SYNCING/ACCEPTED NewPayload it repeatedly calls ForkchoiceUpdatedV3 on the payload hash until VALID, so the gap is specific to proposal verification.

**Proof of Concept**  
PoC not run per instruction. Code path: proposalServer.ExecutionPayload returns true on isSyncing(status), then calls evmEvents(payload.ParentHash). evmEvents retries forever on every event processor Prepare error. The in-scope event processors use eth_getLogs with FilterQuery.BlockHash. Geth returns SYNCING from NewPayload when the parent/state is unavailable, stores the remote block for a later forkchoice update, and notes that NewPayload will not start a sync cycle by itself. Because ProcessProposal never calls ForkchoiceUpdatedV3, a local unknown parent/log block remains an unbounded retry condition.

**Mitigation**  
In ProcessProposal, do not treat SYNCING or ACCEPTED NewPayload as successful validation. Either return REJECT/temporary failure without entering event-log verification, or issue the same forkchoice-sync path used in FinalizeBlock and bound the wait by a proposal-safe context. Classify deterministic eth_getLogs unknown-block errors separately from transient network errors so proposal verification can return instead of retrying forever.

### [I-01] ProcessProposal accepts unsupported blob hash payload fields as ignored metadata

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: omni/octane/evmengine/keeper/keeper.go:113; omni/octane/evmengine/keeper/proposal_server.go:21; omni/octane/evmengine/keeper/msg_server.go:32; omni/octane/evmengine/types/tx.proto:25

**Summary/Description**  
MsgExecutionPayload stores execution_payload as raw JSON bytes, but parseAndVerifyProposedPayload decodes it with json.Unmarshal into geth engine.ExecutableData. Unknown JSON keys are ignored by that decoder. As a result, a proposed payload can include unsupported top-level fields such as blobHashes or versionedHashes and still pass ProcessProposal and FinalizeBlock when the canonical ExecutableData fields are otherwise valid and there are no actual blob transactions.

**Root Cause**  
The proposal flow does not use a strict JSON decoder and does not canonicalize or compare the execution_payload bytes against the supported Engine API payload schema. Extra unsupported fields remain in the consensus message bytes but are dropped before pushPayload, which always supplies its own empty versionedHashes argument to NewPayloadV3.

**Pre_conditions**  
A proposer includes additional top-level blobHashes or versionedHashes fields in the MsgExecutionPayload execution_payload JSON while keeping the supported ExecutableData fields valid. The payload contains no blob transaction hashes, or geth rejects it through the separate versioned-hash mismatch path.

**Impact**  
Malformed payload metadata can be accepted into and finalized by the proposal flow even though Omni does not support those fields. The ignored fields do not change the block sent to the execution client, so this is primarily a validation/canonicalization gap and forward-compatibility risk rather than direct state corruption.

**Proof of Concept**  
PoC not run per instruction. Code reasoning: json.Unmarshal into engine.ExecutableData ignores unknown fields; parseAndVerifyProposedPayload returns the decoded struct; proposalServer and msgServer never compare the original JSON bytes to a strict/canonical encoding; pushPayload calls NewPayloadV3 with emptyVersionHashes regardless of any top-level fields in the original JSON.

**Mitigation**  
Reject unknown execution_payload JSON fields or store the payload in a typed/canonical format that cannot carry unsupported metadata. Explicitly reject top-level blobHashes/versionedHashes until blob payload support is implemented.

### [I-02] FinalizeBlock accepts aggregate-vote shapes that ProcessProposal rejects

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni/halo/attest/keeper/proposal_server.go:31; omni/halo/attest/keeper/keeper.go:891; omni/halo/attest/keeper/msg_server.go:32; omni/halo/attest/keeper/keeper.go:170; omni/halo/attest/keeper/helper.go:45; omni/halo/attest/types/translate.go:37

**Summary/Description**  
ProcessProposal routes MsgAddVotes through verifyAggVotes, which applies aggregate-level structural checks: supported source/conf level, consensus-chain ID, duplicate attestation roots, per-validator vote count limits, validator membership, and vote-window bounds. The FinalizeBlock MsgAddVotes handler only verifies each aggregate signature set individually and then calls Keeper.Add. Keeper.Add checks previous validator-set membership but does not re-run the proposal structural checks, and Keeper.addOne merges duplicate attestation roots into existing storage. Therefore a committed MsgAddVotes that ProcessProposal would reject, such as duplicate same-root aggregates with disjoint validators, a wrong consensus-chain ID, unsupported chain/conf level, or out-of-window offsets, can still be stored and potentially approved if it reaches FinalizeBlock.

**Root Cause**  
The finalized path intentionally treats committed votes as too late to reject and does not share proposal-time normalization or structural validation. The persisted attestation row also stores source chain, conf level, offset, block data, msg root, and attestation root, but not the consensus-chain ID from the signed AttestHeader, while query conversion reconstructs the local consensus-chain ID.

**Pre_conditions**  
A block containing a proposal-rejected MsgAddVotes reaches FinalizeBlock, for example because the block was committed despite local ProcessProposal rejection, a future/replay path bypasses ProcessProposal, or the BFT assumption is already violated. Honest PrepareVotes normally groups same-root votes, filters unsupported or out-of-window votes, and honest validators reject these shapes in ProcessProposal, so this is not reachable through the normal honest proposal path.

**Impact**  
Finalized attestation state can diverge from proposal acceptance rules. Duplicate same-root aggregates can be merged and approved from an aggregate layout ProcessProposal rejects. Wrong-consensus-ID roots are verified and stored using the signed root, but later query conversion reconstructs the protobuf attestation with the local consensus-chain ID, so downstream conversion can expose an attestation header that does not match the stored signed root. This is primarily an invariant and unsafe committed-block handling gap under normal BFT assumptions; it still matters for replay/catch-up safety and future callers that may rely on FinalizeBlock to enforce the same validity envelope as ProcessProposal.

**Proof of Concept**  
Code review only. verifyAggVotes rejects duplicate roots, invalid header chains, count-limit overflow, and out-of-window aggregates. msgServer.AddVotes in finalize mode loops over AggVote.Verify only, then Keeper.addOne fetches or inserts by attestation_root and appends new validator signatures. AttestationFromDB later fills ConsensusChainId from the local context rather than the signed aggregate header.

**Mitigation**  
Share a canonical aggregate normalization and validation routine between ProcessProposal and FinalizeBlock. If committed blocks cannot be rejected, FinalizeBlock should still deterministically drop or quarantine aggregates that fail supported-chain, consensus-chain, window, duplicate-root, and count invariants before storage, and it should persist or validate the consensus-chain ID used in the signed root. Add invariant tests that finalized accepted state cannot be derived from MsgAddVotes shapes rejected by ProcessProposal.

### [I-03] FinalizeBlock trusts proposal-time previous-payload event lists

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni/octane/evmengine/keeper/proposal_server.go:49; omni/octane/evmengine/keeper/proposal_server.go:56; omni/octane/evmengine/keeper/msg_server.go:99; omni/octane/evmengine/keeper/msg_server.go:113; omni/octane/evmengine/keeper/evmmsgs.go:17; omni/halo/evmstaking/evmstaking.go:111

**Summary/Description**  
ProcessProposal recomputes the local previous-payload EVM event set and compares it with MsgExecutionPayload.PrevPayloadEvents before honest validators prevote. The FinalizeBlock MsgExecutionPayload handler does not recompute or compare that set; it pushes the payload, updates forkchoice, and delivers only the event list supplied in the committed message. Therefore a committed MsgExecutionPayload whose PrevPayloadEvents would be rejected in ProcessProposal can still drive event delivery if it reaches FinalizeBlock through replay, future callers, or a block committed despite local proposal rejection.

**Root Cause**  
Proposal-time event agreement and finalized event delivery use different validity envelopes. The equality check is implemented only in proposalServer.ExecutionPayload, while msgServer.ExecutionPayload trusts the committed event list and deliverEvents treats it as the source of truth.

**Pre_conditions**  
A block containing a proposal-rejected MsgExecutionPayload reaches FinalizeBlock, for example because the block is applied from a path that does not run local ProcessProposal, a future/replay integration calls the finalize message path directly, or the BFT assumption is already violated. Under the normal honest proposal path, a proposer cannot omit, duplicate, or substitute previous-payload events because honest validators recompute the local list and reject mismatches before prevoting.

**Impact**  
The finalized path can diverge from proposal acceptance rules. If this branch is reached, omitted staking events are never delivered, duplicated Delegate events can be delivered more than once, and non-local event data can be parsed by processors even though it was not returned by the local EVM log query. In normal operation this is guarded by ProcessProposal and is therefore an invariant/replay-safety gap rather than a directly proposer-reachable exploit. Normal-path staking delivery failures from official events are covered separately by M-01 and L-01.

**Proof of Concept**  
Code review only. proposalServer.ExecutionPayload calls evmEvents(payload.ParentHash) and evmEventsEqual against msg.PrevPayloadEvents. msgServer.ExecutionPayload later calls deliverEvents with msg.PrevPayloadEvents directly and never re-runs evmEvents or evmEventsEqual. deliverEvents verifies protobuf shape and routes by address, but it does not prove that each event came from the parent payload logs.

**Mitigation**  
Share a canonical previous-payload event validation routine between ProcessProposal and FinalizeBlock. If committed blocks cannot be rejected, FinalizeBlock should at least recompute the local event set, deterministically drop or quarantine proposal-rejected event lists, and expose failed or skipped event results instead of silently treating the supplied list as authoritative.

