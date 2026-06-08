### [M-01] OptionTokenV4 undercharges exercises by using sell-side TWAP output as purchase price

**Severity**: Medium  
**Location**: v4-contracts/contracts/OptionTokenV4.sol:320; v4-contracts/contracts/OptionTokenV4.sol:323; v4-contracts/contracts/OptionTokenV4.sol:331; v4-contracts/contracts/OptionTokenV4.sol:335; v4-contracts/contracts/OptionTokenV4.sol:372; v4-contracts/contracts/OptionTokenV4.sol:375; v4-contracts/contracts/Pair.sol:222; v4-contracts/contracts/Pair.sol:226; v4-contracts/contracts/Pair.sol:237; v4-contracts/contracts/Pair.sol:239; v4-contracts/contracts/Pair.sol:388; v4-contracts/contracts/Pair.sol:397

**Summary/Description**  
OptionTokenV4 prices option exercise premiums by calling pair.prices(underlyingToken, amount, twapPoints). Pair.prices/sample then computes _getAmountOut(amountIn, tokenIn, averaged reserves), which for volatile pairs returns amountIn * reserveB / (reserveA + amountIn): the payment-token output for selling the underlying amount into the pool. The option comments describe purchasing the underlying token with payment tokens, so the implementation uses the sell-side output as the purchase quote; this undercharges relative to the inverse amount of payment tokens needed to acquire the same underlying amount, especially for larger amounts.

**Root Cause**  
The pricing helper treats Pair.prices(underlyingToken, amount) as a payment-token purchase cost instead of an underlying-token sell output, and never computes the inverse quote for buying amount underlying with payment token.

**Pre_conditions**  
The configured pair is the underlying/payment pair used for option pricing; an option holder exercises regular, ve, or LP paths for an amount where AMM curvature matters.

**Impact**  
Exercise paths can collect less payment token than the intended discounted purchase price, reducing treasury/gauge reward proceeds and letting option holders acquire underlying exposure or LP/ve exposure too cheaply.

**Mitigation**  
Compute the payment-token input required to receive the requested underlying amount from the TWAP reserves, then apply the discount to that purchase-side quote. Keep the payment direction explicit in function names and tests.

### [M-02] LP exercise paths do not cap spot-reserve LP payment

**Severity**: Medium  
**Location**: v4-contracts/contracts/OptionTokenV4.sol:297; v4-contracts/contracts/OptionTokenV4.sol:300; v4-contracts/contracts/OptionTokenV4.sol:350; v4-contracts/contracts/OptionTokenV4.sol:353; v4-contracts/contracts/OptionTokenV4.sol:354; v4-contracts/contracts/OptionTokenV4.sol:355; v4-contracts/contracts/OptionTokenV4.sol:607; v4-contracts/contracts/OptionTokenV4.sol:608; v4-contracts/contracts/OptionTokenV4.sol:616; v4-contracts/contracts/OptionTokenV4.sol:620; v4-contracts/contracts/OptionTokenV4.sol:626; v4-contracts/contracts/OptionTokenV4.sol:631; v4-contracts/contracts/OptionTokenV4.sol:666; v4-contracts/contracts/OptionTokenV4.sol:667; v4-contracts/contracts/OptionTokenV4.sol:675; v4-contracts/contracts/OptionTokenV4.sol:679; v4-contracts/contracts/OptionTokenV4.sol:685; v4-contracts/contracts/OptionTokenV4.sol:690; v4-contracts/contracts/Router.sol:65; v4-contracts/contracts/Router.sol:182; v4-contracts/contracts/Router.sol:186; v4-contracts/contracts/Router.sol:199; v4-contracts/contracts/Router.sol:210

**Summary/Description**  
exerciseVe and exerciseLp compute two payment-token amounts: paymentAmount, the discounted premium checked against _maxPaymentAmount, and paymentAmountToAddLiquidity, the spot-reserve amount required to pair with the underlying tokens. Only paymentAmount is capped by _maxPaymentAmount. The actual user transfer includes the remaining premium after treasury fees plus paymentAmountToAddLiquidity, and the router receives amountBMin=1, so the spot LP side has no user-controlled maximum or meaningful slippage bound.

**Root Cause**  
The slippage parameter is applied only to the discounted premium, while the larger spot-reserve liquidity payment is computed internally from mutable Pair reserves and then transferred from the user without a max-total-payment check.

**Pre_conditions**  
An option holder calls exerciseVe or exerciseLp with sufficient payment-token allowance/balance; current volatile pair reserves make paymentAmountToAddLiquidity larger than the user would accept, whether naturally or via reserve manipulation before the transaction executes.

**Impact**  
The caller can pay far more payment token than the documented maximum acceptable payment, because the liquidity-side payment is outside the cap. This affects both VE and gauge LP exercise lifecycles and can route the excess into LP locked for the recipient.

**Mitigation**  
Expose and enforce a maximum total payment or a separate maxPaymentAmountToAddLiquidity/maxLPAmount check. Use router quoteAddLiquidity or explicit min-lp/max-token bounds derived by the caller, and avoid hardcoded amountAMin/amountBMin=1 for user-facing exercise paths.

### [M-03] OptionTokenV4 public gauge update can make LP exercise unavailable

**Severity**: Medium  
**Location**: See preception/bank-results references for the original prompt-bank evidence.

**Summary/Description**  
Reconstructed inventory entry for M-03. The original detailed report body was not recoverable after the report helper overwrote the untracked report file; source evidence remains in preception/bank-results and bank-progress metadata.

### [M-04] OptionTokenV4 TWAP can be stale or shaped by limited observations

**Severity**: Medium  
**Location**: See preception/bank-results references for the original prompt-bank evidence.

**Summary/Description**  
Reconstructed inventory entry for M-04. The original detailed report body was not recoverable after the report helper overwrote the untracked report file; source evidence remains in preception/bank-results and bank-progress metadata.

### [M-05] OptionTokenV4 can strand final gauge reward balances below Gauge.left

**Severity**: Medium  
**Location**: v4-contracts/contracts/OptionTokenV4.sol:734; v4-contracts/contracts/OptionTokenV4.sol:739; v4-contracts/contracts/OptionTokenV4.sol:740; v4-contracts/contracts/OptionTokenV4.sol:745; v4-contracts/contracts/OptionTokenV4.sol:747; v4-contracts/contracts/GaugeV4.sol:557

**Summary/Description**  
When rewardsAddress is unset, OptionTokenV4 forwards collected payment-token rewards to the gauge only if the contract's full paymentToken balance is greater than IGaugeV4(gauge).left(paymentToken). If the final collected balance is positive but less than or equal to the gauge's remaining active rewards, _transferRewardToGauge leaves the tokens in OptionTokenV4. There is no public/admin sweep or retry function other than causing a later exercise to call the internal transfer helper again.

**Root Cause**  
Reward delivery is gated by a strict balance > leftRewards condition, but the contract has no standalone reward flush path after leftRewards decays.

**Pre_conditions**  
rewardsAddress is address(0); gauge.left(paymentToken) is greater than or equal to the OptionTokenV4 payment-token balance when an exercise calls _transferRewardToGauge; no later exercise accumulates enough balance to pass the threshold.

**Impact**  
Users' payment-token remainder intended for gauge rewards can remain in OptionTokenV4 indefinitely, so the expected reward stream is not delivered even though exercisers paid it. The issue affects the last or below-threshold accumulation; larger later exercises can incidentally flush the balance.

**Mitigation**  
Add a public or admin flushRewards() that re-runs reward routing, and consider notifying any positive collected balance when the current reward period has ended or using a carry-forward queue rather than a strict leftRewards threshold.

### [M-06] Zero regular discount inherits zero-premium exercise behavior

**Severity**: Medium  
**Location**: See preception/bank-results references for the original prompt-bank evidence.

**Summary/Description**  
Reconstructed inventory entry for M-06. The original detailed report body was not recoverable after the report helper overwrote the untracked report file; source evidence remains in preception/bank-results and bank-progress metadata.

### [M-07] Direct gauge rewards can suppress active gauge counts

**Severity**: Medium  
**Location**: v4-contracts/contracts/Voter.sol:553; v4-contracts/contracts/Voter.sol:555; v4-contracts/contracts/Voter.sol:556; v4-contracts/contracts/Minter.sol:98; v4-contracts/contracts/Minter.sol:117; v4-contracts/contracts/Gauge.sol:510; v4-contracts/contracts/Gauge.sol:516; v4-contracts/contracts/GaugeV4.sol:557; v4-contracts/contracts/GaugeV4.sol:563

**Summary/Description**  
Voter.distribute only counts a gauge as active after the gauge's claimable base reward exceeds IGauge.left(base), then Minter.weekly_emission uses the resulting activeGaugeNumber for the next weekly mint. Gauge and GaugeV4 expose public base-token reward notification paths, so a caller can fund an active gauge's base reward stream before distribution and raise left(base). That causes the gauge to skip Voter notification and skip activeGaugeNumber increment even though it has vote weight and claimable emissions.

**Root Cause**  
The active-gauge counter is derived from the delivery branch guarded by _claimable > IGauge.left(base), rather than from vote weight or the gauge's current emission entitlement. Public gauge reward notification can independently raise left(base).

**Pre_conditions**  
1. A gauge has positive Voter claimable emissions and would otherwise pass the active-gauge share threshold. 2. The base token is accepted as a gauge reward, either from constructor-seeded allowed rewards or the whitelist path. 3. A caller funds the gauge's base reward stream before Voter.distribute processes the gauge.

**Impact**  
The affected weekly lifecycle can undercount active gauges. Because Minter.weekly_emission multiplies weeklyPerGauge by activeGaugeNumber, the next epoch can mint less FLOW for voter-directed gauge emissions than the code's own active-gauge model intends.

**Mitigation**  
Do not couple active-gauge counting to Gauge.left(base). Count active gauges from current Voter weight/share before the left() delivery guard, or track the counter separately from whether a specific notifyRewardAmount call should be made.

### [M-08] Expired veNFT votes can keep directing emissions and earning bribes

**Severity**: Medium  
**Location**: v4-contracts/contracts/Voter.sol:234; v4-contracts/contracts/Voter.sol:249; v4-contracts/contracts/Voter.sol:267; v4-contracts/contracts/Voter.sol:282; v4-contracts/contracts/Voter.sol:485; v4-contracts/contracts/VotingEscrow.sol:1017; v4-contracts/contracts/VotingEscrow.sol:1031; v4-contracts/contracts/VotingEscrow.sol:1175; v4-contracts/contracts/ExternalBribe.sol:212; v4-contracts/contracts/ExternalBribe.sol:250; v4-contracts/contracts/Minter.sol:112

**Summary/Description**  
Voter snapshots balanceOfNFT into persistent pool weights, totalWeight, votes, and ExternalBribe balances when voting. VotingEscrow balance later decays to zero at lock expiry, but the Voter accounting does not decay or clear automatically. Poke is only owner/governor callable and goes through _vote; once balanceOfNFT is zero or rounds a pool to zero, _vote reverts on _poolWeight != 0, so governor cannot clear the stale expired vote with poke. Only an approved owner reset clears it, which lets an expired or inactive veNFT keep directing emissions and bribe accounting until the owner chooses to reset.

**Root Cause**  
Persistent vote/bribe accounting is not reconciled against current veNFT voting power at epoch transition or expiry, and poke reuses the nonzero new-vote path instead of supporting zero-balance cleanup.

**Impact**  
Stale vote weights remain in weights and totalWeight, affecting Voter.notifyRewardAmount distribution ratios, and stale ExternalBribe balances can keep earning epoch bribes even after the underlying veNFT has no current voting power.

**Mitigation**  
Add a permissionless or keeper-safe expiry reconciliation path, or make poke handle zero/rounded-zero current voting power by withdrawing prior votes and abstaining instead of reverting. Alternatively compute distribution/bribe eligibility from epoch snapshots that explicitly exclude expired voting power.

### [M-09] Third-party GaugeV4 locked deposits can extend a recipient's existing lock

**Severity**: Medium  
**Location**: v4-contracts/contracts/GaugeV4.sol:443; v4-contracts/contracts/GaugeV4.sol:452; v4-contracts/contracts/GaugeV4.sol:453; v4-contracts/contracts/GaugeV4.sol:455; v4-contracts/contracts/GaugeV4.sol:458; v4-contracts/contracts/GaugeV4.sol:516; v4-contracts/contracts/GaugeV4.sol:520; v4-contracts/contracts/GaugeV4.sol:522; v4-contracts/contracts/OptionTokenV4.sol:305; v4-contracts/contracts/OptionTokenV4.sol:700

**Summary/Description**  
GaugeV4 tracks each account's locked stake as one aggregate balanceWithLock and one lockEnd. depositWithLock is callable by whitelisted option-token contracts for arbitrary recipients, and it extends lockEnd for the aggregate locked balance when the new lock end is later than the current one. A caller can exercise a small amount of LP to a recipient through OptionTokenV4 and extend the recipient's previously locked GaugeV4 stake.

**Root Cause**  
GaugeV4 does not track locked deposits as independent tranches or restrict third-party locked deposits from changing an existing recipient lockEnd; OptionTokenV4 exposes an arbitrary _recipient LP exercise path into depositWithLock.

**Pre_conditions**  
1. Recipient has an active GaugeV4 balanceWithLock with lockEnd earlier than the attacker's chosen new lock end. 2. A whitelisted option-token path such as OptionTokenV4 can call depositWithLock for that recipient.

**Impact**  
The recipient cannot withdraw the previously locked portion at its original expiry; withdrawal that touches locked balance is blocked until the extended lockEnd. With default OptionTokenV4 parameters, the later lock can be 52 weeks.

**Mitigation**  
Do not let third-party locked deposits extend an existing recipient lock. Track lock tranches separately, require account == msg.sender when extending an existing lock, or only apply the new lock end to the newly deposited amount.

### [M-10] Disabling the last max-locked veNFT leaves a stale max-lock index

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Medium  
**Location**: v4-contracts/contracts/VotingEscrow.sol:893; v4-contracts/contracts/VotingEscrow.sol:897; v4-contracts/contracts/VotingEscrow.sol:901; v4-contracts/contracts/VotingEscrow.sol:904; v4-contracts/contracts/VotingEscrow.sol:907; v4-contracts/contracts/VotingEscrow.sol:912; v4-contracts/contracts/VotingEscrow.sol:917; v4-contracts/contracts/VotingEscrow.sol:918; v4-contracts/contracts/VotingEscrow.sol:955; v4-contracts/contracts/VotingEscrow.sol:956; v4-contracts/contracts/VotingEscrow.sol:960

**Summary/Description**  
VotingEscrow.disable_max_lock uses swap-and-pop removal, but when the removed token is already the last array element it sets maxLockIdToIndex[tokenId] to zero and then immediately writes it back to index + 1 before popping. The NFT is removed from max_locked_nfts but remains marked as max-locked, so later authorization checks still call max_lock and can prevent withdrawal after expiry.

**Root Cause**  
In VotingEscrow.disable_max_lock, the removed token mapping is cleared before max_locked_nfts[index] is assigned from the last slot, and maxLockIdToIndex[max_locked_nfts[index]] is always rewritten even when index is the last slot and the moved token is the same token being removed. The impact is amplified because _isApprovedOrOwner calls max_lock before evaluating permissions, so withdraw and disable paths can refresh or revert a stale-flagged token before the owner can exit.

**Pre_conditions**  
1. A user enables max lock for a veNFT. 2. The user later calls disable_max_lock while that veNFT is the last element in max_locked_nfts, including the common single-element case. 3. max_lock_enabled remains true when the user later performs an authorization-gated action or tries to withdraw after expiry.

**Impact**  
The user's disable action appears to succeed, but maxLockIdToIndex[tokenId] remains nonzero while the token is no longer present in max_locked_nfts. Later _isApprovedOrOwner calls still execute max_lock for that token. If the lock is still active, it can be extended despite being disabled by the user. If the lock has expired, max_lock reverts before withdraw can reach the expiry check, leaving the locked LP inaccessible until the team disables max_lock_enabled globally.

**Proof of Concept**  
Not run; code-level review only. For a one-element max_locked_nfts array [tokenId], disable_max_lock computes index = 0, clears maxLockIdToIndex[tokenId], assigns max_locked_nfts[0] from the last slot which is still tokenId, rewrites maxLockIdToIndex[tokenId] = 1, and then pops the array to empty.

**Mitigation**  
In disable_max_lock, only rewrite the moved token index when index != max_locked_nfts.length - 1. Also separate approval checks from max_lock refreshes, or use a non-refreshing permission check for disable_max_lock and withdraw.

### [M-11] VotingEscrow transfers to the zero address strand locked LP

**Severity**: Medium  
**Location**: v4-contracts/contracts/VotingEscrow.sol

**Summary/Description**  
Reconstructed inventory entry for M-11. Bank notes identify transfer-to-zero behavior in VotingEscrow as stranding locked LP; source evidence remains in preception/bank-results.

### [M-12] VotingEscrow getPastVotes is not historical

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: v4-contracts/contracts/VotingEscrow.sol:43; v4-contracts/contracts/VotingEscrow.sol:1268; v4-contracts/contracts/VotingEscrow.sol:1306; v4-contracts/contracts/VotingEscrow.sol:1337; v4-contracts/contracts/VotingEscrow.sol:1362; v4-contracts/contracts/VotingEscrow.sol:1431

**Summary/Description**  
VotingEscrow.getPastVotes is not historical. Delegate checkpoints are defined with a timestamp field, but the delegate movement writers only copy tokenIds and increment numCheckpoints; they never set checkpoints[account][i].timestamp. getPastVotesIndex therefore sees zero timestamps and returns the latest checkpoint for any normal timestamp. getPastVotes then sums that latest tokenId list using _balanceOfNFT, which itself reads the latest user_point_epoch rather than the token point active at the queried time.

**Root Cause**  
The delegation checkpoint writers _moveTokenDelegates and _moveAllDelegates do not write Checkpoint.timestamp, while getPastVotesIndex relies on that timestamp for binary search. The vote summation also uses _balanceOfNFT(timestamp), which starts from the latest user_point_history entry instead of selecting the user point at or before the requested timestamp.

**Pre_conditions**  
1. An account has at least one delegate checkpoint from mint, burn, transfer, or delegate movement. 2. A caller queries getPastVotes for a timestamp before a later delegate-token or locked-balance change.

**Impact**  
Historical vote reads can return balances based on the latest delegated token list and latest per-token lock point rather than the queried snapshot. Past votes can be overstated after later deposits/extensions/merges, understated after later withdrawals/burns, or assigned to the wrong delegate after later transfers/redelegations.

**Mitigation**  
Write block.timestamp into every delegate checkpoint when tokenIds are written, handle same-timestamp overwrites without incorrectly increasing numCheckpoints, and compute per-token voting power from the user_point_history entry active at the requested snapshot.

### [M-13] Zero-supply reward periods can strand Gauge emissions before the first supply checkpoint

**Severity**: Medium  
**Location**: v4-contracts/contracts/OptionTokenV4.sol:626; v4-contracts/contracts/OptionTokenV4.sol:640; v4-contracts/contracts/OptionTokenV4.sol:647; v4-contracts/contracts/OptionTokenV4.sol:739; v4-contracts/contracts/OptionTokenV4.sol:747; v4-contracts/contracts/OptionTokenV4.sol:749; v4-contracts/contracts/GaugeV4.sol:305; v4-contracts/contracts/GaugeV4.sol:306; v4-contracts/contracts/GaugeV4.sol:368; v4-contracts/contracts/GaugeV4.sol:372; v4-contracts/contracts/GaugeV4.sol:397; v4-contracts/contracts/GaugeV4.sol:399; v4-contracts/contracts/GaugeV4.sol:563; v4-contracts/contracts/GaugeV4.sol:573; v4-contracts/contracts/GaugeV4.sol:592

**Summary/Description**  
GaugeV4 can start a reward stream while it has no positive staking supply. In the option lifecycle this is directly reachable from exerciseVe: the option contract mints LP, locks that LP into VotingEscrow via create_lock_for, and still calls _transferRewardToGauge, which can notify the payment token to GaugeV4 even though no LP was deposited into GaugeV4 by that flow. GaugeV4 rewardPerToken returns the stored value when derivedSupply is zero, and _updateRewardPerToken returns unchanged when no supply checkpoints exist or when the current checkpoint supply is zero, so elapsed rewards before positive gauge supply are not assigned to any account.

**Root Cause**  
Reward notification is independent of positive GaugeV4 staking supply, while reward accrual is driven by derivedSupply and supply checkpoints. The exerciseVe path sends payment rewards to the gauge but locks the LP in VotingEscrow instead of creating gauge supply.

**Pre_conditions**  
A payment reward is notified to GaugeV4 while derivedSupply is zero or before any positive supply checkpoint; this can happen through OptionTokenV4 exerciseVe when no account has staked in the gauge.

**Impact**  
Payment-token rewards funded by option exercisers can remain in the gauge without becoming claimable for any user during the zero-supply interval, reducing delivered gauge rewards.

**Mitigation**  
Defer reward notification when derivedSupply is zero, hold pending rewards until positive supply exists, or route exerciseVe payment rewards to a component that actually has eligible recipients. Alternatively require a positive gauge supply before OptionTokenV4 notifies the gauge.

### [M-14] Expired max-lock veNFTs cannot pass owner checks

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Max-lock veNFT owners can be blocked from claim/reset/withdraw after an unrefreshed max-lock expires, until team disables max_lock_enabled globally  
**Location**: v4-contracts/contracts/VotingEscrow.sol:295, v4-contracts/contracts/VotingEscrow.sol:296, v4-contracts/contracts/VotingEscrow.sol:305, v4-contracts/contracts/VotingEscrow.sol:912, v4-contracts/contracts/VotingEscrow.sol:917, v4-contracts/contracts/VotingEscrow.sol:918, v4-contracts/contracts/VotingEscrow.sol:955, v4-contracts/contracts/VotingEscrow.sol:956, v4-contracts/contracts/Voter.sol:201, v4-contracts/contracts/Voter.sol:287, v4-contracts/contracts/Voter.sol:542, v4-contracts/contracts/RewardsDistributorV2.sol:282, v4-contracts/contracts/RewardsDistributorV2.sol:283

**Summary/Description**  
VotingEscrow uses _isApprovedOrOwner as a shared authorization helper, but that helper first calls max_lock. For tokenIds still marked as max-locked, max_lock reverts once the current locked.end is already expired. As a result, normal owner-authorized flows that should handle expiry, including Voter reset, Voter vote/claimBribes, RewardsDistributorV2 claim, and VotingEscrow withdraw, can revert before reaching their own logic.

**Root Cause**  
The authorization helper mutates/refreshes lock state before checking ownership or approval. max_lock requires _locked.end > block.timestamp when it wants to extend an enabled max-lock token, so an already expired max-lock token cannot pass the helper used by exit and claim paths.

**Pre_conditions**  
1. A user enables max_lock for a veNFT. 2. The veNFT is not refreshed before locked.end. 3. max_lock_enabled remains true when the owner tries to claim, reset, vote, or withdraw.

**Impact**  
The affected token cannot pass isApprovedOrOwner while the expired max-lock marker remains active. This blocks owner-authorized claim/reset paths and also blocks withdraw before the explicit expiry withdrawal logic can run. User funds and rewards are inaccessible unless the team toggles max_lock_enabled globally or another privileged/system change removes the condition.

**Proof of Concept**  
Not run; code-level review only. _isApprovedOrOwner calls max_lock before reading owner/approval. max_lock enters for maxLockIdToIndex[tokenId] != 0 and max_lock_enabled, computes a later unlock_time, and when that unlock_time is greater than the expired locked.end it requires locked.end > block.timestamp.

**Mitigation**  
Separate authorization checks from max-lock refreshes. Use a non-mutating owner/approval helper for exit, claim, reset, and disable paths, or make max_lock no-op rather than revert for already expired locks so owners can withdraw or disable.

### [M-15] RewardsDistributorV2 wraps expired veNFT power into huge claim balances

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Expired tokenId claim accounting can revert or request impossible reward amounts, leaving accrued distributor rewards stuck for affected veNFTs once later reward weeks exist.  
**Location**: v4-contracts/contracts/RewardsDistributorV2.sol:139; v4-contracts/contracts/RewardsDistributorV2.sol:158; v4-contracts/contracts/RewardsDistributorV2.sol:208; v4-contracts/contracts/RewardsDistributorV2.sol:265; v4-contracts/contracts/VotingEscrow.sol:1017-1027; v4-contracts/contracts/VotingEscrow.sol:1125-1147

**Summary/Description**  
RewardsDistributorV2 attempts to clamp decayed signed voting power to zero with Math.max, but it first casts the signed result to uint. When a tokenId point is evaluated after its lock expiry, the negative signed balance becomes a huge unsigned value instead of zero. Claim and claimable then treat an expired veNFT as having enormous balance for post-expiry reward weeks, causing arithmetic overflow or an impossible transfer amount rather than cleanly stopping at zero.

**Root Cause**  
The distributor uses Math.max(uint(int256(point.bias - point.slope * dt)), 0) in ve_for_at, _checkpoint_total_supply, _claim, and _claimable. The non-negative clamp is applied after the signed-to-unsigned conversion, unlike VotingEscrow's own balance calculation which checks for negative bias before casting.

**Pre_conditions**  
1. A veNFT has a user point with positive slope and later reaches lock expiry without a new zero user checkpoint. 2. Rewards are assigned to one or more weeks after that expiry. 3. The tokenId owner calls claim or claimable across those weeks.

**Impact**  
Affected expired tokenIds can be unable to claim otherwise accrued distributor rewards after the loop reaches a post-expiry week with rewards. The wrapped balance generally overflows during balance_of * tokens_per_week or computes an amount far above the distributor balance, so claim reverts and the tokenId's remaining rewards stay in the distributor.

**Proof of Concept**  
Not run; code-level review only. For any user point where dt is greater than bias / slope, old_user_point.bias - dt * old_user_point.slope is negative. RewardsDistributorV2 casts that negative int256 to uint before Math.max, so the zero-balance break condition is not reached.

**Mitigation**  
Clamp in signed space before casting, e.g. compute int256 decayed = int256(point.bias) - int256(point.slope) * int256(dt); return decayed > 0 ? uint256(decayed) : 0. Apply the same helper to ve_for_at, _checkpoint_total_supply, _claim, and _claimable.

### [I-01] OptionTokenV4 exercise path accounting caveat

**Severity**: Info  
**Location**: See preception/bank-results references for the original prompt-bank evidence.

**Summary/Description**  
Reconstructed inventory entry for I-01. The original detailed report body was not recoverable after the report helper overwrote the untracked report file; source evidence remains in preception/bank-results and bank-progress metadata.

### [I-02] OptionTokenV4 pair binding does not bind router stable mode

**Severity**: Info  
**Location**: v4-contracts/contracts/OptionTokenV4.sol:396; v4-contracts/contracts/OptionTokenV4.sol:400; v4-contracts/contracts/OptionTokenV4.sol:354; v4-contracts/contracts/OptionTokenV4.sol:626; v4-contracts/contracts/OptionTokenV4.sol:685; v4-contracts/contracts/Router.sol:47; v4-contracts/contracts/Router.sol:65; v4-contracts/contracts/Router.sol:199

**Summary/Description**  
setPairAndPaymentToken verifies only that the configured pair's token tuple contains underlyingToken and paymentToken. It does not store or validate the pair's stable flag. Later LP and ve exercise paths read reserves and add liquidity through Router with stable=false, then try to approve/lock/stake the configured pair token. If the configured pair is stable, liquidity is minted in the volatile pair while VotingEscrow/GaugeV4 expect the configured stable pair token, causing those exercise paths to revert atomically.

**Root Cause**  
The pair/payment binding omits the stable/volatile pool dimension even though Router addresses are keyed by tokenA, tokenB, and stable.

**Pre_conditions**  
Admin configures OptionTokenV4 with a stable payment pair, or another token-compatible pair whose identity differs from router.pairFor(underlyingToken, paymentToken, false).

**Impact**  
Regular exercise still prices against the configured pair, but exerciseVe and exerciseLp become unavailable because the LP token minted by Router does not match the LP token accepted by VotingEscrow or the configured gauge. Atomic reverts prevent partial asset loss.

**Mitigation**  
Store the configured pair's stable flag or derive it from pair.stable(), then use the same flag for getReserves and addLiquidity. Also validate router.pairFor(underlyingToken, paymentToken, stable) == address(pair).

### [I-03] Voter factory array lifecycle exposes stale entries

**Severity**: Info  
**Location**: v4-contracts/contracts/Voter.sol:155; v4-contracts/contracts/Voter.sol:168; v4-contracts/contracts/Voter.sol:187; v4-contracts/contracts/Voter.sol:323

**Summary/Description**  
Factory replacement requires the replacement pair factory and gauge factory to already be marked active, so adding then replacing can duplicate the same pair/gauge factory across multiple array slots. Removing one duplicated slot sets isFactory/isGaugeFactory false for the address while another nonzero array slot can still be selected by createGauge because createGauge checks only array bounds and nonzero entries, not the active mappings.

**Root Cause**  
Parallel factory arrays are treated as selectable source of truth, while isFactory/isGaugeFactory are toggled per address without reference counts or duplicate-slot cleanup.

**Impact**  
Emergency-council factory lifecycle operations can leave a supposedly removed or inactive factory pair still usable through a surviving nonzero _gaugeType slot.

**Mitigation**  
Prevent duplicate gauge factory entries during replace, either prevent duplicate pair/gauge pairs entirely or maintain reference counts, and make createGauge require isFactory[_factory] and isGaugeFactory[_gaugeFactory].

### [I-04] Voter factory length/count includes tombstones

**Severity**: Info  
**Location**: v4-contracts/contracts/Voter.sol:187; v4-contracts/contracts/Voter.sol:461; v4-contracts/contracts/Voter.sol:469

**Summary/Description**  
removeFactory tombstones the selected factory and gauge-factory entries by writing address(0) but leaves both arrays at the same length. factoryLength and gaugeFactoriesLength therefore include removed slots, and callers can still choose those _gaugeType indexes even though createGauge later reverts on the zero-address checks.

**Root Cause**  
The lifecycle removes entries by zeroing array positions instead of compacting arrays or separately exposing active counts.

**Impact**  
Removed factory indexes remain visible and selectable; this is mainly an inventory/UX and operational correctness issue unless combined with duplicate-slot drift.

**Mitigation**  
Use swap-and-pop or expose active factories separately from historical tombstones.

### [I-05] External bribe replacement can desynchronize Voter, Pair, and bribe accounting

**Severity**: Info  
**Location**: v4-contracts/contracts/Voter.sol:143; v4-contracts/contracts/Voter.sol:148; v4-contracts/contracts/Voter.sol:221; v4-contracts/contracts/Voter.sol:276; v4-contracts/contracts/Pair.sol:120; v4-contracts/contracts/ExternalBribe.sol:262

**Summary/Description**  
setExternalBribeFor immediately switches Voter.external_bribes for a gauge and only best-effort updates the Pair externalBribe. Existing votes were deposited into the previous bribe, but reset later withdraws from the current external_bribes mapping, so replacing the bribe after votes exist can make resets withdraw from a bribe with no matching balance. If the Pair update fails and is swallowed, vote accounting and swap fee routing can also point at different bribes.

**Root Cause**  
The bribe pointer is mutable while per-token vote balances live inside the old bribe, and the Pair binding update is swallowed with try/catch.

**Impact**  
Emergency-council bribe replacement during active votes can temporarily block vote reset/revote or route fees to a different bribe than Voter vote accounting.

**Mitigation**  
Disallow replacement while votes/weights exist, migrate/withdraw old bribe balances before switching, and do not swallow Pair.setExternalBribe failures for pair gauges.

### [I-06] Full-killed gauges leave stale vote and bribe accounting across reset or recreation

**Severity**: Info  
**Location**: v4-contracts/contracts/Voter.sol:31; v4-contracts/contracts/Voter.sol:41; v4-contracts/contracts/Voter.sol:207; v4-contracts/contracts/Voter.sol:217; v4-contracts/contracts/Voter.sol:221; v4-contracts/contracts/Voter.sol:229; v4-contracts/contracts/Voter.sol:407; v4-contracts/contracts/Voter.sol:417; v4-contracts/contracts/Voter.sol:418; v4-contracts/contracts/Voter.sol:423; v4-contracts/contracts/Voter.sol:485; v4-contracts/contracts/Voter.sol:490; v4-contracts/contracts/ExternalBribe.sol:250; v4-contracts/contracts/ExternalBribe.sol:262

**Summary/Description**  
killGaugeTotally deletes gauge-keyed mappings and gauges[pool], but it does not clear pool-keyed weights, votes, poolVote, usedWeights, totalWeight, or the old ExternalBribe balances. Until individual token owners reset, totalWeight can still include the fully removed pool and dilute Voter.notifyRewardAmount distribution ratios. When those owners later reset, _reset resolves the gauge through the now-deleted gauges[pool] and withdraws through the current external_bribes mapping rather than the original bribe, so the old bribe accounting is not reliably decremented. Recreating the gauge can also reuse stale pair-level bribe state.

**Root Cause**  
Full gauge removal clears gauge registration state without migrating or clearing the existing pool-keyed vote weights and bribe deposits.

**Impact**  
Fully removed gauges can leave stale denominator weight in emissions accounting and stale per-token bribe balances in the old ExternalBribe; later reset or gauge recreation can interact with mismatched bribe state.

**Mitigation**  
Before full removal, require zero pool weight or iterate/migrate affected vote state; otherwise preserve the old bribe pointer for withdrawals until all votes are cleared and remove the pool from distribution accounting.

### [I-07] Unrestricted Voter detach can clear veNFT gauge attachments

**Severity**: Info  
**Location**: v4-contracts/contracts/Voter.sol:431; v4-contracts/contracts/Voter.sol:432; v4-contracts/contracts/Voter.sol:433; v4-contracts/contracts/Voter.sol:444; v4-contracts/contracts/VotingEscrow.sol:1185; v4-contracts/contracts/VotingEscrow.sol:1190; v4-contracts/contracts/Gauge.sol:489; v4-contracts/contracts/Gauge.sol:492; v4-contracts/contracts/GaugeV4.sol:536; v4-contracts/contracts/GaugeV4.sol:539

**Summary/Description**  
Voter.attachTokenToGauge is restricted to live registered gauges, but detachTokenFromGauge has no equivalent caller or liveness check before forwarding to VotingEscrow.detach. Because VotingEscrow only checks that the caller is Voter, any external account can decrement an attached token's attachment counter through Voter, desynchronizing VotingEscrow.attachments from Gauge/GaugeV4 tokenIds.

**Root Cause**  
The Voter relay validates attach but not detach; VotingEscrow trusts Voter as the sole attachment mutator.

**Impact**  
An attached veNFT can be externally detached without going through the gauge's tokenIds clearing path. This can clear the transfer/withdraw guard in VotingEscrow while the gauge still records the account tokenId, and can also make later gauge detachment revert unless the user withdraws through the tokenId=0 branch.

**Mitigation**  
Require msg.sender to be a registered gauge for detachTokenFromGauge, and consider checking the gauge lifecycle state and account-token relation before forwarding detach.

### [I-08] Minter activeGaugeNumber can use stale lifecycle counts

**Severity**: Info  
**Location**: v4-contracts/contracts/Minter.sol:98; v4-contracts/contracts/Minter.sol:117; v4-contracts/contracts/Voter.sol:323; v4-contracts/contracts/Voter.sol:380; v4-contracts/contracts/Voter.sol:394; v4-contracts/contracts/Voter.sol:407; v4-contracts/contracts/Voter.sol:487; v4-contracts/contracts/Voter.sol:549; v4-contracts/contracts/Voter.sol:555

**Summary/Description**  
Minter.weekly_emission reads Voter.activeGaugeNumber before update_period notifies Voter for the new epoch. Voter resets activeGaugeNumber only inside notifyRewardAmount and rebuilds it later during successful distribute calls. Gauge create, pause, restart, and full kill paths do not refresh or invalidate the counter. The value consumed by the next weekly mint is therefore the previous distribution result, not a live gauge lifecycle count.

**Root Cause**  
activeGaugeNumber is maintained as a side effect of reward distribution, while Minter uses it as an input before the new epoch distribution side effects rebuild it.

**Impact**  
The next weekly emission can be over- or under-sized after lifecycle changes or incomplete distribution. This is recorded as Info because it depends on lifecycle timing and existing keeper/distribution assumptions, with the stronger public suppression variant tracked separately as M-07.

**Mitigation**  
Derive weekly emissions from current eligible gauge state or update the counter atomically when gauge lifecycle state changes, rather than using the previous distribution side effect.

### [I-09] GaugePlugin emergency-council allowance behavior is restrictive

**Severity**: Info  
**Location**: v4-contracts/contracts/GaugePlugin.sol

**Summary/Description**  
Reconstructed inventory entry for I-09. The original detailed report body was not recoverable after the report helper overwrote the untracked report file; source evidence remains in preception/bank-results and bank-progress metadata.

### [I-10] Delayed claimable can count low-share gauges as active

**Severity**: Info  
**Location**: v4-contracts/contracts/Voter.sol:485-493; v4-contracts/contracts/Voter.sol:517-528; v4-contracts/contracts/Voter.sol:549-559; v4-contracts/contracts/Minter.sol:98-104; v4-contracts/contracts/Minter.sol:112-132

**Summary/Description**  
Voter.notifyRewardAmount overwrites currentEpochRewardAmount for the latest weekly mint and advances the global reward index. If a gauge is not updated/distributed for one or more epochs, _updateFor later accrues the full missed index delta into cumulative claimable. Voter.distribute then uses that cumulative claimable divided by only the latest currentEpochRewardAmount to decide whether activeGaugeNumber should increment, so a gauge whose current-epoch share is below the active threshold can still be counted as active because older undistributed claimable is included.

**Root Cause**  
Per-gauge claimable is cumulative across missed index updates, while active-gauge thresholding uses the latest epoch denominator instead of the claimable amount attributable to that same epoch.

**Impact**  
Delayed or partial distribution can overstate activeGaugeNumber for the next Minter.weekly_emission calculation. This is Info because timely full distribution keeps the values aligned and the project documents keepers for weekly distribution, but the contract does not enforce that alignment.

**Mitigation**  
Track per-epoch gauge shares for active counting, or update/distribute all gauges before overwriting currentEpochRewardAmount. Alternatively separate active-gauge counting from cumulative claimable delivery.

### [I-11] GaugeV4 constructor can seed more rewards than MAX_REWARD_TOKENS

**Severity**: Info  
**Location**: v4-contracts/contracts/GaugeV4.sol:30, v4-contracts/contracts/GaugeV4.sol:105, v4-contracts/contracts/GaugeV4.sol:563

**Summary/Description**  
GaugeV4 enforces MAX_REWARD_TOKENS only when notifyRewardAmount adds a previously unknown reward token. The constructor accepts allowedRewardTokens from the factory and pushes every nonzero entry without checking the cap, duplicate entries, or Voter whitelist. The official Voter.createGauge path builds a fixed three-slot allowedRewards array, so the practical impact is limited to factory/direct or privileged lifecycle configuration rather than normal permissionless Voter creation.

**Root Cause**  
Constructor-time reward token seeding is not validated against the same cap and whitelist rules used by notifyRewardAmount.

**Impact**  
A GaugeV4 instance can start with more reward-list entries than the declared max, increasing per-action reward update work and bypassing the intended notify-time cap for constructor-seeded tokens. In the normal Voter.createGauge path, Voter passes only three entries.

**Mitigation**  
Apply the MAX_REWARD_TOKENS cap, duplicate filtering, and intended whitelist policy during constructor seeding, or make the factory enforce a bounded allowedRewards list before deployment.

### [I-12] Zero-amount splits can fill delegate token lists

**Severity**: Info  
**Location**: v4-contracts/contracts/VotingEscrow.sol

**Summary/Description**  
Reconstructed inventory entry for I-12. Bank notes identify split(amount == 0) as minting zero-power veNFTs and filling delegate checkpoints; source evidence remains in preception/bank-results.

### [I-13] Split and merge bypass same-block veNFT balance guard

**Severity**: Info  
**Location**: v4-contracts/contracts/VotingEscrow.sol

**Summary/Description**  
Reconstructed inventory entry for I-13. Bank notes identify split/merge as not propagating ownership_change to child or destination token IDs; source evidence remains in preception/bank-results.

### [I-14] VotingEscrow merge overstates public locked supply

**Severity**: Info  
**Location**: v4-contracts/contracts/VotingEscrow.sol

**Summary/Description**  
Reconstructed inventory entry for I-14. Bank notes identify merge as zeroing/burning the source and then using _deposit_for on the destination, which increments public supply without a token transfer; source evidence remains in preception/bank-results.

### [I-15] VotingEscrow merge can strand unclaimed tokenId rewards

**Severity**: Info  
**Location**: v4-contracts/contracts/VotingEscrow.sol; v4-contracts/contracts/RewardsDistributorV2.sol

**Summary/Description**  
Reconstructed inventory entry for I-15. Bank notes identify merge as burning a tokenId that may still have distributor or bribe claim state; source evidence remains in preception/bank-results.

### [I-16] RewardsDistributorV2 claim cursor edge case

**Severity**: Info  
**Location**: v4-contracts/contracts/RewardsDistributorV2.sol

**Summary/Description**  
Reconstructed inventory entry for I-16. The original detailed report body was not recoverable after the report helper overwrote the untracked report file; source evidence remains in preception/bank-results and bank-progress metadata.

### [I-17] No-op distributor claims can rewind the user epoch cursor

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Repeated no-op claims or duplicate tokenIds can move user_epoch_of backward while time_cursor_of stays at the already-claimed week. This does not enable double payment, but later claims may spend the bounded 50-iteration loop catching the epoch cursor back up before processing new reward weeks.  
**Location**: v4-contracts/contracts/RewardsDistributorV2.sol:178-219; v4-contracts/contracts/RewardsDistributorV2.sol:282-314

**Summary/Description**  
RewardsDistributorV2 compares the stored week cursor to raw last_token_time, but the claim loop itself is bounded by floor(last_token_time / WEEK) * WEEK. When time_cursor_of[tokenId] equals that rounded boundary, a repeated claim enters _claim, immediately breaks, and still stores user_epoch - 1. claim_many duplicate tokenIds can trigger the same no-op pass after the first occurrence advances the cursor.

**Root Cause**  
_claim uses raw last_token_time for the early return and rounded _last_token_time for the loop break, then unconditionally rewrites user_epoch_of after a zero-iteration claim path.

**Pre_conditions**  
1. The tokenId has an existing nonzero time_cursor_of at the rounded last-token week. 2. last_token_time is inside that same week but greater than the rounded week boundary. 3. The approved owner calls claim again or includes the same tokenId more than once in claim_many.

**Impact**  
The week cursor is not rewound, so already-claimed weeks are not paid again and token_last_balance is not reduced on zero amount. The impact is cursor/liveness friction: enough no-op rewinds can force future claims to consume loop iterations replaying user_point_history advancement.

**Proof of Concept**  
Not run; code-level review only. The path is time_cursor_of -> raw last_token_time check -> rounded _last_token_time break -> user_epoch_of write at lines 178, 190, 196, and 217-219.

**Mitigation**  
Use the same rounded last-token boundary for the early return and loop break, and only update user_epoch_of/time_cursor_of when the loop actually advances a cursor or distributes a claimable week.

### [I-18] RewardsDistributorV2 depositor handoff is one-way after Minter assignment

**Severity**: Info  
**Likelihood**: Low  
**Impact**: After deployment transfers distributor depositor authority to Minter, no in-scope Minter function can rotate that authority again; checkpoint_token remains callable only through Minter's hardcoded weekly checkpoint path.  
**Location**: v4-contracts/contracts/RewardsDistributorV2.sol:50-58; v4-contracts/contracts/RewardsDistributorV2.sol:97-100; v4-contracts/contracts/RewardsDistributorV2.sol:321-329; v4-contracts/contracts/Minter.sol:112-156; v4-contracts/script/Deployment.s.sol:119-141

**Summary/Description**  
RewardsDistributorV2 initializes depositor to the deployer and allows only the current depositor to call checkpoint_token or setDepositor. The deployment template transfers depositor to the Minter for both FLOW and WETH distributors, but Minter exposes no method to call RewardsDistributorV2.setDepositor. This makes the intended handoff effectively one-way after deployment; future rotation requires replacing/removing distributor entries at the Minter layer rather than changing the distributor's own depositor.

**Root Cause**  
setDepositor is authorized solely by the current depositor, and the deployed current depositor is a Minter contract that has no forwarding/admin function for setDepositor.

**Pre_conditions**  
1. Deployment calls rewardsDistributor.setDepositor(address(minter)). 2. The protocol later needs to rotate distributor checkpoint authority or recover from a wrong depositor assignment.

**Impact**  
The team cannot directly rotate the depositor on the existing distributor after the Minter assignment. Weekly checkpointing still works through Minter._checkpointRewardsDistributors, so this is a recoverability/configuration caveat rather than an immediate reward loss path.

**Proof of Concept**  
Not run; code-level review only. RewardsDistributorV2.setDepositor requires msg.sender == depositor, and Minter contains only update_period/addRewardsDistributor/removeRewardsDistributor around distributor calls.

**Mitigation**  
Use an owner/two-step admin role for depositor rotation, add a Minter-controlled forwarding function if Minter is intended to own the role, or keep depositor authority with a multisig and separately authorize checkpoint keepers.

### [I-19] Failing or oversized rewards-distributor list can block Minter weekly update

**Severity**: Info  
**Location**: v4-contracts/contracts/Minter.sol:139-156; v4-contracts/contracts/RewardsDistributorV2.sol:97-99,165-166; v4-contracts/contracts/Voter.sol:549-550

**Summary/Description**  
Minter stores team-configured rewards distributors in an unbounded array and update_period synchronously calls checkpoint_token() and checkpoint_total_supply() on every entry. A wrong-depositor distributor, non-contract entry, reverting implementation, or oversized list can make the weekly update path revert or run out of gas until the team repairs the list. Non-team users cannot mutate the list, so this remains an admin-configuration liveness risk.

**Root Cause**  
addRewardsDistributor accepts arbitrary addresses with no code/depositor/liveness check or length cap, and _checkpointRewardsDistributors uses a single unguarded loop with no try/catch around external distributor calls.

**Pre_conditions**  
1. Team adds a distributor entry that cannot complete both checkpoint calls, or adds enough entries for the loop to exceed practical gas limits. 2. A caller triggers Minter.update_period directly or through Voter.distribute after a new epoch is eligible.

**Impact**  
Weekly Minter.update_period and Voter.distribute cannot complete while the bad or oversized distributor list remains configured. The team can repair by removing entries, so impact is bounded by privileged configuration and recovery availability.

**Mitigation**  
Validate distributor address code and expected depositor before adding, reject duplicates, emit add/remove events, cap the list length, and consider try/catch plus per-distributor failure isolation for non-critical checkpoint entries.

### [I-20] Tiny gauged swaps can revert on zero rounded fee notification

**Severity**: Info  
**Location**: v4-contracts/contracts/Pair.sol:318; v4-contracts/contracts/Pair.sol:321; v4-contracts/contracts/Pair.sol:322; v4-contracts/contracts/Pair.sol:323; v4-contracts/contracts/ExternalBribe.sol:279; v4-contracts/contracts/ExternalBribe.sol:280

**Summary/Description**  
Pair.swap computes fee0/fee1 with integer division, then for gauged pairs calls _sendTokenFees whenever amount0In/amount1In is nonzero. If a nonzero input rounds its fee down to zero, Pair still calls ExternalBribe.notifyRewardAmount(token, 0), and ExternalBribe immediately reverts on require(amount > 0). This makes tiny gauged swaps revert instead of executing as zero-fee swaps.

**Root Cause**  
The Pair routing condition checks nonzero input amount instead of nonzero rounded fee amount before notifying the external bribe.

**Impact**  
Users cannot execute swaps whose input is positive but below the current fee precision threshold on gauged pairs. With the default PairFactory fees, this affects inputs whose fee rounds to zero; if an owner sets a pair fee to zero, every nonzero swap on a gauged pair reaches the same zero-notification path.

**Mitigation**  
Only call _sendTokenFees when the rounded fee amount is greater than zero, or make ExternalBribe.notifyRewardAmount tolerate zero-amount notifications as a no-op.

### [I-21] Pair pause only blocks swaps

**Severity**: Info  
**Location**: v4-contracts/contracts/Pair.sol; v4-contracts/contracts/factories/PairFactory.sol

**Summary/Description**  
Reconstructed inventory entry for I-21. Bank notes identify Pair pause checks as swap-only while mint, burn, skim, and sync remain callable; source evidence remains in preception/bank-results.

### [I-22] Pair current/prices/quote/sample can fail before enough observations

**Severity**: Info  
**Location**: v4-contracts/contracts/Pair.sol

**Summary/Description**  
Reconstructed inventory entry for I-22. Bank notes identify oracle observation readiness issues; source evidence remains in preception/bank-results.

### [I-23] Pair stable invariant can overflow at high reserves

**Severity**: Info  
**Location**: v4-contracts/contracts/Pair.sol

**Summary/Description**  
Reconstructed inventory entry for I-23. Bank notes identify stable invariant intermediate multiplication overflow at high normalized reserves; source evidence remains in preception/bank-results.

### [I-24] ProxyGauge pause leaves pair fee routing enabled

**Severity**: Info  
**Location**: v4-contracts/contracts/ProxyGauge.sol:56; v4-contracts/contracts/Voter.sol:380; v4-contracts/contracts/Voter.sol:389; v4-contracts/contracts/Voter.sol:390; v4-contracts/contracts/Pair.sol:120; v4-contracts/contracts/Pair.sol:128; v4-contracts/contracts/Pair.sol:321; v4-contracts/contracts/Voter.sol:415; v4-contracts/contracts/Voter.sol:424

**Summary/Description**  
ProxyGauge implements IGauge.stake() as address(0). Voter.pauseGauge() and restartGauge() use IGauge(_gauge).stake() to locate the pair and wrap Pair.setHasGauge() in try/catch, so for proxy gauges the call is made against address(0) and the failure is swallowed. The gauge is marked inactive in Voter, but the Pair.hasGauge flag can remain true.

**Root Cause**  
The proxy adapter does not return its associated pool from stake(), while Voter pause/restart relies on stake() instead of poolForGauge[_gauge] to synchronize pair gauge state.

**Impact**  
After an authorized pause of a proxy gauge associated with a pair, the pair can keep routing swap fees through its external bribe path because Pair.swap() checks hasGauge before sending fees. Full kill uses poolForGauge[_gauge] and does not have this specific address(0) lookup problem.

**Mitigation**  
Make ProxyGauge store and return its associated pool, or update Voter pause/restart to use poolForGauge[_gauge] and require the pair-state synchronization to succeed when a pool binding exists.

### [I-25] ProxyGaugeFactory can reuse one proxy gauge across Voter pool keys

**Severity**: Info  
**Location**: v4-contracts/contracts/factories/ProxyGaugeFactory.sol:19; v4-contracts/contracts/factories/ProxyGaugeFactory.sol:24; v4-contracts/contracts/factories/ProxyGaugeFactory.sol:28; v4-contracts/contracts/factories/ProxyGaugeFactory.sol:34; v4-contracts/contracts/Voter.sol:323; v4-contracts/contracts/Voter.sol:354; v4-contracts/contracts/Voter.sol:359; v4-contracts/contracts/Voter.sol:363; v4-contracts/contracts/Voter.sol:364; v4-contracts/contracts/Voter.sol:517

**Summary/Description**  
ProxyGaugeFactory.createGauge returns a previously whitelisted proxy instead of deploying a fresh gauge. Voter.createGauge checks only that gauges[_pool] is empty, then accepts whatever gauge address the factory returns and writes poolForGauge[_gauge] = _pool. If an operator first binds a proxy to a real pool through deployGauge/createGauge and later whitelists that proxy address itself, a privileged non-pair createGauge call can return the same proxy for a second Voter pool key and overwrite poolForGauge for the already-used gauge.

**Root Cause**  
Voter validates pool uniqueness, but not returned gauge uniqueness, while ProxyGaugeFactory supports pre-whitelisted reusable gauge addresses.

**Impact**  
This is privileged configuration dependent, but it can make one proxy gauge represent two Voter pool keys. Because Voter._updateFor resolves the pool through poolForGauge[_gauge], the later binding can cause accounting and lifecycle operations for the earlier pool to use the wrong pool key.

**Mitigation**  
Require !isGauge[_gauge] before accepting a factory return value, or make ProxyGaugeFactory consume whitelist entries / reject already-returned gauges. For proxy pseudo-pools, store an explicit one-to-one binding and prevent reuse.

### [I-26] Flow minter handoff is one-step and has no recovery guard

**Severity**: Info  
**Location**: v4-contracts/contracts/Flow.sol

**Summary/Description**  
Reconstructed inventory entry for I-26. Bank notes identify Flow.setMinter as a one-step handoff without zero-address guard, accept step, event, or recovery path; source evidence remains in preception/bank-results.

### [I-27] GaugeFactoryV4 oFlow update does not synchronize minter roles

**Severity**: Info  
**Location**: v4-contracts/contracts/factories/GaugeFactoryV4.sol:20; v4-contracts/contracts/factories/GaugeFactoryV4.sol:22; v4-contracts/contracts/factories/GaugeFactoryV4.sol:23; v4-contracts/contracts/factories/GaugeFactoryV4.sol:28; v4-contracts/contracts/factories/GaugeFactoryV4.sol:33; v4-contracts/contracts/factories/GaugeFactoryV4.sol:34; v4-contracts/contracts/GaugeV4.sol:609; v4-contracts/contracts/GaugeV4.sol:611; v4-contracts/contracts/GaugeV4.sol:612; v4-contracts/contracts/GaugeV4.sol:613; v4-contracts/contracts/OptionTokenV4.sol:190; v4-contracts/contracts/OptionTokenV4.sol:519; v4-contracts/contracts/GaugeV4.sol:282

**Summary/Description**  
GaugeFactoryV4 grants MINTER_ROLE to a newly created gauge only during createGauge. Later setOFlow and updateOFlowFor update factory/gauge oFlow pointers but do not grant the existing gauge MINTER_ROLE on the new oFlow or revoke the old token role. GaugeV4.setOFlow also marks the new oFlow as an authorized oToken caller without removing the previous oFlow from isOToken. Reward claims are protected from loss because GaugeV4 catches oFlow.mint failures and falls back to raw FLOW, but the lifecycle update can silently degrade intended oFlow minting and leave stale option-token depositWithLock authorization unless the owner performs separate role/token cleanup.

**Root Cause**  
The oFlow replacement lifecycle updates pointers but does not synchronize AccessControl roles or the GaugeV4 isOToken authorization set.

**Impact**  
Existing gauges updated to a new oFlow may fail to mint option-token rewards and instead pay raw FLOW. Old option-token contracts can remain authorized for locked GaugeV4 deposits until explicitly removed.

**Mitigation**  
When updating oFlow for an existing gauge, grant the gauge MINTER_ROLE on the new oFlow, revoke or document the old role, and clear old isOToken authorization when replacement rather than multi-token support is intended.

### [I-28] Legacy Gauge lacks GaugeV4 oFlow code-length guard

**Severity**: Info  
**Location**: v4-contracts/contracts/Gauge.sol:264-280,562-566; v4-contracts/contracts/GaugeV4.sol:269-285,609-614; v4-contracts/contracts/factories/GaugeFactory.sol:18-22,26-33

**Summary/Description**  
Legacy Gauge attempts the FLOW reward path through IOptionToken(oFlow).mint without first checking that oFlow is a deployed contract. Under the project compiler, a high-level no-return interface call to a no-code address performs a code-existence check before the call, so the catch fallback is not reached and the claim reverts. Accounting rolls back, but FLOW reward claims are unavailable until oFlow is corrected. GaugeV4 adds the missing address/code guard and falls back to raw FLOW when oFlow is unset or not deployed.

**Root Cause**  
Gauge.sol lacks the oFlow != address(0) and oFlow.code.length != 0 guard that exists in GaugeV4 before attempting option-token minting.

**Impact**  
Misconfigured or unset legacy Gauge oFlow blocks FLOW reward claims for affected users until the gauge factory owner updates oFlow to a valid contract with the required minting role.

**Mitigation**  
Mirror GaugeV4's guard in Gauge.sol before the mint attempt, and ensure factory oFlow updates also grant the replacement option token's minter role to existing gauges.

### [I-29] Regular setDiscount(0) is accepted for OptionTokenV4 exercise

**Severity**: Info  
**Location**: v4-contracts/contracts/OptionTokenV4.sol

**Summary/Description**  
Reconstructed inventory entry for I-29. Bank notes identify setDiscount(0) as accepted and making normal exercise collect zero payment; source evidence remains in preception/bank-results.

### [I-30] LP discount endpoints can collapse and divide by zero

**Severity**: Info  
**Location**: v4-contracts/contracts/OptionTokenV4.sol

**Summary/Description**  
Reconstructed inventory entry for I-30. Bank notes identify minLPDiscount == maxLPDiscount as passing setters but dividing by zero in lock-duration slope math; source evidence remains in preception/bank-results.

### [I-31] Full-killed pools can break default distribution ranges

**Severity**: Info  
**Location**: v4-contracts/contracts/Voter.sol

**Summary/Description**  
Reconstructed inventory entry for I-31. Bank notes identify killed pool entries remaining in pools[] while gauges[pool] is deleted, breaking range distribution; source evidence remains in preception/bank-results.

### [I-32] activeGaugeNumber == 0 still permits floor emissions

**Severity**: Info  
**Location**: v4-contracts/contracts/Minter.sol:98; v4-contracts/contracts/Minter.sol:100; v4-contracts/contracts/Minter.sol:101; v4-contracts/contracts/Minter.sol:117; v4-contracts/contracts/Minter.sol:131; v4-contracts/contracts/Minter.sol:132; v4-contracts/contracts/Voter.sol:485; v4-contracts/contracts/Voter.sol:490

**Summary/Description**  
Minter.weekly_emission returns weeklyPerGauge when Voter.activeGaugeNumber is zero, so update_period still attempts to fund Voter. Voter.notifyRewardAmount then computes amount * 1e18 / totalWeight without a zero-totalWeight guard. If there are no current votes, the weekly update reverts instead of cleanly minting zero or handling an idle epoch.

**Root Cause**  
The zero-active fallback in Minter is not paired with a totalWeight == 0 branch in Voter.notifyRewardAmount.

**Impact**  
Weekly emission finalization is unavailable while totalWeight is zero. The revert rolls back active_period, minting, team transfer, checkpoints, and approval, so there is no partial state corruption; a later positive vote can unblock the path.

**Mitigation**  
Handle totalWeight == 0 explicitly, either by skipping Voter notification and minting zero voter emissions or by carrying the amount forward in a tracked idle bucket.

### [I-33] Votes to full-killed pools can consume the epoch without voting power

**Severity**: Info  
**Location**: v4-contracts/contracts/Voter.sol:257; v4-contracts/contracts/Voter.sol:261; v4-contracts/contracts/Voter.sol:265; v4-contracts/contracts/Voter.sol:267; v4-contracts/contracts/Voter.sol:282; v4-contracts/contracts/Voter.sol:287; v4-contracts/contracts/Voter.sol:407; v4-contracts/contracts/Voter.sol:423

**Summary/Description**  
_vote sums every user-supplied weight into _totalVoteWeight before checking whether the corresponding pool currently has a valid gauge. In the application loop it only applies weights when isGauge[_gauge] is true. A stale or full-killed pool therefore remains in the denominator but receives no vote, so part or all of the veNFT's voting power can be consumed for the epoch without becoming pool weight or totalWeight.

**Root Cause**  
The vote denominator is built from unvalidated pool entries, while actual pool/total weight increments are gated by isGauge and isAlive.

**Impact**  
Users can accidentally consume their one voting action for an epoch with less voting power applied than intended, especially when a front end or cached pool list includes a pool whose gauge was fully killed.

**Mitigation**  
Validate all pools and gauges before setting lastVoted or computing _totalVoteWeight, or build the denominator only from entries that will actually be applied.

### [I-34] Empty Voter vote leaves veNFT marked voted after clearing weights

**Severity**: Info  
**Location**: v4-contracts/contracts/Voter.sol:249; v4-contracts/contracts/Voter.sol:282; v4-contracts/contracts/Voter.sol:287; v4-contracts/contracts/VotingEscrow.sol:955; v4-contracts/contracts/VotingEscrow.sol:1175; v4-contracts/contracts/VotingEscrow.sol:1180; v4-contracts/contracts/VotingEscrow.sol:1195; v4-contracts/contracts/VotingEscrow.sol:1217

**Summary/Description**  
vote([], []) or a vote containing no valid live gauge sets lastVoted and enters _vote. _vote first clears prior Voter weights through _reset, then with _usedWeight == 0 skips IVotingEscrow.voting and never calls IVotingEscrow.abstain. If the tokenId was already marked voted from an earlier vote, VotingEscrow.voted remains true even though Voter.usedWeights, votes, poolVote, and bribe balances were cleared.

**Root Cause**  
The zero-used-weight vote path synchronizes Voter accounting but not VotingEscrow.voted state.

**Impact**  
The veNFT can be left non-transferable and unable to withdraw, merge, or split until an approved owner performs a reset in a later epoch.

**Mitigation**  
When _vote ends with _usedWeight == 0 after clearing prior votes, call IVotingEscrow.abstain or reject empty/no-valid votes before mutating lastVoted and reset state.

### [I-35] Gauge lifecycle prompt-bank issue

**Severity**: Info  
**Location**: See preception/bank-results references for the original prompt-bank evidence.

**Summary/Description**  
Reconstructed inventory entry for I-35. The original detailed report body was not recoverable after the report helper overwrote the untracked report file; source evidence remains in preception/bank-results and bank-progress metadata.

### [I-36] startActivePeriod at exact week boundary can delay first mint

**Severity**: Info  
**Location**: v4-contracts/contracts/Minter.sol:15; v4-contracts/contracts/Minter.sol:54; v4-contracts/contracts/Minter.sol:57; v4-contracts/contracts/Minter.sol:61; v4-contracts/contracts/Minter.sol:112; v4-contracts/contracts/Minter.sol:114; v4-contracts/script/StartActivePeriod.s.sol:18

**Summary/Description**  
startActivePeriod is one-shot and sets active_period to floor(block.timestamp / WEEK) * WEEK. update_period requires block.timestamp >= active_period + WEEK. If startActivePeriod is called exactly on a week boundary, active_period equals the current timestamp and the first eligible mint is delayed until the next boundary rather than becoming available for the boundary just reached.

**Root Cause**  
The initializer path floors to the current boundary while update_period requires a full additional WEEK after active_period.

**Impact**  
First emission timing can be delayed by one week in the exact-boundary initialization case. This is an initialization timing caveat, not a partial-mint or accounting corruption issue.

**Mitigation**  
If boundary-start minting is desired, set active_period to the previous boundary when block.timestamp is exactly divisible by WEEK, or document that startActivePeriod must be called before the intended first boundary.

### [I-37] Post-epoch votes before distribution can redirect weekly emissions

**Severity**: Info  
**Location**: README.md:76; README.md:87; v4-contracts/contracts/Voter.sol:102; v4-contracts/contracts/Voter.sol:274; v4-contracts/contracts/Voter.sol:283; v4-contracts/contracts/Voter.sol:485; v4-contracts/contracts/Voter.sol:490; v4-contracts/contracts/Minter.sol:112; v4-contracts/contracts/Minter.sol:117; v4-contracts/contracts/Minter.sol:131; v4-contracts/contracts/Minter.sol:132

**Summary/Description**  
onlyNewEpoch permits a token to reset or vote after an epoch flips. If that action happens before the next distribute/update_period path, Voter weights and totalWeight are changed before Voter.notifyRewardAmount computes the new emission index from amount / totalWeight. This can redirect the distribution basis to post-flip votes rather than the prior snapshot. The README explicitly treats actions after epoch flip but before distribute as acceptable risk unless high severity, so this remains an info-level lifecycle caveat.

**Root Cause**  
Distribution uses the current Voter weights at notifyRewardAmount time, while new-epoch voting is allowed before distribution is executed.

**Impact**  
Post-flip/pre-distribution vote timing can change which pools receive the weekly emission index, but the contest README marks this timing class as acceptable unless a high-severity path is shown.

**Mitigation**  
If this behavior is not desired, snapshot weights at epoch close or force distribution before accepting new-epoch vote mutations.

### [I-38] Tiny volatile swaps can round fees to zero

**Severity**: Info  
**Location**: v4-contracts/contracts/Pair.sol

**Summary/Description**  
Reconstructed inventory entry for I-38. Bank notes identify chunked low-decimal swaps as rounding every fee to zero while an aggregate swap would pay nonzero fees; source evidence remains in preception/bank-results.

### [I-39] Exact-boundary reset can clear bribe snapshot

**Severity**: Info  
**Location**: v4-contracts/contracts/Voter.sol:201; v4-contracts/contracts/Voter.sol:207; v4-contracts/contracts/Voter.sol:221; v4-contracts/contracts/Voter.sol:234; v4-contracts/contracts/Voter.sol:249; v4-contracts/contracts/ExternalBribe.sol:155; v4-contracts/contracts/ExternalBribe.sol:166; v4-contracts/contracts/ExternalBribe.sol:212; v4-contracts/contracts/ExternalBribe.sol:234; v4-contracts/contracts/ExternalBribe.sol:238; v4-contracts/contracts/ExternalBribe.sol:262

**Summary/Description**  
At an exact epoch boundary, Voter.onlyNewEpoch permits reset, poke, or vote for a token that voted in the prior epoch. These paths call _reset, which withdraws the prior bribe balance and writes a same-timestamp ExternalBribe checkpoint. ExternalBribe.earned calculates the just-ended epoch using getPriorBalanceIndex(epochEnd) and getPriorSupplyIndex(epochEnd), so the boundary withdrawal checkpoint can be selected as the end-of-epoch balance and reduce or erase the just-ended epoch's bribe entitlement.

**Root Cause**  
The bribe reward snapshot uses checkpoints at epochEnd inclusively, while Voter permits vote-reset mutations at that exact timestamp.

**Impact**  
A user or automation that resets/revotes before claiming at the boundary can lose bribe rewards for the epoch that just ended; changing pools at the boundary is the clearest case because the old bribe receives a zero checkpoint at epochEnd.

**Mitigation**  
Disallow reset/poke/vote at the exact epoch boundary until after the bribe snapshot point, or make bribe earned use the checkpoint strictly before epochEnd for the completed epoch.

### [I-40] Flow zero-address transfers can strand supply without burning

**Severity**: Info  
**Location**: v4-contracts/contracts/Flow.sol

**Summary/Description**  
Reconstructed inventory entry for I-40. Bank notes identify Flow._transfer as allowing address(0) recipients without reducing totalSupply; source evidence remains in preception/bank-results.

### [I-41] Volatile pair math can overflow at extreme reserves

**Severity**: Info  
**Location**: v4-contracts/contracts/Pair.sol

**Summary/Description**  
Reconstructed inventory entry for I-41. Bank notes identify volatile x*y and quote math overflow at extreme reserves; source evidence remains in preception/bank-results.

### [I-42] Per-token approved spender cannot withdraw expired veNFT

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Per-token approved addresses cannot execute the withdraw path even though withdraw first accepts them via _isApprovedOrOwner  
**Location**: v4-contracts/contracts/VotingEscrow.sol:251, v4-contracts/contracts/VotingEscrow.sol:258, v4-contracts/contracts/VotingEscrow.sol:260, v4-contracts/contracts/VotingEscrow.sol:295, v4-contracts/contracts/VotingEscrow.sol:300, v4-contracts/contracts/VotingEscrow.sol:302, v4-contracts/contracts/VotingEscrow.sol:537, v4-contracts/contracts/VotingEscrow.sol:538, v4-contracts/contracts/VotingEscrow.sol:543, v4-contracts/contracts/VotingEscrow.sol:955, v4-contracts/contracts/VotingEscrow.sol:956, v4-contracts/contracts/VotingEscrow.sol:975

**Summary/Description**  
VotingEscrow.withdraw first authorizes the caller with _isApprovedOrOwner, which returns true for the current per-token approved address. After clearing lock state and transferring LP, withdraw calls _burn. _burn repeats _isApprovedOrOwner but then calls approve(address(0), tokenId); approve only permits the owner or an operator, not the already approved address. A per-token approved spender therefore cannot complete withdraw despite passing the entry permission check.

**Root Cause**  
The burn helper clears approval through the public approve function instead of directly clearing idToApprovals or allowing the current approved spender. The public approve permission logic omits the current per-token approved address.

**Pre_conditions**  
1. A veNFT has expired and has no attachments and no active voted flag. 2. The owner approved a single address with approve(spender, tokenId). 3. The spender calls withdraw(tokenId) without also being owner or an operator approved for all.

**Impact**  
The transaction reverts in _burn after the initial withdraw permission check would accept the caller. The parent revert also rolls back the prior LP transfer, so this is a correctness and integration issue rather than direct fund loss; the owner or an operator can still withdraw.

**Proof of Concept**  
Not run; code-level review only. _isApprovedOrOwner treats idToApprovals[tokenId] as authorized, but approve(address(0), tokenId) only accepts idToOwner[tokenId] or ownerToOperators[tokenOwner][msg.sender].

**Mitigation**  
In _burn, clear idToApprovals[tokenId] directly after validating _isApprovedOrOwner, or update approval clearing so the current approved spender can complete burn flows that intentionally authorize approved spenders.

### [I-43] delegateBySig does not match delegate self-delegation and typed-data handling

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: v4-contracts/contracts/VotingEscrow.sol:1258; v4-contracts/contracts/VotingEscrow.sol:1282; v4-contracts/contracts/VotingEscrow.sol:1488; v4-contracts/contracts/VotingEscrow.sol:1502; v4-contracts/contracts/VotingEscrow.sol:1507

**Summary/Description**  
VotingEscrow.delegate treats address(0) as self-delegation before calling _delegate, but delegateBySig forwards address(0) directly. _delegate then stores _delegates[signatory] = address(0) and moves all currently owned tokenIds to dstRep address(0), where _moveAllDelegates skips the destination write. Afterward delegates(signatory) reports self because zero is the default, but the signatory's current tokenIds are absent from the self checkpoint until another movement occurs. The same function also builds its EIP-712 domain with DOMAIN_TYPEHASH that omits version while abi.encode includes version, so standard typed-data signers cannot derive that digest from the declared domain.

**Root Cause**  
delegateBySig bypasses the zero-address normalization in delegate, and DOMAIN_TYPEHASH declares EIP712Domain(string name,uint256 chainId,address verifyingContract) while the domain separator encodes name, version, chainId, and verifyingContract.

**Pre_conditions**  
1. A signer signs a delegation with delegatee = address(0), or uses a standard typed-data flow for delegateBySig. 2. The signed delegation is submitted through delegateBySig.

**Impact**  
A zero-delegate signature can remove the signer's current delegated tokenIds from all vote checkpoints while delegates(signatory) still reports self. Separately, standard typed-data integrations are likely to fail to produce signatures accepted by this implementation.

**Mitigation**  
Normalize delegatee == address(0) to signatory inside delegateBySig before calling _delegate, and make DOMAIN_TYPEHASH match the encoded fields, for example by including string version in the EIP712Domain type or by removing version from the encoded domain.

### [I-44] Fully removed gauges can be restarted into inconsistent Voter state

**Severity**: Info  
**Location**: v4-contracts/contracts/Voter.sol:407; v4-contracts/contracts/Voter.sol:417; v4-contracts/contracts/Voter.sol:423; v4-contracts/contracts/Voter.sol:394; v4-contracts/contracts/Voter.sol:400; v4-contracts/contracts/Voter.sol:401; v4-contracts/contracts/Voter.sol:402; v4-contracts/contracts/Voter.sol:403; v4-contracts/contracts/Voter.sol:431; v4-contracts/contracts/Voter.sol:438

**Summary/Description**  
killGaugeTotally deletes isGauge, poolForGauge, gauges[pool], external_bribes, claimable, and supplyIndex. restartGauge only requires !isAlive, so an authorized caller can pass a fully removed gauge address and set isAlive back to true and Pair.hasGauge back to true without restoring the deleted Voter mappings. The result is a gauge that appears alive but is not registered as a gauge and cannot follow the normal attach/deposit/vote/distribute lifecycle.

**Root Cause**  
restartGauge does not verify isGauge[_gauge] or otherwise distinguish paused gauges from fully removed gauges.

**Impact**  
A lifecycle mistake can re-enable pair fee routing and an alive flag for a fully removed gauge while Voter still lacks the pool and bribe bindings needed for normal operation.

**Mitigation**  
Require isGauge[_gauge] in restartGauge, or add a separate tombstone state that prevents restart after full removal.

### [I-45] Paused gauges can recover skipped emissions after restart

**Severity**: Info  
**Location**: v4-contracts/contracts/Voter.sol:380; v4-contracts/contracts/Voter.sol:394; v4-contracts/contracts/Voter.sol:517; v4-contracts/contracts/Voter.sol:523; v4-contracts/contracts/Voter.sol:527; v4-contracts/contracts/Voter.sol:549

**Summary/Description**  
pauseGauge clears claimable and marks the gauge inactive, but it does not checkpoint the gauge's supplyIndex to the current global index. restartGauge only marks the gauge alive again. If nobody calls updateGauge/distribute for that gauge while it is paused, the next post-restart _updateFor sees isAlive true and accrues the entire index delta since the old supplyIndex, including the period that was supposed to be skipped.

**Root Cause**  
Lifecycle pause/restart mutates isAlive and claimable without synchronizing supplyIndex before or during the status change.

**Impact**  
A paused gauge can later receive emissions for the paused interval after restart, contrary to the intended cleanup performed by pauseGauge. The issue is permissioned/lifecycle-dependent, so it is recorded as Info unless a stronger impact path is demonstrated.

**Mitigation**  
Call _updateFor before changing liveness, or explicitly set supplyIndex[_gauge] = index when pausing/restarting so skipped inactive intervals cannot be re-accrued.

### [I-46] Gauge reward token swap-out bypasses whitelist validation

**Severity**: Info  
**Location**: v4-contracts/contracts/GaugeV4.sol:601; v4-contracts/contracts/GaugeV4.sol:602; v4-contracts/contracts/GaugeV4.sol:603; v4-contracts/contracts/GaugeV4.sol:604; v4-contracts/contracts/GaugeV4.sol:605; v4-contracts/contracts/GaugeV4.sol:606; v4-contracts/contracts/GaugeV4.sol:563; v4-contracts/contracts/GaugeV4.sol:566; v4-contracts/contracts/GaugeV4.sol:567; v4-contracts/contracts/GaugeV4.sol:568; README.md:18

**Summary/Description**  
GaugeV4's normal first-time reward token path checks Voter.isWhitelisted before marking a token as a reward, matching the README requirement for non-pool reward tokens. swapOutRewardToken is team-gated but directly sets isReward[newToken] = true and replaces the rewards array entry without checking the new token against Voter.isWhitelisted, zero address, code length, or stake-token constraints. After the swap, notifyRewardAmount sees the new token as an existing reward and skips the whitelist gate.

**Root Cause**  
The reward-token replacement path does not reuse the validation applied when adding a new reward token through notifyRewardAmount.

**Pre_conditions**  
1. The VotingEscrow team calls swapOutRewardToken with a new token that has not passed the intended reward whitelist policy. 2. The swapped token is later funded through notifyRewardAmount or claimed through the normal reward flow.

**Impact**  
This is not a permissionless reward drain because the swap requires the team address, but it allows lifecycle configuration to bypass the documented reward-token whitelist and can make a non-whitelisted or invalid token part of the gauge reward list.

**Mitigation**  
Require newToken to satisfy the same policy as a first-time notifyRewardAmount token, including whitelist membership for non-pool reward tokens and basic invalid-token rejection.

### [I-47] RewardsDistributorV2 delayed checkpoint can leave rewards unassigned

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Rewards received before a checkpoint delayed by more than 20 weekly buckets can be marked as accounted without being fully assigned to tokens_per_week; follow-on claims may also need manual supply catch-up before they stop reverting on missing ve_supply buckets.  
**Location**: v4-contracts/contracts/RewardsDistributorV2.sol:64-94; v4-contracts/contracts/RewardsDistributorV2.sol:142-162; v4-contracts/contracts/RewardsDistributorV2.sol:195-214; v4-contracts/contracts/RewardsDistributorV2.sol:226-279; v4-contracts/contracts/Minter.sol:152-156

**Summary/Description**  
RewardsDistributorV2 caps both token and ve-supply checkpoint catch-up loops at 20 iterations. _checkpoint_token sets token_last_balance to the full current token balance before the loop, but if more than 20 weekly buckets elapsed it only assigns the first 20 buckets and leaves the remainder of the delta unrepresented in tokens_per_week. Separately, _checkpoint_total_supply may leave later ve_supply buckets unset after one call, so claim and claimable paths that span those buckets can divide by zero. claimable is additionally inconsistent with claim during ordinary stale-supply windows because claim first attempts one _checkpoint_total_supply call, while claimable is a view and uses the existing ve_supply mapping as-is.

**Root Cause**  
The token checkpoint records the entire balance delta as accounted before a bounded 20-iteration allocation loop, and the supply checkpoint advances at most 20 weekly buckets per call while claim performs only one automatic supply checkpoint.

**Pre_conditions**  
1. The distributor has uncheckpointed reward-token balance. 2. More than 20 weekly buckets elapse without a successful token/supply checkpoint through Minter.update_period or direct checkpoint_total_supply. 3. A later checkpoint or claim is executed.

**Impact**  
A delayed token checkpoint permanently excludes the elapsed time beyond the first 20 weekly buckets from tokens_per_week while token_last_balance still tracks the full balance, leaving those reward tokens stuck in the distributor. Claims after a long delay can also revert on ve_supply[week] == 0 for unprocessed weeks until an external caller manually catches up the supply cursor. Before that catch-up, claimable can revert or report from stale supply data even when claim would succeed after its automatic one-call supply checkpoint.

**Proof of Concept**  
Not run; code-level review only. With since_last > 20 weeks, the loop at RewardsDistributorV2.sol:75-93 allocates at most 20 weekly slices, then exits after token_last_balance was already set to token_balance at line 67.

**Mitigation**  
For token checkpointing, either loop until the current timestamp is covered or only advance token_last_balance by the amount actually assigned to tokens_per_week. For supply checkpointing, make claim catch up until time_cursor exceeds the rounded last token time or skip zero-reward weeks before dividing by ve_supply. For views, expose a claimable path that either documents the checkpoint precondition or computes from fresh supply data without relying on stale ve_supply slots.

### [I-48] RewardsDistributorV2 claim_many zero sentinel reverts before stopping

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Callers that use tokenId 0 as the intended terminator cannot stop the batch; the call reverts before the break. This is not a silent third-party grief path because the caller supplies the array and tokenId 0 has no owner.  
**Location**: v4-contracts/contracts/RewardsDistributorV2.sol:303-307; v4-contracts/contracts/VotingEscrow.sol:190-191; v4-contracts/contracts/VotingEscrow.sol:295-306; v4-contracts/contracts/VotingEscrow.sol:846-850

**Summary/Description**  
RewardsDistributorV2 claim_many appears to support tokenId 0 as a sentinel, but V2 checks isApprovedOrOwner before checking _tokenId == 0. VotingEscrow never assigns an owner to tokenId 0, so normal callers cannot pass the approval check and the sentinel break is unreachable.

**Root Cause**  
The zero-token break is placed after the external authorization check in the claim_many loop.

**Pre_conditions**  
1. A caller submits a claim_many array containing tokenId 0. 2. The caller is not approved for tokenId 0, which is the normal VotingEscrow state because token IDs start at 1.

**Impact**  
The batch reverts instead of stopping at tokenId 0. No rewards are misaccounted and no unrelated user can inject the sentinel into another caller's calldata.

**Proof of Concept**  
Not run; code-level review only. claim_many loads _tokenId at line 304, checks approval at line 305, and only then checks _tokenId == 0 at line 306.

**Mitigation**  
Move the _tokenId == 0 break before isApprovedOrOwner, or remove the sentinel branch and require callers to pass the exact array length.

### [I-49] OptionTokenV4 LP exercise return values omit payment amount

**Severity**: Info  
**Location**: v4-contracts/contracts/OptionTokenV4.sol:598; v4-contracts/contracts/OptionTokenV4.sol:607; v4-contracts/contracts/OptionTokenV4.sol:657; v4-contracts/contracts/OptionTokenV4.sol:666

**Summary/Description**  
exerciseVe and exerciseLp declare named return variable paymentAmount, but each function tuple-declares a local variable with the same name when reading getPaymentTokenAmountForExerciseLp. Solidity 0.8.13 treats this as shadowing, so the named paymentAmount return slot remains zero even though the event and payment checks use the local value.

**Root Cause**  
The functions rely on named return variables but shadow paymentAmount with a local tuple declaration instead of assigning into the return variable.

**Pre_conditions**  
A caller or integrator relies on the first returned value from exerciseVe or exerciseLp instead of reading events or recomputing the price.

**Impact**  
Successful ve and LP exercises can return 0 for paymentAmount while the user was charged the nonzero premium and liquidity-side payment. This can mislead integrations that use the return value for accounting, receipts, or UI settlement.

**Mitigation**  
Assign tuple results into the named return variable, for example declare only paymentAmountToAddLiquidity separately or use (paymentAmount, paymentAmountToAddLiquidity) = getPaymentTokenAmountForExerciseLp(...).

### [I-50] Expired OptionTokenV4 supply remains live against future backing

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: v4-contracts/contracts/OptionTokenV4.sol:519; v4-contracts/contracts/OptionTokenV4.sol:535; v4-contracts/contracts/OptionTokenV4.sol:542; v4-contracts/contracts/OptionTokenV4.sol:550; v4-contracts/contracts/OptionTokenV4.sol:569; v4-contracts/contracts/Flow.sol:86

**Summary/Description**  
OptionTokenV4.expire burns the contract's current underlyingToken balance but does not burn outstanding option token balances or prevent future mint/exercise activity. If backing is later deposited into the same expired option token, old expired option balances remain valid in the exercise paths and can consume that later backing.

**Root Cause**  
expire only calls IERC20(underlyingToken).burn(balanceOf(address(this))) after the cooldown. The mint, exercise, exerciseVe, exerciseLp, pause, and unPause lifecycle gates do not check expiryTime or an expired terminal state.

**Pre_conditions**  
1. An OptionTokenV4 instance has a nonzero expiryCooldownTime. 2. Admin calls startExpire and later expire. 3. Some option supply remains outstanding. 4. Later backing is deposited or minted into the same option contract, or the contract otherwise receives underlying tokens.

**Impact**  
Expired option balances remain as ERC20 supply after the backing is destroyed. Later backing in the same contract can be claimed by old balances because the exercise paths only check pause flags, not expiry status. This is most likely an operational lifecycle/accounting issue because the later backing deposit path is permissioned or accidental.

**Mitigation**  
Make expiry terminal: block mint and all exercise paths once expiry has started or completed, and either burn or otherwise account for outstanding option supply when expire destroys backing. Prevent unPause from re-enabling exercises after expiry.

### [I-51] OptionTokenV4 treasury fee list does not enforce MAX_FEES

**Severity**: Info  
**Location**: v4-contracts/contracts/OptionTokenV4.sol:34; v4-contracts/contracts/OptionTokenV4.sol:425; v4-contracts/contracts/OptionTokenV4.sol:432; v4-contracts/contracts/OptionTokenV4.sol:718

**Summary/Description**  
OptionTokenV4 declares MAX_FEES = 50 and an OptionToken_InvalidFee error, but addTreasury and replaceTreasury only validate the treasury address and array position. _takeFees computes each treasury share from the original paymentAmount and subtracts each fee from the remaining gauge/reward amount. Configured total fees above 50% are accepted; totals above 100% make exercise paths revert atomically from remaining underflow.

**Root Cause**  
The V4 treasury-array setter path dropped the V3 total-fee validation and never enforces MAX_FEES across configured TreasuryConfig entries.

**Pre_conditions**  
An admin configures treasury entries whose total fee exceeds the intended cap, or exceeds 100% of paymentAmount.

**Impact**  
Accepted over-cap fees can route more of each exercise payment to treasuries than the declared maximum. If total configured fees exceed paymentAmount, regular and LP exercise paths become unavailable until the treasury list is repaired. Atomic reverts prevent partial token loss.

**Mitigation**  
Track the sum of treasury fees on add/replace/remove and require it to stay at or below MAX_FEES. Also reject per-entry values above MAX_FEES or 100 as appropriate.

### [I-52] Permissionless GaugeFactoryV4 creation grants oFlow minter role to unregistered gauges

**Severity**: Info  
**Location**: v4-contracts/contracts/factories/GaugeFactoryV4.sol:20-24; v4-contracts/contracts/Voter.sol:323-359; v4-contracts/contracts/GaugeV4.sol:94-104,269-285; v4-contracts/contracts/OptionTokenV4.sol:190-194,519-523

**Summary/Description**  
GaugeFactoryV4.createGauge is externally callable and has no caller restriction. When oFlow is configured, any caller can create a GaugeV4 with voter bound to that caller and cause the factory to grant MINTER_ROLE on oFlow to the new, unregistered gauge. The normal Voter.createGauge path performs pair/factory/gauge-creation checks before calling the factory, but those checks are bypassed by direct factory calls. OptionTokenV4.mint still pulls underlying FLOW from the gauge before minting, so this does not create unbacked oFlow by itself; the issue is an unintended expansion of the oFlow minter role surface outside the registered Voter lifecycle.

**Root Cause**  
GaugeFactoryV4.createGauge trusts any external caller and performs the oFlow MINTER_ROLE grant inside that unrestricted path instead of restricting calls to Voter or validating registration.

**Pre_conditions**  
1. GaugeFactoryV4.oFlow is set to an OptionTokenV4 contract. 2. The GaugeFactoryV4 contract has ADMIN_ROLE on that oFlow, as in the deployment script. 3. An untrusted caller calls GaugeFactoryV4.createGauge directly instead of through Voter.createGauge.

**Impact**  
Untrusted callers can create arbitrary, unregistered GaugeV4 contracts that hold MINTER_ROLE on oFlow. Minting remains collateralized because OptionTokenV4.mint transfers underlying from the gauge before minting, so no direct unbacked mint was found, but role management and lifecycle assumptions are weakened.

**Mitigation**  
Restrict GaugeFactoryV4.createGauge to the Voter contract or another intended factory caller. Alternatively move minter-role grants into a privileged registration path and validate the gauge is registered before granting roles.

