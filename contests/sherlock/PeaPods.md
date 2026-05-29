### [H-01] First depositor can capture WeightedIndex assets after supply is burned to dust or zero

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/contracts/WeightedIndex.sol:143; contracts/contracts/WeightedIndex.sol:162; contracts/contracts/WeightedIndex.sol:186; contracts/contracts/DecentralizedIndex.sol:321; contracts/contracts/DecentralizedIndex.sol:424

**Summary/Description**  
WeightedIndex treats totalSupply == 0 as a fresh first deposit even when recorded _totalAssets is still nonzero, and it also lets totalSupply be reduced independently of recorded assets. A first holder can burn shares down to dust through burn(), or flashMint() can burn the caller's fee shares to zero supply while leaving the underlying tracked in _totalAssets. Later bonds mint from the reduced or zero supply base while still adding to the stale recorded asset balances.

**Root Cause**  
The share conversion branches on _totalSupply alone, while burn() and flashMint() can reduce _totalSupply without reducing _totalAssets or enforcing dead shares, a minimum retained supply, or a no-nonzero-assets zero-supply invariant.

**Pre_conditions**  
A pod is empty or a holder controls enough pTKN to reduce supply to dust or zero; for the theft path, a later bonder accepts amountMintMin = 0 or another too-low minimum. The zero-supply tracked-asset state can also be reached by bonding exactly the flashMint fee and then calling flashMint().

**Impact**  
With dust supply, the next bond can round to zero shares while its tokens are still added to _totalAssets, after which the dust holder debonds and receives the victim's assets. With zero supply and nonzero recorded assets, the assets are temporarily unowned and the next first-in bond can capture them or recycle a flashMint fee that was meant to be paid to existing holders.

**Proof of Concept**  
Single-asset theft example with zero fees for clarity: attacker bonds A=100 tokens and receives 100 pTKN, then burns 100e18-1 pTKN through DecentralizedIndex.burn(), leaving _totalSupply=1 and _totalAssets=100e18. Victim bonds V=99 tokens with min=0. _tokensMinted=floor(1*99e18/100e18)=0, but _transferAmt=99e18 and _totalAssets becomes 199e18. Attacker debonds 1 wei pTKN as last holder and receives the recorded assets. Zero-supply variant: bond exactly the flashMint fee, call flashMint so DecentralizedIndex.flashMint burns that fee and leaves totalSupply=0 while _totalAssets remains nonzero; the next first-in bond ignores the stale assets when minting but can later redeem them because _bond adds to the existing _totalAssets bucket.

**Mitigation**  
Maintain the invariant that totalSupply can only be zero when all recorded _totalAssets are zero, or mint and permanently lock dead shares. Prevent burn() and flashMint() from reducing supply below a minimum while assets remain, reject zero-share bonds, and make first-in minting account for or clear existing recorded assets.

### [H-02] Missing non-redeemable aspTKN seed lets first depositors inflate share price

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/contracts/AutoCompoundingPodLp.sol:124; contracts/contracts/AutoCompoundingPodLp.sol:130; contracts/contracts/AutoCompoundingPodLp.sol:176; contracts/contracts/AutoCompoundingPodLp.sol:190; contracts/contracts/AutoCompoundingPodLp.sol:204; contracts/contracts/AutoCompoundingPodLp.sol:213; contracts/contracts/AutoCompoundingPodLpFactory.sol:30; contracts/contracts/AutoCompoundingPodLpFactory.sol:41; contracts/contracts/lvf/LeverageFactory.sol:101; contracts/contracts/lvf/LeverageFactory.sol:109; contracts/contracts/lvf/LeverageFactory.sol:111

**Summary/Description**  
AutoCompoundingPodLp relies on factory seeding instead of virtual shares or dead shares, but the factory seed is not guaranteed to be non-redeemable. Direct/non-LVF deployments mint the creation seed to the create caller, and that holder can redeem it before honest deposits. The self-lending factory flow is currently blocked earlier by M-36, but if that ordering is fixed without changing the seed logic, it creates the aspTKN with _pod == address(0), skips the seed, and later enables a zero-supply vault via setPod(). In either zero-supply state, a first depositor can mint dust shares, force reward conversion before a later deposit, and capture value from the later depositor through the inflated CBR.

**Root Cause**  
AutoCompoundingPodLp._cbr() returns 1e18 whenever totalSupply is zero and the vault has no virtual share/asset offset. AutoCompoundingPodLpFactory.create() only calls _depositMin() when _pod is nonzero, and _depositMin() passes _msgSender() as the deposit receiver instead of a burn or permanently locked address, so the creation seed is either absent or redeemable rather than non-redeemable dead liquidity.

**Pre_conditions**  
An aspTKN becomes enabled with totalSupply == 0. In the current direct factory/script path, this can happen if the caller-owned creation seed is redeemed before honest deposits. In the self-lending path, the current code first hits the separate M-36 ordering revert, but once that deployment ordering is corrected the existing factory seed logic would still skip _depositMin() while _pod is address(0) and then enable the vault with setPod(). A later user deposits after a first depositor can acquire dust aspTKN shares and send processable reward tokens to the aspTKN.

**Impact**  
Later depositors can receive too few aspTKN shares and lose deposited staked-LP value to the first depositor. The loss can be a large fraction of the victim deposit when the first depositor starts from dust supply and adds processable reward value before the victim's share calculation; the attack is repeatable for newly enabled or re-emptied aspTKNs that lack locked seed liquidity.

**Proof of Concept**  
Redeemable-seed path: AutoCompoundingPodLpFactory._depositMin() deposits minimumDepositAtCreation to the new aspTKN but passes _msgSender() as the receiver, so the external create caller receives all seed shares. That seed holder can call redeem() while no honest deposits exist, burning the seed and returning totalSupply and _totalAssets to zero. Self-lending after the M-36 ordering issue is fixed: createSelfLendingPodAndAddLvf is designed to create the aspTKN with _pod = address(0), so AutoCompoundingPodLpFactory.create() skips _depositMin(); after setPod(), totalSupply and _totalAssets remain zero. From either zero-supply state, the first depositor deposits 1 wei of asset and receives 1 share because _cbr() returns FACTOR. The depositor transfers processable reward tokens to the aspTKN. On the next deposit(V), deposit() first calls _processRewardsToPodLp(), which converts held rewards into staked LP and increments _totalAssets, then computes shares as floor(V * totalSupply / totalAssets). Choosing converted net LP just above V / 2 makes the later depositor mint only 1 share, after which the first depositor can redeem 1 of 2 shares for roughly half the vault assets.

**Mitigation**  
Ensure every aspTKN has non-redeemable seed liquidity before public deposits are possible. Mint the creation seed to address(0), address(0xdead), or another permanently locked address; for self-lending creation, set the pod and make the locked seed deposit in the same initialization flow; or add virtual asset/share offsets to the conversion formulas. Do not allow setPod() or creation flows to leave an enabled vault with totalSupply == 0, and reject zero-share deposits.

### [H-03] Partial liquidations can create avoidable bad debt

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: fraxlend/src/contracts/FraxlendPairCore.sol:1135; fraxlend/src/contracts/FraxlendPairCore.sol:1140; fraxlend/src/contracts/FraxlendPairCore.sol:1145; fraxlend/src/contracts/FraxlendPairCore.sol:1149; fraxlend/src/contracts/FraxlendPairCore.sol:1151; fraxlend/src/contracts/FraxlendPairCore.sol:1168; fraxlend/src/contracts/FraxlendPairCore.sol:1174; fraxlend/src/contracts/FraxlendPairCore.sol:1177; fraxlend/src/contracts/FraxlendPairCore.sol:1195

**Summary/Description**  
FraxlendPair lets the liquidator choose any borrow-share amount and then sizes seized collateral only from that chosen amount. A caller can choose a partial liquidation that is more profitable than full repayment while causing avoidable lender loss, and liquidate() does not reject the borrower as caller. In the clean branch, the code writes off all remaining borrower shares whenever the clean-fee collateral quote consumes all borrower collateral, even if the borrower collateral value could cover the full debt at par. In the dirty branch, the code can also allow a partial liquidation that extracts more liquidation penalty than the borrower equity, leaving a residual debt/collateral position that has become bad debt due to the liquidation itself.

**Root Cause**  
The liquidation math checks collateral consumption by the selected repayment amount plus liquidation fee, but it does not separately verify whether the borrower debt exceeds collateral value before writing off debt, and it does not require dirty partial liquidations to preserve or improve borrower equity/health. The same clean-fee _leftoverCollateral value is also reused as the clean/dirty boundary; because dirtyLiquidationFee is lower than cleanLiquidationFee, any selected repayment x with C / (1 + cleanLiquidationFee) <= x < C / (1 + dirtyLiquidationFee) is treated as a clean liquidation/writeoff even though a dirty-fee collateral quote would leave borrower collateral. The clean/dirty branch condition therefore lets caller-selected liquidation size determine when lender accounting absorbs the unpaid remainder.

**Pre_conditions**  
cleanLiquidationFee is nonzero, dirtyLiquidationFee is positive, protocolLiquidationFee is below the liquidation bonus break-even for the relevant branch, and minCollateralRequiredOnDirtyLiquidation does not block the dirty partial. In the current constructor/setter surface, dirtyLiquidationFee is derived as 90% of cleanLiquidationFee and minCollateralRequiredOnDirtyLiquidation defaults to zero. The borrower is liquidatable but still collateral-sufficient at par, with debt value D and collateral value C. The clean one-step path exists when C / (1 + cleanLiquidationFee) <= D <= C. The dirty degradation path exists when D < C and the caller can choose repayment x such that (C - D) / dirtyLiquidationFee < x < C / (1 + cleanLiquidationFee).

**Impact**  
Rational liquidators can maximize profit by repaying less than the full debt and externalizing the remaining loss to lenders; a borrower with enough asset balance can also use the same path to close debt more cheaply than full repayment. For the clean path, the caller can repay only the threshold amount that makes _leftoverCollateral <= 0, receive the same all-collateral allocation, and have the rest of the borrower shares removed through _sharesToAdjust and totalAsset.amount -= _amountToAdjust. For the dirty path, a caller can first take a profitable dirty liquidation fee large enough to consume all borrower equity, leaving the pair with residual bad debt that did not exist before the liquidation. Because totalAsset.amount backs Fraxlend fTokens and can be checkpointed into LendingAssetVault via whitelistUpdate, the loss lowers lender CBR and can propagate into LVF/self-lending accounting and oracle-dependent liquidations.

**Proof of Concept**  
Example A, clean path: exchangeRate = 1, totalBorrow.amount = totalBorrow.shares = 100, borrowerShares = 100, userCollateralBalance = 105.263, maxLTV = 75%, cleanLiquidationFee = 10%, protocolLiquidationFee = 2%. Full liquidation would repay 100 and avoid lender loss. Instead, liquidating 95.694 shares makes optimisticCollateralForLiquidator = 95.694 * 1.1 = 105.263, so _leftoverCollateral <= 0. The function allocates all collateral, charges protocol fee, sets _sharesToAdjust to the remaining shares, subtracts the unpaid amount from totalAsset.amount, and transfers only the 95.694 repayment from the caller. Example B, dirty path: C = 100, D = 95, cleanLiquidationFee = 10%, dirtyLiquidationFee = 9%, protocolLiquidationFee = 2%, and minCollateralRequiredOnDirtyLiquidation = 0. Liquidating x = 90 stays in the dirty branch because 90 * 1.1 = 99 < 100, but it removes 90 * 1.09 = 98.1 collateral while only reducing debt by 90. The borrower is left with 5 debt and 1.9 collateral, so a position that was collateral-sufficient before liquidation has become bad debt after a profitable partial liquidation.

**Mitigation**  
Separate liquidation bonus accounting from bad-debt accounting. Only write off borrower shares when debt value exceeds collateral value before applying liquidation incentives. Cap partial liquidation size so the seized bonus cannot exceed borrower equity, or require the post-liquidation position to be solvent/improved unless the transaction fully closes the debt. For clean liquidations, require full borrower-share repayment whenever collateral can cover the debt at par; for dirty liquidations, enforce a health/equity improvement check and reject partials that create or deepen bad debt.

### [H-04] Pre-writeoff exits shift Fraxlend bad debt into and within LAV

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: fraxlend/src/contracts/FraxlendPairCore.sol:646; fraxlend/src/contracts/FraxlendPairCore.sol:657; fraxlend/src/contracts/FraxlendPairCore.sol:745; fraxlend/src/contracts/FraxlendPairCore.sol:751; fraxlend/src/contracts/FraxlendPairCore.sol:755; fraxlend/src/contracts/FraxlendPairCore.sol:763; fraxlend/src/contracts/FraxlendPairCore.sol:787; fraxlend/src/contracts/FraxlendPairCore.sol:823; fraxlend/src/contracts/FraxlendPairCore.sol:1174; fraxlend/src/contracts/FraxlendPairCore.sol:1177; contracts/contracts/LendingAssetVault.sol:136; contracts/contracts/LendingAssetVault.sol:152; contracts/contracts/LendingAssetVault.sol:180; contracts/contracts/LendingAssetVault.sol:181; contracts/contracts/LendingAssetVault.sol:183; contracts/contracts/LendingAssetVault.sol:184; contracts/contracts/LendingAssetVault.sol:204; contracts/contracts/LendingAssetVault.sol:211; contracts/contracts/LendingAssetVault.sol:230; contracts/contracts/LendingAssetVault.sol:236; contracts/contracts/LendingAssetVault.sol:237

**Summary/Description**  
FraxlendPair only recognizes bad debt inside liquidate(), but both pair-level and LAV-level exits can still price claims from pre-writeoff accounting until liquidation occurs. A pair fToken holder can redeem at the stale totalAsset.amount price; if local cash is short, _redeem() pulls idle LendingAssetVault assets through _depositFromVault(), causing LAV to receive freshly minted fTokens at the stale pre-loss price. Separately, an LAV share holder can redeem or withdraw idle LAV liquidity before the same whitelisted pair's bad debt is reflected in LAV _totalAssets, leaving the later CBR drop to the remaining LAV holders. In both variants, the exiting holder avoids its pro-rata share of an economically existing Fraxlend shortfall.

**Root Cause**  
Bad-debt accounting is lazy and holder-based at liquidation time. Fraxlend withdrawals do not update exchange-rate solvency, do not recognize pending shortfall, and can pull external LAV liquidity by minting new fTokens before the loss is applied. LAV share withdrawals only refresh whitelisted vault metadata through interest/CBR checkpoints and have no mechanism to detect a pending Fraxlend liquidation shortfall before paying idle assets from totalAvailableAssets. The LAV whitelistWithdraw path also has no information that the pair shares it is about to receive are priced before an already-existing shortfall.

**Pre_conditions**  
A Fraxlend pair has an externalAssetVault configured with available allocation or an LAV has both pair exposure and idle assets. A borrower position is economically underwater under the current oracle price, but liquidate() has not yet run the writeoff. A pair fToken holder can redeem while the pair has enough local plus LAV-available liquidity, or an LAV share holder can redeem/withdraw while the LAV has enough totalAvailableAssets. Liquidation is permissionless but not atomic with the price move, so the exit can be bundled before liquidation or race the first liquidation transaction.

**Impact**  
Exiting holders avoid their pro-rata share of bad debt and force the shortfall onto LAV depositors or remaining pair/LAV lenders. In the pair-withdrawal variant, LAV depositors can absorb a loss even though their idle assets were not pair fToken exposure when the shortfall economically arose. In the LAV-withdrawal variant, early LAV redeemers drain idle assets at the stale vault CBR and leave the eventual pair writeoff to the residual LAV supply. The value shifted can approach the unpaid borrow amount, bounded by available LAV liquidity/allocation, idle LAV assets, and exiting share balances.

**Proof of Concept**  
Pair exit example: attacker deposits 1000 assets into a pair and receives 1000 fTokens, then borrows 900 assets against collateral. The pair has only 100 local assets left, while its configured LAV has 900 idle assets available. After the collateral price moves so the 900 debt is backed by only about 10 assets of collateral value, no writeoff has occurred yet, so the attacker's 1000 fTokens still redeem for 1000 assets. redeem(1000) pulls 900 from LAV through _depositFromVault(), mints 900 fTokens to LAV at the stale 1:1 price, and transfers 1000 assets to the attacker. A subsequent liquidation writes off roughly 890 unpaid debt through totalAsset.amount -= _amountToAdjust, reducing the value of the LAV-held fTokens. LAV share exit example: LAV has 2000 total assets, 1000 idle and 1000 utilized in that same pair, with 2000 LAV shares outstanding. A pending liquidation will reduce that utilized pair exposure, but before liquidate() runs a holder with 1000 LAV shares calls redeem(1000). LAV refreshes interest but does not apply the pending shortfall, computes 1000 assets at the stale CBR, and pays the idle assets. After liquidation and LAV metadata refresh, the remaining 1000 LAV shares alone absorb the pair-exposure loss; with the separate down-CBR math bug, the recorded loss can be larger than the actual pair CBR drop.

**Mitigation**  
Apply pending shortfall before any pair or LAV withdrawal can be priced or paid. Possible fixes include blocking LAV-backed pair withdrawals and LAV share withdrawals when a whitelisted pair has liquidatable shortfall until liquidation/writeoff is completed, adding explicit shortfall accounting that is applied before redeem/withdraw pricing, or requiring LAV allocations to enter as standing fToken exposure before losses can accrue. At minimum, prevent insolvent borrowers from redeeming lender shares before their debt is liquidated, but that alone does not solve unrelated underwater borrowers or LAV share holders exiting idle LAV liquidity before the pair loss is recognized.

### [H-05] Untrusted pod metadata can over-credit PodUnwrapLocker locks

**Severity**: High  
**Likelihood**: High  
**Impact**: High  
**Location**: contracts/contracts/PodUnwrapLocker.sol:60; contracts/contracts/PodUnwrapLocker.sol:67; contracts/contracts/PodUnwrapLocker.sol:72; contracts/contracts/PodUnwrapLocker.sol:80; contracts/contracts/PodUnwrapLocker.sol:83; contracts/contracts/PodUnwrapLocker.sol:90; contracts/contracts/PodUnwrapLocker.sol:148; contracts/contracts/PodUnwrapLocker.sol:158

**Summary/Description**  
PodUnwrapLocker.debondAndLock accepts an arbitrary _pod address, snapshots the token addresses returned by that contract, and records a balance delta for each returned entry. Because the asset list is not tied to a canonical WeightedIndex deployment and is not deduplicated, an untrusted pod can return the same real token multiple times, transfer that token once during debond, and make the new lock record the same received balance delta once per duplicate entry.

**Root Cause**  
The locker treats untrusted pod metadata as authoritative custody accounting. It does not verify that _pod is a canonical protocol pod, does not deduplicate getAllAssets() token entries, and stores per-entry deltas that are later paid from the locker-wide token balance rather than from isolated per-lock escrow.

**Pre_conditions**  
At least one user has locked a real underlying token in PodUnwrapLocker. Another caller can supply an ERC20-compatible untrusted pod implementing IDecentralizedIndex and can pre-fund it with a small amount of that same underlying token.

**Impact**  
The caller can create an overstated lock and withdraw more of the target token than the untrusted pod contributed, taking the difference from other users' locked balances. The path is repeatable against any token held by the locker and can be amplified by returning many duplicate asset entries, so the caller can drain most or all currently locked balances for that token subject only to gas and the amount they choose to front.

**Proof of Concept**  
A user locks token X through a real pod, leaving B units of X in the locker with a future unlock time. A second caller supplies UntrustedPod whose getAllAssets() returns [X, X, ..., X], whose config() returns debondCooldown = 0, and whose debond() transfers only a units of X to the locker. When the caller invokes debondAndLock(UntrustedPod, amount), the locker snapshots the same X balance once for each duplicate entry, observes the same +a delta for every entry, and stores total recorded amounts of k * a even though only a was supplied. Because unlockTime equals the current timestamp, the caller can immediately call withdraw and receive k * a X from the shared locker balance, taking (k - 1) * a X from existing locks.

**Mitigation**  
Only accept pods deployed by the trusted factory or registered in IndexManager, and reject non-canonical pod implementations. Additionally, deduplicate the token list before balance-delta accounting or aggregate deltas by token address so each token balance increase can be credited at most once per lock. Apply the same canonical-pod validation before trusting debondCooldown for unlock timing.

### [H-06] Final Fraxlend fToken burns can poison zero-supply markets

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: fraxlend/src/contracts/libraries/VaultAccount.sol:25; fraxlend/src/contracts/libraries/VaultAccount.sol:30; fraxlend/src/contracts/libraries/VaultAccount.sol:38; fraxlend/src/contracts/FraxlendPairCore.sol:663; fraxlend/src/contracts/FraxlendPairCore.sol:668; fraxlend/src/contracts/FraxlendPairCore.sol:674; fraxlend/src/contracts/FraxlendPairCore.sol:678; fraxlend/src/contracts/FraxlendPairCore.sol:729; fraxlend/src/contracts/FraxlendPairCore.sol:763; fraxlend/src/contracts/FraxlendPairCore.sol:764; fraxlend/src/contracts/FraxlendPairCore.sol:787; fraxlend/src/contracts/FraxlendPairCore.sol:823; fraxlend/src/contracts/FraxlendPairCore.sol:840; fraxlend/src/contracts/FraxlendPairCore.sol:843; fraxlend/src/contracts/FraxlendPairCore.sol:1023; fraxlend/src/contracts/FraxlendPairCore.sol:1048

**Summary/Description**  
Fraxlend can burn the entire fToken supply while leaving a positive accounted asset amount. The public withdraw() path does this when the last holder withdraws slightly less than totalAsset.amount after CBR rises above 1: toShares(..., true) rounds the requested amount up to all remaining shares, and _redeem() subtracts only the requested assets. The same invariant break is reachable through the LAV return path: repayAsset() calls _withdrawToVault(), which computes ceil shares for the amount being returned to the externalAssetVault and then burns the vault-owned fTokens through _redeem(). If the return amount rounds up to the LAV full fToken balance but is below the full accounted asset amount, totalAsset.amount remains positive while totalAsset.shares becomes zero.

**Root Cause**  
The redeem helper allows a burn of all remaining fToken shares without requiring the redeemed asset amount to equal the full accounted asset amount. _withdrawToVault() reuses the same _redeem() path after choosing a rounded-up share amount for a partial asset return, so internal LAV repayments can hit the same terminal state as public withdraw(). Deposit and mint logic also treats amount-positive/share-zero and amount-zero/share-positive states as fresh-market branches instead of invalid recovery states.

**Pre_conditions**  
A Fraxlend pair has totalAsset.amount > totalAsset.shares from accrued lender yield. For the public path, one holder owns the remaining fToken supply and withdraws an amount that is less than totalAsset.amount but whose ceil-rounded shares equal the full supply. For the LAV path, the externalAssetVault owns the remaining fToken supply after funding pair liquidity, and a borrower repays enough assets that _withdrawToVault(returnAmount) ceil-rounds to the LAV full fToken balance while returnAmount is still below totalAsset.amount.

**Impact**  
The pair can enter totalAsset.amount > 0 with totalAsset.shares == 0. Later deposit() calls mint zero fTokens for real assets, and mint() can recreate a share through the zero-supply branch that captures the stranded assets and zero-share deposits. In the LAV repayment variant, the remaining positive amount can also leave LAV utilization backed by no fTokens; a later repayment or vault return can reach VaultAccount.toShares(..., true) with total.amount > 0 and total.shares == 0, causing a division-by-zero revert in the round-up check and blocking cleanup until the market is manually recovered.

**Proof of Concept**  
Public withdraw example: totalAsset.amount = 101 and totalAsset.shares = 100, all held by one lender. withdraw(100) computes ceil(100 * 100 / 101) = 100, burns all shares, transfers 100 assets, and leaves amount = 1, shares = 0. LAV return example: LAV supplies 100 assets to the pair and receives 100 fTokens, a borrower owes 101 after interest, then repays 99 borrow shares for 100 assets. repayAsset() updates LAV utilization to about 101, calls _withdrawToVault(100), and that burns all 100 LAV fTokens while subtracting only 100 from totalAsset.amount. The pair is left with amount = 1 and shares = 0 while debt/utilization dust remains.

**Mitigation**  
Preserve the zero-supply invariant everywhere _redeem() is used. If a redeem, withdraw, fee withdrawal, or _withdrawToVault() burn would consume all remaining shares, require the asset amount to equal all accounted assets or explicitly settle both amount and shares to zero under a documented recovery rule. Reject zero-share mints in public deposit and internal _depositFromVault. Treat amount-positive/share-zero and amount-zero/share-positive as invalid states in VaultAccount conversion helpers instead of fresh-market branches.

### [M-01] Uncheckpointed fee rewards can be distributed with a stale totalShares denominator

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/TokenRewards.sol:102; contracts/contracts/TokenRewards.sol:137; contracts/contracts/TokenRewards.sol:141; contracts/contracts/TokenRewards.sol:231; contracts/contracts/TokenRewards.sol:292; contracts/contracts/TokenRewards.sol:313; contracts/contracts/StakingPoolToken.sol:72; contracts/contracts/StakingPoolToken.sol:101; contracts/contracts/DecentralizedIndex.sol:185; contracts/contracts/DecentralizedIndex.sol:228

**Summary/Description**  
TokenRewards processes pending pod fees before applying share changes, then deposits rewards by dividing the new reward amount by the current totalShares. This ordering only preserves reward ownership if the pending fees are fully checkpointed before shares mutate. In practice, DecentralizedIndex can return early because of SWAP_DELAY or the minimum processing threshold, and TokenRewards._swapForRewards can catch a failed paired-token-to-reward-token swap while leaving the paired-token proceeds unaccounted. Later staking shares are then added, and the old fee value is eventually deposited using the later totalShares denominator, letting late stakers receive rewards accrued before they had shares.

**Root Cause**  
Pending fee value is not snapshotted to the reward-share set that existed when it accrued or when it first became processable. TokenRewards._setShares calls _processFeesIfApplicable before _addShares/_removeShares, but deferred DecentralizedIndex processing and caught _swapForRewards failures leave old fee balances outside rewardsDeposited and _rewardsPerShare while totalShares continues to change. When _depositRewards finally runs, it uses the then-current totalShares as the denominator.

**Pre_conditions**  
A pod has fee value pending for LP stakers. This can be pTKN fees in DecentralizedIndex that are below the processing threshold or within SWAP_DELAY, or paired-token proceeds already sent to TokenRewards after depositFromPairedLpToken catches a failed V3 reward swap. At least one existing staker was entitled to that fee value before a later share addition, and a later user can add enough staking shares before the fee value is successfully deposited.

**Impact**  
Existing LP stakers lose a pro-rata portion of fee rewards to later stakers. The value at risk is the uncheckpointed fee batch that is eventually processed with the stale denominator, bounded per processing pass by the pending fee balance but repeatable across fee accrual and failed-conversion cycles. If shares are removed while old fees are uncheckpointed, exiting users can similarly forfeit rewards to the remaining denominator.

**Proof of Concept**  
Threshold path: assume the V2 pool holds 1,000,000 pTKN, so the mainnet processing threshold is 1,000 pTKN, and honest stakers have 1 TokenRewards share. Prior transfers accumulate 999 pTKN of fees, which cannot process because _bal < _min. A late user stakes 99 shares. The StakingPoolToken._mint path calls TokenRewards.setShares; processPreSwapFeesAndSwap returns early, then TokenRewards adds the 99 shares. After 1 more pTKN of fees accrues, the next processing pass swaps and deposits the 1,000 pTKN reward batch using totalShares = 100, giving the late user 99% of a batch where 999 pTKN accrued before their stake existed.

Failed-conversion path: honest stakers have 1 share and a fee batch becomes eligible. A late staker triggers staking while the paired-token/rewardsToken V3 execution price is more than the 2% allowed deviation from the TWAP quote. setShares calls processPreSwapFeesAndSwap before adding the new shares; DecentralizedIndex swaps pTKN fees into PAIRED_LP_TOKEN and sends them to TokenRewards, but TokenRewards._swapForRewards catches the failed V3 swap, records only _rewardsSwapAmountInOverride, and leaves the paired tokens unaccounted. The late shares are then added. Once the V3 swap succeeds, depositFromPairedLpToken deposits the old fee value through _rewardsPerShare using the now larger totalShares, so the late staker receives a pro-rata share of rewards generated before entry.

**Mitigation**  
Checkpoint fee value before allowing reward shares to change. If DecentralizedIndex processing is delayed or below threshold, record the pending balance against the current share set or block share additions/removals until any nonzero pending balance is processed. If _swapForRewards catches a failed conversion, record the paired-token amount in an epoch owned by the current totalShares before returning, and release converted rewards to that epoch instead of the later denominator. More generally, _depositRewards should not be the first point where old fee value is assigned to whatever totalShares happens to be current.

### [M-02] Non-canonical pTKN markets pay only transfer tax instead of configured buy/sell fees

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/DecentralizedIndex.sol:161; contracts/contracts/DecentralizedIndex.sol:168; contracts/contracts/DecentralizedIndex.sol:171; contracts/contracts/DecentralizedIndex.sol:174

**Summary/Description**  
DecentralizedIndex classifies buys and sells only when a pTKN transfer is from or to the single stored V2_POOL. Transfers involving any other pTKN market are treated as ordinary transfers, so a trade through a non-canonical pool pays only the fixed 0.01% transfer tax when hasTransferTax is enabled, or no pod fee when it is disabled, instead of the configured buy or sell fee.

**Root Cause**  
The fee switch depends on exact address equality with V2_POOL rather than on whether the transfer is an economic buy or sell through any active pTKN liquidity market. The fallback transfer tax is a much smaller fixed fee and is also optional.

**Pre_conditions**  
A pod has nonzero buy or sell fees. A user can create or use another pTKN liquidity market that is not the stored V2_POOL, such as a pool on another V2-style factory or another AMM. If hasTransferTax is enabled, the fallback tax is lower than the configured buy/sell fee; if disabled, the alternate-market transfer pays no pod fee.

**Impact**  
Trading volume can move through non-canonical markets while avoiding the intended buy/sell fee accrual to the pod fee balance, burn share, partner share, and LP reward flow. For example, with a 10% sell fee and hasTransferTax enabled, selling 100 pTKN into V2_POOL collects 10 pTKN, while selling 100 pTKN into another pool collects only 0.01 pTKN. The saved fee is value that would otherwise accrue to holders/stakers through the configured fee path, and the route is repeatable up to the liquidity available in alternate markets.

**Proof of Concept**  
Set fees.sell = 1000 and hasTransferTax = true. A transfer of 100e18 pTKN to V2_POOL is classified as _sell and moves 10e18 pTKN to DecentralizedIndex as fees before sending 90e18 to the pool. The same 100e18 pTKN transfer to any other AMM pool address has _buy == false and _sell == false, so the fallback branch collects only 100e18 / 10000 = 0.01e18 pTKN before sending 99.99e18 to that pool. A buy from that alternate pool is classified the same way because _from is not V2_POOL, so it also avoids the configured buy fee.

**Mitigation**  
Do not rely on one stored pool address to identify taxable trades. Track all authorized pTKN liquidity pools and apply the appropriate buy/sell fee to transfers involving them, or make the non-pool transfer tax at least as strict as the intended trade fee unless the sender/receiver is an explicitly fee-exempt protocol path such as bond, debond, or managed liquidity operations.

### [M-03] Burn-fee processing can make threshold-crossing pTKN buys revert

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/DecentralizedIndex.sol:159; contracts/contracts/DecentralizedIndex.sol:165; contracts/contracts/DecentralizedIndex.sol:180; contracts/contracts/DecentralizedIndex.sol:217; contracts/contracts/DecentralizedIndex.sol:223; contracts/contracts/DecentralizedIndex.sol:228; contracts/contracts/dex/UniswapDexAdapter.sol:61

**Summary/Description**  
DecentralizedIndex burn-fee processing can cause canonical V2 pTKN buys to revert because _processBurnFee() calls _burn() while swap-and-fee processing remains enabled. When an accumulated fee balance crosses the _processPreSwapFeesAndSwap threshold during the nested burn update, the token attempts to swap pTKN fees through the same V2 pool that is currently executing the buy.

**Root Cause**  
In DecentralizedIndex._processBurnFee(), _burn(address(this), _burnAmt) routes through the overridden _update() with _swapAndFeeOn still set to 1. The nested _update(address(this), address(0), _burnAmt) can call _processPreSwapFeesAndSwap() before the burn balance is removed, so fee swapping is reachable from inside a burn-fee burn. During a V2 buy the outer transfer comes from V2_POOL, but this nested burn uses address(this) as _from and bypasses the intended buy-side skip at line 165.

**Pre_conditions**  
A pod has a nonzero burn fee and a nonzero buy fee. The pod has V2 liquidity and the pTKN fee balance held by DecentralizedIndex is below or at the fee-swap threshold before a buy. SWAP_DELAY has elapsed. The buy fee makes the held pTKN balance reach the _min threshold and produces a positive burn amount.

**Impact**  
Users cannot execute pTKN buys that cross the fee-processing threshold; their swaps revert and the canonical market can become unavailable for normal-sized buys until a non-buy path processes the pending fee balance. The failure is repeatable whenever residual fee balances accumulate near the threshold.

**Proof of Concept**  
Example with the default verification-pod-style fees burn=2000 and buy=50: let the V2 pool hold 1,000,000e18 pTKN, so mainnet _min is 1,000e18 and _max is 10,000e18. Existing fee residue in DecentralizedIndex is 999e18. A user buys enough pTKN for the buy fee to be 1e18, making _burnAmt = 0.2e18. The outer buy transfer has _from == V2_POOL, so the pre-swap call is skipped, the 1e18 fee is moved to address(this), and _processBurnFee(1e18) calls _burn(address(this), 0.2e18). That _burn enters _update(address(this), address(0), 0.2e18), sees balanceOf(address(this)) == 1,000e18, and calls _feeSwap(1,000e18). The adapter then tries to route a pTKN -> pairedLpToken swap through the same V2 pool while the pool is already in its buy swap, so the swap reverts. Even on a pool that did not reject the nested swap, swapping the full 1,000e18 before burning would leave insufficient balance for the subsequent 0.2e18 burn and revert.

**Mitigation**  
Do not run fee swapping from mint or burn updates. In _processBurnFee(), call a raw parent update for the burn or disable swap-and-fee processing around the internal _burn. Also consider processing accumulated fees before collecting the current buy fee, or reserving the burn amount so _feeSwap cannot consume tokens that are about to be burned.

### [M-04] Paired-token fee swaps can lock staking when no rewards shares exist

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/contracts/DecentralizedIndex.sol:185; contracts/contracts/DecentralizedIndex.sol:228; contracts/contracts/TokenRewards.sol:102; contracts/contracts/TokenRewards.sol:141; contracts/contracts/TokenRewards.sol:152; contracts/contracts/TokenRewards.sol:214; contracts/contracts/StakingPoolToken.sol:67; contracts/contracts/StakingPoolToken.sol:101; contracts/contracts/WeightedIndexFactory.sol:75

**Summary/Description**  
When a pod is configured to leave rewards as the paired LP token while PAIRED_LP_TOKEN differs from lpRewardsToken, DecentralizedIndex fee swapping sends paired-token output to TokenRewards and calls depositFromPairedLpToken. That path deposits PAIRED_LP_TOKEN as the reward token, but TokenRewards._depositRewards reverts while totalShares is zero because its zero-share branch only accepts rewardsToken. A user can accrue enough pod fees before any LP shares are staked, after which fee processing reverts and the first staking transaction cannot create the shares needed to clear the condition.

**Root Cause**  
In TokenRewards._depositRewards the totalShares == 0 branch assumes the deposited token must be rewardsToken. This is incompatible with the explicit leaveRewardsAsPairedLp mode, where depositFromPairedLpToken intentionally calls _depositRewards with PAIRED_LP_TOKEN. The issue is amplified because TokenRewards.setShares processes DecentralizedIndex fees before adding shares.

**Pre_conditions**  
A pod is deployed with leaveRewardsAsPairedLp set to true and PAIRED_LP_TOKEN not equal to lpRewardsToken. The pod has V2 liquidity but TokenRewards.totalShares is zero. DecentralizedIndex has accumulated a pTKN fee balance at or above the _processPreSwapFeesAndSwap threshold, and SWAP_DELAY has elapsed.

**Impact**  
The affected pod can enter a persistent liveness failure. The first or next LP staking attempt reverts before totalShares is increased, because StakingPoolToken._mint calls TokenRewards.setShares and setShares processes the stuck fee balance first. Normal pTKN transfers, sells, and debonds that reach fee processing can also revert, so pod holders can be unable to exit through the standard sell or debond paths until an upgrade or other out-of-band intervention fixes the accounting path.

**Proof of Concept**  
One concrete sequence is: deploy a pod with leaveRewardsAsPairedLp=true, PAIRED_LP_TOKEN=DAI, and lpRewardsToken=PEAS; add pTKN/DAI V2 liquidity; leave the staking pool with totalShares=0; have a user buy enough pTKN from the V2 pool to accumulate pTKN fees in DecentralizedIndex at or above _min; after SWAP_DELAY, the first user tries to stake LP tokens. StakingPoolToken.stake mints sTKN, its _update calls TokenRewards.setShares, setShares calls processPreSwapFeesAndSwap, DecentralizedIndex._feeSwap swaps pTKN to DAI and calls depositFromPairedLpToken, then TokenRewards._depositRewards(DAI, amount) reverts at require(_token == rewardsToken) because totalShares is still zero and rewardsToken is PEAS. The staking transaction reverts before shares can become nonzero, so repeating the same stake cannot clear the condition.

**Mitigation**  
Make zero-share reward handling token-consistent with leaveRewardsAsPairedLp. For non-rewardsToken paired deposits, either leave the paired balance pending without reverting until shares exist, or apply an explicit burn/admin policy for the paired token. Also avoid calling fee processing before the first share is added when that processing can depend on totalShares being nonzero.

### [M-05] Aerodrome exact-output swaps call an unsupported V2 router selector and can block leverage unwinds

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/contracts/dex/AerodromeDexAdapter.sol:17; contracts/contracts/dex/UniswapDexAdapter.sol:84; contracts/contracts/lvf/LeverageManager.sol:465; contracts/contracts/interfaces/IAerodromeRouter.sol:258

**Summary/Description**  
AerodromeDexAdapter inherits UniswapDexAdapter.swapV2SingleExactOut instead of implementing an Aerodrome-specific exact-output path. When LeverageManager needs to sell pTKN for the borrow token during removeLeverage, an Aerodrome pod calls the inherited helper, which invokes IUniswapV2Router02.swapTokensForExactTokens on the Aerodrome V2 router. The Aerodrome router interface only exposes exact-input V2 swaps, so the exact-output call reverts.

**Root Cause**  
The adapter inheritance assumes every V2 router supports the Uniswap exact-output selector. AerodromeDexAdapter overrides exact-input swaps and liquidity methods for Aerodrome routes, but leaves swapV2SingleExactOut on the Uniswap implementation.

**Pre_conditions**  
A leverage position uses a pod whose DEX_HANDLER is AerodromeDexAdapter. During removeLeverage, the paired token received from unstaking/removing LP is less than the flash repayment amount, so LeverageManager calls _swapPodForBorrowToken and then swapV2SingleExactOut.

**Impact**  
Affected Aerodrome leverage positions cannot complete normal deleveraging whenever repayment requires selling pTKN for the borrow token. The transaction reverts before the flash loan can be repaid, leaving the position's debt/collateral open until market conditions avoid the swap requirement or an out-of-band fix is deployed.

**Proof of Concept**  
Code path: LeverageManager._acquireBorrowTokenForRepayment calls _swapPodForBorrowToken when _pairedAmtReceived is insufficient. _swapPodForBorrowToken calls IDecentralizedIndex(_pod).DEX_HANDLER().swapV2SingleExactOut. For Aerodrome pods this dispatches to the inherited UniswapDexAdapter.swapV2SingleExactOut, which calls IUniswapV2Router02(V2_ROUTER).swapTokensForExactTokens. IAerodromeRouter defines swapExactTokensForTokens and swapExactTokensForTokensSupportingFeeOnTransferTokens, but no swapTokensForExactTokens exact-output method.

**Mitigation**  
Implement an Aerodrome-specific exact-output path or remove Aerodrome pods from leverage flows that require exact-output V2 swaps. If Aerodrome has no native exact-output route, quote the required input and perform a bounded exact-input swap with explicit refund/accounting, or route through a supported universal-router exact-output command.

### [M-06] Empty pod V2 pools can be one-sided synced to block official LP creation

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Low  
**Location**: contracts/contracts/DecentralizedIndex.sol:139; contracts/contracts/DecentralizedIndex.sol:141; contracts/contracts/DecentralizedIndex.sol:347; contracts/contracts/DecentralizedIndex.sol:348; contracts/contracts/dex/UniswapDexAdapter.sol:156; contracts/contracts/dex/AerodromeDexAdapter.sol:122; contracts/contracts/dex/CamelotDexAdapter.sol:15

**Summary/Description**  
New pod setup records an empty canonical V2 pool and later sends all official LP creation through the configured router. Before the first LP mint, an untrusted actor can transfer only one side of the pair to the pool and call sync, leaving one reserve nonzero and the other zero. V2-style routers then take the existing-pool branch and revert while quoting liquidity, so addLiquidityV2 cannot bootstrap the official LP/staking token through protocol paths.

**Root Cause**  
Pool setup only creates or accepts the canonical factory pool and stores it as V2_POOL; it does not atomically seed initial liquidity, lock the first mint, or verify that an empty totalSupply pool still has zero reserves before routing user liquidity through addLiquidity. The adapters delegate addLiquidity to V2 routers whose quote paths require both reserves to be nonzero once either reserve is nonzero.

**Pre_conditions**  
A pod's canonical pTKN/paired-token V2 pool exists and has no LP supply. This can be after setup creates the pool, after anyone calls setup for async adapters, or after a predicted future pod pair was created before deployment and then synced once the pod code exists. The attacker can transfer a dust amount of only one token side to the pool and call sync.

**Impact**  
All normal addLiquidityV2 and IndexUtils addLPAndStake attempts revert through the router until someone performs out-of-band pool recovery, such as direct pair mint/burn or reserve repair outside the protocol. This blocks official LP and staking bootstrap for the affected pod and is repeatable for each newly created pod/pool. User deposits in the reverting addLiquidityV2 transaction are rolled back, so the primary impact is persistent availability loss rather than direct theft.

**Proof of Concept**  
For a new pod pool with totalSupply == 0, attacker transfers 1 wei of PAIRED_LP_TOKEN to V2_POOL and calls sync on the pair. Reserves become (0, 1) or (1, 0). DecentralizedIndex.addLiquidityV2 later approves the DexAdapter and calls DEX_HANDLER.addLiquidity. UniswapDexAdapter and CamelotDexAdapter call the V2 router addLiquidity path; AerodromeDexAdapter calls Aerodrome router addLiquidity. In these routers, reserveA == 0 && reserveB == 0 is false, so they call quote/quoteLiquidity, which reverts when either reserve is zero. The whole addLiquidityV2 call reverts.

**Mitigation**  
Bootstrap the canonical V2 pool atomically with a small balanced initial mint during setup, or delay exposing the pool until the first protocol-controlled balanced mint completes. Alternatively, make addLiquidityV2 detect totalSupply == 0 with nonzero one-sided reserves and use a controlled recovery/direct-mint path or require zero reserves before accepting the pool as initialized.

### [M-07] Reserve-skewed auto-compounding failures let new aspTKN deposits capture pending rewards

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/AutoCompoundingPodLp.sol:124; contracts/contracts/AutoCompoundingPodLp.sol:213; contracts/contracts/AutoCompoundingPodLp.sol:311; contracts/contracts/AutoCompoundingPodLp.sol:331; contracts/contracts/AutoCompoundingPodLp.sol:340; contracts/contracts/AutoCompoundingPodLp.sol:392; contracts/contracts/dex/UniswapDexAdapter.sol:53; contracts/contracts/dex/CamelotDexAdapter.sol:28; contracts/contracts/dex/AerodromeDexAdapter.sol:44

**Summary/Description**  
AutoCompoundingPodLp tries to convert accrued reward balances into staked pod LP before deposits, mints, withdrawals, and redeems, but those entrypoints pass a zero minimum LP output and the conversion path catches swap/add-liquidity failures. The conversion is driven by current V2 reserves from the DEX adapter. When podOracle is configured, a temporary adverse skew in the pod/paired pool can make the paired-to-pod swap return below the TWAP-based minimum and revert; separately, _getSwapAmt treats reserve0 as the first function argument even though adapter reserves are pair-order, which can under-convert and leave residual balances when the paired token is not pair.token0. In these cases reward value remains outside _totalAssets while the share-changing action continues at the stale CBR.

**Root Cause**  
Reward conversion before ERC4626 share changes is best-effort instead of mandatory. Current reserve reads decide the one-sided swap amount, but failed swaps and failed addLPAndStake calls only emit events; unprocessed reward and residual token balances are not included in totalAssets or snapshotted to the holders that earned them.

**Pre_conditions**  
The aspTKN has accrued reward-token, paired-token, or residual pod/paired balances. Either podOracle is configured so an untrusted caller can temporarily move the pod/paired V2 spot price below the TWAP min-out for the conversion, or the pair token ordering/reserve ratio makes _getSwapAmt under-convert because it selects the wrong reserve. The caller then deposits or mints while conversion has been skipped or only partially accounted.

**Impact**  
Existing aspTKN holders can lose a pro-rata share of a pending reward batch to a later depositor. The later depositor receives shares using _totalAssets that excludes the skipped rewards, then can restore/wait for normal reserves and trigger a later successful conversion that increases _totalAssets for all current shares, including the newly minted ones. The value at risk is the unprocessed reward/residual balance, and the sequence is repeatable when rewards accumulate faster than they are converted.

**Proof of Concept**  
Sequence: rewards accrue to AutoCompoundingPodLp, but are not yet reflected in _totalAssets. The caller temporarily skews the pod/paired V2 pool reserves so the paired-to-pod swap cannot satisfy the TWAP-based min-out, then calls deposit(). deposit first calls _processRewardsToPodLp(0, block.timestamp). _getSwapAmt reads current reserves through DEX_ADAPTER.getReserves, and _pairedLpTokenToPodLp reverts inside swapV2Single; the catch block swallows the failure and _lpAmtOut remains below the pending reward value. deposit then mints shares at the old CBR. After the pool is restored, any later reward processing converts the same pre-existing balances and increments _totalAssets, so the new shares receive part of rewards that accrued before they entered. The same stale-accounting effect can arise without a revert when the unsorted reserve selection leaves residual pod/paired balances outside _totalAssets.

**Mitigation**  
Do not allow share supply changes to proceed with unaccounted pending rewards. If reward balances are nonzero, either require conversion to succeed with caller/owner supplied minimums, checkpoint pending rewards to the pre-existing share set, or include residual reward/pod/paired balances in totalAssets until they are converted. Also sort adapter reserves by the actual input token before using them in _getSwapAmt.

### [M-08] Zapper V3 multi-hop swaps bypass default slippage when min-out is zero

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/Zapper.sol:119; contracts/contracts/Zapper.sol:154; contracts/contracts/Zapper.sol:182; contracts/contracts/IndexUtils.sol:51

**Summary/Description**  
Zapper treats a zero _amountOutMin as a protected default in the single-hop V3 path by deriving a TWAP-based output and applying the configured pool/default slippage. The multi-hop V3 path receives the same _amountOutMin from _zap, but forwards it unchanged as Uniswap exactInput.amountOutMinimum. For an owner-configured V3 two-hop zap route, a caller using zero min-out gets amountOutMinimum = 0 instead of the default slippage guard.

**Root Cause**  
_swapV3Multi lacks the zero-min fallback and configured slippage application implemented in _swapV3Single, even though both helpers are reached through the same _zap API and the same caller-provided _amountOutMin parameter.

**Pre_conditions**  
A valid zapMap entry has poolType V3 and a nonzero pool2 for the input/paired-token route. A user calls IndexUtils.addLPAndStake with pairedLpTokenProvided different from the pod paired token and amountPairedLpTokenMin set to zero, or any derived flow calls _zap with zero min-out for that configured multi-hop route.

**Impact**  
The swap executes with no aggregate output floor, so a sandwich can force the user's input token through the route at an arbitrarily bad execution price subject only to pool mechanics. The user receives fewer paired LP tokens and therefore less LP/staking output, while excess pTKN is merely refunded; the lost value is captured through the manipulated swap. The impact is bounded to affected zap callers and does not directly drain shared protocol funds.

**Proof of Concept**  
In _zap, a V3 route with pool2 set calls _swapV3Multi(..., _amountOutMin). Unlike _swapV3Single, _swapV3Multi never computes a TWAP minimum when _amountOutMin == 0 and never applies _defaultSlippage/_slippage. It approves the input and calls ISwapRouter.exactInput with amountOutMinimum: _amountOutMin, so the router accepts any positive output when the caller supplied zero.

**Mitigation**  
Mirror the single-hop protection for multi-hop routes: when _amountOutMin is zero, quote or compute a route-level expected output and apply the configured slippage before calling exactInput. At minimum, reject zero min-out for multi-hop routes unless the caller explicitly opts into unlimited slippage.

### [M-09] StakingPoolToken share-update side effects misallocate unclaimed aspTKN rewards

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/StakingPoolToken.sol:72; contracts/contracts/StakingPoolToken.sol:77; contracts/contracts/StakingPoolToken.sol:101; contracts/contracts/StakingPoolToken.sol:104; contracts/contracts/StakingPoolToken.sol:107; contracts/contracts/TokenRewards.sol:102; contracts/contracts/TokenRewards.sol:113; contracts/contracts/TokenRewards.sol:115; contracts/contracts/TokenRewards.sol:126; contracts/contracts/TokenRewards.sol:128; contracts/contracts/TokenRewards.sol:236; contracts/contracts/TokenRewards.sol:252; contracts/contracts/AutoCompoundingPodLp.sol:124; contracts/contracts/AutoCompoundingPodLp.sol:126; contracts/contracts/AutoCompoundingPodLp.sol:130; contracts/contracts/AutoCompoundingPodLp.sol:135; contracts/contracts/AutoCompoundingPodLp.sol:136; contracts/contracts/AutoCompoundingPodLp.sol:162; contracts/contracts/AutoCompoundingPodLp.sol:164; contracts/contracts/AutoCompoundingPodLp.sol:176; contracts/contracts/AutoCompoundingPodLp.sol:178; contracts/contracts/AutoCompoundingPodLp.sol:190; contracts/contracts/AutoCompoundingPodLp.sol:197; contracts/contracts/AutoCompoundingPodLp.sol:199; contracts/contracts/AutoCompoundingPodLp.sol:213; contracts/contracts/IndexUtils.sol:150; contracts/contracts/IndexUtils.sol:154; contracts/contracts/IndexUtils.sol:157

**Summary/Description**  
AutoCompoundingPodLp prices ERC4626 deposits, mints, withdrawals, and redeems after only processing reward tokens already held by the vault. It does not first claim TokenRewards rewards owed to the vault or include those unpaid rewards in totalAssets. The StakingPoolToken asset transfer during deposit/mint and during reward compounding can distribute rewards owed to the vault after the deposit share price or loop balance snapshot has already been taken, letting new shares participate in prior rewards. The reverse direction also exists: withdraw/redeem burns and prices the exiting aspTKN shares before transferring StakingPoolToken assets out, and that outgoing transfer can claim the vaults unpaid rewards only after the exiting holder has been paid, leaving those prior rewards for the remaining aspTKN supply.

**Root Cause**  
StakingPoolToken transfers are not passive asset movements: _update calls TokenRewards.setShares for the sender and receiver, and TokenRewards distributes pending rewards before changing that wallets reward shares. AutoCompoundingPodLp treats the StakingPoolToken asset transfer as a passive ERC20 transfer and computes ERC4626 shares/assets from _totalAssets before forcing a TokenRewards claim or checkpoint for address(this). As a result, rewards can be materialized after the relevant aspTKN share supply change has already been priced.

**Pre_conditions**  
An AutoCompoundingPodLp vault has existing aspTKN supply and owns StakingPoolToken shares. TokenRewards has unpaid rewards for the vault address, while the vault token balance for those rewards is still zero or was already snapshotted earlier in _processRewardsToPodLp. A user then deposits or mints, withdraws or redeems, or a compounding pass stakes newly created LP tokens back to the vault.

**Impact**  
For deposits and mints, existing aspTKN holders lose a pro-rata portion of rewards that accrued before the new deposit. The new depositor mints shares at a CBR that excludes the just-claimed reward balance, then shares in the later conversion of that balance into additional staked LP. For withdrawals and redeems, the exiting holder can lose their pro-rata portion of rewards that accrued before exit because the outgoing StakingPoolToken transfer can claim those rewards for the vault after the exiting aspTKN shares were burned and paid. The value at risk is the unpaid reward batch for the vault and the path is repeatable whenever rewards accrue between vault interactions.

**Proof of Concept**  
Deposit example: the vault has 100 aspTKN supply, _totalAssets = 100 sTKN, and 100 sTKN shares in TokenRewards. TokenRewards has 100 reward tokens unpaid to the vault, but the vault balance is zero. A user deposits 100 sTKN. deposit first calls _processRewardsToPodLp, which sees no held reward balance, then computes 100 new aspTKN shares from _totalAssets = 100. _deposit increases _totalAssets to 200 and transfers the 100 sTKN to the vault. That transfer enters StakingPoolToken._update and TokenRewards._addShares(vault, 100, false); because the vault already had shares, _distributeReward sends the pre-existing 100 reward tokens to the vault before the new reward shares are added. Those rewards were not included in the deposit price. When a later call converts them into staked LP and increases _totalAssets, the new depositor owns half of the reward value even though it accrued before entry.

Withdraw example: the vault again has 100 aspTKN supply, _totalAssets = 100 sTKN, and 100 unpaid reward tokens claimable in TokenRewards but not yet held by the vault. A holder owning 50 aspTKN calls redeem(50). redeem processes only held balances, computes 50 sTKN out, burns the 50 aspTKN, and reduces _totalAssets to 50 before transferring 50 sTKN to the user. That outgoing StakingPoolToken transfer calls TokenRewards._removeShares(vault, 50), which first distributes the old 100 reward tokens to the vault and only then reduces the vaults reward shares. The exiting holder has already been paid and receives none of that prior reward batch; when the vault later compounds it, the remaining 50 aspTKN shares receive all 100 reward tokens.

**Mitigation**  
Before any ERC4626 share supply change, explicitly claim TokenRewards for the vault and either convert the claimed balances or include them in totalAssets before computing shares/assets. During _processRewardsToPodLp, avoid staking back into the vault in a way that can claim additional rewards after token balances have been snapshotted, or repeat the claim/processing loop until no newly claimed rewards remain. Alternatively checkpoint TokenRewards accruals to the aspTKN share set that earned them, so later deposits and exits cannot change ownership of already accrued rewards.

### [M-10] Paused reward checkpoints are erased on reward-tracking share changes

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/RewardsWhitelist.sol:29; contracts/contracts/TokenRewards.sol:113; contracts/contracts/TokenRewards.sol:123; contracts/contracts/TokenRewards.sol:126; contracts/contracts/TokenRewards.sol:134; contracts/contracts/TokenRewards.sol:236; contracts/contracts/TokenRewards.sol:243; contracts/contracts/TokenRewards.sol:258; contracts/contracts/StakingPoolToken.sol:67; contracts/contracts/StakingPoolToken.sol:72; contracts/contracts/StakingPoolToken.sol:101; contracts/contracts/voting/VotingPool.sol:114

**Summary/Description**  
TokenRewards skips paused reward tokens during distribution, but the same share-update paths still call _resetExcluded after adding or removing shares. Once the RewardsWhitelist owner pauses a deposited reward token, any tracked-share change can advance the wallet checkpoint past rewards that were never paid. In the StakingPoolToken path this is not limited to voluntary holder actions: OpenZeppelin ERC20 transferFrom permits value 0 with zero allowance, and unrestricted StakingPoolToken.stake(victim, 0) also mints zero shares to an arbitrary target before calling the reward-share hook. Either path can erase a victim staker's paused-token checkpoint while the token is paused.

**Root Cause**  
In TokenRewards._distributeReward the paused-token branch continues without paying rewards or updating that token checkpoint. However _addShares and _removeShares always call _resetExcluded after the share balance path completes, and _resetExcluded updates every reward token without checking paused state. StakingPoolToken._update invokes these paths for all transfers, including zero-value transfers, and TokenRewards._removeShares accepts _amount == 0 whenever the wallet has shares. StakingPoolToken.stake also lacks a nonzero amount check and accepts an arbitrary _user when stakeUserRestriction is unset, so stake(victim, 0) reaches the same _addShares/_resetExcluded path without victim approval. VotingPool mint and burn updates share the same checkpoint-reset root cause for non-transfer share changes.

**Pre_conditions**  
A reward token has already been deposited and accrued to one or more reward-share holders. The RewardsWhitelist owner pauses that token through setPaused, for example during an emergency or temporary distribution hold. For the StakingPoolToken path, a victim has nonzero staking shares; an attacker can call transferFrom(victim, attacker, 0) without allowance, or when stakeUserRestriction is unset can call stake(victim, 0) with no LP balance or allowance requirement under standard zero-value ERC20 transfer semantics. VotingPool holders are affected when their mint or burn style reward shares change during the pause.

**Impact**  
Affected stakers lose the paused-token rewards that accrued before the checkpoint reset. A permissionless zero-value transferFrom can target any StakingPoolToken holder with shares and repeat across holders, converting their unpaid paused-token rewards into residual balances stuck in TokenRewards. For a full unstake, all unpaid paused rewards for that wallet are erased from future claims; for a partial or zero share update, the wallet checkpoint is advanced to the current accumulator without payment. The value at risk is the unpaid paused-token reward batch for users whose checkpoints are advanced during the pause.

**Proof of Concept**  
Example: user1 has 100 staking shares with rewardsExcluded = 0. A whitelisted reward token is deposited so _rewardsPerShare implies 100 tokens owed to user1. The token is paused, so claimReward(user1) skips it. While paused, any account calls StakingPoolToken.transferFrom(user1, attacker, 0). ERC20Upgradeable._spendAllowance accepts zero value with zero allowance, _transfer executes _update with value 0, and StakingPoolToken._update calls TokenRewards.setShares(user1, 0, true). TokenRewards._removeShares calls _distributeReward, skips the paused token, keeps the same share balance, and then _resetExcluded sets user1 excluded to the current cumulative reward value. In unrestricted staking pools the same checkpoint erase can be triggered by stake(user1, 0): StakingPoolToken._mint(user1, 0) reaches _update with a zero value and TokenRewards.setShares(user1, 0, false), then the trailing zero-value stakingToken.transferFrom succeeds under standard ERC20 behavior. After unpause, getUnpaid for user1 returns zero for the previously accrued amount, and the 100 reward tokens remain in the contract without being attributable to user1.

**Mitigation**  
Do not let share changes advance or inherit paused-token reward checkpoints without first preserving the wallet-level entitlement that existed before the share change. Record skipped paused-token amounts as deferred realized rewards before changing checkpoints, or have _resetExcluded skip paused tokens until they are paid. Also ignore zero-value StakingPoolToken share updates, require stake amounts to be nonzero, and block zero-value transfer and transferFrom side effects so third parties cannot trigger reward-accounting changes for holders.

### [M-11] Hardcoded reward conversion pool can be price-manipulated

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/TokenRewards.sol:141; contracts/contracts/TokenRewards.sol:164; contracts/contracts/TokenRewards.sol:167; contracts/contracts/TokenRewards.sol:170; contracts/contracts/TokenRewards.sol:172; contracts/contracts/TokenRewards.sol:300; contracts/contracts/TokenRewards.sol:303; contracts/contracts/dex/AerodromeDexAdapter.sol:20; contracts/contracts/dex/AerodromeDexAdapter.sol:21; contracts/contracts/dex/AerodromeDexAdapter.sol:31; contracts/contracts/dex/AerodromeDexAdapter.sol:82; contracts/contracts/dex/AerodromeDexAdapter.sol:99; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:18; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:35; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:38; contracts/contracts/libraries/PoolAddressSlipstream.sol:32; contracts/contracts/libraries/PoolAddressSlipstream.sol:37; contracts/contracts/DecentralizedIndex.sol:228; contracts/contracts/lvf/LeverageFactory.sol:264; contracts/contracts/dex/UniswapDexAdapter.sol:39; contracts/contracts/dex/UniswapDexAdapter.sol:126

**Summary/Description**  
TokenRewards converts pending paired-token yield into rewardsToken through a deterministic V3 pool selected only from PAIRED_LP_TOKEN, rewardsToken, and the hardcoded 1% fee or tick-200 fallback. The same pool supplies the 10-minute TWAP quote and executes the swap. If that pool is missing, thin, or attacker-initialized at a bad price, an untrusted actor can make the TWAP and execution price reflect the manipulated pool, trigger depositFromPairedLpToken, and cause LP stakers to receive far fewer reward tokens while the paired-token value is captured through the pool.

**Root Cause**  
depositFromPairedLpToken does not use a configured trusted conversion pool or an independent oracle/liquidity check. It computes _pool through DEX_ADAPTER.getV3Pool using REWARDS_POOL_FEE or REWARDS_TICK_SPACING, reads that pool TWAP through V3_TWAP_UTILS.sqrtPriceX96FromPoolAndInterval, then passes only a 2% TWAP-derived minimum to DEX_ADAPTER.swapV3Single. On Aerodrome, the uint24 fee overload reverts, the fallback hardcodes REWARDS_TICK_SPACING = 200, PoolAddressSlipstream includes that tick spacing in the clone salt, and AerodromeDexAdapter.swapV3Single also hardcodes TICK_SPACING = 200 in the execution path. LeverageFactory can also replace a pod paired token with a FraxlendPair share token while keeping the same rewardsToken, creating paired/reward combinations that are unlikely to have a deep canonical V3 market.

**Pre_conditions**  
A pod has PAIRED_LP_TOKEN != rewardsToken and leaveRewardsAsPairedLp is false. TokenRewards holds paired tokens from DecentralizedIndex fee processing, flash fees, or a direct deposit. The deterministic paired/rewards V3 pool used by the adapter is absent, thin, or otherwise manipulable for the 10-minute TWAP window; this is particularly plausible for non-standard paired tokens such as self-lending FraxlendPair shares. An attacker can initialize or skew that pool and then trigger depositFromPairedLpToken through public reward processing.

**Impact**  
LP stakers lose reward value from the affected paired-token batch. The attacker can capture the paired token by controlling liquidity/trades in the manipulated conversion pool, while TokenRewards records only the reduced rewardsToken output as rewardsDeposited. The loss can approach most of a conversion batch when the selected pool is attacker-created or extremely thin, and the path is repeatable as new fee yield accrues.

**Proof of Concept**  
For a pod whose paired token is DAI and rewardsToken is PEAS, or a self-lending pod whose paired token is a FraxlendPair share and rewardsToken is PEAS, let TokenRewards hold 1,000 paired tokens from DecentralizedIndex._feeSwap. If the hardcoded 1% paired/PEAS pool is not a deep canonical pool, the attacker initializes or skews it so the 10-minute TWAP prices PEAS far above fair value, then supplies the PEAS-side liquidity. When any caller triggers depositFromPairedLpToken(0), TokenRewards reads that pool TWAP, sets amountOutMinimum to only 98% of the manipulated quote, and swaps the entire paired-token batch through the same pool. The pool returns only the small manipulated PEAS amount; TokenRewards deposits that amount for stakers, and the attacker withdraws or back-runs to capture the paired tokens.

**Mitigation**  
Store and validate a trusted reward-conversion route per paired/reward token pair, including minimum liquidity/observation requirements, instead of deriving an arbitrary hardcoded pool. Use an independent oracle or bounded external quote for amountOutMinimum, and reject conversion when the selected pool is missing, too young, too illiquid, or deviates from the independent price. For paired tokens without deep direct rewardsToken markets, route through a trusted intermediate such as the underlying asset or leave rewards as paired tokens.

### [M-12] aspTKN oracle ignores unprocessed compounding rewards and enables stale-CBR liquidations

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/AutoCompoundingPodLp.sol:96; contracts/contracts/AutoCompoundingPodLp.sol:100; contracts/contracts/AutoCompoundingPodLp.sol:203; contracts/contracts/AutoCompoundingPodLp.sol:213; contracts/contracts/TokenRewards.sol:236; contracts/contracts/TokenRewards.sol:325; contracts/contracts/oracle/aspTKNMinimalOracle.sol:24; fraxlend/src/contracts/FraxlendPairCore.sol:225; fraxlend/src/contracts/FraxlendPairCore.sol:535; fraxlend/src/contracts/FraxlendPairCore.sol:1117

**Summary/Description**  
aspTKN collateral is priced for Fraxlend through aspTKNMinimalOracle, which multiplies the spTKN/base price by AutoCompoundingPodLp.convertToShares(1e18). That conversion only reads the cached _totalAssets value. Rewards that are claimable from TokenRewards or already held by the aspTKN but not yet converted into staked pod LP are excluded from the CBR used by the oracle. Fraxlend liquidation decisions therefore use a stale, lower CBR even though the same pending reward value can be realized by whoever later owns the aspTKN shares.

**Root Cause**  
AutoCompoundingPodLp exposes ERC4626 conversion views from _totalAssets only, while reward value is realized lazily through TokenRewards.claimReward and _processRewardsToPodLp. aspTKNMinimalOracle uses this lazy conversion view directly in getPrices, and Fraxlend does not force a claim or compounding pass before consuming the exchange rate for solvency and liquidation checks.

**Pre_conditions**  
An aspTKN is used as Fraxlend collateral through aspTKNMinimalOracle. The aspTKN address has unpaid TokenRewards rewards or held reward-token balances that have not yet been converted into staked pod LP and added to _totalAssets. A borrower is close enough to the liquidation threshold that including the pending compounding value would keep the account solvent, while the cached _totalAssets CBR makes it appear liquidatable.

**Impact**  
Borrowers can be liquidated against an exchange rate that omits value already earned by the aspTKN collateral. The liquidator repays debt, receives the borrower's aspTKN collateral using the stale exchange rate, then can trigger reward claiming/compounding and redeem or hold shares that now include the previously ignored reward value. The loss is the wrongful liquidation bonus and any seized share of pending compounding rewards, and the condition can recur whenever meaningful rewards accrue before CBR is refreshed.

**Proof of Concept**  
Sequence: a leveraged borrower supplies aspTKN collateral. The vault owns spTKN shares in the staking rewards contract and accrues a reward batch, but no one has called claimReward for the aspTKN or converted the held rewards, so AutoCompoundingPodLp.totalAssets remains at the old staked-LP amount. A liquidator calls FraxlendPair.liquidate. _updateExchangeRate reads aspTKNMinimalOracle.getPrices, which uses IERC4626(ASP_TKN).convertToShares(1e18); that conversion uses _cbr = 1e18 * _totalAssets / totalSupply and ignores the pending reward batch. The stale higher aspTKN-per-base exchange rate makes _isSolvent return false. After receiving the seized aspTKN, the liquidator or any caller can call TokenRewards.claimReward(address(aspTKN)) and a later aspTKN redeem/deposit path runs _processRewardsToPodLp, adding the reward-derived LP to _totalAssets for the current aspTKN holders, including the liquidator.

**Mitigation**  
Make the oracle-facing CBR include all claimable and held reward value, or force a claim-and-compound/checkpoint step before Fraxlend consumes the aspTKN exchange rate. If compounding cannot be done in a view, maintain an accounting index for pending rewards owned by the pre-existing aspTKN supply and use that adjusted totalAssets in convertToShares for oracle pricing. Liquidation paths should not rely on a lazy-harvest CBR that excludes rewards transferable to the liquidator after seizure.

### [M-13] Public aspTKN actions can force zero-min reward swaps

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/AutoCompoundingPodLp.sol:54; contracts/contracts/AutoCompoundingPodLp.sol:67; contracts/contracts/AutoCompoundingPodLp.sol:68; contracts/contracts/AutoCompoundingPodLp.sol:124; contracts/contracts/AutoCompoundingPodLp.sol:148; contracts/contracts/AutoCompoundingPodLp.sol:162; contracts/contracts/AutoCompoundingPodLp.sol:176; contracts/contracts/AutoCompoundingPodLp.sol:182; contracts/contracts/AutoCompoundingPodLp.sol:213; contracts/contracts/AutoCompoundingPodLp.sol:226; contracts/contracts/AutoCompoundingPodLp.sol:249; contracts/contracts/AutoCompoundingPodLp.sol:254; contracts/contracts/AutoCompoundingPodLp.sol:255; contracts/contracts/AutoCompoundingPodLp.sol:272; contracts/contracts/AutoCompoundingPodLp.sol:274; contracts/contracts/AutoCompoundingPodLp.sol:279; contracts/contracts/AutoCompoundingPodLp.sol:284; contracts/contracts/AutoCompoundingPodLp.sol:311; contracts/contracts/AutoCompoundingPodLp.sol:323; contracts/contracts/AutoCompoundingPodLp.sol:368; contracts/contracts/AutoCompoundingPodLp.sol:373; contracts/contracts/AutoCompoundingPodLp.sol:382; contracts/contracts/AutoCompoundingPodLp.sol:423; contracts/contracts/AutoCompoundingPodLp.sol:424; contracts/contracts/AutoCompoundingPodLp.sol:458; contracts/contracts/AutoCompoundingPodLp.sol:462; contracts/contracts/AutoCompoundingPodLp.sol:463; contracts/contracts/AutoCompoundingPodLp.sol:464; contracts/contracts/TokenRewards.sol:325

**Summary/Description**  
AutoCompoundingPodLp exposes an owner-only processAllRewardsTokensToPodLp function with an aggregate minimum LP output, but the normal public deposit, mint, withdraw, and redeem paths call _processRewardsToPodLp with _amountLpOutMin = 0. During that processing, reward-token-to-paired-token swaps are executed with amountOutMinimum = 0, whitelisted reward-token V2 routes are also called with zero min-out, and the paired-token-to-pod V2 swap has no guard unless an optional podOracle has been manually configured. The owner-set maxSwap cap does not reliably bound this path: it is skipped when the input token is already the paired LP token, it is absent from the direct paired-token-to-pod swap in _pairedLpTokenToPodLp, the lpRewardsToken retry override can replace a capped amount after the maxSwap check, and once getAllRewardsTokens already contains pod.lpRewardsToken the loop appends the same token again and can process a second cap-sized chunk or replay a freshly written retry override in the same pass. setYieldConvEnabled also cannot atomically re-enable conversion and process backlogged rewards with the owner-supplied _lpMinOut, because it calls _processRewardsToPodLp before flipping yieldConvEnabled; on a false-to-true toggle the processor returns immediately and leaves the rewards for the next public zero-min action. Any caller who can trigger a public vault action can therefore force pending aspTKN rewards through manipulable pools with effectively unlimited slippage despite attempted owner-side safeguards.

**Root Cause**  
The compounding path treats the final LP-output minimum as optional caller input, but all public ERC4626 entrypoints hardcode it to zero. The AutoCompoundingPodLp REWARDS_SWAP_SLIPPAGE constant is never used, _tokenToPairedLpToken passes zero to DEX_ADAPTER.swapV3Single for pod.lpRewardsToken and to _swap for other reward tokens, and _pairedLpTokenToPodLp only computes a min pTKN output when podOracle is nonzero. maxSwap enforcement is incomplete and order-dependent: _tokenToPairedLpToken returns immediately for the paired LP token, _pairedLpTokenToPodLp calls DEX_ADAPTER.swapV2Single directly without clamping _pairedSwapAmt, _tokenToPairedSwapAmountInOverride is applied after the lpRewardsToken maxSwap clamp, and _processRewardsToPodLp does not deduplicate the historical reward-token list before appending pod.lpRewardsToken. The add-liquidity slippage is based on the post-swap token balances, so it does not bound value lost in the preceding reward swaps. The false-to-true setYieldConvEnabled path compounds this by evaluating _triggerRewards under the old disabled flag, so _lpMinOut and _deadline are ignored exactly when the owner tries to restart conversion with a protected flush.

**Pre_conditions**  
An aspTKN has accrued claimable or already held reward tokens. If the rewards are still in TokenRewards, any caller can first call claimReward(address(aspTKN)) because TokenRewards.claimReward is not restricted. The relevant reward-token/paired-token or paired-token/pod pool can be temporarily skewed, thin, or attacker-controlled. The attacker owns or obtains enough aspTKN asset or shares to call deposit, mint, withdraw, or redeem and trigger _processRewardsToPodLp with zero min-out.

**Impact**  
Existing aspTKN holders lose reward value from the affected compounding batch. A sandwich or controlled-liquidity pool can make the vault sell the reward token for far fewer paired tokens, after which the vault adds/stakes only the reduced LP amount and increases _totalAssets by that reduced output. The lost value is captured through the manipulated swap pool rather than credited to the vault. The path is repeatable as rewards accrue, and it affects the shared reward stream for all aspTKN holders.

**Proof of Concept**  
A vault holds or can claim 1,000 PEAS as pod.lpRewardsToken. The owner may have configured maxSwap to limit automatic swap size, but deposit still calls _processRewardsToPodLp(0, block.timestamp). The loop reaches PEAS, _tokenToPairedLpToken calls DEX_ADAPTER.swapV3Single(PEAS, paired, 10000, amountIn, 0, address(this)), and the adapter forwards amountOutMinimum = 0. If PEAS was capped, the resulting paired tokens are still passed to _pairedLpTokenToPodLp, where _pairedSwapAmt is computed and swapped directly from paired token into pod token without checking maxSwap[paired]. If PEAS already appears in getAllRewardsTokens, the unconditional appended pod.lpRewardsToken iteration can then process the remaining PEAS balance again, allowing up to a second cap-sized PEAS chunk in one public processing pass. If a previous PEAS conversion failure set _tokenToPairedSwapAmountInOverride above a newly configured cap, or the first PEAS pass writes a smaller retry override, the later PEAS pass applies that override after the cap check. If conversion had been disabled, the owner cannot remove this exposure in one transaction by calling setYieldConvEnabled(true, true, lpMinOut, deadline): _processRewardsToPodLp observes yieldConvEnabled == false and returns zero before enforcing lpMinOut, then the flag is set true. An attacker can then skew the relevant pool, trigger deposit with a minimal spTKN amount, let the vault accept the reduced output, then restore the pool and capture the value difference.

**Mitigation**  
Require slippage protection on all automatic reward conversions. Public ERC4626 entrypoints should not process nonzero pending rewards with a zero minimum unless the loss is independently bounded. Use trusted routes and oracle/TWAP-derived minimums for reward-token-to-paired-token swaps, apply the existing REWARDS_SWAP_SLIPPAGE constant or remove it, require podOracle or another independent guard for paired-token-to-pod swaps, and consider moving auto-compounding behind a permissionless function that takes caller-supplied per-route min-outs before shares are minted or burned. Deduplicate the processing token list before appending pod.lpRewardsToken, enforce maxSwap after applying any retry override and once per processing pass, clamp or clear _tokenToPairedSwapAmountInOverride when setMaxSwap lowers a cap, and apply maxSwap to the paired-token swap amount inside _pairedLpTokenToPodLp. For setYieldConvEnabled, set the new flag before processing when enabling or allow a guarded internal processing path that honors _lpMinOut while the stored flag is still false.

### [M-14] Secondary reward balances can lock aspTKN deposits and withdrawals

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/contracts/AutoCompoundingPodLp.sol:124; contracts/contracts/AutoCompoundingPodLp.sol:148; contracts/contracts/AutoCompoundingPodLp.sol:162; contracts/contracts/AutoCompoundingPodLp.sol:176; contracts/contracts/AutoCompoundingPodLp.sol:213; contracts/contracts/AutoCompoundingPodLp.sol:217; contracts/contracts/AutoCompoundingPodLp.sol:220; contracts/contracts/AutoCompoundingPodLp.sol:226; contracts/contracts/AutoCompoundingPodLp.sol:249; contracts/contracts/AutoCompoundingPodLp.sol:258; contracts/contracts/AutoCompoundingPodLp.sol:260; contracts/contracts/AutoCompoundingPodLp.sol:265; contracts/contracts/AutoCompoundingPodLp.sol:266; contracts/contracts/AutoCompoundingPodLp.sol:303; contracts/contracts/AutoCompoundingPodLp.sol:308; contracts/contracts/AutoCompoundingPodLp.sol:311; contracts/contracts/AutoCompoundingPodLp.sol:316; contracts/contracts/AutoCompoundingPodLp.sol:319; contracts/contracts/AutoCompoundingPodLp.sol:323; contracts/contracts/AutoCompoundingPodLp.sol:368; contracts/contracts/AutoCompoundingPodLp.sol:377; contracts/contracts/AutoCompoundingPodLp.sol:379; contracts/contracts/dex/UniswapDexAdapter.sol:61; contracts/contracts/dex/UniswapDexAdapter.sol:77; contracts/contracts/dex/UniswapDexAdapter.sol:78; contracts/contracts/oracle/spTKNMinimalOracle.sol:104; contracts/contracts/oracle/spTKNMinimalOracle.sol:170; contracts/contracts/oracle/spTKNMinimalOracle.sol:173; contracts/contracts/oracle/spTKNMinimalOracle.sol:175; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:29; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:32; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:55; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:59; contracts/contracts/oracle/DIAOracleV2SinglePriceOracle.sol:22; contracts/contracts/TokenRewards.sol:180; contracts/contracts/TokenRewards.sol:189; contracts/contracts/TokenRewards.sol:210; contracts/contracts/TokenRewards.sol:331; contracts/contracts/RewardsWhitelist.sol:35

**Summary/Description**  
AutoCompoundingPodLp processes reward balances before deposit, mint, withdraw, and redeem. Several unprocessable reward states can therefore block the ERC4626 action itself. Secondary reward tokens are routed through uncaught V2 swaps, so missing liquidity, invalid routes, same-token self-lending paths, or token-level transfer restrictions such as a reward token blocking the aspTKN, adapter, router, or pool can make every public share-changing action revert while the balance remains. Separately, when podOracle is configured, _pairedLpTokenToPodLp calls podOracle.getPodPerBasePrice() before entering the paired-token-to-pod swap try/catch, so stale or bad oracle data can also revert public share-changing actions.

**Root Cause**  
Automatic reward processing is a mandatory pre-step for public share changes, but not every reward-processing dependency is isolated. _processRewardsToPodLp has no global skip/rescue path for unprocessable balances, secondary reward swaps are not caught, and the podOracle price read is outside the local swap try/catch. In the self-lending branch, _swapOutputTkn is changed to the Fraxlend pair asset, but the function only returns early for _token == pod.PAIRED_LP_TOKEN and misses _token == _swapOutputTkn, so already-correct underlying assets are treated as swap inputs.

**Pre_conditions**  
The aspTKN has yieldConvEnabled true. For the secondary-token path, a reward token different from pod.lpRewardsToken has been added to TokenRewards.getAllRewardsTokens by an earlier valid deposit; it can be currently whitelisted or later removed, because RewardsWhitelist.toggleRewardsToken(false) does not prune the historical rewards list. The aspTKN holds an unprocessable balance of that listed token. The balance can be unprocessable because liquidity/route assumptions fail, because the token-level transfer policy blocks movement from the aspTKN through the adapter/router/pool path, or because in a self-lending pod the token is already the lending pair asset itself, such as DAI for an fDAI paired pod, which makes the adapter/router receive a tokenIn == tokenOut swap path. For the stale podOracle path, podOracle is configured, the aspTKN holds a processable reward balance that reaches _pairedLpTokenToPodLp, and the underlying oracle stack reports bad/stale data or otherwise reverts.

**Impact**  
All public ERC4626 share-changing flows that call _processRewardsToPodLp can revert: deposit, mint, withdraw, and redeem. Existing holders are unable to exit through the normal vault path while the unprocessable balance remains and compounding is enabled. The owner can globally disable yield conversion, clear podOracle, or wait for fresh oracle data in the stale-oracle variant, but the blockage is repeatable whenever a processable listed reward balance is present. For the self-lending asset case this is not limited to dust: any listed borrow-asset reward balance can trigger the invalid same-token swap path, even if the token has since been removed from the active RewardsWhitelist.

**Proof of Concept**  
Dust secondary-token sequence: the owner has whitelisted WETH or another secondary reward token for pod LP rewards. An external caller calls TokenRewards.depositRewards(WETH, 1) if needed so WETH is included in getAllRewardsTokens. The owner may later unwhitelist WETH, but that does not remove the historical list entry. The caller then transfers 1 wei WETH directly to AutoCompoundingPodLp. A holder calls withdraw or redeem. _processRewardsToPodLp sees WETH, _tokenToPairedLpToken routes it into _swap(WETH, paired, 1, 0), and the uncaught V2 adapter path can revert for zero output or missing liquidity, reverting the user action. Restricted-token variant: after a secondary reward token has entered the historical list and a balance sits in the aspTKN, the token starts rejecting transfers from the aspTKN or through the adapter/router/pool path. The same uncaught _swapV2 branch bubbles the transfer revert into deposit, mint, withdraw, and redeem. Self-lending asset sequence: if the secondary reward token is the Fraxlend pair asset, such as DAI for an fDAI paired pod, _tokenToPairedLpToken sets _swapOutputTkn = IFraxlendPair(paired).asset() and then calls _swap(DAI, DAI, amount, 0). UniswapV2-style routing rejects identical-address paths, so the revert bubbles up before the code can call _depositIntoLendingPair. Stale podOracle sequence: podOracle is set and a processable reward balance reaches _pairedLpTokenToPodLp; getPodPerBasePrice() reverts or divides by zero before the DEX swap try/catch, rolling back the user action.

**Mitigation**  
Do not let mandatory public share changes depend on unbounded reward-processing success. Move reward conversion to a separate function with caller-supplied per-route minimums, or make public entrypoints skip/checkpoint unprocessable rewards without blocking deposits and withdrawals. Add per-token minimums, rescue/ignore controls, and consistent try/catch handling for secondary reward swaps. Treat restricted-transfer reward tokens as unsupported at whitelist time or isolate them behind a safe skip/pause path that does not erase accrued rewards. In the self-lending path, if _token == IFraxlendPair(pod.PAIRED_LP_TOKEN()).asset(), bypass _swap and call _depositIntoLendingPair directly. For podOracle, expose and consume a bad-data flag or wrap the oracle read so stale data causes conversion to be skipped or explicitly handled before user share changes.

### [M-15] Failed aspTKN compounding can reclassify or strand rewards

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/AutoCompoundingPodLp.sol:213; contracts/contracts/AutoCompoundingPodLp.sol:221; contracts/contracts/AutoCompoundingPodLp.sol:237; contracts/contracts/AutoCompoundingPodLp.sol:239; contracts/contracts/AutoCompoundingPodLp.sol:241; contracts/contracts/AutoCompoundingPodLp.sol:244; contracts/contracts/AutoCompoundingPodLp.sol:311; contracts/contracts/AutoCompoundingPodLp.sol:323; contracts/contracts/AutoCompoundingPodLp.sol:327; contracts/contracts/AutoCompoundingPodLp.sol:328; contracts/contracts/AutoCompoundingPodLp.sol:331; contracts/contracts/AutoCompoundingPodLp.sol:340; contracts/contracts/AutoCompoundingPodLp.sol:410

**Summary/Description**  
AutoCompoundingPodLp charges its protocol fee immediately after rewards are converted into pod.PAIRED_LP_TOKEN(), before the paired tokens are successfully converted into staked pod LP and before _totalAssets is increased. The paired-token-to-pod-LP path catches swap and addLPAndStake failures and returns zero instead of reverting. If the paired token is scanned by later reward processing, repeated failed attempts can reclassify the same pending reward balance as protocol fees. If the failed balance came from a secondary reward and the paired token is not scanned as a reward token or lpRewardsToken, the net paired balance and any pTKN acquired before the addLPAndStake failure remain idle in the aspTKN with no direct retry or rescue path.

**Root Cause**  
The fee is accounted in _tokenToPodLp before _pairedLpTokenToPodLp has proven that the net paired amount was actually compounded. Failed compounding is best-effort and swallowed, while _protocolFees is permanent state. The retry loop only scans TokenRewards.getAllRewardsTokens() plus pod.lpRewardsToken(), so residual paired-token or pTKN balances created inside a failed conversion are not always re-entered into the compounding path.

**Pre_conditions**  
The aspTKN has a processable reward balance that reaches the paired-token stage. The subsequent paired-token-to-pod-LP conversion fails inside a caught branch, for example because the paired-to-pTKN swap fails, the podOracle min-out is not met, or addLPAndStake reverts due to ratio or slippage. For the protocol-fee reclassification variant, the paired token is later scanned by _processRewardsToPodLp, such as when it is pod.lpRewardsToken or has been added to the rewards token list. For the stranded-balance variant, the source reward is a secondary token and pod.PAIRED_LP_TOKEN is not scanned by the loop.

**Impact**  
Existing aspTKN holders lose pending reward value. In the retryable paired-token variant, the default 5% protocolFee can be charged repeatedly against the same unconverted balance and the owner can later withdraw the reclassified amount as protocol fees. In the non-scanned secondary-reward variant, the net paired tokens and residual pTKN remain outside _totalAssets and outside the retry loop, so they do not back vault shares unless a later unrelated reward conversion happens to reach the same paired-to-LP stage. The failure can be triggered by public deposit, mint, withdraw, redeem, or owner processing while yield conversion is enabled.

**Proof of Concept**  
No test run. Static path: a secondary reward token is converted to pod.PAIRED_LP_TOKEN. _tokenToPodLp books _protocolFees, then _pairedLpTokenToPodLp swaps part of the paired token to pTKN and calls indexUtils.addLPAndStake. If addLPAndStake reverts, the catch emits AddLpAndStakeError and returns zero; _totalAssets is not increased. The aspTKN still holds the paired-token balance and pTKN. If pod.PAIRED_LP_TOKEN is not in TokenRewards.getAllRewardsTokens() and is not pod.lpRewardsToken(), later _processRewardsToPodLp calls skip both balances. If the paired token is scanned, later retries subtract _protocolFees and charge protocolFee again on the remaining unconverted balance, progressively moving the same reward batch into _protocolFees.

**Mitigation**  
Only book _protocolFees after _pairedLpTokenToPodLp succeeds and returns nonzero staked LP, or keep fees in a pending local variable and roll them back when conversion fails. Track pending paired-token and pTKN residuals as retryable reward inventory, include them in _processRewardsToPodLp, or expose a safe recovery/checkpoint path. Alternatively revert reward processing on failed paired-token compounding so a failed conversion cannot silently mutate fee state or leave unaccounted balances.

### [M-16] Permissionless aspTKN and oracle CREATE2 deployment can preempt LVF setup

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Low  
**Location**: contracts/contracts/AutoCompoundingPodLpFactory.sol:19; contracts/contracts/AutoCompoundingPodLpFactory.sol:28; contracts/contracts/AutoCompoundingPodLpFactory.sol:70; contracts/contracts/AutoCompoundingPodLpFactory.sol:76; contracts/contracts/AutoCompoundingPodLpFactory.sol:80; contracts/contracts/oracle/aspTKNMinimalOracleFactory.sol:12; contracts/contracts/oracle/aspTKNMinimalOracleFactory.sol:16; contracts/contracts/oracle/aspTKNMinimalOracleFactory.sol:40; contracts/contracts/oracle/aspTKNMinimalOracleFactory.sol:46; contracts/contracts/oracle/aspTKNMinimalOracleFactory.sol:50; contracts/contracts/lvf/LeverageFactory.sol:90; contracts/contracts/lvf/LeverageFactory.sol:103; contracts/contracts/lvf/LeverageFactory.sol:118; contracts/contracts/lvf/LeverageFactory.sol:137; contracts/contracts/lvf/LeverageFactory.sol:215; contracts/contracts/lvf/LeverageFactory.sol:231; contracts/contracts/lvf/LeverageFactory.sol:241

**Summary/Description**  
AutoCompoundingPodLpFactory and aspTKNMinimalOracleFactory expose permissionless deterministic deployment. A third party cannot deploy different bytecode to the same CREATE2 address, but they can call either factory first with the exact constructor arguments and salt that LeverageFactory will later use. LeverageFactory hardcodes salt 0 and always calls create() instead of accepting an already deployed aspTKN or oracle, so the intended LVF setup reverts once either deterministic address has been occupied.

**Root Cause**  
The deterministic deployment namespaces are public and caller-independent, while the composed LVF setup assumes the target aspTKN and oracle have not already been deployed. AutoCompoundingPodLpFactory.create() and aspTKNMinimalOracleFactory.create() have no access control or reservation mechanism, and LeverageFactory uses fixed salt 0 plus create-only state-changing paths.

**Pre_conditions**  
For addLvfSupportForPod, a target pod exists and the third party knows or observes the pod, dex adapter, index utils, derived aspTKN name/symbol, oracle immutables, and salt 0. If minimumDepositAtCreation is nonzero, preempting the aspTKN requires the small required spTKN seed; preempting only the oracle does not require that seed. For createSelfLendingPodAndAddLvf, the third party knows or observes the planned pod constructor name/symbol, adapter, utils, oracle immutables, and salt 0; the aspTKN factory skips the creation seed because pod is address(0).

**Impact**  
The official LeverageFactory path for adding LVF support to that pod, or creating that self-lending pod configuration, reverts at the aspTKN or oracle CREATE2 call because code already exists at the deterministic address. This blocks the intended setup flow until trusted operators perform out-of-band recovery such as deploying different factory plumbing, changing code to use a different salt, or manually wiring the predeployed aspTKN/oracle/pair/LeverageManager ownership.

**Proof of Concept**  
A third party computes the same aspTKN address with getNewCaFromParams(name, symbol, isSelfLending, pod, dexAdapter, indexUtils, 0), then calls AutoCompoundingPodLpFactory.create() with those exact values. Alternatively, they compute the same oracle address from getNewCaFromParams(aspTKN, requiredImmutables, optionalImmutables, 0), then call aspTKNMinimalOracleFactory.create() with those exact values. The deployment succeeds and create() transfers ownership to the factory owner. Later, LeverageFactory.addLvfSupportForPod() or createSelfLendingPodAndAddLvf() reaches its create-only factory call with the same deployer, salt, and init code, so create2 returns zero/reverts because the destination already contains code, and the setup transaction rolls back.

**Mitigation**  
Make the deterministic namespaces non-preemptable for composed setup. Options include restricting create() to an authorized deployer, deriving salts from an authorized creator or a factory-controlled nonce, exposing salt parameters in LeverageFactory and reserving them before use, or making LeverageFactory check code.length at the predicted address and validate/reuse an existing aspTKN or oracle whose immutable configuration and ownership are exactly expected.

### [M-17] Floor-rounded LAV withdrawals become value-leaking under inflated CBR

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/LendingAssetVault.sol:82; contracts/contracts/LendingAssetVault.sol:86; contracts/contracts/LendingAssetVault.sol:120; contracts/contracts/LendingAssetVault.sol:136; contracts/contracts/LendingAssetVault.sol:176; contracts/contracts/LendingAssetVault.sol:184; contracts/contracts/LendingAssetVault.sol:287; contracts/contracts/LendingAssetVaultFactory.sol:20; contracts/contracts/LendingAssetVaultFactory.sol:32; contracts/contracts/LendingAssetVaultFactory.sol:35

**Summary/Description**  
LendingAssetVault uses floor-rounded conversion helpers for deposit, mint, and withdraw. deposit() correctly rejects zero-share deposits before pulling assets, so direct raw donations to the LAV do not create the classic ERC4626 victim-donation path because _totalAssets is stored separately from token balance. The factory seed also does not reliably prevent a dust-supply state: create() is permissionless and _depositMin() mints the initial shares to _msgSender(), so the seed holder can redeem before honest deposits and return supply/assets to zero. The remaining exploitable surface is the existing rounded withdraw/mint behavior when LAV supply is dust or the CBR is heavily inflated through external-vault accounting.

**Root Cause**  
withdraw() calls convertToShares(), and mint() calls convertToAssets(), but both helpers round down. ERC4626-style mint and withdraw paths must round against the caller: mint should round assets up and withdraw should round shares up. _withdraw also accepts zero shares, so a zero-share result is not rejected before asset transfer.

**Pre_conditions**  
The LAV has positive available assets and CBR > 1. Material impact requires the value of one share unit to be non-dust, for example a dust/low totalSupply state combined with large positive external-vault CBR growth recorded by _updateAssetMetadataFromVault. A dust/low supply state can come from the caller-owned factory seed being redeemed, or from minimumDepositAtCreation being set to zero, before honest deposits. No LAV shares are required for the zero-share withdraw subcase because the attacker can pass owner = msg.sender and _burn(msg.sender, 0) succeeds.

**Impact**  
A caller can withdraw assets while burning fewer shares than the ERC4626 ceil-rounded amount. The value leakage per call is bounded by less than one share unit of value, so normal near-1 CBR states leak only dust; under a share-inflated/dust-supply CBR, that one-share-unit value can become material and can be used to extract value from LAV holders. The mint path has the same wrong direction and can undercharge by up to one asset unit per call.

**Proof of Concept**  
Assume an inflated state where totalSupply = 1,000 share units and _totalAssets = 1,000e18 asset units after external-vault CBR growth, so one share unit is worth 1e18 assets and totalAvailableAssets is nonzero. An attacker calls withdraw(1e18 - 1, attacker, attacker). withdraw() computes _shares = floor((1e18 - 1) * 1e27 / 1e45) = 0, then _withdraw() subtracts the assets, burns 0 shares from attacker, and transfers the assets. With ordinary CBR near 1 this same bug only leaks dust, which is why the meaningful case is the share-inflated CBR state.

**Mitigation**  
Use explicit rounding directions. Keep convertToShares/convertToAssets floor-rounded for the ERC4626 convert functions, but make previewWithdraw/withdraw compute shares with ceiling rounding and make previewMint/mint compute assets with ceiling rounding, as AutoCompoundingPodLp already does. Also reject zero-share withdrawals and zero-share mints in the state-changing paths. If factory seeding is intended as first-depositor protection, mint the creation seed to a permanently locked address or add virtual asset/share offsets instead of minting redeemable shares to the create caller.

### [M-18] LAV down-CBR math over-socializes Fraxlend bad debt

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/LendingAssetVault.sol:287; contracts/contracts/LendingAssetVault.sol:293; contracts/contracts/LendingAssetVault.sol:325; contracts/contracts/LendingAssetVault.sol:328; contracts/contracts/LendingAssetVault.sol:329; contracts/contracts/LendingAssetVault.sol:331; contracts/contracts/LendingAssetVault.sol:332

**Summary/Description**  
When a whitelisted vault CBR decreases, LendingAssetVault tries to lower vaultUtilization and _totalAssets by applying a ratio-change delta. The increase path is equivalent to multiplying utilization by newCbr / oldCbr, but the decrease path computes oldCbr / newCbr - 1 and subtracts that from current utilization. That is not equivalent to currentUtilization * newCbr / oldCbr, so losses are overstated whenever CBR drops. redeemFromVault materializes the same error: it checkpoints with the understated utilization, redeems shares for the true asset amount, and then only subtracts min(vaultUtilization, assetsReceived), leaving any received surplus outside _totalAssets.

**Root Cause**  
The CBR decrease branch uses the inverse ratio as a delta: (oldCbr / newCbr) - 1. Correct accounting should either compute the new utilized value directly from shares or scale current utilization by newCbr / oldCbr. The same incorrect branch is duplicated in the preview and state-update functions.

**Pre_conditions**  
A whitelisted Fraxlend pair or other ERC4626 vault has nonzero LAV utilization and its CBR decreases, which the contest README explicitly allows in bad-debt cases. The LAV metadata for that vault is refreshed through whitelistDeposit, whitelistWithdraw, depositToVault, redeemFromVault, whitelistUpdate(true), or a global update that reaches _updateAssetMetadataFromVault.

**Impact**  
LAV holders absorb more than the actual bad debt. For oldCbr 1.0 and newCbr 0.9, 100 utilized assets should become 90, but the code records about 88.89. If redeemFromVault then redeems the LAV shares, the vault can receive the actual 90 assets while _totalAssets remains 88.89 after utilization is cleared; the difference becomes untracked local balance that holders cannot redeem through normal accounting. For a 50% CBR drop, the code can set the utilized value to zero even though the LAV-owned pair shares are still worth half of the original amount.

**Proof of Concept**  
Let vaultUtilization be 100 and the stored vault CBR be 1e27. After bad debt, the pair CBR is 0.9e27. Correct value is 100 * 0.9 / 1.0 = 90. The LAV code computes ratioChange = 1e27 * 1e27 / 0.9e27 - 1e27 = 0.111...e27 and records 100 - 11.111... = 88.888..., overstating the loss by 1.111.... If the owner calls redeemFromVault for all LAV-owned pair shares, Fraxlend redeem returns about 90 assets, but _redeemAmt is capped to the understated 88.888..., vaultUtilization becomes zero, and _totalAssets stays 88.888... while the LAV now holds 90 local assets.

**Mitigation**  
Replace the delta branch with direct scaling, e.g. newUtilization = currentUtilization * newCbr / oldCbr for decreases and increases, with explicit rounding policy. Prefer deriving the value from the LAV's actual vault share balance and convertToAssets, then update _totalAssets and _totalAssetsUtilized from old utilized value to the exact new utilized value. Apply the same correction to _previewAddInterestAndMdInAllVaults.

### [M-19] Just-in-time LAV deposits can manipulate Fraxlend accrual and capture pending yield

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/LendingAssetVault.sol:73; contracts/contracts/LendingAssetVault.sol:106; contracts/contracts/LendingAssetVault.sol:120; contracts/contracts/LendingAssetVault.sol:136; contracts/contracts/LendingAssetVault.sol:152; contracts/contracts/LendingAssetVault.sol:204; contracts/contracts/LendingAssetVault.sol:211; contracts/contracts/LendingAssetVault.sol:220; contracts/contracts/LendingAssetVault.sol:287; contracts/contracts/LendingAssetVault.sol:303; fraxlend/src/contracts/libraries/VaultAccount.sol:16; fraxlend/src/contracts/libraries/VaultAccount.sol:20; fraxlend/src/contracts/FraxlendPairCore.sol:286; fraxlend/src/contracts/FraxlendPairCore.sol:313; fraxlend/src/contracts/FraxlendPairCore.sol:319; fraxlend/src/contracts/FraxlendPairCore.sol:397; fraxlend/src/contracts/FraxlendPairCore.sol:410; fraxlend/src/contracts/FraxlendPairCore.sol:442; fraxlend/src/contracts/FraxlendPairCore.sol:455; fraxlend/src/contracts/FraxlendPairCore.sol:486; fraxlend/src/contracts/FraxlendPairCore.sol:956; fraxlend/src/contracts/FraxlendPairCore.sol:959; fraxlend/src/contracts/FraxlendPairCore.sol:1042; fraxlend/src/contracts/FraxlendPairCore.sol:1059; fraxlend/src/contracts/FraxlendPairCore.sol:1066

**Summary/Description**  
LendingAssetVault share-changing calls try to checkpoint whitelisted Fraxlend pairs through FraxlendPair.addInterest(false), then price LAV shares from stored _totalAssets. That checkpoint is not sufficient: the public Fraxlend addInterest path can skip on the utilization gate, and it also returns zero when the pair was already internally accrued earlier in the same block. A user can therefore force or backrun a pair accrual that raises fToken CBR without refreshing LAV metadata, deposit into LAV at the stale lower CBR, and then trigger a zero-sized repay/whitelist update so the newly minted LAV shares capture pre-entry Fraxlend yield. Public LAV deposits or withdrawals can also change the external capacity that Fraxlend uses for the next elapsed-interest calculation, letting JIT liquidity lower or raise the utilization used for the whole interval.

**Root Cause**  
The LAV treats a positive interestEarned return from FraxlendPair.addInterest(false) as the condition for refreshing vault metadata, but the returned value is not equivalent to current pair CBR freshness. Fraxlend public addInterest is utilization-gated, while internal _addInterest can move totalAsset.amount and totalBorrow.amount through other pair entry points without notifying LAV. Because _updateInterestAndMdInAllVaults skips _updateAssetMetadataFromVault when interestEarned is zero, LAV deposits/mints/withdraws/redeems can execute against stale _totalAssets even when IFraxlendPair.convertToAssets(PRECISION) already differs from _vaultWhitelistCbr. Separately, Fraxlend _calculateInterest reads externalAssetVault.totalAvailableAssetsForVault(address(this)) at accrual time and applies that current capacity to the full elapsed deltaTime.

**Pre_conditions**  
A Fraxlend pair is whitelisted in a LendingAssetVault and has externalAssetVault set to that LAV. The LAV has nonzero exposure to the pair and the pair has pending or just-realized CBR growth from borrow interest. For the stale-CBR capture path, the attacker can either force internal accrual with a zero/dust addCollateral call or backrun another pair action that calls internal _addInterest without refreshing LAV metadata, then deposit/mint LAV shares before a later whitelist metadata update. Immediate extraction requires enough idle LAV assets to redeem the gained value, but the accounting gain exists once metadata is refreshed.

**Impact**  
Existing LAV holders can be diluted by a just-in-time depositor that receives too many shares before pending Fraxlend yield is recorded in LAV _totalAssets. The attacker can then trigger the metadata update and redeem against the higher CBR when idle LAV liquidity is available. The captured amount is a pro-rata share of accrued Fraxlend yield, and the sequence is repeatable whenever interest accrues between LAV metadata refreshes. In the capacity-manipulation variant, temporary LAV deposits reduce measured utilization for the elapsed period and temporary withdrawals increase it, shifting interest and rate growth between borrowers and lenders based on liquidity that was not present for the full interval.

**Proof of Concept**  
Stale-CBR capture example: LAV has 1000 assets of totalAssets, 1000 LAV shares, and vaultUtilization[pair] = 1000 with _vaultWhitelistCbr[pair] = 1e27. The pair has borrowed funds and 100 assets of unaccrued interest. (1) The attacker calls pair.addCollateral(0, attacker). This reaches internal _addInterest, increasing pair totalAsset.amount and totalBorrow.amount; pair CBR becomes 1.1, but LAV metadata is unchanged. (2) In the same block, the attacker calls LAV.deposit(100, attacker). LAV calls pair.addInterest(false), but because the pair was already accrued at the current timestamp, interestEarned is zero and _updateAssetMetadataFromVault is skipped. The attacker mints 100 LAV shares at the stale 1.0 CBR; with fresh metadata they should receive only about 90.9 shares. (3) The attacker calls pair.repayAsset(0, attacker) or any path that reaches externalAssetVault.whitelistUpdate(true). LAV now updates _totalAssets from the pair CBR increase, so totalAssets becomes 1200 and totalSupply is 1100. The attacker’s 100 shares are worth about 109.09 assets, capturing about 9.09 assets of yield that accrued before their deposit, withdrawable immediately if the LAV has enough idle assets. Existing M-19 utilization manipulation remains: if LAV.deposit changes totalAvailableAssetsForVault after a skipped checkpoint, the next Fraxlend accrual charges the entire elapsed period at the post-deposit capacity.

**Mitigation**  
Before any LAV share mint or burn, refresh each whitelisted pair metadata whenever the current pair CBR differs from _vaultWhitelistCbr, regardless of whether addInterest(false) returns positive interest. Do not use the public threshold-gated addInterest path as a freshness oracle; expose or use a non-skipping pair checkpoint for LAV integrations, or have LAV compare IFraxlendPair.previewAddInterest/current convertToAssets directly and update metadata on any difference. Fraxlend should also avoid applying LAV capacity changes to elapsed time before that capacity existed, for example by force-checkpointing the pair before LAV deposits or withdrawals alter totalAvailableAssetsForVault.

### [M-20] LAV allocation cap ignores accrued utilization when whitelisted pairs draw liquidity

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/LendingAssetVault.sol:73; contracts/contracts/LendingAssetVault.sol:230; fraxlend/src/contracts/FraxlendPairCore.sol:601; fraxlend/src/contracts/FraxlendPairCore.sol:604; fraxlend/src/contracts/FraxlendPairCore.sol:654

**Summary/Description**  
LendingAssetVault limits whitelistWithdraw with vaultMaxAllocation - vaultDeposits, while the LAV's actual current exposure is tracked in vaultUtilization and can grow above deposits when the whitelisted pair CBR increases or when only principal is returned. A pair whose current utilization is already at or above its configured allocation can still draw remaining principal headroom through Fraxlend vault-pull paths, pushing more LAV assets into that pair and bypassing the intended per-pair allocation cap.

**Root Cause**  
totalAvailableAssetsForVault uses vaultDeposits as the allocation denominator even though _updateAssetMetadataFromVault updates vaultUtilization and _totalAssetsUtilized to the current value of the LAV-owned vault shares. Fraxlend's own over-allocation check uses vaultUtilization, so the draw-side check and the return-side check enforce different notions of allocation.

**Pre_conditions**  
A whitelisted Fraxlend pair has drawn less principal than vaultMaxAllocation, but its current LAV-owned share value has increased so vaultUtilization is at or above the configured allocation. The LAV still has local available assets, and a public Fraxlend borrowAsset, withdraw, or redeem flow needs external LAV liquidity through _depositFromVault().

**Impact**  
The pair can consume LAV liquidity beyond its configured current allocation, lowering totalAvailableAssets for LAV withdrawals and for other whitelisted pairs. If the over-allocated pair later incurs bad debt, more LAV value is exposed than the allocation limit was meant to allow.

**Proof of Concept**  
Example with max allocation 100: the pair draws 60 from LAV, then interest raises the LAV-owned pair shares to 100. whitelistWithdraw(40) first refreshes vaultUtilization to 100, but totalAvailableAssetsForVault still returns 40 because it subtracts vaultDeposits=60 from the cap. The call succeeds and records vaultUtilization=140, even though a utilization-based cap would allow 0.

**Mitigation**  
Use the current exposure for draw-side allocation checks, e.g. cap headroom should be max(0, vaultMaxAllocation[vault] - vaultUtilization[vault]) after metadata refresh. Keep vaultDeposits only for separate principal accounting if it is still needed.

### [M-21] Zero-rounded Fraxlend CBR can brick LAV metadata updates

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/LendingAssetVault.sol:268; contracts/contracts/LendingAssetVault.sol:293; contracts/contracts/LendingAssetVault.sol:294; fraxlend/src/contracts/FraxlendPair.sol:221; fraxlend/src/contracts/libraries/VaultAccount.sol:38; fraxlend/src/contracts/libraries/VaultAccount.sol:42; fraxlend/src/contracts/FraxlendPairCore.sol:1043; fraxlend/src/contracts/FraxlendPairCore.sol:1177; fraxlend/src/contracts/FraxlendPairCore.sol:1195

**Summary/Description**  
LendingAssetVault stores each whitelisted Fraxlend pair CBR as convertToAssets(1e27). Fraxlend convertToAssets rounds down as shares * totalAsset.amount / totalAsset.shares, so after a severe bad-debt writeoff this value can round to zero while the pair still has nonzero fToken supply. The LAV down-CBR branch then divides by the new CBR, causing a division-by-zero revert.

**Root Cause**  
The metadata update formula uses oldCbr / newCbr for CBR decreases and does not handle newCbr == 0. The preview path duplicates the same denominator. Fraxlend bad-debt liquidation can reduce totalAsset.amount without reducing totalAsset.shares, making convertToAssets(1e27) return zero when totalAsset.shares exceeds totalAsset.amount * 1e27.

**Pre_conditions**  
A whitelisted Fraxlend pair is connected to LAV as externalAssetVault, LAV has a nonzero stored _vaultWhitelistCbr for the pair, and a bad-debt event or other valid pair state transition drops the pair CBR below the 1e27 measurement granularity.

**Impact**  
The next metadata refresh for that pair reverts. This can revert the liquidation itself because Fraxlend _repayAsset calls externalAssetVault.whitelistUpdate(true) after the bad-debt writeoff, and it also blocks later pair repay/borrow paths that call whitelistUpdate, whitelistDeposit, or whitelistWithdraw. LAV preview/max functions also revert while the zero-rounded vault remains in the whitelist, and state-changing LAV paths can be blocked whenever their global update reaches the pair.

**Proof of Concept**  
Example with 1:1 starting accounting: totalAsset.amount = totalAsset.shares = 2e27 and LAV has stored oldCbr = 1e27. A clean liquidation writes off all but 1 wei of the pair asset claims, leaving totalAsset.amount = 1 and totalAsset.shares = 2e27. Fraxlend convertToAssets(1e27) returns floor(1e27 * 1 / 2e27) = 0. LAV _updateAssetMetadataFromVault then takes the decrease branch and evaluates (1e27 * oldCbr) / 0, reverting.

**Mitigation**  
Do not divide by the new CBR in the loss branch. Scale utilized assets directly by newCbr / oldCbr, or derive the utilized value from the LAV-owned pair share balance with convertToAssets and explicitly allow a zero result. Apply the same zero-safe logic in _previewAddInterestAndMdInAllVaults, and avoid making Fraxlend liquidation depend on a LAV callback that can revert during bad-debt accounting.

### [M-22] Fraxlend public interest skip can block LVF deleveraging

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/lvf/LeverageManager.sol:166; contracts/contracts/lvf/LeverageManager.sol:175; contracts/contracts/lvf/LeverageManager.sol:181; contracts/contracts/lvf/LeverageManager.sol:365; fraxlend/src/contracts/FraxlendPairCore.sol:286; fraxlend/src/contracts/FraxlendPairCore.sol:313; fraxlend/src/contracts/FraxlendPairCore.sol:319; fraxlend/src/contracts/FraxlendPairCore.sol:442; fraxlend/src/contracts/FraxlendPairCore.sol:1059

**Summary/Description**  
LeverageManager.removeLeverage tries to checkpoint the pair by calling IFraxlendPair.addInterest(false), then converts the requested flash-loan amount into borrow shares from the stored totalBorrow. Fraxlend's public addInterest can emit SkipAddingInterest and leave totalBorrow stale because its gate compares current utilization against a stale, differently-denominated _prevUtilizationRate cache rather than against the pair CBR or preview interest. The later callback calls repayAsset with those stale shares; repayAsset uses internal _addInterest, accrues the pending interest, and can compute a repayment amount larger than the flash-loaned amount/allowance, so the deleveraging transaction reverts.

**Root Cause**  
The LVF flow assumes public addInterest(false) is a guaranteed accrual checkpoint, but Fraxlend's public entrypoint can skip based on a stale and unit-mismatched utilization cache. _prevUtilizationRate is written inside internal _addInterest before the caller's later state change and uses borrow divided by available liquidity, while the public wrapper compares it to current borrow divided by total asset capacity. The flow then mixes stale pre-accrual share conversion with a later state-changing Fraxlend repayment that always uses the internal non-gated accrual path.

**Pre_conditions**  
A leveraged position has outstanding borrow shares and time has elapsed since the pair's last internal interest update so previewAddInterest would return positive interest. The public addInterest(false) utilization gate sees a sub-threshold delta. Under the default threshold this requires either very low actual utilization or a prior utilization-increasing action, such as borrow or withdraw, that leaves current utilization close to the stale cached _prevUtilizationRate from the previous pre-action state.

**Impact**  
Normal removeLeverage calls can fail before the flash loan is repaid, preventing users from reducing debt or closing positions through the intended LVF path. The failure is repeatable whenever pending interest exists and the public checkpoint skips. A knowledgeable user or keeper can work around it by first forcing an internal pair accrual through a separate pair action, but the LVF entrypoint itself does not preserve liveness and the failure can matter when users need to deleverage quickly.

**Proof of Concept**  
Sequence: (1) a pair sits at 25% utilization. (2) A borrow/withdraw path calls internal _addInterest before changing utilization, caching _prevUtilizationRate as 25%/(1-25%) = 33.33%, then raises current utilization to about 33.33%. (3) After time passes, removeLeverage calls pair.addInterest(false); the public gate compares current utilization to the stale cached value and skips, leaving totalBorrow stale. (4) removeLeverage encodes totalBorrow.toShares(_borrowAssetAmt, false) using stale totalBorrow and approves exactly _borrowAssetAmt to the pair. (5) In the flash callback, pair.repayAsset(_borrowSharesToRepay, custodian) calls internal _addInterest, accrues pending interest, and computes _amountToRepay from the now-larger totalBorrow. For any material repayment this amount can exceed the approved/borrowed _borrowAssetAmt, so safeTransferFrom reverts and the unwind fails.

**Mitigation**  
Do not use the public, threshold-gated addInterest path as a required checkpoint before LVF share math. Add a Fraxlend function that force-accrues for trusted protocol integrations, or compute repay shares from previewAddInterest and approve/fund the matching preview repayment amount. Alternatively have LeverageManager perform a direct nonzero/zero-safe pair action that reaches internal _addInterest before converting borrow shares.

### [M-23] Low or lagged oracle prices can disable Fraxlend liquidations

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/oracle/spTKNMinimalOracle.sol:120; contracts/contracts/oracle/spTKNMinimalOracle.sol:126; contracts/contracts/oracle/spTKNMinimalOracle.sol:140; contracts/contracts/oracle/aspTKNMinimalOracle.sol:24; contracts/contracts/oracle/aspTKNMinimalOracle.sol:26; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:141; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:145; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:148; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:149; fraxlend/src/contracts/FraxlendPairCore.sol:535; fraxlend/src/contracts/FraxlendPairCore.sol:552; fraxlend/src/contracts/FraxlendPairCore.sol:555; fraxlend/src/contracts/FraxlendPairCore.sol:1117; fraxlend/src/contracts/FraxlendPairCore.sol:1120; fraxlend/src/contracts/FraxlendPairCore.sol:225; fraxlend/src/contracts/FraxlendPairCore.sol:232

**Summary/Description**  
Fraxlend liquidations use the low oracle price returned by _updateExchangeRate and discard the borrow-allowed/deviation result. Peapods spTKN/aspTKN oracles can return a UniV3-derived primary price that is materially below a configured Chainlink secondary price when the AMM/TWAP leg is lagged or manipulated; DualOracleChainlinkUniV3 has the same shape when its Mean Finance UniV3 TWAP quote is below Chainlink. _updateExchangeRate can detect excessive low/high deviation and return _isBorrowAllowed = false, but liquidate still checks _isSolvent with the low price. A borrower that is unhealthy at the healthy/high price can therefore appear solvent and liquidation reverts while the low-price condition persists.

**Root Cause**  
FraxlendPairCore.liquidate ignores _updateExchangeRate return value _isBorrowAllowed and uses priceLow directly for the liquidation solvency gate. spTKNMinimalOracle and DualOracleChainlinkUniV3 both sort the lower of their AMM/TWAP and Chainlink-derived legs into priceLow, while liquidation-side code does not reject large low/high deviation before using that conservative low value.

**Pre_conditions**  
A Fraxlend pair uses an aspTKN/spTKN oracle with a Chainlink comparison leg configured, or it uses DualOracleChainlinkUniV3 with a healthy Chainlink leg and a UniV3 pool/TWAP that can lag or be moved downward relative to Chainlink. The borrower is liquidatable at the valid/high exchange rate and has nonzero recorded collateral, but is solvent under the low lagged or manipulated price.

**Impact**  
Liquidations revert with BorrowerSolvent while the low-price condition persists. New borrows and collateral removals are blocked when deviation is excessive, but existing unhealthy debt cannot be closed, so collateral value can continue falling and leave lenders or LAV depositors with avoidable bad debt. The issue affects a core liquidation path and can recur for any pair using an affected oracle configuration.

**Proof of Concept**  
Code path: spTKNMinimalOracle.getPrices computes a primary price through the UniV3/default path and, when Chainlink comparison feeds are configured, a secondary price through _chainlinkBasePerPaired18(); it returns the lower value as priceLow and the higher value as priceHigh. Separately, DualOracleChainlinkUniV3.getPrices reads a single Mean Finance UniV3 static-oracle quote and a Chainlink quote, then returns the lower value as priceLow. StaticOracle.quoteSpecificPoolsWithTimePeriod uses an arithmetic mean tick over TWAP_DURATION before converting that tick into a quote. If the AMM/TWAP leg is below Chainlink because it lags a fast price move or because the pool TWAP is manipulated down, low = AMM/TWAP and high = Chainlink; _updateExchangeRate computes a high deviation but liquidate discards the false borrow-allowed result and checks borrower solvency using the low value. _isSolvent computes borrow * low / collateral, so the borrower can appear solvent even when insolvent at high.

**Mitigation**  
Make liquidation reject bad-data and excessive-deviation oracle states instead of using priceLow unconditionally. At minimum, check _isBorrowAllowed in liquidate before the solvency gate and reject zero exchange rates. For DualOracleChainlinkUniV3, consider failing closed or using a more robust source when the UniV3 and Chainlink legs diverge materially, including divergence caused by TWAP mean lag after sharp price moves. Enforce deployment-time minimum TWAP/liquidity/pool-authentication requirements for any UniV3 leg used in lending.

### [M-24] Close fee can be avoided on pTKN value routed through repayment

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/lvf/LeverageManager.sol:223; contracts/contracts/lvf/LeverageManager.sol:388; contracts/contracts/lvf/LeverageManager.sol:399; contracts/contracts/lvf/LeverageManager.sol:400; contracts/contracts/lvf/LeverageManager.sol:465

**Summary/Description**  
LeverageManager applies closeFeePerc only after _removeLeveragePostCallback returns the remaining pTKN amount. During the remove flow, if the paired token received from the unstaked LP is lower than the flash repayment amount, the contract sells unlocked pTKN for borrow tokens before returning from _removeLeveragePostCallback. The pTKN value consumed by that repayment is never included in the close-fee base, and any paired/borrow-token remainder is sent to the position owner without a close fee.

**Root Cause**  
The close fee is charged late and only on _ptknToUserAmt, after _acquireBorrowTokenForRepayment may have converted part of the unlocked pTKN into borrow tokens. The fee logic ignores pTKN value used to repay the flash loan/debt and also ignores fee-free borrow-token proceeds returned through _pairedLpToUser.

**Pre_conditions**  
closeFeePerc is nonzero. A position owner can choose removeLeverage parameters where the debt repayment amount exceeds the paired-token amount obtained from the collateral slice, while the removed collateral's pTKN side can be swapped into the missing borrow token amount. This is reachable during ordinary partial deleveraging or by splitting a full close into more than one removeLeverage call.

**Impact**  
Protocol close-fee revenue can be reduced for normal position exits. The saved amount is the configured close fee on the pTKN value internally routed through repayment instead of returned as pTKN. The route is repeatable by any position owner and is economically attractive whenever the avoided close fee exceeds the swap/price-impact cost of converting the pTKN side to the borrow token.

**Proof of Concept**  
Ignoring flash and swap fees for clarity, consider a 2x position whose collateral can unwind to 100 pTKN and 100 borrow tokens, with 100 borrow-token debt. A direct full close repays 100, returns 100 pTKN, and charges closeFeePerc on all 100 pTKN. Instead, the owner first repays the full 100 debt while removing only half the collateral. That slice returns about 50 pTKN and 50 borrow tokens, so the remove flow sells the 50 pTKN for the missing 50 borrow tokens before the close-fee calculation; _ptknToUserAmt is then zero and no close fee is charged on that half. With debt cleared, the owner removes the remaining half and pays close fee only on the remaining 50 pTKN, while receiving the paired-token side fee-free.

**Mitigation**  
Compute the close fee from the gross pTKN amount unlocked before any repayment swap, or charge the close fee before _acquireBorrowTokenForRepayment can consume pTKN. If the intended fee is on total close proceeds, also apply the fee to borrow/paired-token proceeds using an oracle or deterministic LP ratio, or require proportional debt repayment and collateral removal so pTKN value cannot be routed around the fee.

### [M-25] Exact-output repayment swap can spend all returned pTKN despite the slippage parameter

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/lvf/LeverageManager.sol:414; contracts/contracts/lvf/LeverageManager.sol:455; contracts/contracts/dex/UniswapDexAdapter.sol:84; contracts/contracts/dex/CamelotDexAdapter.sol:55

**Summary/Description**  
During removeLeverage, LeverageManager sells returned pTKN when the paired token from LP removal is not enough to repay the flash loan. The public parameter _podSwapAmtOutMin is documented as a slippage/min-output value, but _swapPodForBorrowToken passes all returned pTKN as amountInMax and uses _podSwapAmtOutMin, or the repayment shortfall when it is zero, as the exact amountOut for swapV2SingleExactOut. For exact-output swaps the slippage bound is amountInMax, so the user parameter does not cap how much pTKN can be consumed.

**Root Cause**  
LeverageManager mixes exact-output routing with a min-output-style parameter and never exposes or checks a caller-selected maximum pTKN input or minimum pTKN remaining after the repayment swap. The _podAmtMin check only protects the LP removal leg before the swap consumes the received pTKN.

**Pre_conditions**  
A user removes leverage and the paired token received from unstaking/removing LP, plus any user-provided borrow token, is less than the flash repayment amount. The pod uses a Uniswap/Camelot-style exact-output adapter path, and the pTKN/borrow-token pool price can be moved while the required output is still obtainable with the full pTKN amount received from LP removal.

**Impact**  
A sandwich can make the exact-output repayment swap spend far more pTKN than the user would reasonably tolerate, up to all pTKN returned by the LP removal leg, while the transaction still succeeds and repays the flash loan. The lost pTKN value is captured through the manipulated AMM price, and the user receives little or no pTKN from the unwind despite supplying the available slippage parameters.

**Proof of Concept**  
Code path: removeLeverage encodes _podSwapAmtOutMin, _removeLeveragePostCallback computes repayAmount, and when _pairedAmtReceived < repayAmount it calls _acquireBorrowTokenForRepayment. That function calls _swapPodForBorrowToken with _podAmtReceived as _podAmt. _swapPodForBorrowToken then calls swapV2SingleExactOut(_pod, _targetToken, _podAmt, _podSwapAmtOutMin == 0 ? _targetNeededAmt : _podSwapAmtOutMin, address(this)). Example: LP removal returns 100 pTKN and 90 borrow tokens, while the flash repayment is 100 borrow tokens. With _podSwapAmtOutMin = 0, the exact-output target is 10 borrow tokens and amountInMax is 100 pTKN. If the pool is moved so 10 borrow tokens costs 90 pTKN, the router consumes 90 pTKN and the unwind succeeds; only the remaining 10 pTKN is returned to the user. There is no check for the expected pTKN spend or minimum pTKN remaining.

**Mitigation**  
Keep the exact-output amount fixed to the repayment shortfall, and add a caller-provided maximum pTKN input or minimum pTKN remaining check for the repayment swap. Do not use a min-output-style parameter as the exact output target. Also return or account for any excess borrow token acquired during the swap path.

### [M-26] removeLeverage overfill can be stranded and swept by later leverage calls

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/lvf/LeverageManager.sol:339; contracts/contracts/lvf/LeverageManager.sol:341; contracts/contracts/lvf/LeverageManager.sol:386; contracts/contracts/lvf/LeverageManager.sol:388; contracts/contracts/lvf/LeverageManager.sol:399; contracts/contracts/lvf/LeverageManager.sol:400; contracts/contracts/lvf/LeverageManager.sol:414; contracts/contracts/lvf/LeverageManager.sol:455

**Summary/Description**  
During removeLeverage, LeverageManager can end the flash callback with more borrow token than it returns to the exiting position owner. This happens when the user-specified _borrowAssetAmt floor-converts to borrow shares and Fraxlend pulls less than the flashed amount, when wallet top-up or a repayment swap covers an overstated shortfall, or when the helper acquires more borrow token than the actual flash-repayment deficit. The refund is computed from stale _pairedAmtReceived accounting, so the surplus remains in LeverageManager. A later addLeverage for the same borrow token refunds the full manager-wide post-repayment balance to that later position owner, allowing unrelated capture of the stranded overfill.

**Root Cause**  
The remove path does not snapshot or settle the borrow-token balance delta for the whole callback. It converts _borrowAssetAmt to shares with floor rounding, ignores the actual amount consumed by repayAsset, ignores borrow tokens pulled or acquired inside _acquireBorrowTokenForRepayment when computing _borrowAmtRemaining, and derives the user refund from stale _pairedAmtReceived. Separately, addLeverage assumes the manager has no pre-existing borrow-token balance and transfers IERC20(_d.token).balanceOf(address(this)) to the current position owner instead of only call-local surplus.

**Pre_conditions**  
A removeLeverage call completes with borrow tokens left after flash repayment. This can be caused by _borrowAssetAmt being larger than the rounded-up amount actually consumed by repayAsset for the computed shares, by _userProvidedDebtAmtMax filling a shortfall that was overstated because unspent flashed tokens or stale balances were ignored, or by a repayment swap/redeem acquiring more borrow token than the actual shortfall. Later, any user completes addLeverage using the same borrow token while the leftover remains in LeverageManager.

**Impact**  
The exiting position owner can lose excess borrow tokens from a successful deleveraging transaction. The value is bounded by the overfilled repayment/top-up/swap amount for that call, but it is user value from a core unwind path and can be captured by a later unrelated leverage user because addLeverage sweeps the manager-wide borrow-token balance. If no later workflow captures it, only the owner-only rescueTokens path can move the funds.

**Proof of Concept**  
Example A: removeLeverage flashes 101 borrow tokens, but IFraxlendPair(totalBorrow).toShares(101, false) maps to shares whose repayAsset amount is 100. After LP removal returns enough paired token, the manager repays the 101-token flash but returns only _pairedAmtReceived - 101 to the user, leaving the 1 unspent flash token in the manager. Example B: the flash repayment is 100, LP removal returns 90, and _userProvidedDebtAmtMax or _podSwapAmtOutMin causes the helper to add 20 borrow tokens. The manager transfers 100 to the flash source, but line 400 still computes the refund from the original 90 and returns 0 instead of the actual 10-token surplus. A later addLeverage with the same borrow token reaches line 339 after its own flash repayment and line 341 transfers the stranded balance to that later position owner.

**Mitigation**  
Snapshot the borrow-token balance at the start of each add/remove callback and compute refunds from call-local deltas. Use repayAsset's returned amount, plus the flash fee, to compute the true repayment deficit before pulling wallet top-up or swapping pTKN. Make _acquireBorrowTokenForRepayment return the amount acquired and include it in refund accounting. In addLeverage, refund only surplus generated by the current call and exclude pre-existing manager balances.

### [M-27] Self-lending oracle applies fToken CBR in the wrong direction

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/contracts/oracle/spTKNMinimalOracle.sol:145; contracts/contracts/oracle/spTKNMinimalOracle.sol:166; fraxlend/src/contracts/FraxlendPair.sol:221; fraxlend/src/contracts/FraxlendPairCore.sol:225; contracts/README.md:91

**Summary/Description**  
For self-lending pods, the pod LP is paired against the Fraxlend fToken, while the outer Fraxlend market needs an exchange rate in underlying borrow-token units. spTKNMinimalOracle first values the LP as though the paired fToken reserve were already denominated in the underlying asset, then calls convertToAssets() on the resulting collateral-per-fToken price. Since convertToAssets() multiplies by the fToken asset-per-share CBR, the oracle increases the collateral-per-asset exchange rate when the fToken becomes more valuable, even though the correct conversion should require fewer fTokens, and therefore fewer sp/aspTKN, per underlying asset.

**Root Cause**  
The BASE_IS_FRAX_PAIR branch treats a spTKN-per-fToken exchange rate as an fToken share amount and passes it to IFraxlendPair(BASE_TOKEN).convertToAssets(). The LP math needs the pTKN price converted from underlying-asset units into fToken-share units before applying the V2 invariant, or the final collateral-per-fToken result divided by the Fraxlend asset-per-share rate. The current code multiplies by that rate instead.

**Pre_conditions**  
A documented self-lending setup is used where the pod paired asset is the Fraxlend lending pair token and the sp/aspTKN oracle is configured with BASE_IS_FRAX_PAIR. The Fraxlend fToken CBR differs from 1 because interest accrued or bad debt changed totalAsset/share accounting.

**Impact**  
Borrow and liquidation solvency checks consume the returned exchange rate as collateral tokens required per 1e18 borrow asset. When the fToken CBR increases under normal interest accrual, the oracle overstates the exchange rate and can make otherwise healthy self-lending positions appear liquidatable, allowing collateral to be seized too early. If the fToken CBR decreases after bad debt, the same inverted conversion understates the exchange rate and can keep undercollateralized positions from being liquidated, increasing bad-debt risk.

**Proof of Concept**  
Let one fToken share be worth E underlying assets. Correct LP valuation for a pTKN/fToken pool should use the pTKN price in fToken units, i.e. priceUnderlyingPerPTKN / E, so the resulting collateral-per-underlying exchange rate decreases as E increases. The implementation instead computes the LP price using priceUnderlyingPerPTKN while the reserve is still in fToken shares, then executes convertToAssets(_spTknBasePrice18), multiplying by E. For E > 1 the returned exchange rate moves higher rather than lower, and for E < 1 it moves lower rather than higher. FraxlendPairCore._isSolvent() directly multiplies borrower debt by this exchange rate.

**Mitigation**  
For BASE_IS_FRAX_PAIR, convert the pTKN/base price into fToken-share units before the V2 LP invariant by using convertToShares() on the underlying-denominated price, or divide the final spTKN-per-fToken result by the fToken asset-per-share rate. Add self-lending oracle tests with fToken CBR above and below 1 and assert the returned collateral-per-underlying rate moves in the correct direction.

### [M-28] spTKN oracle double-counts pod debond fees in LP collateral pricing

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/oracle/spTKNMinimalOracle.sol:179; contracts/contracts/oracle/spTKNMinimalOracle.sol:185; contracts/contracts/oracle/spTKNMinimalOracle.sol:231; contracts/contracts/oracle/spTKNMinimalOracle.sol:246; contracts/contracts/WeightedIndex.sol:120; contracts/contracts/WeightedIndex.sol:128; fraxlend/src/contracts/FraxlendPairCore.sol:225; fraxlend/src/contracts/FraxlendPairCore.sol:1117

**Summary/Description**  
spTKNMinimalOracle prices pod LP collateral by first calling _accountForCBRInPrice(), which uses IDecentralizedIndex.convertToAssets() for the pod. WeightedIndex.convertToAssets() already subtracts the pod debond fee from the returned underlying amount. The oracle then calls _accountForUnwrapFeeInPrice() and subtracts the same DEBOND_FEE again before feeding the pTKN price into the V2 reserve-product LP valuation.

**Root Cause**  
The oracle assumes convertToAssets() provides a fee-free pod CBR and explicitly applies the unwrap fee afterward, but the scoped WeightedIndex implementation already returns a debond-fee-discounted asset amount. This makes the reserve math use price * (1 - debondFee)^2 instead of the single-fee liquidation value.

**Pre_conditions**  
A pod with a nonzero debond fee is used as the pTKN side of the spTKN/aspTKN oracle-backed LP collateral. A borrower is near the Fraxlend liquidation threshold or borrow limit where the difference between one fee and two fees changes solvency.

**Impact**  
The LP reserve formula undervalues the pTKN side of the pool and therefore overstates the collateral-per-borrow-asset exchange rate returned to Fraxlend. Borrowers can be denied otherwise valid borrows or liquidated before their collateral is actually below the configured LTV when accounting for the single real unwrap fee. The value at risk is the liquidation penalty and seized collateral for positions that fall between the correct single-fee threshold and the implemented double-fee threshold.

**Proof of Concept**  
For a 10% pod debond fee, WeightedIndex.convertToAssets(1e18) returns 0.9e18 worth of underlying for one pTKN. spTKNMinimalOracle then applies _accountForUnwrapFeeInPrice() and passes 0.81e18 into the LP invariant. Correct liquidation pricing should use one 10% haircut, not 19%. Because _basePerSpTkn18 is proportional to sqrt(priceBasePerPTkn), the oracle reports roughly 1/sqrt(0.81/0.9) = 1.054x more collateral required per borrow asset than the single-fee value, which FraxlendPairCore._isSolvent() uses directly for borrows and liquidations.

**Mitigation**  
Make the fee treatment single-sourced. Either change the pod CBR helper used by the oracle to read a fee-free assets-per-share value before applying DEBOND_FEE once, or stop calling _accountForUnwrapFeeInPrice() when convertToAssets() already returns a debond-fee-adjusted amount. Add oracle tests with nonzero pod debond fees that compare against a one-fee LP unwind value.

### [M-29] BASE_IS_POD CBR adjustment overstates spTKN exchange rates

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/contracts/oracle/spTKNMinimalOracle.sol:145; contracts/contracts/oracle/spTKNMinimalOracle.sol:163; contracts/contracts/oracle/spTKNMinimalOracle.sol:213; contracts/contracts/oracle/spTKNMinimalOracle.sol:231; contracts/contracts/WeightedIndex.sol:120; fraxlend/src/contracts/FraxlendPairCore.sol:225; fraxlend/src/contracts/FraxlendPairCore.sol:1117

**Summary/Description**  
When BASE_TOKEN is itself a pod, spTKNMinimalOracle prices the collateral LP against the base pod underlying token and then applies the base pod CBR after the LP invariant has been inverted into spTKN-per-base units. For a pTKN/base-pod LP, the V2 reserve formula needs the pTKN price denominated in base-pod shares before computing LP value. That requires dividing the underlying-denominated pTKN price by the base pod CBR before the square-root LP valuation, not multiplying the final collateral-per-base exchange rate by the base pod CBR afterward.

**Root Cause**  
The BASE_IS_POD branch handles base pod CBR at the wrong stage and in the wrong unit direction. _calculateBasePerPTkn() returns base-underlying per collateral pTKN, _calculateSpTknPerBase() uses that value with reserves where the base reserve is base pod shares, and _checkAndHandleBaseTokenPodConfig() later calls _accountForCBRInPrice(BASE_TOKEN, ...) on an already inverted spTKN-per-base value. This makes the returned exchange rate scale by the base pod CBR after inversion instead of using base-pod-share units inside the LP pricing formula.

**Pre_conditions**  
A Fraxlend LVF market uses an aspTKN/spTKN oracle with BASE_IS_POD enabled, such as a pod whose paired LP token is another pod. The base pod CBR differs from 1 due normal pod fee/burn mechanics or any other supply/backing change. A borrower is near a borrow or liquidation threshold where the CBR error changes solvency.

**Impact**  
Fraxlend consumes the oracle output as collateral tokens required per 1e18 borrowed asset. If the borrowed base pod CBR is above 1, the implementation reports an exchange rate higher than the economically correct pTKN/base-pod-share LP value, making positions look less solvent than they are and enabling premature liquidations or blocking otherwise valid borrows. If the base pod backing were to move below 1, the same unit error would move the exchange rate in the unsafe direction and could delay liquidations. A caller can also time liquidation around base pod CBR changes, because the oracle overreacts to that CBR.

**Proof of Concept**  
Let the collateral LP hold pA/basePod shares, the collateral pod CBR be 1, the external underlying price be 1 base-underlying per pA-underlying, and the base pod CBR be cB = 4 base-underlying per basePod share. The correct pA price for the LP invariant is 1 / cB = 0.25 basePod shares per pA, so basePerSpTKN is proportional to sqrt(0.25) and spTKNPerBasePod is proportional to 2. The implementation uses price 1 in the invariant, so the inverted value is proportional to 1, then _checkAndHandleBaseTokenPodConfig multiplies by cB and returns a value proportional to 4. The reported collateral-per-base exchange rate is therefore 2x too high, i.e. overstated by sqrt(cB), and FraxlendPairCore._isSolvent() and liquidate() use that value directly.

**Mitigation**  
For BASE_IS_POD, convert the pTKN price into base-pod-share units before the V2 LP invariant, for example by dividing the base-underlying-per-pTKN value by the base pod assets-per-share CBR or by using a fee-aware convertToShares-style helper on the base pod. Do not apply base pod CBR to the final spTKN-per-base exchange rate after inversion. Add oracle tests where the base pod CBR is above and below 1 and assert the returned Fraxlend exchange rate matches the LP value in base-pod-share units.

### [M-30] Missing or stale Chainlink legs collapse Fraxlend oracles to a single manipulable price

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/oracle/spTKNMinimalOracle.sol:83; contracts/contracts/oracle/spTKNMinimalOracle.sol:88; contracts/contracts/oracle/spTKNMinimalOracle.sol:120; contracts/contracts/oracle/spTKNMinimalOracle.sol:125; contracts/contracts/oracle/spTKNMinimalOracle.sol:126; contracts/contracts/oracle/spTKNMinimalOracle.sol:127; contracts/contracts/oracle/spTKNMinimalOracle.sol:128; contracts/contracts/oracle/spTKNMinimalOracle.sol:130; contracts/contracts/oracle/spTKNMinimalOracle.sol:145; contracts/contracts/oracle/spTKNMinimalOracle.sol:172; contracts/contracts/oracle/spTKNMinimalOracle.sol:174; contracts/script/DeployAspTknMinimalOracle.s.sol:27; contracts/script/DeployAspTknMinimalOracle.s.sol:39; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:107; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:111; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:119; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:122; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:145; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:148; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:149; fraxlend/src/contracts/FraxlendPairCore.sol:535; fraxlend/src/contracts/FraxlendPairCore.sol:538; fraxlend/src/contracts/FraxlendPairCore.sol:552; fraxlend/src/contracts/FraxlendPairCore.sol:555; fraxlend/src/contracts/FraxlendPairCore.sol:918; fraxlend/src/contracts/FraxlendPairCore.sol:1003; fraxlend/src/contracts/FraxlendPairCore.sol:1117

**Summary/Description**  
Fraxlend relies on getPrices returning independent low/high values, but the affected oracle compositions can return the same UniV3-derived value for both sides. In spTKN/aspTKN, Chainlink comparison feeds are optional; if they are omitted, getPrices leaves priceTwo equal to the default UniV3/V2-reserve path. The provided DeployAspTknMinimalOracle script builds the oracle with all optional comparison feeds set to address(0), so this single-source mode is reachable through the repository deployment flow. If Chainlink comparison feeds are configured but stale, ChainlinkSinglePriceOracle reports isBadData and spTKNMinimalOracle maps that to a zero sentinel that re-enters the default UniV3 path. In DualOracleChainlinkUniV3, stale Chainlink data explicitly returns the UniV3 TWAP as both priceLow and priceHigh. FraxlendPairCore only emits WarnOracleData for the bad-data flag and gates borrowing/collateral removal on low/high deviation, so equal low/high single-source prices pass the deviation check.

**Root Cause**  
The oracle compositions do not require an independent comparison source before feeding Fraxlend lending decisions. spTKNMinimalOracle permits absent Chainlink comparison feeds and initializes the secondary price to the primary value; when a configured Chainlink leg is bad, it uses 0 as both a bad-secondary marker and a request to fetch the default UniV3 price. DualOracleChainlinkUniV3 sets low/high to the UniV3 value whenever _isBadData is true. FraxlendPairCore does not include the oracle bad-data flag or source-count semantics in borrow, collateral-removal, or liquidation gates.

**Pre_conditions**  
A Fraxlend pair uses an spTKN/aspTKN oracle with Chainlink comparison feeds omitted, as shown by DeployAspTknMinimalOracle, or uses an spTKN/aspTKN or DualOracleChainlinkUniV3 oracle with a configured Chainlink comparison leg that is stale or otherwise marked bad while the UniV3 path still returns a nonzero value. The remaining UniV3-derived price path can be moved or can lag enough to make the stored exchange rate economically unsafe.

**Impact**  
The intended low/high deviation check no longer constrains lending-market pricing. During omitted-comparison configurations or stale-feed windows, an attacker only needs to influence the remaining UniV3-derived path and can still pass maxOracleDeviation because low and high are equal. This can enable over-borrowing, bad debt, blocked or incorrect liquidations, and unsafe collateral removal decisions for any affected Fraxlend market.

**Proof of Concept**  
Code paths: spTKNMinimalOracle.getPrices sets priceTwo18 = priceOne18 before checking Chainlink feeds. If CHAINLINK_BASE_PRICE_FEED and CHAINLINK_QUOTE_PRICE_FEED are zero, the Chainlink branch is skipped and the function returns priceLow = priceHigh = the default UniV3/V2-reserve valuation. DeployAspTknMinimalOracle constructs aspTKNMinimalOracle with abi.encode(address(0), address(0), address(0), address(0), address(0), address(_v2Res)) for the optional immutables, so the comparison feeds are omitted. If feeds are configured but stale, ChainlinkSinglePriceOracle.getPriceUSD18 returns isBadData=true; spTKNMinimalOracle._chainlinkBasePerPaired18 maps that to 0, and _calculateSpTknPerBase(0) fetches the default UniV3 path again. Separately, DualOracleChainlinkUniV3._getChainlinkPrice returns _isBadData=true when block.timestamp - updatedAt > maxOracleDelay for either Chainlink leg; getPrices then sets priceLow = priceHigh = price1, the UniV3 static-oracle quote. FraxlendPairCore._updateExchangeRate emits WarnOracleData but computes deviation from equal low/high prices and allows borrow/collateral-removal paths when deviation is within the configured limit; liquidation uses the resulting low exchange rate.

**Mitigation**  
Require an independent comparison source for any oracle used by Fraxlend solvency checks, or explicitly mark single-source configurations as unusable for borrowing and liquidation-sensitive markets. For spTKN/aspTKN, validate constructor immutables so both Chainlink comparison feeds are present when the oracle will back a Fraxlend pair, and return a separate validity flag instead of overloading 0 as both bad-secondary and default-price sentinel. For DualOracleChainlinkUniV3, do not set both low and high to the UniV3 value when Chainlink is stale; revert, return values that fail the deviation gate, or make Fraxlend reject bad-data oracle reads.

### [M-31] Chainlink boundary answers are accepted as healthy oracle prices

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:79; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:112; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:114; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:115; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:117; contracts/contracts/oracle/spTKNMinimalOracle.sol:126; contracts/contracts/oracle/spTKNMinimalOracle.sol:127; contracts/contracts/oracle/spTKNMinimalOracle.sol:128; contracts/contracts/oracle/spTKNMinimalOracle.sol:130; contracts/contracts/oracle/spTKNMinimalOracle.sol:218; contracts/contracts/oracle/spTKNMinimalOracle.sol:219; contracts/contracts/oracle/spTKNMinimalOracle.sol:221; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:107; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:111; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:119; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:122; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:145; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:148; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:149; fraxlend/src/contracts/FraxlendPairCore.sol:535; fraxlend/src/contracts/FraxlendPairCore.sol:538; fraxlend/src/contracts/FraxlendPairCore.sol:552; fraxlend/src/contracts/FraxlendPairCore.sol:555; fraxlend/src/contracts/FraxlendPairCore.sol:918; fraxlend/src/contracts/FraxlendPairCore.sol:1117

**Summary/Description**  
ChainlinkSinglePriceOracle fetches the underlying aggregator minAnswer and maxAnswer but treats answers equal to either bound as valid because _isValidAnswer only rejects values strictly outside the range. Fraxlend's DualOracleChainlinkUniV3 has the same fail-open class more directly: it reads Chainlink multiply/divide feeds and only rejects answers that are nonpositive or stale, with no minAnswer/maxAnswer boundary check at all. When a configured Chainlink feed reaches a min or max boundary, the affected oracle path returns isBadData=false and a nonzero price. spTKN/aspTKN can use that boundary value as the Chainlink comparison leg, and DualOracleChainlinkUniV3 can use it as the Chainlink leg paired with a UniV3 TWAP. Fraxlend then only checks low/high deviation and does not know the Chainlink leg is at a source failure boundary.

**Root Cause**  
Chainlink boundary conditions are not treated as bad data. ChainlinkSinglePriceOracle uses _answer > _max || _answer < _min, so equality to a meaningful bound is accepted. DualOracleChainlinkUniV3 does not inspect the underlying aggregator bounds at all and only checks _answer <= 0 or stale updatedAt. Both paths can therefore feed bounded Chainlink answers into Fraxlend as healthy prices.

**Pre_conditions**  
A Fraxlend market uses an spTKN/aspTKN oracle with Chainlink comparison feeds configured, or uses DualOracleChainlinkUniV3 with a Chainlink multiply or divide feed configured. One configured Chainlink feed returns an answer exactly equal to a meaningful minAnswer or maxAnswer, or otherwise reaches a documented bound condition. The other price path remains available, or can be moved close enough to the bounded Chainlink value to pass maxOracleDeviation.

**Impact**  
A boundary Chainlink value can be stored as a valid exchange-rate input. If it diverges from the true market price, Fraxlend can block borrowing due to persistent oracle deviation, use the lower wrong rate for liquidation decisions, or allow unsafe borrowing when the remaining price path is moved near the bounded Chainlink value. The strongest realistic outcome is bad debt or incorrect liquidation behavior in affected lending markets.

**Proof of Concept**  
Code path: ChainlinkSinglePriceOracle._getChainlinkPriceFeedPrice18 reads latestRoundData and sets _isBadAnswer = _price <= 0 || !_isValidAnswer(_priceFeed, _price). _isValidAnswer reads minAnswer and maxAnswer from IEACAggregatorProxy(_feed).aggregator(), but returns true for _answer == _min and _answer == _max. spTKNMinimalOracle._chainlinkBasePerPaired18 only returns 0 when getPriceUSD18 reports isBadData=true, so a boundary answer becomes a nonzero Chainlink price. Separately, DualOracleChainlinkUniV3._getChainlinkPrice reads latestRoundData for CHAINLINK_MULTIPLY_ADDRESS and CHAINLINK_DIVIDE_ADDRESS, rejects only _answer <= 0 or stale updatedAt, and then getPrices compares that Chainlink value with the UniV3 static-oracle quote. In both cases, FraxlendPairCore._updateExchangeRate emits WarnOracleData only when the oracle flag is true and otherwise gates borrow by low/high deviation.

**Mitigation**  
Treat meaningful Chainlink boundary answers as bad data in every Chainlink-consuming oracle. For ChainlinkSinglePriceOracle, reject _answer <= _min or _answer >= _max when the bounds are meaningful. For DualOracleChainlinkUniV3, add equivalent minAnswer/maxAnswer checks or route Chainlink reads through a shared validated helper. Carry the bad-data flag into Fraxlend borrow and liquidation gating so source-failure states cannot be used as healthy prices.

### [M-32] DIA, UniV3, and dual-oracle paths bypass the L2 sequencer guard

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/oracle/DIAOracleV2SinglePriceOracle.sol:11; contracts/contracts/oracle/DIAOracleV2SinglePriceOracle.sol:18; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:13; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:20; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:25; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:35; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:51; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:84; contracts/contracts/oracle/spTKNMinimalOracle.sol:188; contracts/contracts/oracle/spTKNMinimalOracle.sol:196; contracts/contracts/oracle/spTKNMinimalOracle.sol:203; contracts/script/DeployAllSinglePriceOracles.s.sol:15; contracts/script/DeployAllSinglePriceOracles.s.sol:19; contracts/script/DeployAllSinglePriceOracles.s.sol:20; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:60; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:107; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:119; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:141; fraxlend/src/contracts/FraxlendPairCore.sol:535; fraxlend/src/contracts/FraxlendPairCore.sol:918; fraxlend/src/contracts/FraxlendPairCore.sol:1117

**Summary/Description**  
The shared single-price oracle deployment passes a SEQUENCER feed into ChainlinkSinglePriceOracle, UniswapV3SinglePriceOracle, and DIAOracleV2SinglePriceOracle. The base Chainlink getPriceUSD18 calls _sequencerCheck(), but both the UniV3 and DIA overrides skip that inherited guard. Separately, DualOracleChainlinkUniV3 has no sequencer-feed parameter or check at all while reading Chainlink feeds directly and combining them with a UniV3 static-oracle quote. On L2 deployments, these helpers can therefore return Chainlink, UniV3 TWAP, DIA, or optional base-conversion prices during the sequencer-down or post-recovery grace window that the configured feed is meant to block. Fraxlend can store the resulting stale prices for borrow and liquidation decisions.

**Root Cause**  
The L2 sequencer guard is implemented only in ChainlinkSinglePriceOracle.getPriceUSD18 rather than being enforced by every oracle path that can feed Fraxlend pricing. _getChainlinkPriceFeedPrice18 only validates feed answers and timestamps; it does not run the sequencer guard. The UniV3 and DIA implementations inherit the sequencer feed storage and are deployed with the feed, but their overridden getPriceUSD18 functions bypass _sequencerCheck(). DualOracleChainlinkUniV3 reads AggregatorV3Interface.latestRoundData directly and has no sequencer guard in its constructor or Chainlink leg.

**Pre_conditions**  
The protocol is deployed on an L2 that uses a sequencer uptime feed, such as Arbitrum, Base, or Mode. A Fraxlend market uses spTKNMinimalOracle/aspTKNMinimalOracle with UniswapV3SinglePriceOracle as the default or base-conversion source, DIAOracleV2SinglePriceOracle as a DIA base-conversion source, or DualOracleChainlinkUniV3 as its pair oracle. The sequencer has recently been down or recovered, and the AMM, DIA, or Chainlink-derived price is stale or can be moved before normal arbitrage restores pricing; for DIA, the stale timestamp can still be within staleAfterLastRefresh.

**Impact**  
Borrowers or liquidators can update Fraxlend exchange rates from oracle values that should have been rejected during the L2 recovery window. If the stale or manipulated price overvalues collateral, a borrower can over-borrow and leave bad debt. If it undervalues collateral, liquidations can be triggered, blocked, or sized incorrectly. The affected paths are core lending borrow and liquidation decisions.

**Proof of Concept**  
Code path: DeployAllSinglePriceOracles reads SEQUENCER and passes it to ChainlinkSinglePriceOracle, UniswapV3SinglePriceOracle, and DIAOracleV2SinglePriceOracle. ChainlinkSinglePriceOracle.getPriceUSD18 starts with _sequencerCheck(), which reverts when the sequencer is down or still inside the one-hour grace period. UniswapV3SinglePriceOracle.getPriceUSD18 immediately reads _getPoolPriceTokenDenomenator() and optionally _getChainlinkPriceFeedPrice18(); DIAOracleV2SinglePriceOracle.getPriceUSD18 immediately reads IDIAOracleV2.getValue() and optionally _getChainlinkPriceFeedPrice18(). Neither override calls _sequencerCheck(). DualOracleChainlinkUniV3._getChainlinkPrice directly calls latestRoundData on its multiply/divide feeds and getPrices also reads the hardcoded UniV3 static oracle, with no sequencer-state check before returning low/high prices. FraxlendPairCore._updateExchangeRate stores the returned low/high prices and borrowAsset()/liquidate() use them for solvency without any sequencer-state check.

**Mitigation**  
Call _sequencerCheck() at the start of every overriding getPriceUSD18 implementation, and add an equivalent sequencer guard to DualOracleChainlinkUniV3 or a shared Fraxlend oracle wrapper. Refactor the guard into a modifier/internal precheck that derived oracles cannot bypass. Add L2 recovery tests for UniV3, DIA, and DualOracleChainlinkUniV3 with a mock sequencer feed, asserting reads revert while the feed is down and during the recovery grace period.

### [M-33] Single-hop zaps use hardcoded V3 pool parameters instead of the configured route

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/Zapper.sol:131; contracts/contracts/Zapper.sol:154; contracts/contracts/Zapper.sol:159; contracts/contracts/Zapper.sol:162; contracts/contracts/Zapper.sol:167; contracts/contracts/Zapper.sol:174; contracts/contracts/Zapper.sol:177; contracts/contracts/dex/UniswapDexAdapter.sol:46; contracts/contracts/twaputils/V3TwapUtilities.sol:18; contracts/contracts/libraries/PoolAddress.sol:29; contracts/contracts/dex/AerodromeDexAdapter.sol:21; contracts/contracts/dex/AerodromeDexAdapter.sol:82; contracts/contracts/dex/AerodromeDexAdapter.sol:99; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:18; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:35; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:38; contracts/contracts/libraries/PoolAddressSlipstream.sol:32; contracts/contracts/libraries/PoolAddressSlipstream.sol:37

**Summary/Description**  
Zapper stores a concrete V3 route pool in zapMap, but the single-hop helper derives the TWAP/slippage pool from hardcoded parameters instead of that route. For Uniswap-style adapters it always asks for the 1 percent fee tier. For Aerodrome, the fee overload reverts, the catch path falls back to tick spacing 200, and AerodromeDexAdapter.swapV3Single also encodes tick spacing 200 in the actual router path. If the configured route uses any other fee tier or Slipstream tick spacing, zero-min zaps quote from the wrong pool; on Aerodrome they also execute through the hardcoded tick-200 pool rather than the configured route.

**Root Cause**  
_swapV3Single receives the route-derived parameter from _getPoolFee(pool1), but it ignores that value while deriving _v3Pool for the TWAP quote and _slippage lookup. It first calls DEX_ADAPTER.getV3Pool(_in, _out, uint24(10000)); Aerodrome rejects that overload and the catch path calls the int24 overload with 200. V3TwapUtilities/PoolAddress and V3TwapAerodromeUtilities/PoolAddressSlipstream include the fee or tick spacing in pool identity, so this changes the selected pool. AerodromeDexAdapter.swapV3Single compounds the issue by ignoring the supplied _fee argument and always encoding TICK_SPACING = 200 in the Universal Router path.

**Pre_conditions**  
A Zapper/IndexUtils route is configured as a single-hop V3 route whose pool1 fee is not 10000 on a Uniswap-style adapter, or whose Aerodrome Slipstream tickSpacing is not 200. Alternatively, the hardcoded 1 percent or tick-200 pool for the same token pair is missing, thin, stale, or easier to move than the configured route. A caller uses addLPAndStake or another zap flow with amountOutMin set to zero or a loose minimum.

**Impact**  
For zero-min single-hop zaps, the output floor and owner-set per-pool slippage can be based on the hardcoded pool rather than the configured route pool. On Uniswap-style adapters the swap still executes through the configured fee tier, but the minimum can be weakened by manipulating the hardcoded 1 percent quote pool. On Aerodrome, the actual swap also executes through the hardcoded tick-200 path, so a route configured to a deeper non-200 Slipstream pool can instead trade against a thin or attacker-created tick-200 pool, or revert if that pool lacks code or observations. The value at risk is the slippage loss on affected zap swaps.

**Proof of Concept**  
Configure a single-hop V3 zap route for tokenIn/tokenOut using a deep pool whose fee is not 10000, or on Aerodrome using a Slipstream pool whose tickSpacing is not 200. A user calls addLPAndStake with amountOutMin = 0. Zapper calls _swapV3Single with the route-derived _fee, but for the quote it asks getV3Pool(tokenIn, tokenOut, 10000); on Aerodrome this falls back to getV3Pool(tokenIn, tokenOut, int24(200)). The helper reads the TWAP and slippage override from that computed pool. The final call DEX_ADAPTER.swapV3Single(..., _fee, ...) uses the configured fee on Uniswap-style adapters, but AerodromeDexAdapter ignores _fee and encodes tokenIn, 200, tokenOut. A manipulated hardcoded pool can therefore set an underprotected minimum, and on Aerodrome can be the pool that actually receives the swap.

**Mitigation**  
Derive quote, slippage, and execution parameters from the configured route pool instead of probing hardcoded values. For Uniswap-style routes, use the route pool fee or the route pool address directly. For Aerodrome, read ICLPool(pool1).tickSpacing() and pass that tick spacing to both pool lookup and router path encoding, or change the adapter interface to accept an explicit pool address/route key. Reject zero-min swaps when the selected quote pool has no code, insufficient observations, or does not match the configured route.

### [M-34] Stale voting conversion factors let old stakes keep excess reward shares

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/voting/VotingPool.sol:38; contracts/contracts/voting/VotingPool.sol:47; contracts/contracts/voting/VotingPool.sol:54; contracts/contracts/voting/VotingPool.sol:58; contracts/contracts/voting/VotingPool.sol:62; contracts/contracts/voting/VotingPool.sol:66; contracts/contracts/voting/VotingPool.sol:73; contracts/contracts/voting/VotingPool.sol:87; contracts/contracts/voting/VotingPool.sol:96; contracts/contracts/voting/VotingPool.sol:114; contracts/contracts/TokenRewards.sol:231; contracts/contracts/TokenRewards.sol:326; contracts/contracts/voting/ConversionFactorSPTKN.sol:35; contracts/contracts/voting/ConversionFactorSPTKN.sol:45

**Summary/Description**  
VotingPool stores each user's last staking-asset conversion factor and denominator, but refreshes that pair only when the same user stakes or explicitly calls update. If the owner changes an asset's conversion-factor metadata through addOrUpdateAsset, if a live conversion factor such as ConversionFactorSPTKN moves down with the PEAS/stable TWAP and LP valuation, or if the owner disables an asset to stop further state refreshes, existing vlPEAS balances and TokenRewards shares remain at the old higher conversion. A user can therefore stake when the factor is high, avoid update after the factor falls, metadata is corrected, or the asset is disabled, and continue claiming VotingPool rewards and any external voting power with stale inflated shares until they exit.

**Root Cause**  
Stake accounting caches stakedToOutputFactor and stakedToOutputDenomenator in the user Stake struct, and neither addOrUpdateAsset, disableAsset, claimReward, nor any third-party path refreshes another user's stake. update routes through _updateUserState and requires assets[_asset].enabled, so disabling an asset blocks voluntary reconciliation while leaving existing ERC20/reward shares intact. unstake also uses the cached pair directly instead of first reconciling to the current conversion factor. Since conversion metadata can be changed by the owner and ConversionFactorSPTKN prices the LP token from live TWAP/reserve data, the cached factor is not guaranteed to match the current asset metadata or valuation.

**Pre_conditions**  
A VotingPool asset is configured with a conversion factor that can decrease, most directly an spTKN using ConversionFactorSPTKN, or the owner updates the asset's conversion-factor metadata to a lower/corrected value or disables the asset after existing stakes are present. A user stakes while the factor is higher than a later value. Rewards are deposited into the VotingPool rewards contract or vlPEAS is used as voting power before the user voluntarily refreshes or exits.

**Impact**  
Users with stale high conversions receive an excessive share of future VotingPool rewards, reducing rewards for users who stake or update at the lower current factor. The same stale ERC20 balance can overstate voting power for any consumer of vlPEAS balances. Disabling an asset does not remove or reconcile existing disabled-asset shares, so holders can continue claiming with the stale weight and then exit through unstake without accepting the lower/current factor. The distortion can persist indefinitely because only the affected user can update their cached conversion while the asset remains enabled, and it is repeatable across price cycles, temporary factor inflation, metadata corrections, or disabled-asset transitions.

**Proof of Concept**  
Example: an spTKN staker deposits 100 spTKN when ConversionFactorSPTKN, or the asset metadata then configured in VotingPool, returns factor = 2 * Q96 and denominator = Q96, so VotingPool mints 200 vlPEAS and TokenRewards tracks 200 shares. Later the fair/current factor becomes 1 * Q96, either from TWAP/LP valuation movement or from addOrUpdateAsset pointing the asset at corrected conversion metadata. If the owner disables the asset at this point, stake and update now revert through _updateUserState because assets[_asset].enabled is false, but the user's 200 vlPEAS and 200 reward shares remain. A second enabled-asset user stakes equivalent value and receives 100 shares. When 300 PEAS of rewards are deposited, TokenRewards distributes by the stale 300 total shares, paying 200 PEAS to the old disabled-asset stake and 100 PEAS to the current stake; with current conversion both positions should have had 100 shares and received 150 PEAS each. The first user can also call unstake with the old 200 vlPEAS amount, which pays accrued rewards during the burn and returns the original 100 spTKN without ever reconciling the stale factor.

**Mitigation**  
Reconcile the user's stake against the current conversion factor before claims, votes, and unstake, or make conversion updates permissionless per user/asset so anyone can force stale positions to the current factor. For assets whose conversion factor is expected to be non-monotonic, avoid persistent cached voting balances and derive voting/reward weight from current staked amount times the current factor, or checkpoint reward epochs with the conversion snapshot that was valid for that epoch. If disabled assets should stop participating, disableAsset should also move existing positions into an exit-only reconciled state or expose a forced reconciliation path before further reward claims.

### [M-35] Just-in-time VotingPool stakes dilute reward deposits

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/voting/VotingPool.sol:34; contracts/contracts/voting/VotingPool.sol:38; contracts/contracts/voting/VotingPool.sol:41; contracts/contracts/voting/VotingPool.sol:62; contracts/contracts/voting/VotingPool.sol:75; contracts/contracts/voting/VotingPool.sol:114; contracts/contracts/TokenRewards.sol:102; contracts/contracts/TokenRewards.sol:113; contracts/contracts/TokenRewards.sol:180; contracts/contracts/TokenRewards.sol:206; contracts/contracts/TokenRewards.sol:231; contracts/contracts/TokenRewards.sol:325

**Summary/Description**  
VotingPool grants reward-tracking vlPEAS shares immediately when a user stakes, while its pre-swap fee hook is a no-op. A user who stakes immediately before a VotingPool reward deposit is included in the current TokenRewards denominator and receives a pro-rata part of rewards that were intended for the already locked voting stake base.

**Root Cause**  
VotingPool stake/update mints current reward shares without any reward epoch, stake-age requirement, or pending-reward checkpoint. The TokenRewards share hook calls processPreSwapFeesAndSwap before adding shares, but VotingPool implements that hook as a no-op, and TokenRewards depositRewards later divides the deposited reward amount by the then-current totalShares.

**Pre_conditions**  
A VotingPool asset is enabled and existing users have staked. A reward deposit to the VotingPool REWARDS contract is pending, predictable, or externally visible before execution. The new staker can acquire the enabled staking asset and tolerate the configured lockupPeriod; the reward amount is large enough to exceed the temporary capital cost.

**Impact**  
Existing VotingPool stakers lose a pro-rata share of each affected reward deposit to the just-in-time staker. The attacker can claim rewards immediately after the deposit through TokenRewards.claimReward, while only the principal unstake is delayed by VotingPool.lockupPeriod. For a deposit R, existing shares E, and just-in-time shares S, the attacker receives S/(E+S) of R; with S equal to E, half of the deposit is diverted from prior stakers.

**Proof of Concept**  
Assume the enabled asset has a 1:1 conversion and Alice has already staked 100 tokens, so TokenRewards.totalShares is 100. A 100 PEAS VotingPool reward deposit is about to be made. Bob stakes 100 tokens first; VotingPool._updateUserState mints 100 vlPEAS and VotingPool._update adds 100 TokenRewards shares. The VotingPool processPreSwapFeesAndSwap hook does nothing, so no prior reward snapshot is created. The reward depositor then calls depositRewards(PEAS, 100). TokenRewards._depositRewards uses the current totalShares of 200, so Alice and Bob each accrue 50 PEAS. Bob can immediately call claimReward(Bob) and later unstake his original 100 tokens after the lockup expires. Without Bob staking just before the deposit, Alice would have received the full 100 PEAS.

**Mitigation**  
Do not activate newly staked voting shares for the current reward deposit. Queue VotingPool stake changes to the next reward epoch, require a minimum stake age for reward eligibility, or snapshot eligible totalShares at reward notification time. Alternatively, make VotingPool reward deposits pull from an internal checkpointed accrual source and fully checkpoint pending rewards before stake changes are applied.

### [M-36] Self-lending LVF setup queries counterfactual aspTKN before deployment

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Low  
**Location**: contracts/contracts/lvf/LeverageFactory.sol:101; contracts/contracts/lvf/LeverageFactory.sol:103; contracts/contracts/lvf/LeverageFactory.sol:106; contracts/contracts/lvf/LeverageFactory.sol:109; contracts/contracts/oracle/aspTKNMinimalOracle.sol:23; contracts/contracts/oracle/aspTKNMinimalOracle.sol:24; fraxlend/src/contracts/FraxlendPairCore.sol:195; fraxlend/src/contracts/FraxlendPairCore.sol:197; fraxlend/src/contracts/FraxlendPairCore.sol:535; fraxlend/src/contracts/FraxlendPairDeployer.sol:304

**Summary/Description**  
LeverageFactory.createSelfLendingPodAndAddLvf computes the future aspTKN address, deploys an aspTKN oracle for that address, and then deploys the Fraxlend pair before the aspTKN is actually created. The Fraxlend pair constructor immediately updates its exchange rate, which calls the oracle. aspTKNMinimalOracle.getPrices() then calls convertToShares() on ASP_TKN, but that address has no code yet, so the pair deployment reverts and the self-lending setup path cannot complete.

**Root Cause**  
The self-lending flow uses the aspTKN as counterfactual collateral during Fraxlend pair deployment, but the selected oracle is not counterfactual-safe. aspTKNMinimalOracle always reads IERC4626(ASP_TKN).convertToShares() during getPrices(), and FraxlendPairCore calls getPrices() in its constructor before LeverageFactory creates the aspTKN.

**Pre_conditions**  
LeverageFactory.createSelfLendingPodAndAddLvf is called with the real FraxlendPairDeployer path and aspTKNMinimalOracle as the oracle factory output. No third-party action is required. The predicted aspTKN address is empty, which is the normal state at the point _createFraxlendPair() is called.

**Impact**  
The intended composed factory path for creating a self-lending pod and adding LVF support reverts before the pod and aspTKN are created. Operators cannot use this in-scope setup function for self-lending configurations without changing the deployment order, changing the oracle behavior, or manually wiring the components outside the factory flow.

**Proof of Concept**  
createSelfLendingPodAndAddLvf first calls _getOrCreateAspTkn(..., true), which only computes _aspTknAddy. It then creates _aspTknOracle for that address and calls _createFraxlendPair(). FraxlendPairDeployer.deploy() deploys FraxlendPair, whose constructor calls _updateExchangeRate(). _updateExchangeRate() calls IDualOracle(_aspTknOracle).getPrices(), and aspTKNMinimalOracle.getPrices() executes IERC4626(ASP_TKN).convertToShares(1e18). Because LeverageFactory does not call AutoCompoundingPodLpFactory.create() until later, ASP_TKN has no code at that point and the constructor call reverts.

**Mitigation**  
Create the aspTKN before deploying the Fraxlend pair and make the later step reuse and initialize that existing aspTKN instead of calling create() again. Alternatively, use a self-lending bootstrap oracle that does not query ASP_TKN until after the aspTKN has been deployed and initialized, then switch to the final oracle atomically.

### [M-37] Sell-fee pTKN repayment swaps can block leverage unwinds

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/DecentralizedIndex.sol:161; contracts/contracts/DecentralizedIndex.sol:171; contracts/contracts/lvf/LeverageManager.sol:388; contracts/contracts/lvf/LeverageManager.sol:448; contracts/contracts/lvf/LeverageManager.sol:465; contracts/contracts/dex/UniswapDexAdapter.sol:84; contracts/contracts/dex/UniswapDexAdapter.sol:101; contracts/contracts/dex/CamelotDexAdapter.sol:55; contracts/contracts/dex/CamelotDexAdapter.sol:72

**Summary/Description**  
removeLeverage sells returned pTKN when LP removal does not return enough paired/borrow token to repay the flash loan. That sell path calls swapV2SingleExactOut, which uses swapTokensForExactTokens. For pods with a nonzero pTKN sell fee, the transfer into the official V2_POOL is fee-on-transfer, but exact-output V2 routers compute and enforce the swap using pre-fee amounts. The pair receives less input than the router calculated, so the exact-output swap reverts instead of acquiring the repayment asset.

**Root Cause**  
The repayment top-up path combines fee-on-transfer pTKN sells with an exact-output V2 router function that does not support fee-on-transfer input tokens. The code uses the supporting-fee-on-transfer router only for exact-input swaps, while the removeLeverage repayment path is hardwired to exact output.

**Pre_conditions**  
The pod has a nonzero sell fee. A user removes leverage and the paired/borrow token received from removing LP, plus any _userProvidedDebtAmtMax, is less than the flash repayment amount, forcing _acquireBorrowTokenForRepayment to sell pTKN. The pod uses the Uniswap/Camelot exact-output adapter path; Aerodrome already has a separate unsupported-selector issue.

**Impact**  
Users can be unable to unwind or reduce leveraged positions through the intended removeLeverage path unless they provide the entire repayment shortfall in borrow tokens from their wallet or choose an unwind size that does not require selling pTKN. Positions that cannot be unwound can remain exposed to liquidation. This affects a core LVF exit path for any sell-fee pod and is repeatable under the same market/position conditions.

**Proof of Concept**  
In _removeLeveragePostCallback, if _pairedAmtReceived < _repayAmount, LeverageManager calls _acquireBorrowTokenForRepayment and then _swapPodForBorrowToken. _swapPodForBorrowToken approves all returned pTKN and calls swapV2SingleExactOut(_pod, _targetToken, _podAmt, targetOut, address(this)). UniswapDexAdapter and CamelotDexAdapter implement this with swapTokensForExactTokens. For a sell-fee pod, the router transfers the computed input amount to the pod's V2_POOL, triggering DecentralizedIndex._update with _sell == true and taking fees.sell before the pair receives tokens. The pair therefore receives less input than swapTokensForExactTokens priced, and the V2 invariant check fails. Example: if the router calculates that 100 pTKN is needed for 10 borrow tokens and the pod sell fee is 10%, only 90 pTKN reaches the pair, so the requested 10-token exact output is underpaid and the swap reverts.

**Mitigation**  
Do not use exact-output V2 swaps for pTKN repayment sells when pod sell fees can be nonzero. Use an exact-input supporting-fee-on-transfer swap with a caller-provided minimum borrow output and then repay up to the received amount, or make the pod fee-free for this explicit protocol unwind path and enforce a separate maximum input/minimum remaining pTKN bound.

### [M-38] Voting conversion factors underweight low-decimal pod assets

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/voting/ConversionFactorPTKN.sol:19; contracts/contracts/voting/ConversionFactorPTKN.sol:22; contracts/contracts/voting/ConversionFactorSPTKN.sol:35; contracts/contracts/voting/ConversionFactorSPTKN.sol:41; contracts/contracts/voting/ConversionFactorSPTKN.sol:43; contracts/contracts/voting/ConversionFactorSPTKN.sol:46; contracts/contracts/voting/VotingPool.sol:38; contracts/contracts/voting/VotingPool.sol:67; contracts/contracts/voting/VotingPool.sol:74; contracts/contracts/WeightedIndex.sol:120; contracts/contracts/WeightedIndex.sol:126

**Summary/Description**  
The voting conversion adapters use raw pod asset amounts as the factor for 18-decimal vlPEAS accounting. ConversionFactorPTKN returns pod.convertToAssets(1e18) directly, so a pTKN backed by one 6-decimal USDC returns 1e6 and staking one pTKN mints only 1e6 vlPEAS units instead of an 18-decimal value. ConversionFactorSPTKN compounds the same raw-unit mismatch in its LP invariant: with a pTKN/6-decimal-stable pool, Uniswap V2 LP totalSupply is sqrt(raw reserves), but the formula does not normalize by the paired asset decimals, so the returned spTKN factor is lower by the stable decimal gap. VotingPool then mints and burns vlPEAS directly from amount * factor / denominator, causing low-decimal pod and staked-pod holders to receive almost no voting/reward shares relative to equal-value 18-decimal pods.

**Root Cause**  
ConversionFactorPTKN assumes the raw token0 amount returned by WeightedIndex.convertToAssets(1e18) is already an 18-decimal value. ConversionFactorSPTKN similarly combines raw V2 reserves, raw LP supply, and the pTKN factor without normalizing low-decimal paired assets into the 18-decimal PEAS/vlPEAS accounting domain. VotingPool performs no independent decimal normalization before minting vlPEAS shares.

**Pre_conditions**  
The VotingPool owner enables a pTKN or spTKN asset with the PTKN/SPTKN conversion adapter, and the underlying pod asset or paired stable uses fewer than 18 decimals, such as a USDC-style 6-decimal token. Users stake that enabled asset and rewards are later distributed to vlPEAS holders.

**Impact**  
Affected low-decimal pod or staked-pod holders are under-credited by the decimal gap, for example about 1e12 for 6-decimal assets. They lose voting power and any rewards distributed through VotingPool's TokenRewards accounting, while correctly or over-credited assets receive a larger share of the same reward stream. The issue is repeatable for every stake/update involving the enabled low-decimal asset.

**Proof of Concept**  
For a single-asset pod backed 1:1 by USDC, WeightedIndex.convertToAssets(1e18) returns 1e6 raw USDC units. ConversionFactorPTKN returns factor = 1e6 and denominator = 1e18. VotingPool.stake(pod, 1e18) then mints 1e18 * 1e6 / 1e18 = 1e6 vlPEAS units, even though an equal-value 18-decimal pod mints 1e18 units. For spTKN, a 100 pTKN / 100 USDC V2 pool has raw reserves 100e18 and 100e6 and LP totalSupply about 100e12. ConversionFactorSPTKN's raw sqrt(k) math returns a factor of roughly 2 raw PEAS per raw LP share; the correct value is roughly 2e6 raw PEAS per raw LP share because the low-decimal paired reserve makes LP totalSupply 1e6 times smaller than an 18-decimal stable pool.

**Mitigation**  
Normalize conversion factors into the intended 18-decimal voting/reward unit. For PTKN, scale convertToAssets(1e18) by 10 ** (18 - underlyingDecimals) when the pod's token0 has fewer than 18 decimals. For SPTKN, include token decimals in the LP valuation, as the oracle does with kDec, or otherwise normalize paired-stable reserves and pTKN CBR before dividing by LP totalSupply. Add tests for 6-decimal paired/underlying assets and equal-value 18-decimal assets receiving comparable vlPEAS.

### [M-39] Q96 oracle prices can zero before decimal scaling

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:75-79; contracts/contracts/oracle/spTKNMinimalOracle.sol:189-209; contracts/contracts/twaputils/V3TwapUtilities.sol:95-108; contracts/contracts/twaputils/V3TwapKimUtilities.sol:94-107; contracts/contracts/twaputils/V3TwapCamelotUtilities.sol:91-104; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:98-111; contracts/contracts/voting/ConversionFactorSPTKN.sol:38-44; contracts/contracts/TokenRewards.sol:172-177; contracts/contracts/Zapper.sol:164-170

**Summary/Description**  
The UniV3 Q96 helpers reduce sqrtPriceX96 to an integer priceX96 before decimal scaling, and the inverse branch also computes Q96**2 / priceX96 before decimal scaling. For valid extreme ticks, either ordering can round a nonzero market price to zero even though a decimal-preserving full-precision formula would still return a nonzero normalized price. UniswapV3SinglePriceOracle can therefore return a zero quote without marking bad data or revert on inverse division by zero, and spTKN/aspTKN oracle, reward quote, zap, and voting conversion paths inherit the same zeroed value from the shared TWAP helpers.

**Root Cause**  
The fixed-point normalization floors the Q96 ratio too early. Direct branches compute floor(sqrtPriceX96^2 / Q96) and multiply by decimal scale afterwards; inverse branches compute floor(Q192 / priceX96) and then apply decimals. In both cases, fractional Q96 precision that should survive decimal scaling is discarded before the decimal multiplier is applied.

**Pre_conditions**  
A UniV3-style oracle path uses a valid but extreme tick and a token decimal gap that would make the final decimal-adjusted price nonzero. Direct zeroing occurs near the low end of the valid tick range; inverse zeroing occurs for high raw prices, for example inverse 18/6 paths above approximately tick 665455. The affected price path is used by a spTKN/aspTKN Fraxlend oracle, zap/reward quote, or voting conversion adapter.

**Impact**  
For lending oracles, a nonzero valid market price can become zero or cause the spTKN oracle calculation to revert, blocking exchange-rate updates and liquidation flows while the price remains in that range. If this occurs while collateral is falling, unhealthy debt can remain unliquidated and accrue avoidable bad debt. In non-lending consumers, the same truncation can understate conversion factors or min-out quotes for affected pools.

**Proof of Concept**  
Direct branch example: TickMath.MIN_SQRT_RATIO is 4295128739, so the current priceX96 calculation floor(4295128739^2 / Q96) returns 0. For an 18/6 decimal normalization, the decimal-preserving formula floor(4295128739^2 * 1e18 / Q96 / 1e6) returns 232, so the zero is caused by applying decimal scaling after flooring. Inverse branch example: let raw price = Q96 + 1 for an inverse branch with token1Decimals=18 and token0Decimals=6. The current formula computes inverseX96 = floor(Q96 / rawPrice) = 0, then normalized = 0 * 1e18 / 1e6 = 0, while floor(Q96 * 1e18 / rawPrice / 1e6) returns 999999999999.

**Mitigation**  
Apply decimal scaling in the same full-precision operation as the Q96 normalization or inverse calculation, for example by using FullMath.mulDiv with sqrtPriceX96^2 or Q192 and the relevant decimal multiplier before the final decimal divisor. Add coverage for direct low-tick and inverse high-tick pools with 18/6 decimal gaps.

### [M-40] V3 Q96 price scaling overflows for valid tick ranges

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:28; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:75; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:76; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:78; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:79; contracts/contracts/twaputils/V3TwapUtilities.sol:53; contracts/contracts/twaputils/V3TwapUtilities.sol:96; contracts/contracts/twaputils/V3TwapUtilities.sol:105; contracts/contracts/twaputils/V3TwapUtilities.sol:107; contracts/contracts/twaputils/V3TwapUtilities.sol:108; contracts/contracts/twaputils/V3TwapCamelotUtilities.sol:49; contracts/contracts/twaputils/V3TwapCamelotUtilities.sol:92; contracts/contracts/twaputils/V3TwapCamelotUtilities.sol:101; contracts/contracts/twaputils/V3TwapCamelotUtilities.sol:103; contracts/contracts/twaputils/V3TwapCamelotUtilities.sol:104; contracts/contracts/twaputils/V3TwapKimUtilities.sol:51; contracts/contracts/twaputils/V3TwapKimUtilities.sol:95; contracts/contracts/twaputils/V3TwapKimUtilities.sol:104; contracts/contracts/twaputils/V3TwapKimUtilities.sol:106; contracts/contracts/twaputils/V3TwapKimUtilities.sol:107; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:56; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:99; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:108; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:110; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:111; contracts/contracts/TokenRewards.sol:173; contracts/contracts/TokenRewards.sol:175; contracts/contracts/TokenRewards.sol:176; contracts/contracts/Zapper.sol:167; contracts/contracts/Zapper.sol:169; contracts/contracts/Zapper.sol:170; contracts/contracts/voting/ConversionFactorSPTKN.sol:39; contracts/contracts/voting/ConversionFactorSPTKN.sol:40; contracts/contracts/voting/ConversionFactorSPTKN.sol:41; contracts/contracts/voting/ConversionFactorSPTKN.sol:44; contracts/contracts/oracle/spTKNMinimalOracle.sol:189; contracts/contracts/oracle/spTKNMinimalOracle.sol:202; contracts/contracts/oracle/spTKNMinimalOracle.sol:204; contracts/contracts/oracle/spTKNMinimalOracle.sol:209; fraxlend/src/contracts/FraxlendPairCore.sol:535; fraxlend/src/contracts/FraxlendPairCore.sol:1117

**Summary/Description**  
The V3/Algebra price helpers correctly use FullMath for the sqrtPriceX96 squaring step, but then compose the resulting Q96 prices with decimal multipliers, Chainlink base prices, native-stable USD legs, voting factors, or user token amounts using plain checked multiplication before division. Valid UniV3-range prices can be as large as 224 bits, so the final normalized value can fit in uint256 while the intermediate product overflows. At the opposite extreme, priceX96 can floor to zero before inverse consumers divide by it. These valid-range reverts propagate through the single-price oracle used by spTKN/aspTKN pricing, the TWAP utility family, voting conversion, zaps, and paired-token reward conversion.

**Root Cause**  
Full precision is applied only to sqrtPriceX96 * sqrtPriceX96 / Q96. Subsequent fixed-point scaling still uses a * b / denominator with uint256 intermediates for 10**decimals normalization, quotePriceX96 * basePrice18, stableLegX96 * mainLegX96, priceX96 * amount, and related voting math. The inverse path also floors the direct Q96 price before inverting it.

**Pre_conditions**  
A configured UniV3/Algebra/Aerodrome oracle or swap-quote pool reaches a valid high tick above roughly 694605 when the Q96 price is multiplied by a 1e18-scale value or an ordinary 18-decimal amount, or a valid low tick below roughly -665455 before an inverse branch. Affected consumers call the oracle, TWAP utility, zap, reward-conversion, voting, or Fraxlend exchange-rate path.

**Impact**  
Oracle reads or quote computations revert instead of returning representable prices. For Fraxlend-backed LVF markets this makes _updateExchangeRate() revert, so borrowing checks and liquidation cannot refresh the exchange rate while the condition persists; an unhealthy position can avoid liquidation and accrue bad debt. The same arithmetic pattern can also block reward conversion, zaps, and voting conversion for affected pools.

**Proof of Concept**  
For any uint160 sqrtPriceX96, FullMath.mulDiv(sqrtPriceX96, sqrtPriceX96, Q96) itself is bounded because prod1 is below 2^64 and Q96 is 2^96. The overflow occurs after that safe step. With equal-decimal pool tokens and numerator token1, correctedPriceX96 equals priceX96 and the expected normalized result is priceX96. type(uint256).max / 1e18 is about 1.1579e59 Q96 units, equivalent to tick about 694605. A pool TWAP at tick 700000 is inside the valid V3 range and the final priceX96 fits below 2^224, but priceX96 * 1e18 reverts before the division by 1e18. The same phantom-overflow shape appears in quotePriceX96 * basePrice18 / Q96, stableLegX96 * mainLegX96 / Q96, and priceX96 * amount / Q96; for amount=1e18 the final output at the maximum tick still fits in uint256, but the intermediate product does not. On the low side, FullMath.mulDiv(sqrtPriceX96, sqrtPriceX96, Q96) can floor to zero for valid low ticks, causing inverse branches to divide by zero even when a decimal-preserving calculation would remain representable.

**Mitigation**  
Use FullMath.mulDiv for every Q96 price composition and decimal normalization step, not only for sqrt-price squaring. Cancel equal decimal factors before multiplying, apply inverse decimal scaling in the same full-precision operation, and compute inverse prices without first flooring the direct Q96 price to zero. Add boundary coverage for high and low valid ticks across oracle, TWAP, zap, reward, and voting consumers.

### [M-41] Multi-asset pod valuation omits non-first components

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/contracts/oracle/spTKNMinimalOracle.sol:145; contracts/contracts/oracle/spTKNMinimalOracle.sol:179; contracts/contracts/oracle/spTKNMinimalOracle.sol:231; contracts/contracts/WeightedIndex.sol:120; contracts/contracts/WeightedIndex.sol:185; contracts/contracts/voting/ConversionFactorPTKN.sol:19; contracts/contracts/voting/ConversionFactorSPTKN.sol:35

**Summary/Description**  
Several pricing and conversion paths treat WeightedIndex.convertToAssets() as a full pod value, but WeightedIndex is a basket: bond() and debond() add and return every indexTokens component, while convertToAssets() returns only the pro-rata amount of indexTokens[0]. For pods with more than one underlying asset, the LVF spTKN oracle and voting conversion adapters omit the value of all non-first components.

**Root Cause**  
WeightedIndex exposes ERC4626-style helpers in units of the first asset only. spTKNMinimalOracle._calculateBasePerPTkn() calls _accountForCBRInPrice(), which converts a pTKN amount through convertToAssets() and feeds that single-component amount into the LP valuation as the pTKN/base price. ConversionFactorPTKN and ConversionFactorSPTKN reuse the same first-asset conversion factor for voting share accounting. None of these callers loops over getAllAssets() to accumulate each component amount multiplied by its own base price or conversion unit.

**Pre_conditions**  
A pod with more than one underlying asset is enabled for LVF oracle-backed borrowing or as a VotingPool asset with the PTKN/SPTKN conversion adapter. The non-first component has nonzero value, which is the normal case for a valid basket.

**Impact**  
The LVF oracle underprices the pTKN side of the pTKN/base LP because the pTKN price includes only component zero. Since the final Fraxlend exchange rate is collateral tokens required per borrow asset, this overstates the required collateral and can deny otherwise valid borrows or make healthy multi-asset pod LP positions appear liquidatable. VotingPool users staking multi-asset pTKN or spTKN are similarly under-credited for voting and reward shares, shifting rewards and voting weight toward assets whose adapters include their full value.

**Proof of Concept**  
For an equal-weight two-asset pod with 18-decimal tokens A and B, no fees, and both assets worth 1 base, bonding 1 pTKN deposits 1 A and 1 B. Debonding 1 pTKN returns both assets, so the pTKN value is 2 base. WeightedIndex.convertToAssets(1e18) returns only 1 A. spTKNMinimalOracle then uses 1 base per pTKN in _calculateSpTknPerBase() instead of 2 base per pTKN, so the LP valuation is low by sqrt(2) in a balanced pool and the returned collateral-per-base exchange rate is high by sqrt(2). The same convertToAssets(1e18) value is returned as the VotingPool conversion factor, crediting only component A and omitting component B.

**Mitigation**  
For price and voting adapters, value a basket by iterating over getAllAssets() and summing each pro-rata component amount in a common unit. The oracle should use configured price sources for each component or reject multi-asset pods unless a full-basket valuation source is provided. Voting conversion factors should either accumulate all components into the intended 18-decimal reward unit or explicitly restrict supported pods to a single asset.

### [M-42] Zero-amount Fraxlend markets let old fTokens capture recapitalization and LAV liquidity

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: fraxlend/src/contracts/libraries/VaultAccount.sol:25; fraxlend/src/contracts/libraries/VaultAccount.sol:26; fraxlend/src/contracts/libraries/VaultAccount.sol:30; fraxlend/src/contracts/FraxlendPairCore.sol:624; fraxlend/src/contracts/FraxlendPairCore.sol:637; fraxlend/src/contracts/FraxlendPairCore.sol:640; fraxlend/src/contracts/FraxlendPairCore.sol:646; fraxlend/src/contracts/FraxlendPairCore.sol:651; fraxlend/src/contracts/FraxlendPairCore.sol:657; fraxlend/src/contracts/FraxlendPairCore.sol:729; fraxlend/src/contracts/FraxlendPairCore.sol:745; fraxlend/src/contracts/FraxlendPairCore.sol:755; fraxlend/src/contracts/FraxlendPairCore.sol:840; fraxlend/src/contracts/FraxlendPairCore.sol:843; fraxlend/src/contracts/FraxlendPairCore.sol:1168; fraxlend/src/contracts/FraxlendPairCore.sol:1177

**Summary/Description**  
VaultAccountingLibrary treats total.amount == 0 as a fresh 1:1 share market without checking whether total.shares is also zero. Fraxlend liquidation writeoffs reduce totalAsset.amount for bad debt but do not burn fToken supply, so a full writeoff can leave totalAsset.amount == 0 while totalAsset.shares remains nonzero. Later direct deposits mint only amount shares, letting old wiped fToken supply capture most recapitalization value. With an external LAV configured, old fToken holders can also call withdraw(amount): toShares returns amount shares, _redeem pulls idle LAV liquidity through _depositFromVault, and the old shares receive real assets despite a zero CBR.

**Root Cause**  
The first-deposit branch in toShares keys only on total.amount == 0. FraxlendPairCore.liquidate can subtract unpaid debt from totalAsset.amount while leaving totalAsset.shares unchanged, and deposit, _depositFromVault, and withdraw/_redeem do not reject or specially handle the inconsistent amount-zero/share-nonzero state.

**Pre_conditions**  
A Fraxlend pair experiences a full bad-debt writeoff that reduces totalAsset.amount to zero while fToken supply remains outstanding. This can occur when remaining borrower debt is written off with no asset repayment, for example a fully depleted collateral case. The pair is not paused. For the LAV draw variant, the pair has externalAssetVault configured and the LAV has available allocation/liquidity for the pair.

**Impact**  
New depositors or LAV holders can lose assets to prior fToken holders whose shares had already been written down to zero value. A direct deposit recapitalizes the pair while minting too few shares, so old shares own most of the new assets. In the LAV variant, old fToken holders can burn amount shares via withdraw(amount), force _redeem to pull the amount from LAV through _depositFromVault, and receive LAV assets even though convertToAssets(oldShares) was zero before the pull. Loss is bounded by later deposits or available LAV liquidity, but can be a large fraction of those assets when old supply is large.

**Proof of Concept**  
State after full writeoff: totalAsset.amount = 0, totalAsset.shares = 1,000, and old holders still own those fTokens. Direct recapitalization: a user calls deposit(100, user). VaultAccount.toShares returns 100 because total.amount == 0. _deposit sets totalAsset.amount = 100 and totalAsset.shares = 1,100, so the new user owns only 100/1,100 while old holders can redeem about 90.9 assets from the new deposit. LAV draw: if an external LAV has at least 100 available for the pair, an old holder with 100 shares calls withdraw(100, holder, holder). toShares again returns 100; _redeem sees LAV liquidity, calls _depositFromVault(100), then subtracts 100 amount and burns 100 old shares. The holder receives 100 real assets while the pair returns to totalAsset.amount = 0.

**Mitigation**  
Treat amount-zero/share-nonzero as an invalid or explicit recovery state. Block public deposits, _depositFromVault, and withdraw paths until the pair is recapitalized under a policy that handles old shares, burn or reset fToken supply when totalAsset.amount is written to zero, or change toShares to account for existing shares/virtual offsets so recapitalizing assets do not subsidize zero-value shares. Also reject zero-share or underpriced share mints in state-changing asset-supplying paths.

### [M-43] Checkpoint splitting distorts VariableInterestRate interest and future rates

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: fraxlend/src/contracts/VariableInterestRate.sol:120; fraxlend/src/contracts/VariableInterestRate.sol:124; fraxlend/src/contracts/VariableInterestRate.sol:126; fraxlend/src/contracts/VariableInterestRate.sol:127; fraxlend/src/contracts/VariableInterestRate.sol:131; fraxlend/src/contracts/VariableInterestRate.sol:133; fraxlend/src/contracts/FraxlendPairCore.sol:286; fraxlend/src/contracts/FraxlendPairCore.sol:319; fraxlend/src/contracts/FraxlendPairCore.sol:405; fraxlend/src/contracts/FraxlendPairCore.sol:410; fraxlend/src/contracts/FraxlendPairCore.sol:479; fraxlend/src/contracts/FraxlendPairCore.sol:481; fraxlend/src/contracts/FraxlendPairCore.sol:956; fraxlend/src/contracts/FraxlendPairCore.sol:959

**Summary/Description**  
VariableInterestRate updates the adaptive full-utilization rate with a non-additive half-life formula, and FraxlendPair charges each elapsed interval using the post-update rate. For the same utilization and wall-clock time, splitting the interval into many checkpoints can reduce charged interest while utilization is high. The same partitioning error also exists below MIN_TARGET_UTIL: repeated public checkpoints can decay the future full-utilization rate faster than a single elapsed-time checkpoint, even at zero borrow utilization.

**Root Cause**  
Both the low-utilization decay branch and high-utilization growth branch update fullUtilizationRate with a per-checkpoint rational multiplier based on deltaTime. The products of many small multipliers are not equal to one multiplier over the same total elapsed time. FraxlendPairCore._calculateInterest then charges high-utilization intervals with the returned post-update rate and _addInterest advances lastTimestamp on every internal checkpoint; the public addInterest entrypoint can also reach _addInterest at zero utilization because the skip gate does not apply when the cached utilization is zero.

**Pre_conditions**  
A pair uses VariableInterestRate. For the direct interest-shortfall path, utilization is above MAX_TARGET_UTIL and a borrower or helper can submit one state-changing no-op or dust call per block, such as addCollateral(0, borrower) on standard ERC20 collateral. For the rate-suppression path, utilization is below MIN_TARGET_UTIL, including zero borrow utilization, and any caller can repeatedly invoke public addInterest(false) or another path that reaches _addInterest before a later large borrow.

**Impact**  
Lenders receive less interest while the borrower saves the difference in high-utilization intervals. With the shipped 2-day half-life parameters at 100 percent utilization, old full-utilization rate of 5 percent APR, and 1,000,000 asset debt, one checkpoint after 172800 seconds charges about 546.90 assets. Checkpointing every 12 seconds charges about 469.87 assets, a 77.03 asset shortfall, or 14.08 percent of the interval interest. At low utilization, one 2-day checkpoint at maximum below-target deviation halves fullUtilizationRate, while 12-second checkpointing over the same time reduces it to about 36.8 percent of the starting value, lowering future borrow rates without corresponding current debt cost when utilization is zero or tiny.

**Proof of Concept**  
Code-level arithmetic: at full utilization deltaUtilization is 1e18. One checkpoint after H seconds returns newFull = 2 * oldFull and charges debt * 2 * oldFull * H / 1e18. Splitting the same H seconds into two H/2 checkpoints returns 1.5 * oldFull for the first slice and 2.25 * oldFull for the second slice, so charged interest is 1.875 * debt * oldFull * H / 1e18, 6.25 percent lower than one checkpoint. In the low-utilization branch at maximum below-target deviation, one checkpoint after H returns oldFull / 2, while two H/2 checkpoints return oldFull / 2.25; many small checkpoints approach oldFull / e over the same H. FraxlendPair.addInterest(false) is public, and when _prevUtilizationRate is zero its skip condition is false, so repeated zero-utilization checkpoints can force this faster decay.

**Mitigation**  
Make rate evolution and interest accrual invariant to checkpoint partitioning. Accrue elapsed high-utilization intervals with the stored previous rate and update the adaptive rate only for future intervals, or compute and charge the exact integrated average rate for the chosen curve. For low-utilization updates, use a time-additive or continuous formula whose result does not depend on checkpoint frequency. Also consider rejecting no-op checkpoint paths or routing public checkpointing through one consistent accrual policy.

### [M-44] Camelot exact-output swaps can use a different V2 pool family than LP setup

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/dex/CamelotDexAdapter.sol:18; contracts/contracts/dex/CamelotDexAdapter.sol:49; contracts/contracts/dex/CamelotDexAdapter.sol:55; contracts/contracts/dex/CamelotDexAdapter.sol:71; contracts/contracts/dex/CamelotDexAdapter.sol:72; contracts/contracts/dex/UniswapDexAdapter.sol:49; contracts/contracts/dex/UniswapDexAdapter.sol:57; contracts/contracts/DecentralizedIndex.sol:139; contracts/contracts/lvf/LeverageManager.sol:455; contracts/contracts/lvf/LeverageManager.sol:465

**Summary/Description**  
CamelotDexAdapter inherits getV2Pool/createV2Pool from UniswapDexAdapter, so the official pod LP pool is resolved from the configured V2_ROUTER.factory(), and exact-input V2 swaps also use V2_ROUTER. However swapV2SingleExactOut ignores the configured router and approves/calls the hardcoded V2_ROUTER_UNI address. A V2 router prices and swaps through its own factory, so exact-output repayment swaps can query a different pair, or no pair, than the Camelot pool that setup/add/remove liquidity and staking use.

**Root Cause**  
The Camelot adapter has two independent V2 router authorities: the constructor-configured V2_ROUTER for pool lookup, LP management, and exact-input swaps, and a hardcoded router for exact-output swaps. The contract never checks that the hardcoded router exists on the current chain or that its factory/WETH match the configured Camelot router.

**Pre_conditions**  
A pod uses CamelotDexAdapter and the hardcoded V2_ROUTER_UNI does not resolve the same factory/pair as the configured Camelot V2 router. A leverage removal receives less paired/borrow token from unstaking/removing LP than the flash repayment amount, so LeverageManager must sell returned pTKN through swapV2SingleExactOut.

**Impact**  
The removeLeverage repayment path can revert even though the configured Camelot pTKN/borrow-token LP exists and has liquidity, leaving users unable to reduce or close leveraged positions through the intended path unless they provide the full repayment shortfall from their wallet. If a different pair exists behind the hardcoded router, the unwind can instead consume pTKN against unintended liquidity/pricing.

**Proof of Concept**  
DecentralizedIndex.setup stores V2_POOL from DEX_HANDLER.getV2Pool(address(this), PAIRED_LP_TOKEN), which Camelot inherits as IUniswapV2Factory(IUniswapV2Router02(V2_ROUTER).factory()).getPair. add/remove liquidity and Camelot exact-input swaps use the same configured V2_ROUTER. During removeLeverage, LeverageManager._swapPodForBorrowToken calls IDexAdapter(_pod.DEX_HANDLER()).swapV2SingleExactOut(_pod, targetToken, podAmt, targetOut, address(this)). CamelotDexAdapter.swapV2SingleExactOut then approves V2_ROUTER_UNI and calls V2_ROUTER_UNI.swapTokensForExactTokens, so the router-internal pair lookup is based on V2_ROUTER_UNI.factory() rather than V2_ROUTER.factory(). If those factories differ, the repayment swap does not use the pod's official Camelot LP pool.

**Mitigation**  
Do not hardcode an alternate exact-output router inside CamelotDexAdapter. Pass the exact-output helper/router through the constructor and require its factory and WETH to match the configured Camelot router, or implement exact-output handling against the same Camelot pair/factory used by getV2Pool. If exact-output cannot be supported safely, remove this path and use an exact-input swap with explicit caller slippage bounds.

### [M-45] Near-last debonds avoid configured debond fees

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: contracts/contracts/WeightedIndex.sol:175; contracts/contracts/WeightedIndex.sol:176; contracts/contracts/WeightedIndex.sol:183; contracts/contracts/DecentralizedIndex.sol:278; contracts/contracts/DecentralizedIndex.sol:281

**Summary/Description**  
WeightedIndex.debond waives the configured debond fee whenever _isLastOut(_amount) returns true, but _isLastOut treats any debond amount of at least 99% of current total supply as last out. A holder with 99% of pTKN supply can therefore redeem almost the entire pod backing without paying the debond fee even though other pTKN supply remains. The skipped fee would otherwise remain in the pod fee balance, be partially burned through the burn-fee path, and later be swapped/distributed through the fee-reward flow.

**Root Cause**  
The last-out fee switch is based on a percentage threshold, _debondAmount >= (_totalSupply * 99) / 100, instead of checking that the debond burns all remaining non-fee-exempt supply or that totalSupply becomes zero. WeightedIndex then applies the exemption before calculating _amountAfterFee, so the 99% exit receives a full pro-rata asset share and _processBurnFee receives zero fee amount.

**Pre_conditions**  
A pod has a nonzero debond fee and at least one non-exempt holder controls enough pTKN to debond at least 99% of totalSupply while some residual supply belongs to other holders or remains outside the exiting holder. The holder calls debond directly rather than using a whitelisted fee-free locker path.

**Impact**  
The exiting holder avoids the configured debond fee on up to 99% of pod backing. For default verification-pod-style fees this skips about 1% of that exit amount; with higher configured debond fees the skipped amount can be much larger. The lost fee would otherwise accrue to the pod fee balance, burn/CBR uplift, partner share, and LP reward flow, so remaining holders and stakers lose the fee value that the code intends to collect on non-final exits.

**Proof of Concept**  
Let totalSupply = 100 pTKN, pod assets = 100 units, debond fee = 10%, and the exiting holder own 99 pTKN while another holder owns 1 pTKN. Calling debond(99) satisfies _isLastOut because 99 >= 100 * 99 / 100. The function sets _amountAfterFee = 99, burns 99 pTKN, transfers 99 assets to the caller, and leaves 1 asset backing the residual 1 pTKN. If the same exit were treated as a normal non-final debond, _amountAfterFee would be 89.1, about 9.9 pTKN would enter the fee path, and only 89.1 assets would be paid to the exiting holder.

**Mitigation**  
Make the fee exemption match the actual invariant being protected. Only waive debond fees when the caller burns all remaining supply, or when amount equals totalSupply and no residual pTKN remains. If a dust tolerance is needed, cap it to a small absolute dust amount rather than 1% of supply, and document the intended exemption explicitly.

### [M-46] Pre-writeoff self-lending compounding shifts Fraxlend bad debt to aspTKN rewards

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/TokenRewards.sol:325; contracts/contracts/TokenRewards.sol:327; contracts/contracts/AutoCompoundingPodLp.sol:124; contracts/contracts/AutoCompoundingPodLp.sol:125; contracts/contracts/AutoCompoundingPodLp.sol:162; contracts/contracts/AutoCompoundingPodLp.sol:177; contracts/contracts/AutoCompoundingPodLp.sol:213; contracts/contracts/AutoCompoundingPodLp.sol:258; contracts/contracts/AutoCompoundingPodLp.sol:260; contracts/contracts/AutoCompoundingPodLp.sol:267; contracts/contracts/AutoCompoundingPodLp.sol:268; contracts/contracts/AutoCompoundingPodLp.sol:292; contracts/contracts/AutoCompoundingPodLp.sol:293; contracts/contracts/AutoCompoundingPodLp.sol:303; contracts/contracts/AutoCompoundingPodLp.sol:308; fraxlend/src/contracts/FraxlendPairCore.sol:624; fraxlend/src/contracts/FraxlendPairCore.sol:637; fraxlend/src/contracts/FraxlendPairCore.sol:640; fraxlend/src/contracts/FraxlendPairCore.sol:1166; fraxlend/src/contracts/FraxlendPairCore.sol:1171; fraxlend/src/contracts/FraxlendPairCore.sol:1177

**Summary/Description**  
In self-lending pods, aspTKN reward processing converts reward balances into the pod paired asset by first depositing borrow tokens into the Fraxlend pair and receiving fTokens. That processing is reachable from public aspTKN deposit, mint, withdraw, and redeem flows, and anyone can first call TokenRewards.claimReward for the aspTKN wallet so claimable staking rewards are held by the aspTKN. If a Fraxlend borrower is already economically underwater but liquidate has not yet applied the writeoff, the reward conversion deposits fresh borrow assets at the stale pre-loss fToken CBR. The later liquidation writeoff then reduces totalAsset.amount across the newly minted fTokens as well, so pending aspTKN reward value absorbs part of an existing pair shortfall and existing fToken holders lose less.

**Root Cause**  
Fraxlend bad debt is only recognized lazily inside liquidate, while AutoCompoundingPodLp is allowed to perform public, zero-min reward compounding into the self-lending Fraxlend pair before that writeoff is applied. The pair deposit path prices new fTokens from totalAsset.toShares using pre-writeoff totalAsset accounting, and the aspTKN reward-processing paths do not check whether the target self-lending pair has a pending liquidatable shortfall.

**Pre_conditions**  
A self-lending pod uses a Fraxlend pair fToken as its paired LP token. The aspTKN has claimable or held rewards that can be converted into the pair asset. A borrower in the same Fraxlend pair is economically underwater under the current oracle price, but liquidate has not yet run the writeoff. An existing fToken holder or any caller able to benefit from reducing fToken loss can call claimReward for the aspTKN and then execute a public aspTKN share-changing action with a small amount to trigger reward processing before liquidation.

**Impact**  
Existing Fraxlend fToken holders can dilute their pending bad-debt loss into aspTKN holders' unprocessed reward value. The shifted value is bounded by the amount of rewards converted into self-lending fTokens and by the pending writeoff percentage. This can be material for reward-heavy aspTKNs and is repeatable whenever meaningful rewards sit unprocessed during a pending bad-debt window.

**Proof of Concept**  
Example: a self-lending Fraxlend pair has totalAsset.amount = 1000 and totalAsset.shares = 1000, with an economically pending liquidation that will write off 500 assets once liquidate runs. Before that writeoff, the aspTKN has 100 assets worth of claimable rewards. A caller invokes TokenRewards.claimReward(address(aspTKN)) and then performs a minimal aspTKN deposit or redeem, causing _processRewardsToPodLp to convert the rewards. Because IS_PAIRED_LENDING_PAIR is true, AutoCompoundingPodLp swaps rewards to the pair asset and calls IFraxlendPair.deposit(100, address(this)), receiving 100 fTokens at the stale 1:1 rate. When liquidate later subtracts the 500 writeoff from totalAsset.amount, the pair has 1100 fTokens sharing 600 assets. The original 1000 fTokens are worth about 545.45 instead of 500, while the aspTKN reward-derived 100 fTokens are worth about 54.55, meaning about 45.45 of the pre-existing pair loss was shifted into aspTKN rewards.

**Mitigation**  
Before self-lending reward compounding deposits into a Fraxlend pair, require the pair to have no pending liquidatable shortfall or force bad-debt recognition/checkpointing first. More generally, Fraxlend deposits should not be priced from pre-writeoff accounting when an existing borrower shortfall is observable. Consider disabling permissionless reward compounding into self-lending pairs during liquidation windows, or routing rewards through a mechanism that applies any pending pair loss before minting new fTokens.

### [M-47] V2 route maps can ignore Aerodrome stable-pool curves

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/Zapper.sol:98; contracts/contracts/Zapper.sol:107; contracts/contracts/Zapper.sol:203; contracts/contracts/Zapper.sol:211; contracts/contracts/AutoCompoundingPodLp.sol:350; contracts/contracts/AutoCompoundingPodLp.sol:358; contracts/contracts/AutoCompoundingPodLp.sol:379; contracts/contracts/dex/AerodromeDexAdapter.sol:40; contracts/contracts/dex/AerodromeDexAdapter.sol:51; contracts/contracts/dex/AerodromeDexAdapter.sol:71; contracts/contracts/dex/AerodromeDexAdapter.sol:122; contracts/contracts/dex/AerodromeDexAdapter.sol:152

**Summary/Description**  
Zapper and AutoCompoundingPodLp expose route maps as pool-based V2 routing, but execution reduces those pool entries to a token path and calls IDexAdapter.swapV2Single. On Aerodrome, swapV2Single always builds Route({stable:false}) and liquidity/pool helpers also resolve the volatile pool through fee 0/false. A V2 route configured to a stable Aerodrome pool can therefore execute against the volatile pool for the same token pair instead of the configured stable curve.

**Root Cause**  
The adapter interface for V2 swaps carries only tokenIn, tokenOut, amount, minimum, and recipient. It does not carry the configured pool address, factory, or Aerodrome stable flag. The route maps store pool addresses, but Zapper._swapV2 and AutoCompoundingPodLp._swap only use those addresses to infer an intermediate token; the Aerodrome adapter then hardcodes the volatile curve for the actual swap and LP operations.

**Pre_conditions**  
The deployment uses AerodromeDexAdapter. A zapMap or swapMaps entry is configured with PoolType.V2 or a V2 pool address that is the stable pool for a token pair, while a volatile pool for the same pair exists or can be created with weaker liquidity. A user calls a zap with zero or loose min-out, or public aspTKN reward processing routes pending rewards through the configured V2 path with zero min-out.

**Impact**  
The swap can be priced on the wrong AMM curve. For user zaps this can produce materially less paired token than the configured stable route would provide when the caller uses a loose minimum. For aspTKN reward conversion, any caller can trigger public processing and route shared reward balances through the volatile pool, so holders receive the reduced LP output while the price difference is captured through the volatile pool. If no volatile pool has usable liquidity, the configured stable route can also be unusable despite existing stable-pool liquidity.

**Proof of Concept**  
Configure an Aerodrome Zapper route or AutoCompoundingPodLp swapMaps entry for tokenA/tokenB using the Aerodrome stable pool. During execution, _zap or _swap builds only [tokenA, tokenB] and calls DEX_ADAPTER.swapV2Single. AerodromeDexAdapter.swapV2Single constructs IAerodromeRouter.Route({from: tokenA, to: tokenB, stable: false, factory: defaultFactory}) and sends the trade through the volatile pool. The configured stable pool address is never passed to the router and cannot affect execution. A thin volatile pool for tokenA/tokenB can therefore set the execution price even though the route was configured with the stable pool.

**Mitigation**  
Carry V2 route identity through the adapter API. For Aerodrome, store and pass the stable flag or pool address and verify that the router executes the configured pool. For generic V2 routes, either require DEX_ADAPTER.getV2Pool(tokenIn, tokenOut) to equal the configured pool before using the route, or reject pool-map entries whose curve/factory cannot be represented by the adapter. Apply nonzero per-swap minimums to public reward processing paths.

### [M-48] Aerodrome LP fees accrue to the staking wrapper and cannot be claimed

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Medium  
**Location**: contracts/contracts/dex/AerodromeDexAdapter.sol:40; contracts/contracts/dex/AerodromeDexAdapter.sol:50; contracts/contracts/DecentralizedIndex.sol:139; contracts/contracts/DecentralizedIndex.sol:143; contracts/contracts/DecentralizedIndex.sol:144; contracts/contracts/StakingPoolToken.sol:67; contracts/contracts/StakingPoolToken.sol:77; contracts/contracts/interfaces/IAerodromePool.sol:46; contracts/contracts/interfaces/IAerodromePool.sol:94; contracts/contracts/interfaces/IAerodromePool.sol:97

**Summary/Description**  
Aerodrome V2 pools account LP trading fees as per-account claimable balances instead of automatically folding them into reserves like Uniswap V2. Pods using AerodromeDexAdapter set the pod V2_POOL to an Aerodrome pool and StakingPoolToken then custodies the LP tokens for users. The staking wrapper only transfers LP tokens in stake/unstake and has no path to call IAerodromePool.claimFees, while setup renounces the wrapper owner. Fees accrued while LP is staked therefore remain claimable by the StakingPoolToken address and never reach spTKN holders or reward accounting.

**Root Cause**  
The Aerodrome adapter and staking wrapper treat Aerodrome V2 LP tokens as if swap fees are represented entirely by LP token principal/reserves. Aerodrome exposes claimable0/claimable1 and claimFees as separate account-level fee accounting, but no in-scope contract calls claimFees on behalf of the StakingPoolToken holder address or books the claimed tokens into rewards.

**Pre_conditions**  
A pod is deployed with AerodromeDexAdapter and users stake the pod/paired Aerodrome V2 LP token through StakingPoolToken. The Aerodrome pool receives swap volume and accrues token0/token1 LP fees while those LP tokens are held by StakingPoolToken.

**Impact**  
Aerodrome LP stakers lose the AMM trading fees earned during staking. The fees are not paid to users on unstake, are not deposited into TokenRewards, and are not included in aspTKN compounding or oracle-facing accounting because the wrapper cannot claim them. This repeats for every Aerodrome-backed pod and can affect all trading-fee yield for staked LP liquidity.

**Proof of Concept**  
No test run. Static path: AerodromeDexAdapter.getV2Pool/createV2Pool select an IAerodromePool. DecentralizedIndex.setup stores that pool as V2_POOL and sets it as the StakingPoolToken stakingToken, then renounces StakingPoolToken ownership. Users stake by transferring LP tokens into StakingPoolToken and later unstake by receiving only the LP token principal. IAerodromePool exposes claimable0(address), claimable1(address), and claimFees(), but StakingPoolToken, DecentralizedIndex, IndexUtils, and AutoCompoundingPodLp never call claimFees. Because claimFees has no beneficiary parameter, the claimable fees for the LP holder address remain attached to StakingPoolToken and cannot be collected by individual stakers.

**Mitigation**  
Add an Aerodrome-aware fee claim path for staking wrappers that hold Aerodrome LP tokens. The wrapper or adapter should call IAerodromePool(stakingToken).claimFees, then route claimed token0/token1 into TokenRewards or a documented compounding path before share changes and withdrawals. The accounting should checkpoint claimed and pending fees to the stakers that earned them, and deployments should avoid renouncing control until this claim/distribution path exists.

### [M-49] FlashMint short-circuit lets zero-min fee swaps use skewed V2 reserves

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/DecentralizedIndex.sol:159; contracts/contracts/DecentralizedIndex.sol:185; contracts/contracts/DecentralizedIndex.sol:198; contracts/contracts/DecentralizedIndex.sol:203; contracts/contracts/DecentralizedIndex.sol:228; contracts/contracts/DecentralizedIndex.sol:232; contracts/contracts/DecentralizedIndex.sol:424; contracts/contracts/DecentralizedIndex.sol:425; contracts/contracts/DecentralizedIndex.sol:428; contracts/contracts/DecentralizedIndex.sol:430; contracts/contracts/DecentralizedIndex.sol:434; contracts/contracts/dex/UniswapDexAdapter.sol:61; contracts/contracts/dex/CamelotDexAdapter.sol:32; contracts/contracts/dex/AerodromeDexAdapter.sol:54

**Summary/Description**  
DecentralizedIndex swaps accumulated pTKN fees to PAIRED_LP_TOKEN through the canonical V2 adapter with amountOutMin set to zero. A plain pTKN sell into V2_POOL normally reaches _processPreSwapFeesAndSwap before pool balances change, but flashMint sets _shortCircuitRewards during the recipient callback. A searcher that has pTKN inventory can flash-mint pTKN, sell the temporary pTKN into the canonical pool while fee processing is suppressed, allow flashMint to burn the pre-funded inventory, then trigger a small non-pool transfer after _shortCircuitRewards is cleared. The queued fee swap then executes against the skewed V2 reserves without an output guard, and the searcher can restore the pool after the protocol trade.

**Root Cause**  
_feeSwap hardcodes a zero minimum output when routing pTKN fees through DEX_HANDLER.swapV2Single. _processPreSwapFeesAndSwap is reachable from ordinary non-pool pTKN transfers once the SWAP_DELAY and LP-relative balance threshold pass. flashMint suppresses fee processing for the whole callback, but it does not prevent the recipient from moving pTKN through the canonical V2 pool during that window, so current reserves can be changed before fee processing is re-enabled and before the zero-min fee swap is triggered.

**Pre_conditions**  
The pod has nonzero canonical V2 liquidity. DecentralizedIndex holds enough accumulated pTKN fees to pass _min after SWAP_DELAY, and the processed amount is economically meaningful relative to pool liquidity. The searcher has enough pTKN inventory or capital to satisfy the flashMint burn and configured transfer/sell/flash fees while leaving the pool skewed until the follow-on trigger transaction step.

**Impact**  
LP stakers and reward recipients can receive materially less PAIRED_LP_TOKEN from a fee-processing batch. Each batch is bounded by the held pTKN fee balance and _max of 1% of the pool pTKN balance, minus any partner share, but the path is repeatable as fees accrue. The lost reward value is captured through the AMM price movement instead of being deposited to TokenRewards.

**Proof of Concept**  
Conceptual sequence: assume the contract holds pTKN fees above the threshold and _lastSwap is old. The searcher pre-funds the flashMint recipient with X pTKN, calls flashMint(recipient, X), and during the callback sells the flash-minted X pTKN into the canonical pTKN/paired V2 pool. Because _shortCircuitRewards is 1, _processPreSwapFeesAndSwap returns and the sale can leave the pool with a lower pTKN price. The callback returns; flashMint burns the recipient's pre-funded X pTKN plus the fee and clears _shortCircuitRewards. The searcher then makes a dust pTKN transfer that is not from V2_POOL, so _update calls _processPreSwapFeesAndSwap. _feeSwap sells the queued fee amount with amountOutMin = 0 into the skewed pool. The searcher buys pTKN back afterward, capturing value from the protocol fee batch when the batch size exceeds the flashMint, transfer-tax, and AMM fee costs.

**Mitigation**  
Do not execute protocol fee conversion with an unconditional zero minimum output. Derive a minimum from an independent oracle or a sufficiently aged TWAP, split or cap fee batches by absolute liquidity, and skip processing when the current V2 quote deviates from the reference price. Also narrow flashMint's reward-processing bypass so callback-initiated pTKN transfers to the canonical V2 pool cannot leave reserves changed before deferred fee processing is re-enabled.

### [M-50] Fresh zero DIA prices can revert spTKN oracle base conversion

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/oracle/DIAOracleV2SinglePriceOracle.sol:20-38; contracts/contracts/oracle/spTKNMinimalOracle.sol:196-202; contracts/contracts/oracle/aspTKNMinimalOracle.sol:24-27; fraxlend/src/contracts/FraxlendPairCore.sol:535; fraxlend/src/contracts/FraxlendPairCore.sol:1117

**Summary/Description**  
DIAOracleV2SinglePriceOracle treats a fresh DIA value of zero as healthy data: it only marks the response bad when the DIA timestamp is stale, then returns price18 = quotePrice8 * basePrice18 / 1e8. When spTKNMinimalOracle uses that DIA wrapper as BASE_CONVERSION_DIA_FEED, _subBadData is false and _baseConvPrice18 is zero, so the default price path divides by zero. Any aspTKN oracle inheriting this path has the same failure before applying its share conversion.

**Root Cause**  
The DIA wrapper validates only freshness and optional Chainlink base-conversion data. It does not reject _quotePrice8 == 0 even though zero is not a usable price, and the spTKN base-conversion path assumes any non-bad conversion price is a nonzero denominator.

**Pre_conditions**  
A spTKN or aspTKN Fraxlend oracle is deployed with BASE_CONVERSION_DIA_FEED set. The configured DIA oracle has a current timestamp for the base conversion key but returns value 0, which the DIA reference contract permits because setValue has no nonzero guard. A Fraxlend update, liquidation, or other consumer calls getPrices while that value is current.

**Impact**  
Fraxlend exchange-rate updates revert through the oracle, blocking liquidations and other oracle-dependent risk checks while the zero DIA value remains current. If this happens during a collateral-price move, unhealthy debt can remain open and accrue avoidable bad debt. Direct consumers of the DIA single-price wrapper can also receive price18 = 0 with isBadData = false.

**Proof of Concept**  
Code path: DIAOracleV2SinglePriceOracle.getPriceUSD18 reads (quotePrice8, refreshedLast) from getValue(BASE_IN_CL/USD). If refreshedLast + staleAfterLastRefresh >= block.timestamp and quotePrice8 == 0, it leaves isBadData false and returns price18 = 0. spTKNMinimalOracle._getDefaultPrice18 then sees _subBadData == false and executes 1e18 * primaryPrice / baseConvPrice18, which reverts because baseConvPrice18 is zero. FraxlendPairCore._updateExchangeRate calls the oracle from update/liquidation flows and bubbles the revert.

**Mitigation**  
Reject zero DIA prices in DIAOracleV2SinglePriceOracle, preferably before any scaling, and make spTKNMinimalOracle fail closed on zero conversion denominators by returning bad data or reverting with an explicit oracle error. Add coverage for fresh zero DIA values in the BASE_CONVERSION_DIA_FEED path.

### [M-51] Zero Chainlink base answers revert instead of returning bad data

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:58-68; contracts/contracts/oracle/spTKNMinimalOracle.sol:218-224; fraxlend/src/contracts/FraxlendPairCore.sol:535; fraxlend/src/contracts/FraxlendPairCore.sol:1117

**Summary/Description**  
ChainlinkSinglePriceOracle marks nonpositive feed answers as bad inside _getChainlinkPriceFeedPrice18, but getPriceUSD18 divides by the base feed's returned price before incorporating the base bad-data flag. If the base conversion feed returns a current zero answer, _baseIsBad is true but _basePrice18 is zero, so the function reverts with division by zero instead of returning isBadData = true. spTKNMinimalOracle uses this wrapper for its Chainlink comparison leg, so affected Fraxlend oracle reads can revert outright.

**Root Cause**  
The base-feed denominator is used before validating that it is nonzero and before propagating _baseIsBad into _isBadData. The helper can return (_price18 = 0, _isBadAnswer = true), but the caller assumes the denominator is usable.

**Pre_conditions**  
A ChainlinkSinglePriceOracle call is made with a nonzero _priceFeedBase. The configured base feed returns answer == 0 with a timestamp that does not otherwise revert the freshness arithmetic. A spTKN/aspTKN oracle or direct consumer calls the Chainlink price path while that zero answer is current.

**Impact**  
The oracle read reverts instead of producing a bad-data flag. In Fraxlend markets that use the Chainlink comparison leg, exchange-rate updates and liquidation flows that call the oracle can be blocked while the zero base answer persists. If liquidations are needed during that period, lenders can accrue avoidable bad debt.

**Proof of Concept**  
Code path: getPriceUSD18 first reads the quote feed and stores _isBadData. When _priceFeedBase is nonzero, it calls _getChainlinkPriceFeedPrice18(_priceFeedBase). For answer == 0, the helper returns _basePrice18 = 0 and _baseIsBad = true. The caller then executes _price18 = (1e18 * _price18) / _basePrice18 before the next line ORs _baseIsBad into _isBadData, so the transaction reverts before bad data can be signaled.

**Mitigation**  
Check _baseIsBad or _basePrice18 == 0 before division and return a bad-data response without using the denominator. Keep zero/negative answer handling consistent across quote and base feeds, and add coverage for zero base-feed answers.

### [M-52] Chainlink timed-out rounds can pass freshness checks as complete oracle data

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:58; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:78; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:79; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:80; contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:81; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:26; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:32; contracts/contracts/oracle/DIAOracleV2SinglePriceOracle.sol:31; contracts/contracts/oracle/DIAOracleV2SinglePriceOracle.sol:36; contracts/contracts/oracle/spTKNMinimalOracle.sol:126; contracts/contracts/oracle/spTKNMinimalOracle.sol:218; contracts/contracts/oracle/spTKNMinimalOracle.sol:219; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:107; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:111; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:119; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:122; fraxlend/src/contracts/FraxlendPairCore.sol:535; fraxlend/src/contracts/FraxlendPairCore.sol:552; fraxlend/src/contracts/FraxlendPairCore.sol:555; fraxlend/src/contracts/FraxlendPairCore.sol:918; fraxlend/src/contracts/FraxlendPairCore.sol:1003; fraxlend/src/contracts/FraxlendPairCore.sol:1117

**Summary/Description**  
The Chainlink-consuming oracle paths discard roundId and answeredInRound from latestRoundData and use only answer plus updatedAt freshness. Legacy Chainlink-style aggregators can roll a timed-out latest round forward by copying the previous answer, setting updatedAt to the timeout time, and leaving answeredInRound behind the new roundId. In that state these helpers treat the copied answer as fresh, complete data and can feed it into spTKN/aspTKN or DualOracleChainlinkUniV3 prices used by Fraxlend.

**Root Cause**  
Chainlink reads are not round-complete. ChainlinkSinglePriceOracle._getChainlinkPriceFeedPrice18 ignores roundId, startedAt, and answeredInRound, and DualOracleChainlinkUniV3 reads latestRoundData directly with the same omission. Callers then rely on updatedAt to decide freshness, which is insufficient for feeds where timed-out rounds refresh updatedAt while the answer was computed in an earlier round.

**Pre_conditions**  
A Fraxlend market uses a Chainlink feed whose latestRoundData can return answeredInRound lower than roundId after a timed-out or incomplete round. The copied prior answer is still positive and inside any min/max bounds, and updatedAt is recent enough to pass the configured max delay. The stale answer is then used as a direct Chainlink price, base-conversion leg, spTKN/aspTKN secondary leg, or DualOracleChainlinkUniV3 leg.

**Impact**  
Fraxlend can store and act on an exchange rate built from an old Chainlink answer that appears fresh by timestamp. If the stale copied answer overvalues collateral or masks divergence from the AMM/TWAP leg, borrowers can pass the deviation gate and over-borrow; if it undervalues collateral, liquidations can be blocked or mis-sized. The strongest realistic outcome is bad debt or incorrect liquidation behavior in affected lending markets.

**Proof of Concept**  
Code path: ChainlinkSinglePriceOracle._getChainlinkPriceFeedPrice18 destructures latestRoundData as (, price,, lastUpdated,) and never checks answeredInRound against roundId. getPriceUSD18 marks data stale only when lastUpdated is older than block.timestamp - maxDelay. UniswapV3SinglePriceOracle and DIAOracleV2SinglePriceOracle reuse this helper for Chainlink base-conversion feeds, and spTKNMinimalOracle uses the same ChainlinkSinglePriceOracle for its comparison leg. DualOracleChainlinkUniV3 directly destructures latestRoundData for CHAINLINK_MULTIPLY_ADDRESS and CHAINLINK_DIVIDE_ADDRESS as (, answer,, updatedAt,) and only rejects nonpositive or stale timestamps. Chainlink FluxAggregator-style timed-out rounds copy the previous round answer, set updatedAt to the timeout update, and preserve the earlier answeredInRound, so these consumers accept a previous-round answer as current. FraxlendPairCore._updateExchangeRate stores the returned low/high prices and borrow, collateral-removal, and liquidation flows use them for solvency and deviation decisions.

**Mitigation**  
Validate round completeness wherever Chainlink feeds are read: require roundId != 0, updatedAt != 0, answeredInRound >= roundId, and timestamps not in the future before treating the answer as usable. Put this in one shared Chainlink helper and make DualOracleChainlinkUniV3 use it instead of direct latestRoundData reads. Consider failing closed in Fraxlend when any configured oracle leg reports incomplete data.

### [M-53] VotingPool strands pod rewards earned by staked spTKN

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Medium  
**Location**: contracts/contracts/voting/VotingPool.sol:38; contracts/contracts/voting/VotingPool.sol:40; contracts/contracts/voting/VotingPool.sol:47; contracts/contracts/voting/VotingPool.sol:53; contracts/contracts/StakingPoolToken.sol:101; contracts/contracts/StakingPoolToken.sol:104; contracts/contracts/StakingPoolToken.sol:107; contracts/contracts/TokenRewards.sol:113; contracts/contracts/TokenRewards.sol:126; contracts/contracts/TokenRewards.sol:236; contracts/contracts/TokenRewards.sol:251; contracts/contracts/TokenRewards.sol:325; contracts/contracts/voting/ConversionFactorSPTKN.sol:29

**Summary/Description**  
VotingPool supports staking StakingPoolToken assets through the spTKN conversion adapter, but it treats the staked asset as passive custody. When spTKN is transferred into VotingPool, StakingPoolToken._update moves the pod TokenRewards share balance to VotingPool. Future pod rewards then accrue to the VotingPool address, and claimReward(VotingPool) or a later unstake can transfer those reward tokens into VotingPool. VotingPool only returns the staked asset on unstake and has no path to claim, account, compound, distribute, or rescue the external pod rewards, so rewards earned during the voting stake are orphaned from the users who supplied the spTKN.

**Root Cause**  
The voting wrapper does not distinguish passive ERC20 assets from reward-bearing StakingPoolToken assets. VotingPool.stake and unstake only update stakes and mint or burn vlPEAS, while StakingPoolToken transfers have cross-contract side effects into the pod TokenRewards contract. No VotingPool function calls the pod rewards contract on behalf of stakers or books claimed reward-token balances into VotingPool.REWARDS.

**Pre_conditions**  
The VotingPool owner enables a pod StakingPoolToken as an asset, for example using ConversionFactorSPTKN. Users stake that spTKN into VotingPool. The underlying pod TokenRewards contract receives reward deposits while VotingPool holds the spTKN shares.

**Impact**  
Users who lock spTKN in VotingPool lose the pod rewards generated by that spTKN during the lock. The value remains claimable by or payable to the VotingPool address, but once transferred there it is not included in user stake accounting, not deposited into VotingPool.REWARDS, and not returned on unstake. The affected amount is the full pod reward yield attributable to VotingPool-held spTKN over the staking period, repeatable for every enabled spTKN voting asset.

**Proof of Concept**  
No test run. Static path: VotingPool enables an spTKN asset and Alice stakes 100 spTKN. The IERC20(_asset).safeTransferFrom in stake transfers the spTKN to VotingPool; because the asset is StakingPoolToken, its _update calls TokenRewards(POOL_REWARDS).setShares(Alice, 100, true) and setShares(VotingPool, 100, false). Later the pod rewards contract receives 100 reward tokens, so those rewards are owed to shares[VotingPool]. Anyone can call TokenRewards.claimReward(address(VotingPool)), or Alice's later VotingPool.unstake transfers spTKN out and StakingPoolToken._update calls _removeShares(VotingPool), which first distributes the owed reward tokens to VotingPool. VotingPool.unstake only transfers the spTKN principal back to Alice and has no function that transfers or redistributes the claimed reward tokens, leaving the 100 reward tokens in VotingPool.

**Mitigation**  
For reward-bearing staking assets, add an explicit pass-through accounting path. Before stake changes and unstakes, claim the underlying pod TokenRewards for VotingPool and distribute or checkpoint those tokens to the VotingPool stakers that earned them, or deposit them into VotingPool.REWARDS under the correct eligible-share snapshot. Alternatively, reject StakingPoolToken assets in VotingPool unless reward forfeiture is explicitly documented and recoverable through a scoped rescue mechanism.

### [M-54] Just-in-time pod LP stakes dilute direct reward deposits

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/StakingPoolToken.sol:67; contracts/contracts/StakingPoolToken.sol:77; contracts/contracts/StakingPoolToken.sol:101; contracts/contracts/TokenRewards.sol:102; contracts/contracts/TokenRewards.sol:113; contracts/contracts/TokenRewards.sol:180; contracts/contracts/TokenRewards.sol:206; contracts/contracts/TokenRewards.sol:231; contracts/contracts/IndexUtils.sol:51; contracts/contracts/DecentralizedIndex.sol:400; contracts/contracts/DecentralizedIndex.sol:407; contracts/contracts/DecentralizedIndex.sol:410

**Summary/Description**  
StakingPoolToken grants TokenRewards shares immediately when LP tokens are staked, and TokenRewards.depositRewards assigns each direct reward deposit across the current totalShares. The share hook tries to process pending pod fees before adding shares, but that does not protect an upcoming direct reward deposit or a protocol flash-fee reward routed through DecentralizedIndex.flash. A user can add or stake pod LP immediately before a visible reward deposit, or before calling flash when the fee is deposited to rewards, receive a pro-rata share of that reward, claim, and then unstake immediately because the pod LP staking wrapper has no lockup.

**Root Cause**  
Direct reward deposits are distributed by a point-in-time totalShares snapshot with no stake-age, epoch, or reward-notification eligibility rule. StakingPoolToken.stake mints the receipt token and the _update hook calls TokenRewards.setShares, which increases totalShares before the later TokenRewards.depositRewards call updates _rewardsPerShare. StakingPoolToken.unstake can remove the same stake immediately after the reward is claimed.

**Pre_conditions**  
A pod LP staking pool has existing stakers. A rewardsToken or whitelisted reward-token deposit to the pod TokenRewards contract is pending, predictable, or visible before execution, or the pod flash-fee branch routes DAI into TokenRewards because lpRewardsToken == DAI or PAIRED_LP_TOKEN == DAI. A user can acquire or create pod LP tokens and stake them before the reward deposit or flash call is processed.

**Impact**  
Existing pod LP stakers lose a pro-rata portion of each affected direct reward deposit or flash-fee reward to a just-in-time staker who did not provide liquidity during the period the reward was meant to compensate. For reward amount R, existing shares E, and just-in-time shares S, the new staker receives S/(E+S) of R. Unlike VotingPool, the pod LP staking wrapper has no principal lockup, so the temporary capital can be removed immediately after claiming. In the flash-fee branch this also rebates part of the borrower's fixed DAI fee away from pre-existing stakers.

**Proof of Concept**  
Direct deposit sequence: Alice has 100 staked pod LP, so TokenRewards.totalShares is 100. A 100 PEAS direct reward deposit is about to call TokenRewards.depositRewards. Bob adds or otherwise obtains 100 pod LP and calls StakingPoolToken.stake(Bob, 100), or routes through IndexUtils.addLPAndStake. StakingPoolToken._update calls TokenRewards.setShares(Bob, 100, false), and TokenRewards._addShares raises totalShares to 200. The reward depositor then calls depositRewards(PEAS, 100). TokenRewards._depositRewards adds 100 / 200 to _rewardsPerShare, so Bob is entitled to 50 PEAS. Bob calls claimReward(Bob) and then unstake(100), recovering his LP principal while Alice receives only 50 PEAS instead of the full deposit. Flash-fee sequence: Bob performs the same temporary stake before calling DecentralizedIndex.flash on a pod where lpRewardsToken == DAI. flash transfers FLASH_FEE_AMOUNT_DAI from Bob and immediately calls TokenRewards.depositRewards(DAI, FLASH_FEE_AMOUNT_DAI), so Bob receives the same S/(E+S) rebate of his own flash fee and can unstake after the flash completes.

**Mitigation**  
Do not make newly staked pod LP immediately eligible for already-announced direct reward deposits. Use reward epochs or a minimum stake age, queue new shares to become active after the current reward notification, or snapshot eligible totalShares before accepting a direct deposit. If direct deposits are intended to reward only current instantaneous stakers, document that policy and route campaign-style rewards through a streaming or epoch-based distributor instead.

### [M-55] aspTKN deposit hooks can redeem against inflated CBR

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: High  
**Location**: contracts/contracts/AutoCompoundingPodLp.sol:124; contracts/contracts/AutoCompoundingPodLp.sol:126; contracts/contracts/AutoCompoundingPodLp.sol:130; contracts/contracts/AutoCompoundingPodLp.sol:134; contracts/contracts/AutoCompoundingPodLp.sol:135; contracts/contracts/AutoCompoundingPodLp.sol:136; contracts/contracts/AutoCompoundingPodLp.sol:176; contracts/contracts/AutoCompoundingPodLp.sol:178; contracts/contracts/AutoCompoundingPodLp.sol:190; contracts/contracts/AutoCompoundingPodLp.sol:197; contracts/contracts/AutoCompoundingPodLp.sol:198; contracts/contracts/AutoCompoundingPodLp.sol:199; contracts/contracts/StakingPoolToken.sol:101; contracts/contracts/StakingPoolToken.sol:104; contracts/contracts/StakingPoolToken.sol:107; contracts/contracts/TokenRewards.sol:126; contracts/contracts/TokenRewards.sol:128; contracts/contracts/TokenRewards.sol:236; contracts/contracts/TokenRewards.sol:252

**Summary/Description**  
AutoCompoundingPodLp.deposit and mint add the incoming StakingPoolToken amount to _totalAssets before transferring the asset and before minting the depositor's aspTKN shares. StakingPoolToken transfers are not passive: their _update path calls TokenRewards.setShares, and TokenRewards can transfer pending reward tokens to the sender or receiver. If one of those reward-token transfers invokes a hook controlled by the depositor, the hook can call redeem() while _totalAssets already includes the deposit but totalSupply does not yet include the new shares. The redeem path prices the attacker's existing aspTKN at this temporarily inflated CBR and transfers out too much staked LP before the original deposit resumes and mints the new shares.

**Root Cause**  
AutoCompoundingPodLp violates checks-effects-interactions for ERC4626 deposits by increasing _totalAssets before an external asset transfer and by delaying _mint until after that transfer. The asset transfer can execute protocol-level callbacks through StakingPoolToken -> TokenRewards -> reward-token transfer, and the contract has no reentrancy guard around deposit/mint/redeem.

**Pre_conditions**  
The vault has existing aspTKN supply and staked LP assets, the attacker owns some existing aspTKN shares and enough StakingPoolToken to deposit, and a pending reward distribution during the StakingPoolToken transfer can invoke a hook controlled by the attacker. This is reachable for hook-capable reward tokens in the staking rewards list; if hook-capable rewards are excluded by token policy, the issue becomes a token-integration policy risk.

**Impact**  
The attacker can withdraw more staked LP than their existing aspTKN shares are entitled to, stealing value from the remaining aspTKN holders. The extraction is principal value from _totalAssets, not only the pending reward batch. With an attacker share fraction f and deposit size D relative to vault assets, the single-call profit is approximately D*f*(1-f)/(1-f+D) of vault assets, bounded by the staking reward-share accounting that has not yet credited the deposit to the vault. Realistic parameter choices can exceed 1% of vault assets and the path is repeatable whenever the callback preconditions can be recreated.

**Proof of Concept**  
Let the vault start with A = 100 staked LP in _totalAssets, S = 100 aspTKN supply, and the attacker holding x = 50 aspTKN. The attacker deposits d = 100 staked LP. deposit computes 100 new shares, then _deposit increments _totalAssets to 200 and calls safeTransferFrom before minting. During that StakingPoolToken transfer, TokenRewards distributes a hook-capable pending reward to the attacker. The hook calls redeem(50), which sees totalSupply still 100 and _totalAssets already 200, so it withdraws 100 staked LP instead of the fair 50 and burns the old 50 aspTKN. The original deposit then completes and mints 100 new aspTKN to the attacker. Final vault accounting is 100 staked LP backing 150 aspTKN; the attacker paid 100 staked LP, received 100 staked LP back, and still owns 100 aspTKN worth about 66.7 staked LP, versus the 50 staked LP value they had before.

**Mitigation**  
Make deposits and mints non-reentrant and avoid externally observable inflated accounting. Transfer the StakingPoolToken in first, force/settle any TokenRewards side effects, then update _totalAssets and mint shares from the post-transfer amount. Alternatively mint shares before increasing _totalAssets so reentrant share pricing cannot exclude the new shares, and add an explicit reentrancy guard around all ERC4626 entry points.

### [I-01] Untracked WeightedIndex token balances cannot be redeemed

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/WeightedIndex.sol:91; contracts/contracts/WeightedIndex.sol:186; contracts/contracts/DecentralizedIndex.sol:400

**Summary/Description**  
WeightedIndex totalAssets(address) returns only the internal _totalAssets mapping, and debond() pays only amounts derived from that mapping. If an index token reaches the pod outside bond accounting, such as a raw ERC20 transfer, positive balance drift, or flash() over-repayment, the extra balance is not included in totalAssets and no sync, skim, or rescue path exists for index assets.

**Root Cause**  
Underlying custody and managed-asset accounting are split, but only bond() and debond() update _totalAssets; other balance-increasing paths are neither rejected nor synchronized.

**Pre_conditions**  
An index token balance is sent or accrued to the pod without passing through bond(), or a flash borrower repays more than the pre-flash balance requirement.

**Impact**  
The excess balance remains in the pod contract but is not claimable by pTKN holders through debond(), is not shown by totalAssets(), and cannot be swept by an admin function. This is primarily a stuck-funds/user-error path rather than a demonstrated value extraction path.

**Proof of Concept**  
Transfer an index token directly to the WeightedIndex contract after a normal bond. totalAssets(token) is unchanged because it reads _totalAssets[token]. A full debond transfers only the recorded _totalAssets share, leaving the direct transfer balance in the contract.

**Mitigation**  
Either reject unsupported direct accounting assumptions in documentation and add an ownerless sync/skim policy, or make totalAssets/debond use actual balances for managed assets while preserving the intended share-price behavior.

### [I-02] Aerodrome stable pools bypass the TKN/pTKN V2 blacklist

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: contracts/contracts/WeightedIndex.sol:73; contracts/contracts/dex/AerodromeDexAdapter.sol:40; contracts/contracts/dex/AerodromeDexAdapter.sol:50; contracts/contracts/interfaces/IAerodromePoolFactory.sol:34

**Summary/Description**  
When blacklistTKNpTKNPoolV2 is enabled, WeightedIndex blacklists only the pool address returned by the configured DexAdapter. The Aerodrome adapter resolves and creates only the volatile pool for a pTKN/index-token pair, but the same Aerodrome factory also supports a separate stable pool for the identical token pair. That stable pool address is never marked in _blacklist, so it can receive pTKN and function as a TKN/pTKN V2 pool despite the configured blacklist.

**Root Cause**  
The blacklist records exact recipient addresses, while Aerodrome has two V2 pool identities per token pair keyed by the stable flag. AerodromeDexAdapter hardcodes stable=false/fee=0 for getV2Pool and createV2Pool, so WeightedIndex never blacklists the stable=true/fee=1 counterpart.

**Pre_conditions**  
The pod is deployed with blacklistTKNpTKNPoolV2 enabled on an Aerodrome-style deployment, and an index token is not the pod's PAIRED_LP_TOKEN. An attacker or user creates the stable Aerodrome pool for pTKN and that index token.

**Impact**  
Users can add liquidity and trade through the unblacklisted stable pool, bypassing the intended TKN/pTKN V2 pool restriction and any behavior that relies on that restriction, such as preventing non-official direct pTKN/index-token liquidity and related buy/sell fee routing assumptions. I did not identify a direct protocol fund-loss path from this alone.

**Proof of Concept**  
WeightedIndex.__WeightedIndex_init calls IDexAdapter.getV2Pool(address(this), token) and, if needed, IDexAdapter.createV2Pool(address(this), token), then marks only that returned address in _blacklist. AerodromeDexAdapter.getV2Pool uses IAerodromePoolFactory.getPool(token0, token1, 0) and createV2Pool uses createPool(token0, token1, 0). IAerodromePoolFactory also exposes getPool/createPool with bool stable, so getPool(pTKN, token, true) is a distinct V2 pool address. Since _blacklist[stablePool] remains false, DecentralizedIndex._update does not revert when _to is that stable pool.

**Mitigation**  
For Aerodrome-style adapters, blacklist both volatile and stable pool addresses for each blocked TKN/pTKN pair, or expose adapter-level enumeration of all V2 pool variants that must be blocked. Consider making the blacklist updateable by governance for new DEX variants or newly discovered pool identities.

### [I-03] DEX swap paths cannot enforce caller deadlines

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/interfaces/IDexAdapter.sol:23; contracts/contracts/interfaces/IDexAdapter.sol:31; contracts/contracts/interfaces/IDexAdapter.sol:39; contracts/contracts/dex/UniswapDexAdapter.sol:78; contracts/contracts/dex/UniswapDexAdapter.sol:101; contracts/contracts/dex/UniswapDexAdapter.sol:126; contracts/contracts/dex/CamelotDexAdapter.sol:49; contracts/contracts/dex/CamelotDexAdapter.sol:72; contracts/contracts/dex/CamelotDexAdapter.sol:95; contracts/contracts/dex/AerodromeDexAdapter.sol:75; contracts/contracts/dex/AerodromeDexAdapter.sol:101; contracts/contracts/Zapper.sol:194; contracts/contracts/Zapper.sol:225; contracts/contracts/DecentralizedIndex.sol:232; contracts/contracts/TokenRewards.sol:300; contracts/contracts/AutoCompoundingPodLp.sol:124; contracts/contracts/AutoCompoundingPodLp.sol:148; contracts/contracts/AutoCompoundingPodLp.sol:162; contracts/contracts/AutoCompoundingPodLp.sol:176; contracts/contracts/AutoCompoundingPodLp.sol:279; contracts/contracts/AutoCompoundingPodLp.sol:323; contracts/contracts/AutoCompoundingPodLp.sol:379; contracts/contracts/AutoCompoundingPodLp.sol:386; contracts/contracts/lvf/LeverageManager.sol:148; contracts/contracts/lvf/LeverageManager.sol:465; contracts/contracts/lvf/LeverageManager.sol:562

**Summary/Description**  
AMM swap paths cannot consistently enforce caller-provided deadlines. IDexAdapter swap methods omit any deadline argument, so the V2 adapters pass block.timestamp to downstream routers, Camelot and Aerodrome V3 swaps do the same, and Uniswap Router02 exactInputSingle has no adapter-level expiry check. Zapper's direct V3 multi-hop path supplies block.timestamp and its Curve path has no local deadline check. DecentralizedIndex fee swaps, TokenRewards reward conversion, and AutoCompoundingPodLp reward conversions all inherit the adapter deadline gap. AutoCompoundingPodLp deposit/mint/withdraw/redeem can trigger reward conversion with block.timestamp, and LeverageManager.removeLeverage exposes no deadline while forwarding block.timestamp into LP removal and using a deadline-less exact-output swap. Add-liquidity and addLeverage paths that do take a deadline preserve it for the LP router call, so an expired LP deadline reverts the whole transaction there.

**Root Cause**  
The shared swap interface omits a deadline parameter, and one user-facing unwind path omits a deadline entirely. Implementations substitute the current execution timestamp instead of validating the caller intent before token movement.

**Pre_conditions**  
A swap or removeLeverage transaction is included later than the caller intended, the configured min-out or exact-output bounds still permit execution, and no later caller-provided LP deadline exists to revert the transaction.

**Impact**  
Delayed execution can violate caller timing intent. The confirmed impact is bounded by existing min-out, exact-output, and slippage settings, and I did not identify shared-accounting corruption from the deadline omission alone.

**Proof of Concept**  
The IDexAdapter swap signatures at lines 23, 31, and 39 have no deadline argument. Uniswap V2 swap calls pass block.timestamp at lines 78 and 101; Camelot passes block.timestamp at lines 49, 72, and 100; Aerodrome passes block.timestamp at lines 75 and 101. Zapper._swapV3Multi calls the canonical V3 router exactInput with deadline: block.timestamp at line 198, while _swapCurve calls Curve exchange at line 225 without a local expiry. DecentralizedIndex._feeSwap and TokenRewards._swapForRewards reach the same adapter surface at lines 232 and 300. AutoCompoundingPodLp public ERC4626 entry points call _processRewardsToPodLp(0, block.timestamp) at lines 125, 149, 163, and 177, and its reward-conversion swaps use the deadline-less adapter calls at lines 279, 323, 379, and 386. LeverageManager.removeLeverage has no deadline parameter, uses swapV2SingleExactOut at line 465, and calls indexUtils.unstakeAndRemoveLP with block.timestamp at line 563.

**Mitigation**  
Add deadline arguments to the swap methods and removeLeverage, and enforce block.timestamp <= deadline before transferring tokens. For router variants without a native deadline field, add a local adapter check or use a deadline-aware wrapper when available.

### [I-04] Balancer flash callback drops nonzero loan fees

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: contracts/contracts/flash/BalancerFlashSource.sol:53; contracts/contracts/flash/BalancerFlashSource.sol:56; contracts/contracts/lvf/LeverageManager.sol:328; contracts/contracts/lvf/LeverageManager.sol:387

**Summary/Description**  
BalancerFlashSource.receiveFlashLoan decodes the flash data and writes _fData.fee from Balancer's _feeAmounts array, but then calls the downstream recipient with the original _userData. LeverageManager therefore decodes fee = 0 for Balancer-backed flash loans.

**Root Cause**  
The callback mutates a memory copy of FlashData but passes _userData instead of abi.encode(_fData), unlike UniswapV3FlashSource which re-encodes the populated struct.

**Pre_conditions**  
A borrow asset is configured to use BalancerFlashSource, Balancer's vault charges a nonzero flash loan fee for the token/amount, and addLeverage or removeLeverage relies on the propagated FlashData.fee to repay the flash loan.

**Impact**  
LeverageManager computes repayment as amount + fee. With the dropped Balancer fee it repays only the principal, so Balancer's post-callback repayment check reverts. This blocks addLeverage and removeLeverage for Balancer-backed positions while the external fee is positive. Current Balancer V2 documentation indicates flash loan fees are zero, so this is a latent liveness issue unless that protocol fee is activated.

**Proof of Concept**  
Static flow: receiveFlashLoan sets _fData.fee = _feeAmounts[0] at line 54, then calls callback(_userData) at line 56. LeverageManager decodes that original struct and uses _d.amount + _d.fee at lines 328 and 387, so the repayment excludes Balancer's fee. UniswapV3FlashSource performs the intended pattern by calling callback(abi.encode(_fData)).

**Mitigation**  
Pass abi.encode(_fData) to the recipient after setting the fee, and add a unit test that invokes receiveFlashLoan with a nonzero _feeAmounts[0] and asserts the receiver observes and repays amount + fee.

### [I-05] Ceil reward checkpoints can discard fractional rewards

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: contracts/contracts/TokenRewards.sol:247; contracts/contracts/TokenRewards.sol:249; contracts/contracts/TokenRewards.sol:258; contracts/contracts/TokenRewards.sol:261; contracts/contracts/TokenRewards.sol:325; contracts/contracts/TokenRewards.sol:347

**Summary/Description**  
TokenRewards floors unpaid rewards when calculating getUnpaid, but records post-claim and post-share-change checkpoints with a ceiling. Because claimReward accepts an arbitrary wallet, any caller can force a staker to checkpoint at the ceiling and discard the fractional remainder that would otherwise become claimable after later reward deposits.

**Root Cause**  
In TokenRewards._distributeReward and _resetExcluded, rewards[token][wallet].excluded is set with _cumulativeRewards(..., true), while getUnpaid reads _cumulativeRewards(..., false). The ceil checkpoint is larger than the floor-paid amount whenever the wallet has a fractional accrued reward.

**Pre_conditions**  
A wallet has reward shares and its cumulative reward for a token is non-integer in token base units. The reward token is not paused. Any caller can call claimReward(wallet), or a share update can reach _resetExcluded for that wallet.

**Impact**  
The affected staker can lose at most one base unit of each affected reward token per forced checkpoint. Repeating this after later reward deposits can keep discarding fractional carry, leaving the skipped units stranded in TokenRewards. The caller does not receive the skipped value, so this is a dust-level griefing/accounting residual rather than extractable reward theft.

**Proof of Concept**  
With two equal reward-share holders and a 1-unit reward deposit, each holder's exact entitlement is 0.5 units and getUnpaid returns 0. A third party calls claimReward(victim), causing excluded to be set to ceil(0.5) = 1 even though no reward was paid. After the next 1-unit deposit, the victim's cumulative floor is 1 and excluded is already 1, so the victim receives 0 instead of the 1 unit they would have received by waiting.

**Mitigation**  
Use floor checkpoints for paid/reward-debt accounting, or separately store fractional carry so public claims and zero-value share checkpoints cannot advance a wallet past unpaid fractional rewards. If the intended behavior is to burn residuals, make claimReward permissioned to the wallet or document the dust-loss tradeoff.

### [I-06] Whitelisted staking receipt rewards can self-own TokenRewards shares

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/RewardsWhitelist.sol:35; contracts/contracts/TokenRewards.sol:189; contracts/contracts/TokenRewards.sol:195; contracts/contracts/TokenRewards.sol:231; contracts/contracts/TokenRewards.sol:247; contracts/contracts/TokenRewards.sol:252; contracts/contracts/TokenRewards.sol:272; contracts/contracts/StakingPoolToken.sol:101; contracts/contracts/WeightedIndexFactory.sol:91

**Summary/Description**  
If a pod staking receipt token is accepted as a reward token, TokenRewards treats the same StakingPoolToken units as both reward inventory and reward shares. Depositing that token as a reward transfers staking shares into TokenRewards before the reward accumulator is updated, and later reward claims transfer the staking token back through StakingPoolToken._update, recursively mutating TokenRewards shares. A portion of the deposited staking receipt can remain owned by TokenRewards itself and become non-withdrawable.

**Root Cause**  
TokenRewards._isValidRewardsToken only checks the canonical rewardsToken or the external rewards whitelist and does not reject protocol-local receipt tokens such as trackingToken. StakingPoolToken transfers are not passive ERC20 transfers: every transfer calls TokenRewards.setShares for the sender and receiver, so using the staking token as a reward token creates self-referential accounting.

**Pre_conditions**  
The RewardsWhitelist owner whitelists a pod's StakingPoolToken address, or a pod is otherwise configured so its accepted reward token is its trackingToken. A holder then calls depositRewards with that staking token.

**Impact**  
The depositor's staking receipt is partly redistributed and partly retained as TokenRewards-owned shares. Those retained shares cannot be unstaked through TokenRewards, and future staking-token reward deposits continue to include TokenRewards as a reward-share holder. This is a configuration-mistake edge case rather than an untrusted-user path against normal whitelisted assets.

**Proof of Concept**  
Example with one holder and 100 staking shares: the holder deposits 10 staking tokens as rewards. The transfer first moves 10 shares from the holder to TokenRewards via StakingPoolToken._update. _depositRewards then distributes the same 10 staking tokens over the now-current 100 total shares, so the holder is owed 9 and TokenRewards is owed 1. When the holder claims, 9 staking tokens transfer back to the holder and TokenRewards self-accounts its 1-token entitlement, leaving 1 staking token/share owned by TokenRewards with no unstake path.

**Mitigation**  
Reject reward tokens that are protocol receipt or accounting tokens for the current rewards instance, at minimum trackingToken and INDEX_FUND. Keep the whitelist limited to passive external reward assets, and handle any intended paired-token reward mode through the explicit depositFromPairedLpToken path.

### [I-07] Historical reward token churn permanently grows reward loops

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: contracts/contracts/TokenRewards.sol:180; contracts/contracts/TokenRewards.sol:206; contracts/contracts/TokenRewards.sol:210; contracts/contracts/TokenRewards.sol:240; contracts/contracts/TokenRewards.sol:258; contracts/contracts/TokenRewards.sol:325; contracts/contracts/TokenRewards.sol:331; contracts/contracts/RewardsWhitelist.sol:35; contracts/contracts/RewardsWhitelist.sol:39; contracts/contracts/StakingPoolToken.sol:101; contracts/contracts/AutoCompoundingPodLp.sol:124; contracts/contracts/AutoCompoundingPodLp.sol:162; contracts/contracts/AutoCompoundingPodLp.sol:213

**Summary/Description**  
TokenRewards keeps an append-only _allRewardsTokens array and every first positive deposit of a whitelisted token permanently adds that token to claim and share-update loops. RewardsWhitelist.toggleRewardsToken(false) only blocks future deposits through _isValidRewardsToken; it does not remove historical entries or stop already-listed tokens from being distributed, checkpointed, or processed by AutoCompoundingPodLp when balances exist. The RewardsWhitelist cap limits only the currently active whitelist; after tokens are removed and new ones are whitelisted, any external caller can dust-deposit each new token into existing rewards contracts so historical entries keep accumulating.

**Root Cause**  
TokenRewards treats the first deposit of a reward token as a permanent loop-membership update, but it has no per-contract maximum, no pruning path, and no tie to the active RewardsWhitelist array. RewardsWhitelist.toggleRewardsToken enforces MAX only on _whitelistAry.length at the moment a token is active, so historical whitelist churn is not bounded.

**Pre_conditions**  
A TokenRewards contract has nonzero totalShares. The RewardsWhitelist owner uses the intended toggleRewardsToken flow over time to remove old reward tokens and add new ones while keeping the active whitelist length at or below MAX. An external caller holds dust amounts of each newly whitelisted token.

**Impact**  
For the active whitelist at one point in time, external dust-deposit growth is bounded to at most the active whitelisted tokens plus the built-in rewards/paired token paths. Across repeated whitelist rotations, however, the per-TokenRewards historical array can grow without a code-level cap. claimReward, staking-token mint/burn/transfer share updates, and AutoCompoundingPodLp deposit/withdraw reward processing all scale with this historical array, so enough churn can make those operations too expensive or uncallable. This is recorded as Info because the strongest path depends on long-term trusted whitelist management rather than a fully externally controlled immediate growth path.

**Proof of Concept**  
While token A is whitelisted, an external caller calls depositRewards(A, 1) after totalShares is nonzero; _depositRewards marks _depositedRewardsToken[A] and pushes A into _allRewardsTokens. If the owner later unwhitelists A and whitelists B, the active whitelist is still within MAX, but A remains in the TokenRewards array. The external caller repeats depositRewards(B, 1). Each repetition adds one permanent historical entry, and later claimReward, setShares-driven stake/unstake/transfer hooks, and AutoCompoundingPodLp._processRewardsToPodLp iterate over all historical entries.

**Mitigation**  
Make the bounded set used for reward loops match the intended active reward-token set. Options include enforcing a hard per-TokenRewards maximum on _allRewardsTokens, pruning tokens after all deposited rewards are distributed or after an explicit migration, using paginated/per-token claiming and checkpoint updates instead of full-array loops in share-changing paths, and preventing deposits that would add a token outside the bounded active set.

### [I-08] Zero-value sTKN transfers can revert for non-stakers

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/StakingPoolToken.sol:101; contracts/contracts/TokenRewards.sol:126; contracts/contracts/BulkPodYieldProcess.sol:14

**Summary/Description**  
StakingPoolToken treats every transfer as a reward-share mutation, including zero-value transfers. When a zero-share address transfers 0 sTKN, the sender-side hook calls TokenRewards.setShares(..., 0, true), which reaches _removeShares and reverts because shares[wallet] is zero. BulkPodYieldProcess.bulkTransferEmpty can therefore revert if a caller includes an sTKN instead of a passive pod token.

**Root Cause**  
StakingPoolToken._update does not skip reward-share hooks when _value is zero, and TokenRewards._removeShares requires the sender to already have shares even for a zero removal.

**Pre_conditions**  
A caller performs a zero-value StakingPoolToken transfer from an address with no TokenRewards shares, or passes that sTKN into BulkPodYieldProcess.bulkTransferEmpty while the helper has no shares.

**Impact**  
This is a standards/operational footgun rather than a protocol-wide blockage. The affected call reverts, but there is no stored queue or mandatory batch state; callers can omit the receipt token and process pod tokens or reward paths directly.

**Proof of Concept**  
Static path: BulkPodYieldProcess.bulkTransferEmpty calls sTKN.transfer(address(this), 0). StakingPoolToken._update calls TokenRewards.setShares(address(BulkPodYieldProcess), 0, true). TokenRewards._removeShares requires shares[wallet] > 0, so the call reverts for the helper's normal zero-share balance.

**Mitigation**  
Skip TokenRewards.setShares calls when _value == 0, or make TokenRewards._removeShares treat zero removals from zero-share wallets as a no-op.

### [I-09] AutoCompoundingPodLp deposit and mint previews ignore pre-pricing reward compounding

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: contracts/contracts/AutoCompoundingPodLp.sol:120; contracts/contracts/AutoCompoundingPodLp.sol:124; contracts/contracts/AutoCompoundingPodLp.sol:144; contracts/contracts/AutoCompoundingPodLp.sol:148; contracts/contracts/AutoCompoundingPodLp.sol:213; contracts/node_modules/@openzeppelin/contracts/interfaces/IERC4626.sol:81; contracts/node_modules/@openzeppelin/contracts/interfaces/IERC4626.sol:120

**Summary/Description**  
AutoCompoundingPodLp.previewDeposit and previewMint calculate from the cached CBR, but deposit and mint first call _processRewardsToPodLp before pricing shares or assets. If the vault already holds processable rewards, the mutable call increases _totalAssets before conversion while the preview result did not include that same current-block condition. previewDeposit can therefore return more shares than deposit actually mints, and previewMint can return fewer assets than mint actually takes, contrary to the ERC4626 preview bounds.

**Root Cause**  
The preview functions are simple cached-CBR reads, while the matching mutable functions perform lazy reward conversion and _totalAssets mutation before computing the ERC4626 exchange amount. The pending reward value is neither reflected in totalAssets/previews nor excluded from the mutable pre-pricing path.

**Pre_conditions**  
The aspTKN has nonzero reward-token or paired-token balances that _processRewardsToPodLp can convert into staked LP, and a user or integration calls previewDeposit or previewMint before calling the matching mutable function in the same transaction or block state.

**Impact**  
Integrations relying on ERC4626 preview bounds can receive fewer shares than previewDeposit advertised or pay more assets than previewMint advertised. The standalone impact is standards non-compliance and incorrect integration/slippage accounting; direct value-transfer variants from the same lazy reward accounting are already tracked separately in H-02, M-07, M-09, and M-12.

**Proof of Concept**  
Example with no fees for simplicity: totalSupply = 100 and _totalAssets = 100, while the vault holds rewards convertible to 100 additional staked LP. previewDeposit(100) reads the old CBR and returns 100 shares. deposit(100) first runs _processRewardsToPodLp, increasing _totalAssets to 200, then mints floor(100 * 100 / 200) = 50 shares. For mint, previewMint(100) returns 100 assets from the old CBR, but mint first increases _totalAssets to 200 and then requires 200 assets for the same 100 shares.

**Mitigation**  
Make previewDeposit and previewMint reflect the same pre-pricing reward value used by deposit and mint, or stop mutating reward accounting before share pricing in the mutable entrypoints. If exact conversion cannot be safely previewed in a view, maintain separate pending-reward accounting included in totalAssets/previews, or require reward compounding to happen through a separate keeper/user action before ERC4626 share changes.

### [I-10] Uninitialized aspTKN reports unlimited ERC4626 deposits and mints

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/AutoCompoundingPodLp.sol:77; contracts/contracts/AutoCompoundingPodLp.sol:86; contracts/contracts/AutoCompoundingPodLp.sol:116; contracts/contracts/AutoCompoundingPodLp.sol:124; contracts/contracts/AutoCompoundingPodLp.sol:140; contracts/contracts/AutoCompoundingPodLp.sol:148; contracts/contracts/AutoCompoundingPodLp.sol:209; contracts/contracts/AutoCompoundingPodLp.sol:428; contracts/README.md:93

**Summary/Description**  
AutoCompoundingPodLp explicitly supports deployment with pod unset and setPod called later, but maxDeposit and maxMint are pure functions that always return uint256.max. In the unset-pod state, deposit and mint cannot complete because the mutable paths resolve the ERC4626 asset through pod.lpStakingPool(), which reverts when pod is address(0). ERC4626 maxDeposit/maxMint should report 0 while deposits or mints are temporarily disabled or otherwise not accepted.

**Root Cause**  
The maxDeposit and maxMint implementations are unconditional constants and do not reflect the vault initialization state, even though the deposit and mint flows depend on pod being configured before _asset() can resolve the staking-pool token.

**Pre_conditions**  
An AutoCompoundingPodLp is deployed with _pod set to address(0), as supported by the constructor comment and self-lending deployment flow, and setPod has not yet been called.

**Impact**  
Integrations or users relying on ERC4626 maxDeposit/maxMint can be told that deposits and mints are unlimited even though the same calls currently revert. The direct impact is standards non-compliance and setup-window integration failure, not confirmed loss of vault funds.

**Proof of Concept**  
Deploy AutoCompoundingPodLp with _pod equal to address(0). maxDeposit(receiver) and maxMint(receiver) return type(uint256).max. A deposit or mint in the same state reaches _processRewardsToPodLp or _deposit, calls _asset(), and attempts pod.lpStakingPool() on address(0), so the mutable operation reverts instead of accepting the advertised limit.

**Mitigation**  
Return 0 from maxDeposit and maxMint until address(pod) is nonzero and the asset can be resolved, or disallow exposing the ERC4626 surface before pod initialization is complete. Optionally make asset() non-reverting in the uninitialized state only if that behavior is intentionally supported.

### [I-11] AutoCompoundingPodLp setPod accepts incompatible pod configuration

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/AutoCompoundingPodLp.sol:56; contracts/contracts/AutoCompoundingPodLp.sol:57; contracts/contracts/AutoCompoundingPodLp.sol:209; contracts/contracts/AutoCompoundingPodLp.sol:249; contracts/contracts/AutoCompoundingPodLp.sol:260; contracts/contracts/AutoCompoundingPodLp.sol:311; contracts/contracts/AutoCompoundingPodLp.sol:428; contracts/contracts/lvf/LeverageFactory.sol:101; contracts/contracts/lvf/LeverageFactory.sol:111; contracts/contracts/lvf/LeverageFactory.sol:264

**Summary/Description**  
AutoCompoundingPodLp lets the owner set a previously unset pod exactly once, but setPod only checks that the address is nonzero. After that assignment, the vault asset, reward conversion path, and LP routing all assume the pod matches the immutable self-lending flag, the immutable DEX adapter, and a valid staking pool for that same pod.

**Root Cause**  
The pod address is mutable during delayed initialization while the compatibility inputs are immutable or external, yet _setPod does not validate relationships such as pod.lpStakingPool() belonging to the pod, pod.DEX_HANDLER() matching the aspTKN/indexUtils adapter, or IS_PAIRED_LENDING_PAIR matching whether pod.PAIRED_LP_TOKEN() is a Fraxlend pair.

**Pre_conditions**  
An aspTKN is deployed with pod set to address(0), or otherwise has not yet had pod assigned. The owner or factory flow calls setPod with a pod whose paired asset, staking pool, or DEX adapter is incompatible with the aspTKN constructor configuration.

**Impact**  
The aspTKN can be permanently initialized into a bad state because _setPod rejects any later correction once pod is nonzero. Depending on the mismatch, deposits can use the wrong ERC4626 asset, reward processing can revert when a non-Fraxlend paired token is treated as a Fraxlend pair, or compounding can route against a different DEX than the pod uses. I did not confirm an untrusted-user path that forces the protocol factory to set a different pod than the one it deployed, so this is classified as an owner-misconfiguration/info-level issue.

**Proof of Concept**  
For a delayed aspTKN deployed with _isSelfLendingPod = true and _pod = address(0), the owner can call setPod with a regular pod whose PAIRED_LP_TOKEN is a plain ERC20. The call succeeds because setPod only checks nonzero and _setPod only checks that pod was unset. Later, any reward conversion for a token other than the paired token enters _tokenToPairedLpToken, sees IS_PAIRED_LENDING_PAIR true, and calls IFraxlendPair(pod.PAIRED_LP_TOKEN()).asset() against a non-Fraxlend token. Alternatively, if the pod was created with a different DEX adapter than the aspTKN constructor adapter, _pairedLpTokenToPodLp uses the aspTKN adapter for the paired-to-pod swap while indexUtils/pod.addLiquidityV2 use the pod/indexUtils adapter for LP operations.

**Mitigation**  
Validate pod compatibility before assignment. At minimum require a nonzero lpStakingPool whose INDEX_FUND is the pod, require the pod DEX handler to match the aspTKN and/or configured IndexUtils adapter, and when self-lending is enabled require pod.PAIRED_LP_TOKEN() to expose the expected Fraxlend pair fields and have collateralContract() equal this aspTKN. Consider moving the delayed pod assignment into an initializer that atomically wires the pod, oracle, and seed deposit.

### [I-12] Active LAV vault removal leaves orphaned utilized assets in global accounting

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/LendingAssetVault.sol:65; contracts/contracts/LendingAssetVault.sol:73; contracts/contracts/LendingAssetVault.sol:204; contracts/contracts/LendingAssetVault.sol:220; contracts/contracts/LendingAssetVault.sol:246; contracts/contracts/LendingAssetVault.sol:347; fraxlend/src/contracts/FraxlendPairCore.sol:654; fraxlend/src/contracts/FraxlendPairCore.sol:674; fraxlend/src/contracts/FraxlendPairCore.sol:1043; fraxlend/src/contracts/FraxlendPairCore.sol:1206

**Summary/Description**  
LendingAssetVault keeps utilized assets in both per-vault mappings and the global _totalAssetsUtilized total. setVaultWhitelist(false) removes the vault from _vaultWhitelistAry and deletes only vaultMaxAllocation, but it does not require the vault to be fully unwound and does not clear or settle vaultUtilization, vaultDeposits, _vaultWhitelistCbr, or _totalAssetsUtilized. If a live vault is removed, totalAssetsUtilized and totalAvailableAssets still include the orphaned utilization while global refresh loops no longer visit that vault.

**Root Cause**  
The whitelist removal path updates the enumerable whitelist/authorization state without validating or settling the accounting state that feeds _totalAssetsUtilized and totalAvailableAssets.

**Pre_conditions**  
The owner removes a vault that still has nonzero vaultUtilization or LAV-owned vault shares. This is a trusted-owner/operational path, so the issue is classified as Info.

**Impact**  
The removed vault can no longer call whitelistUpdate, whitelistDeposit, or whitelistWithdraw, and its CBR changes are skipped by _updateInterestAndMdInAllVaults. If a Fraxlend pair still points to the LAV as externalAssetVault, borrow/repay/liquidation paths that need the LAV callbacks can revert, while LAV availability can remain artificially constrained or stale until the owner re-whitelists the vault or manually redeems/settles the position.

**Proof of Concept**  
1. A whitelisted pair draws 100 assets from LAV, so vaultUtilization[pair] and _totalAssetsUtilized increase by 100. 2. The owner calls setVaultWhitelist(pair, false). The pair is removed from _vaultWhitelistAry and vaultMaxAllocation is deleted, but the 100 utilization remains in vaultUtilization[pair] and _totalAssetsUtilized. 3. Later LAV deposits/withdrawals update only _vaultWhitelistAry entries, so this orphaned utilization is not refreshed, while totalAvailableAssets remains reduced by it.

**Mitigation**  
Before removal, require vaultUtilization[vault] == 0 and no LAV-owned vault shares, or atomically redeem/settle the vault and update _totalAssetsUtilized before removing authorization. Keep enough state or authorization for a removed vault to return funds only if deliberate wind-down support is needed.

### [I-13] LAV maxVaults can block vault replacement after an over-cap update

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/LendingAssetVault.sol:338; contracts/contracts/LendingAssetVault.sol:347; contracts/contracts/LendingAssetVault.sol:351

**Summary/Description**  
LendingAssetVault lets the owner set maxVaults to any uint8 value without checking the current whitelisted vault count. setVaultWhitelist only enforces _vaultWhitelistAry.length < maxVaults when adding a vault, so lowering maxVaults below the existing whitelist length leaves the vault in an over-cap state where new vaults cannot be added or replaced until the cap is raised again or enough entries are removed.

**Root Cause**  
setMaxVaults does not preserve the intended length <= maxVaults invariant, and the whitelist add path treats maxVaults only as a future-add gate rather than validating the current configured state.

**Pre_conditions**  
The owner lowers maxVaults below the current _vaultWhitelistAry length. The deployment then needs to add or rotate to a new lending pair while existing active vaults should not be removed before their utilization is unwound.

**Impact**  
This is an owner-driven configuration liveness footgun rather than an untrusted-user exploit. New supported lending pairs can be blocked from configuration, and removing active vaults to get back under the cap can interact poorly with the existing active-vault removal accounting issue. The owner can usually recover by raising maxVaults again, so the issue is classified as Info.

**Mitigation**  
Require _newMax >= _vaultWhitelistAry.length in setMaxVaults, or add an explicit safe replacement/wind-down flow that prevents active vault removal while allowing controlled rotation. Consider exposing the current whitelist length so operators can validate cap changes before submitting them.

### [I-14] LAV ERC4626 views can diverge from stale state-changing CBR

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: contracts/contracts/LendingAssetVault.sol:102; contracts/contracts/LendingAssetVault.sol:106; contracts/contracts/LendingAssetVault.sol:116; contracts/contracts/LendingAssetVault.sol:120; contracts/contracts/LendingAssetVault.sol:126; contracts/contracts/LendingAssetVault.sol:132; contracts/contracts/LendingAssetVault.sol:136; contracts/contracts/LendingAssetVault.sol:142; contracts/contracts/LendingAssetVault.sol:148; contracts/contracts/LendingAssetVault.sol:152; contracts/contracts/LendingAssetVault.sol:190; contracts/contracts/LendingAssetVault.sol:204; contracts/contracts/LendingAssetVault.sol:211; contracts/contracts/LendingAssetVault.sol:256; fraxlend/src/contracts/FraxlendPairCore.sol:286; fraxlend/src/contracts/FraxlendPairCore.sol:313; fraxlend/src/contracts/FraxlendPairCore.sol:334; fraxlend/src/contracts/FraxlendPairCore.sol:386

**Summary/Description**  
LendingAssetVault previewDeposit, previewMint, previewWithdraw, previewRedeem, maxWithdraw, and maxRedeem price from _previewCbr(), which simulates whitelisted-vault CBR changes through previewAddInterest. The matching mutable functions first call _updateInterestAndMdInAllVaults(), but that path refreshes stored _totalAssets only when IFraxlendPair.addInterest(false) returns positive interest. If the pair CBR has changed but addInterest(false) skips or returns zero, the ERC4626 view answers use a fresh denominator while deposit, mint, withdraw, and redeem price from stale _totalAssets.

**Root Cause**  
The preview and mutable ERC4626 paths do not share the same accounting source. _previewAddInterestAndMdInAllVaults always compares stored vault CBR against the pair preview/current CBR, while _updateInterestAndMdInAllVaults gates _updateAssetMetadataFromVault on positive interestEarned. Fraxlend external addInterest can skip through minURChangeForExternalAddInterest, and same-block already-updated interest can leave the LAV refresh with zero returned interest even when preview views observe a different denominator.

**Pre_conditions**  
A whitelisted Fraxlend pair has nonzero LAV utilization and its fToken CBR differs from _vaultWhitelistCbr, but a LAV state-changing call reaches _updateInterestAndMdInAllVaults when addInterest(false) returns zero or skips. This can happen after interest is previewable but external addInterest is below the utilization-change threshold, or after interest was already checkpointed in the pair without LAV metadata being refreshed.

**Impact**  
ERC4626 preview bounds can be wrong in the same transaction, and maxWithdraw/maxRedeem can report amounts that the matching withdrawal path would not accept. With uncheckpointed CBR growth, maxWithdraw can return more assets than an owner's shares can withdraw under the stale mutable CBR, so withdraw(maxWithdraw(owner), ...) can revert. Integrations relying on ERC4626 max/preview bounds can misprice slippage, allowances, or queue actions that revert. The standalone severity is Info because the direct value-transfer stale-CBR paths are separately covered by M-17 and M-19.

**Proof of Concept**  
Positive-yield maxWithdraw example: LAV has cached _totalAssets 100, totalSupply 100, enough liquid assets, and an owner has 10 shares. A whitelisted pair has accrued yield so _previewCbr() reflects 2 assets/share, but addInterest(false) skips and mutable pricing remains 1 asset/share. maxWithdraw(owner) returns 20 assets, while withdraw(20, ..., owner) computes 20 shares from stale _cbr() and reverts because the owner has only 10.

**Mitigation**  
Use the same CBR source for previews, max functions, and mutable pricing. Before pricing any LAV share change, refresh vault metadata whenever the current or preview vault CBR differs from _vaultWhitelistCbr, not only when addInterest(false) returns positive interest. Alternatively make previews and max functions intentionally mirror the exact cached denominator that mutable calls will use, though that would preserve the stale-value issues tracked separately. Use safe zero-CBR handling so maxRedeem returns a conservative value instead of reverting.

### [I-15] Fraxlend finite deposit cap is unreachable in the current fork

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fraxlend/src/contracts/FraxlendPairAccessControl.sol:46; fraxlend/src/contracts/FraxlendPair.sol:489; fraxlend/src/contracts/FraxlendPair.sol:503; fraxlend/src/contracts/FraxlendPair.sol:551; fraxlend/src/contracts/FraxlendPair.sol:558; fraxlend/src/contracts/FraxlendPairCore.sol:633; fraxlend/src/contracts/FraxlendPairCore.sol:705

**Summary/Description**  
FraxlendPair still contains depositLimit accounting and deposit/mint cap checks, but the external setDepositLimit, pauseDeposit, and revokeDepositLimitAccessControl functions are commented out in this fork. The only live writes to depositLimit are pause(), which sets it to 0 while also disabling borrow and withdraw paths, and unpause(), which restores type(uint256).max. Because no live external path can set a finite nonzero cap, the previously modeled LAV pull path cannot bypass a configured supply cap in the current code.

**Root Cause**  
The finite deposit-cap control surface is left as storage/internal helpers plus deposit/mint checks, while the external setter/revoke/pause-deposit functions that would make a finite cap reachable are disabled. Internal LAV liquidity pulls do not check depositLimit, but the finite-cap state needed for that to be an exploitable cap bypass is not reachable through the current contract ABI.

**Pre_conditions**  
A finite nonzero depositLimit would need to be configured for a meaningful supply-cap bypass. In this fork, that precondition is not reachable through live external functions; depositLimit starts at type(uint256).max and only global pause/unpause change it to 0/max.

**Impact**  
No practical Medium-severity supply-cap bypass was confirmed for current deployed code. If the protocol expected per-pair finite supply caps, operators cannot enforce them with this ABI; they can only globally pause deposits together with other controls or leave deposits uncapped.

**Proof of Concept**  
Reachability check: rg shows _setDepositLimit is only called by pause() and unpause(). pause() also sets borrowLimit to 0 and pauses withdraw/repay/liquidation/interest where not revoked, so the internal _depositFromVault paths that could mint LAV fTokens without a cap check are not usable as a finite-cap bypass under pause. The setDepositLimit, pauseDeposit, and revokeDepositLimitAccessControl functions are present only as comments.

**Mitigation**  
If finite per-pair supply caps are intended, restore a timelock/owner-gated setDepositLimit or equivalent and apply the same cap semantics to all supply-increasing paths, including _depositFromVault. If finite caps are intentionally unsupported, remove or rename the dead cap surface to avoid treating depositLimit as an enforceable risk limit.

### [I-16] FraxlendPair accepts zero-share deposits in inflated CBR states

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fraxlend/src/contracts/libraries/VaultAccount.sol:25; fraxlend/src/contracts/FraxlendPairCore.sol:578; fraxlend/src/contracts/FraxlendPairCore.sol:624; fraxlend/src/contracts/FraxlendPairCore.sol:646

**Summary/Description**  
FraxlendPair deposit paths compute fToken shares with floor rounding and do not reject a zero-share result. Direct ERC20 balance donations do not inflate the pair because deposits price from totalAsset.amount rather than token balance, but an already inflated accounted CBR can make a later deposit or LAV pull add assets while minting zero fTokens.

**Root Cause**  
VaultAccount.toShares floors amount * shares / total.amount, and FraxlendPairCore._deposit mints the returned share amount without requiring it to be nonzero. The public deposit path and internal _depositFromVault path both use this helper directly.

**Pre_conditions**  
totalAsset.amount is nonzero and larger than depositAmount * totalAsset.shares, which can occur only in an already high raw-CBR state from accrued interest, previous zero-share additions, or unusual low-decimal/extreme-rate conditions. I did not confirm a cheap raw-donation path because direct token transfers are not included in totalAsset.amount.

**Impact**  
The deposited assets become part of totalAsset.amount without creating ownership shares for the depositor, so value is allocated to existing fToken holders or distorts LAV utilization accounting. For normal 18-decimal markets with ordinary yield this is dust-bounded because the zero-share threshold is the raw price of one smallest fToken unit; it becomes material only if that raw CBR is made large.

**Proof of Concept**  
Assume totalAsset.amount = 1001 and totalAsset.shares = 1. A call to deposit(1000, receiver) computes shares = floor(1000 * 1 / 1001) = 0. _deposit then increments totalAsset.amount by 1000, mints 0 fTokens, and transfers 1000 assets from the caller. The same zero-share result is possible in _depositFromVault before calling _deposit(..., address(externalAssetVault), false).

**Mitigation**  
Reject zero-share mints in all asset-supplying paths, including _deposit and _depositFromVault, or require caller-specified minimum shares for integrations. If high-CBR states are expected, add virtual share/asset offsets or permanently locked seed liquidity so the raw one-share-unit value cannot become material.

### [I-17] maxLTV zero disables Fraxlend solvency checks

**Severity**: Info  
**Likelihood**: Low  
**Impact**: High  
**Location**: fraxlend/src/contracts/FraxlendPairCore.sol:181; fraxlend/src/contracts/FraxlendPairCore.sol:225; fraxlend/src/contracts/FraxlendPairCore.sol:226; fraxlend/src/contracts/FraxlendPair.sol:337; fraxlend/src/contracts/FraxlendPair.sol:341

**Summary/Description**  
FraxlendPairCore._isSolvent returns true immediately when maxLTV == 0. A pair configured with maxLTV set to zero therefore treats every borrower as solvent, regardless of debt, collateral amount, or oracle price, and borrowAsset will only be bounded by available liquidity and borrowLimit.

**Root Cause**  
The code treats maxLTV == 0 as a sentinel that disables solvency enforcement, but the constructor and timelock setter accept zero without a guard or explicit separate disable flag.

**Pre_conditions**  
A pair is deployed with _maxLTV = 0 or the timelock calls setMaxLTV(0). This is a trusted-configuration path for normal protocol pairs, so it is classified as an Info-level footgun rather than an untrusted primary exploit.

**Impact**  
If reached on a live lending pair with available assets, borrowers can open or increase debt with no effective collateralization requirement. Lenders or LAV allocators can be left with bad debt once the borrowed assets leave the pair.

**Proof of Concept**  
Code path: borrowAsset executes _addInterest, _updateExchangeRate, optional _addCollateral, and _borrowAsset. The post-body isSolvent modifier then calls _isSolvent(msg.sender, highExchangeRate). With maxLTV == 0, _isSolvent returns true before reading userBorrowShares, userCollateralBalance, or the exchange rate, so the transaction does not revert for insolvency.

**Mitigation**  
Reject maxLTV == 0 in constructor and setMaxLTV unless a fully explicit no-collateral mode is intended and isolated from lender-funded pairs. If zero is meant to pause borrowing, set borrowLimit to zero instead of bypassing solvency.

### [I-18] Global pause does not block collateral removal

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: fraxlend/src/contracts/FraxlendPair.sol:489; fraxlend/src/contracts/FraxlendPair.sol:492; fraxlend/src/contracts/FraxlendPairCore.sol:993; fraxlend/src/contracts/FraxlendPairCore.sol:975; fraxlend/src/contracts/FraxlendPairCore.sol:1000; fraxlend/src/contracts/FraxlendPairCore.sol:1062; fraxlend/src/contracts/FraxlendPairCore.sol:1107

**Summary/Description**  
FraxlendPair.pause() is documented as pausing all contract functionality and it disables borrow, deposit, repay, asset withdrawals, liquidation, and interest accrual. It does not set any collateral-removal pause state, and removeCollateral() remains externally callable during the paused period.

**Root Cause**  
The pause model has flags for asset repayment, asset withdrawal, liquidation, interest, and borrow/deposit limits, but no flag is checked by removeCollateral(). The collateral path only relies on post-state solvency and oracle deviation checks.

**Pre_conditions**  
A pair is globally paused and a borrower has collateral above the amount needed to remain solvent under the current cached or refreshed exchange rate.

**Impact**  
During an emergency pause, an untrusted borrower can still move excess collateral out of the pair while repay and liquidation are disabled and interest is frozen. This weakens the emergency freeze property and can reduce lender collateral buffers if the pause was meant to contain a collateral, oracle, or liquidation incident. The issue is Info because the function still enforces solvency after the removal and does not by itself create deterministic bad debt.

**Proof of Concept**  
pause() sets borrowLimit to 0, depositLimit to 0, isRepayPaused, isWithdrawPaused, isLiquidatePaused, and isInterestPaused. A borrower then calls removeCollateral(amount, receiver). removeCollateral has no pause check, calls _addInterest(), refreshes the exchange rate if the caller has debt, and calls _removeCollateral(), which decreases userCollateralBalance and transfers collateral to receiver. The call only reverts if the borrower is insolvent after the removal.

**Mitigation**  
If global pause is intended to freeze all economically sensitive exits, add a collateral pause flag or have removeCollateral() check isWithdrawPaused/a dedicated isCollateralPaused flag. If collateral exits should remain available, update the pause documentation and incident runbooks to state that only asset withdrawals are paused.

### [I-19] setRateContract applies new rate model to elapsed interest

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fraxlend/src/contracts/FraxlendPair.sol:363; fraxlend/src/contracts/FraxlendPair.sol:367; fraxlend/src/contracts/FraxlendPairCore.sol:380; fraxlend/src/contracts/FraxlendPairCore.sol:405; fraxlend/src/contracts/FraxlendPairCore.sol:410; fraxlend/src/contracts/FraxlendPairCore.sol:481

**Summary/Description**  
FraxlendPair.setRateContract replaces the interest rate calculator without first checkpointing accrued interest or resetting the adaptive rate metadata. The next internal _addInterest call computes the full elapsed interval since currentRateInfo.lastTimestamp with the newly configured rateContract and passes the old currentRateInfo.fullUtilizationRate into that new model, so a normal rate-model migration can retroactively apply the new model to time and full-utilization state that belonged to the old model.

**Root Cause**  
The rate-contract setter writes rateContract directly, while _calculateInterest later reads rateContract, uses block.timestamp - currentRateInfo.lastTimestamp as the elapsed interval, and forwards currentRateInfo.fullUtilizationRate to the selected calculator. Unlike changeFee(), the setter does not call _addInterest() before the parameter takes effect and does not provide a migration value for the new calculator's full-utilization state.

**Pre_conditions**  
A pair has outstanding borrow debt, time has elapsed since the last internal interest update, and the timelock calls setRateContract() to a valid calculator that returns a materially different rate or interprets fullUtilizationRate differently for the current utilization inputs. This is a trusted-timelock transition path, so it is classified as Info rather than an untrusted exploit.

**Impact**  
Borrowers can be overcharged for the elapsed interval if the new model returns a higher rate, or lenders/protocol can under-accrue if it returns a lower rate. The effect applies to all outstanding borrow amount for the uncheckpointed interval, but it requires a trusted parameter migration.

**Proof of Concept**  
At T0, currentRateInfo.lastTimestamp and currentRateInfo.fullUtilizationRate are maintained under the old rate model and totalBorrow.amount is nonzero. Time passes without an internal _addInterest checkpoint. At T1, the timelock calls setRateContract(newRateContract), which only assigns rateContract. The next deposit/borrow/repay/withdraw/liquidation path calls _addInterest(); _calculateInterest uses deltaTime = T1 - T0, calls IRateCalculatorV2(rateContract).getNewRate(deltaTime, utilization, oldFullUtilizationRate) on the new contract, and charges interestEarned = deltaTime * totalBorrow.amount * newRate / RATE_PRECISION. The old contract is never used for the elapsed T0..T1 interval, and the new contract receives the old model's full-utilization value as its starting state.

**Mitigation**  
In setRateContract(), force an _addInterest() checkpoint under the old calculator before writing the new rateContract, matching the changeFee() transition pattern. Also validate that the new calculator address is nonzero and supports the expected interface/version, and either require an explicit migrated fullUtilizationRate or clamp/reset it according to the new calculator's validated bounds.

### [I-20] Protocol liquidation fee is based on seized collateral instead of liquidation bonus

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: fraxlend/src/contracts/FraxlendPairCore.sol:76; fraxlend/src/contracts/FraxlendPairCore.sol:1153; fraxlend/src/contracts/FraxlendPairCore.sol:1154; fraxlend/src/contracts/FraxlendPairCore.sol:1155; fraxlend/src/contracts/FraxlendPairCore.sol:1201; fraxlend/src/contracts/FraxlendPairCore.sol:1203; fraxlend/src/contracts/FraxlendPair.sol:466; fraxlend/src/contracts/FraxlendPair.sol:467

**Summary/Description**  
FraxlendPair documents protocolLiquidationFee as the portion of the liquidation fee given to the protocol, but liquidate() calculates the protocol amount as a percentage of the full collateral seized for the liquidation. The same 1e5-precision parameter is not bounded during pair creation, so values above LIQ_PRECISION make the fee exceed seized collateral and cause liquidation to revert during subtraction.

**Root Cause**  
The liquidation code uses _collateralForLiquidator, the gross seized collateral amount, as the protocol-fee base instead of using only the liquidation bonus portion above the repaid debt value.

**Pre_conditions**  
protocolLiquidationFee is nonzero and a liquidation executes. No unusual token behavior is required. A stronger liveness impact requires the configured protocol share to be high enough to consume the liquidator incentive, which is a trusted deployment/configuration condition.

**Impact**  
The protocol receives more collateral than a fee-on-bonus interpretation implies, and the liquidator receives less. With the default-style 1-2% protocol fee seen in local scripts/tests, liquidation remains profitable when the normal clean or dirty bonus is available. If protocolLiquidationFee is configured above 100000, _feesAmount exceeds _collateralForLiquidator and the checked subtraction reverts, blocking liquidations for the pair until configuration is changed. This remains Info because the path is trusted deployment/configuration controlled; H-03 and H-04 cover the stronger bad-debt liquidation impacts.

**Proof of Concept**  
For a dirty liquidation repaying 100 collateral-value units with dirtyLiquidationFee = 9% and protocolLiquidationFee = 2%, the gross seizure is 109. The code charges 2% of 109 = 2.18 to the protocol and sends 106.82 to the liquidator. If protocolLiquidationFee is a portion of only the 9-unit liquidation bonus, the protocol fee would be 0.18 and the liquidator would receive 108.82. For the unbounded precision case, setting protocolLiquidationFee = 100001 makes _feesAmount = 100001 * grossCollateral / 100000, which is greater than the gross seized collateral and underflows at _collateralForLiquidator - _feesAmount.

**Mitigation**  
If the intended design is a protocol share of the liquidation bonus, compute the protocol fee from gross seized collateral minus the repaid debt value in collateral units, clamped to zero. Validate protocolLiquidationFee during pair creation and fee-setting so it cannot exceed LIQ_PRECISION or the intended share of the liquidation bonus.

### [I-21] Override position initialization accepts incompatible lending pairs

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/lvf/LeverageManager.sol:241; contracts/contracts/lvf/LeverageManager.sol:247; contracts/contracts/lvf/LeverageManager.sol:252; contracts/contracts/lvf/LeverageManager.sol:580; contracts/contracts/lvf/LeverageManager.sol:594; contracts/contracts/lvf/LeverageFactory.sol:142

**Summary/Description**  
For pods without a registered lendingPairs entry, initializePosition only requires a nonzero override lending pair and then stores it as the position pair. The override path does not validate that the pair is the pod paired token for self-lending, that the pair collateral is the aspTKN for the same pod, or that the pair asset/flash-source relationship is compatible. Later add/remove flows infer self-lending from pod.PAIRED_LP_TOKEN() != pair.asset() and fail or route through zap/deposit paths based on the unchecked pair.

**Root Cause**  
The position initializer treats the absence of a registered lending pair as permission to trust a caller-supplied pair, while the compatibility invariants are only implicit in later LeverageManager, aspTKN, and Fraxlend calls.

**Pre_conditions**  
A user initializes a position for a pod whose lendingPairs[pod] is unset and supplies an override pair that is not the intended self-lending pair, or a compatible but unregistered pair exists for the same pod collateral.

**Impact**  
Most incompatible overrides revert during flash-source lookup, ERC4626 collateral deposit, or pair borrow/remove calls, leaving an unusable user-created position. If a compatible but unregistered pair exists, the override path can bypass the owner-set lendingPairs mapping that normally turns on LVF for a pod. I did not confirm an untrusted path in the scoped factories to create a malicious compatible pair for an existing official aspTKN, so this is classified as Info.

**Mitigation**  
Validate override pairs during initialization. For self-lending, require address(IFraxlendPair(_overrideLendingPair)) == IDecentralizedIndex(_pod).PAIRED_LP_TOKEN() or explicitly validate the supported podded-pair variant. Also require IFraxlendPair(_overrideLendingPair).collateralContract() to be the expected aspTKN for the pod and require a configured flash source for IFraxlendPair(_overrideLendingPair).asset().

### [I-22] Open fee is charged against gross paired amount and reduces net LP input

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/lvf/LeverageManager.sol:74; contracts/contracts/lvf/LeverageManager.sol:302; contracts/contracts/lvf/LeverageManager.sol:315; contracts/contracts/lvf/LeverageManager.sol:318; contracts/contracts/lvf/LeverageManager.sol:322; contracts/contracts/lvf/LeverageManager.sol:328

**Summary/Description**  
LeverageManager computes the open fee from _props.pairedLpDesired and transfers that fee to feeReceiver before LP creation. The remaining borrow token amount is then passed to _lpAndStakeInPod, so the amount actually used for LP is pairedLpDesired minus the open fee. This is not a second open-fee transfer, but it means _pairedLpDesired is treated as a gross amount even though NatSpec describes it as the total paired token amount for the pod to use to add LP.

**Root Cause**  
The open-fee calculation is applied directly to the LP input amount stored in pairedLpDesired instead of separating gross borrow/funding amount, fee amount, and net LP amount.

**Pre_conditions**  
openFeePerc is nonzero and a user opens or increases leverage through addLeverage or addLeverageFromTkn.

**Impact**  
Users receive a smaller LP/collateral position than the _pairedLpDesired wording implies. If _userProvidedDebtAmt is nonzero, the fee base also includes the user-provided portion, not only the flash-borrowed portion. I did not confirm a duplicate fee transfer or a direct fund-drain path, so this is classified as Info.

**Proof of Concept**  
Static flow: _addLeveragePreCallback flash-borrows _pairedLpDesired - _userProvidedDebtAmt, while _addLeveragePostCallback sets _borrowTknAmtToLp = _props.pairedLpDesired, transfers (_borrowTknAmtToLp * openFeePerc) / 1000 to feeReceiver, subtracts it, and passes only the reduced amount to _lpAndStakeInPod. The later borrow amount is max(overrideBorrowAmt, _d.amount + _d.fee), so the open fee is not transferred twice.

**Mitigation**  
Clarify whether _pairedLpDesired is a gross funding amount or the desired net LP amount. If it should be net LP input, calculate and fund the open fee separately; if it should be gross, update parameter names/docs and front-end quoting to show net LP amount and the fee base explicitly.

### [I-23] addLeverage pTKN refunds go to position owner instead of funder

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/lvf/LeverageManager.sol:90; contracts/contracts/lvf/LeverageManager.sol:126; contracts/contracts/lvf/LeverageManager.sol:220; contracts/contracts/lvf/LeverageManager.sol:306; contracts/contracts/lvf/LeverageManager.sol:609

**Summary/Description**  
LeverageManager pulls pTKN, or underlying TKN that is bonded into pTKN, from msg.sender when addLeverage/addLeverageFromTkn is called. For an existing position, approved ERC721 operators may call these functions for a position owned by someone else. The callback then refunds unused pTKN to LeverageFlashProps.owner, which is the position NFT owner, rather than to LeverageFlashProps.sender, which is the address that supplied the input tokens.

**Root Cause**  
The add-leverage flash data records both the current position owner and the transaction sender, but callback() uses _posProps.owner for the unused-pTKN refund. This conflates the position beneficiary with the funding address.

**Pre_conditions**  
An existing position owner has approved another address for the position NFT, or setApprovalForAll for it. The approved operator calls addLeverage or addLeverageFromTkn using tokens held by the operator, and the LP add path uses less pTKN than the amount supplied so a pTKN refund exists.

**Impact**  
The approved operator can lose the unused pTKN refund to the position owner. The loss is bounded by the difference between the supplied pTKN amount and the pTKN actually consumed by LP creation. I classify this as Info because the caller must already be authorized for the position and voluntarily fund the transaction; no third party can force the call, and approved operators can already transfer the position NFT.

**Proof of Concept**  
Static path: addLeverage transfers _pTknAmt from _msgSender() to LeverageManager, and addLeverageFromTkn transfers TKN from _msgSender() before bonding it into pTKN. _getFlashDataAddLeverage stores sender: _sender and owner: positionNFT.ownerOf(_positionId). In callback(), the ADD branch calls _addLeveragePostCallback and transfers _ptknRefundAmt to _posProps.owner. If sender != owner, the leftover pTKN funded by sender is paid to owner.

**Mitigation**  
Refund unused pTKN to the token funder, i.e. _posProps.sender, or explicitly separate beneficiary and refund-recipient parameters. Keep collateral/position accounting tied to the NFT owner, but return unspent caller-provided inputs to the caller unless the API intentionally opts into donating leftovers to the position owner.

### [I-24] Floor-rounded self-lending share target can underfill repayment

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/lvf/LeverageManager.sol:435; contracts/contracts/lvf/LeverageManager.sol:439; contracts/contracts/lvf/LeverageManager.sol:442; contracts/contracts/lvf/LeverageManager.sol:465; fraxlend/src/contracts/FraxlendPair.sol:225

**Summary/Description**  
When a self-lending removeLeverage flow must sell returned pTKN to cover a borrow-token shortfall, LeverageManager converts the asset shortfall into Fraxlend pair shares with convertToShares(). Fraxlend convertToShares() floor-rounds. The floor-rounded share amount is then used as the exact-output swap target and redeemed, so the redeemed borrow-token amount can be lower than the original shortfall.

**Root Cause**  
The repayment helper needs enough Fraxlend asset shares to redeem at least _borrowAmtNeededToSwap assets, but it uses the ERC4626-style floor-rounded convertToShares() view instead of a round-up asset-share conversion or an explicit plus-one buffer.

**Pre_conditions**  
A self-lending position is being unwound, the paired assets received from LP removal are less than the flash repayment amount, and the Fraxlend pair asset/share ratio is not an exact divisor for the remaining asset shortfall.

**Impact**  
The unwind can revert at the final flash repayment transfer because the pTKN swap acquired too few Fraxlend shares and redeeming those shares leaves LeverageManager short by rounding dust. Users can often work around it by providing extra borrow token or over-targeting the share output, so this is classified as Info.

**Mitigation**  
Use a round-up conversion for the self-lending shortfall target, e.g. expose/call toAssetShares(amount, true, true), previewWithdraw if compatible, or add a bounded one-share buffer and refund any excess after redemption.

### [I-25] removeLeverage wallet top-up pulls from zero address

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/lvf/LeverageManager.sol:177; contracts/contracts/lvf/LeverageManager.sol:181; contracts/contracts/lvf/LeverageManager.sol:187; contracts/contracts/lvf/LeverageManager.sol:425; contracts/contracts/lvf/LeverageManager.sol:429

**Summary/Description**  
removeLeverage exposes _userProvidedDebtAmtMax so the caller can contribute borrow tokens when the LP unwind does not return enough paired/borrow token to repay the flash loan. The REMOVE flash data, however, only sets method, positionId, and owner. It never sets LeverageFlashProps.sender. In the shortfall helper, _acquireBorrowTokenForRepayment tries to pull the user contribution from _props.sender, which is the default zero address for REMOVE callbacks.

**Root Cause**  
The ADD path uses _getFlashDataAddLeverage to populate both owner and sender, but the REMOVE path manually constructs LeverageFlashProps and omits sender even though _acquireBorrowTokenForRepayment relies on it for _userProvidedDebtAmtMax.

**Pre_conditions**  
A removeLeverage call reaches the shortfall branch where paired tokens from LP removal are less than the flash repayment amount, and the caller sets _userProvidedDebtAmtMax greater than zero to cover all or part of that shortfall from their wallet.

**Impact**  
For standard ERC20 borrow tokens, safeTransferFrom(address(0), address(this), amount) reverts, so the wallet top-up fallback is unavailable. Users must either set the max contribution to zero and rely on the pTKN swap path, choose different remove parameters, repay the Fraxlend debt out of band, or fail to deleverage through this advertised route.

**Proof of Concept**  
Static path: removeLeverage stores _sender locally for access checks and flash-fee payment, but the encoded LeverageFlashProps for REMOVE does not assign sender. Later _removeLeveragePostCallback decodes those props and calls _acquireBorrowTokenForRepayment if _pairedAmtReceived < _d.amount + _d.fee. With _userProvidedDebtAmtMax > 0, line 429 executes IERC20(_borrowToken).safeTransferFrom(_props.sender, address(this), _borrowAmtFromUser); since _props.sender is address(0), the call reverts for normal ERC20 tokens.

**Mitigation**  
Populate _props.sender = _sender in removeLeverage before encoding the flash data, or pass the caller/funding address explicitly through the remove additional info. Keep owner as the position beneficiary, but use the transaction sender or an explicit payer for wallet-funded repayment.

### [I-26] Self-lending factory registers pods with non-self-lending metadata

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: contracts/contracts/lvf/LeverageFactory.sol:90; contracts/contracts/lvf/LeverageFactory.sol:107; contracts/contracts/lvf/LeverageFactory.sol:259; contracts/contracts/IndexManager.sol:37; contracts/contracts/IndexManager.sol:101; contracts/contracts/IndexManager.sol:112; contracts/contracts/interfaces/IIndexManager.sol:11

**Summary/Description**  
LeverageFactory.createSelfLendingPodAndAddLvf deploys a new self-lending pod through IndexManager.deployNewIndex(), but deployNewIndex always registers the new index with creator = msg.sender, verified = false, selfLending = false, and makePublic = false. Because msg.sender is LeverageFactory, the external user who initiated self-lending setup is not recorded as creator, and the new self-lending pod is left in the registry as non-self-lending unless an authorized IndexManager role later corrects it.

**Root Cause**  
The self-lending composed setup does not use the IndexManager.addIndex path that accepts explicit creator/selfLending/makePublic metadata, and it does not call a post-deploy IndexManager update after the pod address is known. The generic deployNewIndex helper hardcodes non-self-lending metadata for all deployed pods.

**Pre_conditions**  
A self-lending pod is created through LeverageFactory.createSelfLendingPodAndAddLvf. IndexManager.deployNewIndex is the path used to deploy and register the pod.

**Impact**  
On-chain leverage can still be used when the user supplies the returned or discoverable Fraxlend pair as the override lending pair, and LeverageManager dynamically detects self-lending from pod.PAIRED_LP_TOKEN() versus pair.asset(). The confirmed impact is registry and module state inconsistency: allIndexes() reports the self-lending pod as non-self-lending, the initiating user cannot use creator-only updateMakePublic/updateSelfLending permissions, and downstream registry consumers can classify or hide the pod incorrectly until a trusted authorized role manually fixes it. No direct fund loss path was confirmed.

**Proof of Concept**  
createSelfLendingPodAndAddLvf calls _createSelfLendingPod(), which calls IIndexManager(indexManager).deployNewIndex(...). IndexManager.deployNewIndex then executes _addIndex(_index, _msgSender(), false, false, false). In this call tree _msgSender() is LeverageFactory, not the external setup caller, and the fourth argument is hardcoded false even though the pod's PAIRED_LP_TOKEN was just set to the new Fraxlend pair for self-lending. IndexManager.updateSelfLending can correct the flag only for the owner, authorized roles, or the recorded creator; the recorded creator is LeverageFactory and the factory exposes no forwarding call.

**Mitigation**  
Add a self-lending-aware registration path. For example, extend deployNewIndex to accept creator/selfLending/makePublic metadata, or have LeverageFactory call explicit IndexManager update functions immediately after deployment to mark the pod self-lending and set the intended creator or public state. Keep this registry update atomic with the rest of createSelfLendingPodAndAddLvf.

### [I-27] Fraxlend deployment config accepts dirty-fee field as protocol fee

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/lvf/LeverageFactory.sol:172; contracts/contracts/lvf/LeverageFactory.sol:185; contracts/contracts/lvf/LeverageFactory.sol:186; fraxlend/src/contracts/FraxlendPairConstants.sol:38; fraxlend/src/contracts/FraxlendPairDeployer.sol:244; fraxlend/src/contracts/FraxlendPairDeployer.sol:289; fraxlend/src/contracts/FraxlendPairDeployer.sol:297; fraxlend/src/contracts/FraxlendPairCore.sol:142; fraxlend/src/contracts/FraxlendPairCore.sol:160; fraxlend/src/contracts/FraxlendPairCore.sol:178

**Summary/Description**  
Fraxlend deployment comments describe a ten-field config containing cleanLiquidationFee, dirtyLiquidationFee, and protocolLiquidationFee, but the current deployer and pair constructor decode only nine final fields and derive dirtyLiquidationFee from cleanLiquidationFee. LeverageFactory also decodes only a six-field suffix. Because Solidity ABI decoding of static tuples accepts trailing words, a plausible stale deployment payload that still includes dirtyLiquidationFee is not rejected: the dirty-fee word is decoded as protocolLiquidationFee and the intended protocol-fee word is ignored.

**Root Cause**  
The deployment config is an untyped bytes blob with stale NatSpec/schema references and no exact-length check. FraxlendPairDeployer.deploy and FraxlendPairCore decode nine fields, while comments still advertise ten; LeverageFactory decodes a six-field suffix with no length/schema enforcement.

**Pre_conditions**  
An operator or integration encodes direct Fraxlend _configData using the documented ten-field order, or encodes LeverageFactory's suffix using the old dirtyLiquidationFee slot before protocolLiquidationFee. This is most plausible when adapting older Fraxlend deployment tooling or comments.

**Impact**  
The deployed pair can have an unintended protocolLiquidationFee, commonly equal to the intended dirtyLiquidationFee, while the intended protocol fee is ignored. This can change liquidation incentives and make deployment troubleshooting difficult. The issue remains Info because the caller controls pair configuration and no untrusted path was found to override the factory-inserted asset, collateral, or oracle addresses.

**Proof of Concept**  
Direct Fraxlend example: abi.encode(asset, collateral, oracle, uint32(5000), rateContract, uint64(1000), uint256(75000), uint256(10000), uint256(9000), uint256(2000)) is accepted by the nine-field decode; protocolLiquidationFee becomes 9000 and the trailing intended 2000 is ignored. LeverageFactory has the same shift for a seven-field suffix: the dirty-fee word is decoded as protocolLiquidationFee before the final nine-field config is re-encoded.

**Mitigation**  
Document a single current deployment schema and require exact encoded lengths before decoding: 9 * 32 for direct Fraxlend config and 6 * 32 for the LVF suffix. Prefer named structs or typed function parameters over raw bytes for new deployment wrappers, and remove stale dirtyLiquidationFee config references.

### [I-28] TokenBridge accepts zero-receiver messages that cannot be executed

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: contracts/contracts/ccip/TokenBridge.sol:24; contracts/contracts/ccip/TokenBridge.sol:72; contracts/contracts/ccip/TokenBridge.sol:92; contracts/contracts/ccip/TokenBridge.sol:109; contracts/contracts/ERC20Bridgeable.sol:41

**Summary/Description**  
TokenBridge.bridgeTokens accepts _tokenReceiver without checking for address(0), then encodes that receiver into the CCIP payload. On the destination chain _ccipReceive forwards the encoded receiver into _processOutboundTokens, which either mints to that address or safeTransfers unlocked tokens. For standard ERC20Bridgeable/OpenZeppelin ERC20 behavior, minting or transferring to address(0) reverts, so the CCIP message moves to failed/manual-execution state after the source-side lock or burn has already been accepted.

**Root Cause**  
The bridge validates route enablement but not the destination receiver, and it has no app-level failed-message recovery path that can replace an invalid receiver or refund the source-side lock/burn.

**Pre_conditions**  
A user calls bridgeTokens on an enabled route with _tokenReceiver = address(0), supplies sufficient fee, and CCIP accepts the source message.

**Impact**  
The user cannot receive the bridged amount. In lock/unlock mode the source tokens remain stranded in the source TokenBridge with no refund function; in mint/burn mode the source tokens are burned and the destination mint cannot complete. The sender loses fees and needs out-of-band operator compensation or token/admin intervention. This is Info because the trigger is an avoidable user input mistake and no third-party extraction path was found.

**Proof of Concept**  
Static flow: bridgeTokens(..., address(0), token, amount) pulls/locks or burns amount, builds TokenTransfer({tokenReceiver: address(0), ...}), and calls ccipSend. On delivery, _ccipReceive decodes the zero receiver and calls _processOutboundTokens. ERC20Bridgeable.mint(address(0), amount) reverts through OpenZeppelin _mint, and lock/unlock safeTransfer(address(0), amount) reverts through ERC20 _transfer, so CCIP records failure and manual retry repeats the same zero receiver.

**Mitigation**  
Reject zero receivers in bridgeTokens before _processInboundTokens, for example require(_tokenReceiver != address(0), "RECEIVER"). If supporting broader delivery failures, add an app-level failed-message recovery flow that can refund or redirect only after authenticating the original message and preventing double release.

### [I-29] ERC20Bridgeable minters can burn arbitrary holder balances

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: contracts/contracts/ERC20Bridgeable.sol:20; contracts/contracts/ERC20Bridgeable.sol:36; contracts/contracts/ccip/TokenBridge.sol:96

**Summary/Description**  
ERC20Bridgeable uses the minter mapping as the only authorization for burnFrom, so any address granted minter status can burn tokens from any holder without that holder approving or initiating the burn. The scoped TokenBridge integration does not need this authority for its normal mint-burn path because it pulls tokens from the caller and then self-burns its own balance.

**Root Cause**  
burnFrom checks only minter[msg.sender] and then calls _burn(_wallet, _amount). It does not require _wallet == msg.sender, does not spend ERC20 allowance, and does not bind the burn to a bridge message or prior transfer into the bridge.

**Pre_conditions**  
The token owner grants minter status to an address or integration that is expected to have limited bridge/mint authority rather than full balance-destruction authority.

**Impact**  
A granted minter can destroy arbitrary holder balances up to each holder's balance. This is recorded as Info because no untrusted path to become a minter was found: setMinter is onlyOwner, mint is onlyMinter, and the in-scope TokenBridge receive path is router and remote-bridge authenticated before minting.

**Proof of Concept**  
Static path: setMinter(minter, true) makes minter[minter] true. That address can call burnFrom(victim, amount), onlyMinter passes, and _burn(victim, amount) reduces the victim balance without checking allowance or victim consent. The normal TokenBridge source path instead calls safeTransferFrom(user, this, amount) and burn(amount), so it does not exercise burnFrom.

**Mitigation**  
Separate mint and burn roles, or make burnFrom spend allowance like ERC20Burnable unless the caller is a narrowly bound bridge contract. If only TokenBridge should support mint-burn bridging, remove burnFrom or restrict it to a specific bridge and bridge-validated flow.

### [I-30] Dust top-ups can shorten existing VotingPool lockups after a lockup reduction

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/voting/VotingPool.sol:38; contracts/contracts/voting/VotingPool.sol:41; contracts/contracts/voting/VotingPool.sol:42; contracts/contracts/voting/VotingPool.sol:47; contracts/contracts/voting/VotingPool.sol:50; contracts/contracts/voting/VotingPool.sol:101

**Summary/Description**  
VotingPool snapshots the current global lockupPeriod into a user/asset Stake, but every later stake top-up overwrites that snapshot and lastStaked for the entire existing position. If the owner reasonably lowers the global lockup while older positions are still locked, a user can stake a dust amount and make the shorter new period apply to their whole prior balance.

**Root Cause**  
The contract stores only one Stake record per user and asset. stake() assigns stakes[user][asset].lastStaked = block.timestamp and stakes[user][asset].lockupPeriod = lockupPeriod before adding the new amount, so it cannot preserve the original cooldown for the pre-existing amount.

**Pre_conditions**  
A user has an existing VotingPool stake whose original lockup has not expired. The trusted owner later lowers lockupPeriod, for example to change future staking terms. The user can transfer at least a dust amount of the same enabled asset into VotingPool.

**Impact**  
The user can exit the old stake earlier than the original per-stake lockup would allow by applying the new shorter period to the entire user/asset balance. This weakens the lockup commitment and can reduce the capital-time cost of already-reported VotingPool reward-timing behavior, but by itself it does not transfer funds from other users and depends on a trusted configuration change.

**Proof of Concept**  
Example: lockupPeriod is 7 days and Alice stakes 100 tokens at time T, so her Stake stores lastStaked = T and lockupPeriod = 7 days. At T + 1 day, the owner lowers the global lockupPeriod to 0 for future stakes. Alice stakes 1 wei of the same asset; stake() overwrites her existing Stake with lastStaked = T + 1 day and lockupPeriod = 0 while preserving the old 100-token amtStaked plus the dust. On the next block, unstake() passes block.timestamp > lastStaked + 0 and Alice can withdraw the whole position roughly 6 days earlier than the original snapshot.

**Mitigation**  
Track lockups per deposit, or split each user/asset position into locked buckets so top-ups only assign the current lockup to the newly added amount. If a single aggregate position is retained, keep the maximum of the existing unlock time and the new deposit unlock time unless intentional retroactive shortening is documented.

### [I-31] V3Locker lock duration starts at deployment instead of funding or configuration

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/V3Locker.sol:13; contracts/contracts/V3Locker.sol:18; contracts/contracts/V3Locker.sol:33; contracts/contracts/V3Locker.sol:38

**Summary/Description**  
V3Locker stores CREATED in the constructor and computes the unlock deadline as CREATED + lockedTime. Because lockedTime is increased by addTime() but remains anchored to deployment, a locker that is funded or configured after deployment can have a newly added lock duration that is already partly or fully elapsed.

**Root Cause**  
The contract has no per-position lock start and does not reset or compare against block.timestamp when time is added. The only deadline state is a deployment timestamp plus a cumulative duration.

**Pre_conditions**  
A V3Locker is deployed before the Uniswap V3 position is transferred in or before the intended lock duration is added; or the owner tries to re-lock after the previous unlock time has already passed.

**Impact**  
The owner can unlock a position earlier than an observer would expect from the newly configured duration. For example, if the locker is 30 days old and addTime(30 days) is called when the NFT is deposited, getUnlockTime() is still the current timestamp or earlier, so unlock() can transfer the NFT to the owner immediately after the strict greater-than boundary. This weakens the lock-time assurance but does not give an untrusted caller custody because unlock() is onlyOwner.

**Proof of Concept**  
Static flow: constructor records CREATED = deployment time. Later, addTime(secs) only executes lockedTime += secs. unlock(lpId) checks block.timestamp > CREATED + lockedTime. If block.timestamp - CREATED already exceeds secs at configuration/funding time, the check is already satisfied or becomes satisfied immediately after the boundary.

**Mitigation**  
Track the active unlock timestamp directly and extend from max(currentUnlockTime, block.timestamp), or start a per-position lock when the NFT is received/deposited. If the contract is meant to lock all held positions globally, addTime should update unlockTime = max(unlockTime, block.timestamp) + secs.

### [I-32] Approved V3 positions can have fees collected to the locker owner

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/V3Locker.sol:22; contracts/contracts/V3Locker.sol:26

**Summary/Description**  
V3Locker.collect() accepts any Uniswap V3 position id and always sets the collect recipient to owner(). For NFTs actually owned by the locker, a public caller only triggers fee payment to the locker owner. However, if a third-party position owner approves V3Locker as an operator or token spender, any caller can invoke collect() for that approved token id and route the accrued fees to V3Locker.owner() instead of the NFT owner.

**Root Cause**  
The wrapper relies on the Uniswap position manager's owner-or-approved authorization but does not verify that the specified token id is owned by address(this), nor does it restrict collect() to the locker owner. Because Uniswap authorization also accepts approved operators, V3Locker can act on approved but non-custodied positions.

**Pre_conditions**  
A Uniswap V3 position owner approves V3Locker for a token id or via setApprovalForAll while the NFT remains outside the locker. The position has uncollected fees. Any address then calls V3Locker.collect(tokenId).

**Impact**  
The accrued token0/token1 fees for the approved non-custodied position are transferred to V3Locker.owner(). This does not let public callers redirect fees from positions held by the locker because the recipient is fixed to owner(), and it does not transfer the NFT itself; the risk is limited to non-intended approvals to this locker contract.

**Proof of Concept**  
Static flow: an external position owner approves V3Locker for tokenId. V3Locker.collect(tokenId) calls the position manager from address(V3Locker), so the position manager treats the call as authorized. The CollectParams recipient is owner(), so all currently owed fees are paid to the locker owner rather than to the position owner.

**Mitigation**  
Require the specified position to be custodied by the locker before collecting, e.g. check ownerOf(tokenId) == address(this) through the position manager, and consider adding onlyOwner if permissionless fee triggering is not required.

### [I-33] V3Locker addTime can push lockedTime past the safe unlock-time bound

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/V3Locker.sol:18; contracts/contracts/V3Locker.sol:33; contracts/contracts/V3Locker.sol:38

**Summary/Description**  
addTime() uses checked uint256 addition, so _secs does not truncate or wrap. However, it only bounds lockedTime against type(uint256).max and does not ensure CREATED + lockedTime remains representable. A very large owner-supplied extension can therefore succeed and make getUnlockTime() and unlock() revert on overflow.

**Root Cause**  
The contract stores a duration but validates only lockedTime += _secs. The later deadline expression CREATED + lockedTime has a stricter safe range: lockedTime must be at most type(uint256).max - CREATED.

**Pre_conditions**  
The owner calls addTime() with a value that leaves lockedTime greater than type(uint256).max - CREATED but not greater than type(uint256).max. For example, from zero, _secs = type(uint256).max - CREATED + 1 succeeds in addTime() and breaks subsequent deadline addition.

**Impact**  
Held Uniswap V3 position NFTs cannot be unlocked through unlock(), because the require expression evaluates CREATED + lockedTime before comparing timestamps and reverts with Solidity arithmetic overflow. Since addTime() only increases lockedTime and there is no setter/decrease/rescue path, this state is permanent unless the contract is replaced. This is owner-induced and not externally triggerable.

**Proof of Concept**  
Static flow: addTime(_secs) writes lockedTime += _secs under Solidity 0.8 checked arithmetic. If the new lockedTime is in (type(uint256).max - CREATED, type(uint256).max], the write succeeds but both getUnlockTime() and unlock() evaluate CREATED + lockedTime and revert before returning or transferring the NFT.

**Mitigation**  
Store an unlock timestamp directly, or require lockedTime + _secs <= type(uint256).max - CREATED before updating. Prefer computing from max(currentUnlockTime, block.timestamp) and bounding the resulting unlock timestamp.

### [I-34] Protocol fee router accepts incompatible fee modules

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/ProtocolFeeRouter.sol:17; contracts/contracts/TokenRewards.sol:152; contracts/contracts/TokenRewards.sol:276

**Summary/Description**  
ProtocolFeeRouter.setProtocolFees can install a zero address or a contract that does not satisfy the ProtocolFees denominator/value assumptions. TokenRewards defensively treats a zero fee module as zero fees in _getYieldFees(), but some downstream fee calculations still call PROTOCOL_FEE_ROUTER.protocolFees().DEN(), so an incompatible module can make reward processing revert for affected pods.

**Root Cause**  
The router setter only stores the supplied IProtocolFees address and does not validate that it is nonzero, has code, returns a nonzero DEN, and exposes bounded yieldAdmin/yieldBurn values compatible with TokenRewards.

**Pre_conditions**  
The router owner sets protocolFees to address(0) or to a non-conforming IProtocolFees implementation, or a pod is deployed with an incompatible fee router in its immutables. Affected reward flow reaches a path that reads DEN, such as leaveRewardsAsPairedLp depositFromPairedLpToken or nonzero admin/burn fee processing.

**Impact**  
Reward processing for affected pods can revert until the fee module is corrected. This is classified as Info because it requires trusted-owner or creator misconfiguration and does not create an untrusted extraction path by itself.

**Proof of Concept**  
In ProtocolFeeRouter.setProtocolFees, set protocolFees to address(0). TokenRewards._getYieldFees then returns zero values, but depositFromPairedLpToken in LEAVE_AS_PAIRED_LP_TOKEN mode still evaluates PROTOCOL_FEE_ROUTER.protocolFees().DEN() when computing _burnAmount, causing the denominator call to a zero/non-contract fee module to revert.

**Mitigation**  
Validate the fee module in the router constructor and setter: require a nonzero contract address, require DEN() > 0, and require yieldAdmin()/yieldBurn() are within the same bounds expected from ProtocolFees. Alternatively centralize denominator access inside _getYieldFees and skip DEN reads when both fees are zero.

### [I-35] Fee updates can apply new rates to previously accrued rewards

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/ProtocolFees.sol:15; contracts/contracts/ProtocolFees.sol:21; contracts/contracts/TokenRewards.sol:141; contracts/contracts/TokenRewards.sol:198; contracts/contracts/TokenRewards.sol:222; contracts/contracts/TokenRewards.sol:276; contracts/contracts/TokenRewards.sol:325; contracts/contracts/AutoCompoundingPodLp.sol:213; contracts/contracts/AutoCompoundingPodLp.sol:237; contracts/contracts/AutoCompoundingPodLp.sol:458; contracts/contracts/AutoCompoundingPodLp.sol:468

**Summary/Description**  
Protocol reward fee setters update live fee rates without snapshotting rewards that have already accrued but have not yet been deposited or compounded. TokenRewards and AutoCompoundingPodLp read the current fee at processing time, so a fee increase can charge the new rate against old pending rewards. FraxlendPair.changeFee avoids this class by accruing interest before changing feeToProtocolRate, but the reward-fee paths do not have an equivalent complete checkpoint.

**Root Cause**  
ProtocolFees.setYieldAdmin and setYieldBurn only write the new values. TokenRewards later calculates admin and burn fees from PROTOCOL_FEE_ROUTER.protocolFees() when depositFromPairedLpToken, depositRewards, or _depositRewards runs. AutoCompoundingPodLp.setProtocolFee processes only reward balances already held by the vault; it does not first claim unpaid TokenRewards rewards for address(this), and processing is skipped entirely while yieldConvEnabled is false. As a result, prior-period reward balances can be processed under a later fee configuration.

**Pre_conditions**  
An owner or timelock changes yieldAdmin, yieldBurn, or AutoCompoundingPodLp.protocolFee. Before the change, there are accrued pod fees, TokenRewards balances, or aspTKN claimable rewards that have not yet been deposited/claimed/compounded.

**Impact**  
Existing reward recipients or aspTKN holders can lose the fee delta on the pending reward batch when it is later processed. The value is redirected to the protocol fee recipient or burn path. This is classified as Info because it depends on a trusted fee update or migration action, but it is a real transition cliff for honest fee changes.

**Proof of Concept**  
Example: an aspTKN has 1,000 paired-token rewards claimable in its StakingPoolToken TokenRewards contract while protocolFee is 0. The owner calls setProtocolFee(1000, ...). setProtocolFee calls _processRewardsToPodLp, but that function only scans balances already held by the aspTKN and does not claim TokenRewards.claimReward(address(this)), so the 1,000 pending rewards remain outside the checkpoint. Later anyone calls TokenRewards.claimReward(address(aspTKN)) and triggers an aspTKN deposit/withdraw; _tokenToPodLp now charges the new 100% protocolFee against the old 1,000 reward batch.

**Mitigation**  
Checkpoint the relevant reward state before fee updates take effect. AutoCompoundingPodLp.setProtocolFee should claim its pending TokenRewards rewards and process held balances under the old fee before writing protocolFee, or use fee epochs so pending rewards keep their accrual-period rate. For global ProtocolFees changes, provide an operational flush/checkpoint path for affected pods or explicitly document that pending rewards are charged at processing-time rates.

### [I-36] Fraxlend self-recipient withdrawals can strand accounted tokens

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fraxlend/src/contracts/FraxlendPair.sol:452; fraxlend/src/contracts/FraxlendPairCore.sol:729; fraxlend/src/contracts/FraxlendPairCore.sol:787; fraxlend/src/contracts/FraxlendPairCore.sol:823; fraxlend/src/contracts/FraxlendPairCore.sol:863; fraxlend/src/contracts/FraxlendPairCore.sol:975; fraxlend/src/contracts/FraxlendPairCore.sol:993

**Summary/Description**  
Fraxlend public receiver/recipient parameters only reject the zero address, while the internal asset and collateral movement helpers treat the pair address itself as a sentinel meaning no token transfer. Calling withdraw, redeem, borrowAsset, removeCollateral, or owner-only withdrawFees with the pair as receiver debits accounting, burns shares, increases debt, or removes collateral accounting while the underlying ERC20 tokens remain in the pair as raw untracked balances.

**Root Cause**  
The internal _redeem, _borrowAsset, and _removeCollateral helpers special-case _receiver == address(this) for internal flows such as LAV returns and liquidation-fee bookkeeping. The external entrypoints expose the same receiver value without rejecting it or using a separate internal-only flag, so an externally supplied pair receiver can trigger accounting effects without token movement.

**Pre_conditions**  
A caller supplies the Fraxlend pair address as the receiver/recipient. In public user paths, the caller must own the affected shares/collateral/debt position or have the normal allowance/position authority. In the fee path, the pair has protocol fee fToken shares and/or protocol liquidation-fee collateral recorded under address(this), and the owner calls withdrawFees with _recipient equal to the pair address.

**Impact**  
Tokens can become stranded as raw asset or collateral balances no longer represented by totalAsset shares, userCollateralBalance, or the intended fee accounting. Public user paths are mostly self-loss or allowance-spender griefing because the authorized caller could otherwise route assets to an arbitrary receiver, so this remains Info. The owner-only fee path can strand protocol fees through an avoidable trusted-input mistake.

**Proof of Concept**  
No test run. Static paths: withdraw or redeem with _receiver = address(this) reaches _redeem, which reduces totalAsset and burns shares but skips assetContract.safeTransfer. borrowAsset with _receiver = address(this) adds user debt and totalBorrow but skips transferring the borrowed asset. removeCollateral with _receiver = address(this) reduces userCollateralBalance and totalCollateral but skips collateralContract.safeTransfer. withdrawFees(_shares, address(this)) combines the same _redeem and _removeCollateral no-transfer branches for protocol-owned fee balances.

**Mitigation**  
Reject receiver/recipient == address(this) in external user and fee entrypoints unless a dedicated internal-only flow is being executed. Prefer making the no-transfer behavior an explicit internal parameter that is not reachable through arbitrary external receiver input.

### [I-37] Fraxlend default swappers cannot be installed after constructor ownership handoff

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fraxlend/src/contracts/FraxlendPairAccessControl.sol:66; fraxlend/src/contracts/FraxlendPairDeployer.sol:171; fraxlend/src/contracts/FraxlendPairDeployer.sol:276; fraxlend/src/contracts/FraxlendPair.sol:480

**Summary/Description**  
FraxlendPairDeployer supports owner-configured defaultSwappers and tries to approve each default on a newly deployed pair, but the pair constructor has already transferred ownership from the deployer to the configured comptroller. If defaultSwappers is non-empty and the comptroller is not the deployer contract, deploy() reaches FraxlendPair.setSwapper() with msg.sender equal to the deployer and reverts on onlyOwner.

**Root Cause**  
The pair ownership handoff occurs inside the pair constructor before FraxlendPairDeployer performs its post-deploy owner-only setup. setDefaultSwappers stores a deployer-level default list, but _deploy() does not retain pair ownership long enough to apply it.

**Pre_conditions**  
The FraxlendPairDeployer owner sets a non-empty defaultSwappers array, and comptrollerAddress for newly deployed pairs is an address other than the deployer contract.

**Impact**  
Whitelisted deployers cannot deploy new pairs while this configuration is active, because every deployment that reaches the default-swapper loop reverts. This is classified as Info because it requires trusted owner configuration and does not let an untrusted caller seize ownership or funds.

**Proof of Concept**  
Set defaultSwappers to any non-empty array while comptrollerAddress is not address(this). During deploy(), the new pair constructor executes _transferOwnership(_comptrollerAddress). _deploy() then iterates defaultSwappers and calls _fraxlendPair.setSwapper(...), but FraxlendPair.setSwapper is onlyOwner, so the call reverts because msg.sender is FraxlendPairDeployer, not the comptroller owner.

**Mitigation**  
Apply default swappers before transferring ownership, pass default swappers into the pair constructor, or remove the deployer-side default loop and require the configured pair owner/comptroller to set swappers after deployment.

### [I-38] IndexManager permits duplicate index registrations

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/IndexManager.sol:57; contracts/contracts/IndexManager.sol:65; contracts/contracts/IndexManager.sol:94; contracts/contracts/IndexManager.sol:101; contracts/contracts/IndexManager.sol:112; contracts/contracts/interfaces/IIndexManager.sol:7

**Summary/Description**  
IndexManager.addIndex can register the same index address more than once because _addIndex overwrites _indexIdx[_index] with the latest array position and then appends a new IIndexAndStatus row without checking prior registration. Once duplicated, allIndexes() exposes conflicting rows for the same pod while address-based metadata updates and creator authorization resolve only through the newest row.

**Root Cause**  
In contracts/contracts/IndexManager.sol:65, _addIndex lacks a uniqueness check for _index before writing _indexIdx[_index] and pushing to indexes. The mapping stores only one array index, so duplicate array entries cannot all remain address-addressable.

**Pre_conditions**  
Owner or an authorized curator calls addIndex with an _index address that already exists in indexes, or a configured factory path returns an already-registered address to deployNewIndex.

**Impact**  
The registry can contain duplicate entries with different creator, verified, selfLending, or makePublic metadata for one pod. updateMakePublic(), updateSelfLending(), and onlyAuthorizedOrCreator() use _indexIdx[_index], so they operate on the latest duplicate only; older duplicate rows remain visible through allIndexes() and their recorded creators cannot update those stale rows through the address-based path. Removing a duplicate can also leave the remaining row disconnected from _indexIdx, requiring trusted curation to repair the registry. No direct fund loss path was confirmed.

**Proof of Concept**  
Call addIndex(index, creatorA, false, false, false), then addIndex(index, creatorB, true, true, true). The second call succeeds and _indexIdx[index] points to the second row. allIndexes() now returns two rows for the same index. creatorA no longer passes onlyAuthorizedOrCreator(index), and updateMakePublic(index, ...) or updateSelfLending(index, ...) can only change the newest row.

**Mitigation**  
Track registration explicitly and reject repeated addresses, for example require(!_isIndex[_index]) before push and set _isIndex[_index] = true. Clear that registration flag on removal only when no other row for the same address can exist; with a uniqueness guard, normal removal can clear it directly.

### [I-39] IndexManager address-based status setters can update wrong rows

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/IndexManager.sol:79; contracts/contracts/IndexManager.sol:81; contracts/contracts/IndexManager.sol:83; contracts/contracts/IndexManager.sol:94; contracts/contracts/IndexManager.sol:101; contracts/contracts/IndexManager.sol:112

**Summary/Description**  
IndexManager.updateMakePublic() and updateSelfLending() resolve the target row through _indexIdx[_index] but never validate that indexes[_idx].index equals the supplied _index. Missing mapping keys default to 0, so unknown or removed non-last addresses can resolve to row 0. removeIndex adds another stale-reference case: when removing the last row, it deletes the removed address mapping but then unconditionally writes that same removed address back to _indexIdx before pop, leaving the removed address mapped to an out-of-bounds slot that can later alias a newly appended row.

**Root Cause**  
The address-based status update path uses a uint256 index mapping without an existence sentinel, bounds check, or target-address equality check. removeIndex also updates the moved-row mapping without checking whether a row was actually moved, so the last-element removal case recreates the mapping for the address being removed.

**Pre_conditions**  
At least one index exists. A caller with owner/authorized permissions, or the creator of the row reached by the stale mapping, calls updateMakePublic() or updateSelfLending() with an address that is not currently present in the registry. For the last-removal variant, an authorized curator removes the final row and a later addIndex/deployNewIndex may reuse the stale slot.

**Impact**  
The registry can report incorrect makePublic or selfLending metadata for a row that does not match the supplied _index address. Unknown or removed non-last addresses can mutate row 0, and a removed last address can either revert before slot reuse or alias a newly appended row after slot reuse. In scoped contracts these fields are exposed through allIndexes() and used as registry/UI classification metadata; no direct funds-at-risk path or untrusted privilege expansion was confirmed because authorization is evaluated against the destination row's creator, while owner/authorized curators can already edit registry metadata.

**Proof of Concept**  
Default-zero case: with an index at row 0, call updateMakePublic(unknownAddress, newValue) from owner, an authorized curator, or row 0's creator. _indexIdx[unknownAddress] is 0, so the call updates row 0. Last-removal case: register A, B, C, then call removeIndex(2). removeIndex deletes _indexIdx[C], self-assigns indexes[2] from indexes[2], then writes _indexIdx[C] = 2 before popping. The array length is now 2 while C still maps to slot 2. If D is later added, D occupies slot 2 and updateMakePublic(C, ...) or updateSelfLending(C, ...) resolves to D's row.

**Mitigation**  
Track index existence separately or store one-based indexes in _indexIdx. In each address-based setter, require that the resolved row is in bounds and indexes[_idx].index == _index before applying status changes. In removeIndex, use a guarded swap-and-pop that only copies and updates the last row when _idxInAry != lastIdx, then clears the removed address registration after pop.

### [I-40] setExternalAssetVault can apply new LAV capacity to elapsed interest

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fraxlend/src/contracts/FraxlendPairAccessControl.sol:202; fraxlend/src/contracts/FraxlendPairAccessControl.sol:210; fraxlend/src/contracts/FraxlendPairCore.sol:286; fraxlend/src/contracts/FraxlendPairCore.sol:307; fraxlend/src/contracts/FraxlendPairCore.sol:380; fraxlend/src/contracts/FraxlendPairCore.sol:398; fraxlend/src/contracts/libraries/VaultAccount.sol:16; contracts/contracts/LendingAssetVault.sol:73

**Summary/Description**  
FraxlendPair.setExternalAssetVault replaces the external LAV pointer without first checkpointing accrued interest under the previous liquidity configuration. The next internal _addInterest call computes the full elapsed interval since currentRateInfo.lastTimestamp using totalAsset.totalAmount(address(externalAssetVault)), so a live vault migration can count newly added LAV capacity, or ignore removed capacity, for time before the setter executed.

**Root Cause**  
The setter writes externalAssetVault directly and exposes no non-skipping sync path. Fraxlend's public addInterest(false) is not a reliable pre-setter checkpoint because it can emit SkipAddingInterest when the utilization delta is below minURChangeForExternalAddInterest, while only the internal _addInterest path always accrues when timestamp and pause conditions allow. _calculateInterest later reads the new externalAssetVault address for the whole deltaTime.

**Pre_conditions**  
A pair has nonzero borrow debt, time has elapsed since the last internal interest update, and the timelock changes externalAssetVault to a valid LAV with materially different totalAvailableAssetsForVault(address(pair)) capacity. This is a trusted-timelock transition path; the README setup order sets the vault before LAV funding, so the issue is classified as Info rather than an untrusted exploit.

**Impact**  
Adding a funded or high-headroom LAV before checkpointing lowers measured utilization for the elapsed interval, causing borrowers to underpay interest and suppressing rate growth for existing lenders/protocol. Removing or switching away from capacity can overcharge borrowers for the same elapsed interval. The effect applies to all outstanding borrow amount over the uncheckpointed time window, but is bounded to the migration window and requires trusted governance/operator action.

**Proof of Concept**  
No test run. Code path: at T0 currentRateInfo.lastTimestamp is last checkpointed with externalAssetVault unset or pointing to a lower-capacity vault. Time passes with totalBorrow.amount nonzero. At T1 the timelock calls setExternalAssetVault(newVault), which only assigns externalAssetVault. At T2 any deposit/borrow/repay/withdraw/liquidation path calls internal _addInterest(); _calculateInterest uses deltaTime = T2 - T0 and computes utilization from totalAsset.totalAmount(address(newVault)), which adds newVault.totalAvailableAssetsForVault(address(this)) for the whole elapsed interval even though that capacity was only connected at T1.

**Mitigation**  
In setExternalAssetVault(), force an internal _addInterest() checkpoint before assigning the new vault pointer. If operators need an explicit pre-migration sync, expose a non-skipping timelock/owner checkpoint or make the setter itself synchronize both the pair and relevant LAV metadata before the pointer changes.

### [I-41] Zero timelock initialization permanently disables Timelock2Step administration

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fraxlend/src/contracts/Timelock2Step.sol:30; fraxlend/src/contracts/Timelock2Step.sol:35; fraxlend/src/contracts/Timelock2Step.sol:63; fraxlend/src/contracts/Timelock2Step.sol:74; fraxlend/src/contracts/Timelock2Step.sol:95; fraxlend/src/contracts/Timelock2Step.sol:117; fraxlend/src/contracts/FraxlendPairAccessControl.sol:64; fraxlend/src/contracts/FraxlendPairAccessControl.sol:66; fraxlend/src/contracts/FraxlendPairDeployer.sol:95; fraxlend/src/contracts/FraxlendPairDeployer.sol:186; fraxlend/src/contracts/FraxlendPairDeployer.sol:301; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:69

**Summary/Description**  
Timelock2Step uses address(0) as the default pending timelock and also as the final renounced timelock. The renounce flow itself is guarded for a normal nonzero timelock because renounceTimelock requires msg.sender to be both the current and pending timelock. However, concrete constructors and FraxlendPairDeployer.setTimelock accept address(0), which can initialize future pairs or an oracle with timelockAddress == pendingTimelockAddress == address(0). From that state no account can satisfy _requireTimelock or _requirePendingTimelock, so transferTimelock, acceptTransferTimelock, renounceTimelock, and timelock-only administration are permanently unavailable.

**Root Cause**  
Initialization and configuration paths reuse the same zero-address value that represents a completed renounce, but they do not reject address(0) before writing timelockAddress. The code therefore allows a live contract to start in an unrecoverable post-renounce-like state without an explicit renounce action.

**Pre_conditions**  
A trusted deployment/configuration path passes address(0) as the timelock: for example FraxlendPairDeployer is constructed with _params.timelock == address(0), the deployer owner calls setTimelock(address(0)) before future deploy() calls, or DualOracleChainlinkUniV3 is deployed with _timelockAddress == address(0). This is a trusted-configuration path, so it is classified as Info rather than an untrusted primary issue.

**Impact**  
Affected Fraxlend pairs cannot later execute timelock-only functions such as setExternalAssetVault, setMaxLTV, revokeRateContractSetter, setRateContract, or changeFee. Affected dual oracles cannot update maxOracleDelay. Existing pair owner/comptroller privileges remain, but the Timelock2Step role itself cannot be recovered on affected deployments.

**Proof of Concept**  
Code path: pendingTimelockAddress defaults to address(0). A constructor or deployer-provided immutable calls _setTimelock(address(0)), so timelockAddress also becomes address(0). transferTimelock and renounceTimelock both begin with _requireTimelock, and acceptTransferTimelock begins with _requirePendingTimelock. Because msg.sender can never be address(0), all Timelock2Step transition functions revert forever, and every external function gated by _requireTimelock is unreachable.

**Mitigation**  
Reject address(0) on initialization and normal timelock transfer/configuration paths. Keep zero assignment limited to an explicit renounceTimelock path after the current timelock has deliberately staged itself as pending, or add a separate deployment-time recovery path before any user-facing pair/oracle is activated.

### [I-42] Fraxlend creation-code storage lacks length cleanup

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fraxlend/src/contracts/FraxlendPairDeployer.sol:57; fraxlend/src/contracts/FraxlendPairDeployer.sol:58; fraxlend/src/contracts/FraxlendPairDeployer.sol:160; fraxlend/src/contracts/FraxlendPairDeployer.sol:163; fraxlend/src/contracts/FraxlendPairDeployer.sol:253

**Summary/Description**  
FraxlendPairDeployer.setCreationCode can leave the deployer with an incomplete or stale creation-code representation because the deploy path always concatenates two SSTORE2 pointer slots, while the setter only overwrites the second pointer when the new payload is longer than 13,000 bytes and stores no chunk count, length, or expected code hash. A boundary or future shorter creation-code update can make subsequent pair deployment revert or build malformed init code.

**Root Cause**  
contractAddress1 and contractAddress2 are treated as the full creation-code cache, but setCreationCode writes a fixed 13,000-byte first chunk and conditionally writes contractAddress2 only for length greater than 13,000. It never clears contractAddress2 for a one-chunk payload and _deploy unconditionally reads both pointers. BytesLib.slice also rejects payloads below 13,000 bytes, while SSTORE2.read(address(0)) panics because address(0).code.length is zero.

**Pre_conditions**  
The deployer owner updates the stored pair creation code to a one-chunk or exact-threshold payload, or future FraxlendPair creation bytecode drops below the current threshold. Current compiled FraxlendPair creation bytecode is 28,188 bytes, so the normal scoped setup writes both pointers.

**Impact**  
Future pair deployments can revert or use malformed init code until the owner corrects the stored creation code. This is a trusted-configuration liveness issue; no untrusted current funds-at-risk path was confirmed.

**Proof of Concept**  
Current FraxlendPair creation bytecode is 28,188 bytes, producing a 13,000-byte first chunk and 15,188-byte second chunk. If a later setCreationCode call stores exactly 13,000 bytes, contractAddress2 is not updated. If it was unset, _deploy's SSTORE2.read(contractAddress2) panics; if it was previously set, _deploy appends stale old bytes before the constructor args.

**Mitigation**  
Store explicit creationCodeLength and chunk count, clear contractAddress2 when the new payload has no second chunk, and make _deploy read only initialized chunks. Slice min(_creationCode.length, 13,000) so shorter valid payloads can be stored. Consider storing and checking an expected creation-code hash and emitting an update event.

### [I-43] globalPause returns a sparse success list after partial failures

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: fraxlend/src/contracts/FraxlendPairDeployer.sol:319; fraxlend/src/contracts/FraxlendPairDeployer.sol:324; fraxlend/src/contracts/FraxlendPairDeployer.sol:327; fraxlend/src/contracts/FraxlendPairDeployer.sol:329

**Summary/Description**  
FraxlendPairDeployer.globalPause() is documented as returning the addresses for which pause() was successful, but it allocates the return array to the full input length and only writes successful entries at their original indexes. Failed pause attempts are silently caught and remain address(0), so callers cannot treat the returned length as the number of paused pairs and address(0) is only an implicit failure marker.

**Root Cause**  
The function uses try/catch to ignore per-pair pause failures, but it does not maintain a success counter, compact the returned list, emit failures, or otherwise return explicit per-address status.

**Pre_conditions**  
The configured circuitBreakerAddress calls globalPause() with at least one address whose pause() call reverts or otherwise is not a successful FraxlendPair pause.

**Impact**  
Emergency tooling or an integrating circuit-breaker contract that assumes the returned array contains only successful pauses can overstate completion of a batch pause. The issue is Info because globalPause is restricted to the trusted circuit breaker, the code intentionally ignores reverts, and a careful caller can inspect zero entries to identify failures.

**Proof of Concept**  
Call globalPause([validPair, failingPair]). The valid pair pause succeeds and _updatedAddresses[0] is set to validPair. The failing pair call is caught by catch {}, _updatedAddresses[1] remains address(0), and the function returns an array of length 2 despite only one successful pause.

**Mitigation**  
Track a success count and return a compact array of only successful addresses, or return a parallel success bitmap/status array and emit events for failed pause attempts.

### [I-44] Fraxlend registry deployer role is easy to confuse with deployment whitelist

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fraxlend/src/contracts/FraxlendPairRegistry.sol:73; fraxlend/src/contracts/FraxlendPairRegistry.sol:90; fraxlend/src/contracts/FraxlendPairDeployer.sol:291; fraxlend/src/contracts/FraxlendPairDeployer.sol:306; fraxlend/src/contracts/FraxlendWhitelist.sol:44

**Summary/Description**  
Fraxlend uses two similarly named deployer lists with different principals. FraxlendWhitelist.fraxlendDeployerWhitelist gates the external caller of FraxlendPairDeployer.deploy(), while FraxlendPairRegistry.deployers gates the immediate caller of FraxlendPairRegistry.addPair(), which is the deployer contract in the canonical flow. Because setDeployers accepts arbitrary addresses and addPair does not validate pair provenance beyond name(), mirroring an EOA deployment whitelist entry into the registry lets that EOA directly register arbitrary ERC20-like addresses; omitting the deployer contract from the registry makes otherwise valid deployments revert at registration.

**Root Cause**  
The registry role is named and managed as a deployer list but it is actually a registrar/caller allowlist for addPair(). The code does not separate end-user deployment permission from contract-level registry insertion permission in naming or validation.

**Pre_conditions**  
The trusted registry owner configures FraxlendPairRegistry.deployers with an EOA or other address intended for FraxlendWhitelist.fraxlendDeployerWhitelist, or fails to include the actual FraxlendPairDeployer contract.

**Impact**  
This can corrupt the public registry with arbitrary name-bearing addresses or block legitimate pair creation, but it requires avoidable trusted-owner misconfiguration. No untrusted path was found that makes an unlisted caller pass the addPair deployer check or makes FraxlendPairDeployer register an arbitrary caller-supplied address.

**Proof of Concept**  
Canonical setup in LivePOC whitelists address(this) in FraxlendWhitelist and address(deployer) in FraxlendPairRegistry. If the registry instead whitelists address(this), address(this) can call addPair(fakeERC20WithUniqueName) directly, while deployer.deploy(...) reverts at IFraxlendPairRegistry.addPair because msg.sender at the registry is the FraxlendPairDeployer contract, not address(this).

**Mitigation**  
Rename the registry list and setter to registrar/factory terminology, document that it must contain FraxlendPairDeployer contracts rather than deployment EOAs, and consider validating registered pairs by checking IFraxlendPair(_pairAddress).DEPLOYER_ADDRESS() == msg.sender or by restricting addPair to a single configured factory.

### [I-45] LeverageManager access mappings accept incompatible pair and flash-source domains

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/lvf/LeverageManagerAccessControl.sol:17; contracts/contracts/lvf/LeverageManagerAccessControl.sol:24; contracts/contracts/lvf/LeverageManager.sol:293; contracts/contracts/lvf/LeverageManager.sol:522; contracts/contracts/lvf/LeverageManager.sol:580; contracts/contracts/lvf/LeverageManager.sol:590; contracts/contracts/lvf/LeverageFactory.sol:99; contracts/contracts/lvf/LeverageFactory.sol:127; contracts/contracts/lvf/LeverageFactory.sol:143

**Summary/Description**  
LeverageManagerAccessControl stores lendingPairs[pod] and flashSource[borrowAsset] after only shallow interface checks. setLendingPair only checks that pair.collateralContract() is nonzero and does not validate that the pair asset/collateral matches the pod. setFlashSource only checks that source() is nonzero and does not validate that the source is bound to this LeverageManager or can flash the mapped borrow asset. Later leverage flows treat these mappings as authoritative and branch on pair.asset(), pair.collateralContract(), and flashSource[pair.asset()].

**Root Cause**  
The mapping keys represent specific domains, pod to lending pair and borrow asset to flash source, but the setters only validate that the target exposes one nonzero value. The compatibility invariants are implicit in later LeverageManager and flash-source calls instead of enforced at mapping update time.

**Pre_conditions**  
A trusted owner or owner-controlled factory registers a lending pair whose asset or collateral does not match the intended pod, or registers a flash source that is not authorized for this LeverageManager or does not support the mapped borrow asset.

**Impact**  
No untrusted path was found to mutate these mappings. Most incompatible configurations revert during flash-source lookup, onlyLeverageManager checks, ERC4626 collateral deposit, LP zapping, or Fraxlend borrow/remove calls. The confirmed impact is configuration-driven LVF unavailability or positions routed into the wrong self-lending/zap branch, so this is Info rather than a Medium or High issue.

**Mitigation**  
Validate mappings at update time. For lending pairs, require a nonzero pod and pair, require IFraxlendPair(_pair).collateralContract() to be the expected aspTKN shape for the pod, for example ERC4626(collateral).asset() == IDecentralizedIndex(_pod).lpStakingPool(), and explicitly validate the normal versus self-lending pair asset relationship. For flash sources, add a supportsToken/manager check or source-specific validation that the source is bound to this LeverageManager and can flash _borrowAsset.

### [I-46] PoolAddressAlgebra hardcodes Camelot-specific init code hash

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/libraries/PoolAddressAlgebra.sol:6; contracts/contracts/twaputils/V3TwapCamelotUtilities.sol:19; contracts/contracts/interfaces/IAlgebraFactory.sol:89

**Summary/Description**  
PoolAddressAlgebra derives Algebra pool addresses with the fixed 0x6c1bebd370ba84753516bc1393c0d0a6c645856da55f5393ac8ab3d6dbc861d3 init-code hash. That value matches the verified Camelot Arbitrum V3 factory, but Algebra pool init-code hashes are deployment-specific and the factory already exposes poolByPair. If the Camelot TWAP utility is deployed against a different Algebra factory or a future deployment with a different pool bytecode hash, getV3Pool will silently return a non-canonical address.

**Root Cause**  
The helper manually performs CREATE2 with a hardcoded pool init-code hash instead of querying the factory's poolByPair/computePoolAddress path or reading the deployment-specific hash from the factory.

**Pre_conditions**  
TWAP_TYPE=1 / V3TwapCamelotUtilities is wired to an Algebra-style factory whose pool deployer/hash differs from the Camelot Arbitrum V3 deployment.

**Impact**  
Pool-dependent quote, TWAP, slippage, and reward-conversion paths can read an address with no pool code or a wrong derived pool address, causing reverts or mispriced protection. This is not confirmed for the current Camelot Arbitrum factory because the hash matches that verified deployment.

**Proof of Concept**  
Not run; the issue is a configuration/portability mismatch. The local constant can be compared directly against the verified Camelot Arbitrum factory hash and against Algebra documentation that treats the hash as deployment-specific.

**Mitigation**  
Use IAlgebraFactory.poolByPair for existing pools, or extend the interface to use the factory's computePoolAddress/POOL_INIT_CODE_HASH for the exact deployment. Add deployment-time validation that the selected TWAP utility, router factory, poolDeployer, and init-code hash match.

### [I-47] Transfer-tax pTKN paired assets can block pod LP flows

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: contracts/contracts/DecentralizedIndex.sol:174; contracts/contracts/DecentralizedIndex.sol:344; contracts/contracts/DecentralizedIndex.sol:348; contracts/contracts/IndexUtils.sol:70; contracts/contracts/IndexUtils.sol:82; contracts/contracts/lvf/LeverageManager.sol:483; contracts/contracts/lvf/LeverageManager.sol:538

**Summary/Description**  
A pod can be configured with another pod token as PAIRED_LP_TOKEN, including in the podded self-lending flow. If that paired pTKN has config.hasTransferTax enabled, addLiquidityV2 and the IndexUtils/LeverageManager wrappers pass nominal paired amounts through multiple transfers. The paired pTKN taxes the transfer into the pod, so the pod receives less than _pairedLPTokens but still approves and asks the dex adapter to pull the nominal amount. That makes direct LP, IndexUtils addLPAndStake, and LVF addLeverage revert or otherwise operate on stale/residual balances.

**Root Cause**  
DecentralizedIndex.addLiquidityV2 disables fees only for the current pod token. It does not validate that PAIRED_LP_TOKEN is non-taxed, measure the actual paired-token balance delta after safeTransferFrom, or pass the actual received amount into the dex adapter. The deployment and LVF paths also do not reject a transfer-tax pTKN as a paired asset.

**Pre_conditions**  
A pod's PAIRED_LP_TOKEN is another pTKN whose config.hasTransferTax is true, for example a podded self-lending paired wrapper selected with _hasSelfLendingPairPod. A user or LeverageManager attempts to add LP through addLiquidityV2 or IndexUtils.addLPAndStake.

**Impact**  
The strongest confirmed impact is configuration-driven unavailability of LP creation and leverage adds for that pod pairing. Because the issue requires selecting an incompatible protocol-owned paired asset and can be avoided by disabling the paired wrapper's transfer tax, this is recorded as Info rather than a contest-level Medium.

**Proof of Concept**  
For pod A with PAIRED_LP_TOKEN = pTKN B and B.config.hasTransferTax = true: IndexUtils receives B from the caller and passes its observed B balance to A.addLiquidityV2. A.addLiquidityV2 then calls B.safeTransferFrom(IndexUtils, A, _pairedLPTokens). B._update treats A as a normal transfer recipient, charges the 0.01% transfer tax, and A receives less than _pairedLPTokens. A nevertheless approves _pairedLPTokens to DEX_HANDLER and calls addLiquidity with _pairedLPTokens. The adapter's transferFrom(A, adapter, _pairedLPTokens) cannot be satisfied from A's lower balance, so the LP operation reverts. LeverageManager reaches the same path after bonding lending-pair shares into the paired wrapper pTKN at lines 538-541 and then calling indexUtils.addLPAndStake at line 483.

**Mitigation**  
Reject transfer-tax pTKNs as PAIRED_LP_TOKEN, or require official paired wrapper pods to have hasTransferTax disabled. If fee-on-transfer paired assets are meant to be supported, measure the paired-token balance delta inside addLiquidityV2 after the transfer, approve only the actual received amount, and ensure the downstream router path enforces slippage against actual received amounts.

### [I-48] Rebasing pod assets desynchronize pTKN backing accounting

**Severity**: Info  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/WeightedIndex.sol:91; contracts/contracts/WeightedIndex.sol:145; contracts/contracts/WeightedIndex.sol:166; contracts/contracts/WeightedIndex.sol:186; contracts/contracts/DecentralizedIndex.sol:250; contracts/contracts/DecentralizedIndex.sol:413; README.md:13

**Summary/Description**  
WeightedIndex prices bond and debond operations from the internal _totalAssets mapping rather than the actual underlying token balances. This is compatible with fixed-supply ERC20 assets, but if a configured pod asset rebases, the recorded backing and custody balance diverge. Positive rebases become untracked surplus, while negative rebases leave _totalAssets overstated and let early debonders withdraw against the stale pre-rebase ledger until the actual token balance is exhausted.

**Root Cause**  
Only bond() and debond() update _totalAssets. The transfer validation checks the incoming balance delta for the requested amount, and flash() only checks that the post-flash balance is at least the pre-flash balance, but no path synchronizes external balance drift from rebases into managed-asset accounting before share conversion or redemption.

**Pre_conditions**  
A pod is configured with a rebasing or elastic-supply index asset despite the README stating that weird tokens are not formally supported. The asset balance changes after users have bonded into the pod.

**Impact**  
For a negative rebase, pTKN holders who exit first can redeem more than their fair share of the post-rebase backing, leaving later holders with undercollateralized pTKN or withdrawals that revert once the actual token balance is insufficient. For a positive rebase, the extra asset balance is not reflected by totalAssets() or debond() and remains stuck/unclaimable by pTKN holders. The formal severity is informational because rebasing assets appear outside the documented token assumptions.

**Proof of Concept**  
With zero fees for clarity, Alice and Bob each own 50 pTKN backed by 100 recorded units of a rebasing asset. The asset negatively rebases the pod balance to 50 while _totalAssets remains 100. Alice debonds 50 pTKN, percShares = 50%, so debond transfers 50 recorded assets and succeeds, consuming the entire actual balance. Bob still owns 50 pTKN against _totalAssets = 50, but the pod holds 0 tokens, so his later debond cannot be paid.

**Mitigation**  
Do not allow rebasing or elastic-supply assets in pod creation, or wrap them into non-rebasing shares before using them as pod assets. If rebasing assets must be supported, synchronize each managed asset's recorded backing to actual balance before share conversion/redemption and define how positive/negative rebases are socialized across pTKN holders. Add explicit deployment validation and documentation for unsupported token classes.

### [I-49] Raw Fraxlend approvals can reject nonstandard lending assets

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fraxlend/src/contracts/FraxlendPairCore.sol:673; fraxlend/src/contracts/FraxlendPairDeployer.sol:273

**Summary/Description**  
Fraxlend uses SafeERC20 for asset transfers but two lending-asset approval paths call IERC20.approve directly. A no-return ERC20 asset can make the high-level approve call revert during LAV return/deployment seeding, and a false-return approve is not checked before the later transferFrom path. In the LAV return path, _withdrawToVault can also be reached with amountToReturn == 0, so lending assets that revert on approve(spender, 0) can block that path.

**Root Cause**  
The lending integration approves the LAV or newly deployed pair through the raw ERC20 interface instead of optional-return-safe, token-behavior-aware approval handling. The LAV return logic does not skip approval/deposit work when the computed return amount is zero.

**Pre_conditions**  
A Fraxlend market or LAV-backed pair is configured with an asset whose approve has no return value, can return false, or reverts on approve(spender, 0). For the zero-approval variant, the pair reaches _withdrawToVault(0), for example when the external LAV is globally over-utilized but this pair has zero current vaultUtilization. The README says weird tokens are not formally supported, so this is an integration/configuration risk.

**Impact**  
Return-to-LAV paths such as repay or large deposits that call _withdrawToVault can revert before assets are returned to the LAV, and default-deposit pair deployment can revert for no-return/false-return assets. The zero-approval variant can make an otherwise ordinary pair deposit revert when the computed amount to return is zero. Under standard ERC20 assets the path works normally.

**Proof of Concept**  
No test run per instruction. Code trace: _withdrawToVault computes shares, then calls assetContract.approve(address(externalAssetVault), amount) before externalAssetVault.whitelistDeposit(amount). A no-return approve is incompatible with the high-level IERC20 bool-return call; a false approval is ignored and the later LAV safeTransferFrom reverts atomically if allowance was not set. If amount is zero, the same raw call is approve(vault, 0), which a zero-approval-reverting lending asset rejects. FraxlendPairDeployer.deploy also uses raw approve before default pair seeding.

**Mitigation**  
Use optional-return-safe approval handling and skip approval/whitelistDeposit when the LAV return amount is zero. Keep deployment allowlists excluding no-return, false-return, and zero-approval-reverting assets if the protocol intentionally does not support them.

### [I-50] Raw pod-token approval can block addLeverageFromTkn for no-return or zero-reset assets

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/lvf/LeverageManager.sol:575; contracts/contracts/WeightedIndex.sol:162; contracts/contracts/WeightedIndex.sol:167; contracts/contracts/DecentralizedIndex.sol:250

**Summary/Description**  
LeverageManager.addLeverageFromTkn pulls the pod's first underlying token with SafeERC20, then approves the pod with a raw IERC20.approve(). A no-return token can make that high-level approve call revert before bonding, and a false-return approval is not checked before the pod later pulls funds. Separately, on non-first WeightedIndex bonds the pod may spend less than the approved amount due to floor rounding, leaving residual allowance that can make zero-reset assets revert on later calls.

**Root Cause**  
The helper uses raw IERC20.approve() instead of SafeERC20 optional-return and force-approve handling, while the downstream bond flow can spend less than the approved amount.

**Pre_conditions**  
A pod's first asset has no-return, false-return, or zero-reset approval behavior. For the residual-allowance variant, a successful addLeverageFromTkn call reaches a non-first bond amount where WeightedIndex._bond spends less than the amount approved, and a later call uses the same LeverageManager, pod, and token.

**Impact**  
addLeverageFromTkn can revert for affected pod assets, either immediately at approve() for no-return assets or on a later nonzero-to-nonzero approval for zero-reset assets. Users can still manually bond to pTKN and call addLeverage(), and the README excludes unsupported token behavior, so this remains a compatibility/liveness issue rather than a core-loss finding.

**Proof of Concept**  
Static trace: addLeverageFromTkn calls _bondToPod; _bondToPod safeTransferFroms the first pod asset from the user, then executes _tkn.approve(_pod, receivedAmount). Solidity's high-level bool-return call reverts when the token returns no data, and ignores a decoded false value. For the zero-reset variant, a non-first bond with totalAssets[source] = 3 and amount = 2 approves 2, but WeightedIndex transfers floor(3 * floor(2 * 2^96 / 3) / 2^96) = 1, leaving residual allowance so the next nonzero approve can revert.

**Mitigation**  
Use SafeERC20.forceApprove or an equivalent optional-return-safe zero-reset pattern. Prefer approving only the exact amount the pod will spend, or revoke any residual allowance after bond completes.

### [I-51] Bridge messages are not bound to the remote source token

**Severity**: Info  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/ccip/TokenBridge.sol:72; contracts/contracts/ccip/TokenBridge.sol:85; contracts/contracts/ccip/TokenBridge.sol:88; contracts/contracts/ccip/TokenBridge.sol:109; contracts/contracts/ccip/TokenBridge.sol:111; contracts/contracts/ccip/TokenRouter.sol:25

**Summary/Description**  
TokenBridge bridge messages carry the receiver, local destination token, and amount, but they do not carry the remote source token, the remote mint/burn mode, or a route identifier. On receive, TokenBridge looks up the local token config by the payload targetToken and CCIP source chain selector, checks only that the CCIP sender is the configured remote bridge, then mints when the local config is mint-burn. If the remote router is configured to map a different source token into the same local target token, the local bridge cannot detect that the destination mint was backed by the wrong remote asset or by a lock path instead of the expected burn path.

**Root Cause**  
The bridge authenticates the remote bridge contract at lane level, but the payload schema and receive validation do not bind the mint to the expected remote source token or route mode. TokenRouter.setConfig stores routes independently and does not enforce a bidirectional token-pair or mint/burn-mode invariant.

**Pre_conditions**  
The local route for a token and source chain is enabled and trusts the remote bridge. The remote route to this chain is mistakenly enabled for another source token while using this local token as targetToken. A user then bridges through that remote route.

**Impact**  
The local bridge can mint or unlock the configured target token for a transfer backed by a different remote token or an unexpected remote custody mode. In mint/burn mode this can inflate the local token relative to the intended cross-chain asset; in lock/unlock mode it can release local bridge reserves to transfers that were not backed by the matching remote token. This is Info because no user-controlled bypass exists when route configuration is correct; it depends on trusted-operator misconfiguration.

**Proof of Concept**  
Static flow: on the remote chain, setConfig(remoteBridgeLocal, localSelector, cheapToken, false or true, valuableLocalToken, true) makes bridgeTokens encode TokenTransfer({targetToken: valuableLocalToken, amount: amount}). The payload contains no cheapToken field and no proof that cheapToken was burned. On the local chain, _ccipReceive decodes targetToken, loads getConfig(valuableLocalToken, remoteSelector), and only checks that message.sender is the configured remote bridge. If the local config uses sourceTokenMintBurn=true, _processOutboundTokens mints valuableLocalToken to the receiver without validating which remote token or route mode backed the message.

**Mitigation**  
Include the remote source token and route mode, or a route id, in TokenTransfer and require it to match the local config's expected remote counterpart before minting or unlocking. Validate nonzero route addresses and consider configuring bidirectional routes atomically with explicit pair and mint/burn-mode checks.

### [I-52] Wrapper and locker asset balance drift can desynchronize fixed-amount ledgers

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/AutoCompoundingPodLp.sol:96; contracts/contracts/AutoCompoundingPodLp.sol:134; contracts/contracts/AutoCompoundingPodLp.sol:197; contracts/contracts/AutoCompoundingPodLp.sol:199; contracts/contracts/LendingAssetVault.sol:57; contracts/contracts/LendingAssetVault.sol:181; contracts/contracts/LendingAssetVault.sol:185; contracts/contracts/StakingPoolToken.sol:67; contracts/contracts/StakingPoolToken.sol:72; contracts/contracts/StakingPoolToken.sol:73; contracts/contracts/StakingPoolToken.sol:77; contracts/contracts/StakingPoolToken.sol:79; contracts/contracts/voting/VotingPool.sol:38; contracts/contracts/voting/VotingPool.sol:47; contracts/contracts/voting/VotingPool.sol:52; contracts/contracts/voting/VotingPool.sol:54; contracts/contracts/PodUnwrapLocker.sol:80; contracts/contracts/PodUnwrapLocker.sol:89; contracts/contracts/PodUnwrapLocker.sol:137; contracts/contracts/PodUnwrapLocker.sol:158

**Summary/Description**  
AutoCompoundingPodLp, LendingAssetVault, StakingPoolToken, VotingPool, and PodUnwrapLocker account user claims through internal fixed-amount ledgers rather than continuously reconciling actual custody balances. Direct surplus transfers and positive rebases are not credited to holders, while negative rebases of a configured wrapper or locked asset can leave the stored withdrawal amounts above the contract balance.

**Root Cause**  
These integrations separate custody from accounting but only normal deposit, stake, or lock entrypoints update the internal managed-asset state. totalAssets, sTKN supply, VotingPool stakes, and PodUnwrapLocker lock amounts are not synchronized to live token balances after balance drift, and there is no scoped sync/skim/socialization policy for surplus or deficit.

**Pre_conditions**  
The wrapper or locker asset is transferred directly to the contract, otherwise accrues there outside the normal accounting path, or is a rebasing token whose balance changes while held. In intended deployments the main staking token is a V2 LP token and aspTKN uses sTKN, both non-rebasing; rebasing pod assets and nonstandard generic voting assets remain outside the README token assumptions.

**Impact**  
Positive drift can remain in the contract without being redeemable by share holders, stakers, or lock owners. Negative drift in a rebasing configured asset can make later withdrawals or unstakes revert after earlier exits consume the remaining balance. This is kept at Info because confirmed supported assets are non-rebasing and the main positive-drift path is a direct transfer or unsupported-token integration mistake.

**Proof of Concept**  
No test run. Static path: transfer or positively rebase the wrapper asset directly into one of these contracts; totalAssets, stakes[user][asset].amtStaked, sTKN supply, or locks[lockId].amounts are unchanged, so a full exit pays only the tracked amount and leaves surplus. Conversely, if a configured held asset negatively rebases after amounts are recorded, the stored withdrawal amount remains nominal and later safeTransfer calls can fail once actual custody is insufficient.

**Mitigation**  
Add an explicit policy for surplus and deficit asset balances. Either reject rebasing wrapper and locker assets, provide a sync function that socializes balance drift to the correct holders, or add tightly scoped skim/rescue logic for balances above internally tracked obligations. Document unsupported direct transfers and rebasing assets if they are intentionally rejected.

### [I-53] PEAS paired-yield burn is paid to the fee owner

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/TokenRewards.sol:141; contracts/contracts/TokenRewards.sol:152; contracts/contracts/TokenRewards.sol:153; contracts/contracts/TokenRewards.sol:154; contracts/contracts/TokenRewards.sol:155; contracts/contracts/TokenRewards.sol:158; contracts/contracts/TokenRewards.sol:265; contracts/contracts/PEAS.sol:14

**Summary/Description**  
TokenRewards burns PEAS correctly when PEAS is the rewardsToken, but in leaveRewardsAsPairedLp mode the yieldBurn slice is added to the admin fee bucket and transferred as PAIRED_LP_TOKEN. If a pod leaves PEAS-denominated paired yield as the reward asset, the configured burn amount does not call PEAS.burn and PEAS totalSupply is not reduced.

**Root Cause**  
The LEAVE_AS_PAIRED_LP_TOKEN branch in depositFromPairedLpToken calculates _burnAmount from yieldBurn, adds it to _adminAmt, and calls _processAdminFee. That helper transfers PAIRED_LP_TOKEN to the owner. It bypasses _burnRewards, the only TokenRewards path that calls IPEAS(rewardsToken).burn and reduces PEAS totalSupply.

**Pre_conditions**  
A pod is configured with leaveRewardsAsPairedLp=true, PAIRED_LP_TOKEN set to PEAS, rewardsToken set to a different token to satisfy the PAIRED_LP_TOKEN != rewardsToken check, and protocolFees.yieldBurn > 0. Paired-token yield is then processed through depositFromPairedLpToken.

**Impact**  
The configured burn slice of PEAS-denominated paired yield is redirected to the protocol fee owner instead of reducing PEAS totalSupply. Stakers receive the same net post-fee amount, but PEAS holders do not get the intended supply reduction and the owner receives value that was accounted as burn. This is Info because the main deployed-style PEAS rewardsToken path burns correctly and the mismatch depends on a specific pod reward configuration.

**Proof of Concept**  
Static path: depositFromPairedLpToken computes _adminAmt, then in LEAVE_AS_PAIRED_LP_TOKEN mode computes _burnAmount = _amountTkn * yieldBurn / DEN. It executes _adminAmt += _burnAmount and _processAdminFee(_adminAmt), which transfers PAIRED_LP_TOKEN to the owner. For PAIRED_LP_TOKEN=PEAS this transfer leaves PEAS.totalSupply unchanged. The supply-reducing PEAS.burn path exists only in _burnRewards and is not reached by this branch.

**Mitigation**  
When the paired reward asset is PEAS, route the yieldBurn amount through IPEAS(PAIRED_LP_TOKEN).burn or a dedicated paired-token burn helper before depositing the remaining paired rewards. If non-burnable paired assets are supported, split burn and admin accounting explicitly rather than folding yieldBurn into _adminAmt.

### [I-54] SPTKN sqrt conversion can overflow on high-liquidity pools

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/voting/ConversionFactorSPTKN.sol:41; contracts/contracts/voting/ConversionFactorSPTKN.sol:43; contracts/contracts/voting/ConversionFactorSPTKN.sol:44; contracts/contracts/voting/VotingPool.sol:38; contracts/contracts/voting/VotingPool.sol:58; contracts/contracts/voting/VotingPool.sol:67

**Summary/Description**  
ConversionFactorSPTKN builds the square-root input as _pricePPeasNumX96 * _k, where _pricePPeasNumX96 is Q96-scaled and _k is the raw V2 reserve product. For 18-decimal LP reserves at a 1:1 Q96 price, symmetric reserves above roughly 1.21 million units each exceed uint256 before the sqrt helper is reached; the threshold is lower when the PEAS quote or pod CBR is higher. Those are valid uint112 reserve values, but getConversionFactor() reverts.

**Root Cause**  
The LP valuation multiplies a full Q96 price by raw reserves before any range reduction, decimal normalization, or 512-bit mulDiv-style operation. The sqrt helper itself floors correctly, but it never receives the input once the pre-sqrt product exceeds uint256.

**Pre_conditions**  
VotingPool enables an spTKN asset using ConversionFactorSPTKN. The spTKN staking token is a V2 LP whose raw reserves and PEAS quote make _pricePPeasNumX96 * reserve0 * reserve1 exceed uint256. This is most reachable for paired tokens with 18 decimals and large pool balances.

**Impact**  
stake() and update() for the enabled spTKN revert while the overflow condition persists, so users cannot enter or voluntarily refresh that voting position. Existing holders can still unstake through the cached factor path, and the stronger stale-factor reward impact is already covered separately, so this is recorded as an Info-level liveness/range issue.

**Proof of Concept**  
At _pricePPeasNumX96 = FixedPoint96.Q96 and reserve0 = reserve1 = 1,300,000e18, the expression _pricePPeasNumX96 * reserve0 * reserve1 is greater than type(uint256).max and reverts before _sqrt(). At 1,200,000e18 each it is still below the limit, showing the failure boundary is around normal high-liquidity 18-decimal reserve sizes rather than uint112 extremes.

**Mitigation**  
Normalize reserve units and price scale before multiplying, or use a full-precision mulDiv/range-reduced formulation for the LP invariant. Add boundary coverage for 18-decimal reserve products around the overflow threshold and for higher PEAS quote/CBR values.

### [I-55] WeightedIndex initialization overflows for 256+ constituents

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/WeightedIndex.sol:53-55; contracts/contracts/DecentralizedIndex.sol:48; contracts/contracts/IndexManager.sol:30-37; contracts/contracts/WeightedIndexFactory.sol:65-73

**Summary/Description**  
WeightedIndex accepts an uncapped uint256 token-array length from the deployment config, but __WeightedIndex_init iterates that length with a uint8 counter. A 256-or-more-token index reaches the checked _i++ increment at 255 and reverts during initialization.

**Root Cause**  
In contracts/contracts/WeightedIndex.sol:55 the loop counter is uint8 while _tl is _tokens.length as uint256. There is no explicit _tokens.length <= 255 validation in the deployment path, and DecentralizedIndex stores token indexes in a uint8 mapping.

**Pre_conditions**  
A creator or factory flow attempts to deploy a WeightedIndex with _tokens.length == _weights.length and at least 256 constituents.

**Impact**  
The affected pod deployment cannot complete. BeaconProxy construction and the factory call revert atomically, so no partially initialized pod state or direct fund loss was identified for this specific loop issue.

**Proof of Concept**  
Static path: IndexManager.deployNewIndex forwards baseConfig to WeightedIndexFactory.deployPodAndLinkDependencies, which constructs the WeightedIndex BeaconProxy with initialize calldata. initialize decodes _tokens and calls __WeightedIndex_init. For _tokens.length == 256, the loop body runs with _i == 255, then the checked uint8 post-increment attempts 256 and reverts.

**Mitigation**  
Use uint256 for the initialization loop counter, or add an explicit require(_tokens.length <= type(uint8).max) before the loop and document the maximum constituent count. If more than 255 constituents should be supported, _fundTokenIdx should also be widened.

### [I-56] Oversized oracle max-delay settings can revert freshness checks

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/oracle/ChainlinkSinglePriceOracle.sol:55-68; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:29-32; contracts/contracts/oracle/DIAOracleV2SinglePriceOracle.sol:32-36

**Summary/Description**  
The Chainlink, UniV3, and DIA single-price oracle freshness checks compute the stale boundary as block.timestamp - maxDelay before comparing the feed timestamp. If an owner-set feed-specific or default delay is greater than the current block timestamp, the subtraction underflows and the oracle call reverts instead of treating the data as fresh or bad.

**Root Cause**  
The freshness boundary is formed with unchecked-in-practice subtraction on block.timestamp rather than comparing as updatedAt + maxDelay < block.timestamp or guarding maxDelay <= block.timestamp. The max-delay setters do not enforce an upper bound.

**Pre_conditions**  
A trusted oracle owner sets defaultMaxOracleDelay or feedMaxOracleDelay to a value greater than the current chain timestamp, or otherwise deploys/configures the oracle with an oversized delay before a read path is used.

**Impact**  
Affected price reads revert until the delay is lowered. For Fraxlend markets using the affected single-price oracle through spTKN or aspTKN pricing, this can temporarily block oracle-dependent borrows, collateral removals, and liquidations. The owner can recover by setting a sane delay, and the trigger is trusted-configuration error, so this is Info.

**Proof of Concept**  
Static path: ChainlinkSinglePriceOracle.getPriceUSD18 calculates _isBadTimeQuote = block.timestamp - _maxDelayQuote and _isBadTimeBase = block.timestamp - _maxDelayBase before comparing updatedAt. UniswapV3SinglePriceOracle uses _updatedAt < block.timestamp - _maxDelay. DIAOracleV2SinglePriceOracle uses the same subtraction for its optional Chainlink base-conversion feed. With maxDelay > block.timestamp, Solidity 0.8 arithmetic reverts before the stale comparison executes.

**Mitigation**  
Compute freshness as updatedAt + maxDelay < block.timestamp, or explicitly handle maxDelay >= block.timestamp as non-stale. Add sane upper bounds and nonzero validation to delay setters so configuration mistakes cannot make oracle reads revert.

### [I-57] Scoped LinearInterestRate implements obsolete rate interface

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: fraxlend/src/contracts/LinearInterestRate.sol:28; fraxlend/src/contracts/LinearInterestRate.sol:32; fraxlend/src/contracts/LinearInterestRate.sol:74; fraxlend/src/contracts/interfaces/IRateCalculator.sol:11; fraxlend/src/contracts/interfaces/IRateCalculatorV2.sol:9; fraxlend/src/contracts/FraxlendPairCore.sol:81; fraxlend/src/contracts/FraxlendPairCore.sol:174; fraxlend/src/contracts/FraxlendPairCore.sol:405; fraxlend/src/contracts/FraxlendPair.sol:363; fraxlend/src/contracts/FraxlendPair.sol:367

**Summary/Description**  
LinearInterestRate is still in audit scope and documents a two-segment utilization curve, but it implements the legacy IRateCalculator interface with getNewRate(bytes,bytes). The active FraxlendPair stores and calls rateContract through IRateCalculatorV2, whose selector is getNewRate(uint256,uint256,uint64). A pair configured with the scoped LinearInterestRate cannot accrue or preview interest through the current pair code.

**Root Cause**  
The fork keeps both IRateCalculator and IRateCalculatorV2 in scope, but FraxlendPair was migrated to the V2 calculator interface while LinearInterestRate was not migrated or marked unsupported. setRateContract also accepts any address and only casts it to IRateCalculatorV2 without probing version() or the expected selector.

**Pre_conditions**  
A deployer or timelock configures a current FraxlendPair with the scoped LinearInterestRate address, either at deployment or through setRateContract. This is a trusted/operator configuration path, so the issue is classified as Info.

**Impact**  
Deployment with LinearInterestRate reverts when the constructor calls _addInterest, or an existing pair set to LinearInterestRate reverts on later paths that call _addInterest or previewAddInterest, including deposits, borrows, withdrawals, repayments, liquidations, ERC4626 previews, and pricePerShare. No untrusted path was found to force this configuration.

**Proof of Concept**  
LinearInterestRate exposes selector getNewRate(bytes,bytes) via IRateCalculator. FraxlendPairCore._calculateInterest calls IRateCalculatorV2(rateContract).getNewRate(_deltaTime, _utilizationRate, _currentRateInfo.fullUtilizationRate). Because LinearInterestRate has no V2 selector or fallback, the high-level call reverts. The constructor invokes _addInterest after assigning rateContract, so a pair deployed with this in-scope calculator fails immediately; setRateContract can also install the incompatible address on a live pair.

**Mitigation**  
Either remove/mark LinearInterestRate as legacy unsupported for current pairs, or migrate it to IRateCalculatorV2 by adding version() and getNewRate(uint256,uint256,uint64) with the fixed linear curve parameters stored immutably. Also validate rateContract compatibility before accepting deployment or setter config.

### [I-58] VariableInterestRate accepts invalid curve parameters

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fraxlend/src/contracts/VariableInterestRate.sol:70; fraxlend/src/contracts/VariableInterestRate.sol:120; fraxlend/src/contracts/VariableInterestRate.sol:127; fraxlend/src/contracts/VariableInterestRate.sol:155; fraxlend/src/contracts/VariableInterestRate.sol:159; fraxlend/src/contracts/VariableInterestRate.sol:160; fraxlend/src/contracts/VariableInterestRate.sol:174; fraxlend/src/contracts/FraxlendPairCore.sol:155; fraxlend/src/contracts/FraxlendPairCore.sol:167; fraxlend/src/contracts/FraxlendPair.sol:363; fraxlend/src/contracts/FraxlendPair.sol:367

**Summary/Description**  
VariableInterestRate stores all curve parameters without validating the ordering and range invariants that make the two-segment curve continuous and callable. The shipped script/test parameters are continuous around the target band and vertex, but the scoped contract itself accepts parameter sets such as min target above max target, vertex at or above 100%, or zero-utilization rate above the full-utilization bounds. A pair can also be deployed or migrated to such a calculator without checking these invariants.

**Root Cause**  
The constructor assigns MIN_TARGET_UTIL, MAX_TARGET_UTIL, VERTEX_UTILIZATION, ZERO_UTIL_RATE, MIN_FULL_UTIL_RATE, MAX_FULL_UTIL_RATE, RATE_HALF_LIFE, and VERTEX_RATE_PERCENT directly. getFullUtilizationInterest and getNewRate then assume min <= max <= UTIL_PREC, 0 < vertex < UTIL_PREC, zero rate <= full-utilization rates, RATE_HALF_LIFE > 0, and sane uint64-sized caps, while FraxlendPair only stores the calculator address and initial fullUtilizationRate without compatibility validation.

**Pre_conditions**  
A trusted deployer or timelock deploys VariableInterestRate with an invalid parameter set, or installs such a calculator through setRateContract. This is configuration/operator-driven; the deployed Peapods parameter set reviewed here does not show the discontinuity.

**Impact**  
Invalid curve parameters can make interest accrual and preview paths revert, or can return rates with a hard jump at a target boundary. For zero-divisor coverage, VERTEX_UTILIZATION == UTIL_PREC makes the high-utilization branch divide by UTIL_PREC - VERTEX_UTILIZATION at exactly 100% utilization. MAX_TARGET_UTIL == UTIL_PREC makes the growth-adjustment denominator zero if utilization ever exceeds 100%, which normal pair accounting tries to avoid but external capacity changes can make unsafe to assume at the calculator boundary. If MIN_TARGET_UTIL is zero, the low-utilization division is unreachable because no uint256 utilization is below zero, but the missing invariant checks still allow other non-callable or discontinuous target configurations. These failures can block deposits, borrows, repays, withdrawals, liquidations, and previews because state-changing pair paths use _addInterest.

**Proof of Concept**  
Code-level examples: construct VariableInterestRate with _vertexUtilization=100000; getNewRate(..., 100000, ...) reaches the else branch and divides by UTIL_PREC - VERTEX_UTILIZATION. Construct it with _maxUtil=100000; getFullUtilizationInterest divides by UTIL_PREC - MAX_TARGET_UTIL for any caller-supplied utilization above 100000, and pair integrations should not rely on the calculator accepting only bounded utilization. Construct it with _minUtil=85000 and _maxUtil=75000; utilization 84999 decays while utilization 85000 grows using (85000-75000)/(100000-75000), so the curve jumps at the boundary. Construct it with _zeroUtilizationRate greater than _minFullUtilizationRate; the clamped full-utilization rate can be lower than ZERO_UTIL_RATE and the vertex calculation underflows.

**Mitigation**  
Validate rate calculator parameters at construction and validate pair configuration before accepting a rate calculator. Enforce min target <= max target < UTIL_PREC, 0 < vertex < UTIL_PREC, vertex percent <= RATE_PREC, ZERO_UTIL_RATE <= MIN_FULL_UTIL_RATE <= MAX_FULL_UTIL_RATE <= type(uint64).max, RATE_HALF_LIFE > 0, and an initial fullUtilizationRate within the calculator's accepted range or explicitly clamped before use.

### [I-59] Raw ETH sent to pods cannot be recovered

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/DecentralizedIndex.sol:454

**Summary/Description**  
DecentralizedIndex exposes an unrestricted receive() hook but neither DecentralizedIndex nor WeightedIndex accounts for native ETH or provides a rescue path. Any raw ETH sent directly to a pod remains outside _totalAssets and cannot be withdrawn by holders or an authorized role.

**Root Cause**  
The pod base contract accepts native ETH even though all pod asset accounting is ERC20-based and there is no native-ETH forwarding, refund, sync, or rescue function.

**Pre_conditions**  
A user or integration sends native ETH directly to a pod address, or a WETH-style unwrap/transfer target is accidentally set to a pod.

**Impact**  
The sent ETH is stranded in the pod contract. No third-party extraction path was confirmed, so this is classified as Info for avoidable stuck value rather than shared-fund loss.

**Mitigation**  
Reject direct native ETH by reverting in receive(), or add a narrowly scoped native-ETH recovery/sync policy if pods are intended to accept raw ETH.

### [I-60] CCIP bridge gas limit accepts underfunded receiver settings

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/ccip/TokenRouter.sol:43; contracts/contracts/ccip/TokenBridge.sol:76; contracts/node_modules/@chainlink/contracts-ccip/src/v0.8/ccip/onRamp/EVM2EVMOnRamp.sol:433; contracts/node_modules/@chainlink/contracts-ccip/src/v0.8/ccip/offRamp/EVM2EVMOffRamp.sol:229

**Summary/Description**  
TokenRouter.setTargetChainGasLimit accepts any changed value, including zero or a value below the gas needed by TokenBridge._ccipReceive. TokenBridge encodes that value into CCIP EVMExtraArgsV1 for every bridge send. The local CCIP on-ramp validates only an upper bound, while the off-ramp manual-execution path exists specifically to retry messages that failed from insufficient gas.

**Root Cause**  
The bridge stores one global receiver gas limit without a lower bound, destination-specific sizing, or validation against the actual receive path cost.

**Pre_conditions**  
The trusted operator configures targetChainGasLimit below the destination receive cost, and a user bridges while that setting is active.

**Impact**  
Destination execution can fail and leave the bridge transfer pending until CCIP manual execution retries with enough gas. Because the setting is owner-only and the CCIP flow supports manual recovery, this is an operational liveness issue rather than confirmed permanent loss.

**Proof of Concept**  
Set targetChainGasLimit to 0 through TokenRouter.setTargetChainGasLimit, then call bridgeTokens. _buildMsg still encodes EVMExtraArgsV1({gasLimit: 0}) while carrying nonempty data for TokenBridge._ccipReceive. Chainlink validation rejects gas limits above maxPerMsgGasLimit but not low values; failed receiver execution can later be manually executed with a higher gas override.

**Mitigation**  
Enforce a nonzero minimum gas limit sized for TokenBridge._ccipReceive on each supported destination chain, preferably with per-chain configuration and a tested safety margin. Reject values below the measured minimum before accepting route configuration or sends.

### [I-61] Exact-output adapter balance mode reverts after spending input

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/dex/UniswapDexAdapter.sol:84; contracts/contracts/dex/UniswapDexAdapter.sol:91; contracts/contracts/dex/UniswapDexAdapter.sol:92; contracts/contracts/dex/UniswapDexAdapter.sol:104; contracts/contracts/dex/CamelotDexAdapter.sol:55; contracts/contracts/dex/CamelotDexAdapter.sol:62; contracts/contracts/dex/CamelotDexAdapter.sol:63; contracts/contracts/dex/CamelotDexAdapter.sol:74

**Summary/Description**  
swapV2SingleExactOut has a branch where _amountInMax == 0 means use the adapter's existing tokenIn balance as the swap budget. The function records that existing balance in _inBefore, lets the router spend from the same balance, then computes _inRemaining as balanceAfter - _inBefore. Any successful exact-output swap that spends a positive amount from the pre-existing balance leaves balanceAfter below _inBefore, so the refund/accounting subtraction reverts.

**Root Cause**  
The exact-output refund calculation always treats _inBefore as residual balance that should be preserved, but in the zero-input branch that same balance is the input budget being consumed by the swap. The function needs a different baseline when no new tokens are pulled from msg.sender.

**Pre_conditions**  
tokenIn has already been sent to the adapter, a caller invokes swapV2SingleExactOut with _amountInMax == 0, and the router spends a positive amount to satisfy _amountOut. I did not find an in-scope protocol caller currently passing zero for _amountInMax.

**Impact**  
The public adapter balance-spend mode for exact-output V2 swaps is unusable and reverts after a successful positive-input router swap. This can strand integrations that rely on the adapter's documented-by-code zero-amount pattern, but the current LVF caller passes a nonzero amountInMax, so the confirmed impact is integration liveness rather than active shared-fund loss.

**Proof of Concept**  
Let the adapter hold 100 tokenIn before the call. swapV2SingleExactOut stores _inBefore = 100 and sets _amountInMax = balanceOf(adapter) = 100. If the V2 router spends 40 tokenIn for the requested exact output, the adapter balance becomes 60. Line 104 computes 60 - 100 and reverts under Solidity 0.8 checked arithmetic.

**Mitigation**  
When _amountInMax == 0, set the refund baseline to zero or compute spent amount as preBalance - postBalance for the adapter-funded mode. Keep the current pre-existing-balance subtraction only for the branch that first pulls a fresh max amount from msg.sender.

### [I-62] Exact-output V2 swaps under-deliver fee-charging output tokens

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/dex/UniswapDexAdapter.sol:84; contracts/contracts/dex/UniswapDexAdapter.sol:101; contracts/contracts/dex/CamelotDexAdapter.sol:55; contracts/contracts/dex/CamelotDexAdapter.sol:72; contracts/contracts/DecentralizedIndex.sol:161; contracts/contracts/DecentralizedIndex.sol:168; contracts/contracts/DecentralizedIndex.sol:180

**Summary/Description**  
swapV2SingleExactOut uses the standard V2 swapTokensForExactTokens path. For a pTKN tokenOut with a nonzero buy fee, the pair sends the requested gross _amountOut, but DecentralizedIndex._update takes the buy fee before crediting the recipient. The router does not support or verify net fee-on-transfer output for exact-output swaps, and the adapter does not measure the recipient balance delta in this function, so callers can receive less than the requested exact output.

**Root Cause**  
The exact-output adapter path assumes router _amountOut equals the recipient's received amount. That assumption is false for output tokens whose transfer from the V2 pool applies a buy or transfer fee.

**Pre_conditions**  
A caller uses swapV2SingleExactOut with a fee-charging pTKN as _tokenOut, the pTKN has a nonzero buy fee or applicable transfer fee, and the V2 router can satisfy the requested gross output. I did not find an in-scope protocol caller currently using exact-output swaps with pTKN as tokenOut.

**Impact**  
The direct adapter caller or future integration can pay the exact-output input cost while receiving only the post-fee net amount. Current LVF removeLeverage uses pTKN as tokenIn and borrow/fTKN as tokenOut, so this is confirmed as adapter integration risk rather than an active protocol loss path.

**Proof of Concept**  
For a pTKN with a 5% buy fee, call swapV2SingleExactOut(pairedToken, pTKN, maxInput, 100e18, user). The V2 pair transfers 100e18 pTKN to user, but DecentralizedIndex._update classifies _from == V2_POOL as a buy and sends 5e18 pTKN to the pod fee balance, crediting only 95e18 to user. The router and adapter do not check the user's net balance delta for the exact-output path.

**Mitigation**  
Do not expose exact-output V2 swaps for fee-charging output tokens unless the adapter measures the recipient's net balance delta and enforces it against the requested output. For pTKN routes, prefer exact-input supporting-fee-on-transfer swaps with an explicit net minimum output.

### [I-63] Hardcoded Mean StaticOracle address blocks unsupported-chain DualOracle deployments

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: README.md:10; fraxlend/src/contracts/oracles/dual-oracles/DualOracleChainlinkUniV3.sol:141; fraxlend/src/contracts/FraxlendPairCore.sol:197; fraxlend/node_modules/@mean-finance/uniswap-v3-oracle/README.md:46; fraxlend/node_modules/@mean-finance/uniswap-v3-oracle/deploy/001_deploy.ts:7; fraxlend/node_modules/@mean-finance/uniswap-v3-oracle/deploy/001_deploy.ts:22; fraxlend/node_modules/@mean-finance/deterministic-factory/utils/deployment.ts:59

**Summary/Description**  
DualOracleChainlinkUniV3 hardcodes the Mean Finance StaticOracle address 0xB210CE856631EeEB767eFa666EC7C1C57738d438 instead of taking the static-oracle dependency as a constructor parameter. The contest target chains include Ethereum, Arbitrum One, Base, Mode, and Berachain, while the bundled Mean v1.0.3 deployment registry documents that address for Ethereum, Arbitrum, Optimism, Polygon, BNB, and testnets, and its deploy script has no Base, Mode, or Berachain chain-id entries. On a target chain without code at the hardcoded address, getPrices reverts when the interface call returns no data.

**Root Cause**  
The UniV3 helper oracle address is fixed in getPrices rather than being configured per deployment or validated in the constructor. The imported Mean deployment package uses a deterministic factory and salt for the same address, but the scoped package does not include deployment support or registry entries for every protocol target chain.

**Pre_conditions**  
A Fraxlend market is deployed with DualOracleChainlinkUniV3 on Base, Mode, Berachain, or another target chain where the Mean StaticOracle has not been deployed at 0xB210CE856631EeEB767eFa666EC7C1C57738d438 with compatible constructor parameters.

**Impact**  
DualOracleChainlinkUniV3 cannot be used for Fraxlend markets on those chains until the external dependency exists at the exact hardcoded address. FraxlendPairCore calls _updateExchangeRate in its constructor, so the pair deployment should revert before deposits or borrows occur; this makes the impact deployment liveness and portability rather than direct user fund loss.

**Proof of Concept**  
Code path: DualOracleChainlinkUniV3.getPrices always calls IStaticOracle(0xB210CE856631EeEB767eFa666EC7C1C57738d438).quoteSpecificPoolsWithTimePeriod. The package README registry lists Ethereum, Optimism, Arbitrum, Polygon, BNB, and testnets, but the contest README lists Base, Mode, and Berachain as target chains too. The Mean deploy script maps only chain ids 1, 3, 4, 5, 42, 10, 56, 69, 42161, 421611, 137, and 80001 for factory/cardinality arguments. If the hardcoded address has no code, the external staticcall succeeds with empty return data and Solidity ABI decoding reverts. FraxlendPairCore constructor calls _updateExchangeRate, which calls the oracle, so affected pair deployment reverts.

**Mitigation**  
Make the StaticOracle address an immutable constructor parameter and validate it with a code-size or interface check, or deploy and verify the Mean StaticOracle at the same address on every intended chain before enabling DualOracleChainlinkUniV3 markets. Add deployment-time checks that the static oracle exists and exposes the expected Uniswap V3 factory/cardinality for the current chain.

### [I-64] V3 multi-hop zaps bypass the configured DEX adapter

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/Zapper.sol:26; contracts/contracts/Zapper.sol:119; contracts/contracts/Zapper.sol:123; contracts/contracts/Zapper.sol:125; contracts/contracts/Zapper.sol:182; contracts/contracts/Zapper.sol:192; contracts/contracts/Zapper.sol:194; contracts/contracts/dex/AerodromeDexAdapter.sol:82; contracts/contracts/dex/AerodromeDexAdapter.sol:97; contracts/contracts/dex/AerodromeDexAdapter.sol:99; contracts/contracts/dex/AerodromeDexAdapter.sol:101; contracts/contracts/dex/CamelotDexAdapter.sol:95; contracts/contracts/dex/CamelotDexAdapter.sol:96; contracts/contracts/dex/CamelotDexAdapter.sol:100

**Summary/Description**  
The Zapper single-hop V3 path routes through the configured IDexAdapter, but the two-hop V3 path bypasses the adapter and calls the hardcoded Uniswap V3 SwapRouter address with a Uniswap fee-encoded path. This is only correct for Uniswap-style two-hop routes. For Aerodrome Slipstream routes, the intended execution path is the adapter Universal Router command with tick-spacing encoding; for Camelot Algebra routes, the intended router/path format is Algebra-specific. A correctly configured non-Uniswap V3 two-hop zap can therefore revert or execute against unrelated Uniswap-family pools instead of the configured route pools.

**Root Cause**  
Zapper stores adapter-specific DEX state but _swapV3Multi does not use DEX_ADAPTER or DEX_ADAPTER.V3_ROUTER. It builds abi.encodePacked(token, fee, token, fee, token) and calls the constant V3_ROUTER. The pool type enum only says V3 and does not distinguish Uniswap fee-tier pools from Algebra or Slipstream pools, while _getPoolFee also returns a value that is not sufficient to encode non-Uniswap multi-hop routes.

**Pre_conditions**  
The owner configures a zapMap entry with poolType V3 and pool2 set for a two-hop route. The deployment uses an Aerodrome or Camelot adapter, or otherwise expects the route to use the configured adapter pools rather than Uniswap V3 fee-tier pools. A user calls a zap flow such as IndexUtils.addLPAndStake with _pairedLpTokenProvided different from the pod paired token.

**Impact**  
Affected two-hop V3 zaps can be unusable on supported non-Uniswap deployments, or can trade through a different AMM family if a compatible hardcoded router and fee-tier pools exist. With a strict user min-out this is primarily a repeatable route failure. With a loose or zero min-out, the value-loss side overlaps the existing multi-hop zero-min finding, because the wrong route can still satisfy the call at an unexpected execution price.

**Proof of Concept**  
In _zap, a V3 Pools entry with pool2 set reaches _swapV3Multi. The helper approves the hardcoded V3_ROUTER constant and calls ISwapRouter.exactInput with a Uniswap path built from _getPoolFee(pool1) and _getPoolFee(pool2). It never calls DEX_ADAPTER.swapV3Single, DEX_ADAPTER.V3_ROUTER, IAerodromeUniversalRouter.execute, or ISwapRouterAlgebra.exactInput. On an Aerodrome route, the configured single-hop adapter would encode tokenIn,tickSpacing,tokenOut and call the Universal Router, but the multi-hop helper instead passes fee-style bytes to the hardcoded Uniswap router. On Camelot, the adapter single-hop uses the Algebra router params, while the multi-hop helper still uses the Uniswap router interface and fee path.

**Mitigation**  
Route all V3 swaps through adapter-specific code. Add a multi-hop swap function to IDexAdapter, or store an explicit route type and router/path encoder per adapter. For Aerodrome, encode tick-spacing paths and call the configured Universal Router. For Camelot, call the configured Algebra router with its path format. Reject multi-hop V3 routes when the adapter cannot execute that route family.

### [I-65] Nominal Fraxlend collateral accounting can over-credit taxed or rebasing collateral

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fraxlend/src/contracts/FraxlendPairCore.sol:225; fraxlend/src/contracts/FraxlendPairCore.sol:940; fraxlend/src/contracts/FraxlendPairCore.sol:947; fraxlend/src/contracts/FraxlendPairCore.sol:975; fraxlend/src/contracts/FraxlendPairCore.sol:984; fraxlend/src/contracts/FraxlendPairCore.sol:1100

**Summary/Description**  
FraxlendPair keeps borrower collateral in userCollateralBalance and totalCollateral as a nominal ledger rather than reconciling it to the collateral token balance. Initial deposits are credited by the requested amount, and later rebases are never synchronized. A fee-on-transfer or negative-rebasing collateral can therefore leave collateral accounting above actual custody, while a positive rebase leaves unallocated surplus collateral in the pair.

**Root Cause**  
The collateral path assumes exact-transfer, non-rebasing collateral. _addCollateral increments userCollateralBalance and totalCollateral before safeTransferFrom and does not measure the received balance delta. After collateral is held, _isSolvent, removeCollateral, and liquidation use the stored nominal amounts without checking collateralContract.balanceOf(address(this)) or applying any rebase socialization policy.

**Pre_conditions**  
A Fraxlend pair is deployed with collateral that can transfer less than requested or whose balances can rebase while held by the pair. The intended LVF path uses aspTKN collateral, which is a non-rebasing internal ERC20, and the README says nonstandard tokens are not formally supported, so this is an integration/configuration risk.

**Impact**  
The pair can overstate borrower collateral and total collateral. Negative drift lets early collateral removals or liquidations consume the remaining real balance while later borrowers or liquidators face failed transfers or bad solvency accounting. Positive drift is not assigned to any borrower and can remain surplus. Under the documented LVF setup and no-nonstandard-token assumption this remains Info.

**Proof of Concept**  
Static trace: borrowAsset and addCollateral route through _addCollateral. The helper adds _collateralAmount to userCollateralBalance and totalCollateral, then pulls _collateralAmount from the sender. If the token delivers less, or later negatively rebases the pair balance, the stored ledger remains at the larger nominal amount. Solvency checks and liquidation collateral calculations continue to use that nominal ledger until a transfer fails or another user absorbs the shortfall.

**Mitigation**  
Reject rebasing and fee-on-transfer collateral at pair deployment, or measure collateral balance deltas and credit only received amounts. If rebasing collateral must be supported, add an explicit sync/socialization rule before solvency checks, collateral removals, and liquidations so positive and negative rebases are allocated predictably.

### [I-66] TWAP accumulator averaging corrupts oversized intervals

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: contracts/contracts/twaputils/V3TwapUtilities.sol:86-90; contracts/contracts/twaputils/V3TwapAerodromeUtilities.sol:89-93; contracts/contracts/twaputils/V3TwapCamelotUtilities.sol:82-86; contracts/contracts/twaputils/V3TwapKimUtilities.sol:85-89; contracts/contracts/oracle/UniswapV3SinglePriceOracle.sol:21,57-61; contracts/contracts/oracle/spTKNMinimalOracle.sol:281-285

**Summary/Description**  
The V3 TWAP utilities and UniV3 single-price oracle derive the arithmetic mean tick from cumulative tick data, but divide the int56 accumulator delta by int32(_interval). A uint32 interval above 2^31 - 1 is reinterpreted as a negative int32 divisor, which can flip or distort the mean tick once the underlying pool/plugin can serve that lookback. The public UniV3 helper also truncates a uint256 _twapInterval to uint32 before the accumulator read, so oversized helper calls can wrap to a shorter interval or zero.

**Root Cause**  
Accumulator deltas are int56, while the elapsed seconds value is narrowed to int32 instead of being widened to a signed type that preserves every uint32 interval value. The spTKN oracle setter only checks _interval > 0 and does not bound the configured TWAP interval to the signed divisor range.

**Pre_conditions**  
A TWAP read is made with _interval > type(int32).max and the target pool/plugin has an old enough observation window for that secondsAgo value, or an external caller uses UniswapV3SinglePriceOracle.getPriceUSD18 with a _twapInterval that truncates when cast to uint32. In normal in-scope spTKN/aspTKN flows, this depends on trusted owner configuration and very old pool history.

**Impact**  
The oracle can compute a materially wrong mean tick or, for uint256-to-uint32 truncation, use an unintended shorter/spot interval. This can misprice dependent helper reads and future oracle configurations; current protocol impact is limited because configured intervals default to 10 minutes, the setter is owner-only, and pools deployed today do not have multi-decade observation history.

**Proof of Concept**  
Static path: each affected TWAP implementation sets secondsAgo[0] = _interval, reads tickCumulatives, then computes int24(tickCumulativesDelta / int32(_interval)). For _interval = 2^31 and an average tick of -1000 over that period, tickCumulativesDelta is negative, but int32(_interval) is -2147483648, making the quotient positive. The mean tick sign is therefore corrupted before TickMath.getSqrtRatioAtTick is called.

**Mitigation**  
Use a signed divisor that preserves the full uint32 range, e.g. int56(int256(uint256(_interval))), for both division and modulo. Also bound owner-configurable TWAP intervals to a practical maximum and validate or reject uint256 helper intervals that exceed type(uint32).max.

### [I-67] Zero comptroller initialization permanently disables Fraxlend owner controls

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fraxlend/src/contracts/FraxlendPairAccessControl.sol:64; fraxlend/src/contracts/FraxlendPairAccessControl.sol:67; fraxlend/src/contracts/FraxlendPairDeployer.sol:95; fraxlend/src/contracts/FraxlendPairDeployer.sol:210; fraxlend/src/contracts/FraxlendPairDeployer.sol:301; fraxlend/src/contracts/FraxlendPair.sol:452; fraxlend/src/contracts/FraxlendPair.sol:480

**Summary/Description**  
FraxlendPair ownership is assigned from the deployer-supplied comptroller address during construction. FraxlendPairDeployer accepts that address from constructor configuration and later setComptroller without rejecting address(0), so future pairs can be deployed with owner() permanently set to zero.

**Root Cause**  
FraxlendPairAccessControl calls _transferOwnership(_comptrollerAddress) directly during construction, and the deployer stores and forwards comptrollerAddress without a nonzero validation. OpenZeppelin internal _transferOwnership permits zero, unlike the external transferOwnership guard.

**Pre_conditions**  
A trusted deployment or deployer-owner configuration passes address(0) as the Fraxlend comptroller before deploying one or more pairs. This is a trusted configuration path, so it is classified as Info.

**Impact**  
Affected pairs cannot recover the Ownable role, so owner-only controls such as withdrawFees and setSwapper are unavailable for the lifetime of the pair. Timelock and circuit-breaker paths remain available if separately configured, but pair owner administration and fee withdrawal are permanently disabled.

**Proof of Concept**  
Code path: FraxlendPairDeployer is constructed with _params.comptroller == address(0), or the deployer owner calls setComptroller(address(0)). Later deploy() builds immutables with that zero comptroller. FraxlendPairAccessControl decodes the value and calls _transferOwnership(address(0)) in the pair constructor. Since msg.sender can never be address(0), onlyOwner functions on that pair cannot be called and ownership cannot be transferred back.

**Mitigation**  
Reject address(0) for comptrollerAddress in the deployer constructor and setComptroller, and consider validating the value again in FraxlendPairAccessControl before calling _transferOwnership. If owner renounce is intended, expose it as an explicit post-deployment action rather than allowing accidental zero-owner initialization.

### [I-68] Fraxlend maxDeposit and maxMint ignore uint128 storage ceiling

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fraxlend/src/contracts/FraxlendPairAccessControl.sol:46; fraxlend/src/contracts/FraxlendPair.sol:238; fraxlend/src/contracts/FraxlendPair.sol:244; fraxlend/src/contracts/FraxlendPair.sol:248; fraxlend/src/contracts/FraxlendPairCore.sol:624; fraxlend/src/contracts/FraxlendPairCore.sol:637; fraxlend/src/contracts/FraxlendPairCore.sol:640; fraxlend/src/contracts/FraxlendPairCore.sol:693; fraxlend/src/contracts/FraxlendPairCore.sol:703; fraxlend/src/contracts/FraxlendPairCore.sol:709; fraxlend/src/contracts/libraries/VaultAccount.sol:6; fraxlend/src/contracts/libraries/VaultAccount.sol:25; fraxlend/src/contracts/libraries/VaultAccount.sol:29

**Summary/Description**  
FraxlendPair.maxDeposit and maxMint model capacity from the uint256 depositLimit even though the matching deposit and mint paths later cast assets and shares to uint128 and store them in a uint128 VaultAccount. With the default unpaused depositLimit of type(uint256).max, maxDeposit can advertise amounts above type(uint128).max or above the remaining uint128 accounting headroom. Once the market is non-empty, maxMint can also revert in the view itself because it passes a near-uint256.max amount into VaultAccount.toShares(), which multiplies that amount by totalAsset.shares before any uint128 clamp.

**Root Cause**  
The ERC4626 max functions only model depositLimit and current totalAsset amount. They do not clamp by the uint128 amount/share storage bounds that are enforced later by toUint128 casts and checked uint128 additions in _deposit, and maxMint performs the shares conversion before applying any storage-width ceiling.

**Pre_conditions**  
The pair is unpaused or otherwise has depositLimit larger than the remaining uint128 accounting capacity. A user or integration queries maxDeposit or maxMint and then attempts to use a value above the amount/share that can be represented in the pair's uint128 VaultAccount.

**Impact**  
The direct impact is ERC4626 standards non-compliance and integration liveness risk: callers can be told that a deposit or mint size is accepted when the matching call reverts, and maxMint can itself revert after ordinary deposits despite ERC4626 max functions being expected to be conservative views. This is Info because the unreachable range is extremely large for normal token supplies and no fund-loss path was confirmed.

**Proof of Concept**  
Fresh pair example: depositLimit defaults to type(uint256).max and totalAsset.amount is zero, so maxDeposit(receiver) returns type(uint256).max. deposit(type(uint128).max + 1, receiver) passes the depositLimit check and computes shares equal to assets, then reverts at _amount.toUint128() or _sharesReceived.toUint128(). Non-empty maxMint example: after a user deposits 2 assets and receives 2 shares, maxMint(receiver) computes _maxDeposit = type(uint256).max - 2 and calls toShares(_maxDeposit, false); toShares evaluates _maxDeposit * totalAsset.shares, so multiplying by 2 overflows and the view reverts before returning a maximum.

**Mitigation**  
Clamp maxDeposit to the minimum of the configured remaining depositLimit and the remaining uint128 amount capacity, and clamp maxMint by both the converted deposit cap and the remaining uint128 share capacity. Compute maxMint through bounded values so the view cannot overflow before returning. The mutable paths should keep explicit zero/overflow guards close to the max-function model so advertised ERC4626 limits stay conservative.

### [I-69] Fraxlend interest overflow guard can discard accrued interest

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fraxlend/src/contracts/FraxlendPairCore.sol:388; fraxlend/src/contracts/FraxlendPairCore.sol:398; fraxlend/src/contracts/FraxlendPairCore.sol:410; fraxlend/src/contracts/FraxlendPairCore.sol:413; fraxlend/src/contracts/FraxlendPairCore.sol:415; fraxlend/src/contracts/FraxlendPairCore.sol:416; fraxlend/src/contracts/FraxlendPairCore.sol:463; fraxlend/src/contracts/FraxlendPairCore.sol:465; fraxlend/src/contracts/FraxlendPairCore.sol:481; fraxlend/src/contracts/FraxlendPairCore.sol:486; fraxlend/src/contracts/FraxlendPairCore.sol:487; fraxlend/src/contracts/libraries/VaultAccount.sol:16; fraxlend/src/contracts/libraries/VaultAccount.sol:20; contracts/contracts/LendingAssetVault.sol:73; contracts/contracts/LendingAssetVault.sol:77

**Summary/Description**  
FraxlendPair._calculateInterest computes interest in uint256 but only applies it if the resulting borrow amount and asset capacity fit within the uint128 VaultAccount storage envelope. If either guard fails, the function leaves totalAsset and totalBorrow unchanged while _addInterest still treats the checkpoint as updated, returns/emits the nonzero interestEarned value, and advances currentRateInfo.lastTimestamp. The elapsed interest is therefore discarded rather than accrued or retried.

**Root Cause**  
The overflow guard is implemented as a conditional around only the total updates, not around the whole checkpoint. isInterestUpdated is set before the uint128 capacity check, and _addInterest updates the timestamp and rate state whenever isInterestUpdated is true even if the uint128-bounded accounting writes were skipped. The second guard also compares interest against totalAsset.totalAmount(externalAssetVault), which can include uint256 LAV available capacity rather than only the uint128 stored totalAsset.amount that is actually incremented.

**Pre_conditions**  
A pair has nonzero borrow debt and an interest checkpoint where interestEarned + totalBorrow.amount exceeds type(uint128).max, or where an attached LAV exposes enough available allocation that interestEarned + totalAsset.totalAmount(address(externalAssetVault)) exceeds type(uint128).max. Reaching this requires extremely large token balances, debt, or LAV allocation relative to normal 18-decimal asset supplies, so likelihood is low.

**Impact**  
Borrowers stop being charged the elapsed interest once the cap condition is hit, while lenders and the protocol do not receive the corresponding asset/share growth. Because the timestamp is advanced, later checkpoints only consider future time and cannot recover the discarded interval. The effect can repeat while the pair remains near or above the uint128 envelope, but it requires unusually large market scale or configuration.

**Proof of Concept**  
No test run. Code path: _calculateInterest sets _results.isInterestUpdated = true, computes _results.interestEarned, then only mutates _results.totalBorrow.amount and _results.totalAsset.amount inside the uint128-capacity if block. If interestEarned is positive but interestEarned + totalBorrow.amount > type(uint128).max, or interestEarned + _totalAssetsAvailable > type(uint128).max, the block is skipped. _addInterest still copies _results.interestEarned into the return/event values and writes currentRateInfo.lastTimestamp = block.timestamp while storing unchanged totalAsset and totalBorrow, so the elapsed interest is lost.

**Mitigation**  
When the uint128 accounting envelope would be exceeded, revert or pause accrual instead of advancing the checkpoint with unchanged totals. Apply capacity checks to the exact stored fields being mutated, and clamp LAV allocations or pair limits so external capacity cannot make a uint128 storage guard fail without the stored totalAsset.amount itself being near the cap.

### [I-70] Nested vault wrapper routes do not enforce final share minimums

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/Zapper.sol:77; contracts/contracts/Zapper.sol:84; contracts/contracts/Zapper.sol:138; contracts/contracts/Zapper.sol:140; contracts/contracts/Zapper.sol:228; contracts/contracts/Zapper.sol:232; contracts/contracts/IndexUtils.sol:51; contracts/contracts/IndexUtils.sol:72; contracts/contracts/IndexUtils.sol:73; contracts/contracts/lvf/LeverageManager.sol:471; contracts/contracts/lvf/LeverageManager.sol:522; contracts/contracts/lvf/LeverageManager.sol:534; contracts/contracts/lvf/LeverageManager.sol:540

**Summary/Description**  
IndexUtils.addLPAndStake exposes amountPairedLpTokenMin as the minimum paired token received when the provided token must be zapped into the pod's paired LP token. Most Zapper routes pass that value to the final swap, but the special pOHM and stYETH routes use it only before an additional vault-like conversion: pOHM first swaps into OHM and then calls IDecentralizedIndex(pOHM).bond(OHM, amountOut, 0), while stYETH first swaps WETH into yETH and then calls IERC4626(STYETH).deposit(boughtYeth, address(this)). LeverageManager has the same shape in the podded self-lending add path: borrowed assets are deposited into the Fraxlend pair and then bonded into the paired wrapper pod with amountMintMin = 0 before LP slippage is applied. In these branches, the caller cannot express a minimum final wrapper-share amount.

**Root Cause**  
The wrapper helpers reuse a single min-out parameter for an intermediate swap output and then hardcode zero for the subsequent vault/pod share mint. The downstream add-liquidity slippage is computed from the actual post-conversion balances, so it does not restore the missing final-share floor.

**Pre_conditions**  
A route uses Zapper's pOHM or stYETH special case, or an LVF addLeverage flow uses a podded self-lending paired wrapper. The final wrapper share conversion rate, fee, or CBR differs from the caller's quote before the transaction executes.

**Impact**  
The transaction can continue with fewer final paired wrapper shares than the caller intended. This can reduce the amount of pod LP/staking exposure created or make quoting/UI slippage checks misleading. A direct value-loss path depends on the external wrapper economics and market pricing, so this is classified as Info rather than a standalone Medium finding.

**Proof of Concept**  
For pOHM, IndexUtils.addLPAndStake passes amountPairedLpTokenMin into _zap. When pOHM == _out, _zap swaps the input into OHM using that minimum, then calls IDecentralizedIndex(pOHM).bond(OHM, _amountOut, 0). Any lower-than-quoted pOHM mint still proceeds. For stYETH, _wethToYeth enforces the minimum on yETH from Curve and then deposits into STYETH without checking the stYETH shares minted. For podded self-lending, LeverageManager deposits borrowed assets into the Fraxlend pair and calls IDecentralizedIndex(_finalPairedTkn).bond(_lendingPair, _finalPairedAmt, 0), with no user-supplied minimum for the paired wrapper pTKN amount.

**Mitigation**  
For each nested wrapper route, add a minimum for the final share token or reinterpret the existing parameter as a final-output minimum and check the post-conversion balance delta. For LVF self-lending, include a minimum paired wrapper pTKN or minimum aspTKN collateral amount in the addLeverage config before borrowing against the result.

### [I-71] Empty LAV can make attached Fraxlend deposits revert

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: fraxlend/src/contracts/FraxlendPairCore.sol:597; fraxlend/src/contracts/FraxlendPairCore.sol:600; fraxlend/src/contracts/FraxlendPairCore.sol:603; contracts/contracts/LendingAssetVault.sol:57; contracts/contracts/LendingAssetVault.sol:136; contracts/contracts/LendingAssetVault.sol:152; contracts/contracts/LendingAssetVaultFactory.sol:32; contracts/contracts/LendingAssetVaultFactory.sol:35

**Summary/Description**  
FraxlendPairCore._deposit tries to return newly supplied assets to the external LendingAssetVault when the vault is over-utilized or the pair is over allocation. That check computes totalAssetsUtilized() / totalAssets() on the external LAV. LendingAssetVault factory seed shares are minted to the creator and can be fully redeemed while there is no utilization, leaving totalAssets() equal to zero. Once such an empty LAV is attached to a Fraxlend pair, ordinary pair deposits above the 0.1% threshold reach the over-utilization check and revert on division by zero.

**Root Cause**  
The pair assumes an attached externalAssetVault always has nonzero totalAssets when _deposit performs the utilization-ratio check. The LAV does not enforce permanent seed liquidity: minimumDepositAtCreation is redeemable by the seed receiver through withdraw/redeem, and totalAssets can legitimately return zero.

**Pre_conditions**  
A Fraxlend pair has externalAssetVault set to a LendingAssetVault. The LAV has no assets and no shares, for example because the factory seed receiver redeemed the initial seed before meaningful LAV deposits. A user deposits or mints into the Fraxlend pair with an amount greater than totalAsset.amount / 1000 after the deposit accounting update.

**Impact**  
The affected Fraxlend pair rejects normal deposits/mints that cross the auto-return threshold until someone deposits any nonzero amount into the LAV. This can delay market bootstrapping or recovery after a fully emptied LAV. It is classified Info because the condition is easy to repair permissionlessly by reseeding the LAV, and an attacker generally must control the last LAV shares to create the zero-total state.

**Proof of Concept**  
No test run. Static sequence: LendingAssetVaultFactory.create deposits minimumDepositAtCreation and mints LAV shares to the caller. With no utilization, that holder can call LAV.redeem(allShares, holder, holder), which sets _totalAssets to zero. If the pair's externalAssetVault points to this LAV, FraxlendPair.deposit(amount, receiver) calls _deposit; for any amount satisfying amount > totalAsset.amount / 1000, _deposit calls externalAssetVault.whitelistUpdate(true) and then evaluates 1e18 * totalAssetsUtilized() / totalAssets(). Since totalAssets() is zero, the deposit reverts before the new lender can supply the pair.

**Mitigation**  
Handle the zero-total LAV case before computing the utilization ratio. For example, treat totalAssets == 0 as not over-utilized, or skip the auto-return check unless both totalAssets and totalAssetsUtilized are nonzero. If the protocol expects permanent LAV seed liquidity, mint the creation seed to a non-redeemable address or enforce a minimum total supply.

### [I-72] VotingPool overcredits transfer-taxed pTKN stakes

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: contracts/contracts/voting/VotingPool.sol:38; contracts/contracts/voting/VotingPool.sol:40; contracts/contracts/voting/VotingPool.sol:43; contracts/contracts/voting/VotingPool.sol:47; contracts/contracts/voting/VotingPool.sol:51; contracts/contracts/voting/VotingPool.sol:53; contracts/contracts/DecentralizedIndex.sol:159; contracts/contracts/DecentralizedIndex.sol:174; contracts/contracts/DecentralizedIndex.sol:177; contracts/contracts/voting/ConversionFactorPTKN.sol:13; contracts/contracts/voting/ConversionFactorSPTKN.sol:29

**Summary/Description**  
VotingPool supports pTKN and spTKN-style assets, but stake records the caller-supplied _amount after a raw ERC20 transfer rather than the balance delta actually received. If an enabled pod token has config.hasTransferTax, its DecentralizedIndex._update deducts the transfer tax when the token is moved into VotingPool. VotingPool still adds the full nominal _amount to the user's stake and mints reward/voting shares from that amount, leaving the pool undercollateralized for that asset.

**Root Cause**  
VotingPool assumes enabled assets transfer exactly. In-scope pod tokens can charge a transfer tax on ordinary non-pool transfers, and VotingPool does not measure balanceBefore/balanceAfter or reject taxed pod tokens before using the nominal input amount for stake accounting.

**Pre_conditions**  
The VotingPool owner enables a pTKN or spTKN whose underlying pod has config.hasTransferTax set. A user stakes that token into VotingPool through stake.

**Impact**  
The staker receives vlPEAS and VotingPool reward shares for more pTKN/spTKN than VotingPool actually received. A full unstake can revert when the contract lacks the nominal balance, or later stakers can subsidize the shortfall. Partial exits can leave residual unbacked voting/reward shares corresponding to the transfer-tax gap. This is classified as Info because the built-in pTKN transfer tax is small and the setup depends on enabling taxed pod tokens as voting assets, but it is a code-level accounting mismatch under supported pTKN assets.

**Proof of Concept**  
No test run. Static path: a taxed pTKN transfer of amount A from a user to VotingPool enters DecentralizedIndex._update, which takes A / 10000 as transfer tax and credits VotingPool with A - tax. VotingPool.stake then calls _updateUserState with _addAmt = A, increasing stakes[user][pTKN].amtStaked by A and minting vlPEAS from A. Later unstake burns shares and asks VotingPool to transfer the nominal _amtStakeToRemove, even though the contract never received that full amount and the outgoing pTKN transfer can be taxed again.

**Mitigation**  
For each stake, measure the actual asset balance delta received and use that amount for stake accounting, or reject pTKN/spTKN assets whose transfers are taxed. On unstake, account for net transfer behavior or require exact-transfer assets so VotingPool stake accounting remains fully backed.

### [I-73] Self-lending aspTKN strands borrow-asset primary rewards

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: contracts/contracts/AutoCompoundingPodLp.sol:217; contracts/contracts/AutoCompoundingPodLp.sol:220; contracts/contracts/AutoCompoundingPodLp.sol:249; contracts/contracts/AutoCompoundingPodLp.sol:260; contracts/contracts/AutoCompoundingPodLp.sol:264; contracts/contracts/AutoCompoundingPodLp.sol:272; contracts/contracts/AutoCompoundingPodLp.sol:279; contracts/contracts/AutoCompoundingPodLp.sol:295; contracts/contracts/AutoCompoundingPodLp.sol:303

**Summary/Description**  
For self-lending pods, AutoCompoundingPodLp changes the intended paired-token output to the Fraxlend pair asset before depositing into the pair. If pod.lpRewardsToken is already that borrow asset, the primary reward branch still treats it as a token that must be swapped and calls the V3 adapter with tokenIn equal to tokenOut. That call is caught, so deposits and withdrawals do not revert, but the reward balance remains idle in the aspTKN and is never deposited into the lending pair, staked as LP, or counted in totalAssets.

**Root Cause**  
The self-lending conversion path only has an early return for token == pod.PAIRED_LP_TOKEN. It does not handle token == IFraxlendPair(pod.PAIRED_LP_TOKEN()).asset() before the primary lpRewardsToken branch, so an already-correct borrow-asset reward is routed through a same-token V3 swap instead of _depositIntoLendingPair.

**Pre_conditions**  
A self-lending aspTKN is configured for a pod whose PAIRED_LP_TOKEN is a Fraxlend pair, and pod.lpRewardsToken is set to the Fraxlend pair asset, for example DAI for an fDAI paired pod. The aspTKN receives that primary reward token through a claim or direct balance, and yieldConvEnabled is true.

**Impact**  
The affected primary rewards are not compounded into the aspTKN asset and are not included in _totalAssets. Holders lose the benefit of those rewards while they sit in the vault contract, and there is no token-specific rescue or alternate conversion path in AutoCompoundingPodLp. This is classified as Info because the repository examples use PEAS as lpRewardsToken and the issue depends on an avoidable reward-token configuration.

**Proof of Concept**  
No test run. Static path: _processRewardsToPodLp scans getAllRewardsTokens plus pod.lpRewardsToken and reaches the borrow-asset lpRewardsToken balance. _tokenToPairedLpToken sets _swapOutputTkn = IFraxlendPair(pod.PAIRED_LP_TOKEN()).asset() because IS_PAIRED_LENDING_PAIR is true. Since _token equals pod.lpRewardsToken, the code skips the secondary-token branch and calls DEX_ADAPTER.swapV3Single(_rewardsToken, _swapOutputTkn, ...). With _rewardsToken == _swapOutputTkn, the adapter/router path fails and the catch only records _tokenToPairedSwapAmountInOverride and emits TokenToPairedLpSwapError. The function returns zero, _depositIntoLendingPair is never called, and _totalAssets is not increased.

**Mitigation**  
In the self-lending branch, handle token == IFraxlendPair(pod.PAIRED_LP_TOKEN()).asset() before the primary/secondary split by depositing that asset directly into the Fraxlend pair. Also reject self-lending pod configurations where lpRewardsToken equals the borrow asset unless direct handling is implemented, and add a recovery path for idle reward balances.

### [I-74] Nominal lending-asset accounting overcredits non-exact or rebasing assets

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: fraxlend/src/contracts/FraxlendPairCore.sol:210; fraxlend/src/contracts/FraxlendPairCore.sol:586; fraxlend/src/contracts/FraxlendPairCore.sol:595; fraxlend/src/contracts/FraxlendPairCore.sol:745; fraxlend/src/contracts/FraxlendPairCore.sol:772; fraxlend/src/contracts/FraxlendPairCore.sol:1028; fraxlend/src/contracts/FraxlendPairCore.sol:1040; contracts/contracts/LendingAssetVault.sol:57; contracts/contracts/LendingAssetVault.sol:164; contracts/contracts/LendingAssetVault.sol:166; contracts/contracts/LendingAssetVault.sol:181; contracts/contracts/LendingAssetVault.sol:185; contracts/contracts/LendingAssetVault.sol:238; contracts/contracts/LendingAssetVault.sol:249; contracts/contracts/LendingAssetVault.sol:252; contracts/contracts/LendingAssetVault.sol:313; contracts/contracts/LendingAssetVault.sol:314

**Summary/Description**  
FraxlendPair and LendingAssetVault account for lending assets with cached nominal amounts rather than live token balances. They also credit transfer amounts before or without checking the actual balance delta. If a configured lending asset charges a transfer fee or negatively rebases while held, fToken/LAV accounting can exceed custody; if it positively rebases, the surplus is not included in share pricing or normal withdrawals.

**Root Cause**  
The asset paths assume exact-transfer, non-rebasing ERC20 behavior. Fraxlend _deposit and _repayAsset mutate totalAsset or totalBorrow from the requested amount and then call safeTransferFrom, while withdraw/borrow capacity is computed from totalAsset.amount rather than assetContract.balanceOf. LendingAssetVault similarly uses _totalAssets, _totalAssetsUtilized, vaultUtilization, and vaultDeposits as nominal ledgers without syncing to IERC20(_asset).balanceOf(address(this)).

**Pre_conditions**  
A Fraxlend pair or LendingAssetVault is configured with a lending asset that transfers less than requested or rebases while held. The documented normal LVF setup uses exact-transfer paired assets and the contest README states nonstandard tokens are not formally supported, so this is primarily an integration/configuration risk.

**Impact**  
Accounting can diverge from actual custody. In Fraxlend, deposits can over-mint fTokens, repayments can reduce borrower debt by more than the pair received, and negative rebases can let early withdrawals or borrows consume the remaining real balance while later users fail. In LAV, share pricing, utilization, and attached-pair liquidity accounting can drift from available assets. Positive rebases are left as unpriced surplus. Under intended exact-transfer, non-rebasing assets the paths are not affected, so the finding is kept at Info.

**Proof of Concept**  
Static trace: FraxlendPair.deposit computes shares from _amount, _deposit increments totalAsset.amount and totalAsset.shares, mints fTokens, and only then calls assetContract.safeTransferFrom. repayAsset subtracts _amountToRepay from totalBorrow before pulling assets. LendingAssetVault.deposit increments _totalAssets and mints shares before pulling the asset; whitelistWithdraw, whitelistDeposit, and depositToVault also book nominal amounts across LAV and Fraxlend. None of these paths reconciles later positive or negative rebases against the live token balance.

**Mitigation**  
Reject fee-on-transfer and rebasing lending assets at deployment/configuration, or account from balance deltas and add a sync policy for balance drift. If rebasing assets must be supported, update totalAsset/_totalAssets and utilization ledgers before deposits, withdrawals, repayments, borrows, and vault movements, and explicitly define how positive and negative rebases are socialized.

### [I-75] Rebasing reward tokens desynchronize TokenRewards liabilities

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: contracts/contracts/TokenRewards.sol:146; contracts/contracts/TokenRewards.sol:180; contracts/contracts/TokenRewards.sol:189; contracts/contracts/TokenRewards.sol:196; contracts/contracts/TokenRewards.sol:220; contracts/contracts/TokenRewards.sol:231; contracts/contracts/TokenRewards.sol:247; contracts/contracts/TokenRewards.sol:251; contracts/contracts/TokenRewards.sol:252; contracts/contracts/RewardsWhitelist.sol:35

**Summary/Description**  
TokenRewards records reward liabilities in rewardsDeposited, rewardsDistributed, and _rewardsPerShare after each nominal reward deposit. If an accepted reward token rebases while unclaimed balances are held by the rewards contract, those ledgers are not synchronized. Positive rebases become unassigned surplus, while negative rebases leave the contract owing more rewards than it holds.

**Root Cause**  
Reward accounting is a static accumulator over deposited token amounts. depositRewards measures the initial received delta, but after that getUnpaid and _distributeReward use _rewardsPerShare and rewardsDistributed without checking the current reward-token balance or adjusting for rebases. There is no token-class validation in RewardsWhitelist and no sync/socialization hook before deposits, claims, or reward-share changes.

**Pre_conditions**  
The pod lpRewardsToken or a whitelisted secondary reward token is rebasing or elastic-supply, and a rebase occurs while TokenRewards holds undistributed balances. The README states nonstandard tokens are not formally supported and reward tokens are owner-whitelisted, so this is treated as an integration/configuration risk.

**Impact**  
For a negative rebase, claim order determines who receives the remaining balance; later claimReward calls or staking share updates can revert when safeTransfer tries to pay the stale nominal amount. For a positive rebase, extra reward balance is not included in _rewardsPerShare and remains unclaimable by the stakers who earned it. The affected value is the reward-token balance held during the rebase.

**Proof of Concept**  
Static trace: depositRewards records _depositAmount in rewardsDeposited and increases _rewardsPerShare by PRECISION * _depositAmount / totalShares. Later, _distributeReward computes getUnpaid from that accumulator, increments rewardsDistributed by the computed amount, and transfers exactly that amount. If the reward token negatively rebases the contract balance between deposit and claim, the accumulator is unchanged and some later transfer can fail or leave later stakers unpaid. If it positively rebases, no accumulator increases, so the surplus is not claimable.

**Mitigation**  
Reject rebasing reward tokens in the whitelist and pod reward configuration, or wrap them into non-rebasing share tokens before distribution. If support is required, add a sync step that compares current balance to undistributed liabilities and explicitly socializes surplus or deficit across current unpaid rewards before deposits, claims, and share changes.

### [I-76] Rebasing lock-release bridge tokens break cross-chain backing

**Severity**: Info  
**Likelihood**: Low  
**Impact**: High  
**Location**: contracts/contracts/ccip/TokenRouter.sol:25; contracts/contracts/ccip/TokenBridge.sol:24; contracts/contracts/ccip/TokenBridge.sol:32; contracts/contracts/ccip/TokenBridge.sol:96; contracts/contracts/ccip/TokenBridge.sol:100; contracts/contracts/ccip/TokenBridge.sol:102; contracts/contracts/ccip/TokenBridge.sol:109; contracts/contracts/ccip/TokenBridge.sol:113

**Summary/Description**  
TokenBridge supports a lock/release mode when sourceTokenMintBurn is false. In that mode the bridge escrows tokens on one chain and releases them on the return path, but it does not reject or account for rebasing escrow tokens. Any positive or negative rebase of the bridge balance changes backing without updating already-sent cross-chain messages or remote minted/released claims.

**Root Cause**  
The bridge records each transfer amount only at message creation time. _processInboundTokens computes the received balance delta and sends that amount across CCIP, while _processOutboundTokens later transfers the message amount. There is no persistent liability ledger, no balance sync, and no validation in TokenRouter.setConfig that lock/release tokens are non-rebasing.

**Pre_conditions**  
A route is configured with sourceTokenMintBurn = false for a token whose balance can rebase while held by the bridge. The scoped ERC20Bridgeable mint/burn token is not rebasing, and the README excludes nonstandard tokens, so this is an integration/configuration risk.

**Impact**  
A negative rebase can make the bridge unable to honor later outbound transfers for previously bridged amounts, causing failed receives or underbacked remote supply depending on route design. A positive rebase creates surplus escrow that is not assigned to any user or remote supply. The potential impact is high for that bridged token balance, but likelihood is low because it requires an unsupported token configuration.

**Proof of Concept**  
Static trace: bridgeTokens pulls the token and sends _amountActual in the CCIP payload. In lock/release mode the tokens remain in the bridge. If the bridge balance later negatively rebases, the outstanding remote-side claims are unchanged. When a return message reaches _processOutboundTokens, the bridge attempts safeTransfer(_user, _amount) even though the escrow balance may no longer cover all outstanding messages.

**Mitigation**  
Reject rebasing tokens for lock/release routes, or require rebasing assets to be wrapped into non-rebasing shares before bridging. If rebasing support is required, maintain explicit liabilities and define how balance drift is allocated across outstanding bridged claims before sending or executing messages.

### [I-77] Zero-approval-reverting tokens can block allowance cleanup paths

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: contracts/contracts/DecentralizedIndex.sol:345; contracts/contracts/DecentralizedIndex.sol:358; contracts/contracts/TokenRewards.sol:299; contracts/contracts/TokenRewards.sol:316; contracts/contracts/AutoCompoundingPodLp.sol:278; contracts/contracts/AutoCompoundingPodLp.sol:298; contracts/contracts/AutoCompoundingPodLp.sol:322; contracts/contracts/AutoCompoundingPodLp.sol:337; contracts/contracts/AutoCompoundingPodLp.sol:341; contracts/node_modules/@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol:87

**Summary/Description**  
Several token-integration paths increase allowance and then try to clean up or no-op the allowance afterward. With OZ v5, safeDecreaseAllowance reduces by calling forceApprove with the new allowance, and safeIncreaseAllowance(..., 0) still calls forceApprove with the current allowance. When that target allowance is zero, the helper issues approve(spender, 0). If a configured paired/reward token reverts on zero-value approvals, these cleanup/no-op calls revert and defeat the intended flow or fallback.

**Root Cause**  
The code assumes allowance cleanup to zero is universally safe for ERC20 integrations. It also uses safeIncreaseAllowance(..., 0) as if it were a no-op after addLiquidityV2, but the helper still performs an approval call.

**Pre_conditions**  
A pod, TokenRewards, or aspTKN route is configured with a paired LP token or primary reward token whose approve(spender, 0) reverts. One of the affected paths either reaches addLiquidityV2 after the DEX adapter consumed the paired-token allowance, or catches a failed reward conversion, V2 paired-token swap, or addLPAndStake attempt after increasing allowance to DEX_ADAPTER or indexUtils. The README says weird tokens are not formally supported, so this is an integration/configuration risk.

**Impact**  
Direct addLiquidityV2/IndexUtils/LVF LP flows can revert at the post-router safeIncreaseAllowance(..., 0) line when the paired token does not allow zero approvals. Failed reward conversion paths also revert during catch-block cleanup instead of recording the smaller retry amount and continuing. In TokenRewards this can block fee processing reached by reward claims or staking share changes. In AutoCompoundingPodLp it can block deposit, mint, withdraw, or redeem paths that first call _processRewardsToPodLp. Under standard ERC20 approval behavior these paths work as intended.

**Proof of Concept**  
Static trace only; no tests run. DecentralizedIndex.addLiquidityV2 increases PAIRED_LP_TOKEN allowance for DEX_HANDLER, calls DEX_HANDLER.addLiquidity, then calls safeIncreaseAllowance(address(DEX_HANDLER), 0). If the router used the full allowance, OZ SafeERC20 reads oldAllowance == 0 and forceApprove calls approve(spender, 0), which a zero-approval-reverting token rejects. TokenRewards._swapForRewards and AutoCompoundingPodLp failure handlers have the same zero target through safeDecreaseAllowance after a failed swap/LP operation leaves the just-increased allowance unspent.

**Mitigation**  
Reject tokens whose approve(0) reverts in paired-token and reward-token configuration, and remove safeIncreaseAllowance(..., 0) as a cleanup/no-op. Make failure-path allowance cleanup best-effort so it cannot revert the intended fallback, or use token-specific approval handling validated at configuration time.

### [I-78] PodUnwrapLocker rejects transfer-taxed pTKN locks

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/PodUnwrapLocker.sol:64; contracts/contracts/PodUnwrapLocker.sol:78; contracts/contracts/WeightedIndex.sol:177; contracts/contracts/DecentralizedIndex.sol:168

**Summary/Description**  
PodUnwrapLocker.debondAndLock pulls the caller's pTKN using the nominal _amount, then calls the pod's debond(_amount). For pods with config.hasTransferTax enabled, the transfer into the locker is treated as an ordinary pTKN transfer and the locker receives less than _amount. The later debond attempts to spend the original nominal amount from the locker and reverts for insufficient pTKN balance, so taxed pTKNs cannot use the fee-free lock flow.

**Root Cause**  
The locker assumes the pTKN transfer into the locker is exact. It does not measure the actual received pTKN balance delta before calling debond, even though pTKN itself can charge the built-in transfer tax on non-pool transfers.

**Pre_conditions**  
A pod has config.hasTransferTax enabled and a user calls PodUnwrapLocker.debondAndLock for that pod. The direct pod debond path remains available, so this is an integration liveness issue rather than a custody loss.

**Impact**  
Users of transfer-taxed pods cannot use PodUnwrapLocker to access the intended delayed fee-free debond path. They must use direct debonding and pay the configured debond fee, or the locker flow remains unavailable until the pod transfer tax is disabled. This affects a protocol-owned token mode but does not by itself trap user funds because the failed transaction rolls back.

**Proof of Concept**  
Static trace: debondAndLock first calls IERC20(_pod).safeTransferFrom(user, locker, _amount). With hasTransferTax enabled, DecentralizedIndex._update takes the ordinary transfer tax and credits the locker with _amount - fee. The function then calls _podContract.debond(_amount,...). WeightedIndex.debond transfers _amount pTKN from msg.sender, which is the locker, to the pod before burning. Since the locker only received _amount - fee, the transfer fails and the whole lock attempt reverts.

**Mitigation**  
Measure the pTKN balance delta received by the locker and call debond with that actual amount, or make PodUnwrapLocker fee-exempt in the pTKN transfer-tax logic. Alternatively, document that pods with hasTransferTax enabled are unsupported by the locker.

### [I-79] LeverageFactory creation seeds are minted to an unrecoverable factory balance

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: contracts/contracts/lvf/LeverageFactory.sol:128; contracts/contracts/lvf/LeverageFactory.sol:131; contracts/contracts/lvf/LeverageFactory.sol:134; contracts/contracts/lvf/LeverageFactory.sol:207; contracts/contracts/lvf/LeverageFactory.sol:209; contracts/contracts/lvf/LeverageFactory.sol:212; contracts/contracts/AutoCompoundingPodLpFactory.sol:30; contracts/contracts/AutoCompoundingPodLpFactory.sol:39; contracts/contracts/AutoCompoundingPodLpFactory.sol:41; fraxlend/src/contracts/FraxlendPairDeployer.sol:271; fraxlend/src/contracts/FraxlendPairDeployer.sol:274

**Summary/Description**  
When LVF setup is routed through LeverageFactory, configured creation seed deposits are funded by the caller but the resulting receipt tokens are minted to LeverageFactory. LeverageFactory has no token rescue, redeem, or forwarding function, so those aspTKN and Fraxlend fToken receipt balances remain unreachable.

**Root Cause**  
The composed factory flow preserves msg.sender as LeverageFactory when calling downstream factories. AutoCompoundingPodLpFactory._depositMin deposits with receiver _msgSender(), and FraxlendPairDeployer._deploy deposits defaultDepositAmt with receiver msg.sender. In both cases msg.sender is LeverageFactory, but LeverageFactory never transfers or redeems the received ERC20 share tokens and exposes no recovery path.

**Pre_conditions**  
addLvfSupportForPod is called while aspTknFactory.minimumDepositAtCreation is nonzero, or either LVF setup path is called while FraxlendPairDeployer.defaultDepositAmt is nonzero. The caller provides the required spTKN or borrow-token seed through LeverageFactory.

**Impact**  
The configured creation seed value is locked in LeverageFactory as aspTKN shares or Fraxlend fTokens. This can act like accidental dead liquidity, but it is not documented or explicitly burned, and the funder cannot recover it. The impact is bounded by creation seed settings and does not give an attacker a direct extraction path, so this remains Info.

**Proof of Concept**  
Static trace: addLvfSupportForPod pulls _aspMinDep spTKN into LeverageFactory and approves aspTknFactory. AutoCompoundingPodLpFactory.create then calls _depositMin, which deposits minimumDepositAtCreation into the new aspTKN with receiver _msgSender(); because the caller is LeverageFactory, the aspTKN shares are minted to LeverageFactory. Later _createFraxlendPair pulls _fraxMinDep borrow tokens into LeverageFactory and calls FraxlendPairDeployer.deploy; the deployer deposits defaultDepositAmt into the new pair with receiver msg.sender, again minting fTokens to LeverageFactory. LeverageFactory only has ownership setters and setup functions, with no token transfer, redeem, or rescue path.

**Mitigation**  
Make creation seed ownership explicit. If the seed is intended to be dead liquidity, mint shares to address(0), address(0xdead), or a documented locked recipient. If it should belong to the funder, forward the received aspTKN/fToken shares to the caller or add a narrowly scoped recovery function for setup receipt tokens.

### [I-80] All-or-nothing token payouts can block unrelated exits under transfer restrictions

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Medium  
**Location**: contracts/contracts/WeightedIndex.sol:175; contracts/contracts/WeightedIndex.sol:185; contracts/contracts/WeightedIndex.sol:189; contracts/contracts/PodUnwrapLocker.sol:129; contracts/contracts/PodUnwrapLocker.sol:135; contracts/contracts/PodUnwrapLocker.sol:137; contracts/contracts/PodUnwrapLocker.sol:156; contracts/contracts/PodUnwrapLocker.sol:158; contracts/contracts/TokenRewards.sol:236; contracts/contracts/TokenRewards.sol:240; contracts/contracts/TokenRewards.sol:252; contracts/contracts/StakingPoolToken.sol:101; contracts/contracts/voting/VotingPool.sol:114

**Summary/Description**  
Several exit and reward paths push multiple tokens atomically. WeightedIndex.debond sends every constituent token to msg.sender, PodUnwrapLocker withdraw paths send every recorded lock token to the user and fee recipient, and TokenRewards distributes every historical reward token during claim and reward-share updates. If any supported token refuses a transfer because the recipient or protocol contract is restricted, the whole call reverts instead of isolating that token, so unrelated tokens in the same exit or reward update remain inaccessible.

**Root Cause**  
The affected flows combine multi-token accounting with mandatory push transfers and have no per-token withdrawal, alternate receiver, skip, or failure-isolation path. TokenRewards has a global pause control, but direct pod debonds and locker withdrawals do not have an equivalent per-token escape path, and reward share hooks can be reached from staking/voting token mint, burn, transfer, stake, and unstake flows.

**Pre_conditions**  
A pod constituent, recorded locker token, or reward token is a standard ERC20 with transfer restrictions. The user or one of the protocol payout contracts becomes restricted after positions or rewards exist, or the token refuses transfers from the protocol contract. For TokenRewards, the token is not paused before the share update or claim is attempted.

**Impact**  
The strongest direct-payout effect is liveness loss. Affected users cannot complete a full pod debond, locker withdrawal, or reward-share update even for unrelated unrestricted tokens included in the same payout. If the restricted address is the protocol payout contract rather than one user, every holder using that payout path can be affected until governance/admin intervention or token-policy changes. This is recorded as Info because the trigger depends on external token-policy action or a restricted recipient rather than a protocol-native attacker-controlled state transition; the stronger shared aspTKN conversion lock is tracked separately in M-14.

**Proof of Concept**  
Static trace: WeightedIndex.debond burns pTKN and then loops indexTokens, calling safeTransfer(msg.sender, amount) for each constituent. A revert on one constituent transfer reverts the entire debond, so prior unrestricted transfers are rolled back and later tokens are never sent. PodUnwrapLocker.withdraw and earlyWithdraw have the same all-token loop over lock.tokens. TokenRewards._distributeReward loops _allRewardsTokens and safeTransfers each unpaid token to the wallet; the same call is reached by StakingPoolToken._update and VotingPool._update, so one blocked reward token can revert the whole stake/unstake/share-update path.

**Mitigation**  
Expose per-token or paginated pull withdrawals with an explicit receiver, and preserve unpaid entitlements when one token cannot be transferred. For rewards, add per-token claiming and make skipped or paused tokens keep their checkpoints until paid or explicitly forfeited. For pods and lockers, allow partial withdrawal of unrestricted tokens or reject restricted-transfer constituents at configuration time.

