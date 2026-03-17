# CurrentSui Audit Report

## Table of Contents

### Security Findings
- [CurrentSui Audit Report](#currentsui-audit-report)
  - [Table of Contents](#table-of-contents)
    - [Security Findings](#security-findings)
    - [Invalid / Not Sherlock Valid](#invalid--not-sherlock-valid)
    - [Informational](#informational)
  - [MEDIUM-1: EMode Borrow Cap Drift Understates Group Debt Between User Touches](#medium-1-emode-borrow-cap-drift-understates-group-debt-between-user-touches)
  - [MEDIUM-2: Interest Model Update Does Not Pre-Accrue Interest Leading to Retroactive Rate Application](#medium-2-interest-model-update-does-not-pre-accrue-interest-leading-to-retroactive-rate-application)
  - [MEDIUM-3: ADL Borrow Trigger Uses Reserve-Wide Debt While Configuration Is Per EMode Group](#medium-3-adl-borrow-trigger-uses-reserve-wide-debt-while-configuration-is-per-emode-group)
  - [MEDIUM-4: Passive Liquidity Miners Can Lose Untracked Rewards When a Reward Pool Is Closed](#medium-4-passive-liquidity-miners-can-lose-untracked-rewards-when-a-reward-pool-is-closed)
  - [INVALID-1: Deposit Cap Double-Subtraction Is a Real Bug but Not Sherlock-Medium](#invalid-1-deposit-cap-double-subtraction-is-a-real-bug-but-not-sherlock-medium)
  - [INVALID-2: Liquidation Pause Inconsistency Is Not Sherlock-Valid](#invalid-2-liquidation-pause-inconsistency-is-not-sherlock-valid)
  - [INVALID-3: Oracle Refresh Tolerance Mismatch Is Not Sherlock-Medium](#invalid-3-oracle-refresh-tolerance-mismatch-is-not-sherlock-medium)
  - [INFO-1: Liquidation Uses EMA for Eligibility and Spot for Execution by Design](#info-1-liquidation-uses-ema-for-eligibility-and-spot-for-execution-by-design)
  - [INFO-2: Rate Limiter Cross-Segment Non-Netting Is Intentional Gross-Outflow Accounting](#info-2-rate-limiter-cross-segment-non-netting-is-intentional-gross-outflow-accounting)
  - [INFO-3: EMode Full-Parameter Admin Update Rebuilds Limiter State](#info-3-emode-full-parameter-admin-update-rebuilds-limiter-state)
  - [INFO-4: Borrow Guard Can Arithmetic-Abort Instead of Returning a Clean Error When No Borrowable Cash Remains](#info-4-borrow-guard-can-arithmetic-abort-instead-of-returning-a-clean-error-when-no-borrowable-cash-remains)

### Invalid / Not Sherlock Valid
- [INVALID-1: Deposit Cap Double-Subtraction Is a Real Bug but Not Sherlock-Medium](#invalid-1-deposit-cap-double-subtraction-is-a-real-bug-but-not-sherlock-medium)
- [INVALID-2: Liquidation Pause Inconsistency Is Not Sherlock-Valid](#invalid-2-liquidation-pause-inconsistency-is-not-sherlock-valid)
- [INVALID-3: Oracle Refresh Tolerance Mismatch Is Not Sherlock-Medium](#invalid-3-oracle-refresh-tolerance-mismatch-is-not-sherlock-medium)

### Informational
- [INFO-1: Liquidation Uses EMA for Eligibility and Spot for Execution by Design](#info-1-liquidation-uses-ema-for-eligibility-and-spot-for-execution-by-design)
- [INFO-2: Rate Limiter Cross-Segment Non-Netting Is Intentional Gross-Outflow Accounting](#info-2-rate-limiter-cross-segment-non-netting-is-intentional-gross-outflow-accounting)
- [INFO-3: EMode Full-Parameter Admin Update Rebuilds Limiter State](#info-3-emode-full-parameter-admin-update-rebuilds-limiter-state)
- [INFO-4: Borrow Guard Can Arithmetic-Abort Instead of Returning a Clean Error When No Borrowable Cash Remains](#info-4-borrow-guard-can-arithmetic-abort-instead-of-returning-a-clean-error-when-no-borrowable-cash-remains)

---

## MEDIUM-1: EMode Borrow Cap Drift Understates Group Debt Between User Touches

**Severity:** Medium

**Title**
EMode Borrow Cap Drift Understates Group Debt Between User Touches

**Summary**
The touched-user-only debt update will cause understated EMode group debt for the protocol and its lenders as a borrower can take additional borrows against a stale group total after other obligations accrue untouched interest.

**Root Cause**
The group total is updated with only the touched obligation delta:

```move
let current_borrow = self.assets_borrows.load_mut_by_type(coin);
let new_borrow = new_value.add(*current_borrow).saturating_sub(old_value);
*current_borrow = new_borrow;
```

That stale total is then enforced immediately as the cap source of truth:

```move
let emode_group_total_borrow = emode_group.update_asset_borrow(...);
assert!(emode_group_total_borrow <= emode.borrow().max_borrow_amount(), ...);
```

Interest accrued by other untouched obligations is never synchronized before the borrow-cap check.

**Internal Pre-conditions**
1. At least two obligations in the same EMode group borrow the same asset.
2. One or more of those obligations remain untouched while interest accrues.
3. A borrower later touches their obligation and attempts another borrow while the stale tracker still appears below the configured cap.

**External Pre-conditions**
_None._

**Attack Path**
1. User A and User B both borrow the same asset inside one EMode group.
2. Time passes and reserve interest accrues globally.
3. User B remains untouched, so their accrued interest is absent from `assets_borrows`.
4. User A borrows again.
5. The protocol refreshes only User A's debt delta, compares the stale group total to `emode_max_borrow_amount`, and allows the borrow even though real aggregate group debt is already above the intended limit.

**Impact**
The protocol's per-EMode borrow cap can be exceeded by the aggregate unrealized interest on untouched obligations. This weakens a core risk limit and can materially increase concentrated bad-debt exposure, which is sufficient for a Sherlock `Medium`.

**Mitigation**
Derive group debt from a synchronized source of truth, or re-synchronize `assets_borrows` from reserve / borrow-index state before enforcing the cap.

---

## MEDIUM-2: Interest Model Update Does Not Pre-Accrue Interest Leading to Retroactive Rate Application

**Severity:** Medium

**Title**
Interest Model Update Does Not Pre-Accrue Interest Leading to Retroactive Rate Application

**Summary**
The immediate interest-model replacement will cause borrower or protocol fee loss as the next accrual prices the entire elapsed period under the new rate curve instead of the curve that was active during that period.

**Root Cause**
The admin path replaces the model immediately:

```move
let asset = market.market_asset_borrow_mut<MarketType, CoinType>();
asset.update_interest_model<MarketType>(interest_model);
```

Later accrual uses whatever model is currently stored for the full pending interval:

```move
let interest_rate = interest_model.calc_interest(reserve.util_rate());
reserve.accrue_interest(asset.repay_fee_rate(), interest_rate, now);
```

Because there is no pre-accrual checkpoint, old-period interest is repriced by the new curve.

**Internal Pre-conditions**
1. The reserve has outstanding debt and `last_updated < now`.
2. Governance updates the asset's interest model.
3. A later user interaction triggers interest accrual.

**External Pre-conditions**
_None._

**Attack Path**
1. Borrowers open debt under interest model A.
2. Time passes without an accrual-triggering interaction.
3. Governance replaces model A with model B.
4. The next borrow / repay / liquidation / other accrual-triggering action runs `accrue_interest`.
5. The protocol prices the entire pending interval using model B, retroactively replacing the rate that should have applied under model A.

**Impact**
If the new model is lower, borrowers underpay and the protocol loses fees. If the new model is higher, borrowers overpay relative to the previously active curve. Because this directly distorts user yield / protocol fee accounting, it fits Sherlock `Medium`.

**Mitigation**
Accrue interest with the old model immediately before replacing the stored interest model.

---

## MEDIUM-3: ADL Borrow Trigger Uses Reserve-Wide Debt While Configuration Is Per EMode Group

**Severity:** Medium

**Title**
ADL Borrow Trigger Uses Reserve-Wide Debt While Configuration Is Per EMode Group

**Summary**
The reserve-wide debt trigger will cause wrongful borrow-side ADL for users in otherwise safe EMode groups as an authorized ADL caller can execute group-specific ADL once another group's debt pushes the shared reserve total above the threshold.

**Root Cause**
The ADL config is loaded per group:

```move
let timelock = *adl_registry.get_borrow_deleverage(debt_type, emode_group_id);
```

But the trigger uses reserve-wide debt:

```move
let total_debt = (*self.reserves.load_by_type(debt_type).debt()).floor();
debt_params.ensure_limit_breached(total_debt);
```

And the stop path goes back to a per-group total:

```move
let total_borrow = emode_group.borrow_amount(debt_type).floor();
adl.try_stop_borrow_deleverage<DebtType>(..., emode_group_id, total_borrow);
```

Configuration, trigger, and stop conditions are therefore scoped inconsistently.

**Internal Pre-conditions**
1. Multiple EMode groups borrow the same underlying asset.
2. Borrow ADL is configured for a specific `(emode_group_id, debt_asset)` pair.
3. Reserve-wide debt exceeds the ADL target while the target group's own debt remains below that target.

**External Pre-conditions**
_None._

**Attack Path**
1. Governance enables debt ADL for asset X in EMode group A.
2. Users in EMode group B heavily borrow asset X and drive reserve-wide debt above the configured target.
3. Group A remains below its own intended debt threshold.
4. An authorized ADL caller executes borrow ADL against a group A obligation.
5. The protocol allows the ADL because it checks reserve-wide debt, not group A debt.

**Impact**
Borrowers in one EMode group can be forcibly deleveraged because of debt accumulated by another group. That can lead to wrongful collateral seizure / forced position reduction and is a valid Sherlock `Medium`.

**Mitigation**
Use the same scope for configuration, trigger, and auto-stop. If borrow ADL is intended to be per group, compare against `emode_group.borrow_amount(debt_type)`.

---

## MEDIUM-4: Passive Liquidity Miners Can Lose Untracked Rewards When a Reward Pool Is Closed

**Severity:** Medium

**Title**
Passive Liquidity Miners Can Lose Untracked Rewards When a Reward Pool Is Closed

**Summary**
The lazy creation of obligation reward trackers will cause passive liquidity miners to lose earned rewards as admin can close a finished reward pool before those users ever get a tracker for that reward period.

**Root Cause**
New reward pools start with no tracked obligation managers:

```move
let pool_reward = PoolReward {
    ...
    num_obligation_reward_managers: 0,
    ...
};
```

Trackers are created only on later obligation updates:

```move
if (!pool_reward_manager.borrow_obligation_reward_managers().contains(obligation_id)){
    pool_reward_manager.new_obligation_reward_manager(obligation_id, clock);
};
```

Pool closure only checks whether any trackers were ever created:

```move
assert!(num_obligation_reward_managers == 0, error::liquidity_mining_not_all_rewards_claimed());
```

Passive users are therefore invisible to `close_pool_reward` if they never interact after the pool starts.

**Internal Pre-conditions**
1. A reward pool is created while eligible obligations already exist.
2. One or more of those obligations remain passive for the entire reward period.
3. Admin closes the reward pool before those obligations interact again.

**External Pre-conditions**
_None._

**Attack Path**
1. Users deposit or borrow and obtain shares that would qualify for liquidity-mining rewards.
2. Governance adds a new reward pool for that market.
3. Some users remain passive and never trigger `update_obligation_reward_manager` during the reward period.
4. The reward period ends.
5. Admin closes the pool and reclaims the remaining balance because no tracker was ever created for those passive users.

**Impact**
Passive users can lose reward yield they should have earned. This is a constrained but real loss-of-yield path and is reasonably Sherlock `Medium`.

**Mitigation**
Initialize all eligible obligations when a pool is created, or make pool closure depend on complete accounting of all obligations that had qualifying shares during the reward window.

---

## INVALID-1: Deposit Cap Double-Subtraction Is a Real Bug but Not Sherlock-Medium

**Severity:** Invalid (Sherlock)

**Title**
Deposit Cap Double-Subtraction Is a Real Bug but Not Sherlock-Medium

**Summary**
The extra reserve subtraction will cause deposit-cap undercounting and small reopen deposits to revert for depositors as they interact with a reserve that has non-zero `cash_reserve`.

**Root Cause**
The reserve-owned amount is already excluded by exchange-rate math:

```move
public(package) fun total_deposit_plus_interest<MarketType>(self: &Reserve<MarketType>): Decimal {
    self.exchange_rate().mul_u64(self.total_supply)
}
```

But the cap check subtracts reserves one more time:

```move
public(package) fun deposit_limit_breached<MarketType>(self: &Reserve<MarketType>, increment: u64, limit: u64): bool {
    let total_deposit_plus_interest = self.total_deposit_plus_interest();
    total_deposit_plus_interest.ceil() + increment - self.cash_reserve.ceil() > limit
}
```

**Internal Pre-conditions**
1. The reserve has non-zero `cash_reserve`.
2. A deposit limit is configured.
3. For the reopen path, `total_supply == 0` and the new deposit is smaller than `cash_reserve.ceil()`.

**External Pre-conditions**
_None._

**Attack Path**
1. Interest accrues and the reserve accumulates protocol reserves.
2. A depositor interacts with `deposit_limit_breached`.
3. The protocol undercounts current deposits by subtracting reserves twice.
4. Deposits can exceed the configured cap, or if `total_supply == 0`, a small reopening deposit arithmetic-aborts.

**Impact**
This is a real accounting bug, but under Sherlock it is hard to elevate to `Medium`: the cap-bypass side breaks an admin-set risk limit rather than directly causing relevant loss, and the reopen side blocks new deposits only rather than locking existing funds or disabling a clearly time-sensitive function for a week-plus.

**Mitigation**
Remove the redundant `- self.cash_reserve.ceil()` term from `deposit_limit_breached()`.

---

## INVALID-2: Liquidation Pause Inconsistency Is Not Sherlock-Valid

**Severity:** Invalid (Sherlock)

**Title**
Liquidation Pause Inconsistency Is Not Sherlock-Valid

**Summary**
The missing pause enforcement will cause incomplete liquidation pausing for governance as an authorized liquidator can still execute some liquidation paths after admin pauses an asset.

**Root Cause**
Normal liquidation checks the pause flag only on the seized collateral:

```move
let asset = self.assets.load_mut_by_type(collateral_name);
assert!(!asset.liquidation_paused(), error::liquidation_paused_for_asset());
```

The ADL paths do not perform an analogous check before calling `liquidation_inner`, so the pause semantics are not enforced consistently across liquidation modes.

**Internal Pre-conditions**
1. Governance pauses liquidation for an asset.
2. A position involving that asset remains liquidatable or ADL-eligible.
3. An authorized liquidator / ADL caller executes the liquidation flow.

**External Pre-conditions**
_None._

**Attack Path**
1. Governance uses `liquidation_paused` during an incident.
2. A position involving the paused asset becomes or remains liquidatable.
3. An authorized caller uses a liquidation path whose pause checks do not match the intended pause scope.
4. The liquidation succeeds despite the pause.

**Impact**
The inconsistency is real, but Sherlock's invalid list explicitly excludes admin-call issues where an admin action such as pausing collateral leads to unfair liquidation or similar harm. I would treat this as a fix-worthy control gap, not a Sherlock-valid submission.

**Mitigation**
Define whether `liquidation_paused` should block liquidation when an asset appears as collateral, as debt, or in either role, then enforce that rule consistently in normal liquidation and ADL flows.

---

## INVALID-3: Oracle Refresh Tolerance Mismatch Is Not Sherlock-Medium

**Severity:** Invalid (Sherlock)

**Title**
Oracle Refresh Tolerance Mismatch Is Not Sherlock-Medium

**Summary**
The hardcoded 30-second refresh window will cause raw refresh failures for oracle users as a refresh can revert even though governance configured a looser cached-price tolerance.

**Root Cause**
The raw refresh path hardcodes a 30-second staleness window:

```move
const PRICE_FEED_DELAY_SECONDS: u64 = 30;
assert!(price_updated_time >= now - PRICE_FEED_DELAY_SECONDS, oracle_error::pyth_price_too_old());
```

Cached reads, by contrast, rely on governance-configured delay tolerance, so the two oracle paths enforce different freshness rules.

**Internal Pre-conditions**
1. Governance sets `price_delay_tolerance_ms` above 30 seconds.
2. A user attempts a raw refresh.

**External Pre-conditions**
1. The latest Pyth update visible on-chain is older than 30 seconds at refresh time.

**Attack Path**
1. Governance configures a loose cached-price tolerance.
2. The on-chain Pyth update cadence or timing leaves the latest update older than 30 seconds.
3. A user attempts a raw refresh.
4. The refresh reverts because the raw path still applies the hardcoded 30-second limit.

**Impact**
This is an operational inconsistency, but it still does not show a direct relevant loss path or a clearly time-sensitive DOS strong enough for Sherlock `Medium`. Under Sherlock it should stay below the submission threshold.

**Mitigation**
Pass the configured tolerance, or a separately documented refresh-specific tolerance, into the raw Pyth staleness check.

---

## INFO-1: Liquidation Uses EMA for Eligibility and Spot for Execution by Design

**Severity:** Informational

**Title**
Liquidation Uses EMA for Eligibility and Spot for Execution by Design

**Summary**
The choice to use EMA prices for liquidation eligibility and spot prices for seizure sizing will cause spot-based liquidation execution for unhealthy borrowers as liquidations are processed during price dislocations.

**Root Cause**
Eligibility and execution intentionally use different price modes:

```move
// health / liquidation eligibility
weighted_debts > weighted_collateral
```

```move
// seizure sizing
liquidate_calculate_seize_ctokens(..., collateral_price, debt_price, ...)
```

That is a design choice rather than an implementation mistake.

**Internal Pre-conditions**
1. A position becomes liquidatable under the EMA-based check.
2. Spot and EMA prices diverge during execution.

**External Pre-conditions**
_None._

**Attack Path**
1. Market volatility causes spot and EMA to diverge.
2. A position becomes eligible for liquidation under the EMA-based safety check.
3. The protocol computes seizure amounts using spot prices.

**Impact**
This prioritizes liquidation liveness and solvency over borrower-friendly execution during dislocations. It is a design note, not a submission-grade vulnerability.

**Mitigation**
_None required._

---

## INFO-2: Rate Limiter Cross-Segment Non-Netting Is Intentional Gross-Outflow Accounting

**Severity:** Informational

**Title**
Rate Limiter Cross-Segment Non-Netting Is Intentional Gross-Outflow Accounting

**Summary**
The choice to preserve prior-segment outflow will cause gross rather than net flow accounting for users as a deposit in a later segment will not erase outflow recorded in an earlier segment.

**Root Cause**
The limiter intentionally mutates only the active segment on offsetting flow:

```move
self.current_segment_mut(clock).reduce(amount);
```

Older segments stay intact until they age out of the rolling window, so the model is gross-outflow accounting rather than fully netted cycle accounting.

**Internal Pre-conditions**
1. A user consumes outflow budget in one segment.
2. The user offsets that flow only in a later segment.

**External Pre-conditions**
_None._

**Attack Path**
1. A user withdraws or borrows in one limiter segment.
2. The segment rolls forward.
3. The user deposits or repays in a later segment.
4. Prior-segment outflow remains recorded until the segment expires.

**Impact**
This is intentional gross-outflow accounting, not a vulnerability. It only matters if governance wants a different limiter model.

**Mitigation**
_None required unless governance wants a net-flow limiter._

---

## INFO-3: EMode Full-Parameter Admin Update Rebuilds Limiter State

**Severity:** Informational

**Title**
EMode Full-Parameter Admin Update Rebuilds Limiter State

**Summary**
The full-parameter EMode update flow will cause limiter state reset for governance as admin replaces an EMode configuration with fresh limiter parameters.

**Root Cause**
The update path replaces the full EMode object, including fresh limiter configs:

```move
emode.update(
    ...
    new_emode.borrow_outflow_limiter,
    new_emode.deposit_outflow_limiter,
    ...
);
```

Because the limiters are rebuilt from new parameters, prior tracked usage is not preserved.

**Internal Pre-conditions**
1. Governance updates an EMode entry through the full replacement API.
2. The replacement includes limiter parameters.

**External Pre-conditions**
_None._

**Attack Path**
1. Governance performs a full EMode update.
2. The protocol reconstructs deposit and borrow limiters from the new parameters.
3. Historical limiter usage is cleared.

**Impact**
This is an admin-footgun / API-design note, not a permissionless bypass or Sherlock-valid issue.

**Mitigation**
Split the API or preserve limiter state when limiter parameters are unchanged.

---

## INFO-4: Borrow Guard Can Arithmetic-Abort Instead of Returning a Clean Error When No Borrowable Cash Remains

**Severity:** Informational

**Title**
Borrow Guard Can Arithmetic-Abort Instead of Returning a Clean Error When No Borrowable Cash Remains

**Summary**
The unchecked subtraction will cause an arithmetic abort for would-be borrowers as they attempt to borrow from a reserve where `cash_reserve.ceil() > cash`.

**Root Cause**
The insufficient-liquidity guard subtracts before it validates:

```move
assert!(self.cash - self.cash_reserve.ceil() > amount, error::reserve_not_enough_error());
```

If `cash_reserve.ceil() > cash`, the subtraction underflows before the intended clean error is returned.

**Internal Pre-conditions**
1. The reserve reaches a state where `cash_reserve.ceil() > cash`.
2. A user attempts a borrow.

**External Pre-conditions**
_None._

**Attack Path**
1. The reserve accumulates fractional protocol reserves while available cash becomes very small.
2. `cash_reserve.ceil()` becomes greater than `cash`.
3. A user tries to borrow.
4. The guard underflows and aborts before returning the intended clean insufficient-liquidity error.

**Impact**
This changes the error path only. It does not create new borrowable liquidity loss beyond the fact that no borrowable cash remains, so it is informational.

**Mitigation**
Guard the subtraction explicitly or reorder the check so the function returns the intended error code when `cash_reserve.ceil() >= cash`.
