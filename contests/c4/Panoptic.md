### [H-01] BuilderWallet admin can be reset by anyone

**Severity**: High  
**Likelihood**: High  
**Impact**: High  
**Location**: contracts/RiskEngine.sol:2315

**Summary/Description**  
BuilderWallet.init is external and writes builderAdmin without checking msg.sender, FACTORY, prior initialization, or a nonzero initial state. BuilderFactory.deployBuilder calls init after CREATE2 deployment, but any later caller can call init again and replace builderAdmin. Because sweep only checks msg.sender == builderAdmin, the caller can then sweep the wallet token balances.

**Root Cause**  
The wallet keeps one-time admin initialization in a public storage setter instead of restricting it to FACTORY and refusing reinitialization once builderAdmin is set.

**Pre_conditions**  
A BuilderWallet has been deployed through BuilderFactory and has received ERC20 balances such as collateral tracker shares from the builder commission path.

**Impact**  
Any external account can take over any deployed builder wallet, sweep all ERC20 balances from it, and repeat the takeover over time. Builder commission shares are ERC20 collateral tracker shares and can be transferred or redeemed by an account with no open positions, so this is direct loss of accrued builder fee value.

**Mitigation**  
Make init callable only by FACTORY, require builderAdmin == address(0) before setting it, and reject address(0) as the admin. Alternatively pass builderAdmin into the constructor or immutable clone args so there is no post-deployment public initializer.

### [H-02] Liquidation refund reentry can transfer delegated collateral shares

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/PanopticPool.sol:1515-1590; contracts/PanopticPool.sol:1330-1336; contracts/CollateralTracker.sol:422-430; contracts/CollateralTracker.sol:1221-1255; contracts/CollateralTracker.sol:1262-1360; contracts/libraries/SafeTransferLib.sol:19-29

**Summary/Description**  
During liquidation, PanopticPool delegates virtual shares in both CollateralTrackers, burns all positions, and clears the liquidatee position hash before settling the two collateral trackers. settleLiquidation on token0 can make a full-gas ETH refund before token1 has removed its delegated shares. A self-liquidating contract, or a colluding liquidator with share allowance, can reenter token1 CollateralTracker.transfer/transferFrom while numberOfLegs(liquidatee) is already zero and move the temporary delegated shares out before token1 settlement revokes them.

**Root Cause**  
Liquidation settlement performs an external ETH refund from the first CollateralTracker before all delegated-share cleanup has completed. The transferable-share gate only checks PanopticPool.numberOfLegs(from), which is cleared during _burnOptions before settleLiquidation removes the second tracker delegation.

**Pre_conditions**  
The liquidatee is being liquidated and token0 settleLiquidation reaches an ETH-refund path, either because bonus0 is nonnegative and msg.value was attached, or because a native negative-bonus settlement overpays and refunds surplus. The liquidatee is the liquidator contract or has approved the reentering liquidator to move token1 CollateralTracker shares.

**Impact**  
The reentrant transfer can move approximately 2^248 temporary shares plus any real shares to an attacker-controlled account. When token1 settleLiquidation resumes, it restores internal supply against the now-low liquidatee balance, leaving the transferred temporary shares as real withdrawable shares. The recipient has no open positions and can redeem most or all token1 deposited assets, causing vault-wide LP fund loss.

**Proof of Concept**  
Not run per instruction; code-level path verified from the liquidation and CollateralTracker line ranges.

**Mitigation**  
Do not perform external refunds until both CollateralTrackers have revoked delegated shares. Add a PanopticPool-level liquidation/settlement reentrancy guard, or mark accounts under delegated settlement so CollateralTracker transfer/transferFrom and withdrawal paths reject them until all delegations are revoked.

### [M-01] Self-dispatchFrom can consume delegated shares or capture liquidation bonuses

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/PanopticPool.sol:1360-1475, contracts/PanopticPool.sol:1482-1590, contracts/PanopticPool.sol:1598-1661, contracts/PanopticPool.sol:1671-1702, contracts/CollateralTracker.sol:1221-1255, contracts/CollateralTracker.sol:1262-1360, contracts/CollateralTracker.sol:1369-1380, contracts/CollateralTracker.sol:1395-1500

**Summary/Description**  
dispatchFrom does not reject account == msg.sender. In the equal-list branch, _settlePremium delegates virtual shares to the owner, settles owed long premium, then refunds owner to msg.sender; in the one-position-short branch, _forceExercise delegates virtual shares before _burnOptions calls settleBurn. In the liquidation branch, an insolvent caller can set account to itself, pass an empty positionIdListFrom for the post-liquidation zero-position check, and receive the liquidation settlement as both liquidator and liquidatee. When the caller is also the account, CollateralTracker.refund becomes a self-transfer in settlement paths, and positive liquidation bonus settlement transfers existing shares to the same address or mints any shortfall to that same address.

**Root Cause**  
The delegated-share settlement and liquidation-bonus flows assume the caller is a distinct third party, but dispatchFrom permits the target account to be msg.sender and has no self-target guard.

**Pre_conditions**  
For settlement/force exercise, the account has a position eligible for dispatchFrom settlement or force exercise, owes premium or other positive settleBurn tokenToPay in one collateral token, and lacks enough real shares to pay the full amount while passing relevant checks. For self-liquidation, the account is insolvent at all dispatchFrom liquidation ticks, supplies its full current positionIdListTo and empty positionIdListToFinal, and uses an empty positionIdListFrom after the liquidation removes all positions.

**Impact**  
Owed premium or burn-settlement value can be credited into PanopticPool/CollateralTracker accounting without being paid from real user collateral in the self-settlement/force-exercise branches. In the liquidation branch, an insolvent account can capture the liquidation bonus that should compensate an external liquidator; if the bonus exceeds remaining real shares, settleLiquidation mints the shortfall to the same account, increasing PLP/protocol loss instead of imposing liquidation penalty on the liquidatee.

**Mitigation**  
Reject account == msg.sender in dispatchFrom branches that rely on delegation, refund, or liquidation-bonus settlement, or require a distinct third-party payer/recipient for those flows. At minimum, self-liquidation should not pay the liquidation bonus to the liquidatee and self-refund paths should not allow delegated shares to satisfy real settlement obligations.

### [M-02] Builder-code commission path leaves part of the fee uncollected

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Medium  
**Location**: contracts/CollateralTracker.sol:1553-1578, contracts/CollateralTracker.sol:1612-1657, contracts/RiskEngine.sol:118-124, README.md:461-462

**Summary/Description**  
When a builder code is present, settleMint and settleBurn compute a full commission fee but transfer only protocolSplit + builderSplit shares to the protocol and builder. The configured splits are 6,500 and 2,500 bps, so 10% of the fee remains with the option owner instead of being burned/transferred. The no-builder path burns the full sharesToBurn amount, so builder-code users receive an undocumented fee discount and fee recipients/PLPs lose expected commission value. The emitted builder amount also uses protocolSplit instead of builderSplit, making the accounting event inconsistent with the actual transfer.

**Root Cause**  
The builder-recipient branch splits sharesToBurn by protocolSplit and builderSplit but does not handle any residual DECIMALS - protocolSplit - builderSplit amount, and the CommissionPaid event uses protocolSplit for both protocol and builder fields.

**Pre_conditions**  
A user opens or closes a position with a nonzero builder code that maps to a feeRecipient. The configured protocolSplit plus builderSplit is less than DECIMALS, as in the current constants 6,500 + 2,500 < 10,000.

**Impact**  
Each affected commission collects only 90% of the computed fee under current parameters. The missing 10% of commission remains with the trader, reducing protocol/builder/PLP fee value and making builder-code settlement economically inconsistent with the no-builder path. The loss is repeatable across builder-code mints and, if premium fees are enabled, burns.

**Mitigation**  
After transferring protocol and builder shares, burn or otherwise allocate the residual sharesToBurn - protocolShares - builderShares according to the intended fee policy. Emit the builder commission amount using builderSplit rather than protocolSplit.

### [M-03] Empty final dispatch can clear positions while leaving accrued interest uncollectible

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/PanopticPool.sol:694; contracts/PanopticPool.sol:967; contracts/PanopticPool.sol:1829; contracts/PanopticPool.sol:1859; contracts/CollateralTracker.sol:916; contracts/CollateralTracker.sol:1514; contracts/CollateralTracker.sol:1065

**Summary/Description**  
PanopticPool.dispatch performs its final solvency validation only after all requested position changes. The final list is fingerprint-checked, so non-empty incomplete lists should fail, but when the correct final list is empty _validateSolvency skips _checkSolvencyAtTicks entirely. If a borrower cannot pay all accrued interest, CollateralTracker._accrueInterest burns only available shares and preserves the stale borrow index; the last position close can then reduce netBorrows to zero, making _owedInterest return zero and leaving the unpaid interest unattributed.

**Root Cause**  
The final dispatch gate treats a validated empty position list as automatically safe, while CollateralTracker only tracks borrower interest while netBorrows is positive. A last-position close can zero the borrow base after only partial interest payment.

**Pre_conditions**  
The account has accrued interest greater than its collateral balance in at least one collateral tracker; it closes its last positive-borrow position through dispatch; and the close path does not require additional real shares after the partial interest burn.

**Impact**  
Unpaid interest can become uncollectible while remaining reflected in global collateral accounting, shifting loss into PLP/accounting state and allowing an account to exit its last position without a final economic solvency check.

**Mitigation**  
Do not skip the final economic check for an empty final position list. Require accrued interest and positive netBorrows to be fully settled before the last position can be removed, or track any unpaid interest independently of open-position netBorrows.

### [M-04] Cross-buffered insolvent accounts with excess aggregate collateral cannot be liquidated

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/RiskEngine.sol:510-516, contracts/RiskEngine.sol:1029-1051, contracts/RiskEngine.sol:2106-2149, contracts/PanopticPool.sol:1399-1407, contracts/PanopticPool.sol:1539-1546

**Summary/Description**  
Liquidation eligibility uses isAccountSolvent, which only allows cross-token surplus after applying the utilization-dependent cross buffer. getLiquidationBonus then assumes the full cross-token aggregate balance is below the aggregate requirement and computes thresholdCross - balanceCross. At high utilization the cross buffer can be zero, so an account can be insolvent because one token requirement is not met while still having excess full-value collateral in the other token. In that state getLiquidationBonus underflows before settlement, blocking liquidation.

**Root Cause**  
The liquidation bonus formula uses full-value aggregate cross balances, while the liquidation predicate uses cross-buffered per-token solvency. No code ensures thresholdCross >= balanceCross under the predicate actually used to enter liquidation.

**Pre_conditions**  
A position records high enough utilization for the relevant cross buffer to be reduced or zero. The account has a deficit in one collateral token and a surplus in the other token large enough that full-value aggregate balance exceeds aggregate requirements, but not usable under the cross-buffered solvency check.

**Impact**  
Liquidation of a margin-called account can revert before bonus settlement. The distressed position can remain open even though the protocol says it is liquidatable, allowing risk to persist and potentially turn into PLP/protocol loss if prices or premiums move further before collateral is rebalanced.

**Mitigation**  
Compute liquidation bonus from the same cross-buffered solvency basis used by isAccountSolvent, or clamp aggregate deficiency at zero and handle cross-buffer-only insolvency with a bounded bonus that cannot underflow.

### [M-05] Force exercise prices all long legs using a single in-range flag

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/RiskEngine.sol:407-435, contracts/RiskEngine.sol:475-484, contracts/PanopticPool.sol:1413-1435, contracts/PanopticPool.sol:1595-1645, contracts/types/TokenId.sol:517-528, contracts/libraries/Math.sol:367-378, README.md:429-434

**Summary/Description**  
exerciseCost computes each long leg's tick range as lower-inclusive/upper-exclusive, but it still aggregates all non-credit long notional under a single position-level in-range flag and PanopticPool force-exercise eligibility only checks that some non-credit long leg exists. At a lower-boundary tick, the leg is treated as in range and can still be force-exercised for the high flat fee; at an upper-boundary tick it is treated as out of range and can switch the entire aggregate to the 1 bps rate. A position with a small in-range long leg and a large out-of-range long leg is charged the in-range 1.024% rate on all long notional instead of the 1 bps out-of-range rate for the out-of-range leg, while fully in-range long positions can be force exercised at the high fee.

**Root Cause**  
Range status is tracked as one position-level boolean instead of per long leg, and validateIsExercisable does not enforce that at least one long leg is out of range before the position can be force exercised.

**Pre_conditions**  
A user holds a valid multi-leg position with at least one non-credit long leg. At force-exercise time, at least one long leg is in range while another long leg is out of range, or all long legs are in range and the caller is willing to pay the flat in-range fee.

**Impact**  
Out-of-range liquidity can be made uneconomic to force exercise by pairing it with an in-range long leg, delaying sellers from recovering liquidity that is no longer generating premia. Separately, in-range long positions can be closed against the owner for a fixed fee despite docs describing force exercise as targeting out-of-range long positions.

**Mitigation**  
Check exercisability per long leg against the oracle/current range and require at least one out-of-range long leg. Compute the base force-exercise fee per leg so in-range and out-of-range long notional are charged at their respective rates.

### [M-06] Collateral withdrawals ignore active safe mode

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/CollateralTracker.sol:744-770, contracts/PanopticPool.sol:409-420, contracts/PanopticPool.sol:694-700, contracts/PanopticPool.sol:1740-1750, contracts/RiskEngine.sol:908-939

**Summary/Description**  
When dispatch validates solvency it forwards the computed RiskEngine safeMode value, causing _checkSolvencyAtTicks to force 100% utilization and disable normal cross-margin assumptions during safe mode. The CollateralTracker withdrawal overload for accounts with open positions burns shares and calls PanopticPool.validateCollateralWithdrawable, but that function computes risk parameters and then hardcodes safeMode to 0. As a result, even if RiskEngine.isSafeMode is nonzero because of an oracle shock or guardian lock, open-position withdrawals are checked under normal margin rules.

**Root Cause**  
validateCollateralWithdrawable retrieves riskParameters but discards riskParameters.safeMode() when forwarding to _validateSolvency, unlike dispatch which forwards the computed safe-mode value.

**Pre_conditions**  
The account has open positions and collateral that is withdrawable under normal margin/cross-collateral rules but would fail under safe-mode 100% utilization/no-cross-margin rules. Safe mode is active due to oracle deviation or guardian lock.

**Impact**  
During oracle stress or a guardian safe-mode lock, an account with open positions can remove collateral using normal margin assumptions even though the same post-action state would fail the intended safe-mode basis. This is not only a local missed flag: safe mode is the mechanism that makes _checkSolvencyAtTicks use 100% utilization and no normal cross-margining before collateral leaves the system. If the stressed state persists or liquidation follows, the withdrawn collateral is no longer available to absorb the account deficit, so the shortfall can be realized through liquidation settlement and shifted to LP/protocol accounting.

**Mitigation**  
Pass riskParameters.safeMode() into _validateSolvency from validateCollateralWithdrawable, matching the dispatch path, or otherwise disable open-position withdrawals while safe mode is nonzero.

### [M-07] Same-transaction oracle updates can be consumed by later actions

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/types/OraclePack.sol:536-565, contracts/types/OraclePack.sol:578-600, contracts/PanopticPool.sol:551-558, contracts/PanopticPool.sol:572-702, contracts/PanopticPool.sol:1368-1398, contracts/PanopticPool.sol:1944-1945, contracts/base/Multicall.sol:12-16

**Summary/Description**  
OraclePack.getOracleTicks returns pre-update oracle ticks while also returning a post-update OraclePack when a new 64-second epoch is reached. PanopticPool.pokeOracle and dispatch can persist that post-update pack, and Multicall or an external transaction orchestrator can execute a later PanopticPool action in the same transaction. That later action reads the updated EMAs/latest observation through getTWAP, getRiskParameters, or getSolvencyTicks, so operations in one transaction can be validated against different oracle snapshots.

**Root Cause**  
The oracle helper combines pre-update tick outputs with a post-update OraclePack return value, and PanopticPool stores the returned pack immediately without a transaction/block boundary that prevents later same-transaction actions from consuming the newly inserted current-tick observation and EMA update.

**Pre_conditions**  
At least one new 64-second epoch has elapsed since the stored oracle update. The caller can move the underlying pool tick for the transaction, then call pokeOracle or a dispatch path that stores the updated OraclePack, and then execute another PanopticPool action such as dispatchFrom in the same transaction.

**Impact**  
A same-transaction current-tick move can be incorporated into the stored EMA/latest-observation state before a later action evaluates risk. For dispatchFrom, getTWAP is read before the liquidation staleness guard, so a prior same-transaction oracle update can move the TWAP toward the manipulated current tick and allow liquidation, force-exercise, or settlement flows to proceed under an oracle view that was not fixed for the transaction. This weakens the documented invariant that all operations in a single transaction use consistent oracle ticks and can expose accounts to unfair liquidation or force-exercise decisions around the oracle thresholds.

**Mitigation**  
Make state-changing actions use a transaction-stable oracle snapshot, or defer stored OraclePack updates so they are not consumable until a later block/transaction. Alternatively, update and validate against the same post-update snapshot consistently, and make pokeOracle/update sampling use an elapsed-period TWAP rather than the transaction's instantaneous current tick.

### [M-08] Keep-open short premium settlement understates the gross-premium baseline

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/PanopticPool.sol:646-649, contracts/PanopticPool.sol:988-1012, contracts/PanopticPool.sol:1187-1320

**Summary/Description**  
The dispatch equality branch lets a short seller settle premium while keeping the position open. _updateSettlementPostBurn then updates the seller personal premium snapshot to the current accumulator, but recomputes s_grossPremiumLast with the burn/removal formula even though the seller liquidity remains in totalLiquidity. In the settlePremium to burn ordering, the first keep-open settlement removes the current accumulator contribution from the global baseline without actually removing liquidity, and the later burn removes that contribution again. Repeating the keep-open settlement path, including through PanopticPool.multicall, can re-enter the same short-leg baseline update even after s_options has been advanced and no new premium is owed, further reducing the baseline without a corresponding premium payment or liquidity removal. The burn to settlePremium ordering does not have this mismatch because the full burn uses the removal denominator and then deletes the position.

**Root Cause**  
The keep-open settlement path reuses the burn grossPremiumLast formula. For an open settlement, the seller's weighted snapshot should be replaced from old accumulator to current accumulator; the current formula effectively removes the old contribution without re-adding the current one, so a later burn can double-apply the removal term.

**Pre_conditions**  
At least two short sellers share a chunk with settled premium available. One seller calls dispatch with positionSizes[i] equal to its stored size, triggering the settle branch while keeping the position open.

**Impact**  
After one seller self-settles, _getAvailablePremium computes future availability against an inflated aggregate owed amount. Repeated keep-open settlement can amplify this because a zero-new-premium repeat still executes the short-leg s_grossPremiumLast update, so other sellers can receive less than their pro-rata share of already-settled premium and residual s_settledTokens for the chunk can remain unavailable until later accounting changes. The action is permissionless for any seller in the shared chunk and can be batched through multicall.

**Mitigation**  
Use a separate keep-open settlement update for s_grossPremiumLast that replaces the settling seller's old weighted snapshot with the current accumulator, or skip global burn-style adjustment unless liquidity is actually removed.

### [M-09] Liquidation classification uses inconsistent safe-mode and TWAP ticks

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/PanopticPool.sol:1368-1407; contracts/PanopticPool.sol:1740-1750; contracts/RiskEngine.sol:828-839; contracts/RiskEngine.sol:947-980; contracts/types/OraclePack.sol:202-225

**Summary/Description**  
dispatchFrom classifies liquidation, force exercise, and premium settlement eligibility with a hand-built tick set and hardcoded safeMode=0. It also reads getTWAP() from stored s_oraclePack before getOracleTicks and ignores the post-update OraclePack returned when a new 64-second epoch is eligible, so liquidate-before-pokeOracle can use stale TWAP, spot, and latest health inputs while pokeOracle-before-liquidate uses updated EMAs and latest observation. In addition, twapEMA destructures OraclePack.getEMAs() out of order, so the liquidation TWAP uses actual slow, fast, and spot weights instead of the documented fast, slow, and eons weights.

**Root Cause**  
The liquidation and force-exercise classifier does not reuse getSolvencyTicks, does not persist or consistently consume the fresh OraclePack returned by getOracleTicks, does not pass riskParameters.safeMode() into _checkSolvencyAtTicks, and the custom TWAP helper binds OraclePack EMA return values in the wrong order.

**Pre_conditions**  
At least one new 64-second oracle epoch is eligible, or safe mode is nonzero because of oracle divergence or lock mode. The target account health differs between the stale stored oracle pack and the pack that pokeOracle would persist, or between normal utilization/TWAP classification and safe-mode/median-tick classification.

**Impact**  
Liquidation and forced-action eligibility become order-dependent at the oracle epoch frontier. A caller can run dispatchFrom before pokeOracle and classify the target with stale TWAP, spot, and latest observation data even though a same-block pokeOracle would update the pack first and could make the account solvent at one or more checked ticks. The reverse order can also block or reroute a liquidation that succeeds under the stale pack. This exposes accounts to premature liquidation or delayed liquidation around stale-health thresholds, while the safe-mode and TWAP tuple mismatches further change liquidation bonus, exercise, and refund pricing from the intended oracle basis.

**Mitigation**  
Use the same tick source and safe-mode flag for dispatchFrom classification as the general solvency path, fix twapEMA destructuring to use spot/fast/slow/eons in the documented order, and ensure liquidation bonus/refund pricing uses the intended oracle tick.

### [M-10] Force exercise closes embedded short legs without short-leg compensation

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/PanopticPool.sol:1431-1435; contracts/PanopticPool.sol:1598-1645; contracts/RiskEngine.sol:409-423; contracts/libraries/PanopticMath.sol:453-469; contracts/types/TokenId.sol:523-528

**Summary/Description**  
A tokenId only needs one non-credit long leg to enter the forceExercise branch, but _forceExercise burns the entire tokenId. If that tokenId also contains short legs, those short legs are closed even though exerciseCost explicitly skips short legs and therefore charges no inconvenience fee for their forced closure.

**Root Cause**  
Force-exercise eligibility is checked at the tokenId level using existence of any long leg, while execution burns all legs in the tokenId and the fee calculation is long-leg-only.

**Pre_conditions**  
The target account is solvent at the dispatchFrom checked ticks and owns a multi-leg tokenId containing at least one nonzero-width long leg plus one or more short legs.

**Impact**  
A third party can forcibly close the target's short option exposure and its future premium stream as a side effect of exercising a long leg, without paying compensation based on the short legs. This breaks the expected long-leg-only force-exercise boundary and can disrupt valid multi-leg strategies.

**Mitigation**  
Reject force exercise for tokenIds containing short legs, implement partial long-leg exercise, or include explicit compensation and validation for any short legs that will be closed.

### [M-11] Global unrealized interest accrues on gross AMM assets instead of users' net borrow base

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/CollateralTracker.sol:503; contracts/CollateralTracker.sol:887; contracts/CollateralTracker.sol:1013; contracts/CollateralTracker.sol:1065; contracts/CollateralTracker.sol:1412; contracts/CollateralTracker.sol:1508; contracts/PanopticPool.sol:787; contracts/libraries/PanopticMath.sol:453

**Summary/Description**  
CollateralTracker grows unrealizedGlobalInterest from s_assetsInAMM, which is increased by gross short amounts, but individual owed interest is computed only for users with positive netBorrows. A valid mixed long/short position can leave netBorrows at zero or below while still increasing s_assetsInAMM, causing totalAssets() to include interest that no account owes.

**Root Cause**  
The global interest accumulator uses gross assets deployed in the AMM as its principal, while per-account interest settlement uses netBorrows = shortAmount - longAmount and skips users whose netBorrows are non-positive.

**Pre_conditions**  
A valid position has shortAmount for a collateral token and enough longAmount for the same collateral token to reduce the owner's netBorrows below the gross short amount; time passes and _accrueInterest is called.

**Impact**  
This is not only a local accumulator mismatch. totalAssets() includes unrealizedGlobalInterest, and totalAssets() is used for share conversion, deposit, withdrawal, redemption, liquidation settlement, and utilization calculations. When the accumulator includes interest that no positive-net-borrow account owes, the vault can reflect phantom yield in its share price; early redeemers or settlement flows can consume real assets against that inflated accounting, leaving remaining LPs and protocol accounting to absorb the deficit.

**Mitigation**  
Accrue global unrealized interest from the same aggregate positive net borrow base used for user-level interest, or track per-account gross borrow obligations consistently and ensure every unit added to unrealizedGlobalInterest is attributable to a collectible user debt.

### [M-12] Borrow index can exceed its 80-bit packed field

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/CollateralTracker.sol:985; contracts/CollateralTracker.sol:1009; contracts/CollateralTracker.sol:1021; contracts/CollateralTracker.sol:970; contracts/types/MarketState.sol:59; contracts/types/MarketState.sol:155

**Summary/Description**  
CollateralTracker compounds the global borrow index as a uint128, but MarketState reserves only 80 bits for the packed borrowIndex field. Once sustained high-rate compounding pushes the index above 2^80-1, storeMarketState adds the oversized value without masking or rejecting it, so high index bits spill into adjacent packed fields while borrowIndex() later returns only the low 80 bits.

**Root Cause**  
The borrow index type used during accrual is wider than the packed storage field, and MarketStateLibrary.storeMarketState does not validate that _borrowIndex fits in 80 bits before combining it with marketEpoch, rateAtTarget, and unrealizedInterest.

**Pre_conditions**  
High utilization keeps the borrow rate near the maximum curve rate and the market is accrued frequently enough for compounding to grow the index over the 80-bit boundary. With the current constants, 4-second accruals at the 800% maximum curve rate cross 2^80 after about 55,208,668 seconds, or 638.99 days.

**Impact**  
After the boundary is crossed, the global borrow index wraps when read from MarketState and the overflow bit increments/corrupts adjacent packed state. Users whose stored borrow-index snapshots exceed the wrapped global index can revert on interest calculation, while other users can be charged from a corrupted lower index. This can halt or distort deposits, withdrawals, position settlement, liquidations, and interest distribution for the affected collateral market.

**Mitigation**  
Store the borrow index in a field wide enough for the uint128 accumulator, or explicitly revert/saturate before packing if currentBorrowIndex exceeds type(uint80).max. If the 80-bit field is intentional, add an operational cap/migration path before the boundary is reachable.

### [M-13] Long credited shares are not cleared after share-price changes

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/CollateralTracker.sol:511-528, contracts/CollateralTracker.sol:795-872, contracts/CollateralTracker.sol:1403-1467, contracts/CollateralTracker.sol:1498-1514

**Summary/Description**  
Long-position open and close convert the same longAmount at different share prices. Opening adds ceil(longAmount * totalSupply / totalAssets) to s_creditedShares, while closing subtracts floor(longAmount * currentTotalSupply / currentTotalAssets). If share price changes while the long is open, closing can leave residual unowned credited shares or charge the owner more than a rounding haircut. The close path is also per-tokenId: a single tokenId close computes one floor over the tokenId's aggregate longAmount, while split closes across tokenIds compute sequential floors after each state update, so split closures can leave strictly more creditedShares than an economically equivalent aggregate close. Share donation is an explicit trigger: donate burns internal supply while leaving totalAssets and s_creditedShares unchanged, and repeated donations can amplify the price movement beyond the initial per-call maxRedeem amount because s_depositedAssets is not reduced by donation.

**Root Cause**  
s_creditedShares stores a share-denominated credit but is settled using the current global share price instead of tracking and clearing the per-position credited share amount created at open. The close path treats all excess close-side creditDelta as rounding, even when the delta is caused by real share-price movement.

**Pre_conditions**  
A long position is opened, s_creditedShares is increased, and then the CollateralTracker share price changes before that long is closed. Share-price changes are reachable through normal protocol flows such as commission burns or yield/loss accounting, and also through share donation. For donation, the donor must have no open positions, but repeated calls can keep recomputing maxRedeem from the unchanged s_depositedAssets bucket and burn substantially more shares than a single max-sized donation when s_depositedAssets is much smaller than totalAssets.

**Impact**  
After a positive share-price change, a full long close can leave s_creditedShares positive even if no long position remains, so totalSupply includes unowned shares and real LP share value is diluted. After a negative share-price change, the close path can charge the option owner a non-rounding haircut. The effect is shared accounting corruption and claimable value loss, not merely one-wei Uniswap rounding.

**Mitigation**  
Track credited shares per long position or per account at open and clear that recorded amount at close, with a separate explicit mechanism for protocol-owned yield or losses if intended. Do not classify share-price movement deltas as rounding haircuts.

### [M-14] Public known: capped unpaid interest misreports account deficits

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/RiskEngine.sol:495-607, contracts/RiskEngine.sol:996-1051, contracts/RiskEngine.sol:1151-1189, contracts/CollateralTracker.sol:1083-1105, contracts/libraries/PanopticMath.sol:663-689, README.md:109-117

**Summary/Description**  
RiskEngine._getMargin caps interest owed to the account balance when interest exceeds balance, then adds only that capped amount to requirements and zeros the balance. The correct remaining interest shortfall after consuming the balance is interest - balance, but the code reports balance. This can understate the deficit when interest > 2 * balance, overstate it when balance < interest < 2 * balance, and feed the masked amount into isAccountSolvent and liquidation bonus calculations. README already lists the same Nethermind pre-contest Masking Insolvency Magnitude and Broken Bonus Calculations issues.

**Root Cause**  
The margin input builder replaces the true unpaid interest amount with min(interest, balance) before the solvency predicate, losing deficit magnitude.

**Pre_conditions**  
A user owes interest in one collateral tracker greater than their token balance, and has enough other counted collateral or lower other requirements to satisfy the masked deficit but not the true unpaid interest.

**Impact**  
Solvency checks and liquidation bonus/protocol-loss calculations can use a deficit that is not the actual unpaid-interest shortfall. Depending on the interest/balance ratio, this can delay liquidation, allow collateral-affecting actions under an understated requirement, or overstate liquidation pressure/bonus inputs. This is public/known from README, so it is inventory rather than a new contest submission unless contest rules allow it.

**Mitigation**  
Carry the full interest owed and the available balance separately. After consuming balance toward interest, add max(interest - balance, 0) to requirements rather than min(interest, balance), and feed the same true deficit into liquidation bonus/protocol-loss calculations.

### [M-15] Temporary deposits can lock in low utilization requirements for undercollateralized positions

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Medium  
**Location**: contracts/PanopticPool.sol:745-763, contracts/PanopticPool.sol:1728-1764, contracts/CollateralTracker.sol:744-770, contracts/CollateralTracker.sol:1133-1153, contracts/RiskEngine.sol:1192-1296, contracts/RiskEngine.sol:2043-2097

**Summary/Description**  
PositionBalance stores the pool utilizations returned at mint and RiskEngine later uses the maximum stored utilization across the user's positions for collateral ratios and cross buffers. Open-position withdrawals burn shares and validate solvency against those same stored utilization fields, without recomputing current utilization after the withdrawal. A user can temporarily deposit assets to lower utilization at mint, open positions with low collateral requirements, then withdraw the temporary assets while the position keeps the stale low-utilization requirement.

**Root Cause**  
Utilization is a one-time PositionBalance snapshot used as an ongoing risk input, while withdrawals with open positions do not refresh or cap the stored utilization against current post-withdrawal utilization.

**Pre_conditions**  
A collateral vault's current utilization is materially higher than the utilization obtainable by a temporary deposit. The user can supply temporary liquidity, mint a position, and then withdraw collateral while satisfying the stale at-mint utilization requirement.

**Impact**  
The account can retain positions collateralized for an artificially low-utilization market after the temporary liquidity is removed. This weakens the utilization-scaled seller requirement and cross-buffer logic, can delay or prevent liquidation under the intended current-risk basis, and can shift losses to remaining PLPs if the position moves against the account.

**Proof of Concept**  
Not constructed; this was code-level review only.

**Mitigation**  
For solvency and withdrawals, use the maximum of stored at-mint utilization and current post-action utilization, or otherwise prevent immediate withdrawal of liquidity used to lower utilization for a mint. The transient utilization snapshot should be initialized before deposit-influenced mint paths or replaced with a time-weighted/utilization floor for margin.

### [M-16] Dispatch reuses pre-action safeMode after price-changing operations

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/PanopticPool.sol:592-700, contracts/PanopticPool.sol:1719-1750, contracts/PanopticPool.sol:1808-1812, contracts/RiskEngine.sol:864-940, contracts/SemiFungiblePositionManager.sol:620-651

**Summary/Description**  
PanopticPool.dispatch computes RiskParameters once before executing all mint/burn/settlement actions. Those actions can change the Uniswap pool tick and return a finalTick, and dispatch only enforces a cumulative tick movement bound after the actions. The final _validateSolvency call recomputes currentTick/getSolvencyTicks, but it still receives riskParameters.safeMode() from the pre-action tick. Therefore, a dispatch that starts outside safe mode can end at a tick that would make RiskEngine.isSafeMode nonzero, while the final solvency check does not force 100% utilization/no-cross-margin safe-mode requirements. If one safe-mode condition was already active at the start, the same stale value can also miss a post-action escalation above the covered-position threshold.

**Root Cause**  
RiskParameters, including safeMode, are captured before price-changing SFPM calls and are not recomputed after the final tick is known. The final solvency path recomputes the tick set but reuses the stale safeMode flag.

**Pre_conditions**  
A dispatch action moves the pool tick enough that RiskEngine.isSafeMode(final currentTick, s_oraclePack) would be higher than the mode computed at dispatch entry, while remaining within the dispatch price-impact and user tick-limit checks.

**Impact**  
A dispatch can start outside safe mode, perform price-changing SFPM actions, and end at a market/oracle state that would require safe-mode collateralization, while the final solvency check still uses the pre-action safeMode value. The consequence is broader than a stale local parameter: the accepted final position set can be undercollateralized under the state that actually exists after the transaction. Later validation or liquidation may observe the fresh safe mode only after the account has already opened or kept exposure that should have been rejected or constrained, increasing bad-debt and LP/protocol-loss risk during the stressed window.

**Mitigation**  
Recompute RiskParameters or at least safeMode after the price-changing loop, before the final _validateSolvency call, and use the fresh safeMode for both the final solvency check and any post-action restrictions that depend on the final currentTick. Alternatively make the initial safeMode check conservatively account for the allowed post-action tick movement.

### [M-17] Safe mode omits the documented fast-slow EMA divergence check

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/RiskEngine.sol:70-72, contracts/RiskEngine.sol:903-929, README.md:444-449, contracts/PanopticPool.sol:622-631, contracts/PanopticPool.sol:1740-1750

**Summary/Description**  
RiskEngine documents safe mode as guarding fast-vs-slow oracle divergence, and the README states that the maximum allowed delta between fast and slow oracle ticks is 953 ticks. The implementation does not compare fastEMA to slowEMA. It compares currentTick to spotEMA, spotEMA to fastEMA, and medianTick to slowEMA. Therefore a state with currentTick == spotEMA == fastEMA and medianTick near fastEMA can return safeMode 0 even when abs(fastEMA - slowEMA) exceeds MAX_TICKS_DELTA.

**Root Cause**  
The internal-disagreement condition is implemented as spotEMA-fastEMA instead of fastEMA-slowEMA, and the high-divergence condition uses a separate median-slow threshold of 2 * MAX_TICKS_DELTA rather than enforcing the documented fast-slow bound.

**Pre_conditions**  
The oracle pack reaches a state where fastEMA has moved more than MAX_TICKS_DELTA away from slowEMA while spotEMA/currentTick remain near fastEMA and medianTick is within 2 * MAX_TICKS_DELTA of slowEMA or has moved near fastEMA.

**Impact**  
The protocol can continue with safeMode 0 even though the documented fast-slow EMA divergence threshold has been exceeded. This is not merely a documentation mismatch: safeMode controls both dispatch-time mint restrictions and the solvency path that forces 100% utilization/no normal cross-margining. In the missed divergence state, accounts can open, modify, or keep positions under normal margin while the documented oracle-stress invariant says the conservative basis should apply; if the price/oracle stress persists, later liquidation can realize collateral shortfall that the omitted safe-mode trigger was intended to prevent.

**Mitigation**  
Implement an explicit abs(fastEMA - slowEMA) > MAX_TICKS_DELTA safe-mode condition, or update the documented oracle-safety invariant and downstream assumptions to match the actual spot-fast/median-slow checks.

### [M-18] V4 hooks can observe and alter SFPM accounting before settlement is finalized

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/SemiFungiblePositionManagerV4.sol:534-557; contracts/SemiFungiblePositionManagerV4.sol:768-770; contracts/SemiFungiblePositionManagerV4.sol:990-993; contracts/SemiFungiblePositionManagerV4.sol:1023-1050; contracts/SemiFungiblePositionManagerV4.sol:1072-1078; contracts/SemiFungiblePositionManagerV4.sol:870-899; contracts/PanopticPool.sol:1048-1049; lib/v4-core/src/interfaces/IPoolManager.sol:137-141; lib/v4-core/src/PoolManager.sol:154-181

**Summary/Description**  
SFPM V4 accepts hook-enabled PoolKeys and updates local liquidity accounting before PoolManager.modifyLiquidity runs V4 hooks. PoolManager can then execute before/after liquidity hooks and after hooks can alter caller deltas; SFPM correctly nets the final PoolManager transient delta with ERC6909 mint/burn, but it still records raw feesAccrued as collected premium even though the V4 interface warns that this value is informational and can be artificially inflated. PanopticPool later credits collectedByLeg into settled premium accounting, so hook-enabled/donation-manipulated V4 pools can make Panoptic premium state depend on hook-visible, pre-finalized SFPM state and raw fee-growth collection rather than a constrained no-hook AMM model.

**Root Cause**  
The implementation assumes V4 hooks do not reenter dependent Panoptic/SFPM observation paths and do not change the economic meaning of feesAccrued. PoolManager explicitly executes hooks around modifyLiquidity and lets after hooks modify callerDelta, while SFPM has already changed local liquidity state and later reports feesAccrued separately into premium accounting.

**Pre_conditions**  
The Panoptic V4 pool uses a hook with before/after liquidity behavior or fee-growth manipulation, and the pool has existing in-range SFPM liquidity so modifyLiquidity can return nonzero feesAccrued. PanopticFactoryV4 accepts arbitrary initialized V4 PoolKeys and does not restrict hooks.

**Impact**  
PoolManager currency deltas are forced to zero before unlock returns, so the issue is not a leftover PoolManager settlement delta. The impact is that shared premium availability and downstream collateral settlement can be driven by raw feesAccrued and hook-visible intermediate SFPM state. In hook-enabled V4 pools, hooks can observe s_accountLiquidity after SFPM writes it but before PanopticPool finalizes position, premium, collateral, and solvency updates; donation or hook-mediated fee-growth manipulation can also inflate the collected premium reported to PanopticPool and credited into s_settledTokens.

**Mitigation**  
Restrict supported V4 hooks to none or audited zero-delta hooks, or move SFPM state writes until after hook-bearing PoolManager calls and account premium from post-hook caller deltas instead of raw feesAccrued. Add a PanopticPool-level reentrancy guard around settlement paths used during SFPM callbacks.

### [M-19] Unrealized interest can exceed its 106-bit packed field

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/types/MarketState.sol:59; contracts/types/MarketState.sol:65; contracts/types/MarketState.sol:68; contracts/types/MarketState.sol:130; contracts/types/MarketState.sol:140; contracts/types/MarketState.sol:182; contracts/CollateralTracker.sol:233; contracts/CollateralTracker.sol:887; contracts/CollateralTracker.sol:1006; contracts/CollateralTracker.sol:1013; contracts/CollateralTracker.sol:1016; contracts/CollateralTracker.sol:970; contracts/RiskEngine.sol:169

**Summary/Description**  
MarketState reserves only 106 bits for unrealizedInterest, but the production accrual path accumulates it as uint128 and stores it through storeMarketState without a uint106 range check. When accumulated unrealized interest exceeds 2^106-1 but still fits in uint128, shl(150, _unrealizedInterest) drops the high bits and the decoded unrealizedInterest becomes the value modulo 2^106.

**Root Cause**  
The packed field width is narrower than the accumulator type used by CollateralTracker, and storeMarketState neither masks with an explicit error nor reverts when _unrealizedInterest exceeds 106 bits. The safer updateUnrealizedInterest helper masks to 106 bits but is not used by the production _accrueInterest write.

**Pre_conditions**  
A collateral market has high assets in AMM and sustained or delayed high-rate accrual. For example, with assets near the documented per-call deposit cap of type(uint104).max and the documented 800% maximum borrow rate, one high-rate accrual of roughly 78 days produces interest above 2^106 while remaining far below uint128 max.

**Impact**  
Once the value is packed, totalAssets(), utilization, share conversion, and later settlement read a truncated unrealizedGlobalInterest value. This can abruptly remove or distort accrued lender interest and can misprice deposits, withdrawals, redemptions, liquidations, and interest settlement for that collateral market.

**Mitigation**  
Validate _unrealizedInterest <= type(uint106).max before packing and revert or introduce a migration/saturation strategy. Alternatively widen the packed field or keep the production write path using a helper that makes the overflow behavior explicit and acceptable.

### [M-20] Inverted short strangles receive a strangle collateral discount while both legs can be in the money

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Medium  
**Location**: contracts/types/TokenId.sol:473; contracts/RiskEngine.sol:1606; contracts/RiskEngine.sol:1897

**Summary/Description**  
TokenId validation only requires mutual risk partners, while RiskEngine grants the short-strangle discount to any same-asset, same-ratio, both-short option pair with different tokenTypes. It does not require the put strike to be below the call strike, so users can encode an inverted short strangle where both legs can be in the money at the same price and still receive the reduced strangle collateral requirement.

**Root Cause**  
The semantic assumptions for the strangle partner path are enforced in RiskEngine comments/math rather than in TokenId validation or the partner branch predicate. The predicate checks asset, optionRatio, width, tokenType difference, and both-short status, but omits strike ordering/non-overlap for put-call pairs.

**Pre_conditions**  
A user mints a two-leg partnered position with both option legs short, same asset and optionRatio, different tokenTypes, and an inverted strike ordering such that the short call strike is below the short put strike.

**Impact**  
The account is evaluated with the strangle path, which passes negative utilization into the single-leg collateral formula and halves the basal short collateral ratio. In the inverted range both short legs can be in the money simultaneously, so the reason for the discount is false and the account can be under-margined against its actual combined short-option exposure.

**Mitigation**  
Require semantic validation for partnered short put/call positions before applying the strangle discount, e.g. enforce putStrike <= callStrike with non-overlapping ranges, or fall back to unpartnered requirements when the pair can both be in the money.

### [M-21] Spread collateral depends on arbitrary leg ordering and can understate the intended requirement

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Medium  
**Location**: contracts/RiskEngine.sol:1637; contracts/RiskEngine.sol:1757; contracts/RiskEngine.sol:1799; contracts/RiskEngine.sol:1868

**Summary/Description**  
The spread partner path charges the spread requirement only when index < partnerIndex, but _computeSpread documents and implements its parameters as if index is the long leg and partnerIndex is the short leg. A valid TokenId can place the short leg at the lower index, causing the asymmetric calendar-width and contracts terms to be computed from the short leg instead of the long leg and lowering the required collateral versus the intended long-first spread calculation.

**Root Cause**  
TokenId riskPartner validation does not canonicalize or constrain long/short ordering for spreads, and RiskEngine uses leg index ordering as a duplicate-counting guard while _computeSpread uses the passed index semantically.

**Pre_conditions**  
A user mints a same-asset, same-optionRatio, same-tokenType partnered spread with one long and one short option leg, placing the short leg at the lower leg index.

**Impact**  
The account receives a spread collateral requirement calculated with the wrong leg as the primary leg. For asymmetric strikes or widths, the calendar adjustment and contracts/notional term can be smaller than the long-first requirement, allowing the account to pass solvency with less collateral than the spread formula intends.

**Mitigation**  
In the spread branch, identify the long leg and short leg explicitly before calling _computeSpread, or make _computeSpread symmetric by using conservative max/min values independent of leg index. TokenId validation can also reject non-canonical spread ordering if the engine relies on it.

### [M-22] LPs can withdraw at the pre-loss share price before liquidation socializes bad debt

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/CollateralTracker.sol:503-506,651-657,690-733,795-858,1257-1354; contracts/PanopticPool.sol:1392-1461,1478-1590; contracts/RiskEngine.sol:487-607

**Summary/Description**  
CollateralTracker does not mark or reserve for bad debt while an account is liquidatable but before liquidation settlement. A passive LP with no open positions can withdraw up to the available deposited assets using the current pre-loss share price, because maxWithdraw()/withdraw() only check the owner's position count and local available balance. The loss is later socialized only when settleLiquidation() mints/transfers shares for the liquidation bonus/protocol loss.

**Root Cause**  
Prospective liquidation loss is external to CollateralTracker accounting until the liquidation transaction reaches settleLiquidation(); totalAssets() continues to use tracked deposited assets, AMM assets, and unrealized interest, while withdrawal limits do not include a vault-wide solvency or pending-loss adjustment.

**Pre_conditions**  
At least one account is insolvent across the liquidation ticks and will cause positive liquidation bonus/protocol-loss settlement; the vault still has withdrawable s_depositedAssets; another LP has no open positions and exits before the liquidation is executed.

**Impact**  
Early withdrawers receive assets at the pre-loss exchange rate and avoid the later dilution/minted-share socialization. The same bad debt is concentrated on remaining LPs, enabling bank-run behavior around observable liquidations and unfair loss distribution.

**Mitigation**  
Introduce loss marking or a conservative withdrawal gate when accounts are liquidatable, or settle/mark protocol loss before allowing unrelated withdrawals to use a pre-loss exchange rate. At minimum, withdrawal limits should account for known pending liquidation shortfalls.

### [M-23] Stale share price in margin checks can make borrowers liquidatable before consistent interest accrual

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/CollateralTracker.sol:503-528, contracts/CollateralTracker.sol:1087-1105, contracts/CollateralTracker.sol:886-976, contracts/RiskEngine.sol:1120-1189, contracts/PanopticPool.sol:1399-1461

**Summary/Description**  
RiskEngine margin checks call CollateralTracker.assetsAndInterest(), which values the user's shares with stored totalAssets() while computing owed interest from a simulated current borrow index. Pending global interest is therefore deducted as fresh borrower debt without first being reflected in the borrower's share value. The state-changing accrual path uses a consistent current totalAssets basis before burning interest shares, so liquidation eligibility can be stricter than the actual post-accrual state.

**Root Cause**  
CollateralTracker.assetsAndInterest() mixes stored vault accounting for convertToAssets(balanceOf[user]) with fresh simulated borrower interest from _owedInterest(). RiskEngine._getMargin then subtracts or requirements-adds that fresh interest when deciding solvency.

**Pre_conditions**  
A collateral market has nonzero s_assetsInAMM, time has passed since the last interest accrual, and an account close to its margin threshold owns a nontrivial share balance while owing interest.

**Impact**  
A liquidator can evaluate the account against stale share value plus fresh debt and route liquidation even though applying the same pending interest consistently would increase the borrower's share price before interest shares are burned. The account can be prematurely liquidated and pay liquidation bonus or lose positions at a state that is not insolvent under consistent accrual accounting.

**Proof of Concept**  
Not constructed; code-level review only.

**Mitigation**  
Return both balance and owed interest from one simulated current market state, or accrue/preview pending global interest into the totalAssets used for the user's share valuation before subtracting owed interest in margin checks.

### [M-24] Public known: forced closure delegation can leave unowned shares after interest burns

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: README.md:119-127, contracts/CollateralTracker.sol:1214-1255, contracts/CollateralTracker.sol:886-976, contracts/CollateralTracker.sol:1395-1515, contracts/PanopticPool.sol:1627-1661, contracts/PanopticPool.sol:1680-1702, contracts/RiskEngine.sol:301-388

**Summary/Description**  
Forced settlement paths temporarily delegate type(uint248).max shares without increasing supply. If interest or settlement burns occur while the delegated balance is present, revoke() restores the full missing amount to _internalSupply whenever the post-settlement balance is below the delegation amount. When the user entered with real shares that should have been consumed, this restoration can leave those real shares represented in supply without an owner.

**Root Cause**  
delegate() and revoke() only track one aggregate temporary balance threshold. They do not separately track how many real shares existed before delegation, how many temporary shares were burned, and how many replacement shares were transferred in by refund(), so revoke() can restore too much supply after a forced closure.

**Pre_conditions**  
A forced exercise, keep-open settlement, or liquidation-style forced flow delegates shares to an account that owes interest or settlement amounts. A burn occurs against the delegated balance before revoke()/settleLiquidation removes the temporary shares.

**Impact**  
The ERC20Minimal invariant that user balances should not exceed or diverge from internal supply is broken in the other direction: real shares can remain in total supply after the account balance is zeroed. This dilutes remaining shareholders and corrupts share-price accounting. The specific orphan-share pattern is listed as public known in the contest README, so this report entry is for inventory, not novelty.

**Proof of Concept**  
Not constructed; code-level review only.

**Mitigation**  
Record the pre-delegation real balance and the exact temporary-share amount, then restore only temporary shares actually burned. Alternatively avoid using ERC20 balance inflation for forced settlement and settle temporary credit through separate accounting.

### [M-25] Native CollateralTracker settlement can be blocked by stale V4 synced currency

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: contracts/CollateralTracker.sol:440-483; contracts/CollateralTracker.sol:557-585; contracts/CollateralTracker.sol:611-640; contracts/CollateralTracker.sol:1262-1308; contracts/PanopticPool.sol:1529-1590; contracts/SemiFungiblePositionManagerV4.sol:503-557; contracts/SemiFungiblePositionManagerV4.sol:871-899; lib/v4-core/src/PoolManager.sol:277-363; lib/v4-core/src/interfaces/IPoolManager.sol:183-189; lib/v4-core/test/Sync.t.sol:254-337

**Summary/Description**  
For V4 native-currency positive deltas, CollateralTracker calls PoolManager.settle{value: amount}() without first resetting the PoolManager synced currency to address(0). V4 PoolManager interprets settle() according to the transient syncedCurrency slot; if an earlier same-transaction sync left that slot set to an ERC20, native settle reverts with NonzeroNativeValue instead of accounting the ETH payment. The PoolManager claim arithmetic is otherwise balanced when native settlement is correctly selected.

**Root Cause**  
The native branch in unlockCallback assumes PoolManager.settle{value: ...} always settles address(0), but PoolManager._settle only treats msg.value as native when CurrencyReserves.getSyncedCurrency() is address(0). The branch should reset/sync the native currency before paying ETH.

**Pre_conditions**  
A V4 CollateralTracker uses address(0) as its underlying asset; a positive native delta is being settled through deposit/mint or negative-bonus liquidation; earlier in the same transaction PoolManager.sync(nonNativeCurrency) was called and not reset with sync(address(0)) before CollateralTracker's native settle. This can be self-induced by a batching contract/router, or created during PanopticPool liquidation if a V4 hook called during the preceding SFPM burn leaves syncedCurrency set to an ERC20.

**Impact**  
The positive native settlement reverts before PoolManager claims are minted to the PanopticPool. Direct standalone EOA calls without a prior same-transaction sync are not affected, and the claim conversion is balanced when syncedCurrency is address(0). In composed/batched flows and hooked pools, however, native deposits/mints or native negative-bonus liquidation payments can be blocked, leaving liquidations unresolved while the same hook/batch ordering repeats.

**Proof of Concept**  
Not run per instruction; code-level path verified against V4 PoolManager and Sync.t.sol behavior.

**Mitigation**  
In the native branch, call poolManager().sync(Currency.wrap(address(0))) before poolManager().settle{value: amount}(), or otherwise ensure the synced currency is reset before native settlement. Continue rejecting or refunding incorrect msg.value on ERC20 branches separately.

### [M-26] Multiple zero-width credit legs overwrite margin credit

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/RiskEngine.sol:1317-1343, contracts/RiskEngine.sol:1173-1183, contracts/types/TokenId.sol:473-513, contracts/SemiFungiblePositionManager.sol:815-837

**Summary/Description**  
RiskEngine._getRequiredCollateralAtTickSinglePosition loops over every leg that matches the requested token type, but when it encounters a long zero-width credit leg it assigns credits = amountMoved instead of accumulating with +=. TokenId validation allows multiple active zero-width long legs of the same token type as long as their strike/width/tokenType chunks are not identical and risk partners are mutual. SFPM treats each zero-width long leg as a real token delta, but RiskEngine only returns the last matching credit for that token within the position.

**Root Cause**  
The credit accumulator is overwritten inside the per-leg loop instead of adding each matching credit leg, and TokenId validation does not enforce at most one same-token credit leg per position.

**Pre_conditions**  
A user holds a valid multi-leg tokenId containing two or more long zero-width credit legs with the same tokenType. The account relies on the aggregate credit amount for margin, withdrawal, or liquidation classification.

**Impact**  
The margin calculation understates the user's available credit balance for that token. This is conservative for protocol solvency, but it can make a solvent account appear less healthy or insolvent, enabling unfair liquidation/force-exercise routing or blocking withdrawals/position changes for valid multi-credit positions.

**Mitigation**  
Accumulate credits with += for every matching long zero-width leg, or reject tokenIds with more than one credit leg per token type if the engine intends to support only one such credit. Add coverage for multiple same-token credit legs in _getTotalRequiredCollateral/getMargin.

### [M-27] Capped IRM adaptation undercharges long idle high-utilization periods

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/RiskEngine.sol:90-92, contracts/RiskEngine.sol:2217-2253, contracts/CollateralTracker.sol:985-1024, contracts/CollateralTracker.sol:1047-1053, README.md:438-440

**Summary/Description**  
RiskEngine._borrowRate caps the elapsed time used to adapt rateAtTarget and to compute the average adaptive curve rate, but CollateralTracker._calculateCurrentInterestState applies that returned average rate over the full market epoch delta. If a high-utilization market is not accrued for longer than IRM_MAX_ELAPSED_TIME, borrowers are charged for the entire idle period at an average rate from only the first capped window while rateAtTarget also advances only by that capped amount.

**Root Cause**  
The IRM cap is applied inside _borrowRate before the average-rate integration, while the caller separately computes interest over uncapped deltaTime. The code does not either integrate the adaptive curve over the full elapsed interval or split the full idle interval into capped update windows before compounding interest.

**Pre_conditions**  
The market has nonzero rateAtTarget, utilization is above TARGET_UTILIZATION, and no one accrues that collateral market for more than IRM_MAX_ELAPSED_TIME before a borrower or other caller triggers accrual.

**Impact**  
At 100% utilization from the initial target rate, the capped 4096-second window keeps the returned average borrow rate near 16% APR even after a 30-day idle period, whereas the uncapped adaptive curve would ramp toward the 800% APR maximum and produce a much higher average. The difference is foregone interest for LPs and can be a material percentage of the borrowed assets over long idle periods. Permissionless accrueInterest can reduce likelihood, but it is not enforced or incentivized by the code path.

**Mitigation**  
When elapsed time exceeds the cap, either accrue the full interval by iterating/splitting into capped adaptive windows, or explicitly apply the capped elapsed to both adaptive-rate movement and the interest period while storing the remaining elapsed state for later. Alternatively, require periodic accrual or document and enforce a bounded idle-window assumption.

### [M-28] SFPM fee-base signed boundary can block fee settlement

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/SemiFungiblePositionManager.sol:1073-1115, contracts/SemiFungiblePositionManager.sol:1193-1210, contracts/SemiFungiblePositionManager.sol:1391-1438, contracts/types/LeftRight.sol:271-287, lib/v3-core/contracts/libraries/Position.sol:63-89, lib/v3-core/contracts/libraries/Tick.sol:69-96

**Summary/Description**  
V3 SFPM stores cumulative Uniswap fee-growth products as int128 feesBase values, then computes new fees by subtracting the signed slots. Uniswap treats fee growth as uint256 modular arithmetic, so when the lower 128-bit feesBase representation crosses the int128 sign boundary, SFPM can turn a positive modular fee delta into a signed underflow/revert or into a rectified zero delta.

**Root Cause**  
s_accountFeesBase encodes a modular uint128 fee-growth product as a signed int128 value, but LeftRightSigned.subRect subtracts signed values with an int128 fit check instead of computing the modular uint128 delta used by Uniswap fee accounting.

**Pre_conditions**  
A V3 position chunk has nonzero net liquidity and the stored feesBase and current feesBase straddle the int128 sign boundary. This can arise from long-lived cumulative fee growth or from Uniswap feeGrowthInside values whose lower bits are already near the boundary after tick fee-growth underflow/overflow behavior.

**Impact**  
Mint, burn, settle, and premium view paths that need to collect or simulate fees for the chunk can revert, making affected positions difficult or impossible to close until a very large additional fee-growth movement changes the signed difference. In paths where the signed difference is negative but fits int128, SFPM rectifies it to zero and can advance the baseline without accounting the positive modular fees, stranding seller premium.

**Mitigation**  
Store feesBase as an unsigned modular value and compute deltas with unchecked uint128/uint256 modular subtraction, then rectify only true rounding dust. Alternatively keep the full uint256 Uniswap feeGrowthInsideLast snapshot and derive fees using Uniswap's modular delta formula.

### [M-29] Long idle interest accrual under-compounds with third-order Taylor approximation

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/libraries/Math.sol:1225-1233; contracts/CollateralTracker.sol:985-1024; contracts/RiskEngine.sol:169-171; contracts/CollateralTracker.sol:879-892

**Summary/Description**  
CollateralTracker applies Math.wTaylorCompounded to the entire elapsed market delta when updating interest. The helper keeps only the first three non-zero Taylor terms of e^(rate*time)-1, so it is accurate for small rate*time windows but materially underestimates continuous compounding over long idle periods at high borrow rates.

**Root Cause**  
The interest accrual path does not split long elapsed time into small compounding windows or use a bounded-error exponential. It passes the full deltaTime into a low-order polynomial approximation whose omitted positive terms become dominant as rate*time grows.

**Pre_conditions**  
A collateral market has nonzero borrowed assets, a high borrow rate, and no accrual-triggering interaction for a long period before accrueInterest or another user action updates the market.

**Impact**  
At the documented maximum curve rate of 800% APR, a 4096-second accrual is effectively exact, but a single 365-day accrual returns about 125.33x interest from the Taylor terms while exact continuous compounding is about 2979.96x. The gap is foregone interest for LPs and an underpayment by borrowers whenever the market is allowed to remain idle for long high-rate intervals.

**Mitigation**  
Accrue long elapsed periods by iterating over bounded time slices, or replace the third-order approximation with an exponential routine whose error is bounded for the maximum supported rate*time. Alternatively enforce and document a maximum idle accrual interval.

### [M-30] Temporary pre-mint deposits can capture no-builder commission burns

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Medium  
**Location**: contracts/CollateralTracker.sol:547-580, contracts/CollateralTracker.sol:744-770, contracts/CollateralTracker.sol:1531-1560, contracts/PanopticPool.sol:745-763, contracts/PanopticPool.sol:692-700, contracts/RiskEngine.sol:864-885

**Summary/Description**  
In the no-builder path, option mint commission is paid by burning the option owner's CollateralTracker shares. Because deposits mint shares immediately at the pre-action exchange rate, a trader can deposit a large temporary amount before minting, own most of the supply during their own commission burn, and later withdraw the temporary collateral while remaining solvent. The burn then increases the value of the trader's remaining shares instead of transferring the full commission value to passive PLPs.

**Root Cause**  
Commission distribution is implemented as an instantaneous share burn against current totalSupply, and open-position withdrawals do not distinguish long-term collateral from temporary shares that were present only for the commission event.

**Pre_conditions**  
The mint uses builderCode 0 or otherwise routes a burned commission portion; the trader can supply temporary collateral before minting; after minting, the account can withdraw most of that collateral while still passing the stored-utilization solvency check.

**Impact**  
Passive PLPs can receive only a small fraction of the intended no-builder commission. If the trader's temporary deposit dominates the pre-mint supply, the externally paid commission approaches commission * existingSupply / postDepositSupply, so repeated large mints can materially reduce PLP fee income without requiring the temporary assets to remain supplied.

**Proof of Concept**  
Not constructed; this was code-level review only.

**Mitigation**  
Distribute mint commission to a pre-action or time-weighted eligible supply, exclude the option owner's just-added shares from their own commission distribution, or require liquidity used for commission participation to remain locked/eligible through a minimum period.

### [M-31] Stale settledTokens can be captured by the first seller after a chunk empties

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/PanopticPool.sol:197-203, contracts/PanopticPool.sol:647-649, contracts/PanopticPool.sol:1048-1115, contracts/PanopticPool.sol:1168-1225, contracts/PanopticPool.sol:1260-1307, contracts/PanopticPool.sol:2075-2121

**Summary/Description**  
When the last short liquidity in a chunk exits, PanopticPool resets s_grossPremiumLast to the current SFPM gross accumulator but leaves any remaining s_settledTokens in storage. The first later seller minted into the now-empty chunk starts with both its personal snapshot and the global baseline at the current accumulator, so old settled tokens are excluded from the new owed denominator but remain in the settled-token numerator used by _getAvailablePremium. After any new gross premium accrues, that seller can settle or burn and receive stale tokens that were settled for a prior liquidity epoch.

**Root Cause**  
s_settledTokens is keyed only by chunk and is not cleared, epoch-scoped, or otherwise reconciled when total short liquidity reaches zero. _getAvailablePremium combines this stale numerator with a denominator derived from only post-empty gross premium because s_grossPremiumLast and s_options are reset to the current accumulator on the later mint.

**Pre_conditions**  
A chunk reaches zero total short liquidity while s_settledTokens[chunk] remains nonzero. This can happen from precision residuals or from keep-open/self-settlement paths that advance seller snapshots before SFPM-collected fees are committed, then a later burn collects those tokens after the seller no longer has attributable premium. A new seller is the first short liquidity provider in the same chunk and later accrues nonzero gross premium.

**Impact**  
The new seller can receive settled premium that was generated before it provided liquidity, misallocating old sellers' or otherwise residual chunk funds. The amount is bounded by the stale s_settledTokens balance and the new seller's accrued premium, but the stale balance can be larger than dust when prior settlement paths leave collected fees in the chunk. This breaks the documented proportional-to-liquidity premium distribution and affects the core premium withdrawal path.

**Mitigation**  
When total short liquidity becomes zero, explicitly clear s_settledTokens or move residual tokens according to a documented policy, and ensure first-mint baseline initialization cannot pair old settled-token balances with a new gross-premium epoch. Alternatively include an epoch id in the chunk key or track per-epoch residual ownership.

### [M-32] Pre-liquidation premium settlement or force exercise bypasses liquidation haircut

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/PanopticPool.sol:1360-1461,1482-1590,1598-1663,1671-1702,1135-1320,1989-2063; contracts/RiskEngine.sol:399-485,620-799; contracts/libraries/InteractionHelper.sol:112-173; contracts/CollateralTracker.sol:1262-1381,1395-1520

**Summary/Description**  
A third party can settle a buyer's long premium, or force-exercise a tokenId containing accrued long premium, while the buyer is still solvent and then liquidate the same account after an allowed price or interest transition. Both pre-liquidation paths commit the paid long premium into s_settledTokens and/or remove the exercised tokenId from the account's future position list, so a later liquidation no longer sees that premium in premiasByLeg and haircutPremia cannot claw it back. If the same account were liquidated at the final liquidatable state before the pre-settlement/force-exercise step, the liquidation path would keep that long premium uncommitted and haircut it to reduce protocol loss.

**Root Cause**  
_settlePremium and _forceExercise commit settled long premium before liquidation. _forceExercise additionally burns the whole tokenId with COMMIT_LONG_SETTLED and removes it from the owner's position hash, while _liquidate only passes the positions still in positionIdList and the current burn's premiasByLeg to haircutPremia. Previously committed settledTokens, and premium belonging to a previously force-exercised tokenId, are treated as seller-available premium and are not reserved for later liquidation loss.

**Pre_conditions**  
The target has accrued long premium and remains solvent enough for dispatchFrom's settle-premium or force-exercise branch plus the post-action solvency check. For the force-exercise variant, the target owns an exercisable tokenId with at least one non-credit long leg. The account can then become liquidatable, for example through an allowed price move or interest accrual, before liquidation executes. The eventual liquidation has protocol loss that would otherwise be mitigated by haircutting the unsettled premium.

**Impact**  
Protocol loss minimization becomes order-dependent. Settlers or sellers/exercisors can pull premium out of the future haircut base before liquidation, shifting avoidable loss to PLPs while sellers keep premium that the liquidation-first path would have clawed back. In the force-exercise variant the exercise cost paid to the buyer only offsets this by the RiskEngine exercise fee and can be much smaller than the premium removed from the future haircut base.

**Proof of Concept**  
Not run; code-level trace only, per instruction not to run tests unless requested.

**Mitigation**  
When third-party premium settlement or force exercise is performed for a near-liquidation account, reserve the settled amount for liquidation haircut until a fresh solvency buffer is established, or make liquidation haircut consider recent/pre-liquidation settled long premium for the affected positions. For force exercise specifically, carry the exercised tokenId's settled long premium into a later liquidation reserve or require stronger post-force-exercise solvency buffers.

### [M-33] Created-but-unpriced V3 pools can seed Panoptic oracle at zero

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/SemiFungiblePositionManager.sol:330-343,348-356; contracts/PanopticFactory.sol:125-134,183-201; contracts/PanopticPool.sol:315-335,1367-1465,1719-1771,1808-1813,1944-1950; lib/v3-core/contracts/UniswapV3Pool.sol:97-105,272-289

**Summary/Description**  
SFPM.initializeAMMPool and PanopticFactory.deployNewPool treat a nonzero Uniswap V3 factory getPool result as an initialized AMM pool. A V3 pool address exists after createPool but before UniswapV3Pool.initialize sets sqrtPriceX96/unlocked, so PanopticPool.initialize can read slot0.tick as zero and seed all internal oracle observations and EMAs at zero before the real pool price is initialized.

**Root Cause**  
The V3 registration path checks only FACTORY.getPool(token0, token1, fee) != address(0), not IUniswapV3Pool(univ3pool).slot0().sqrtPriceX96 != 0. PanopticPool.initialize then trusts SFPM.getCurrentTick(poolKey()) for oracle bootstrap, and Uniswap V3 exposes zero slot0 values until initialize is called.

**Pre_conditions**  
A participant creates a Uniswap V3 pool through the canonical factory but does not initialize its price. The Panoptic pool is deployed for that created pool before UniswapV3Pool.initialize is called. The V3 pool is later initialized at a tick materially different from zero.

**Impact**  
The Panoptic pool starts with stale zero oracle state. Later actions can run with safeMode derived from the live-vs-zero deviation, while liquidation classification requires insolvency at stale spot/latest ticks as well as current/TWAP ticks. This can delay or block complete liquidation for accounts that are unsafe at the real pool price but still solvent at the bad bootstrap tick until the internal oracle catches up.

**Proof of Concept**  
Not run; code-level trace only, per instruction not to run tests unless requested.

**Mitigation**  
Require the V3 pool to be price-initialized before SFPM registration or PanopticPool deployment, for example by reading slot0 and reverting when sqrtPriceX96 == 0. Bootstrap PanopticPool oracle state only from an initialized V3 pool.

### [M-34] Per-leg premium amounts can wrap when packed into signed slots

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/PanopticPool.sol:2038-2063, contracts/PanopticPool.sol:501-533, contracts/PanopticPool.sol:1173-1185, contracts/CollateralTracker.sol:1412-1413, contracts/CollateralTracker.sol:1498-1500, contracts/libraries/Math.sol:456-465

**Summary/Description**  
_getPremia computes each leg premium as an unsigned token amount, then explicitly casts it to int128 before packing it in LeftRightSigned. If a per-position premium slot exceeds type(int128).max, the explicit cast wraps. For short legs this can turn the returned unsigned shortPremium into a near-uint128 value; for long legs the later negation can flip the sign or revert at the exact boundary. Downstream settlement and margin paths treat the sign as economically meaningful, so wrapped long premium can be accounted as a credit instead of a debt.

**Root Cause**  
Per-leg premium token amounts are downcast with raw int128(...) conversions instead of checked Math.toInt128(...) or an unsigned representation with explicit boundary handling. The SFPM accumulator cap is uint128, but the final (accumulatorDelta * liquidity) / 2**64 amount is not proven to fit in int128 before signing.

**Pre_conditions**  
A position accrues more than type(int128).max raw token units of premium in one token before its premium snapshot is advanced. This requires extreme accumulated fee volume, but it is not the same as using a token whose total supply exceeds the contest's stated 2^127-1 token-supply bound because cumulative fees can accrue over time.

**Impact**  
Margin views and solvency checks can overstate credits or debits, and settlement can apply the wrong premium direction. In the long-leg settlement path, a wrapped positive premium is added to realizedPremia and then subtracted in CollateralTracker.settleBurn, which can mint value to the premium payer and increase tracked deposited assets without matching incoming tokens. The same boundary can also block liquidation or settlement if the signed negation overflows at exactly type(int128).min.

**Proof of Concept**  
Not run; code-level trace only, per instruction not to run tests unless requested.

**Mitigation**  
Avoid signed 128-bit packing for the computed token amount until after a checked bound decision. Use checked casts such as Math.toInt128 for any signed slots and define a safe cap/revert/partial-settlement policy before the premium amount crosses the signed range. Apply the same policy consistently in margin, settlement, and available-premium calculations.

### [M-35] Force exercise double-shortage substitution can pay caller from delegated shares

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/RiskEngine.sol:301-387; contracts/PanopticPool.sol:1598-1661; contracts/CollateralTracker.sol:1221-1255,1369-1380,1395-1520; contracts/tokens/ERC20Minimal.sol:103-110

**Summary/Description**  
RiskEngine.getRefundAmounts handles a delegated payor shortage by returning immediately after the first deficient token side. In the token0-shortage branch it makes the caller pay extra token0 and adds positive token1 compensation; in the token1-shortage branch it does the inverse. If the payor is also short on the compensation side, PanopticPool still calls CollateralTracker.refund before revoke, so the positive compensation transfer can draw from the payor's delegated phantom shares instead of real surplus collateral.

**Root Cause**  
The refund substitution logic assumes at most one token-side shortage and does not verify that the opposite token-side compensation is backed by real payor shares before CollateralTracker.refund runs against a delegated balance.

**Pre_conditions**  
A force exercise or delegated settlement leaves the target short on both collateral token sides after burn settlement and fee effects. This is reachable under current force-exercise eligibility for in-range or mixed long positions, where closing liquidity can require both token0 and token1. The caller supplies a final list that passes the normal post-action checks.

**Impact**  
The caller can be required to cover only the first detected shortage while receiving opposite-token compensation that is effectively minted through delegated-share consumption and revoke supply restoration. This shifts the unhandled second-side deficit to the affected collateral vault/PLPs and makes caller payment bounds depend on branch order rather than total settlement deficit.

**Proof of Concept**  
Not run; code-level review only, per instruction not to run tests unless requested.

**Mitigation**  
Compute shortages for both token sides before returning, and only grant opposite-token compensation up to the payor's real surplus after accounting for delegated shares and pending fees. If both sides are short, require the caller to cover both deficits or revert before any refund transfer.

### [M-36] One-tick long positions can revert solvency checks at the strike

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/RiskEngine.sol:1506; contracts/RiskEngine.sol:1539; contracts/PanopticPool.sol:692; contracts/PanopticPool.sol:950; contracts/PanopticPool.sol:1753

**Summary/Description**  
_getRequiredCollateralSingleLegNoPartner computes long-leg OTM decay with distanceFromStrike = max(positionWidth / 2, abs(atTick - strike)). For a valid one-tick-wide long leg in a tickSpacing=1 pool, asTicks gives tickLower = strike and tickUpper = strike + 1, so positionWidth / 2 == 0 and atTick == strike makes distanceFromStrike == 0. The next expression divides by distanceFromStrike, causing RiskEngine margin and solvency checks to revert instead of returning a requirement.

**Root Cause**  
The long-leg decay formula assumes every non-zero-width position has a non-zero minimum distance, but TokenId/PanopticMath allow width=1 with tickSpacing=1 and the SFPM mint validators only check tick multiples and enforced tick bounds.

**Pre_conditions**  
A V3 or V4 pool with tickSpacing=1; an account holds a non-zero-width long leg with encoded width=1 and strike inside enforced bounds; a solvency tick passed to RiskEngine equals that strike.

**Impact**  
Any protocol path that calls _validateSolvency/_checkSolvencyAtTicks/getMargin for the affected account can revert. This can block withdrawals or burns for the account and can also block liquidation/settlement attempts while one checked tick equals the one-tick long leg's strike.

**Mitigation**  
Clamp distanceFromStrike to at least 1 before using it as a denominator, or reject width=1 long legs/tick ranges whose half-width floors to zero. Add coverage for width=1, tickSpacing=1, atTick == strike in reqSingleNoPartner and full isAccountSolvent paths.

### [M-37] Oracle rebase clears guardian safe-mode lock

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/types/OraclePack.sol:62-65, contracts/types/OraclePack.sol:129-132, contracts/types/OraclePack.sol:280-283, contracts/types/OraclePack.sol:448-453, contracts/types/OraclePack.sol:485-498, contracts/types/OraclePack.sol:629-632, contracts/RiskEngine.sol:230-237, contracts/RiskEngine.sol:931-938, contracts/PanopticPool.sol:293-300, contracts/PanopticPool.sol:622-631

**Summary/Description**  
OraclePack.rebaseOraclePack uses UPPER_118BITS_MASK to preserve the upper fields while replacing the reference tick and residual block, but the mask preserves only bits 126-255. The OraclePack layout places lockMode at bits 118-119 and spotEMA at bits 120-141, so every rebase clears the guardian lock bits and the low six bits of the stored spot EMA. insertObservation then reads lockMode from the rebased pack and persists the cleared value in storeOraclePack, so a pool locked by the guardian can return to unlocked safe-mode behavior without unlockPool being called.

**Root Cause**  
UPPER_118BITS_MASK is shifted eight bits too high for the documented layout. It should preserve all bits above the 118-bit reference/residual region, but the literal at OraclePack.sol:64-65 clears bits 118-125 before rebaseOraclePack writes the new reference and residuals.

**Pre_conditions**  
The guardian has called lockPool on a PanopticPool, setting OraclePack lockMode to 3. Later, an oracle update is eligible in a new epoch and the new clamped tick is more than MAX_RESIDUAL_THRESHOLD from the stored reference, causing insertObservation to call rebaseOraclePack. This can be reached through public pokeOracle or through normal dispatch solvency validation after enough price movement across oracle epochs.

**Impact**  
The forced safe-mode override is intended to remain active until the guardian explicitly unlocks the pool. After a rebase, RiskEngine.isSafeMode no longer adds the lockMode value, so dispatch no longer treats the pool as forced level-3 safe mode. New mints that should revert under the lock can proceed if the automatic oracle checks do not independently produce safeMode greater than 2, and collateral checks that rely on safeMode can use normal assumptions instead of the guardian-enforced emergency assumptions. The same mask error also perturbs the prior spotEMA by clearing its low six bits before updateEMAs recomputes the new EMA value.

**Mitigation**  
Replace UPPER_118BITS_MASK with the mask that preserves bits 118-255 and clears only bits 0-117, for example ~((uint256(1) << 118) - 1). Prefer bitwise OR over addition after non-overlap is asserted, and add regression coverage that lockMode and all EMA fields are unchanged across rebase except for the intentional EMA update done by insertObservation.

### [M-38] Public known: partial interest payment keeps stale borrow index

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: README.md:103-108; 2025_12_panoptic_v12_findings.md:63-84; contracts/CollateralTracker.sol:886-976; contracts/CollateralTracker.sol:1061-1077

**Summary/Description**  
CollateralTracker._accrueInterest handles the insufficient-share, non-deposit path by burning the user's available shares and reducing global unrealized interest, but then persists the user's old borrow-index checkpoint. Because future owed interest is recomputed from netBorrows and the stale checkpoint, interest already partially paid can be charged again. This is listed in the public V12/Nethermind-known area, so this entry is inventory rather than a fresh contest finding.

**Root Cause**  
Partial interest settlement is represented only by burning shares and reducing the global unrealized-interest bucket; there is no per-user unpaid-interest field or proportional checkpoint update, so the old checkpoint is written back after a partial payment.

**Pre_conditions**  
A borrower has positive netBorrows, accrued interest exceeds their current CollateralTracker share balance, and a non-deposit accrual path is reached.

**Impact**  
The account can be repeatedly charged for already-paid interest portions, while global unrealized interest has already been reduced by the partial payment. This desynchronizes per-user debt from global accounting and can distort later settlement, liquidation, and collateral-exit behavior.

**Proof of Concept**  
Not constructed; code-level review only.

**Mitigation**  
Represent partial interest payments explicitly, either by tracking unpaid interest separately or by advancing/rebasing the user's checkpoint in a way that subtracts the paid amount without losing the remaining debt.

### [M-39] Public known: zero-position collateral exits ignore residual netBorrows

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: High  
**Location**: README.md:28-32; 2025_12_panoptic_v12_findings.md:3-19; contracts/CollateralTracker.sol:399-430; contracts/CollateralTracker.sol:651-656; contracts/CollateralTracker.sol:795-800; contracts/CollateralTracker.sol:1395-1515; contracts/PanopticPool.sol:1918-1922; contracts/libraries/PanopticMath.sol:697-728; contracts/libraries/PanopticMath.sol:738-773

**Summary/Description**  
Collateral exits key off PanopticPool.numberOfLegs while debt principal lives in CollateralTracker.s_interestState. Open and close accounting can leave residual positive netBorrows because the notional amounts used for opening and closing are rounded in opposite directions. Once the position hash reaches zero, transfer, withdraw, and redeem gates allow collateral movement based on zero legs without requiring netBorrows to be zero or fully settled. The transfer variant is listed in V12 public findings, so this entry is inventory rather than a fresh contest finding.

**Root Cause**  
PanopticPool position membership and CollateralTracker borrow principal are separate state mirrors. Exit gates trust only the pool's leg count and do not enforce that the collateral tracker has no residual debt for the exiting owner.

**Pre_conditions**  
A user opens and fully closes positions such that rounding or another accounting path leaves positive netBorrows after the final position is removed; the user then calls transfer, withdraw, or redeem from the CollateralTracker.

**Impact**  
Collateral can be moved away from an account whose debt principal remains in the CollateralTracker. That leaves debt attached to an account with no enforceable position list and can create bad debt/accounting drift for PLPs if repeated or combined with collateral migration.

**Proof of Concept**  
Not constructed; code-level review only.

**Mitigation**  
Require netBorrows == 0, or an equivalent full debt/interest settlement, before zero-position transfers, withdrawals, redeems, or donations. Alternatively keep position and borrow state in one authoritative accounting structure.

### [L-01] SFPM leg accounting omits vegoid from the position key

**Severity**: Low  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/SemiFungiblePositionManager.sol:929; contracts/SemiFungiblePositionManager.sol:1371; contracts/SemiFungiblePositionManager.sol:1403; contracts/SemiFungiblePositionManagerV4.sol:927; contracts/SemiFungiblePositionManagerV4.sol:1196; contracts/types/TokenId.sol:94

**Summary/Description**  
The V3 and V4 SFPM store/query per-leg liquidity and premium state without the vegoid or full SFPM poolId, while token IDs and pool data support distinct vegoids for the same underlying pool. A single account or wrapper using multiple vegoids for the same pool and leg geometry will merge liquidity and premium accounting across distinct SFPM pool IDs.

**Root Cause**  
The per-leg position key excludes tokenId.vegoid() and tokenId.poolId() even though initializeAMMPool stores PoolData by (underlying pool, vegoid) and premium math depends on vegoid.

**Pre_conditions**  
Same caller/account uses the same underlying pool, tokenType, tick range, and different initialized vegoids through the public SFPM or through a wrapper. The official PanopticPool dispatch path checks tokenId.poolId() against its immutable poolId, limiting this path for normal PanopticPool use.

**Impact**  
Cross-vegoid positions for the same account can consume, close, or accrue against the same s_accountLiquidity and premium accumulator state despite having different ERC1155 token IDs and premium spread parameters.

**Mitigation**  
Include the SFPM pool id or vegoid in the per-leg position key and in all matching query keys.

### [L-02] SFPM premium spread floors vegoid term before accumulator scaling

**Severity**: Low  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/SemiFungiblePositionManager.sol:1309; contracts/SemiFungiblePositionManager.sol:1332; contracts/SemiFungiblePositionManagerV4.sol:1134; contracts/SemiFungiblePositionManagerV4.sol:1157; README.md:221-253; README.md:269-282

**Summary/Description**  
The documented premium spread uses a fractional ν = 1 / VEGOID multiplier on removed liquidity, but _getPremiaDeltas computes removedLiquidity / vegoid and removedLiquidity ** 2 / vegoid before the final full-precision mulDiv. For removed-liquidity values not divisible by VEGOID, the fractional spread component is discarded before X64 accumulator scaling, so owed and gross premium deltas can be lower than the documented formulas even when the position is within the PanopticPool spread cap.

**Root Cause**  
_getPremiaDeltas represents the fractional VEGOID term by integer-dividing the liquidity numerator first instead of carrying the VEGOID factor into the final denominator, e.g. using (netLiquidity * vegoid + removedLiquidity) / (totalLiquidity * vegoid) for owed premium and an equivalent scaled gross numerator.

**Pre_conditions**  
A chunk has nonzero removed liquidity not divisible by VEGOID and later collects fees. In the shipped RiskEngine VEGOID is 4, so removed-liquidity values with nonzero remainder modulo 4 lose part or all of the spread term. The PanopticPool max-spread check bounds removed/net liquidity but does not enforce divisibility or a minimum liquidity that removes this rounding.

**Impact**  
Buyers can owe less long premium and sellers can receive less gross premium than the documented spread formula. The effect is most visible for small-liquidity chunks or low-decimal pools and overlaps general public-known premium rounding limitations, but this specific loss occurs before X64 scaling and is tied to the VEGOID spread term rather than final token-unit dust.

**Proof of Concept**  
Not run; code-level trace only, per instruction not to run tests unless requested.

**Mitigation**  
Carry the VEGOID denominator through the final mulDiv instead of pre-flooring the liquidity term. For example, compute the owed multiplier with netLiquidity * vegoid + removedLiquidity over totalLiquidity * vegoid, and compute the gross multiplier with (totalLiquidity ** 2 - totalLiquidity * removedLiquidity) * vegoid + removedLiquidity ** 2 over totalLiquidity ** 2 * vegoid, with overflow-safe full-precision arithmetic.

### [L-03] Direct SFPM premium gross formula can overflow total-liquidity square

**Severity**: Low  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/SemiFungiblePositionManager.sol:620; contracts/SemiFungiblePositionManager.sol:951; contracts/SemiFungiblePositionManager.sol:986; contracts/SemiFungiblePositionManager.sol:1282; contracts/SemiFungiblePositionManager.sol:1329; contracts/SemiFungiblePositionManager.sol:1335; contracts/SemiFungiblePositionManagerV4.sol:603; contracts/SemiFungiblePositionManagerV4.sol:951; contracts/SemiFungiblePositionManagerV4.sol:986; contracts/SemiFungiblePositionManagerV4.sol:1107; contracts/SemiFungiblePositionManagerV4.sol:1154; contracts/SemiFungiblePositionManagerV4.sol:1160; lib/v3-core/contracts/libraries/Tick.sol:128-133; lib/v4-core/src/libraries/Pool.sol:164-171

**Summary/Description**  
_getPremiaDeltas computes totalLiquidity = netLiquidity + removedLiquidity and then uses totalLiquidity ** 2 and totalLiquidity * removedLiquidity inside an unchecked gross-premium formula. Each lane is only uint128-bounded, so direct SFPM usage can make their sum exceed the 2^128 square boundary even though current in-AMM liquidity remains within the Uniswap per-tick cap. Once totalLiquidity is above the square boundary, the gross-premium denominator and numerator wrap before Math.mulDiv sees them.

**Root Cause**  
The helper assumes totalLiquidity is safely squareable because net and removed liquidity are individually uint128, but the sum of two packed uint128 accounting lanes is not itself uint128-bounded. The calculation is in an unchecked block and does not use overflow-safe mulDiv decomposition for the gross multiplier denominator.

**Pre_conditions**  
A direct SFPM user or wrapper repeatedly opens long liquidity and replenishes short liquidity in the same account/chunk so removedLiquidity accumulates while netLiquidity remains bounded by Uniswap tick liquidity. The canonical PanopticPool path is more constrained by position-list limits and _checkLiquiditySpread, so this is scoped to direct SFPM or custom wrapper usage.

**Impact**  
Gross premium accumulator deltas for the affected chunk can be wrong or revert/freeze once the wrapped denominator is used. This can misstate seller premium snapshots for direct SFPM users. No canonical PanopticPool loss path was confirmed because the normal path applies spread and account-position constraints before settlement.

**Proof of Concept**  
Not run; code-level trace only, per instruction not to run tests unless requested.

**Mitigation**  
Before squaring, prove and enforce totalLiquidity <= type(uint128).max, or rewrite the gross multiplier with full-precision overflow-safe arithmetic that carries the denominator without unchecked exponentiation. Consider imposing a direct-SFPM total sold liquidity cap matching the assumptions used by PanopticPool.

### [I-01] High-price conversions can deviate by one unit after price reduction

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/libraries/PanopticMath.sol:480-488, contracts/libraries/PanopticMath.sol:497-508, contracts/libraries/PanopticMath.sol:517-525, contracts/libraries/PanopticMath.sol:559-630, contracts/libraries/PanopticMath.sol:669-688, contracts/RiskEngine.sol:510-516, contracts/RiskEngine.sol:529-540, contracts/RiskEngine.sol:570-591, contracts/RiskEngine.sol:666-700, contracts/RiskEngine.sol:1047-1049, contracts/RiskEngine.sol:2031-2033

**Summary/Description**  
In the reduced-precision branch used when sqrtPriceX96 >= type(uint128).max, the conversion helpers first floor sqrtPriceX96^2 / 2^64 and then use that reduced Q128 price. This is not always equivalent to applying floor or ceiling to the exact Q192 conversion. convert0to1 and convert0to1RoundingUp can return one token1 unit less than the exact floor/ceiling for very large high-price inputs, while convert1to0 can return one token0 unit more than the exact floor for aligned high-branch amounts.

**Root Cause**  
The high-price branch reduces precision by flooring the scaled price before the final mulDiv. For token0-to-token1, flooring the multiplier can understate the exact Q192 result; for token1-to-token0, flooring the denominator can overstate the exact floor result.

**Pre_conditions**  
sqrtPriceX96 is in the high-price branch and the converted amount is large or aligned enough for the discarded sqrtPriceX96^2 / 2^64 remainder to affect the final integer result. Under protocol-sized 128-bit magnitudes, the observed error is bounded to one smallest token unit.

**Impact**  
A high-price token0-to-token1 conversion can understate a balance, surplus, shortage, or requirement by one smallest token1 unit. A high-price token1-to-token0 floor conversion can overstate the token0 equivalent by one smallest token0 unit in liquidation-bonus and premium-haircut paths. This is a precision-level accounting edge rather than a scalable drain under current supported amount ranges.

**Proof of Concept**  
Arithmetic inspection: with amount = type(int128).max and sqrtPriceX96 = MAX_POOL_SQRT_RATIO, exactFloor(amount * sqrtPriceX96^2 / 2^192) exceeds convert0to1 by 1 and exactCeil exceeds convert0to1RoundingUp by 1. Separately, with sqrtPriceX96 = 2^128 + 2 and amount = 85070591730234615865843651857942052865, exactFloor(amount * 2^192 / sqrtPriceX96^2) is 4611686018427387903 while convert1to0 returns 4611686018427387904.

**Mitigation**  
For high-price conversions, preserve the discarded sqrtPriceX96^2 / 2^64 remainder in the final rounding decision or use a full-precision formulation that preserves exact floor and ceiling semantics across the scaling step.

### [I-02] dispatchFrom does not enforce documented operator approval

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/PanopticPool.sol:1360-1475, README.md:220-223, README.md:455-456

**Summary/Description**  
README documentation says dispatchFrom is for approved operators and requires operator approval from the account owner, but the implementation never checks msg.sender against any account approval state. The function validates the target position list, routes by solvency and list lengths, then only checks post-action solvency for the target and caller.

**Root Cause**  
The dispatchFrom entry point has no owner/operator authorization gate and no link to SFPM ERC1155 approvals or CollateralTracker ERC20 allowances.

**Pre_conditions**  
A caller knows or reconstructs the target account's current position list. The target account satisfies the solvency/list-shape conditions for settlement, force exercise, or liquidation.

**Impact**  
If operator approval is intended as an access boundary, any caller can trigger account-affecting dispatchFrom actions without that approval. Liquidation, force exercise, and premium settlement also have third-party role semantics elsewhere in the docs, so this is recorded as a code/documentation mismatch pending design clarification.

**Mitigation**  
Either enforce the documented owner/operator approval check before operating on account, or update the documentation to state that dispatchFrom liquidation, force exercise, and premium-settlement paths are permissionless and rely only on their operation-specific guards.

### [I-03] Small or split force exercises can round prorated cost down

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/RiskEngine.sol:399-484; contracts/PanopticPool.sol:1413-1435; contracts/PanopticPool.sol:1598-1663; contracts/CollateralTracker.sol:1369-1380; contracts/libraries/PanopticMath.sol:453-472

**Summary/Description**  
exerciseCost sums non-credit long notional only within the single tokenId being force exercised, then applies one signed integer division by DECIMALS. Because dispatchFrom removes exactly one tokenId per force-exercise branch and _forceExercise settles that tokenId's exerciseFees immediately, economically equivalent notional split across multiple tokenIds or calls is floored separately. Small out-of-range long amounts can therefore pay zero 1 bps fee, and split positions can pay less than a single aggregate tokenId would have paid. The same rounding shape exists for the in-range FORCE_EXERCISE_COST fee at a smaller smallest-unit threshold.

**Root Cause**  
The base force-exercise fee is prorated with toward-zero signed integer division after per-tokenId aggregation, with no account-level accumulator and no rounding away from zero or minimum nonzero fee for nonzero prorated costs.

**Pre_conditions**  
A force-exercised account holds valid non-credit long liquidity whose charged token-side notional is small enough, or deliberately split across tokenIds/calls so each per-tokenId amount leaves a fractional fee after multiplying by ONE_BPS or FORCE_EXERCISE_COST and dividing by DECIMALS.

**Impact**  
The exercised account receives less inconvenience fee than the documented rate on the same aggregate notional. For out-of-range exercises, any charged token-side amount below 10,000 raw units pays zero base fee, and repeated split positions can multiply this rounding loss. For in-range exercises, amounts below 98 raw units can similarly round to zero. The effect is bounded by per-position rounding dust but is repeatable up to the open-leg limit.

**Mitigation**  
Compute force-exercise fees with rounding away from zero for nonzero prorated fees, carry a per-account residual across exercised positions, or aggregate all positions being force exercised before applying the fee division. If a minimum fee is intended, enforce at least one smallest unit for each nonzero fee-bearing token side.

### [I-04] Builder feeRecipient is packed into too few bits

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/types/RiskParameters.sol:14-24, contracts/types/RiskParameters.sol:52-78, contracts/types/RiskParameters.sol:169-172, contracts/RiskEngine.sol:253-263, contracts/RiskEngine.sol:864-885, contracts/CollateralTracker.sol:1558-1572, contracts/CollateralTracker.sol:1637-1650

**Summary/Description**  
PanopticPool obtains feeRecipient through RiskEngine.getRiskParameters. _computeBuilderWallet returns address(0) only for builderCode == 0, and for nonzero builderCode the address is computed with CREATE2. getRiskParameters then downcasts the 160-bit address to uint128 with Math.toUint128, so a nonzero computed address with nonzero upper 32 bits reverts instead of silently becoming address(0). If the computed address fits in uint128, getRiskParameters still does not check that wallet code exists, while CollateralTracker.settleMint only checks feeRecipient != 0 before transferring builder-split commission shares to that address. The separate getFeeRecipient helper does reject missing wallet code, and BuilderFactory.deployBuilder only supports uint48 builder codes, so the execution path can either reject most deployed/random builder wallet addresses due to the uint128 packing or route fee shares to a no-code or non-deployable address when the low-address condition happens to hold.

**Root Cause**  
RiskParameters stores feeRecipient in only 128 bits even though builder wallets are 160-bit addresses, and the builder-code validation performed in getFeeRecipient is not applied in getRiskParameters. getRiskParameters also accepts a uint256 salt while BuilderFactory.deployBuilder only deploys uint48 salts.

**Pre_conditions**  
A user supplies a nonzero builderCode whose computed wallet address fits in uint128 but has no deployed BuilderWallet, or uses a salt outside the uint48 range deployable by BuilderFactory.

**Impact**  
A nonzero builder code cannot practically make feeRecipient equal address(0) through truncation because Math.toUint128 is checked; it will revert unless the CREATE2 address is below 2^128. For low-address codes that pass the cast, the builder split of settleMint commission shares can still be sent to an address with no deployed BuilderWallet and become inaccessible. This mainly affects builder-code commission routing and availability, not ordinary no-builder commission accounting.

**Mitigation**  
Store the full 160-bit builder wallet address or otherwise use a separate validated address field, and apply code-existence/deployability validation in getRiskParameters. Explicitly restrict builderCode to deployed uint48 BuilderFactory wallets before packing feeRecipient.

### [I-05] Epoch-truncated oracle timestamps allow sub-period observations

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/types/OraclePack.sol:530-565, contracts/PanopticPool.sol:328-335, contracts/PanopticPool.sol:551-558, contracts/PanopticPool.sol:694-702, README.md:67-71

**Summary/Description**  
computeInternalMedian stores only a 64-second epoch and accepts any update where currentEpoch != recordedEpoch. At the exact next epoch boundary this makes the update eligible, so there is no missed-update off-by-one at the bucket boundary. However, because the lower six timestamp bits are discarded, that bucket boundary can arrive far less than 64 real seconds after the previous update if the previous update landed near the end of its epoch. The function then computes timeDelta as a full 64 seconds for adjacent epochs.

**Root Cause**  
The update gate compares truncated epoch numbers instead of storing and comparing the actual last update timestamp or enforcing block.timestamp >= lastTimestamp + period.

**Pre_conditions**  
An oracle update is made late in one 64-second epoch, and another update is attempted shortly after the next epoch begins. The underlying pool tick at the second update is the value to be sampled, subject to clampTick.

**Impact**  
The median/EMA oracle can accept a new observation before the documented period has elapsed, and EMAs can be advanced as if 64 seconds passed. This weakens the timing assumption around internal oracle updates; broader price-manipulation impact is limited by the repository scope notes excluding non-atomic oracle manipulation and EMA-holding attacks.

**Mitigation**  
Store enough timestamp precision to enforce the actual minimum elapsed period, or require block.timestamp / period to advance only when the previous update occurred at least period seconds ago.

### [I-06] Repeated same-epoch accrual can reapply IRM adaptation

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/RiskEngine.sol:2202-2222, contracts/CollateralTracker.sol:879-892, contracts/CollateralTracker.sol:970-975, contracts/CollateralTracker.sol:1047-1053, contracts/base/Multicall.sol:12-16

**Summary/Description**  
CollateralTracker.accrueInterest is public and Multicall can delegatecall it repeatedly in one transaction. Each call updates rateAtTarget through RiskEngine._borrowRate using elapsed = block.timestamp - (marketEpoch << 2), while the stored market epoch has only 4-second precision. After the first accrual in an epoch, later same-epoch calls can still reuse the intra-epoch seconds and adjust rateAtTarget again even though no real time elapsed between calls.

**Root Cause**  
The IRM stores only block.timestamp >> 2 as marketEpoch but computes adaptation elapsed from the truncated epoch start, not from an exact last-update timestamp or a per-epoch once-only guard.

**Pre_conditions**  
The market has nonzero rateAtTarget, utilization is away from TARGET_UTILIZATION, and a caller invokes accrueInterest or another accrual path more than once in the same 4-second epoch.

**Impact**  
The end rate remains bounded by MIN_RATE_AT_TARGET and MAX_RATE_AT_TARGET, and a single extra call changes the rate by at most about three seconds of adaptation. However, repeated calls can accelerate persistent rateAtTarget movement relative to real elapsed time, affecting future borrower/lender rates. The effect is gas-limited and therefore assessed as Info.

**Mitigation**  
Store the exact last update timestamp, or skip rateAtTarget adaptation when currentEpoch equals the stored marketEpoch after the first update in that epoch.

### [I-07] dispatchFrom empty target list panics before action classification

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/PanopticPool.sol:1360-1465, contracts/PanopticPool.sol:1833-1859, contracts/PanopticPool.sol:1719-1769

**Summary/Description**  
dispatchFrom validates positionIdListTo and computes solvency before selecting an action by list length. In the solvent branch it reads positionIdListTo[toLength - 1] before checking that toLength is nonzero. For an account with no open positions and an empty target list, the empty list can validate and the empty-position solvency loop can return solvent at every tick, so the call reverts through Solidity calldata array bounds instead of a typed InputListFail/NotMarginCalled path.

**Root Cause**  
The action selector derives tokenId from the last element of positionIdListTo before handling the zero-length case.

**Pre_conditions**  
The target account has an empty stored positions hash and dispatchFrom is called with positionIdListTo.length == 0.

**Impact**  
The call cannot complete a state-changing action and reverts before mutating the target account, but integrators receive an unexpected low-level bounds failure for an otherwise classifiable bad input state.

**Mitigation**  
Check toLength == 0 before reading positionIdListTo[toLength - 1] and route empty target-list inputs through an explicit typed error.

### [I-08] ERC4626 maxMint and maxRedeem can overestimate after pending interest accrual

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/CollateralTracker.sol:503-528,592-616,795-823,985-1017

**Summary/Description**  
CollateralTracker maxMint() and maxRedeem() use convertToShares() against stored totalAssets(), but mint() and redeem() first call _accrueInterest(), which can increase unrealizedGlobalInterest and totalAssets before enforcing the same limits. When interest has accrued since the last interaction, maxMint can return shares whose post-accrual previewMint cost exceeds the uint104 asset cap, and maxRedeem can return more shares than can be redeemed against the available deposited assets after the share price increases.

**Root Cause**  
ERC4626 max views do not simulate the same pending global interest accrual that state-changing mint and redeem execute before limit checks.

**Pre_conditions**  
s_assetsInAMM is nonzero, time has passed since the last market-state update, and a caller or integrator uses maxMint() or maxRedeem() as the input to mint() or redeem() before another interaction accrues interest.

**Impact**  
The issue is primarily ERC4626 integration breakage: spec-required max views can overestimate and a state-changing call sized from the view can revert. No direct asset theft was identified from this mismatch alone.

**Mitigation**  
Compute maxMint and maxRedeem from a simulated current interest state, or have state-changing functions enforce limits against the same stored state used by the public max views.

### [I-09] Borrower interest settlement rounds up in favor of the vault

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: contracts/CollateralTracker.sol:906-910; contracts/CollateralTracker.sol:943-945; contracts/CollateralTracker.sol:1061-1077; contracts/libraries/Math.sol:782-792; README.md:383-386

**Summary/Description**  
CollateralTracker computes per-user interest with mulDivRoundingUp and then converts that asset-denominated interest into shares with another rounding-up conversion before burning shares. This can charge a borrower more than the exact formula stated in the interest invariant, with the excess benefiting the remaining vault share price.

**Root Cause**  
The user interest formula and share settlement both use ceiling division, while the documented invariant states the exact netBorrows * (currentBorrowIndex - userBorrowIndex) / userBorrowIndex formula without specifying borrower-unfavorable rounding.

**Pre_conditions**  
A borrower has positive netBorrows, the current borrow index exceeds the user's checkpoint, and either the interest formula or the interest-to-share conversion has a nonzero remainder during a full-payment accrual.

**Impact**  
The excess is bounded to less than one asset unit from the interest calculation plus less than one share from the share burn conversion per full settlement, so this is primarily dust-level value transfer unless the market has unusually coarse asset units or very high share price.

**Mitigation**  
Document the borrower-unfavorable rounding as intentional, or track fractional residuals / use a consistent rounding policy so borrowers are not charged above the stated formula.

### [I-10] ERC20 CollateralTracker positive settlement can trap accidental ETH

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/CollateralTracker.sol:440-483; contracts/CollateralTracker.sol:557-585; contracts/CollateralTracker.sol:611-640; contracts/CollateralTracker.sol:1262-1308; contracts/PanopticPool.sol:1583-1590; contracts/base/Multicall.sol:12-36

**Summary/Description**  
CollateralTracker.deposit, mint, and negative-bonus liquidation are payable to support native-token V4 collateral. When the underlying asset is ERC20 and delta > 0, unlockCallback settles the ERC20 by sync/transferFrom/settle and mints PoolManager claims to the PanopticPool, but any msg.value supplied to the payable entry point is neither used nor refunded.

**Root Cause**  
The native branch refunds valueOrigin surplus, but the ERC20 positive-delta branch does not reject or refund nonzero msg.value.

**Pre_conditions**  
A caller sends native ETH to an ERC20 CollateralTracker deposit/mint, or sends msg.value through PanopticPool liquidation for a non-native token0 positive settlement.

**Impact**  
The attached ETH remains in the CollateralTracker contract and is not reflected in vault accounting or PoolManager claims. This is accidental user-value loss, not a repeatable protocol-drain path.

**Proof of Concept**  
Not run; code-level review only.

**Mitigation**  
For non-native underlying assets, require msg.value == 0 or refund valueOrigin in the ERC20 positive-delta branch.

### [I-11] ERC4626 previewDeposit and previewMint can misquote after pending interest accrual

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/CollateralTracker.sol:503-505,547-562,599-613,887-892,997-1025,970-975

**Summary/Description**  
CollateralTracker.previewDeposit() and previewMint() price shares against stored totalAssets(), but deposit() and mint() first call _accrueInterest(), which can advance global unrealized interest before recomputing the conversion. When pending interest exists, deposit() can mint fewer shares than previewDeposit() reported for the same assets, and mint() can require more assets than previewMint() reported for the same shares in the same transaction.

**Root Cause**  
The public ERC4626 preview paths do not simulate the same pending global interest accrual that deposit and mint execute before pricing the conversion.

**Pre_conditions**  
s_assetsInAMM is nonzero, time has elapsed since the last market-state update, and a caller or integrator reads previewDeposit(assets) or previewMint(shares) before calling the corresponding state-changing function.

**Impact**  
Integrators can overestimate deposit shares or underestimate mint assets, violating ERC4626 preview direction guarantees. The value difference is priced at the post-accrual share price and no direct protocol drain was identified, so this is primarily standards/integration risk.

**Proof of Concept**  
Not run; code-level review only.

**Mitigation**  
Make previewDeposit and previewMint compute against a simulated current interest state, or move pricing before accrual only if the state-changing paths can preserve ERC4626 preview guarantees.

### [I-12] SFPM fee-base rounding can strand fee dust after baseline advances

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/SemiFungiblePositionManager.sol:1022-1043, contracts/SemiFungiblePositionManager.sol:1193-1253, contracts/SemiFungiblePositionManager.sol:1073-1115, lib/v3-core/contracts/UniswapV3Pool.sol:496-520

**Summary/Description**  
SFPM computes amountToCollect from a rounded-down current feesBase minus a rounded-up stored feesBase, then unconditionally advances s_accountFeesBase to the latest rounded-up post-touch baseline. If the rounded delta is lower than Uniswap's credited tokensOwed by one unit, the residual is not collected or included in premium accumulators, and advancing the baseline excludes it from future SFPM accounting.

**Root Cause**  
The internal baseline is advanced from fee-growth snapshots instead of from the actual amount collected or remaining Uniswap tokensOwed, while collect requests only the rounded SFPM delta.

**Pre_conditions**  
A position touch occurs after fee growth where the rounded-down current base minus rounded-up stored base undercounts the tokens credited by Uniswap Position.update. This is most relevant for low-decimal tokens or many repeated touches.

**Impact**  
At most a small token-unit residual per token per touch can remain owed to the SFPM position in Uniswap but absent from s_accountPremiumOwed, s_accountPremiumGross, and s_settledTokens. This reduces distributed premium/fees and can strand dust in the AMM position; practical impact is bounded by token-unit rounding.

**Mitigation**  
Track and collect any residual tokensOwed before advancing the baseline, or account for actual received/remaining tokens when updating premium accumulators and feesBase.

### [I-13] Public known: small long-premium settlements can reset buyer owed dust

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: contracts/PanopticPool.sol:988-1012, contracts/PanopticPool.sol:1135-1320, contracts/PanopticPool.sol:1989-2063, contracts/SemiFungiblePositionManager.sol:1391-1464, README.md:82

**Summary/Description**  
Long premium is computed by multiplying the accumulator delta by leg liquidity and dividing by 2^64. settleLongPremium can then advance the buyer's stored premium accumulator even when that multiplication floors to zero or leaves a sub-token residual, so the unpaid dust is not carried into the next settlement period. The README already lists this as a public known issue for sufficiently small owed premium.

**Root Cause**  
The settlement snapshot is advanced to the current premium accumulator after integer-flooring the owed token amount, without carrying the fractional remainder per position.

**Pre_conditions**  
A long leg has accrued less than one token unit, or a nonzero fractional remainder below the settlement precision, and the position is settled before that remainder accumulates into a whole token unit.

**Impact**  
A buyer can avoid paying bounded sub-token premium residuals by settling frequently, reducing tokens credited to sellers. This is a real accounting behavior but is explicitly public-known in the contest README and bounded by token-unit rounding per settlement.

**Mitigation**  
Store per-position remainder/carry data or avoid advancing the buyer snapshot unless the floored premium is collected; alternatively document the rounding threshold as an intentional minimum collectible premium.

### [I-14] SFPM gross-premium floors can leave settled residual outside seller snapshots

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/SemiFungiblePositionManager.sol:266-282, contracts/SemiFungiblePositionManager.sol:1256-1347, contracts/SemiFungiblePositionManagerV4.sol:263-281, contracts/SemiFungiblePositionManagerV4.sol:1081-1172, contracts/PanopticPool.sol:1048-1049, contracts/PanopticPool.sol:1216-1225, contracts/PanopticPool.sol:2075-2121

**Summary/Description**  
The gross-premium formula is algebraically consistent with the documented Eqn 4, but the implementation floors the X64 per-liquidity accumulator while PanopticPool adds exact collected tokens to s_settledTokens. Seller withdrawals are later capped by the floored per-position premium, so collected token units can remain in s_settledTokens without being attributable to the seller snapshot that generated them.

**Root Cause**  
s_accountPremiumGross is a floored per-liquidity X64 accumulator, while s_settledTokens tracks exact collected or paid token units. The conversion back from accumulator delta to token units floors again at the seller-position level.

**Pre_conditions**  
Collected or paid premium amounts are small relative to liquidity granularity, or repeated touches occur where the gross accumulator delta multiplied by seller liquidity rounds below the exact settled token amount.

**Impact**  
Residual settled tokens can drift at chunk level and may remain unavailable after sellers close, or later improve availability for unrelated sellers in the same chunk. The effect is bounded by precision and token-unit granularity per touch and overlaps the public rounding caveat, so this is inventory-level rather than a new high-impact issue.

**Mitigation**  
Track gross residual explicitly per chunk, or carry remainder during accumulator-to-token conversion so settled tokens and seller snapshots reconcile when all liquidity exits.

s.

### [I-15] Stale PanopticPool MAX_OPEN_LEGS constant conflicts with enforced limit

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Documentation/code consistency only; no runtime enforcement impact found.  
**Location**: contracts/PanopticPool.sol:121; contracts/RiskEngine.sol:155; contracts/RiskEngine.sol:873; contracts/PanopticPool.sol:1031; contracts/PanopticPool.sol:1885

**Summary/Description**  
README and RiskEngine define the active MAX_OPEN_LEGS limit as 33, while PanopticPool still declares an internal MAX_OPEN_LEGS constant equal to 25. The PanopticPool constant is not referenced by the mint/burn enforcement path, which instead uses riskParameters.maxLegs() populated by RiskEngine.getRiskParameters().

**Root Cause**  
The local PanopticPool constant was left behind after the enforced max-leg value moved to RiskEngine-packed RiskParameters.

**Pre_conditions**  
A reviewer or integration reads PanopticPool.MAX_OPEN_LEGS as authoritative instead of tracing riskParameters.maxLegs().

**Impact**  
No protocol bypass or unintended cap was found. The risk is stale-code confusion around whether the supported total open-leg boundary is 25 or 33.

**Mitigation**  
Remove the unused PanopticPool.MAX_OPEN_LEGS constant or update it to mirror the RiskEngine source of truth with an explanatory comment.

### [I-16] Dispatch tick-delta sentinel rejects exact cumulative boundary

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/PanopticPool.sol:595-607, contracts/PanopticPool.sol:662-682

**Summary/Description**  
dispatch stores a +1 initialization sentinel in cumulativeTickDeltas.rightSlot(), then compares that slot directly against 2 * riskParameters.tickDeltaLiquidation(). Because the sentinel is counted as movement, actual cumulative tick movement equal to 2 * tickDeltaLiquidation reverts even though the comment says only movement more than twice the threshold should revert.

**Root Cause**  
The sentinel used to distinguish an initialized transient slot shares the same accumulator slot as the tick-movement sum, and the threshold comparison does not subtract or otherwise account for that sentinel.

**Pre_conditions**  
A dispatch or same-transaction sequence of dispatch calls accumulates actual tick movement exactly equal to 2 * riskParameters.tickDeltaLiquidation().

**Impact**  
The transaction reverts with PriceImpactTooLarge one tick earlier than the documented boundary. This is conservative and does not relax liquidation or collateral safety, but it can reject a boundary-exact action that the code comments describe as admissible.

**Mitigation**  
Either compare cumulativeTickDeltas.rightSlot() - 1 against the threshold after initialization, use a separate initialization flag, or update the comment/specification to state the effective cap is 2 * tickDeltaLiquidation - 1.

### [I-17] Deposits can mint zero shares after share price exceeds one raw asset per share

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/CollateralTracker.sol:284-296,520-528,547-580,651-681,795-808,860-874; contracts/libraries/Math.sol:479-489,777-792

**Summary/Description**  
CollateralTracker initializes with 1 virtual asset and 1e6 virtual shares, then uses floor division for convertToShares/previewDeposit. If share price is increased above one raw asset per share, for example by depositing assets and burning the received shares through donate(), previewDeposit(1) returns 0 while deposit(1, receiver) only rejects assets == 0 and will mint zero shares while increasing s_depositedAssets. The 1e6 virtual-share offset makes value capture economically unfavorable for the manipulator, but the state-changing deposit path still accepts a nonzero asset transfer for zero shares at the rounding boundary.

**Root Cause**  
deposit validates the asset input but never validates that previewDeposit(assets) returns nonzero shares; convertToShares floors assets * totalSupply / totalAssets, and donate can burn supply without reducing totalAssets.

**Pre_conditions**  
CollateralTracker share price has risen above the size of the depositor's raw asset amount, which is reachable by share burns such as donate() after sufficient deposited assets. The depositor or integrator then calls deposit with an amount whose floored share output is zero.

**Impact**  
A depositor who ignores previewDeposit or lacks a minimum-share guard can transfer nonzero assets and receive no shares. The direct value capture is bounded and economically disfavored by the 1e6 virtual shares, so no profitable drain was identified from this rounding edge alone.

**Proof of Concept**  
Not run; code-level review only.

**Mitigation**  
Reject deposits when the computed share amount is zero, or add a minimum-share parameter/wrapper for user-facing deposits. Keep using ceil for mint/withdraw and floor for convert/redeem where ERC4626 requires it.

### [I-18] Sub-epoch borrow changes can skip or backdate interest

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/types/MarketState.sol:14-18,59-68,164-167; contracts/CollateralTracker.sol:299,886-892,970-975,985-1024,1395-1415,1494-1519,1531-1550,1595-1610

**Summary/Description**  
CollateralTracker stores only block.timestamp >> 2 as marketEpoch and _calculateCurrentInterestState accrues borrow index/unrealized interest for uint32(currentEpoch - previousEpoch) << 2 seconds. For unchanged borrow exposure this prevents duplicate same-epoch compounding and catches up on completed 4-second buckets. When a user opens or closes borrow exposure inside a bucket, however, the sub-epoch seconds are not time-weighted: debt opened and closed within one bucket pays no borrow interest, while debt opened just before the next bucket can be charged for the full 4-second bucket on the next accrual.

**Root Cause**  
The market state records a 4-second epoch instead of an exact last-accrual timestamp, and global interest accrual uses the current s_assetsInAMM/user net-borrow state for whole epoch buckets rather than tracking exposure changes within the bucket.

**Pre_conditions**  
A position or other borrow-exposure change occurs inside a 4-second market epoch, and accrual is later triggered before or after the epoch boundary. The effect is most relevant on chains or batched flows where users can reliably place open/close actions within the same epoch.

**Impact**  
The direct value impact is bounded to less than one 4-second accrual bucket per exposure change. Timed borrowers can avoid the current partial-bucket interest by closing before the next epoch, and borrowers opening just before an epoch boundary can be overcharged on the next accrual. Because the per-change amount is small and repeated use requires high-frequency position churn with normal trading costs, this is assessed as Info.

**Proof of Concept**  
Not run; code-level review only.

**Mitigation**  
Store the exact last accrual timestamp and accrue by actual elapsed seconds, or checkpoint exposure changes so each completed bucket is attributed only to debt that existed during that bucket. If 4-second quantization is intentional, document the rounding rule and apply it consistently to previews and settlement.

### [I-19] Factory does not verify PanopticPool reference SFPM binding

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/PanopticFactory.sol:86-99,133-197; contracts/PanopticFactoryV4.sol:90-103,134-202; contracts/PanopticPool.sol:307-310,319-366

**Summary/Description**  
PanopticFactory stores its canonical SFPM and the PanopticPool reference implementation as independent constructor inputs. deployNewPool initializes/registers the AMM pool through the factory SFPM, while each PanopticPool clone later uses the SFPM immutable baked into the reference implementation for current-tick reads, approvals, and future position operations. The initializer does not verify that these two SFPM bindings are the same.

**Root Cause**  
The clone setup relies on an off-chain deployment invariant: the supplied POOL_REFERENCE must have been constructed with the same SFPM passed to the factory. The code neither exposes nor checks that binding during factory construction or pool initialization.

**Pre_conditions**  
The factory is deployed with a PanopticPool reference whose constructor SFPM differs from the factory SFPM, or with an otherwise incorrect pool reference implementation.

**Impact**  
Pools deployed by the misconfigured factory can be registered with a poolId from one SFPM but read ticks, approve assets, and execute pool logic against another SFPM. This is a deployment-mistake hazard rather than an attacker-triggered runtime exploit under intended deployment.

**Mitigation**  
Validate constructor inputs by exposing the PanopticPool reference SFPM or adding a reference self-check, and reject a factory deployment when the reference implementation is not bound to the same SFPM.

### [I-20] V4 enforced tick range uses V3 max-liquidity denominator

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/SemiFungiblePositionManagerV4.sol:349-380; contracts/SemiFungiblePositionManagerV4.sol:442-480; contracts/libraries/Math.sol:176-180; lib/v4-core/src/libraries/Pool.sol:565-580

**Summary/Description**  
V4 pool initialization and range expansion store PoolData.maxLiquidityPerTick from Math.getMaxLiquidityPerTick(key.tickSpacing), but that helper implements the V3 symmetric tick-count denominator. V4 core computes maxLiquidityPerTick with an extra negative-side tick index when MAX_TICK is not divisible by tickSpacing, so the Panoptic-stored value is higher than the V4 pool's actual per-tick cap for most V4 tick spacings.

**Root Cause**  
SemiFungiblePositionManagerV4 reuses Math.getMaxLiquidityPerTick at initialization and stores the result in PoolData, while Math.sol:176-180 matches the V3 Tick.tickSpacingToMaxLiquidityPerTick formula rather than V4 Pool.tickSpacingToMaxLiquidityPerTick at lib/v4-core/src/libraries/Pool.sol:565-580.

**Pre_conditions**  
A V4 pool uses a tickSpacing for which 887272 % tickSpacing != 0. The effect is larger for coarse custom V4 tick spacings; for example tickSpacing 32767 uses 55 ticks in the Panoptic formula but 56 in the V4 core cap.

**Impact**  
The enforced tick estimator receives an overstated maxLiquidityPerTick, so it can store min/max enforced ticks farther from zero than the same computation using the V4 pool's actual cap. Newly minted V4 positions are still checked against stored limits and tick spacing, but the range gate can be more permissive than the stated minimum fill-cost policy.

**Proof of Concept**  
Not run; analysis only per request constraints.

**Mitigation**  
Use a V4-specific maxLiquidityPerTick calculation matching Pool.tickSpacingToMaxLiquidityPerTick for SemiFungiblePositionManagerV4, or pass the canonical V4 cap into the enforced tick estimator before storing PoolData.

### [I-21] BuilderFactory accepts unusable zero builder wallet parameters

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/RiskEngine.sol:2371-2385; contracts/RiskEngine.sol:253-254; contracts/RiskEngine.sol:2315-2320

**Summary/Description**  
BuilderFactory.deployBuilder allows builderCode == 0 and builderAdmin == address(0). A zero builder code can deploy a CREATE2 wallet at salt zero, but RiskEngine._computeBuilderWallet treats builderCode zero as the no-builder sentinel and returns address(0), so that wallet is never selected by the normal fee-recipient path. A zero builderAdmin is also accepted and makes sweep impossible because sweep requires msg.sender == builderAdmin; the current unrestricted init bug can overwrite it, but after fixing that bug a zero-admin deployment would leave the wallet without an authorized sweeper.

**Root Cause**  
deployBuilder does not validate lifecycle parameters before deployment and post-deployment initialization, while RiskEngine reserves builderCode zero as a sentinel and BuilderWallet.sweep uses builderAdmin as the only authority.

**Pre_conditions**  
The BuilderFactory owner deploys a builder wallet with builderCode equal to zero or builderAdmin equal to address(0).

**Impact**  
This is primarily an admin-misconfiguration correctness issue. The zero-code wallet is unusable for intended builder commission routing, and a zero admin can leave wallet balances unsweepable once initializer access is correctly restricted.

**Mitigation**  
Reject builderCode == 0 and builderAdmin == address(0) in deployBuilder before CREATE2 deployment. Keep the init hardening from H-01 so only the factory can initialize exactly once.

### [I-22] ERC4626 previewDeposit can overestimate after pending interest accrual

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/CollateralTracker.sol:503-505,547-562,887-892,997-1025,970-975

**Summary/Description**  
CollateralTracker.previewDeposit() prices deposited assets against stored totalAssets(), but deposit() first calls _accrueInterest(), which can advance global unrealized interest before recomputing previewDeposit(assets). When pending interest exists, the state-changing deposit can mint fewer shares than previewDeposit reported for the same assets in the same transaction.

**Root Cause**  
The public ERC4626 preview path does not simulate the same pending global interest accrual that deposit executes before pricing the deposited assets.

**Pre_conditions**  
s_assetsInAMM is nonzero, time has elapsed since the last market-state update, and a caller or integrator reads previewDeposit(assets) before calling deposit(assets, receiver).

**Impact**  
Integrators can overestimate the shares minted by deposit, violating the ERC4626 previewDeposit direction. The depositor still receives shares at the post-accrual price and no direct protocol drain was identified, so this is primarily standards/integration risk.

**Proof of Concept**  
Not run; code-level review only.

**Mitigation**  
Make previewDeposit compute against a simulated current interest state, or move pricing before accrual only if the state-changing path can preserve ERC4626 preview guarantees.

w guarantees.

### [I-23] Settle-only premium path undercollects configured burn premium fee

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: contracts/PanopticPool.sol:988-1011, contracts/PanopticPool.sol:1173-1185, contracts/PanopticPool.sol:1216-1235, contracts/PanopticPool.sol:1313-1320, contracts/CollateralTracker.sol:1612-1635, contracts/RiskEngine.sol:110-116

**Summary/Description**  
When a position's premium is settled without closing the position, PanopticPool._settleOptions advances the long-leg snapshot and, for self-settlement, the short-leg snapshot, then calls CollateralTracker.settleBurn with longAmount and shortAmount both zero. If PREMIUM_FEE is configured above zero, settleBurn computes a positive premium-fee candidate from abs(realizedPremium) but also computes the notional-fee cap from longAmount + shortAmount. The zero notional inputs make the cap zero, so the min is zero and no burn premium fee is collected on premium that has just been paid or received. A later burn only sees newly accrued premium because the snapshot was advanced.

**Root Cause**  
settleBurn caps premium-fee commission by the current call's longAmount + shortAmount notional, but the settle-only premium path intentionally passes zero notional while still settling and checkpointing realized premium.

**Pre_conditions**  
The RiskEngine PREMIUM_FEE constant or equivalent packed premiumFee is set to a nonzero value. A user settles premium through dispatch's equal-size settle path or dispatchFrom's settlePremium path before closing the position.

**Impact**  
Under a nonzero premium fee configuration, users can settle premium before closing and avoid the configured burn premium fee on that settled premium. The avoided amount is repeatable and can cover all premium accrued before the final burn. Under the current contest constants, PREMIUM_FEE is zero, so this has no active economic impact today.

**Proof of Concept**  
Not run; code-level trace only, per instruction not to run tests unless requested.

**Mitigation**  
When realizedPremium is nonzero, apply the premium fee before the notional cap for settle-only calls, pass the position notional into the settle-only call, or carry settled premium into a later fee base so pre-burn settlement cannot erase fee liability under a nonzero premiumFee.

o premiumFee.

### [I-24] Collateral refund share conversion can under-settle asset deltas

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/CollateralTracker.sol:520-528,1369-1380; contracts/PanopticPool.sol:1647-1661; contracts/RiskEngine.sol:301-387

**Summary/Description**  
CollateralTracker.refund documents an asset-denominated refund amount, but both positive and negative branches convert the requested asset delta to shares with convertToShares(), which floors. In _forceExercise, refundAmounts from RiskEngine.getRefundAmounts are passed directly into ct0.refund/ct1.refund, so a nonzero exercise fee or substituted refund can transfer fewer shares than the requested asset amount represents.

**Root Cause**  
The refund path uses ERC4626 convertToShares floor rounding for a value-moving liability/compensation transfer instead of a rounding-up conversion or an exact-asset settlement primitive.

**Pre_conditions**  
A force exercise or delegated settlement produces a refund amount whose asset value is not an exact multiple of the current CollateralTracker share price. The effect is largest after share price increases, but the rounding direction exists for ordinary non-exact conversions too.

**Impact**  
The payer can under-settle the requested refund by less than one collateral share worth of assets on each affected token side. This is usually small because the vault starts with 1e6 virtual shares per virtual asset, but the code-level settlement amount is still not exact and can combine with other small-value force-exercise rounding.

**Proof of Concept**  
Not run; code-level review only, per instruction not to run tests unless requested.

**Mitigation**  
For refund liabilities, convert asset amounts with previewWithdraw()/mulDivRoundingUp or otherwise transfer enough shares to cover at least the requested asset amount. If dust forgiveness is intended, document the rounding policy explicitly.

### [I-25] RiskEngine isAccountSolvent accepts mismatched position arrays

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/RiskEngine.sol:996-1020, contracts/RiskEngine.sol:1072-1091, contracts/RiskEngine.sol:1236-1300, contracts/PanopticPool.sol:950-979, contracts/PanopticPool.sol:1728-1803

**Summary/Description**  
RiskEngine.getMargin rejects mismatched positionIdList and positionBalanceArray lengths, but RiskEngine.isAccountSolvent calls _getMargin without the same guard. Because _getTotalRequiredCollateral loops over positionBalanceArray.length and indexes positionIdList[i], a shorter balance array with a longer tokenId list causes the extra tokenIds to be ignored in a direct external view call, while the opposite mismatch reverts.

**Root Cause**  
The length check is implemented only in getMargin, not in the external isAccountSolvent entry point that reaches the same aggregation helper.

**Pre_conditions**  
A direct external caller supplies custom RiskEngine.isAccountSolvent inputs with positionBalanceArray.length less than positionIdList.length. Protocol PanopticPool callers build positionBalanceArray from the already validated positionIdList, so this is not an enforced-solvency bypass in the reviewed path.

**Impact**  
Direct integrations or monitoring that call RiskEngine.isAccountSolvent with mismatched arrays can receive a partial-position result rather than a length-mismatch failure. The core PanopticPool path remains protected by position-list validation and matched array construction.

**Proof of Concept**  
Not run; code-level review only, per instruction not to run tests unless requested.

**Mitigation**  
Add the same length check used by getMargin to isAccountSolvent before calling _getMargin.

### [I-26] RiskEngine guardian and BuilderFactory owner can be configured inconsistently

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: README.md:505-509; contracts/RiskEngine.sol:192-204; contracts/RiskEngine.sol:219-244; contracts/RiskEngine.sol:253-263; contracts/RiskEngine.sol:273-287; contracts/RiskEngine.sol:2351-2385

**Summary/Description**  
The README's trusted-role table states that the Guardian can lock/unlock pools, collect RiskEngine fees, and is the owner of BuilderFactory that deploys builder wallets. The contracts keep these authorities independent: RiskEngine stores immutable GUARDIAN and BUILDER_FACTORY constructor arguments, while BuilderFactory stores an independent immutable OWNER. RiskEngine never checks that BuilderFactory.OWNER equals GUARDIAN, and BuilderFactory.deployBuilder is gated only by OWNER. A deployment can therefore bind fee-recipient address computation to a factory controlled by a different account than the RiskEngine guardian.

**Root Cause**  
The constructor-level lifecycle binds RiskEngine to a builder factory address but does not verify or expose the factory owner relationship documented for the trusted guardian role.

**Pre_conditions**  
RiskEngine is deployed with a BUILDER_FACTORY whose BuilderFactory.OWNER is not the same address as RiskEngine.GUARDIAN, or deployment tooling accidentally supplies mismatched addresses.

**Impact**  
This does not let the factory owner call guardian-only RiskEngine functions or let the guardian bypass BuilderFactory.onlyOwner. It can still split the documented trusted role: the guardian can lock/unlock and collect RiskEngine balances but cannot deploy intended builder wallets, while a different factory owner can choose builderAdmin values for deployed builder wallets used by the fee-recipient path. This can leave builder fee routing under an unexpected authority or make incident response inconsistent with the documented role model.

**Mitigation**  
Enforce the relationship at deployment, for example by requiring BuilderFactory(BUILDER_FACTORY).OWNER() == GUARDIAN through an interface, deploying the factory from the same configuration path, or updating documentation and operational checks if the roles are intentionally separate.

### [I-27] balanceOfOrZero can return stale memory on invalid balanceOf data

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/libraries/SafeTransferLib.sol:40-67; contracts/libraries/SafeTransferLib.sol:74-100; contracts/libraries/SafeTransferLib.sol:107-122; contracts/RiskEngine.sol:281-287

**Summary/Description**  
SafeTransferLib.balanceOfOrZero is documented to return zero when balanceOf fails or returns invalid data, but after setting bal to zero for failed staticcall or short return data it unconditionally assigns bal from memory slot 0. A token with failed or malformed balanceOf return data can therefore make consumers observe stale memory instead of zero.

**Root Cause**  
The assembly validation branches do not stop execution before the final bal := mload(0), so invalid return data is not actually enforced.

**Pre_conditions**  
A caller uses balanceOfOrZero against an arbitrary or malformed token whose balanceOf call fails or returns fewer than 32 bytes. Direct in-scope consumers are transfer failure diagnostics and guardian token collection.

**Impact**  
The safeTransfer/safeTransferFrom consumers only use the value in revert metadata after a failed transfer, so user funds are not moved based on the stale value. RiskEngine.collect(token, recipient) without an amount can request an incorrect amount for arbitrary tokens, but it is guardian-only and used for sweeping incidental balances. This is therefore an Info-level correctness issue rather than a core protocol loss path.

**Proof of Concept**  
Not run per instruction; code-level path verified from the assembly and call sites.

**Mitigation**  
Only load mload(0) when staticcall succeeds and returndatasize() is at least 32 bytes; otherwise leave bal as zero or return immediately from the assembly branch.

### [I-28] Long-option collateral decay adds ratio constant as raw token amount

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/RiskEngine.sol:54-62,126-135,1505-1544,1723-1748,2102-2103

**Summary/Description**  
RiskEngine defines DECIMALS as 10,000,000 and defines ONE_BPS/TEN_BPS in that ratio scale. The long-option collateral decay branch first computes a raw-token requirement, then adds TEN_BPS directly to that raw-token value before taking min(baseRequired, decayedRequirement). Because TEN_BPS is a scaled ratio constant rather than an amount derived from amountMoved or _required, far-OTM long collateral can bottom out at a fixed 10,000 smallest units regardless of notional.

**Root Cause**  
The long-leg decay formula mixes a DECIMALS-scaled basis-point constant with raw token amounts without multiplying by the relevant notional/requirement and dividing by DECIMALS.

**Pre_conditions**  
A nonzero-width long option is evaluated far enough from strike for the decayed branch to bind, and the intended floor is proportional to notional or base requirement rather than a fixed raw-token amount. No source comment documents a fixed 10,000 raw-unit dust floor.

**Impact**  
Large far-OTM long positions can receive a collateral floor independent of position size. The impact depends on token decimals, liquidity, premium dynamics, and the intended buyer-collateral policy, so this is recorded as Info rather than a proven scalable loss.

**Proof of Concept**  
Not run; code-level review only, per instruction not to run tests unless requested.

**Mitigation**  
If the floor is intended to be ten basis points, compute it as mulDivRoundingUp(amountMoved or _required, TEN_BPS, DECIMALS). If a fixed raw-token dust floor is intended, rename/document the constant and avoid using the basis-point constant name in the raw-token expression.

### [I-29] Liquidation PremiumSettled event reports haircut instead of net settled premium

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: contracts/libraries/InteractionHelper.sol:138-151; contracts/PanopticPool.sol:1526-1580; contracts/RiskEngine.sol:735-795

**Summary/Description**  
During liquidation, PanopticPool burns positions with long-premium commitment disabled, then RiskEngine computes per-leg haircut amounts. InteractionHelper.settleAmounts commits the adjusted long premium to s_settledTokens as total premium minus haircut, but the PremiumSettled event emits only the negative haircut amount. The emitted event therefore disagrees with the committed accounting value for liquidated long premium.

**Root Cause**  
The liquidation settlement helper uses haircutPerLeg as the event payload while using premiasByLeg minus haircutPerLeg for the actual s_settledTokens update.

**Pre_conditions**  
A liquidation burns a position with nonzero long premium and haircutPremia returns a nonzero or zero per-leg haircut; the event still does not represent the same quantity as the state update unless the net settled amount happens to equal the haircut.

**Impact**  
On-chain accounting is updated with the adjusted net settled premium, so this is not a direct solvency or fund-loss issue. Off-chain indexers or accounting tools that rely on PremiumSettled can record the wrong premium amount for liquidation settlements.

**Proof of Concept**  
Not run; code-level trace only, per instruction not to run tests unless requested.

**Mitigation**  
Emit the same adjusted amount that is added to s_settledTokens, or add a distinct event field for haircut amount if both gross and haircut values are needed.

### [I-30] Zero-size zero-width positions can poison a user's position hash

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/PanopticPool.sol:629-640; contracts/PanopticPool.sol:1029-1032; contracts/PanopticPool.sol:1330-1337; contracts/PanopticPool.sol:1829-1859; contracts/SemiFungiblePositionManagerV4.sol:772-794; contracts/SemiFungiblePositionManager.sol:815-837; contracts/libraries/PanopticMath.sol:697-728

**Summary/Description**  
PanopticPool.dispatch treats a token with no existing balance as a mint and passes the user supplied positionSize into SFPM. SFPM rejects zero-size liquidity legs because _createLegInAMM reverts on zero chunk liquidity, but all-zero-width loan/credit positions bypass _createLegInAMM entirely and compute zero moved amounts. PanopticPool then calls _updatePositionsHash before storing a PositionBalance with positionSize == 0, so the user's stored position fingerprint can include a token that normal burn/settle paths later classify as unowned.

**Root Cause**  
There is no explicit positionSize > 0 check at the PanopticPool mint boundary or SFPM mint boundary, and the only effective zero-size rejection is inside the width-bearing liquidity-leg path.

**Pre_conditions**  
A user calls dispatch to mint a valid TokenId whose active legs all have width == 0, with positionSizes[i] == 0 and finalPositionIdList containing that tokenId.

**Impact**  
The affected user can create a stored positions hash containing a zero-size position. Later dispatch processing either treats the token as unowned or as a fresh mint instead of a burn, while _validatePositionList still expects the token in the list. This is primarily self-inflicted account state corruption/DoS and not a demonstrated third-party loss path.

**Proof of Concept**  
Not run; code-level review only, per instruction not to run tests unless requested.

**Mitigation**  
Reject positionSize == 0 before calling SFPM.mintTokenizedPosition, or require at least one nonzero token movement before _updatePositionsHash and PositionBalance storage. Keep the explicit check at the PanopticPool boundary so both V3 and V4 SFPM variants are covered.

### [I-31] Repay-like collateral settlement rounds debits up and credits down

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: contracts/CollateralTracker.sol:520-528,674-681,1395-1520,1531-1662; contracts/libraries/Math.sol:782-792

**Summary/Description**  
CollateralTracker._updateBalancesAndSettle computes an asset-denominated tokenToPay for mint, burn, and settle-like collateral deltas. Positive deltas burn shares with Math.mulDivRoundingUp, while negative deltas mint shares with floor Math.mulDiv. This means exact-asset repayment-like debits can burn slightly more share value than tokenToPay, and exact-asset credits can mint slightly less share value than the amount owed, with the residual retained by the vault/shareholders.

**Root Cause**  
The settlement path converts exact asset deltas into share movements with asymmetric rounding: debits ceil and credits floor, without carrying per-account residuals.

**Pre_conditions**  
A mint, burn, settleOptions, or force-exercise/liquidation burn path reaches _updateBalancesAndSettle with tokenToPay whose asset-to-share conversion has a nonzero remainder.

**Impact**  
The value captured is bounded by less than one collateral share of value per affected settlement conversion, so this is dust-level unless share price becomes very large or the pattern is repeated across many small settlements. It favors the vault rather than creating collateral undercollection.

**Proof of Concept**  
Not run; code-level review only per instruction not to run tests unless requested.

**Mitigation**  
Document the settlement rounding policy as intentional, or track residuals/settle exact asset deltas so neither side loses the fractional share remainder.

### [I-32] BuilderWallet sweep is incompatible with no-return ERC20 tokens

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/RiskEngine.sol:2301-2305; contracts/RiskEngine.sol:2319-2329

**Summary/Description**  
BuilderWallet.sweep uses a high-level IERC20 interface whose transfer function returns bool. Tokens such as USDT that perform a successful transfer with no return data can cause Solidity ABI return decoding to revert before the explicit ok check runs, unlike the core SafeTransferLib paths that accept missing return values.

**Root Cause**  
The builder wallet defines IERC20.transfer(address,uint256) as returning bool and calls it directly instead of using SafeTransferLib.safeTransfer.

**Pre_conditions**  
A no-return ERC20 or otherwise nonstandard token is sent to a BuilderWallet, and the builder admin attempts to recover it through sweep(token,to). Core commission shares are CollateralTracker ERC20Minimal shares, which do return bool.

**Impact**  
The affected no-return token balance can be stuck in the BuilderWallet. This does not affect CollateralTracker deposits, SFPM callbacks, RiskEngine.collect, or normal builder commission shares, so impact is limited to arbitrary token recovery.

**Proof of Concept**  
Not run per instruction; verified by static inspection of the high-level IERC20 return-value path.

**Mitigation**  
Use SafeTransferLib.safeTransfer in BuilderWallet.sweep, or perform a low-level call that accepts either a true boolean return or no return data.

### [I-33] Sub-asset CollateralTracker shares cannot be redeemed or donated

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/CollateralTracker.sol:503-528,651-681,795-808,817-874; contracts/libraries/Math.sol:479-489,777-792

**Summary/Description**  
CollateralTracker can leave holders with a positive share balance whose asset value floors to zero. maxWithdraw converts the owner's shares to assets with floor rounding, so such an account has maxWithdraw == 0. maxRedeem can still return the positive share balance when deposited liquidity is available, but redeem then computes previewRedeem(shares) with floor rounding and reverts when the result is zero. donate follows the same previewRedeem(shares) zero-asset check, so the holder also cannot burn the dust shares through donate.

**Root Cause**  
The exit and cleanup paths require a nonzero asset-denominated result, but share balances can be smaller than one underlying base unit because convertToAssets floors share value to raw asset units.

**Pre_conditions**  
The CollateralTracker share price makes a holder's positive share balance worth less than one raw unit of the underlying asset. This can occur after ordinary rounding or share-price increases; no exotic token behavior is required.

**Impact**  
The affected shares are below one underlying base unit of value, so no protocol drain was identified. The user-facing effect is a stuck positive share balance that cannot be withdrawn, redeemed, or donated unless combined with more shares or the share price later increases enough to make previewRedeem nonzero.

**Proof of Concept**  
Not run; code-level review only.

**Mitigation**  
Allow donate to burn zero-asset share dust, or provide an explicit dust-burn path that does not require previewRedeem(shares) to be nonzero. Keep withdraw/redeem nonzero-asset checks if zero-asset withdrawals should remain disallowed.

### [I-34] V3 pool initialization ignores false approve results

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/PanopticPool.sol:354-366; contracts/libraries/InteractionHelper.sol:39-46; contracts/tokens/interfaces/IERC20Partial.sol:4-22; contracts/CollateralTracker.sol:566-574; contracts/CollateralTracker.sol:717-724; contracts/SemiFungiblePositionManager.sol:1138-1154

**Summary/Description**  
PanopticPool initialization delegates V3 token approvals to InteractionHelper, which calls approve through IERC20Partial without a return value. This intentionally supports no-return tokens, but it also cannot detect a standards-compliant token that returns false without setting allowance. A pool can therefore initialize and accept deposits while later CollateralTracker withdrawals or SFPM liquidity operations that rely on PanopticPool allowances revert.

**Root Cause**  
The ERC20 approval adapter erases the approve return value, so false-returning approvals are treated the same as successful or no-return approvals.

**Pre_conditions**  
A V3 Panoptic pool is initialized for an underlying token whose approve call returns false without reverting and does not set the requested allowance. Users then interact with the pool before the missing allowance is discovered by an SFPM or CollateralTracker transferFrom path.

**Impact**  
For the affected pool, assets can be moved into PanopticPool by CollateralTracker.deposit, but later withdrawals from PanopticPool through CollateralTracker and SFPM operations that pull PanopticPool funds can revert due to missing allowance. The issue is limited to unusual token behavior and does not affect V4 setOperator, which returns true always in the bundled PoolManager implementation.

**Proof of Concept**  
Not run per instruction; code-level path verified statically.

**Mitigation**  
Use a low-level approve helper that accepts either no return data or a true boolean return, and reverts on an explicit false return.

### [I-35] Oracle median rounds negative residual pairs up by one tick

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/types/OraclePack.sol:250-255, contracts/types/OraclePack.sol:316-328, contracts/types/OraclePack.sol:401-417, contracts/types/OraclePack.sol:536-545, contracts/types/OraclePack.sol:613-620, contracts/RiskEngine.sol:913-929, contracts/RiskEngine.sol:951-966

**Summary/Description**  
OraclePack.getMedianTick reconstructs the median as referenceTick + (rank3_residual + rank4_residual) / 2. Because Solidity signed division truncates toward zero, a negative odd residual sum is rounded up before the reference tick is applied. For example, with referenceTick = 200000 and the two middle residuals -2 and -1, the reconstructed middle ticks are 199998 and 199999, whose floor median is 199998, but the implementation returns 199999. The returned median can therefore be one tick higher than the median of the reconstructed observations.

**Root Cause**  
The code applies integer division to the signed residual sum instead of applying a defined tick-rounding rule to the reconstructed middle ticks, and it has no negative-odd correction for Solidity's toward-zero signed division.

**Pre_conditions**  
The ordered rank-3 and rank-4 residuals have a negative odd sum, which is reachable for any sorted oracle pack with middle observations such as residuals -2 and -1 around a positive reference tick. The value is then read through getEMAs, computeInternalMedian, getOracleTicks, or rebaseOraclePack.

**Impact**  
The error is one tick, but it feeds the median used in RiskEngine safe-mode highDivergence and the getSolvencyTicks volatility fallback. Both use strict greater-than threshold checks, so boundary states can be classified with the rounded-up median instead of the true reconstructed median, potentially avoiding an all-ticks solvency fallback or one safe-mode condition at the exact one-tick edge.

**Mitigation**  
Define the oracle median tick rounding convention explicitly and implement it after reconstructing the two middle ticks or by applying a negative-odd correction to the residual sum before adding the reference tick. For Uniswap-style tick flooring, subtract one when the residual sum is negative and odd.

### [I-36] Direct zero-vegoid SFPM pools revert premium snapshots

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/SemiFungiblePositionManager.sol:330-356; contracts/SemiFungiblePositionManager.sol:800-807; contracts/SemiFungiblePositionManager.sol:1050-1059; contracts/SemiFungiblePositionManager.sol:1308-1334; contracts/SemiFungiblePositionManagerV4.sol:322-340; contracts/SemiFungiblePositionManagerV4.sol:768-770; contracts/SemiFungiblePositionManagerV4.sol:1058-1067; contracts/SemiFungiblePositionManagerV4.sol:1133-1160; contracts/PanopticFactory.sol:133-134; contracts/PanopticFactoryV4.sol:134-135; contracts/RiskEngine.sol:2291-2293

**Summary/Description**  
SFPM pool initialization accepts vegoid = 0 and embeds that byte into the pool id. Later premium snapshot updates and current-premium views divide removedLiquidity-derived spread terms by vegoid. A directly initialized zero-vegoid SFPM pool can pass the poolId checks and mint positions, but later fee collection or live premium calculation reverts once the premium formula is reached.

**Root Cause**  
initializeAMMPool does not reject the zero vegoid value even though _getPremiaDeltas treats vegoid as a denominator for the utilization spread formula.

**Pre_conditions**  
A user or wrapper directly initializes SFPM/SFPMV4 with vegoid = 0 and uses tokenIds carrying that poolId. Factory-created Panoptic pools are not affected because the factories pass riskEngine.vegoid(), and RiskEngine returns the canonical VEGOID value of 4.

**Impact**  
Affected direct-SFPM positions can become unable to auto-collect fees into premium accumulators or query current premium with an atTick value because the owed/gross premium formula divides by zero. The impact is scoped to the zero-vegoid pool namespace and does not block a separate canonical vegoid pool for the same underlying AMM.

**Mitigation**  
Reject vegoid == 0 in both V3 and V4 initializeAMMPool, or define an explicit zero-spread sentinel and branch the formula without division by zero.

### [I-37] TokenId validation accepts inactive partner for leg zero

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/types/TokenId.sol:157-160; contracts/types/TokenId.sol:473-512; contracts/RiskEngine.sol:1361-1376; contracts/RiskEngine.sol:1590-1594; contracts/RiskEngine.sol:1707-1715

**Summary/Description**  
TokenId.validate accepts a non-self riskPartner when the pointed slot decodes back to the current leg, but it does not require that pointed slot to be active. A one-leg token can set leg 0's riskPartner to inactive leg 1, 2, or 3; the inactive slot is all zeros, so riskPartner(inactive) decodes to 0 and the mutuality check passes even though the partner is not another active leg in the position.

**Root Cause**  
The mutuality check compares decoded riskPartner fields without also checking riskPartnerIndex < countLegs() or optionRatio(riskPartnerIndex) != 0. Because zero is both leg 0's index and the default inactive-leg encoding, inactive higher slots can appear reciprocal to leg 0.

**Pre_conditions**  
A tokenId has exactly one active leg at index 0 and encodes riskPartner(0) as 1, 2, or 3 while all higher leg slots remain zero.

**Impact**  
Current RiskEngine collateral logic falls back to the unpartnered requirement for this shape because the inactive partner slot has optionRatio == 0 and fails the same-ratio partner predicate. No direct collateral reduction was found in this review, but TokenId-level validation accepts a partner reference that is not actually another active leg in the position.

**Proof of Concept**  
Not run per instruction; code-level review only.

**Mitigation**  
In TokenId.validate, when riskPartnerIndex != i, require riskPartnerIndex < countLegs() or optionRatio(riskPartnerIndex) != 0 before checking reciprocity. Alternatively normalize inactive/no-partner references to self-partner only.

### [I-38] Zero-width synthetic amounts can truncate before signed bounds

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/libraries/PanopticMath.sol:706-728; contracts/libraries/PanopticMath.sol:752-769; contracts/SemiFungiblePositionManager.sol:815-837; contracts/SemiFungiblePositionManager.sol:893-897; contracts/RiskEngine.sol:1398-1406

**Summary/Description**  
PanopticMath.getAmountsMoved computes synthetic token amounts for width-zero loan/credit legs by temporarily substituting width 2, then casts Math.getAmount0ForLiquidityUp/getAmount1ForLiquidityUp results directly to uint128. If the raw computed amount exceeds uint128, the explicit cast truncates before calculateIOAmounts reaches the checked Math.toInt128 conversion. Normal width-bearing legs are bounded by the SFPM PositionTooLarge check, but width-zero legs skip that bound and still feed settlement and margin through the truncated amount.

**Root Cause**  
Width-zero loan/credit accounting reuses the liquidity amount formula but downcasts with raw uint128(...) and bypasses the nonzero-width amount0/amount1 PositionTooLarge guard.

**Pre_conditions**  
A valid all-zero-width or mixed tokenId has a strike/asset/tokenType/positionSize combination whose synthetic opposite-token amount exceeds uint128 while the temporary liquidity calculation itself does not revert.

**Impact**  
The position can be accepted and accounted using the modulo-truncated synthetic amount instead of rejecting as oversized. I did not confirm direct asset extraction because settlement, SFPM synthetic movement, and margin all appear to consume the same truncated value, but the helper no longer represents the mathematical notional encoded by the tokenId and can create nonsensical loan/credit positions.

**Proof of Concept**  
Not run; code-level review only, per instruction not to run tests unless requested.

**Mitigation**  
Use Math.toUint128/Math.toInt128 on the raw getAmount* result before packing, and apply the same int128 PositionTooLarge-style bound to width-zero synthetic amount0/amount1 paths in both V3 and V4 SFPM before settlement and margin can consume them.

### [I-39] PanopticPool accepts unrelated single ERC1155 transfers

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/PanopticPool.sol:373-385

**Summary/Description**  
PanopticPool.onERC1155Received ignores operator, from, token id, amount, data, and msg.sender, and always returns the single-transfer ERC1155 receiver selector. The comment and notice describe SFPM-only acceptance, but any ERC1155 token using the standard single-transfer callback can receive an acceptance response. The pool has no ERC1155 recovery path, so unrelated tokens sent this way can be stranded. Reviewed SFPM transfer overrides and Panoptic accounting paths; this does not move existing SFPM positions or mutate Panoptic state.

**Root Cause**  
The receiver hook implements unconditional selector acceptance instead of validating msg.sender == address(SFPM) or otherwise rejecting unsupported ERC1155 senders.

**Pre_conditions**  
A non-SFPM ERC1155 token contract performs a standard single safe transfer or mint to PanopticPool and treats the returned receiver selector as acceptance.

**Impact**  
Unrelated ERC1155 balances can be accepted by the pool and remain unrecoverable. Core Panoptic accounting is not updated by the hook, and SFPM single/batch transfers are disabled, so no protocol position-transfer or collateral-accounting impact was identified.

**Mitigation**  
Restrict onERC1155Received to calls from the configured SFPM, or add a narrowly scoped recovery path for unrelated ERC1155 tokens if broad acceptance is intentionally preserved.

