### [H-01] Self-owned tax-pool units let unlock tax recycle back to the unlocking locker

**Severity**: High  
**Likelihood**: High  
**Impact**: High  
**Location**: fluid/packages/contracts/src/FluidLocker.sol:335

**Summary/Description**  
FluidLocker.unlock only checks that the staker and LP tax pools have nonzero total units, but does not prevent the unlocking locker from being the recipient of those same tax distributions. A locker owner can stake and provide enough liquidity to hold all units in both tax pools, then instant-unlock available SUP. The penalty is distributed back to the same locker and can be unlocked again, allowing repeated extraction of almost the entire locked balance while paying no meaningful tax to other participants. The same root cause affects vesting unlocks below 365 days because Fontaine streams the tax back to the owner's locker over the selected period.

**Root Cause**  
FluidLocker.unlock gates taxable unlocks with only STAKER_DISTRIBUTION_POOL.getTotalUnits() and LP_DISTRIBUTION_POOL.getTotalUnits(), while _instantUnlock and _vestUnlock distribute tax to pools that may be entirely owned by the unlocking locker itself. stake() and provideLiquidity() connect the same locker to those pools and update its units, so the tax recipient set is not disjoint from the taxpayer.

**Pre_conditions**  
1. The locker owner has available SUP to unlock. 2. The same locker stakes enough SUP to receive nonzero staker tax-pool units. 3. The same locker provides enough SUP/ETH liquidity to receive nonzero LP tax-pool units. 4. Other tax-pool units are zero or small enough that the locker recaptures most of the tax.

**Impact**  
For instant unlocks, if the locker controls both pools, each unlock transfers only the untaxed share to the recipient while the penalty returns to the locker as pool distributions. The owner can repeat unlocks on the returned balance until only MIN_UNLOCK_AMOUNT-scale dust and committed stake/LP principal remain, bypassing the intended early-unlock tax. For vesting periods below 365 days, the Fontaine tax streams return to the same locker over the vesting period and can be recursively unlocked, so the tax does not economically compensate independent stakers or LPs.

**Mitigation**  
Exclude the unlocking locker from receiving its own unlock tax, require taxable unlocks to distribute only to independent/pre-existing units, or route unlock tax through accounting that burns/escrows the payer's pro-rata pool share instead of crediting it back to the same locker. A simple totalUnits > 0 check is insufficient; the taxable path should verify effective non-self units for both tax pools or compute tax distribution net of the payer's units.

### [H-02] Staked SUP can be reused for LP while staking units remain

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: fluid/packages/contracts/src/FluidLocker.sol:421; fluid/packages/contracts/src/FluidLocker.sol:572

**Summary/Description**  
FluidLocker.provideLiquidity() approves and deposits supAmount from the locker's raw FLUID balance without checking supAmount against getAvailableBalance(). Because stake() only records _stakedBalance and leaves the SUP in the locker, the same SUP can be moved into a Uniswap position while the locker keeps staker tax-pool units for the full stale _stakedBalance.

**Root Cause**  
The LP entry path bypasses the available-balance accounting used by unlock() and stake(). getAvailableBalance() subtracts _stakedBalance, but provideLiquidity() never calls it and withdrawLiquidity() never reconciles _stakedBalance when LP principal that came from staked SUP is removed or transferred to the owner.

**Pre_conditions**  
1. Locker owner has locked SUP and UNLOCK_AVAILABLE logic is deployed. 2. The owner stakes SUP, creating _stakedBalance and staker distribution units. 3. The owner calls provideLiquidity() with supAmount that is covered by total FLUID balance but not by getAvailableBalance().

**Impact**  
The same locked SUP can be counted as staked and as LP liquidity at the same time, diluting legitimate stakers and liquidity providers in tax distributions. After TAX_FREE_WITHDRAW_DELAY, withdrawLiquidity() can transfer the SUP side of that LP position to the owner while _stakedBalance and staker units remain, leaving unbacked staking units that can continue receiving future tax distributions. While the balance is below _stakedBalance, getAvailableBalance() also underflows and blocks normal available-balance based flows for that locker.

**Mitigation**  
Require supAmount <= getAvailableBalance() before wrapping ETH or approving the position manager. More generally, every FLUID-moving path should either consume only available balance or explicitly reduce/reconcile _stakedBalance and staker units before moving staked principal into another accounting bucket.

### [H-03] Stack unit signatures can be replayed on other chains or managers

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: fluid/packages/contracts/src/EPProgramManager.sol:296

**Summary/Description**  
Stack claim signatures are verified over only the user, unit amount(s), program id(s), and nonce. The digest excludes the verifying manager, chain id, token/pool, and any domain separator, while nonces are stored only inside the current manager. A signature issued for one deployment is therefore accepted by any other manager or chain that uses the same Stack signer, the same numeric program id(s), and a lower local nonce.

**Root Cause**  
EPProgramManager._verifySignature uses eth_sign style hashes of abi.encodePacked(user, units, programIds, nonce) without binding the authorization to block.chainid, address(this), token, pool, or an EIP-712 domain. The replay guard is also local to mapping(programId => user => nonce) in each manager.

**Pre_conditions**  
1. Two FluidEPProgramManager deployments or chains use the same Stack signer and same numeric program id(s). 2. The user has a locker on the target deployment. 3. The target deployment nonce for the user/program is lower than the signed nonce. 4. The target program is funded or later becomes funded.

**Impact**  
A user can reuse one valid Stack authorization to set the same units on unintended matching deployments. Those units receive the target program stream and dilute legitimate lockers on that chain/manager. Because the README lists multi-chain deployment and every manager keeps independent nonces and funding, the same signature can be reused across each matching funded domain instead of authorizing only the intended one.

**Mitigation**  
Use an EIP-712 typed-data domain that includes chainId and verifyingContract, and sign typed structs that include the claim kind, user, program id(s), unit amount(s), nonce, and any intended token/pool context. At minimum, include block.chainid and address(this) in both single and batch signed messages, and migrate nonces so old domainless signatures cannot be reused.

### [H-04] Early LP withdrawal releases WETH value before the tax-free delay

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: fluid/packages/contracts/src/FluidLocker.sol:474; fluid/packages/contracts/src/FluidLocker.sol:478; fluid/packages/contracts/src/FluidLocker.sol:482; fluid/packages/contracts/src/FluidLocker.sol:514

**Summary/Description**  
FluidLocker.withdrawLiquidity() decreases a locker-owned Uniswap V3 position and collects the principal to the locker, but then unwraps the full WETH balance and transfers the entire native ETH balance to lockerOwner regardless of taxFreeExitTimestamps[tokenId]. Only the SUP transfer is gated by the 180-day timestamp. If the LP inventory has shifted from locked SUP into WETH before the delay, that WETH value is released immediately. withdrawDustETH() has the same unrestricted native-ETH drain and would bypass any attempted fix that leaves early-withdrawn ETH in the locker.

**Root Cause**  
The tax-free window is applied only to withdrawnSup. ETH/WETH proceeds are handled as generic dust via address(this).balance with no accounting for whether the ETH came from the owner's paired deposit, from LP inventory converted out of locked SUP, or from an early withdrawal that should remain locked.

**Pre_conditions**  
1. A locker owner provides liquidity with locked SUP and ETH. 2. The position is withdrawn before taxFreeExitTimestamps[tokenId]. 3. The position returns WETH, including cases where pool price movement has converted part of the locked SUP side into WETH.

**Impact**  
The owner can extract ETH value from the LP position before satisfying the 180-day tax-free requirement, while the code only keeps the remaining SUP in the locker. With favorable price movement or self-directed swaps around a large LP share, locked SUP value can be converted into WETH and withdrawn without using unlock() or paying the early-unlock tax. This breaks the documented condition that withdrawLiquidity is tax-free only after providing liquidity for over 6 months and that both ETH and SUP are withdrawn under that eligibility.

**Proof of Concept**  
Not run; code-level path is withdrawLiquidity(): _decreasePosition collects WETH/SUP to the locker, line 475 unwraps all WETH, line 478 transfers all ETH unconditionally, and line 482 gates only withdrawnSup on the tax-free timestamp.

**Mitigation**  
Track LP withdrawal proceeds by provenance and apply the tax-free timestamp to both assets. Before the timestamp, keep the WETH/ETH proceeds that represent LP principal in the locker or convert/tax them consistently with early SUP unlock rules. Also restrict withdrawDustETH() so it cannot withdraw ETH that came from LP withdrawal proceeds still inside the tax-free window.

### [H-05] LP entry can start and complete tax-free SUP exit while unlocks are disabled

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: fluid/packages/contracts/src/FluidLocker.sol:421; fluid/packages/contracts/src/FluidLocker.sol:449; fluid/packages/contracts/src/FluidLocker.sol:482; fluid/packages/contracts/src/FluidLocker.sol:890

**Summary/Description**  
FluidLocker.provideLiquidity() and withdrawLiquidity() are callable by the locker owner without the unlockAvailable modifier. During the initial phase, a locker owner can move locked SUP into a Uniswap V3 position and start taxFreeExitTimestamps[tokenId]. Once the 180-day timestamp has elapsed, withdrawLiquidity() transfers the withdrawn SUP to the owner even if the beacon implementation still has UNLOCK_AVAILABLE set to false.

**Root Cause**  
The phase gate is applied to unlock(), stake(), and unstake(), but the LP entry and exit paths omit unlockAvailable. The tax-free LP timer is based only on the position creation timestamp, not on the unlock phase becoming active.

**Pre_conditions**  
1. The locker beacon points to a FluidLocker implementation constructed with UNLOCK_AVAILABLE=false. 2. The locker owner has SUP in the locker and enough ETH to pair liquidity. 3. The owner can leave the LP position open until taxFreeExitTimestamps[tokenId] elapses, or at least start the 180-day timer before the unlock phase begins.

**Impact**  
Locked SUP can leave the initial lock accounting path before unlocks are enabled. If the initial phase remains active for at least 180 days after LP entry, the owner can withdraw the SUP side directly to the EOA while UNLOCK_AVAILABLE is still false, bypassing unlock() and all unlock tax. Even if governance enables unlocks earlier, the user has pre-matured part or all of the six-month LP exemption before the tax-free path should be available.

**Proof of Concept**  
Not run per instruction. Code trace: provideLiquidity() lacks unlockAvailable and sets taxFreeExitTimestamps in _createPosition(); withdrawLiquidity() lacks unlockAvailable and transfers withdrawnSup to lockerOwner once block.timestamp >= taxFreeExitTimestamps[tokenId].

**Mitigation**  
Apply the unlockAvailable phase gate to provideLiquidity(), withdrawLiquidity(), collectFees(), and withdrawDustETH() as appropriate, or prevent taxFreeExitTimestamps from starting before unlocks are enabled. LP positions created before the unlock phase should not mature the tax-free SUP exit timer.

### [M-01] cancelProgram refund recomputes mutable Superfluid buffer

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: fluid/packages/contracts/src/FluidEPProgramManager.sol:264

**Summary/Description**  
FluidEPProgramManager.startFunding transfers an initial reserve based on the then-current Superfluid buffer requirement, but cancelProgram does not store that funded reserve. It recomputes getBufferAmountByFlowRate during cancellation, after the outgoing GDA flows have been stopped. If Superfluid minimum-deposit or liquidation-period config changes while a program is active, the cancellation refund no longer equals the amount originally funded or the buffer actually released by the stopped flows.

**Root Cause**  
FluidProgramDetails stores only flow rates and timing, not the initialDeposit or buffer amount funded at startFunding. cancelProgram rebuilds the refund with the current getBufferAmountByFlowRate result at FluidEPProgramManager.sol:265-268, while Superfluid flow buffers are stored per flow and GDA buffer adjustment releases the previously stored buffer when the flow is set to zero.

**Pre_conditions**  
1. A program is funded through startFunding. 2. Superfluid governance changes the token minimum deposit or PPP liquidation period before cancellation. 3. The owner calls cancelProgram for that active program.

**Impact**  
If the current buffer requirement is higher than the value used at startFunding, cancelProgram attempts to transfer more than the program's funded reserve. With no spare manager balance this reverts and blocks the intended early cancellation path; with spare balance from other programs it returns extra funds to the treasury and can leave active program reserves underfunded for later early-end compensation. If the current buffer requirement is lower, cancellation returns too little and leaves the canceled program's treasury reserve stranded in the manager until a manual recovery path is used.

**Proof of Concept**  
Not run per instruction. Code-level trace: startFunding computes and transfers initialDeposit at FluidEPProgramManager.sol:309-313; cancelProgram recomputes the buffer at FluidEPProgramManager.sol:265-268; SuperTokenV1Library.getBufferAmountByFlowRate reads current CFA config; GDA _adjustBuffer stores and releases the actual per-flow buffer from agreement data.

**Mitigation**  
Store the exact initialDeposit, or at least the funded buffer amount, in FluidProgramDetails when startFunding succeeds and refund that stored amount on cancelProgram. For shared subsidy flows, account for actual per-program reserves separately from current aggregate Superfluid buffer configuration.

### [M-02] Zero-unit tax adjustment pool lets repeated calls redirect the missing pool's share

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: fluid/packages/contracts/src/StakingRewardController.sol:208

**Summary/Description**  
StakingRewardController.distributeTaxAdjustment() splits the controller's entire current FLUID balance on every call and ignores the actual amounts returned by Superfluid GDA distribution. The controller can hold adjustment balances because it is the admin/adjustment recipient for the tax pools. If either the staker tax pool or LP tax pool has zero total units while the other has units, Superfluid distributes an actual amount of zero to the empty pool and leaves that share in the controller. Because the next permissionless call re-splits the remaining balance instead of preserving per-pool pending amounts, the non-empty pool receives a portion of the empty pool's share on every call.

**Root Cause**  
distributeTaxAdjustment() computes stakerTaxAmount and liquidityProviderTaxAmount from FLUID.balanceOf(address(this)) and blindly calls FLUID.distribute() for both pools. Superfluid instant distribution rounds actualAmount to zero when a pool has no units, so the undistributed share remains in the controller but is not accounted as owed to that pool. A later call treats it as fresh adjustmentAmount and splits it again.

**Pre_conditions**  
1. The StakingRewardController has a positive FLUID balance, for example from GDA adjustment flow when tax/subsidy streams cannot be fully distributed after pool-unit changes or due unit granularity. 2. taxAllocation gives a positive share to both stakers and liquidity providers. 3. Exactly one of taxDistributionPool or lpDistributionPool has getTotalUnits() == 0 while the other pool has nonzero total units. 4. Any caller repeatedly calls distributeTaxAdjustment().

**Impact**  
The pool with zero units can lose most or all of the adjustment amount that should have been reserved for it once it later has participants. With the deployment split of 10% stakers and 90% LPs, if LP units are zero and staker units are nonzero, the first call pays 10% to stakers and leaves 90%; each additional call pays another 10% of the remainder to stakers. Repeating the call drives the LP-reserved balance toward zero. The symmetric case lets LPs absorb the staker share when staker units are zero.

**Proof of Concept**  
Not run per instruction. Code-level trace: FLUID.distribute() returns actualAmount <= requestedAmount; Superfluid PDPoolIndex.shift1 only sets x1 when total_units != 0, so a zero-unit pool gets actualAmount 0. distributeTaxAdjustment() ignores both return values and recomputes the next split from the whole remaining controller balance. With the deployed 10%/90% split, if LP units are zero and staker units are nonzero, repeated calls pay 10% of the remaining balance to stakers each time until the LP-reserved amount is nearly depleted; the symmetric staker-zero case pays 90% of the remaining balance to LPs each time.

**Mitigation**  
Require both distribution pools to be initialized and to have nonzero total units before distributing, or keep separate pendingStakerAdjustment and pendingLPAdjustment balances and only reduce each bucket by the actualAmount returned by FLUID.distribute(). Skip zero requested amounts and never re-split a previously failed pool allocation as fresh adjustment balance.

### [M-03] Vesting tax flows can round down against pool units and return tax to the unlock recipient

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: fluid/packages/contracts/src/Fontaine.sol:137; fluid/packages/contracts/src/Fontaine.sol:156

**Summary/Description**  
Fontaine stores the requested provider/staker tax flow rates, while Superfluid GDA may set actual pool flow rates lower than requested. terminateUnlock then calculates early-end compensation from the stored requested rates and instant-distributes those requested remaining amounts before transferring the final Fontaine balance to the unlock recipient. Any tax that was not actually delivered during the elapsed streaming interval, plus any instant-distribution residual, remains in the Fontaine and is paid to the recipient.

**Root Cause**  
Fontaine.initialize ignores the actualFlowRate returned by FLUID.distributeFlow for the provider and staker pools, and terminateUnlock multiplies the remaining time by the stored requested rates instead of accounting from actual GDA flow and distribution results. SuperTokenV1Library documents that GDA actual flow and actual instant distribution can be lower than requested depending on pool state.

**Pre_conditions**  
1. A taxable vesting unlock is created. 2. The provider or staker GDA pool state makes actualFlowRate lower than the requested Fontaine tax flow rate, or makes an instant compensation distribution settle below the requested amount. 3. terminateUnlock is called in the early-end window or after endDate.

**Impact**  
Stakers and liquidity providers receive less vesting tax than the tax formula requested. For one pool with requested rate r, actual flow a, elapsed time E, and remaining time R, termination pays roughly a*E plus the actual instant distribution of r*R; the missed (r-a)*E and any instant distribution residual are left as Fontaine balance and transferred to the unlock recipient. Waiting until endDate maximizes the recipient-returned tax, while early termination only compensates the future interval at the nominal rate.

**Proof of Concept**  
Not run per instruction. Code-level trace: SuperTokenV1Library.distributeFlow calls GDA then returns gda.getFlowRate, i.e. the actual distributor-to-pool rate, not the requested rate. GDA pool flow is floored to pool units in SemanticMoney: r1 = (requested / totalUnits) * totalUnits, so requested r=100 with totalUnits=101 sets actual flow to 0. FluidLocker._vestUnlock passes requested provider/staker rates into Fontaine.initialize; Fontaine stores those requested rates and ignores both distributeFlow return values. terminateUnlock then computes early-end compensation from the stored requested rates, performs instant distributions that may also settle below requested, and transfers the final Fontaine balance to the unlock recipient. Therefore any requested-minus-actual streamed tax for the elapsed interval, plus final distribution residuals, is returned to the recipient instead of the tax pools.

**Mitigation**  
Track actual provider and staker flow rates returned by distributeFlow and keep explicit per-pool pending tax accounting. On termination, reduce each tax bucket only by actual streamed/distributed amounts and route undistributed residuals to the intended tax pools or a tax adjustment bucket, rather than blindly transferring the final Fontaine balance to the unlock recipient.

### [M-04] Early stop lets cooled stakers capture future subsidy rewards before exiting

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Medium  
**Location**: fluid/packages/contracts/src/FluidEPProgramManager.sol:387

**Summary/Description**  
FluidEPProgramManager.stopFunding is permissionless once the early-end window opens and converts the remaining subsidy stream into an immediate distribution to the current tax distribution pool. A staker whose cooldown has already elapsed can call stopFunding at the first allowed timestamp, receive their share of the full remaining subsidy window, and then unstake immediately, taking rewards that would otherwise accrue only to lockers that stayed staked during that remaining period.

**Root Cause**  
In FluidEPProgramManager.sol:350-389, stopFunding can be called by anyone from endDate - EARLY_PROGRAM_END and distributes (endDate - block.timestamp) * subsidyFlowRate as an instant GDA payment to TAX_DISTRIBUTION_POOL. The tax pool units are controlled by FluidLocker.stake/unstake through StakingRewardController.updateStakerUnits, and FluidLocker.unstake only checks that the caller's stakingUnlocksAt has elapsed; it does not require the staker to remain staked for the compensated future window.

**Pre_conditions**  
1. A subsidized program is active with subsidyFlowRate > 0. 2. The current time reaches fundingStartDate + duration - EARLY_PROGRAM_END while block.timestamp < endDate. 3. A locker has tax-pool units and its stakingUnlocksAt timestamp is already elapsed.

**Impact**  
The exiting locker receives a pro-rata share of up to EARLY_PROGRAM_END seconds of future subsidy rewards, then can remove its staker units immediately. Remaining or later stakers receive less than they would under the intended streaming model, where rewards accrue only while units remain in the tax distribution pool. The value shift is bounded per program by the remaining subsidy compensation but can be repeated across subsidized programs and scales with the exiting staker's unit share.

**Proof of Concept**  
Not run per instruction. Code-level trace: at the first allowed stop time, subsidyEarlyEndCompensation equals EARLY_PROGRAM_END * programDetails.subsidyFlowRate. stopFunding closes the subsidy flow and calls program.token.distribute(address(this), TAX_DISTRIBUTION_POOL, subsidyEarlyEndCompensation). Because GDA instant distribution settles to current pool units, a locker with elapsed stakingUnlocksAt can call stopFunding and then FluidLocker.unstake, which updates its tax-pool units to the reduced balance after the future subsidy has already been allocated to it.

**Mitigation**  
Do not distribute future subsidy compensation to a user-controlled dynamic pool snapshot. Either keep the subsidy stream active until endDate, route the remaining subsidy to a time-weighted/accounted recipient set, or require/settle the subsidy compensation only for units that remain locked through the compensated interval. If permissionless early stop must remain, separate program flow cleanup from user-controlled subsidy reward crystallization.

### [M-05] Tax allocation updates can re-split pending adjustment rewards

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: fluid/packages/contracts/src/StakingRewardController.sol:178; fluid/packages/contracts/src/StakingRewardController.sol:208

**Summary/Description**  
StakingRewardController.distributeTaxAdjustment() applies the current taxAllocation to the controller's entire FLUID balance, while taxable unlocks and Fontaine tax streams snapshot the allocation when the tax is created. If the controller already holds adjustment FLUID, or continues receiving adjustment flow from tax distributions created under a prior allocation, any tax-pool participant can order the permissionless distributeTaxAdjustment() call around a later setTaxAllocation() transaction so that prior-allocation rewards are distributed under whichever split benefits their pool.

**Root Cause**  
setTaxAllocation() overwrites taxAllocation without settling or checkpointing existing adjustment balances, and distributeTaxAdjustment() derives both shares from FLUID.balanceOf(address(this)) and the mutable current taxAllocation instead of per-allocation or per-pool pending accounting.

**Pre_conditions**  
1. The controller has a positive FLUID adjustment balance, or active adjustment flow, generated while taxAllocation is allocation A. 2. The owner performs a valid setTaxAllocation() update to allocation B. 3. The staker and LP distribution pools are initialized and have units, so the zero-unit redistribution issue is not required. 4. A staker or LP can call distributeTaxAdjustment() before or after the allocation update is mined.

**Impact**  
Prior-allocation adjustment rewards can be shifted between stakers and liquidity providers by transaction ordering. For example, with 100 FLUID pending and an update from 10% staker / 90% LP to 90% staker / 10% LP, distributing before the update pays 10/90 while distributing after pays 90/10, shifting 80 FLUID between the pools. At 0/100 versus 100/0, the entire pending adjustment balance can move. The caller benefits only through its pro-rata units in the favored pool, but the disfavored pool's members lose the corresponding rewards.

**Proof of Concept**  
Not run per instruction. Code trace: FluidLocker._instantUnlock and _vestUnlock read getTaxAllocation() when the tax is created, and Fontaine stores the resulting provider/staker flow rates. Superfluid pool adjustment flow is paid to the pool admin, which is the StakingRewardController. Later, distributeTaxAdjustment() reads FLUID.balanceOf(address(this)) at line 209 and computes both shares from the then-current taxAllocation at lines 216-218. setTaxAllocation() at lines 178-190 can change that split without first settling or recording the old allocation, so the same pending balance is distributed differently depending only on whether distributeTaxAdjustment() is mined before or after the update.

**Mitigation**  
Bind adjustment balances to their allocation epoch or source pool and distribute each bucket with its recorded split. At minimum, make setTaxAllocation() atomically settle the current adjustment balance under the old allocation before writing the new allocation, and avoid exposing a separate permissionless ordering window for old balances.

### [M-06] Recipient-deleted vesting flow can leave SupVesting funds permanently stuck

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: High  
**Location**: fluid/packages/contracts/src/vesting/SupVesting.sol:120

**Summary/Description**  
SupVesting.emergencyWithdraw can no longer recover a vesting contract balance after the receiver deletes the active Superfluid flow and the scheduler later finalizes the schedule. The scheduler deletes its schedule even when no flow is active, while emergencyWithdraw always calls deleteVestingSchedule before sweeping SUP. Once the scheduler entry is gone, that call reverts and the remaining SUP has no recovery path.

**Root Cause**  
In SupVesting.sol:120-131 emergencyWithdraw unconditionally calls VESTING_SCHEDULER.deleteVestingSchedule(SUP, RECIPIENT, bytes("")) before transferring the contract's SUP balance. VestingSchedulerV2.deleteVestingSchedule reverts when the schedule no longer exists. VestingSchedulerV2.executeEndVesting deletes the schedule before checking whether the flow is active and only emits VestingEndFailed when the receiver already deleted the flow, leaving the sender balance in SupVesting.

**Pre_conditions**  
1. A SupVesting schedule has reached the cliff and executeCliffAndFlow has created the SupVesting -> recipient flow. 2. The recipient deletes the incoming CFA flow before the vesting end; Superfluid permits both the flow sender and receiver to delete a flow. 3. At or after the scheduler end execution window, executeEndVesting is called for the schedule. 4. The SupVesting contract still holds the unstreamed SUP balance.

**Impact**  
The remaining unstreamed SUP in that SupVesting contract becomes permanently inaccessible through the in-scope contracts. The scheduler entry has been deleted, the flow is gone, SupVesting exposes no generic sweep function, and emergencyWithdraw reverts with ScheduleDoesNotExist before transferring the balance. The stuck amount can be most of the streaming portion if the receiver deletes the flow soon after the cliff. Severity is Medium despite high fund impact because the path is receiver-initiated and self-griefing, with no direct token extraction.

**Proof of Concept**  
Not run per instruction. Code trace: SupVesting.emergencyWithdraw first calls SUP.flow(RECIPIENT, 0), then VESTING_SCHEDULER.deleteVestingSchedule, then transfers SUP.balanceOf(address(this)). SuperTokenV1Library.flow is a no-op when no SupVesting -> recipient flow exists. VestingSchedulerV2.executeEndVesting deletes vestingSchedules[agg.id] before _isFlowOngoing; if the recipient already deleted the flow, it emits VestingEndFailed and returns without moving the remaining SupVesting balance. A later emergencyWithdraw reaches deleteVestingSchedule with schedule.endDate == 0 and reverts ScheduleDoesNotExist, so the transfer to treasury never runs.

**Mitigation**  
Make recovery independent from scheduler-entry existence. For example, wrap scheduler deletion in a try/catch or check the schedule first and continue sweeping when the schedule is already absent and no active flow exists. Alternatively, add a separate admin sweep path for inactive/no-schedule vesting contracts, and consider clearing or updating factory state when a vesting is cancelled.

### [I-01] Partial unstake disconnects remaining positive stake from the staker tax pool

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/FluidLocker.sol:376

**Summary/Description**  
FluidLocker.unstake() always disconnects the locker from STAKER_DISTRIBUTION_POOL after updating units. If amountToUnstake is less than the current staked balance and the remaining balance still maps to nonzero units, the locker keeps positive staked balance and tax-pool units but becomes a disconnected GDA member. Subsequent tax distributions still allocate to those units, but rewards are not reflected in the locker's live SuperToken balance until claimAll/reconnect.

**Root Cause**  
FluidLocker.unstake() calls FLUID.disconnectPool(STAKER_DISTRIBUTION_POOL) unconditionally instead of disconnecting only when the remaining staked balance or remaining units become zero.

**Pre_conditions**  
1. Locker owner has staked enough SUP to receive nonzero staker tax-pool units. 2. Staking cooldown has elapsed. 3. Locker owner partially unstakes while leaving a remaining staked balance that still maps to nonzero units.

**Impact**  
The affected locker stops being a connected staker recipient despite retaining positive staked balance and pool units. The impact is delayed/less visible reward receipt rather than confirmed permanent loss: Superfluid GDA still accounts distributions against total units, disconnected rewards remain claimable, and ISuperfluidPool.claimAll(address(locker)) can settle them into the locker. The in-locker reconnect path is a later stake call, including stake(0), but that also resets stakingUnlocksAt and extends the cooldown.

**Mitigation**  
Only disconnect from STAKER_DISTRIBUTION_POOL when the remaining staked balance or remaining staker units are zero. Alternatively add an explicit staker tax-pool reconnect/claim helper and avoid making partial unstake transition the locker to disconnected state.

### [I-02] Duplicate startFunding overwrites program flow accounting and leaves orphan flows

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fluid/packages/contracts/src/FluidEPProgramManager.sol:299

**Summary/Description**  
startFunding can be called again for an already funded program. The second call overwrites _fluidProgramDetails[programId], while the treasury flow update is additive and the GDA program-pool flow is set to one absolute requested flow for the (manager, pool) pair. A repeated call can therefore add a new treasury inflow/deposit while replacing the program outflow and losing the previous per-program flow record. Under the README trusted owner/admin parameter assumption, this is primarily an operational correctness issue rather than a trustless user path.

**Root Cause**  
FluidEPProgramManager.startFunding does not check whether _fluidProgramDetails[programId].fundingStartDate is already nonzero before assigning new details. It then calls _updateFundingFlowRateFromTreasury with the new rate as an additive delta, but calls program.token.distributeFlow(address(this), program.distributionPool, fundingFlowRate), which sets the absolute GDA flow for that distributor/pool pair.

**Pre_conditions**  
1. Owner starts funding a program that has pool units. 2. Before stopFunding or cancelProgram clears that program, owner or governance execution calls startFunding again for the same programId with valid token allowance and flow permissions.

**Impact**  
After a repeated call, the program pool streams at only the latest fundingFlowRate while the treasury-to-manager flow includes both the previous and latest total rates. stopFunding and cancelProgram can subtract only the latest stored details, leaving the previous treasury inflow active; if subsidy was enabled on the previous call, the previous subsidy flow to the tax distribution pool also remains active. This breaks funding-flow conservation until manually repaired.

**Proof of Concept**  
Not run. Code-level trace only: Superfluid GDA stores one flow per distributor/pool pair, so distributeFlow updates the existing flow to the requested value; the manager stores only the latest FluidProgramDetails record.

**Mitigation**  
Reject startFunding when _fluidProgramDetails[programId].fundingStartDate is nonzero, or require the existing funding record to be stopped/cancelled before restarting. If top-ups are intended, compute deltas against the existing program flow and merge accounting without overwriting previous flow details.

### [I-03] Treasury changes can orphan and misapply active program funding flows

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fluid/packages/contracts/src/FluidEPProgramManager.sol:622

**Summary/Description**  
Active FluidEPProgramManager programs do not remember which treasury address created their funding stream and paid their initial reserve. If fluidTreasury is changed while a program is active, stopFunding and cancelProgram apply that program's negative funding delta to the current treasury address, not necessarily the treasury that is actually streaming to the manager. Because _updateFundingFlowRateFromTreasury clamps a negative aggregate result to zero, the call can also reduce or delete unrelated active programs' aggregate inflow from the new treasury. cancelProgram additionally transfers the reserve refund to the current treasury.

**Root Cause**  
FluidProgramDetails does not store a funding treasury. setTreasury can replace the global fluidTreasury while programs are active, and _updateFundingFlowRateFromTreasury always reads token.getFlowRate(fluidTreasury, address(this)) then applies the per-program delta to that current treasury at FluidEPProgramManager.sol:626-637. The negative branch silently sets the current treasury stream to zero instead of reverting on an accounting mismatch. cancelProgram refunds to the current fluidTreasury at FluidEPProgramManager.sol:270.

**Pre_conditions**  
1. Program A is funded while fluidTreasury is treasury A. 2. The owner updates fluidTreasury to treasury B before Program A is stopped or canceled. 3. Optional but stronger: treasury B has one or more active programs for the same token, so token.getFlowRate(treasury B, manager) is funding unrelated programs. 4. stopFunding or cancelProgram is called for Program A.

**Impact**  
The original treasury A funding flow is not reduced or deleted when Program A is stopped/canceled, because the manager no longer reads treasury A's stream. If treasury B has no active stream, the negative delta clamps treasury B's zero flow to zero and Program A's old stream remains orphaned. If treasury B has unrelated active programs, Program A's stored rate is subtracted from treasury B's aggregate stream; when Program A's rate is at least treasury B's current aggregate rate, the clamp deletes treasury B's stream entirely. The unrelated program GDA outflows remain open, so they are funded from the manager's residual balance or from treasury A's orphan stream until another trusted correction is made. For cancelProgram, Program A's initial reserve is also sent to treasury B even though treasury A funded it.

**Proof of Concept**  
Not run per instruction. Code-level trace only: startFunding uses the current fluidTreasury for transferFrom and flowFrom; setTreasury can later replace that address; stopFunding/cancelProgram call _updateFundingFlowRateFromTreasury without a per-program treasury snapshot. SuperTokenV1Library.flowFrom(..., 0) deletes the existing CFA flow when prevFlowRate > 0 and otherwise no-ops.

**Mitigation**  
Store the funding treasury in FluidProgramDetails and use that address for the matching flow-rate decrease and cancellation refund. Also make _updateFundingFlowRateFromTreasury revert when a negative delta exceeds the recorded treasury's current flow, unless an explicit reconciliation path is being used. Alternatively disallow treasury changes while any program is active unless all old flows and reserves are explicitly migrated.

### [I-04] Subsidized startFunding can underfund split GDA buffers

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fluid/packages/contracts/src/FluidEPProgramManager.sol:309

**Summary/Description**  
SuperTokenV1Library.getBufferAmountByFlowRate is not a single-flow underquote for current GDA: for the same positive flow rate, CFA returns max(minimumDeposit, roundUp(flowRate * liquidationPeriod)) while GDA stores max(minimumDeposit, actualFlowRate * liquidationPeriod), and actualFlowRate is floored to at most the requested flow. The mismatch appears in FluidEPProgramManager because calculateAllowances and startFunding reserve one aggregate CFA-style buffer for fundingFlowRate + subsidyFlowRate, then startFunding opens or updates separate GDA distribution-flow records. Since GDA applies buffers per distribution-flow hash and minimumDeposit per positive flow, the two GDA buffer deltas can exceed the single aggregate reserve pulled from the treasury.

**Root Cause**  
startFunding uses getBufferAmountByFlowRate(fundingFlowRate + subsidyFlowRate), and calculateAllowances mirrors the same aggregate calculation. The actual outgoing operations are program.token.distributeFlow(..., program.distributionPool, fundingFlowRate) and token.distributeFlow(..., TAX_DISTRIBUTION_POOL, currentSubsidyFlowRate + subsidyFlowRate). GDA _adjustBuffer computes and stores the buffer per distribution-flow hash using the actual GDA flow, so the required reserve should be based on the program-flow buffer plus the tax-flow buffer delta, not one buffer over the sum of requested rates.

**Pre_conditions**  
1. subsidyFundingRate is positive and both computed fundingFlowRate and subsidyFlowRate are positive. 2. The program pool and tax pool already have units as assumed by the README. 3. The SuperToken minimum deposit or current tax-flow buffer state makes the program-flow buffer plus tax-flow buffer delta exceed getBufferAmountByFlowRate(fundingFlowRate + subsidyFlowRate) plus the 3-day early-end cushion. 4. The manager has no unrelated surplus balance large enough to cover the shortfall.

**Impact**  
The documented MacroForwarder permission path can grant the treasury allowance produced by calculateAllowances, yet startFunding can still revert while opening the outgoing GDA flows because the manager did not pull enough upfront balance for the split GDA buffers. If unrelated manager surplus covers the shortfall, the program can start but temporarily relies on funds not reserved for that program until the GDA buffers are released.

**Proof of Concept**  
Not run per instruction. Code-level trace: SuperTokenV1Library.getBufferAmountByFlowRate delegates to CFA getDepositRequiredForFlowRate. CFA rounds flowRate * liquidationPeriod up to a 2^32 multiple and applies minimumDeposit, while GDA _adjustBuffer stores actualFlowRate * liquidationPeriod with the same minimumDeposit floor per distribution-flow hash. startFunding transfers only buffer(fundingFlowRate + subsidyFlowRate) + (fundingFlowRate + subsidyFlowRate) * EARLY_PROGRAM_END, then calls distributeFlow for the program pool and _updateSubsidyFlowRate for TAX_DISTRIBUTION_POOL. With subsidy enabled, those are distinct GDA flow records, so per-flow minimums or tax-flow buffer deltas can require more balance than the aggregate helper result.

**Mitigation**  
Compute and transfer reserves from the actual outgoing GDA buffer deltas: the program-pool flow buffer plus the tax-pool buffer delta from currentSubsidyFlowRate to currentSubsidyFlowRate + subsidyFlowRate, then add early-end compensation. Store the funded reserve used for each program so cancel/stop accounting does not recompute against a different aggregate state.

### [I-05] Program funding accounts for requested instead of live GDA rates

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fluid/packages/contracts/src/FluidEPProgramManager.sol:299

**Summary/Description**  
FluidEPProgramManager persists requested program and subsidy flow rates, then uses requested rates and requested stopFunding compensation amounts even though GDA distribution flows and instant distributions can settle to lower actual values. The difference can remain unassigned in FluidEPProgramManager or, for flow adjustment paths, in the pool admin, while later treasury deltas, events, stop/cancel settlement, and subsidy aggregation keep using the requested accounting.

**Root Cause**  
startFunding stores fundingFlowRate and subsidyFlowRate before opening the GDA flows and ignores the actual flow rate returned by distributeFlow. stopFunding also ignores the actualAmount returned by program.token.distribute for earlyEndCompensation and subsidyEarlyEndCompensation. Later stopFunding/cancelProgram pass the stored requested subsidyFlowRate as a negative delta into _updateSubsidyFlowRate, which adds that requested delta to getFlowDistributionFlowRate live GDA rate and clamps any negative result to zero instead of reconciling per-program actual subsidy contributions. cancelProgram also subtracts the stored requested aggregate from the treasury flow and transfers only buffer plus EARLY_PROGRAM_END times the stored aggregate rate, without reconciling current GDA flow, instant-distribution residuals, pool adjustment flow, or accrued surplus caused by live flow drift.

**Pre_conditions**  
1. A program or subsidy pool has total GDA units that do not divide the requested flow or stop compensation amount, total units exceed the requested value, or program-pool units are changed after funding starts. 2. The program is started with nonzero pool units as required by startFunding. 3. The program is later stopped or cancelled before the accumulated live-rate surplus and requested-minus-actual distribution residuals are explicitly recovered.

**Impact**  
Lockers or stakers can receive less than the funded amount while the manager records and settles against requested values. On stopFunding, even an on-time early stop can leave requested-minus-actual early-end compensation in FluidEPProgramManager because GDA instant distribution floors the amount to pool units or distributes zero when there are no units. In the aggregate subsidy path, stopping or cancelling one subsidized program can subtract its requested subsidy rate from a lower live aggregate tax-pool flow; _updateSubsidyFlowRate then rounds the remaining flow down or clamps it to zero, underfunding still-active subsidy programs until another trusted funding update repairs the aggregate stream. In cancelProgram, when the live delivered program flow is lower than the stored fundingFlowRate, the function refunds only the upfront buffer and early-end reserve and leaves the streamed difference in FluidEPProgramManager; for tax-pool adjustment remainders the value can sit in StakingRewardController. These balances are not attributed to the cancelled or stopped program and generic recovery paths can interfere with reserves for other active programs.

**Proof of Concept**  
Not run per instruction. Code-level trace: GDA distributeFlow returns the actual flow at SuperTokenV1Library.sol:1540-1551, and SuperfluidPool/SemanticMoney floors pool flows and instant distributions to total units. FluidEPProgramManager.startFunding ignores the returned actual flow at FluidEPProgramManager.sol:320 and stores requested rates at lines 299-305. stopFunding computes requested earlyEndCompensation and subsidyEarlyEndCompensation at lines 362-366, calls distribute at lines 382-390, and ignores actualAmount, so requested-minus-actual remains in the manager. On stop/cancel, _updateSubsidyFlowRate reads the current live aggregate at lines 651-655, subtracts the stored requested subsidy delta, and if the result is negative calls distributeFlow(..., 0) at lines 658-663, closing the shared tax-pool flow even when other programs remain active. cancelProgram also closes flows at lines 251-262, then transfers only buffer plus EARLY_PROGRAM_END times the stored requested aggregate at lines 264-270.

**Mitigation**  
Store and account with actual GDA flow rates returned by distributeFlow, and track per-program reserve plus accrued surplus separately from requested rates. For stopFunding instant compensation, reduce each owed bucket only by the actualAmount returned by distribute and keep the residual attributed to the same program or pool. For subsidy aggregation, track each program actual subsidy contribution and subtract actual rates rather than requested deltas; if a negative aggregate would occur, revert or reconcile the accounting instead of silently clamping to zero. On cancelProgram, reconcile current program-pool flow, instant residuals, and adjustment state before refunding, then return or explicitly account for all program-attributable undistributed funds.

### [I-06] Locker implementation snapshots unset or mismatched pool immutables

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/StakingRewardController.sol:194; fluid/packages/contracts/src/StakingRewardController.sol:222; fluid/packages/contracts/src/FluidLocker.sol:228; fluid/packages/contracts/src/FluidLocker.sol:341; fluid/packages/contracts/src/FluidLocker.sol:504; fluid/packages/contracts/src/Fontaine.sol:110; fluid/packages/contracts/src/Fontaine.sol:137

**Summary/Description**  
StakingRewardController.initialize creates only the staker tax pool and leaves the LP pool to the later owner-only setupLPDistributionPool step. FluidLocker and Fontaine snapshot that LP pool into immutables, so an implementation deployed or upgraded before LP setup, or against a stale controller, keeps a zero or wrong LP pool even after setup is later called. StakingRewardController.distributeTaxAdjustment also assumes lpDistributionPool is already set. The normal deployment script calls setupLPDistributionPool before deploying the locker and Fontaine implementations, so this remains setup/upgrade dependent; collectFees can also mislabel token returns after a mismatched immutable upgrade.

**Root Cause**  
LP pool creation is a separate owner step from controller initialization, while downstream constructors and distribution functions assume the LP pool already exists. FluidLocker and Fontaine do not validate that the cached pool addresses are nonzero Superfluid pools for FLUID, distributeTaxAdjustment has no unset-pool guard, and later setupLPDistributionPool cannot refresh already deployed implementation immutables.

**Pre_conditions**  
A trusted deployer or upgrade operator deploys FluidLocker or Fontaine before setupLPDistributionPool, deploys them against a stale or mismatched StakingRewardController, or calls distributeTaxAdjustment while lpDistributionPool is still unset. The normal Deploy.s.sol path sets the LP pool before deploying those implementations, so the issue requires nonstandard setup or upgrade ordering.

**Impact**  
Taxable locker unlocks can fail when the cached LP_DISTRIBUTION_POOL is zero or wrong, because unlock checks LP_DISTRIBUTION_POOL.getTotalUnits before tax distribution and _instantUnlock/_vestUnlock route the LP tax through the cached pool. Fontaine instances deployed with a bad provider pool fail or misroute vesting tax flow setup. distributeTaxAdjustment can also fail before LP pool setup when it tries to distribute to the unset LP pool. Existing affected implementations are not repaired by calling setupLPDistributionPool later; governance must deploy and upgrade to a fresh implementation. Because the normal deployment path is ordered correctly and the bad state depends on trusted setup or upgrade inputs, severity remains informational.

**Proof of Concept**  
Not run per instruction. Code trace: StakingRewardController.initialize creates only taxDistributionPool at line 130; setupLPDistributionPool later creates lpDistributionPool at line 204; distributeTaxAdjustment distributes to lpDistributionPool at line 222 without checking it is set. FluidLocker snapshots lpDistributionPool at line 228, checks LP_DISTRIBUTION_POOL.getTotalUnits for taxable unlocks at line 341, and distributes instant LP tax at line 600. Fontaine snapshots providerDistributionPool at line 111 and starts provider tax flow at line 137. Superfluid GDA rejects non-pool destinations in distribute when _isPool is false.

**Mitigation**  
Create the LP distribution pool during controller initialization, or make locker and Fontaine deployment impossible until both tax pools are nonzero and verified as FLUID Superfluid pools. Add explicit unset-pool guards to distributeTaxAdjustment and deployment scripts, and prefer reading current pool addresses from the controller only when upgrade compatibility is safe. For beacon upgrades, validate immutable compatibility with existing locker storage and Uniswap positions; collectFees should sort by the actual position token order.

### [I-07] Program creation does not enforce the locker token

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fluid/packages/contracts/src/FluidEPProgramManager.sol:199

**Summary/Description**  
FluidEPProgramManager.createProgram accepts any nonzero ISuperToken and stores that token's pool as the program pool. FluidLocker later calls FLUID.connectPool(programPool), but Superfluid GDA derives authorization from pool.superToken(): if the wrong-token pool uses the same host, the locker can connect and receive the wrong SuperToken, which FluidLocker cannot unlock or account as FLUID; if it uses a different host, connection reverts. Under the README trusted owner/admin parameter assumption, this remains an operational correctness issue rather than a contest-grade trustless path.

**Root Cause**  
createProgram validates only that token is nonzero before creating and storing the pool. It does not enforce token == FLUID even though FluidLocker accounting, unlocks, staking, and liquidity paths are hardwired to the immutable FLUID token. SuperTokenV1Library.connectPool does not itself enforce that the library receiver token equals pool.superToken(); the downstream GDA authorizes against pool.superToken() and host compatibility.

**Pre_conditions**  
1. Owner calls createProgram with a valid SuperToken that is not the lockers' FLUID token. 2. Users later try to claim or connect to that program through FluidLocker.

**Impact**  
A wrong-token program pool is permanently associated with the programId because duplicate creation is blocked. For a same-host SuperToken, locker claim/connect can attach the locker to the wrong-token pool and program funding can stream that wrong token to the locker, but FluidLocker exposes no accounting or withdrawal path for it as FLUID. For a different-host pool, the connect path reverts. In both cases the normal FLUID locker program path is broken until governance deploys or upgrades a repair path.

**Proof of Concept**  
Not run. Code-level trace only: createProgram stores the caller-supplied token/pool; FluidLocker calls FLUID.connectPool(programPool); SuperTokenV1Library forwards only the pool to gda.connectPool; GeneralDistributionAgreementV1.connectPool reads token = pool.superToken() and AgreementLibrary.authorizeTokenAccess requires token.getHost() == msg.sender. Same-host wrong-token pools can therefore connect, while different-host pools revert.

**Mitigation**  
Store the intended FLUID token in FluidEPProgramManager and require address(token) == address(FLUID) in createProgram, or expose a validated token registry plus matching locker accounting/withdrawal support if multiple SuperTokens are intentionally supported.

### [I-08] Positive startFunding can create zero-rate active programs

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/FluidEPProgramManager.sol:296

**Summary/Description**  
FluidEPProgramManager.startFunding divides the funding and subsidy amounts by programDuration before validating that the duration and resulting flow rates are usable. A zero programDuration reverts through Solidity division-by-zero, while a positive duration larger than the positive amount can floor both computed flow rates to zero. The call then records the program as funded and emits positive funded amounts even though no treasury flow, GDA flow, deposit, or later compensation is created.

**Root Cause**  
At FluidEPProgramManager.sol:296-297, fundingFlowRate and subsidyFlowRate are computed with integer division and there is no explicit programDuration > 0 check or nonzero effective-flow check before _fluidProgramDetails is persisted at lines 299-305.

**Pre_conditions**  
1. Owner calls startFunding for an existing program whose pool has units. 2. programDuration is zero, or programDuration is greater than the positive funding/subsidy amount being converted into a per-second flow. 3. For the non-reverting zero-rate case, the computed fundingFlowRate plus subsidyFlowRate is zero.

**Impact**  
For programDuration == 0, the transaction reverts without a project-specific validation error. For positive durations that floor both rates to zero, the program becomes active but streams nothing, pulls no initial deposit, and stopFunding/cancelProgram later clear it without distributing compensation. If only the subsidy component floors to zero, the event can report a positive subsidyAmount while stakers receive no subsidy flow. Under the README trusted owner/admin parameter assumption and SUP 18-decimal amounts, this is primarily an operational correctness issue with dust-sized practical bounds for normal inputs.

**Proof of Concept**  
Not run per instruction. Code-level trace: totalAmount is split at lines 292-293, both components are divided by programDuration at lines 296-297, zero rates are stored at lines 299-305, initialDeposit becomes zero at lines 309-310, flowFrom(0) and distributeFlow(0) are no-ops, and later stopFunding computes zero compensation from the stored zero rates.

**Mitigation**  
Validate programDuration > 0 before division and require fundingFlowRate + subsidyFlowRate > 0 for positive totalAmount. If exact total distribution is required, account for amount % duration remainders explicitly, and require a nonzero subsidy flow whenever a positive subsidyAmount is expected.

### [I-09] Short program durations overfund early-end reserve

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/FluidEPProgramManager.sol:307

**Summary/Description**  
FluidEPProgramManager.startFunding always pulls an early-end reserve equal to EARLY_PROGRAM_END times the requested aggregate flow, while stopFunding opens once block.timestamp reaches endDate - EARLY_PROGRAM_END. If a trusted start uses a programDuration shorter than EARLY_PROGRAM_END, that boundary is before funding starts, so stopFunding is callable immediately and the reserve can exceed the whole scheduled program amount.

**Root Cause**  
startFunding does not require programDuration >= EARLY_PROGRAM_END before computing the upfront reserve at FluidEPProgramManager.sol:309-310, and stopFunding does not special-case boundaries earlier than fundingStartDate at FluidEPProgramManager.sol:348-352.

**Pre_conditions**  
1. Owner starts a program with 0 < programDuration < EARLY_PROGRAM_END. 2. The program has pool units and the treasury grants the large reserve allowance produced by the current formula. 3. stopFunding is called at or after funding start.

**Impact**  
For short programs, the earliest stop boundary has already passed at funding time. An immediate stop distributes only programDuration * flowRate as compensation while at least (EARLY_PROGRAM_END - programDuration) * flowRate plus released buffer remains in the manager; a stop at the actual end can leave the full 3-day reserve. This is mainly an operational overfunding/stranded-balance issue under the README trusted owner-parameter assumption.

**Proof of Concept**  
Not run per instruction. Code-level trace: startFunding transfers buffer + flowRate * EARLY_PROGRAM_END regardless of duration, emits an earlyEndDate that can be before fundingStartDate, and stopFunding permits calls whenever block.timestamp is not less than endDate - EARLY_PROGRAM_END.

**Mitigation**  
Require programDuration >= EARLY_PROGRAM_END for funded programs, or cap the upfront early-end reserve and maximum compensation window to min(programDuration, EARLY_PROGRAM_END).

### [I-10] stopFunding after the early boundary leaves unused early-end reserve

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/FluidEPProgramManager.sol:362

**Summary/Description**  
FluidEPProgramManager.startFunding pulls an upfront reserve for the Superfluid buffer plus EARLY_PROGRAM_END seconds of aggregate funding and subsidy flow. stopFunding only distributes the remaining time until endDate as early-end compensation, so any call after the first allowed early-stop timestamp leaves the elapsed portion of that reserve, plus released buffer, idle in the manager; a call at or after endDate leaves the full reserve.

**Root Cause**  
startFunding funds buffer + uint96(fundingFlowRate + subsidyFlowRate) * EARLY_PROGRAM_END at FluidEPProgramManager.sol:309-313. stopFunding opens at endDate - EARLY_PROGRAM_END, but compensation is calculated as (endDate - block.timestamp) * each stored flow rate at FluidEPProgramManager.sol:362-366 and there is no branch that refunds or otherwise accounts for the unused reserve when block.timestamp is later than the first allowed stop timestamp.

**Pre_conditions**  
1. A program is started and the treasury funds the upfront reserve. 2. stopFunding is called after fundingStartDate + duration - EARLY_PROGRAM_END instead of exactly at that first allowed timestamp. 3. The call can be before, at, or after endDate; the unused reserve grows with the delay and is maximized at endDate.

**Impact**  
Program and subsidy recipients are not underpaid by this branch because they have already received the elapsed early-window stream and then receive remaining-time compensation. However, the treasury-funded reserve is not conserved per program: each late stop leaves approximately (block.timestamp - (endDate - EARLY_PROGRAM_END)) * (fundingFlowRate + subsidyFlowRate), capped at EARLY_PROGRAM_END * aggregate flow, plus released Superfluid buffer state, unassigned in FluidEPProgramManager. The generic recovery path is emergencyWithdraw, which sweeps the full token balance and is unsafe as normal per-program settlement while other programs still rely on their reserves.

**Proof of Concept**  
Not run per instruction. Code-level trace: startFunding transfers initialDeposit and opens equal incoming treasury and outgoing program/subsidy flows. At stop time t in [endDate - EARLY_PROGRAM_END, endDate], stopFunding closes the flows and pays only (endDate - t) * fundingFlowRate plus (endDate - t) * subsidyFlowRate. The elapsed early-window portion (t - (endDate - EARLY_PROGRAM_END)) * aggregate flow remains in the manager; at t >= endDate the compensation branch is skipped and the whole early-end reserve remains.

**Mitigation**  
Track the funded initial deposit or reserve per program. In stopFunding, after closing flows and paying remaining-window compensation, refund the unused reserve and released buffer to the treasury or explicitly account for it as a per-program residual. Avoid relying on emergencyWithdraw for normal settlement.

### [I-11] calculateAllowances can overstate treasury permissions after subsidy split truncation

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/FluidEPProgramManager.sol:587

**Summary/Description**  
FluidEPProgramManager.calculateAllowances computes the treasury CFA and token allowances from plannedFundingAmount / plannedProgramDuration, while startFunding first splits totalAmount into fundingAmount and subsidyAmount and then divides each component by programDuration. Because floor((fundingAmount + subsidyAmount) / duration) can exceed floor(fundingAmount / duration) + floor(subsidyAmount / duration) by one flow-rate unit, the MacroForwarder path can grant more CFA allowance and deposit approval than startFunding actually consumes.

**Root Cause**  
calculateAllowances does not mirror startFunding's split-then-divide arithmetic. It calculates flowRateAllowance from the unsplit total at line 587, but startFunding uses the sum of separately rounded fundingFlowRate and subsidyFlowRate at lines 292-310 and consumes only that sum in _updateFundingFlowRateFromTreasury.

**Pre_conditions**  
1. subsidyFundingRate is between 0 and 10000 so totalAmount is split into two components. 2. The remainders of fundingAmount / programDuration and subsidyAmount / programDuration sum to at least programDuration, so the unsplit floor division is one unit higher than the split floor divisions. 3. The treasury uses paramsGivePermission or equivalent calculateAllowances output before startFunding.

**Impact**  
The calculated CFA allowance is conservative, so it does not underpermission startFunding. However, the treasury can leave one extra wei-per-second of flow allowance per permission grant, and the ERC20 allowance can also exceed the initialDeposit that startFunding pulls. In the edge case where totalAmount / duration is one but both split components individually round to zero, permissions are granted even though startFunding opens zero-rate flows. Under the README trusted owner/admin parameter assumption this is an operational least-privilege issue.

**Proof of Concept**  
Not run per instruction. Algebraic trace: let S = subsidyAmount, F = fundingAmount, D = programDuration, and S + F = totalAmount. calculateAllowances returns floor((S + F) / D). startFunding consumes floor(S / D) + floor(F / D). The difference is floor(((S % D) + (F % D)) / D), which is either 0 or 1. Example: totalAmount = D, subsidyFundingRate = 5000, S = F = D/2. calculateAllowances returns 1, while startFunding's fundingFlowRate and subsidyFlowRate are both 0.

**Mitigation**  
Have calculateAllowances reproduce startFunding's split arithmetic using the current subsidyFundingRate, and compute depositAllowance from the same aggregate rate that startFunding will transfer and use. Alternatively return both exact required allowances and an explicit optional safety margin.

### [I-12] Vesting unlock division remainder bypasses the streamed tax path

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/FluidLocker.sol:72

**Summary/Description**  
calculateVestUnlockFlowRates floors amountToUnlock / unlockPeriod before deriving the unlock and tax flow rates. The combined stream rate therefore pays only floor(amountToUnlock / unlockPeriod) * unlockPeriod over the nominal unlock period, leaving amountToUnlock % unlockPeriod in the Fontaine instead of streaming it through the recipient/tax split.

**Root Cause**  
The vesting unlock flow formula performs division before allocating the full amount across recipient and tax streams, and there is no explicit remainder allocation for amountToUnlock % unlockPeriod.

**Pre_conditions**  
1. Locker owner performs a vesting unlock with amountToUnlock not divisible by unlockPeriod. 2. The Fontaine is later terminated, or otherwise retains the unstreamed remainder until termination.

**Impact**  
The remainder is not permanently lost: Fontaine.terminateUnlock stops the streams and transfers leftover balance to the unlock recipient. For taxable unlocks, however, the remainder bypasses the tax split and is ultimately paid to the recipient. The maximum effect is less than unlockPeriod wei per unlock, capped below 31,536,000 wei by the 365 day maximum period, and the minimum unlock amount keeps the relative impact negligible.

**Mitigation**  
Allocate the division remainder explicitly, for example by adding it to the intended tax/recipient terminal payment according to the unlock percentage, or by documenting that sub-period wei dust is intentionally paid to the recipient on termination.

### [I-13] Max-period unlock can be completed one day before the tax-free boundary

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/Fontaine.sol:145

**Summary/Description**  
A 365-day unlock is correctly assigned zero tax, but the Fontaine can be terminated starting at endDate - 1 days. Because max-period unlocks have zero provider and staker tax flow, terminating in this early window sends the remaining Fontaine balance to the recipient and lets the user complete the documented 365-day tax-free unlock after only 364 days.

**Root Cause**  
FluidLocker.unlock exempts only unlockPeriod == 365 days from tax by skipping tax-pool checks and producing taxFlowRate == 0, while Fontaine.terminateUnlock permits finalization when block.timestamp >= endDate - EARLY_END and transfers all leftover balance to the recipient after stopping zero tax flows.

**Pre_conditions**  
1. Locker owner starts a vesting unlock with unlockPeriod exactly 365 days. 2. Time reaches block.timestamp == fontaine.endDate() - 1 days. 3. Anyone calls terminateUnlock on the Fontaine.

**Impact**  
Stakers and liquidity providers receive no tax even though the practical completion time is 364 days. The avoided tax is bounded to the tax curve's final-day boundary, e.g. a 364-day selected unlock would have 11 bp tax and a 365 days - 1 second selected unlock would have 1 bp tax, so this is a low-impact boundary leak rather than a broad tax bypass.

**Proof of Concept**  
Not run per instruction. Code-level trace: unlockPeriod == 365 days gives getUnlockingPercentage() == 10000 and taxFlowRate == 0; Fontaine.endDate is block.timestamp + unlockPeriod; terminateUnlock is allowed when block.timestamp is not less than endDate - 1 days and then transfers leftoverBalance to recipient because provider/staker compensation is zero.

**Mitigation**  
Either require max-period tax-free Fontaines to reach endDate before leftover transfer, or calculate tax/exemption from the earliest permitted completion time by adding EARLY_END to the tax-free threshold.

### [I-14] Sub-minimum locker balances cannot be fully unlocked

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/FluidLocker.sol:320

**Summary/Description**  
FluidLocker.unlock rejects every unlockAmount below MIN_UNLOCK_AMOUNT before considering whether the requested amount is the locker's full available balance. A locker owner with a positive available balance below 10 SUP therefore cannot exit that balance through the normal unlock path.

**Root Cause**  
The minimum unlock check is an absolute lower bound and has no exception for unlockAmount == getAvailableBalance(), nor does unlock enforce that partial unlocks leave either zero balance or another unlockable balance.

**Pre_conditions**  
1. UNLOCK_AVAILABLE is true. 2. A locker has available SUP greater than zero and less than MIN_UNLOCK_AMOUNT. This can be a final program-distribution tail, a small total allocation, or a residual left after earlier actions. 3. The locker owner attempts to unlock the full available balance.

**Impact**  
The owner cannot unlock the sub-minimum balance through FluidLocker.unlock even when requesting the entire available amount. The funds are not globally lost because the owner can wait for or add more SUP to reach the minimum, or use the LP path with extra ETH and the tax-free delay, but the normal exit path can leave less than 10 SUP unavailable per locker occurrence.

**Proof of Concept**  
Not run. Code-level trace: FluidLocker.unlock reverts on unlockAmount < MIN_UNLOCK_AMOUNT at line 320 before the available-balance check at line 330, so unlock(getAvailableBalance(), ...) reverts whenever 0 < getAvailableBalance() < 10 ether.

**Mitigation**  
Compute availableBalance first and allow unlockAmount == availableBalance even when it is below MIN_UNLOCK_AMOUNT. Also consider rejecting partial unlocks that would leave a nonzero available balance below MIN_UNLOCK_AMOUNT.

### [I-15] Instant unlock LP rounding is reassigned to stakers

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/FluidLocker.sol:599

**Summary/Description**  
FluidLocker._instantUnlock uses the actual LP distribution amount, not the requested LP allocation, when calculating the staker distribution request. If Superfluid GDA rounds the LP distribution down, the LP adjustment amount is requested from the staker pool instead of being preserved for LPs or handled as explicit adjustment accounting.

**Root Cause**  
In FluidLocker._instantUnlock, the LP request is penaltyAmount * providerAllocation / BP_DENOMINATOR, but the staker request is penaltyAmount - actualProviderDistributionAmount. Superfluid GDA actualAmount can be smaller than requestedAmount because pool distributions are floored to total pool units, so any requested-minus-actual LP delta is folded into the staker request.

**Pre_conditions**  
1. Locker owner performs an instant unlock. 2. Staker and LP pools both have nonzero total units, satisfying the unlock guard. 3. The requested LP tax amount is not an exact multiple of LP_DISTRIBUTION_POOL.getTotalUnits(), or is lower than that unit total.

**Impact**  
The recipient transfer is conservation-safe because it subtracts only the two actual distributed amounts, so no FLUID is stranded and the user does not receive the LP shortfall unless the staker distribution also rounds down. However, LPs can receive less than the configured tax allocation while stakers receive the LP rounding adjustment. The final recipient-side leftover is limited to the second distribution's rounding remainder, making this an allocation correctness issue rather than a stronger tax-bypass finding.

**Proof of Concept**  
Not run per instruction. Code-level trace: GDA computes actualAmount as floor(requestedAmount / totalUnits) * totalUnits. If LP requested is 7.2e18 and LP total units leave a nonzero remainder, actualProviderDistributionAmount is lower than the configured LP allocation. _instantUnlock then calls the staker pool with penaltyAmount - actualProviderDistributionAmount, so the LP remainder is redirected into the staker distribution request.

**Mitigation**  
Compute providerRequested and stakerRequested from the configured allocation independently, and handle each requested-minus-actual adjustment explicitly. If the intended behavior is to collect the full penalty even when LP rounding occurs, document that LP GDA adjustment is intentionally redirected to stakers; otherwise retain pending LP adjustment or route all adjustment through StakingRewardController.distributeTaxAdjustment with per-pool buckets.

### [I-16] Vesting unlock provider flow is rounded down to stakers

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/FluidLocker.sol:634

**Summary/Description**  
FluidLocker._vestUnlock computes the provider tax flow with truncating integer division, then assigns the full residual taxFlowRate to the staker flow. Whenever taxFlowRate * providerAllocation is not divisible by BP_DENOMINATOR, the requested provider/staker stream split deviates from the configured basis-point allocation before any downstream Superfluid pool rounding.

**Root Cause**  
In FluidLocker._vestUnlock, providerFlowRate is floor(taxFlowRate * providerAllocation / BP_DENOMINATOR), while stakerFlowRate is taxFlowRate - providerFlowRate. This conserves the requested taxFlowRate, but it makes stakers receive the provider-side fractional remainder instead of applying an explicit configured rounding policy.

**Pre_conditions**  
1. A locker owner performs a taxable vesting unlock with unlockPeriod greater than zero and less than 365 days. 2. The configured provider allocation is neither 0 nor 10000, or otherwise taxFlowRate * providerAllocation is not exactly divisible by 10000. 3. The Fontaine is initialized with the rounded provider and residual staker flow rates.

**Impact**  
Liquidity providers receive slightly less than the configured tax allocation and stakers receive the residual. The local split does not return value to the unlock recipient and the drift is bounded by less than 1 wei per second per Fontaine, or less than unlockPeriod wei over the vesting period, below 31,536,000 wei for the 365-day maximum. This makes it an allocation correctness issue rather than a material tax bypass; larger recipient-return behavior from GDA actual-flow rounding is tracked separately in M-03.

**Proof of Concept**  
Not run per instruction. Code-level example: if taxFlowRate is 1 wei/sec and providerAllocation is 9000, providerFlowRate becomes 0 and stakerFlowRate becomes 1, so the requested 90/10 split is delivered as 0/100 for that Fontaine's tax stream. For normal larger flows the same bias is less than 1 wei/sec.

**Mitigation**  
Make the rounding policy explicit. If providers should receive the configured floor and stakers the residual, document it. Otherwise calculate both target shares with an explicit residual bucket, alternate or accumulate residuals across unlocks, or route residual tax through adjustment accounting instead of silently assigning it to stakers.

### [I-17] provideLiquidity leaves unused mint inventory unaccounted

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/FluidLocker.sol:433; fluid/packages/contracts/src/FluidLocker.sol:437; fluid/packages/contracts/src/FluidLocker.sol:438; fluid/packages/contracts/src/FluidLocker.sol:441; fluid/packages/contracts/src/FluidLocker.sol:679; fluid/packages/contracts/src/FluidLocker.sol:693; fluid/packages/contracts/src/FluidLocker.sol:710

**Summary/Description**  
FluidLocker.provideLiquidity() approves the full post-pump WETH balance and the requested SUP amount, but _createPosition() ignores the actual WETH/SUP amounts deposited by Uniswap V3 mint. When the current pool ratio consumes less than the desired amounts while still satisfying the 95% minimums, the unused WETH or SUP remains in the locker instead of being refunded or separately accounted.

**Root Cause**  
The LP entry path treats desired amounts as if they were fully deposited. _createPosition() returns depositedEthAmount and depositedSupAmount, but provideLiquidity() ignores those values, does not compare them against ethLPAmount and supAmount, and has no direct WETH sweep/refund path for mint leftovers.

**Pre_conditions**  
1. Locker owner calls provideLiquidity() with ETH and SUP. 2. The pool price or the caller supplied ratio makes Uniswap V3 mint consume less than one desired side, but not less than the 95% amountMin check. 3. The position manager pulls only the actual minted amounts and leaves the residual token balance in the locker.

**Impact**  
The owner can end up with idle WETH or SUP in the locker. WETH has no direct withdrawDustWETH-style recovery path and is only recovered indirectly by a future provideLiquidity() or by later withdrawLiquidity(), which unwraps the full locker WETH balance. SUP leftovers are mixed with ordinary locker SUP and are not part of the tax-free LP position until supplied again. No third party can use the residual allowance to pull from the locker through the standard Uniswap position manager payer flow, so this is low severity.

**Proof of Concept**  
Not run per instruction. Code trace: provideLiquidity() reads the whole WETH balance, approves desired WETH/SUP, and calls _createPosition(); Uniswap V3 NonfungiblePositionManager.mint() may return depositedAmount0/depositedAmount1 below desired amounts as long as amount0Min/amount1Min pass; _createPosition() returns the actual deposited amounts, but the caller discards them.

**Mitigation**  
Use the actual deposited amounts returned by _createPosition() to compute residual WETH and SUP. Refund or unwrap unused WETH to lockerOwner, clear stale allowances after mint, and either leave SUP explicitly as normal locked balance with an event or include/refund it according to the intended LP commitment model.

### [I-18] LP unit downscaler floors small liquidity out of provider rewards

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/StakingRewardController.sol:147; fluid/packages/contracts/src/FluidLocker.sol:699; fluid/packages/contracts/src/FluidLocker.sol:747

**Summary/Description**  
FluidLocker records aggregate Uniswap V3 liquidity and StakingRewardController converts it to LP distribution-pool units with floor(liquidity / 1e16). Positive liquidity below 1e16 receives zero units, and every non-multiple remainder below 1e16 is omitted from the provider reward denominator. That omitted liquidity share is effectively redistributed to lockers that do have integer LP units.

**Root Cause**  
updateLiquidityProviderUnits() uses truncating division by _LIQUIDITY_UNIT_DOWNSCALER with no minimum-unit check, residual accounting, or user-facing preview. _createPosition() connects the locker and accepts the position even when the resulting units are zero, and _decreasePosition() can leave a remaining LP balance whose units are lower than the raw liquidity would imply.

**Pre_conditions**  
1. A locker mints or keeps aggregate Uniswap liquidity that is not an exact multiple of 1e16, including the sub-threshold range [1, 1e16 - 1]. 2. The LP tax pool receives instant or streamed tax distributions. 3. For reward capture rather than only local reward loss, at least one other locker has nonzero LP units.

**Impact**  
A sub-threshold LP has positive Uniswap liquidity but receives no provider tax rewards and does not make LP_DISTRIBUTION_POOL.getTotalUnits() positive, so taxable unlocks can still fail if all LPs are sub-threshold. Above the threshold, each locker loses the reward weight of its remainder below 1e16; that lost weight is captured pro-rata by other nonzero LP-unit holders. The distortion can be large for very small positions, e.g. liquidity of 2e16 - 1 and 1e16 both map to one unit and split rewards 50/50 despite nearly a 2:1 raw-liquidity ratio. For normal-sized positions the omitted weight is less than one LP unit per locker, so this remains an Info-level allocation issue. Separate Superfluid requested-vs-actual rounding effects are tracked in I-15/M-03.

**Proof of Concept**  
Not run per instruction. Code trace: _createPosition() adds the minted liquidity to _liquidityBalance and calls updateLiquidityProviderUnits(_liquidityBalance); _decreasePosition() subtracts removed liquidity and calls the same updater. The controller writes uint128(lockerLiquidityBalance) / 1e16 units. Thus liquidity in [1, 1e16 - 1] maps to zero, while 2e16 - 1 maps to one unit, the same as exactly 1e16.

**Mitigation**  
Enforce a minimum LP size that produces at least one unit, expose/return the exact unit amount before accepting the position, and document the granularity if truncation is intended. If proportional precision matters for small LPs, use a larger unit precision with an explicit cap against GDA total-unit limits, or keep per-locker residual accounting so omitted liquidity is not silently reassigned.

### [I-19] withdrawLiquidity token amount parameters are re-sorted as WETH/SUP

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/interfaces/IFluidLocker.sol:185; fluid/packages/contracts/src/interfaces/IFluidLocker.sol:186; fluid/packages/contracts/src/FluidLocker.sol:449; fluid/packages/contracts/src/FluidLocker.sol:467; fluid/packages/contracts/src/FluidLocker.sol:731

**Summary/Description**  
The public withdrawLiquidity interface documents amount0ToRemove and amount1ToRemove as Uniswap token0/token1 amounts, but FluidLocker forwards them to _decreasePosition as pairedAssetAmountToRemove and supAmountToRemove. _decreasePosition then sorts those values again before deriving amount0Min/amount1Min. This only aligns when WETH is token0 and SUP is token1; if SUP is token0, callers following the public ABI documentation get the slippage minima swapped.

**Root Cause**  
The external function uses token0/token1 parameter names while the internal helper expects asset-semantic WETH/SUP ordering. The adapter layer does not convert token0/token1 values into pairedAsset/SUP values before calling _decreasePosition.

**Pre_conditions**  
1. The configured SUP/ETH Uniswap V3 pool has SUP as token0. 2. The locker owner or integration supplies amount0ToRemove and amount1ToRemove according to the public interface documentation. 3. The supplied amounts are intended as the expected token0/token1 withdrawal values for the liquidity being removed.

**Impact**  
On SUP-token0 deployments, the slippage check is applied to the wrong token sides. With ordinary SUP/ETH ratios this usually causes withdrawLiquidity to revert because the WETH minimum is set from the much larger SUP amount. The owner can still withdraw by using the undocumented WETH/SUP argument order or by setting loose minima, so this is an integration/liveness correctness issue rather than a direct fund-loss issue. The current Base RC token address sorts after WETH, so this mismatch is not triggered for that pool ordering.

**Proof of Concept**  
Not run per instruction. Code trace: IFluidLocker documents amount0ToRemove/amount1ToRemove as token0/token1; withdrawLiquidity passes them unchanged into _decreasePosition; _decreasePosition treats arg3 as paired asset and arg4 as SUP, then calls _sortInAmounts before calculating amount0Min/amount1Min.

**Mitigation**  
Either expose the external arguments as pairedAssetAmountToRemove and supAmountToRemove, or adapt the public token0/token1 inputs into pairedAsset/SUP order before calling _decreasePosition. Tests should cover both token orderings.

### [I-20] External Uniswap liquidity increases desync locker LP accounting

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/FluidLocker.sol:465; fluid/packages/contracts/src/FluidLocker.sol:747; fluid/packages/contracts/lib/uniswap-v3-periphery/contracts/NonfungiblePositionManager.sol:197

**Summary/Description**  
Uniswap V3 positions can be increased by any caller through NonfungiblePositionManager.increaseLiquidity(), with tokens paid by msg.sender. FluidLocker only increments _liquidityBalance when it creates a new position, so externally added liquidity makes the NFT's actual liquidity exceed the locker accounting used for LP units and later withdrawals.

**Root Cause**  
FluidLocker assumes the NFT position liquidity can only change through its own _createPosition() and _decreasePosition() paths. Canonical Uniswap V3 allows permissionless increaseLiquidity() on an existing tokenId, and FluidLocker has no reconciliation against NONFUNGIBLE_POSITION_MANAGER.positions(tokenId).liquidity before subtracting liquidityToRemove.

**Pre_conditions**  
1. A locker owns a Uniswap V3 position created through provideLiquidity(). 2. Any external caller approves/pays WETH and SUP to NonfungiblePositionManager.increaseLiquidity() for that tokenId. 3. The locker owner later calls withdrawLiquidity() using the actual position liquidity, or tries to fully close and burn the NFT.

**Impact**  
The locker _liquidityBalance and LP units undercount the actual Uniswap position liquidity. A full withdrawal using the NFT's current liquidity can underflow at _liquidityBalance -= liquidityToRemove and revert when the external increase exceeds the tracked aggregate balance. If the owner withdraws only the tracked liquidity, the externally added residual remains in the NFT, preventing a clean burn and keeping the position timestamp/count alive. With multiple positions, fully withdrawing an externally increased position can shift the undercount to the remaining positions. The issue is low impact because the desync amount is funded by the external liquidity adder and does not directly let them extract value.

**Proof of Concept**  
Not run per instruction. Code trace: FluidLocker._createPosition() adds only mint-returned liquidity to _liquidityBalance; canonical NonfungiblePositionManager.increaseLiquidity() has no isAuthorizedForToken modifier and pays from msg.sender; withdrawLiquidity() reads actual positionLiquidity but _decreasePosition() subtracts the caller-supplied liquidityToRemove from the stale _liquidityBalance after Uniswap accepts the decrease.

**Mitigation**  
Reconcile tracked liquidity with the position manager before withdrawal, or maintain per-token tracked liquidity and cap withdrawals to the tracked amount while explicitly handling externally added liquidity. If complete NFT closure is intended, update _liquidityBalance from the sum of live locker-owned positions or reject/ignore externally increased residuals in a documented way.

### [I-21] Staker unit calculation wraps balances before downscaling

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/StakingRewardController.sol:140

**Summary/Description**  
StakingRewardController.updateStakerUnits() casts lockerStakedBalance from uint256 to uint128 before dividing by the 1e18 staking unit downscaler. A locker staked balance above type(uint128).max wraps modulo 2^128 first, so the pool units can be far below floor(lockerStakedBalance / 1e18).

**Root Cause**  
The staker unit formula performs uint128(lockerStakedBalance) / _STAKING_UNIT_DOWNSCALER instead of downscaling the uint256 balance first and then safely casting the resulting unit count.

**Pre_conditions**  
1. An approved locker reaches a recorded _stakedBalance greater than type(uint128).max wei. 2. The locker calls stake() or unstake(), causing updateStakerUnits(_stakedBalance) to execute.

**Impact**  
The affected locker receives truncated staker tax-pool units, reducing or potentially zeroing its reward share and reallocating tax/subsidy distributions to other staker units. The practical impact is low because the threshold is about 3.4e20 SUP with 18 decimals, above the expected finite SUP supply in normal deployment assumptions; without an explicit in-code cap, the arithmetic remains incorrect for the function's uint256 input type.

**Proof of Concept**  
Not run per instruction. Code trace: updateStakerUnits(type(uint128).max + 1e18) computes uint128(type(uint128).max + 1e18) / 1e18 instead of (type(uint128).max + 1e18) / 1e18, so the high bits are discarded before the intended unit conversion.

**Mitigation**  
Compute uint256 units = lockerStakedBalance / _STAKING_UNIT_DOWNSCALER, then use a checked cast such as SafeCast.toUint128(units), or explicitly cap lockerStakedBalance before updating pool units.

### [I-22] distributeTaxAdjustment mixes subsidy and stray FLUID as tax adjustment

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/StakingRewardController.sol:208

**Summary/Description**  
StakingRewardController.distributeTaxAdjustment() treats the controller's entire current FLUID balance as tax adjustment. The intended source is tax-pool GDA adjustment, but the controller is also the admin of the staker distribution pool used by FluidEPProgramManager subsidies. Any FLUID adjustment residue that reaches the controller from subsidy flow/unit changes, plus any direct SuperToken transfer, is indistinguishable from unlock-tax adjustment and is split between stakers and LPs by the current taxAllocation.

**Root Cause**  
The function derives adjustmentAmount from FLUID.balanceOf(address(this)) instead of maintaining source-specific adjustment accounting. FluidEPProgramManager sends subsidy flow to TAX_DISTRIBUTION_POOL, whose pool admin is the StakingRewardController; GDA pool-admin adjustment for that staker-subsidy pool can therefore increase the same controller balance that distributeTaxAdjustment later reclassifies as tax adjustment. The controller also has no recovery path for unrelated direct FLUID transfers.

**Pre_conditions**  
1. The controller has a FLUID balance not attributable to unlock-tax adjustment, either from staker-subsidy GDA adjustment residue or from a direct/future-integration transfer. 2. taxAllocation and both tax pools are configured. 3. Any account calls distributeTaxAdjustment().

**Impact**  
Staker-subsidy adjustment residue can be redistributed to the LP pool even though FluidEPProgramManager subsidies target the staker tax distribution pool. With the deployed 10% staker / 90% LP tax split, a subsidy-derived controller balance would mostly be sent to LP units. Directly transferred unrelated FLUID is likewise distributed to tax-pool members rather than returned to the sender. This is weaker than M-02 because the in-protocol amount depends on GDA adjustment/rounding residue and unit-change conditions, and direct transfers require out-of-flow funds being sent to the controller.

**Proof of Concept**  
Not run per instruction. Code trace: FluidEPProgramManager.startFunding opens subsidy flow through _updateSubsidyFlowRate at lines 322-324, and _updateSubsidyFlowRate calls token.distributeFlow(address(this), TAX_DISTRIBUTION_POOL, newSubsidyFlowRate) at lines 651-659. The GDA pool admin is the StakingRewardController because it created taxDistributionPool at StakingRewardController.sol:130. GDA adjustment flow is paid to the pool admin, while distributeTaxAdjustment reads the controller's raw FLUID.balanceOf(address(this)) at line 209 and distributes the derived staker/LP split at lines 221-222. SuperToken.transfer is also public, so a direct transfer to the controller increases the same balance.

**Mitigation**  
Track adjustment balances by source and intended recipient set. Unlock-tax adjustment, staker-subsidy adjustment, and LP-tax adjustment should be separate buckets, each reduced only by actual distributed amounts. Route subsidy-derived adjustment back to the staker pool or treasury according to the intended subsidy model, and add a rescue path for non-accounted FLUID or other tokens that cannot withdraw accounted adjustment balances.

### [I-23] Final-window lockedSUP balance overstates remaining vesting balance

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/vesting/SupVestingFactory.sol:187

**Summary/Description**  
SupVestingFactory.balanceOf is intended to report the receiver's unvested lockedSUP balance as the vesting contract's token balance plus the active scheduler flow deposit. This is exact while the vesting contract's available SuperToken balance is nonnegative. In the final active-flow window, however, the SuperToken available balance can become negative while the account is still solvent through the locked CFA deposit; SuperToken.balanceOf clamps that negative available balance to zero, and the factory then adds the full deposit, overstating the remaining vesting balance.

**Root Cause**  
The function combines SUP.balanceOf(supVestings[vestingReceiver]) with the per-flow deposit returned by getFlowInfo. SuperToken.balanceOf returns max(availableBalance, 0), where availableBalance already subtracts the outgoing-flow deposit. When availableBalance is negative but greater than -deposit, the correct real-time remaining balance is availableBalance + deposit, but the factory computes 0 + deposit.

**Pre_conditions**  
1. A SupVesting schedule has started and the SupVesting contract has an active CFA flow to the recipient. 2. Time is late enough that the remaining real-time balance is below the CFA deposit, but executeEndVesting or emergencyWithdraw has not yet deleted the flow. 3. An off-chain strategy or observer relies on SupVestingFactory.balanceOf or totalSupply during that window.

**Impact**  
The view balance and totalSupply can be overstated by up to the active CFA deposit per vesting schedule during the final active-flow window. I found no in-scope contract path that consumes this value for transfers, claims, or permissions; the concrete effect is incorrect lockedSUP/Snapshot-style accounting until the scheduler deletes the flow.

**Proof of Concept**  
Not run per instruction. Code trace: SupVestingFactory.balanceOf reads the mapped vesting contract, gets the CFA deposit for vestingContract -> receiver, and returns SUP.balanceOf(vestingContract) + deposit. SuperToken.balanceOf returns zero whenever realtime availableBalance is negative. For an active stream with real-time remaining balance R and deposit D where 0 < R < D, availableBalance = R - D < 0, so SuperToken.balanceOf returns 0 while the factory returns D instead of R.

**Mitigation**  
Compute the signed real-time balance before clamping. For example, read SUP.realtimeBalanceOfNow(supVesting) and add the relevant flow deposit to the signed available balance, then clamp only after the addition. Alternatively return zero once the scheduler schedule has reached its executable end window and require off-chain accounting to use scheduler schedule state rather than ERC20-style balanceOf during active flows.

### [I-24] SupVesting grants scheduler excess permissions

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fluid/packages/contracts/src/vesting/SupVesting.sol:96

**Summary/Description**  
SupVesting grants the vesting scheduler full CFA flow-operator control and unlimited SUP allowance in its constructor, even though the stored V2 vesting schedule only needs create/delete flow permission at the scheduled flow rate and bounded token allowance for cliff, delay, remainder, and end compensation.

**Root Cause**  
In SupVesting.sol:96-98 the constructor calls setMaxFlowPermissions(address(vestingScheduler)) and approve(address(vestingScheduler), type(uint256).max). The integrated VestingSchedulerV2 only spends tokens through schedule-bound transferFrom calls and exposes getMaximumNeededTokenAllowance, while its tests demonstrate exact token allowance plus create/delete-only flow permission is sufficient.

**Pre_conditions**  
1. A SupVesting contract is created through the factory. 2. The configured scheduler is later able to exercise more authority than the current schedule-bound V2 implementation, for example due to a scheduler defect, wrong scheduler address, or a future scheduler variant with a broader public path.

**Impact**  
No direct arbitrary-drain path was found in the current local VestingSchedulerV2 code: arbitrary callers can only execute the stored schedule for the stored recipient. The residual risk is unnecessary authority persistence: if the configured scheduler can ever call arbitrary transferFrom or flowFrom paths, every SupVesting contract has already granted it full access to its SUP balance and full flow control instead of only the amount and rate required by the vesting schedule.

**Proof of Concept**  
Not run per instruction. Code-level trace only: SupVesting constructor grants max flow permissions and type(uint256).max token allowance before creating the schedule; VestingSchedulerV2 uses transferFrom only at executeCliffAndFlow/executeEndVesting and computes a bounded maximum allowance in getMaximumNeededTokenAllowance.

**Mitigation**  
Grant least-privilege permissions: set flow permissions with create=true, update=false, delete=true, and flowRateAllowance=flowRate, and approve only the scheduler's getMaximumNeededTokenAllowance for the created schedule. Revoke remaining token and flow permissions after emergencyWithdraw and after the schedule is fully executed, if an on-chain callback or cleanup path is added.

### [I-25] Critical-window emergencyWithdraw can route remaining vesting balance to Superfluid rewards

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fluid/packages/contracts/src/vesting/SupVesting.sol:120

**Summary/Description**  
SupVesting.emergencyWithdraw closes the active SupVesting-to-recipient CFA flow before sweeping the vesting contract balance to treasury. For a normal non-critical active flow, this releases the CFA deposit and the post-delete SUP.balanceOf contains the remaining unstreamed balance. If the call is made after the account has become critical and while Superfluid still classifies it as Patrician-period solvent, CFA deletion first pays the remaining deposit-backed balance to the Superfluid reward account, so the later treasury sweep can receive zero instead of the remaining active-stream balance.

**Root Cause**  
The function calls SUP.flow(RECIPIENT, 0) without checking whether the SupVesting sender is already critical. Superfluid CFA deleteFlow calls _makeLiquidationPayouts whenever availableBalance < 0; during the Patrician branch, makeLiquidationPayoutsV2 transfers rewardAmount from the target account to the default reward account before _changeFlow releases the deposit.

**Pre_conditions**  
1. A SupVesting schedule has started and the SupVesting contract has an active CFA flow to the recipient. 2. The vesting contract's remaining unstreamed balance is lower than the CFA deposit, making availableBalance negative, but the account is still in Superfluid's Patrician-period critical window. 3. A Superfluid reward account is configured so Patrician rewards do not resolve back to the SupVesting contract. 4. The trusted admin calls emergencyWithdraw during that window instead of before criticality or after the Patrician window.

**Impact**  
The treasury can fail to recover the remaining active-stream balance for that schedule; the amount routed away is bounded by the current remaining balance and therefore by the active CFA deposit window. This is Info-level because README scope marks SupVesting admin and treasury as trusted and states Superfluid vesting automation guarantees timely starts and terminations, so the path depends on avoidable privileged timing rather than an untrusted caller.

**Proof of Concept**  
Not run per instruction. Code trace: SupVesting.emergencyWithdraw calls SUP.flow(RECIPIENT, 0), which uses SuperTokenV1Library.flow with sender address(this) and calls CFA deleteFlow. In ConstantFlowAgreementV1._deleteFlow, any negative availableBalance triggers _makeLiquidationPayouts before _changeFlow. For one active flow with remaining real-time balance R and deposit D, availableBalance = R - D. In the Patrician branch, rewardAmount = D * (availableBalance + D) / D = R and makeLiquidationPayoutsV2 subtracts R from SupVesting and credits the default reward account. _changeFlow then settles the stream and releases the deposit, leaving no R for the later SUP.transfer(FACTORY.treasury(), remainingBalance).

**Mitigation**  
Before emergency withdrawal, either require the sender not to be critical or call the normal scheduler end path before the critical window. If emergency recovery must support critical active streams, compute the expected remaining balance and handle the Patrician liquidation case explicitly, or wait until deletion by the sender no longer routes rewards away before sweeping.

