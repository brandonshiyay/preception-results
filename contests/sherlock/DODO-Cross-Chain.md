### [H-01] Non-EVM refund claims bypass bot binding and pay the caller

**Severity**: High  
**Likelihood**: High  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:244; omni-chain-contracts/contracts/GatewayCrossChain.sol:265; omni-chain-contracts/contracts/GatewayCrossChain.sol:517; omni-chain-contracts/contracts/GatewayCrossChain.sol:572; omni-chain-contracts/contracts/GatewayTransferNative.sol:253; omni-chain-contracts/contracts/GatewayTransferNative.sol:274; omni-chain-contracts/contracts/GatewayTransferNative.sol:612; omni-chain-contracts/contracts/GatewayTransferNative.sol:666

**Summary/Description**  
GatewayCrossChain and GatewayTransferNative build contract-call revert messages as externalId || receiver. For Solana routes, receiver is the user-provided Solana receiver bytes, so onRevert stores the returned amount as refundInfo whenever the envelope is not exactly 52 bytes. claimRefund then initializes receiver to msg.sender and only replaces it for 20-byte walletAddress values; for Solana/base58, raw pubkey, Bitcoin, or other non-20-byte refunds, msg.sender == receiver is true for every caller, bypassing the configured bot/manual refund domain.

**Root Cause**  
The refund envelope has no explicit address-domain tag, and claimRefund treats non-20-byte refund owners as the caller instead of requiring bot/manual processing or a bound payout recipient.

**Pre_conditions**  
A gateway-authenticated GatewayCrossChain.onRevert/onAbort or GatewayTransferNative.onRevert/onAbort stores a refundInfo whose walletAddress length is not 20 bytes. Normal Solana contract-call reverts satisfy this because withdrawAndCall sets revertMessage to externalId || decoded.receiver, and Solana receivers are encoded as non-20-byte bytes in the route payload. The externalId is observable from EddyCrossChainRefund or transaction data.

**Impact**  
Any address can watch for a Solana/non-EVM refund, front-run or later call claimRefund(externalId), and receive the full stored ZRC20 amount on ZetaChain. The intended Solana recipient or trusted refund bot loses the refund.

**Mitigation**  
Include an explicit refund address-domain/mode in revertMessage. For non-EVM domains such as Solana, require bots[msg.sender] or a dedicated manual-refund processor and do not default the payout receiver to msg.sender. Keep direct EVM refunds restricted to envelopes that are explicitly tagged as EVM, not inferred from byte length alone.

### [H-02] Native sentinel can bypass token intake and spend contract-held ZRC20

**Severity**: High  
**Likelihood**: High  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewayTransferNative.sol:535; omni-chain-contracts/contracts/GatewayTransferNative.sol:540; omni-chain-contracts/contracts/GatewayTransferNative.sol:556; omni-chain-contracts/contracts/GatewayTransferNative.sol:343; omni-chain-contracts/contracts/GatewayTransferNative.sol:428; omni-chain-contracts/contracts/GatewayTransferNative.sol:437; omni-chain-contracts/contracts/libraries/TransferHelper.sol:12

**Summary/Description**  
GatewayTransferNative.withdrawToNativeChain treats zrc20 == _ETH_ADDRESS_ as a native-token route only for the initial intake branch. That branch skips IZRC20.transferFrom and never requires msg.value == amount or wraps native value, yet execution continues into fee accounting, DODO swap, and GatewayZEVM withdrawal. The fee call passes the sentinel into TransferHelper.safeTransfer, whose low-level ERC20 transfer call to a non-contract address succeeds with empty return data. With empty swapDataZ, the post-fee amount is treated as freshly available targetZRC20; with non-empty swapDataZ, _doMixSwap approves route-selected params.fromToken from this contract even though no source asset was collected.

**Root Cause**  
The public exit path mixes the ETH sentinel domain with the ZRC20 custody domain. Native input is not rejected or backed by msg.value/WZETA before fee, swap, approval, and GatewayZEVM withdrawal logic executes.

**Pre_conditions**  
GatewayTransferNative holds, or can be made to hold, ZRC20 balances from residual funds, pending refunds, failed routes, or prior routes. Caller supplies zrc20 as _ETH_ADDRESS_, an amount, and message fields selecting a targetZRC20/receiver path; optionally swapDataZ names a held params.fromToken.

**Impact**  
A caller can trigger fee accounting with no real fee transfer and make the contract approve, swap, or withdraw existing ZRC20 balances to the selected destination. Empty swapDataZ can spend held targetZRC20 directly; non-empty swapDataZ can spend held params.fromToken through DODO before withdrawal. This can drain residual or refund-backed balances and leave later refunds or routes underfunded.

**Mitigation**  
Reject _ETH_ADDRESS_ in withdrawToNativeChain, or implement a separate native path that requires msg.value == amount, wraps/unwraps consistently, and never reaches ZRC20 fee/withdraw logic without verified backing.

### [H-03] GatewaySend native reverts strand refunds by treating empty asset as ERC20

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:393; omni-chain-contracts/contracts/libraries/TransferHelper.sol:12; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/Revert.sol:20; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/evm/GatewayEVM.sol:106

**Summary/Description**  
GatewaySend.onRevert always refunds with TransferHelper.safeTransfer(context.asset, sender, context.amount). For connected-chain gas-token reverts, Zeta's RevertContext.asset is empty while GatewayEVM.executeRevert delivers the native value to GatewaySend before calling onRevert. Calling the ERC20 transfer selector on address(0) succeeds with empty return data and does not send ETH, so the callback emits a refund event while the gas-token refund remains stuck in GatewaySend. This applies to any GatewaySend route whose bridged asset is the connected-chain gas token, whether the user supplied native value directly or the route locally swapped into native value before deposit.

**Root Cause**  
The revert refund path assumes every RevertContext.asset is an ERC20 token address and does not branch on the connected-chain gas-token representation where asset is empty and the native value has already been delivered to the receiver contract balance before onRevert runs.

**Pre_conditions**  
GatewaySend initiates a GatewayEVM depositAndCall whose bridged asset is the connected-chain gas token (asset == _ETH_ADDRESS_ in GatewaySend, direct native input or post-swap native output), with callOnRevert enabled. The cross-chain transaction reverts and GatewayEVM calls executeRevert with native value and RevertContext.asset == address(0).

**Impact**  
Users whose GatewaySend routes bridge the connected-chain gas token do not receive refunds when the cross-chain route reverts. The native value is transferred into GatewaySend and remains there without route accounting or an automatic recovery path, breaking the core revert-refund guarantee for ETH/native deposits and post-swap native deposits.

**Mitigation**  
In GatewaySend.onRevert, validate context.revertMessage.length == 52, then branch on the connected-chain gas-token case where context.asset == address(0). For that branch, refund native value from GatewaySend balance with TransferHelper.safeTransferETH(sender, context.amount); keep TransferHelper.safeTransfer only for nonzero ERC20 asset addresses. Consider checking that the contract balance covers context.amount before emitting the refund event.

### [H-04] GatewayTransferNative swaps the pre-fee amount after transferring the platform fee

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewayTransferNative.sol:373; omni-chain-contracts/contracts/GatewayTransferNative.sol:398; omni-chain-contracts/contracts/GatewayTransferNative.sol:428

**Summary/Description**  
GatewayTransferNative.onCall transfers the platform fee from the inbound zrc20 amount, but in the different-token local settlement branch it then calls _doMixSwap(decoded.swapData, amount, params) with the original pre-fee amount. A route that swaps the gross amount can therefore revert after the fee transfer or, if the contract holds residual/refund balance of the same token, consume that unrelated balance so the current receiver gets output as if no platform fee had been deducted.

**Root Cause**  
The local different-token branch does not reduce amount by platformFeesForTx before passing it to the DODO swap helper. This differs from GatewayCrossChain.onCall and GatewayTransferNative.withdrawToNativeChain, which both subtract the platform fee before swapping.

**Pre_conditions**  
GatewayZEVM calls GatewayTransferNative.onCall with decoded.targetZRC20 != zrc20, feePercent > 0, and non-empty DODO swapData. The route's DODO params spend more than the post-fee amount, or the contract has residual/refund zrc20 balance that can cover the fee shortfall.

**Impact**  
Honest different-token local settlements can fail because the route tries to spend more than the post-fee balance. When residual/refund balance exists, a crafted route can repeatedly use that balance to fund the platform fee while the current receiver obtains output for the gross inbound amount, depleting balances that should remain available for other refunds or routes.

**Mitigation**  
Subtract platformFeesForTx from amount before the different-token swap, and bind the DODO route amount to the post-fee amount. For non-empty swapData, require params.fromToken == zrc20 and params.fromTokenAmount == amount - platformFeesForTx before approving and calling DODO.

### [H-05] GatewayTransferNative treats native ZETA callbacks as WZETA ERC20 and does not forward the value

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewayTransferNative.sol:343; omni-chain-contracts/contracts/GatewayTransferNative.sol:373; omni-chain-contracts/contracts/GatewayTransferNative.sol:376; omni-chain-contracts/contracts/GatewayTransferNative.sol:398; omni-chain-contracts/contracts/GatewayTransferNative.sol:438; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:124; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:383

**Summary/Description**  
GatewayZEVM native-ZETA depositAndCall transfers native ZETA to the target and then calls onCall with zrc20 set to zetaToken/WZETA and no msg.value. GatewayTransferNative.onCall treats that amount as ERC20-backed WZETA: the same-token path transfers WZETA, while the swap path approves WZETA or forwards msg.value, which is zero, to DODO. The native value received through receive() is never wrapped into WZETA or forwarded to the receiver/router.

**Root Cause**  
The callback uses the gateway-supplied zrc20 address as proof of token custody. That assumption is false for GatewayZEVM native-ZETA callbacks, where zrc20 identifies WZETA but the target receives native value before onCall and the callback itself carries no msg.value.

**Pre_conditions**  
GatewayZEVM executes its native-ZETA depositAndCall path to GatewayTransferNative. The encoded native message selects WZETA same-token settlement or a local swap whose input is WZETA/native ZETA. If GatewayTransferNative has no WZETA or target-token residual balance the route reverts; if it has unrelated residual or refund balance, the branch can spend that balance while the delivered native ZETA remains unaccounted.

**Impact**  
Native-ZETA local settlements cannot reliably complete. When unrelated WZETA or target-token balances are present, a current native-ZETA route can consume those balances for fees, receiver payout, or DODO input while leaving the newly delivered native ZETA as untracked contract balance, underfunding refunds/residual balances and breaking settlement accounting.

**Mitigation**  
Handle GatewayZEVM native-ZETA callbacks as a separate asset domain before fee, same-token payout, and DODO swap logic. For zrc20 == WZETA/native-ZETA callbacks, wrap the received native amount with WZETA.deposit before ERC20 accounting or route and pay native value directly. If native ZETA input is unsupported, reject it explicitly before accepting value.

### [H-06] Public GatewayTransferNative withdraw can spend residual gateway allowance

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewayTransferNative.sol:286; omni-chain-contracts/contracts/GatewayTransferNative.sol:292; omni-chain-contracts/contracts/GatewayTransferNative.sol:462; omni-chain-contracts/contracts/GatewayTransferNative.sol:482; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:106

**Summary/Description**  
GatewayTransferNative.withdraw is public and forwards caller-supplied receiver bytes, outputToken, and amount to GatewayZEVM.withdraw. In that external call, GatewayTransferNative is msg.sender, so GatewayZEVM pulls both the gas fee and withdrawal amount from GatewayTransferNative's balances and allowances rather than from the public caller. Same-gas-token outbound branches approve outputAmount + gasFee but GatewayZEVM only pulls gasFee + (outputAmount - gasFee) == outputAmount, leaving residual allowance that can later be consumed through the public wrapper when it exceeds the current gas-fee pull.

**Root Cause**  
The raw GatewayZEVM withdraw helper is exposed as a public function and has no owner/gateway/internal restriction or caller-funded token pull. The same-gas-token dispatch helpers also over-approve GatewayZEVM, leaving allowance residue after successful withdrawals.

**Pre_conditions**  
A prior GatewayTransferNative same-gas-token Bitcoin, EVM, or Solana outbound route leaves GatewayZEVM allowance for the target ZRC20. GatewayTransferNative later holds the same ZRC20, for example residual or refund-backed balance. The residual allowance is greater than the gas fee charged by the later GatewayZEVM.withdraw call.

**Impact**  
Any address can call withdraw with attacker-chosen receiver bytes and route contract-held ZRC20 through GatewayZEVM up to the positive allowance and balance envelope. This can drain shared residual or refund-backed balances and make later refunds or routes underfunded.

**Mitigation**  
Make GatewayTransferNative.withdraw internal like GatewayCrossChain.withdraw, or gate it to an authorized role and require explicit route context. Also approve GatewayZEVM only for the exact amount it will pull in each branch, or reset allowance to zero after the GatewayZEVM call.

### [H-07] Stored refunds can be overwritten with duplicate externalIds

**Severity**: High  
**Likelihood**: High  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:442; omni-chain-contracts/contracts/GatewayCrossChain.sol:449; omni-chain-contracts/contracts/GatewayCrossChain.sol:244; omni-chain-contracts/contracts/GatewayCrossChain.sol:265; omni-chain-contracts/contracts/GatewayCrossChain.sol:277; omni-chain-contracts/contracts/GatewayCrossChain.sol:291; omni-chain-contracts/contracts/GatewayCrossChain.sol:517; omni-chain-contracts/contracts/GatewayCrossChain.sol:540; omni-chain-contracts/contracts/GatewayCrossChain.sol:551; omni-chain-contracts/contracts/GatewayCrossChain.sol:562; omni-chain-contracts/contracts/GatewayTransferNative.sol:612; omni-chain-contracts/contracts/GatewayTransferNative.sol:634; omni-chain-contracts/contracts/GatewayTransferNative.sol:645; omni-chain-contracts/contracts/GatewayTransferNative.sol:656; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/evm/GatewayEVM.sol:275; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/evm/GatewayEVM.sol:301; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:136; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:171; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:281

**Summary/Description**  
GatewayCrossChain and GatewayTransferNative key stored refunds only by externalId and their refund callbacks never reject an already unresolved key. GatewayCrossChain also imports externalId directly from the first 32 bytes of the gateway-delivered message and copies that value into outbound RevertOptions. A later authenticated onRevert stores refundInfos[externalId] whenever the revert message is not exactly 52 bytes, while onAbort stores refundInfos[externalId] for any prefixed revert message, including 52-byte messages. Public GatewayEVM and GatewayZEVM entrypoints let users choose payloads and revert or abort options, so an observed pending refund id can be reused in a later gateway callback that replaces the original RefundInfo.

**Root Cause**  
The refund callbacks trust msg.sender == gateway as sufficient provenance and treat externalId as globally unique, while GatewayCrossChain imports externalId from caller-controlled inbound message bytes and the callbacks neither verify that the id was issued by this contract nor reject duplicate refundInfos[externalId] writes.

**Pre_conditions**  
A victim has an unresolved refundInfos[externalId] entry in GatewayCrossChain or GatewayTransferNative. An untrusted actor observes that id and causes a gateway-authenticated callback with a revertMessage beginning with the same 32-byte id, either by sending a connected-chain GatewayEVM depositAndCall payload and chosen abort/revert options, by routing through GatewayCrossChain so the supplied id is copied into an outbound revert message, or by funding a public GatewayZEVM withdraw/withdrawAndCall/call with revertAddress or abortAddress targeting the refund contract. For onRevert the replacement path needs revertMessage.length != 52; for onAbort any length >= 32 reaches the storage write.

**Impact**  
The later gateway callback replaces the original RefundInfo under the same key. The victim previously had a claimable refund record, but the token, amount, and walletAddress needed by claimRefund are lost or replaced, so the intended refund can no longer be claimed through the protocol. The original funds remain in the contract as untracked balance, and the grief can be repeated against any observed pending stored refund without finding a hash collision.

**Mitigation**  
Treat externalId as a protocol-owned nonce/status key, not as trusted caller input. For GatewayCrossChain, derive a fresh outbound refund key on ZEVM or require the inbound id to match an authenticated source and route digest. Track pending ids with expected token, amount, recipient, and route provenance before initiating gateway withdrawals. In onRevert/onAbort, require the callback to match a pending record and reject duplicate writes unless an explicit merge state exists; at minimum require refundInfos[externalId].externalId == bytes32(0) before storing.

### [H-08] GatewayCrossChain and GatewayTransferNative treat empty swapData as a successful different-token conversion

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:56; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:86; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:108; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:112; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:131; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:145; omni-chain-contracts/contracts/GatewayCrossChain.sol:337; omni-chain-contracts/contracts/GatewayCrossChain.sol:342; omni-chain-contracts/contracts/GatewayCrossChain.sol:453; omni-chain-contracts/contracts/GatewayCrossChain.sol:465; omni-chain-contracts/contracts/GatewayCrossChain.sol:468; omni-chain-contracts/contracts/GatewayCrossChain.sol:490; omni-chain-contracts/contracts/GatewayTransferNative.sol:398; omni-chain-contracts/contracts/GatewayTransferNative.sol:428; omni-chain-contracts/contracts/GatewayTransferNative.sol:433; omni-chain-contracts/contracts/GatewayTransferNative.sol:535; omni-chain-contracts/contracts/GatewayTransferNative.sol:548; omni-chain-contracts/contracts/GatewayTransferNative.sol:560; omni-chain-contracts/contracts/GatewayTransferNative.sol:563; omni-chain-contracts/contracts/GatewayTransferNative.sol:585

**Summary/Description**  
SwapDataHelperLib.decodeMessage and decodeNativeMessage allow the swapDataZ or swapData section to be zero length, and decodeCompressedMixSwapParams returns default zero-token params for that case. GatewayCrossChain.onCall and GatewayTransferNative then call _doMixSwap, which returns the current amount unchanged when swapData is empty. If decoded.targetZRC20 differs from the actual input zrc20, the unchanged input amount is still treated as targetZRC20 output. In GatewayTransferNative.withdrawToNativeChain specifically, the caller can deposit zrc20 A, provide empty swapDataZ, declare targetZRC20 B, and the Bitcoin/EVM/Solana withdrawal branches quote gas, approve, and withdraw B even though no DODO conversion from A to B occurred.

**Root Cause**  
The empty swapData identity path is shared across same-token and different-token settlement. The ZEVM wrappers do not require decoded.targetZRC20 to equal the actual input zrc20 when swap data is empty, and they do not require a non-empty, token-bound DODO route before paying or withdrawing a different target token.

**Pre_conditions**  
A gateway callback to GatewayCrossChain or GatewayTransferNative, or a public GatewayTransferNative.withdrawToNativeChain call, supplies an input zrc20 amount and a packed route whose targetZRC20 is a different token while the relevant swapData or swapDataZ section is empty. The affected wrapper holds enough targetZRC20 balance from residual funds, pending refunds, failed routes, or prior routes to cover the payout or withdrawal. For the public GatewayTransferNative outbound path, the caller only needs to provide some source zrc20 amount; the post-fee source tokens remain in the wrapper while existing targetZRC20 is used for settlement.

**Impact**  
The route can spend contract-held targetZRC20 while leaving the actual input zrc20 in the contract, minus any platform fee. This can underfund pending refunds or residual balances for the target token and gives the current receiver targetZRC20 without executing the required swap. In the GatewayTransferNative outbound path, a caller can choose a lower-value or differently-decimal source token and an empty swapDataZ, then have GatewayZEVM pull an existing balance of the declared target token for Bitcoin, EVM, or Solana settlement.

**Mitigation**  
Treat empty swapData as valid only for same-token settlement. Before any different-token payout or withdrawal, require the relevant swap data to be non-empty, params.fromToken to equal the actual input zrc20, params.toToken to equal decoded.targetZRC20, and params.fromTokenAmount to equal the post-fee amount. Also verify the expected target-token balance delta before using the DODO return amount for payout or GatewayZEVM withdrawal.

### [H-09] DODO route targets can redirect approved swap input

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:189; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:202; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:215; omni-chain-contracts/contracts/GatewaySend.sol:195; omni-chain-contracts/contracts/GatewaySend.sol:202; omni-chain-contracts/contracts/GatewayCrossChain.sol:338; omni-chain-contracts/contracts/GatewayCrossChain.sol:347; omni-chain-contracts/contracts/GatewayTransferNative.sol:429; omni-chain-contracts/contracts/GatewayTransferNative.sol:438

**Summary/Description**  
Compressed swap data lets the route caller choose DODO mixAdapters, mixPairs, and assetTo. The gateway decodes those fields and forwards them unchanged to DODORouteProxy.mixSwap after approving DODOApprove for the selected fromToken amount. In DODO Fee Route Proxy semantics, assetTo[0] is the initial receiver used by _deposit/claimTokens, and each mixAdapter address is called during route execution. A crafted route can therefore make DODO pull approved tokens from the gateway to an attacker-selected assetTo[0], then use an attacker adapter to provide only the small route-controlled toToken amount needed for the call to pass.

**Root Cause**  
The gateways treat user-supplied DODO route execution targets as trusted swap metadata. They do not require assetTo[0] to be the expected pool/router, do not restrict mixAdapters to trusted adapters, and do not prove that params.fromToken, params.toToken, params.fromTokenAmount, and the final output balance delta correspond to the route's declared input and output assets before settlement.

**Pre_conditions**  
An attacker controls a route payload reaching GatewaySend.depositAndCall/GatewaySend.onCall, GatewayCrossChain.onCall, GatewayTransferNative.onCall, or GatewayTransferNative.withdrawToNativeChain. The gateway has approved DODOApprove for the selected params.fromToken amount, and the relevant contract holds or receives enough params.fromToken for DODOApproveProxy to claim. The attacker sets assetTo[0] to an address they control and uses a callable adapter path that supplies at least minReturnAmount of params.toToken back to the DODO route proxy or gateway.

**Impact**  
Approved swap input can be redirected to an arbitrary attacker-controlled address instead of being sent to a real liquidity pool. When params.fromToken is a contract-held residual or refund-backed token, the route can drain those balances while leaving the caller's actual input token in the contract or spending only a small amount of attacker-provided output token. This can underfund pending refunds and lets current routes settle using assets not produced by a legitimate swap.

**Mitigation**  
Treat compressed DODO route data as untrusted. Bind params.fromToken to the actual received input token, params.fromTokenAmount to the post-fee amount, and params.toToken to the declared output token. Additionally require the DODO call to increase this contract's expected output-token balance by the returned amount before payout/withdrawal, and only accept route target fields from a trusted router/quote source or enforce an allowlist of approved adapters/pools/assetTo shapes.

### [H-10] DODO route tokens and amount are unbound from gateway input/output

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:195; omni-chain-contracts/contracts/GatewaySend.sol:202; omni-chain-contracts/contracts/GatewaySend.sol:341; omni-chain-contracts/contracts/GatewaySend.sol:350; omni-chain-contracts/contracts/GatewaySend.sol:366; omni-chain-contracts/contracts/GatewaySend.sol:372; omni-chain-contracts/contracts/GatewayCrossChain.sol:337; omni-chain-contracts/contracts/GatewayCrossChain.sol:347; omni-chain-contracts/contracts/GatewayCrossChain.sol:465; omni-chain-contracts/contracts/GatewayCrossChain.sol:369; omni-chain-contracts/contracts/GatewayCrossChain.sol:389; omni-chain-contracts/contracts/GatewayCrossChain.sol:413; omni-chain-contracts/contracts/GatewayTransferNative.sol:359; omni-chain-contracts/contracts/GatewayTransferNative.sol:428; omni-chain-contracts/contracts/GatewayTransferNative.sol:535; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:144

**Summary/Description**  
Non-empty DODO swap helpers decode params.fromToken, params.toToken, params.fromTokenAmount, and minReturnAmount from route bytes and pass them to DODORouteProxy without requiring params.fromToken to equal the token just received, params.fromTokenAmount to equal the current route amount, or params.toToken to equal the declared downstream asset/target. In GatewayCrossChain.onCall specifically, _doMixSwap returns the DODO return value and the code immediately treats it as newly received decoded.targetZRC20 for Bitcoin, EVM, or Solana withdrawal handling, without taking a decoded.targetZRC20 balance snapshot before/after the external route. A route can satisfy DODO accounting for a route-selected token while the wrapper approves or withdraws outputAmount of the separately declared target token from pre-existing custody.

**Root Cause**  
The gateway wrappers do not enforce a pre-swap invariant between the outer route state and the compressed DODO params, and they do not prove an expected output-token balance delta before settlement. GatewayCrossChain separates decoded.targetZRC20 from params.toToken and uses the external mixSwap return value as targetZRC20 accounting truth.

**Pre_conditions**  
A wrapper holds a residual or pending-refund balance of a token selected as the downstream asset/target, or holds a residual input token named by params.fromToken. For GatewayCrossChain, a gateway-authenticated onCall message carries non-empty swapDataZ whose compressed DODO params are not bound to decoded.targetZRC20 and the route output is not newly delivered as decoded.targetZRC20 to the wrapper. For the GatewaySend destination path, a gateway-authenticated callback reaches onCall with fromToken matching the gateway-delivered token, declared toToken set to a token held by GatewaySend, and non-empty swapData whose params.toToken differs from that declared toToken while satisfying DODO minReturnAmount.

**Impact**  
The current route can spend, pay, bridge, or withdraw token balances unrelated to the caller-funded input. In GatewayCrossChain, if the DODO route output is not newly delivered as decoded.targetZRC20 to this contract, the later gas-fee and withdrawal branches can approve or withdraw existing targetZRC20 balances, including residual balances or funds awaiting refund, to the route-selected destination. This underfunds later routes or refunds and makes settlement depend on stale custody rather than the current swap result.

**Mitigation**  
For every non-empty DODO route, require params.fromToken to equal the actual route input token, params.fromTokenAmount to equal the current route amount after any fee, and params.toToken to equal the declared downstream asset or target token. Around the DODO call, snapshot the expected output token balance of the executing wrapper and use only the positive balance delta as outputAmount before any payout, gateway deposit, gas conversion, or withdrawal. Treat empty swapData as valid only for same-token settlement and avoid forwarding unrelated msg.value.

### [H-11] GatewaySend empty ETH destination swap can drain stored native funds

**Severity**: High  
**Likelihood**: Medium  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:128; omni-chain-contracts/contracts/GatewaySend.sol:138; omni-chain-contracts/contracts/GatewaySend.sol:143; omni-chain-contracts/contracts/GatewaySend.sol:149; omni-chain-contracts/contracts/GatewaySend.sol:150; omni-chain-contracts/contracts/GatewaySend.sol:153; omni-chain-contracts/contracts/GatewaySend.sol:358; omni-chain-contracts/contracts/GatewaySend.sol:363; omni-chain-contracts/contracts/GatewaySend.sol:369; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/evm/GatewayEVM.sol:167; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/evm/GatewayEVM.sol:181

**Summary/Description**  
GatewaySend.onCall trusts the destination message fromToken and toToken fields to identify the asset delivered by GatewayEVM. decodePackedData also reads tokenA and tokenB with calldataload before requiring crossChainData.length >= 40, while decodePackedMessage does not require the message to end after the declared crossChainData slice. A gateway-delivered message can therefore declare zero or short crossChainData, place token bytes in trailing calldata, and still have fromToken/toToken decoded from those trailing bytes while swapData remains empty. When the decoded fields are fromToken == toToken == the ETH sentinel during an ERC20-backed GatewayEVM.executeWithERC20 callback, GatewaySend skips ERC20 transferFrom and transfers native ETH from its own balance to the receiver.

**Root Cause**  
The destination callback does not canonicalize the packed message length before assembly token decoding, and it treats decoded token equality plus the ETH sentinel as proof of custody. onCall never binds the decoded token fields to GatewayEVM's actual asset delivery and never requires msg.value == amount when fromToken is the ETH sentinel.

**Pre_conditions**  
GatewaySend holds native ETH, for example from stranded native revert refunds or other residual value. A cross-chain withdrawal or routed call reaches destination GatewaySend through GatewayEVM.executeWithERC20 with an ERC20 asset. The callback message either canonically encodes fromToken == toToken == the ETH sentinel, or declares zero/short crossChainData and leaves trailing bytes at the decoder offsets that decode to that sentinel pair. GatewayEVM approves ERC20 to GatewaySend but does not send matching native value.

**Impact**  
The attacker can spend or route an ERC20-backed withdrawal but receive native ETH from GatewaySend stored balance. The ERC20 approved by GatewayEVM is not consumed by GatewaySend and is returned to the asset handler after the callback, so the loss is borne by GatewaySend native funds such as stranded refunds. The short-crossChainData variant also makes the message non-canonical: the length prefix can say no token fields exist while assembly still consumes hidden trailing bytes as token addresses.

**Mitigation**  
In decodePackedMessage/decodePackedData, require message.length == 68 + receiverLen + crossChainDataLen and require crossChainData.length >= 40 before reading token fields; reject trailing bytes and malformed packed messages. In GatewaySend.onCall, treat ETH as valid input only when msg.value == amount and reject ETH-sentinel input on ERC20 callbacks. More generally, bind decoded fromToken/toToken and any DODO params to the actual GatewayEVM-delivered asset before paying the receiver.

### [H-12] Plain EVM and Solana withdrawals are routed through withdrawAndCall

**Severity**: High  
**Likelihood**: High  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:379; omni-chain-contracts/contracts/GatewayCrossChain.sol:402; omni-chain-contracts/contracts/GatewayCrossChain.sol:431; omni-chain-contracts/contracts/GatewayTransferNative.sol:471; omni-chain-contracts/contracts/GatewayTransferNative.sol:495; omni-chain-contracts/contracts/GatewayTransferNative.sol:524; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:34; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:171

**Summary/Description**  
DecodedMessage.contractAddress is documented as empty for plain withdraw and non-empty for withdrawAndCall, but the non-Bitcoin outbound handlers route every EVM/Solana destination through _handleEvmOrSolanaWithdraw. In both GatewayCrossChain and GatewayTransferNative, the same-gas and different-gas branches call withdrawAndCall with decoded.contractAddress. A valid EVM or Solana plain-withdraw route therefore passes an empty receiver to GatewayZEVM.withdrawAndCall instead of calling GatewayZEVM.withdraw.

**Root Cause**  
The non-Bitcoin dispatch logic branches only on dstChainId and gas-token mode. It never branches on decoded.contractAddress.length to select GatewayZEVM.withdraw for plain withdrawals versus GatewayZEVM.withdrawAndCall for destination contract calls.

**Pre_conditions**  
GatewayZEVM calls GatewayCrossChain.onCall, or a user calls GatewayTransferNative.withdrawToNativeChain, with a valid non-Bitcoin route whose decoded.contractAddress.length is zero, as specified by the message schema for plain withdraw. The route has enough post-fee output to reach _handleEvmOrSolanaWithdraw.

**Impact**  
GatewayZEVM.withdrawAndCall rejects zero-length receiver bytes before withdrawal execution, so the documented plain withdrawal workflow for EVM and Solana destinations is persistently unavailable. Users cannot receive a direct outbound withdrawal through these wrappers even with a correctly encoded plain-withdraw message; contract-call routes remain separate and do not restore plain-withdraw semantics.

**Mitigation**  
In _handleEvmOrSolanaWithdraw, select the gateway mode from decoded.contractAddress.length. If it is zero, validate the destination receiver for the selected chain and call the internal withdraw helper with decoded.receiver. If it is nonzero, validate contractAddress length/domain and call withdrawAndCall. Keep revert-message encoding domain-aware for Solana/non-EVM plain withdrawals.

### [M-01] GatewaySend cannot deposit supported no-return ERC20 assets

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Medium  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:184; omni-chain-contracts/contracts/GatewaySend.sol:199; omni-chain-contracts/contracts/GatewaySend.sol:239; omni-chain-contracts/contracts/GatewaySend.sol:317; omni-chain-contracts/contracts/GatewaySend.sol:359; omni-chain-contracts/contracts/GatewaySend.sol:372

**Summary/Description**  
GatewaySend performs ERC20 transfer, transferFrom, and approve through the strict IERC20 interface. Tokens that successfully execute but return no data cause the Solidity high-level call return decoder to revert, so GatewaySend direct deposits, source swaps, gateway approvals, authenticated inbound pulls, and local ERC20 payouts cannot handle supported no-return assets such as USDT-backed ZetaChain assets.

**Root Cause**  
GatewaySend imports TransferHelper but does not use safeApprove/safeTransfer/safeTransferFrom at the ERC20 interaction sites; raw IERC20 calls require a 32-byte boolean return even when the token operation itself succeeds.

**Pre_conditions**  
A user routes through GatewaySend with a no-return ERC20 as the direct deposit asset, DODO input token, DODO output asset to be approved for GatewayEVM, or destination payout token. ZetaChain supported-asset data includes USDT assets on ETH, BSC, and Polygon, whose origin token style is the canonical no-return ERC20 pattern.

**Impact**  
All affected GatewaySend routes revert before the gateway deposit/call or final payout, making supported no-return assets unusable through GatewaySend. This blocks a core deposit/swap path for multiple users and chains; no direct asset loss is required because the failure happens atomically.

**Mitigation**  
Replace raw IERC20 interactions in GatewaySend with TransferHelper.safeApprove, safeTransfer, and safeTransferFrom, or OpenZeppelin SafeERC20/forceApprove where zero-reset approval semantics are needed. Apply the same optional-return handling consistently before gateway and DODO calls.

### [M-02] Bitcoin withdrawal failures refund to an EVM-projected address

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:277; omni-chain-contracts/contracts/GatewayCrossChain.sol:363; omni-chain-contracts/contracts/GatewayCrossChain.sol:517; omni-chain-contracts/contracts/GatewayTransferNative.sol:286; omni-chain-contracts/contracts/GatewayTransferNative.sol:454; omni-chain-contracts/contracts/GatewayTransferNative.sol:612

**Summary/Description**  
GatewayCrossChain and GatewayTransferNative route Bitcoin-style withdrawals through withdraw(), but that helper builds RevertOptions.revertMessage as externalId || bytes20(sender) while the caller passes decoded.receiver as sender. Short receiver bytes are zero-padded, exact-20-byte non-EVM values are treated as EVM addresses, and overlong receiver bytes are truncated. When the outbound withdrawal later reverts, onRevert sees a 52-byte message and transfers the returned ZRC20 directly to address(uint160(bytes20(walletAddress))) instead of registering a non-EVM/manual refund.

**Root Cause**  
The Bitcoin withdrawal path converts address-domain-agnostic receiver bytes into a fixed 20-byte EVM refund envelope, and the revert handler classifies refund mode solely by revertMessage.length == 52 rather than by the destination/address domain.

**Pre_conditions**  
A route reaches the Bitcoin branch in GatewayCrossChain.onCall or GatewayTransferNative.withdrawToNativeChain with non-empty receiver bytes that are not an EVM address. GatewayZEVM later triggers onRevert for the outbound withdrawal after the withdrawal fails on the connected chain.

**Impact**  
The full reverted amount is sent to a ZetaChain EVM address derived from the Bitcoin/non-EVM receiver bytes by padding, direct 20-byte interpretation, or truncation. The intended user or refund bot cannot claim it through refundInfos, so failed Bitcoin withdrawals can lose the user's entire returned amount.

**Mitigation**  
For Bitcoin and other non-EVM plain withdrawals, keep the full receiver bytes in revertMessage with an explicit refund-mode or address-domain tag, and make onRevert store non-EVM refunds for bot/manual processing. Only cast receiver bytes to bytes20 after the route mode has proven the refund recipient is an EVM address.

### [M-03] Uniswap gas-token route can be forced through an unusable direct pair

**Severity**: Medium  
**Likelihood**: High  
**Impact**: Medium  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:207; omni-chain-contracts/contracts/GatewayCrossChain.sol:222; omni-chain-contracts/contracts/GatewayCrossChain.sol:302; omni-chain-contracts/contracts/GatewayTransferNative.sol:216; omni-chain-contracts/contracts/GatewayTransferNative.sol:231; omni-chain-contracts/contracts/GatewayTransferNative.sol:306; omni-chain-contracts/contracts/libraries/UniswapV2Library.sol:42

**Summary/Description**  
When targetZRC20 and gasZRC20 differ, both gateways choose the direct Uniswap path whenever balanceOf() for both tokens is positive at the deterministic pair address. Positive balances do not prove that a factory pair exists, has code, has minted liquidity, or has reserves sufficient for the exact gas-fee output. An attacker can create or pre-fund the direct pair with dust, causing getPathForTokens to skip the WZETA fallback and making getAmountsIn or the router swap revert or price against attacker-controlled shallow reserves.

**Root Cause**  
_existsPairPool treats raw token balances at a CREATE2-derived pair address as a usability proof for the direct gas-token route instead of checking factory registration, deployed pair code, nonzero reserves, and sufficient reserveOut for gasFee. The fallback path is selected only when those raw balances are zero.

**Pre_conditions**  
A route reaches _handleEvmOrSolanaWithdraw with decoded.targetZRC20 != gasZRC20 in GatewayCrossChain.onCall or GatewayTransferNative.onCall/withdrawToNativeChain. The intended safe route is via WZETA or a liquid direct pair is absent. An attacker can transfer tiny amounts of targetZRC20 and gasZRC20 to the deterministic direct pair address, or create the pair and leave zero/tiny reserves.

**Impact**  
Different-gas-token outbound settlements for the affected token pair can be repeatedly blocked or forced into a shallow attacker-controlled direct pool. Because there is no caller-selectable fallback path, affected cross-chain withdrawals and calls cannot reliably buy the required gas token even when the WZETA route is liquid. This breaks a core settlement path and can delay or revert user withdrawals until the pair state is remediated.

**Mitigation**  
Use the Uniswap factory getPair result and require deployed pair code before selecting a direct path. Read getReserves and require both reserves to be nonzero and reserveOut > gasFee, and consider a minimum-liquidity threshold. If the direct pair is missing or unusable, validate both WZETA fallback hops before use, or let trusted route data choose an explicitly validated path.

### [M-04] Gas-token exact-output swap accepts manipulated spot reserves without a user max spend

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:309; omni-chain-contracts/contracts/GatewayCrossChain.sol:315; omni-chain-contracts/contracts/GatewayCrossChain.sol:319; omni-chain-contracts/contracts/GatewayCrossChain.sol:334; omni-chain-contracts/contracts/GatewayTransferNative.sol:313; omni-chain-contracts/contracts/GatewayTransferNative.sol:319; omni-chain-contracts/contracts/GatewayTransferNative.sol:324; omni-chain-contracts/contracts/GatewayTransferNative.sol:340; omni-chain-contracts/contracts/libraries/UniswapV2Library.sol:86

**Summary/Description**  
For different-gas-token withdrawals, the gateways buy the exact gasFee by quoting Uniswap spot reserves inside the execution transaction, adding the global slippage margin, and sending the remaining targetZRC20 to the receiver. The route message has no per-route maximum gas-swap input or minimum final target output. A block builder or front-runner can move the selected Uniswap reserves before the gateway transaction, make gasZRC20 temporarily expensive, and the contract will accept that manipulated quote as long as amountInMax stays below targetAmount.

**Root Cause**  
The gas-funding swap treats the current AMM spot reserve quote as the user's accepted gas conversion price. The only cap is amountInMax derived from that same current quote plus global slippage; it is not bound to a user-supplied max input, source-chain quote, TWAP, or final-output minimum.

**Pre_conditions**  
A route reaches the different-gas-token branch with targetAmount materially larger than the honest gas swap cost. The chosen direct or WZETA Uniswap path has reserves that can be moved before the gateway call. The attacker can trade the pool before the gateway transaction and unwind after the exact-output gas purchase.

**Impact**  
The attacker can make the gas-token purchase consume more targetZRC20 than the honest market price, reducing amountsOutTarget and the value delivered to the receiver. This is repeatable for high-value cross-chain withdrawals and is not protected by the DODO route minReturnAmount because the Uniswap gas conversion happens after the DODO output is produced.

**Mitigation**  
Carry a user- or quote-service-specified maxGasSwapInput/minFinalOutput through the route payload and enforce it after the gas swap. Alternatively use a trusted gas-swap route with explicit slippage controls or a manipulation-resistant price bound, and reject execution when the exact gas purchase would consume more than the caller accepted.

### [M-05] Different-gas-token withdrawals reject positive actual remainders

**Severity**: Medium  
**Likelihood**: Medium  
**Impact**: Medium  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:315; omni-chain-contracts/contracts/GatewayCrossChain.sol:319; omni-chain-contracts/contracts/GatewayCrossChain.sol:329; omni-chain-contracts/contracts/GatewayCrossChain.sol:334; omni-chain-contracts/contracts/GatewayTransferNative.sol:319; omni-chain-contracts/contracts/GatewayTransferNative.sol:324; omni-chain-contracts/contracts/GatewayTransferNative.sol:335; omni-chain-contracts/contracts/GatewayTransferNative.sol:340

**Summary/Description**  
When targetZRC20 differs from gasZRC20, _swapAndSendERC20Tokens buys the exact gasFee and the router returns the actual targetZRC20 spent as amounts[0]. The code later sends targetAmount - amounts[0] to GatewayZEVM, but it first rejects unless targetAmount is greater than amountInMax, where amountInMax is the current quote plus the global slippage cushion. Therefore a route with amounts[0] < targetAmount <= amountInMax has enough targetZRC20 to pay for the exact gas-token purchase and still withdraw a positive amount, but the function reverts.

**Root Cause**  
The post-swap liveness check uses the maximum permitted gas-swap input instead of the actual input consumed by the exact-output swap. amountInMax is only a cap; it is not the amount the router necessarily spends.

**Pre_conditions**  
A different-gas-token route reaches _swapAndSendERC20Tokens. The exact gas-token quote is Q and the configured slippage cushion makes amountInMax = Q + slippage * Q / 1000. The route output targetAmount is greater than Q but less than or equal to amountInMax.

**Impact**  
Low-margin different-gas-token withdrawals and cross-chain settlements can be rejected even though the exact gas-token purchase succeeds and would leave a nonzero target-token amount for GatewayZEVM. This is repeatable for the affected amount band and blocks a valid settlement boundary instead of using the actual router spend.

**Mitigation**  
After the exact-output swap, require amounts[0] < targetAmount and compute the withdraw amount from targetAmount - amounts[0]. If a minimum final output is desired, carry it explicitly in the route payload instead of deriving it from the gas-swap max-spend cushion.

### [M-06] ZEVM outbound aborts have no recovery target

**Severity**: Medium  
**Likelihood**: Low  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:252; omni-chain-contracts/contracts/GatewayCrossChain.sol:261; omni-chain-contracts/contracts/GatewayCrossChain.sol:283; omni-chain-contracts/contracts/GatewayCrossChain.sol:287; omni-chain-contracts/contracts/GatewayCrossChain.sol:551; omni-chain-contracts/contracts/GatewayTransferNative.sol:261; omni-chain-contracts/contracts/GatewayTransferNative.sol:270; omni-chain-contracts/contracts/GatewayTransferNative.sol:292; omni-chain-contracts/contracts/GatewayTransferNative.sol:296; omni-chain-contracts/contracts/GatewayTransferNative.sol:645; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/Revert.sol:4; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:443

**Summary/Description**  
GatewayCrossChain and GatewayTransferNative both implement onAbort to store a RefundInfo for aborted cross-chain transactions, but their ZEVM outbound helpers set RevertOptions.abortAddress to address(0). The Zeta RevertOptions type defines abortAddress as the address that receives funds if the cross-chain transaction is aborted, and GatewayZEVM.executeAbort rejects a zero target before calling Abortable(target).onAbort. As a result, if either wrapper outbound withdraw or withdrawAndCall reaches the abort state after destination/revert recovery cannot complete, the contract-owned onAbort refund path is unreachable.

**Root Cause**  
The outbound RevertOptions bind revertAddress to address(this) but leave abortAddress as address(0), even though the wrappers are the contracts that implement the abort refund handler and GatewayZEVM requires a nonzero abort target.

**Pre_conditions**  
GatewayCrossChain.onCall or GatewayTransferNative.onCall/withdrawToNativeChain processes a route and initiates a GatewayZEVM.withdraw or GatewayZEVM.withdrawAndCall. The outbound cross-chain transaction cannot complete normally and the revert-handling leg cannot complete, causing protocol abort handling to use the recorded RevertOptions.abortAddress.

**Impact**  
Affected funds are not recorded in refundInfos through onAbort and cannot be recovered through claimRefund or the configured bot flow. Depending on protocol handling, the abort execution reverts on the zero target or recovery value is assigned to an unusable recipient, leaving the user output/refund stuck outside the intended DEX refund path.

**Mitigation**  
Set abortAddress to address(this) for GatewayCrossChain and GatewayTransferNative outbound withdraw and withdrawAndCall RevertOptions, and validate that all outbound recovery targets are nonzero. Keep onAbort tolerant of all route recipient encodings so abort handling can always store a claimable refund record.

### [I-01] Direct ERC20 deposits can retain stray native value

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:287; omni-chain-contracts/contracts/GatewaySend.sol:315; omni-chain-contracts/contracts/GatewaySend.sol:406

**Summary/Description**  
GatewaySend.depositAndCall direct is payable for both native and ERC20 routes. In the ERC20 branch, the contract pulls and deposits only the ERC20 amount, while any attached native msg.value is neither forwarded to GatewayEVM nor refunded and remains in GatewaySend.

**Root Cause**  
The direct ERC20 branch does not require msg.value == 0, and GatewaySend exposes receive/fallback but no native sweep, refund, or per-route accounting path.

**Pre_conditions**  
A caller invokes the direct deposit overload with asset not equal to the ETH sentinel and sends a nonzero msg.value.

**Impact**  
The attached native value is not forwarded through GatewayEVM, is not refunded, and remains as residual contract balance. The confirmed impact is caller-funded/user-mistake loss rather than a third-party asset-loss path, so this remains an Info-level note.

**Mitigation**  
For non-ETH direct deposits, require msg.value == 0. If passive native value is intentionally supported, add explicit accounting and a controlled recovery/refund path.

### [I-02] GatewaySend dstChainId is not bound to bridged payload

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:218; omni-chain-contracts/contracts/GatewaySend.sol:248; omni-chain-contracts/contracts/GatewaySend.sol:287; omni-chain-contracts/contracts/GatewaySend.sol:297; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:56; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:69

**Summary/Description**  
GatewaySend.depositAndCall accepts a dstChainId argument but builds the cross-chain message as externalId || payload. GatewayCrossChain later decodes dstChainId from the first four bytes of that payload, so the GatewaySend argument is only event metadata and is not part of the executable route.

**Root Cause**  
Destination chain id is duplicated across API metadata and caller-supplied payload without a binding check. GatewaySend neither prepends dstChainId to payload nor verifies that a GatewayCrossChain payload starts with the same dstChainId argument.

**Pre_conditions**  
A caller routes through GatewaySend to GatewayCrossChain and supplies a payload missing the four-byte destination chain id, or supplies a payload whose encoded chain id differs from the dstChainId argument.

**Impact**  
A missing chain id causes the ZEVM decoder to treat the first four bytes of the next field as dstChainId and misparse the rest of the route, typically failing and relying on revert handling. A mismatched chain id can make EddyCrossChainSend telemetry disagree with the route actually executed. Current proof is caller-funded failed or misreported routing, not third-party asset loss.

**Mitigation**  
Make the payload schema canonical at the GatewaySend boundary: either prepend dstChainId when building the message, or decode the first four payload bytes for GatewayCrossChain routes and require it to equal the dstChainId argument. Consider separate APIs for Zeta-local native payloads that do not contain a destination chain id.

### [I-03] GatewaySend onCall projects malformed recipient bytes into an EVM address

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:341; omni-chain-contracts/contracts/GatewaySend.sol:356; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:40

**Summary/Description**  
GatewaySend.onCall decodes receiver as variable-length bytes from the packed destination message, but unconditionally derives the payout address with address(bytes20(recipient)). Exact 20-byte packed EVM addresses settle correctly, while 32-byte ABI-encoded addresses, empty bytes, Solana/BTC-style bytes, or any overlong value are silently projected to the first 20 bytes and paid as an EVM address.

**Root Cause**  
The destination settlement path does not validate that the recipient field is exactly the 20-byte packed EVM address format before converting it to an address.

**Pre_conditions**  
A gateway-authenticated destination message reaches GatewaySend.onCall with recipient.length != 20. The receiver bytes originate from the route payload preserved by SwapDataHelperLib.buildOutputMessage.

**Impact**  
Funds can be delivered to an unintended EVM address or, for native payouts, irrecoverably sent to the zero address or another projected address. Current code proof shows sender/route-producer induced misdelivery rather than theft of third-party funds, so this is classified as Info.

**Mitigation**  
Require recipient.length == 20 before payout in GatewaySend.onCall, or include an explicit destination address-domain tag and reject unsupported domains on the EVM settlement contract. Decode 32-byte ABI addresses only if that format is intentionally supported.

### [I-04] GatewaySend destination swap slippage applies to an unbound DODO amount

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:341; omni-chain-contracts/contracts/GatewaySend.sol:362; omni-chain-contracts/contracts/GatewaySend.sol:366; omni-chain-contracts/contracts/GatewaySend.sol:195; omni-chain-contracts/contracts/GatewaySend.sol:205; omni-chain-contracts/contracts/GatewaySend.sol:207

**Summary/Description**  
GatewaySend.onCall decodes the gateway-delivered destination amount from the authenticated message, but the swap branch calls _doMixSwap and forwards the user-supplied DODO params.fromTokenAmount and params.minReturnAmount without requiring params.fromTokenAmount == amount. The DODO slippage check is therefore applied to the route-data amount rather than the amount GatewaySend just received for settlement.

**Root Cause**  
The destination settlement path treats the authenticated callback amount and the DODO route amount as independent fields. It does not bind params.fromTokenAmount to the decoded amount before approving and calling DODORouteProxy.mixSwap.

**Pre_conditions**  
A gateway-authenticated destination message reaches GatewaySend.onCall with fromToken != toToken and crafted swapData whose params.fromTokenAmount differs from the decoded amount. The message bytes are preserved from the route payload carried through SwapDataHelperLib.buildOutputMessage.

**Impact**  
If params.fromTokenAmount is lower than amount, only that smaller amount is swapped and the remaining delivered fromToken stays in GatewaySend while the recipient receives output for the smaller swap. If params.fromTokenAmount is higher than amount and GatewaySend has residual fromToken balance, the route can spend those residual tokens through DODO. Current proof depends on route-producer error or pre-existing residual balances, so this is recorded as Info rather than a standalone Medium/High finding.

**Mitigation**  
In GatewaySend.onCall, decode the destination DODO params before swapping and require params.fromToken == fromToken and params.fromTokenAmount == amount for non-empty destination swaps. Consider also enforcing params.toToken == toToken and proving the toToken balance delta before payout so the returned amount is backed by freshly received output.

### [I-05] GatewaySend native payouts use transfer and fail contract receivers

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:341; omni-chain-contracts/contracts/GatewaySend.sol:369; omni-chain-contracts/contracts/GatewaySend.sol:370

**Summary/Description**  
GatewaySend.onCall pays native ETH outputs with payable(evmWalletAddress).transfer(outputAmount). A gateway-authenticated native-output settlement to a valid payable contract receiver can revert if the receiver's receive/fallback needs more than the 2300-gas stipend or otherwise rejects direct ETH. The callback then reverts at final payout instead of completing settlement or recording an alternate claim/refund path.

**Root Cause**  
The native destination payout branch uses Solidity transfer as the final settlement primitive and does not provide a call-style payout, wrapped-native fallback, or pull-payment escrow for contract recipients that cannot accept transfer.

**Pre_conditions**  
A gateway-authenticated GatewaySend.onCall reaches the native payout branch with toToken equal to the ETH sentinel, outputAmount established, and recipient bytes projecting to a contract address whose receive/fallback fails under transfer.

**Impact**  
Native-output routes to those contract recipients cannot settle through GatewaySend; GatewayEVM.execute reverts during the callback and the route must rely on retry, revert, or manual recovery semantics. The recipient is route-selected and no third-party extraction path was proven, so this is recorded as Info.

**Mitigation**  
Use a checked low-level call or TransferHelper.safeTransferETH for native payouts, and define a fallback for failed recipient calls such as storing a claimable balance, refunding through the gateway, or delivering wrapped native value. If only EOAs are intended, reject contract recipients before dispatch and document that constraint.

### [I-06] GatewayCrossChain onCall does not reject duplicate externalIds

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:442; omni-chain-contracts/contracts/GatewayCrossChain.sol:449; omni-chain-contracts/contracts/GatewayCrossChain.sol:460; omni-chain-contracts/contracts/GatewayCrossChain.sol:467

**Summary/Description**  
GatewayCrossChain.onCall decodes externalId from gateway-delivered message bytes and immediately performs fee transfer, optional DODO swap, and GatewayZEVM withdrawal without recording that the externalId has already been processed. A duplicate delivery of the same message would execute the side effects again and emit another lifecycle event with the same correlation id.

**Root Cause**  
The inbound handler treats externalId only as telemetry and revert-correlation data. It has no processedExternalIds mapping and does not bind the id to context.chainID, context.sender, zrc20, amount, or the message hash before executing side effects.

**Pre_conditions**  
The configured GatewayZEVM calls GatewayCrossChain.onCall more than once with the same externalId/message, or an upstream route producer/gateway path supplies an already-used externalId. The second call has enough token balance or freshly delivered tokens to reach the side-effecting fee/swap/withdraw branches.

**Impact**  
Duplicate gateway delivery can charge fees, perform swaps, and initiate outbound withdrawals more than once under the same externalId. If each replay is backed by fresh gateway-delivered tokens, the direct loss is limited to duplicate user-funded execution and accounting ambiguity; if a gateway/protocol fault replays without fresh backing, contract-held residual or refund-backed balances can be consumed by the repeated path. Because untrusted users cannot directly call onCall, this is classified as Info under the current trust model.

**Mitigation**  
Add an idempotency guard such as mapping(bytes32 => bytes32) processedMessages, compute a digest over context.chainID, context.sender, zrc20, amount, and message, reject reused externalIds with a different digest, and mark the id consumed before external calls. If legitimate retries are required after a revert, store an explicit status machine instead of allowing successful side effects to repeat.

### [I-07] GatewaySend onRevert trusts unbound refund messages

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:393; omni-chain-contracts/contracts/GatewaySend.sol:253; omni-chain-contracts/contracts/GatewaySend.sol:303; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/Revert.sol:18

**Summary/Description**  
GatewaySend creates canonical revert messages as externalId || bytes20(msg.sender), but onRevert does not verify that the callback belongs to a GatewaySend-originated deposit. It ignores RevertContext.sender, does not track issued externalIds, and does not require context.revertMessage.length == 52 before decoding the refund recipient with bytes20(context.revertMessage[32:]). A gateway-authenticated callback with foreign or malformed revertMessage bytes can therefore be processed as a GatewaySend refund and can project the recipient to the first 20 bytes, the zero address, or revert during slicing.

**Root Cause**  
The revert callback relies only on msg.sender == gateway and treats arbitrary RevertContext.revertMessage bytes as a valid GatewaySend refund envelope without origin, externalId, or length binding.

**Pre_conditions**  
GatewayEVM calls GatewaySend.onRevert with a revertMessage not produced by GatewaySend.depositAndCall, for example a direct GatewayEVM deposit configured with GatewaySend as revertAddress, or a malformed callback envelope.

**Impact**  
For ERC20 reverts this does not currently prove third-party fund extraction because GatewayEVM.revertWithERC20 transfers the reverted amount into GatewaySend before calling onRevert. The issue can still misroute or fail a user-funded refund and pollute refund telemetry; native refund loss is covered separately by H-03.

**Mitigation**  
Require context.sender == address(this) for GatewaySend-originated reverts, require context.revertMessage.length == 52 before decoding, and consider storing issued externalId to expected sender/asset/amount so onRevert can consume only known pending refunds.

### [I-08] GatewayCrossChain accepts non-canonical route bytes

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:56; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:79; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:95; omni-chain-contracts/contracts/GatewayCrossChain.sol:448; omni-chain-contracts/contracts/GatewayCrossChain.sol:453; omni-chain-contracts/contracts/GatewayCrossChain.sol:468; omni-chain-contracts/contracts/GatewayCrossChain.sol:490

**Summary/Description**  
GatewayCrossChain.onCall decodes a length-prefixed route message through SwapDataHelperLib.decodeMessage, but the decoder never requires the consumed offset after accounts to equal message.length. Bytes after the declared accounts section are accepted and ignored. The same non-canonical parsing also lets Bitcoin routes carry ignored contractAddress, swapDataB, or accounts fields, and lets non-Solana EVM routes carry ignored accounts bytes. The callback can therefore process a message as successful while surplus or branch-incompatible destination payload bytes are dropped.

**Root Cause**  
The packed DecodedMessage schema is shared across Bitcoin, EVM, and Solana modes, but decodeMessage does not enforce exact total length and GatewayCrossChain does not enforce mode-specific field invariants before dispatching. It does not require the final parser offset to equal message.length, does not require unused fields to be empty, and has no explicit route mode that distinguishes plain withdrawal from contract-call payloads.

**Pre_conditions**  
GatewayZEVM calls GatewayCrossChain.onCall with a message whose decoded length fields fit within the route body but do not consume it exactly, or whose dstChainId selects Bitcoin while contractAddress, swapDataB, or accounts are non-empty, or selects a non-Solana EVM route while accounts is non-empty. The route otherwise decodes and reaches the withdrawal branch.

**Impact**  
The proven impact is route-local non-canonical execution and misleading settlement metadata: surplus bytes can be accepted but ignored, and destination call/swap/account bytes supplied in the message can be silently dropped for the selected branch. No direct third-party fund extraction was proven from trailing bytes alone, so this remains Info.

**Mitigation**  
Validate decoded messages before side effects. In SwapDataHelperLib.decodeMessage, require message.length == 36 + senderLen + receiverLen + swapDataZLen + contractAddressLen + swapDataBLen + accountsLen. In GatewayCrossChain, require Bitcoin routes to have empty contractAddress, swapDataB, and accounts; require non-Solana EVM routes to have empty accounts; and define explicit plain-withdraw versus withdrawAndCall modes instead of relying on ignored fields.

### [I-09] Outbound withdrawals accept unsupported or mismatched destination declarations

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:24; omni-chain-contracts/contracts/GatewayCrossChain.sol:398; omni-chain-contracts/contracts/GatewayCrossChain.sol:427; omni-chain-contracts/contracts/GatewayCrossChain.sol:468; omni-chain-contracts/contracts/GatewayTransferNative.sol:25; omni-chain-contracts/contracts/GatewayTransferNative.sol:491; omni-chain-contracts/contracts/GatewayTransferNative.sol:520; omni-chain-contracts/contracts/GatewayTransferNative.sol:563; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:56

**Summary/Description**  
SwapDataHelperLib.decodeMessage accepts dstChainId and targetZRC20 as independent message fields. In both ZEVM outbound handlers, dstChainId == BITCOIN_EDDY selects the Bitcoin withdraw path, dstChainId == SOLANA_EDDY only enables Solana AccountEncoder wrapping, and every other value selects the generic EVM/Solana withdrawAndCall path. Because GatewayZEVM ZRC20 withdrawals receive targetZRC20 but no dstChainId, the actual external chain follows the supplied ZRC20 while the wrapper's branch, Solana payload envelope, and events follow the message-provided dstChainId. A Solana targetZRC20 paired with any non-SOLANA_EDDY route ID therefore reaches a Solana outbound call without the required Solana account envelope.

**Root Cause**  
Destination dispatch uses hardcoded app-specific chain ID constants as route classifiers while trusting message-provided targetZRC20 for the GatewayZEVM withdrawal. There is no supported-chain allowlist, no chain-family registry, no configurable Solana classifier, and no check that decoded.dstChainId matches the external chain represented by decoded.targetZRC20. The scoped IZRC20 interface also exposes no CHAIN_ID view for a local binding check.

**Pre_conditions**  
A GatewayCrossChain.onCall message or GatewayTransferNative.withdrawToNativeChain call supplies an unsupported dstChainId, supplies a supported dstChainId with a targetZRC20 for another external chain, or supplies a Solana targetZRC20 while the final-route classifier has drifted away from the hardcoded SOLANA_EDDY value 1399811149. The route otherwise has enough output to reach GatewayZEVM withdrawal.

**Impact**  
The proven impact is route-integrity failure and possible self-selected misdelivery or failed settlement: a malformed or drifted route can be processed under one declared destination while GatewayZEVM withdraws according to the supplied ZRC20 token. In the Solana subcase, the outbound message omits AccountEncoder account metadata and is likely unusable by the Solana destination program. If the outbound later fails, refund-domain issues overlap with M-02 and the existing Solana refund-domain obligations. No direct third-party asset extraction was proven from this mismatch alone, so this remains Info.

**Mitigation**  
Reject unsupported dstChainId values before fee transfer, DODO swap, approval, or gateway withdrawal. Bind every supported dstChainId to an allowed targetZRC20 set or chain-family registry, and make the Solana route classifier configurable or derived from the same registry rather than a hardcoded Eddy DB value. For Solana target ZRC20s, require the Solana account envelope regardless of off-chain classifier spelling; for Bitcoin, require the target ZRC20 to be a Bitcoin-chain asset before calling GatewayZEVM.withdraw.

### [I-10] GatewayTransferNative accepts non-canonical destination message encodings

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayTransferNative.sol:535; omni-chain-contracts/contracts/GatewayTransferNative.sol:548; omni-chain-contracts/contracts/GatewayTransferNative.sol:563; omni-chain-contracts/contracts/GatewayTransferNative.sol:585; omni-chain-contracts/contracts/GatewayTransferNative.sol:491; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:56

**Summary/Description**  
GatewayTransferNative.withdrawToNativeChain accepts a caller-supplied packed route message and decodes dstChainId, receiver, contractAddress, swapDataB, and accounts without enforcing a canonical schema for the selected destination mode. Bitcoin routes can carry ignored contract-call, destination-swap, or account bytes; non-Solana routes can carry ignored account bytes; unsupported non-Bitcoin dstChainId values enter the EVM/Solana withdrawAndCall path; and EVM contract-call routes pass nonzero contractAddress bytes of any length to GatewayZEVM, which only checks that the receiver bytes are non-empty.

**Root Cause**  
The shared DecodedMessage byte schema is parsed with caller-controlled length prefixes, but withdrawToNativeChain routes only on dstChainId == BITCOIN_EDDY and does not validate exact total length, supported chain id, address byte lengths, or mode-specific empty fields before fee, swap, approval, and gateway withdrawal logic.

**Pre_conditions**  
A caller invokes withdrawToNativeChain with a message whose declared length fields fit within message.length but whose branch-specific fields are non-canonical, such as extra trailing bytes, non-empty ignored fields, unsupported dstChainId, non-Solana account bytes, or an EVM contractAddress length other than 20. The route otherwise reaches the withdrawal branch.

**Impact**  
The proven impact is route-local misexecution and misleading settlement metadata: bytes supplied by the caller can be silently ignored, unsupported destinations can be processed as non-Bitcoin routes, and malformed contract receiver bytes can be emitted to GatewayZEVM for downstream handling. A stronger fund-loss outcome is covered by the existing non-20-byte refund claim issue when such malformed routes later create stored refunds, so this standalone entry remains Info.

**Mitigation**  
Validate decoded messages before side effects. Require message.length to equal the fixed header plus all declared sections, reject unsupported dstChainId values, bind targetZRC20 to the declared destination chain, require EVM receiver or contractAddress fields to be exactly 20 bytes, require unused fields to be empty for the selected mode, and define explicit plain-withdraw versus contract-call modes.

### [I-11] GatewayTransferNative onCall projects malformed native receiver bytes into an EVM address

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:115; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:121; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:127; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:129; omni-chain-contracts/contracts/GatewayTransferNative.sol:369; omni-chain-contracts/contracts/GatewayTransferNative.sol:374; omni-chain-contracts/contracts/GatewayTransferNative.sol:378; omni-chain-contracts/contracts/GatewayTransferNative.sol:404

**Summary/Description**  
SwapDataHelperLib.decodeNativeMessage parses the native payload receiver as an arbitrary length-prefixed byte string and does not enforce the 20-byte EVM address shape required by GatewayTransferNative.onCall. The caller then derives the payout target with address(uint160(bytes20(decoded.receiver))). Exact 20-byte packed receivers settle correctly, but empty or short receiver bytes are zero-padded and overlong or ABI-encoded receiver bytes are truncated to the first 20 bytes before funds are sent.

**Root Cause**  
The packed native-message schema allows variable-length receiver bytes, while the local ZEVM settlement path treats that field as an EVM address without a length or encoding-mode check.

**Pre_conditions**  
A gateway-authenticated onCall reaches GatewayTransferNative with a native payload whose targetZRC20 and declared length fields fit within message.length, but whose receiverLen is not exactly 20. The route reaches the same-token payout branch or the post-swap payout branch.

**Impact**  
Malformed receiver encodings can silently send the settled output to a projected address that differs from the intended receiver, or to the zero address for empty receiver bytes. The proven impact is route-local misdelivery and misleading settlement metadata from a malformed payload; no stronger third-party fund-loss path was proven for this decodeNativeMessage issue.

**Mitigation**  
Validate decoded native messages before side effects. For GatewayTransferNative.onCall's local EVM payout path, require decoded.receiver.length == 20 before casting to bytes20, reject empty receivers, and consider rejecting non-canonical sender lengths or moving to an explicit typed schema for each destination mode.

### [I-12] SwapDataHelperLib.decodeMessage panics on short route payloads

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:56; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:79; omni-chain-contracts/contracts/GatewayCrossChain.sol:448; omni-chain-contracts/contracts/GatewayCrossChain.sol:453; omni-chain-contracts/contracts/GatewayTransferNative.sol:535; omni-chain-contracts/contracts/GatewayTransferNative.sol:548

**Summary/Description**  
SwapDataHelperLib.decodeMessage assumes the packed route body contains the full 36-byte fixed header before any length-prefixed fields. It reads the header with assembly and then immediately slices message[36:36+senderLen]. For any route body shorter than 36 bytes, the first slice is out of bounds even when the decoded length fields are zero, so malformed gateway or withdrawToNativeChain payloads revert with a compiler bounds-check panic instead of a controlled validation error.

**Root Cause**  
The packed decoder does not validate message.length >= 36, nor does it validate that message.length equals the fixed header plus all declared section lengths before slicing length-prefixed sections.

**Pre_conditions**  
A caller supplies a malformed DecodedMessage route body shorter than 36 bytes. In GatewayCrossChain.onCall this corresponds to a gateway message with the 32-byte externalId prefix followed by fewer than 36 route bytes; if the whole gateway message is shorter than 32 bytes it reverts earlier at the externalId slice. In GatewayTransferNative.withdrawToNativeChain the public message argument itself is shorter than 36 bytes.

**Impact**  
The proven impact is route-local failure for malformed payloads. In withdrawToNativeChain, any ERC20 transferFrom and globalNonce increment are reverted with the transaction. In GatewayCrossChain.onCall, decoding happens before fee transfer, DODO swap, approvals, or withdrawal, and the GatewayZEVM call bubbles the revert before target-side state changes. No third-party extraction or system-wide blocking was proven, so this is recorded as Info.

**Mitigation**  
Validate the fixed header before assembly-derived field use, then validate the declared dynamic lengths before slicing. Prefer a custom InvalidMessageLength error and require message.length == 36 + senderLen + receiverLen + swapDataZLen + contractAddressLen + swapDataBLen + accountsLen for canonical route messages.

### [I-13] AccountEncoder accepts underlength Solana account blobs

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/libraries/AccountEncoder.sol:19; omni-chain-contracts/contracts/libraries/AccountEncoder.sol:24; omni-chain-contracts/contracts/libraries/AccountEncoder.sol:35; omni-chain-contracts/contracts/libraries/AccountEncoder.sol:40; omni-chain-contracts/contracts/libraries/AccountEncoder.sol:44; omni-chain-contracts/contracts/GatewayCrossChain.sol:398; omni-chain-contracts/contracts/GatewayTransferNative.sol:491

**Summary/Description**  
AccountEncoder.decompressAccounts reads the account count with mload at the start of decoded.accounts before validating that the blob contains the two-byte count prefix, then allocates and iterates len Account entries without checking that 2 + len * 33 bytes are present. The outer message parser only bounds the copied accounts blob by accountsLen, so a Solana route can provide an empty/one-byte or otherwise underlength account blob that drives count/entry decoding from bytes outside the declared account payload.

**Root Cause**  
The decoder has no canonical length check before assembly decoding. It should first require input.length >= 2, decode the uint16 count, and require input.length == 2 + uint256(len) * 33 or at least >= that amount before reading each public key and writable flag.

**Pre_conditions**  
A caller supplies a decoded dstChainId equal to SOLANA_EDDY and controls decoded.accounts through SwapDataHelperLib.decodeMessage. The account blob is shorter than the two-byte count prefix, or its decoded count requires more 33-byte entries than the blob contains.

**Impact**  
Malformed Solana routes can consume excessive gas, revert before GatewayZEVM.withdrawAndCall, or ABI-wrap account metadata/counts derived from memory outside the declared account blob. I did not prove a stronger third-party fund-loss path; the current impact is non-canonical malformed-input handling in the caller's route.

**Mitigation**  
Reject malformed account blobs before assembly decoding: require input.length >= 2, decode the uint16 count, require input.length == 2 + uint256(len) * 33 or at least >= that value if trailing bytes are intentionally allowed, and consider a maximum count consistent with GatewayZEVM.MAX_MESSAGE_SIZE.

### [I-14] AccountEncoder reads Solana writable flags from trailing bytes

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/libraries/AccountEncoder.sol:44; omni-chain-contracts/contracts/GatewayCrossChain.sol:398; omni-chain-contracts/contracts/GatewayCrossChain.sol:427; omni-chain-contracts/contracts/GatewayTransferNative.sol:491; omni-chain-contracts/contracts/GatewayTransferNative.sol:520

**Summary/Description**  
AccountEncoder.decompressAccounts advances by one byte for each Solana writable flag, but it converts the flag with iszero(iszero(mload(ptr))). mload(ptr) reads 32 bytes starting at the one-byte flag position, so a zero flag is decoded as true whenever any of the following 31 bytes are nonzero. In a multi-account envelope, a read-only flag before a nonzero next public key will therefore be ABI-wrapped as writable.

**Root Cause**  
The decoder treats a one-byte packed boolean field as 32-byte truthiness instead of reading only byte(0, mload(ptr)) or the exact flag byte. The account envelope also lacks canonical validation that writable flags are exactly 0 or 1.

**Pre_conditions**  
A caller supplies a Solana destination route where decoded.dstChainId equals SOLANA_EDDY and controls decoded.accounts. At least one account has a 0x00 writable byte followed within the next 31 bytes by nonzero account-envelope data, such as the next account public key.

**Impact**  
Malformed or even nominally read-only multi-account envelopes can be forwarded with different writable metadata than the packed bytes declare. I did not prove standalone fund loss in the scoped Solidity contracts because the route caller already controls the Solana account list and destination semantics, but the encoder can misrepresent account permissions to the Solana-side program and should reject or decode the envelope canonically.

**Mitigation**  
Decode the writable flag from one byte only, for example let flag := byte(0, mload(ptr)); require flag == 0 or flag == 1 if canonical encoding is required; and combine this with an exact length check input.length == 2 + uint256(len) * 33 before decoding.

### [I-15] Outbound contract calls accept caller-selected target contracts

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:34; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:56; omni-chain-contracts/contracts/GatewayCrossChain.sol:244; omni-chain-contracts/contracts/GatewayCrossChain.sol:402; omni-chain-contracts/contracts/GatewayCrossChain.sol:431; omni-chain-contracts/contracts/GatewayTransferNative.sol:253; omni-chain-contracts/contracts/GatewayTransferNative.sol:495; omni-chain-contracts/contracts/GatewayTransferNative.sol:524

**Summary/Description**  
DecodedMessage.contractAddress is parsed from caller-controlled route bytes and is passed unchanged as the receiver for GatewayZEVM.withdrawAndCall in both ZEVM outbound handlers. The handlers set isArbitraryCall to false, which requests an authenticated destination-chain callback, but they do not require the target to be the intended GatewaySend contract, a chain-specific allowlisted receiver, or even an exact 20-byte EVM contract address for EVM routes.

**Root Cause**  
The route schema treats contractAddress as data, while the dispatch logic treats it as the authenticated destination-call target without binding it to dstChainId, a trusted destination gateway, or a canonical address format. GatewayZEVM only rejects an empty receiver and oversized messages before emitting the outbound call.

**Pre_conditions**  
A GatewayCrossChain.onCall payload or GatewayTransferNative.withdrawToNativeChain message reaches the non-Bitcoin branch with non-empty contractAddress bytes and enough output to pay the destination gas fee. The selected targetZRC20 determines the actual connected chain, while contractAddress selects the destination callback receiver.

**Impact**  
The proven impact is route-integrity failure and unintended authenticated target selection: user-supplied route bytes can make the protocol emit an authenticated withdrawAndCall to a destination contract outside the intended DODO GatewaySend path. Without evidence that an in-scope third-party balance can be extracted solely from this target selection, the issue is kept at Info; stronger fund-loss paths through route amount/token binding are tracked separately.

**Mitigation**  
Validate contractAddress before any fee, swap, approval, or gateway withdrawal side effects. For EVM contract-call routes, require contractAddress.length == 20 and require the address to match the configured destination GatewaySend or a trusted allowlist for the selected destination chain. For plain withdrawals, call GatewayZEVM.withdraw instead of withdrawAndCall; for non-EVM routes, enforce the chain-specific address format and reject incompatible target fields.

### [I-16] Direct ETH deposits underreport bridged native value

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:307; omni-chain-contracts/contracts/GatewaySend.sol:309; omni-chain-contracts/contracts/GatewaySend.sol:329

**Summary/Description**  
GatewaySend.depositAndCall direct accepts ETH when msg.value is greater than or equal to amount, but forwards the full msg.value to GatewayEVM while EddyCrossChainSend records amount as both amount and outputAmount. For overpaid native calls, the protocol event underreports the value actually sent through the gateway.

**Root Cause**  
The ETH branch validates msg.value >= amount instead of msg.value == amount and passes msg.value into _handleETHDeposit, while the event uses the independent amount parameter.

**Pre_conditions**  
A caller invokes the direct deposit overload with asset equal to the ETH sentinel and msg.value greater than amount.

**Impact**  
The extra native value is bridged through GatewayEVM but GatewaySend accounting events report only the smaller amount. Current proof is limited to caller-funded overpayment and inaccurate event accounting; destination execution receives the gateway-delivered amount, so no third-party fund-loss path was verified.

**Mitigation**  
Require msg.value == amount for direct ETH deposits, or emit msg.value as the actual sent/output amount and make the parameter semantics explicit.

### [I-17] GatewaySend onCall return ABI does not match Callable

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:341; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/evm/interfaces/IGatewayEVM.sol:229; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/evm/GatewayEVM.sol:431

**Summary/Description**  
GatewaySend.onCall has the same selector as ZetaChain Callable.onCall but declares returns (bytes4) instead of returns (bytes memory). The current body returns the empty bytes4 value, which Solidity decodes as empty dynamic bytes when GatewayEVM calls through Callable, so this is not a delivery-breaking issue in the current code. The mismatch still leaves the hook ABI non-compliant and fragile.

**Root Cause**  
GatewaySend does not implement the Callable interface, so the compiler does not enforce the expected dynamic bytes return type.

**Pre_conditions**  
GatewayEVM performs an authenticated call to GatewaySend.onCall and GatewaySend returns the current empty bytes4 value.

**Impact**  
No practical settlement DoS was verified for the current implementation because the zero fixed-bytes return decodes as empty bytes. The risk is interface fragility and future regression if the function returns a non-empty bytes4 value or external tooling relies on the generated GatewaySend ABI.

**Mitigation**  
Import and implement Callable, change GatewaySend.onCall to returns (bytes memory), and return bytes("") explicitly.

### [I-18] GatewaySend swapped ETH deposits over-forward msg.value to DODO

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:195; omni-chain-contracts/contracts/GatewaySend.sol:202; omni-chain-contracts/contracts/GatewaySend.sol:231; omni-chain-contracts/contracts/GatewaySend.sol:234; omni-chain-contracts/contracts/GatewaySend.sol:245

**Summary/Description**  
The swapped GatewaySend.depositAndCall overload accepts native input when msg.value is greater than or equal to amount, but _doMixSwap forwards the entire msg.value to DODORouteProxy.mixSwap. The amount parameter and DODO params.fromTokenAmount can be smaller than the native value sent to DODO.

**Root Cause**  
The ETH intake branch validates msg.value >= amount instead of exact equality, and the shared swap helper uses msg.value as the external call value instead of deriving the value from the actual route input amount or rejecting unrelated native value.

**Pre_conditions**  
A caller invokes the swapped deposit overload with fromToken equal to the ETH sentinel and sends msg.value greater than the intended amount, or otherwise attaches native value that is not meant to be part of the DODO swap.

**Impact**  
The excess native value leaves GatewaySend during the DODO call and is not separately accounted, refunded, or reflected by the amount field in the GatewaySend event. The confirmed impact is caller-funded overpayment or route-local failure depending on DODO's value checks; no standalone third-party asset-loss path was proven beyond the already reported DODO route-binding findings.

**Mitigation**  
For ETH source swaps, require msg.value == amount and require the decoded DODO params.fromToken/fromTokenAmount to match fromToken/amount. Pass value amount only for native-source swaps and require msg.value == 0 for ERC20-source swaps.

### [I-19] GatewaySend ERC20 payouts ignore transfer return values

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:341; omni-chain-contracts/contracts/GatewaySend.sol:363; omni-chain-contracts/contracts/GatewaySend.sol:369; omni-chain-contracts/contracts/GatewaySend.sol:372

**Summary/Description**  
GatewaySend.onCall settles ERC20 outputs with IERC20(toToken).transfer(evmWalletAddress, outputAmount) but does not require the returned bool to be true. Both same-token callbacks and destination-swap callbacks can reach this payout branch, after which the function emits EddyCrossChainReceive and returns success even if a false-return token did not move the funds.

**Root Cause**  
The destination ERC20 payout uses the raw IERC20 interface and discards the transfer return value instead of using TransferHelper.safeTransfer, SafeERC20.safeTransfer, or an explicit require around the returned bool.

**Pre_conditions**  
A gateway-authenticated GatewaySend.onCall reaches the ERC20 payout branch with toToken set to a token that returns false on transfer failure instead of reverting. No repo-local proof was found that the current supported ZetaChain asset set includes such a false-return payout token, so this is kept as code-level Info.

**Impact**  
For a false-return payout token, the recipient receives no ERC20 while GatewaySend emits a successful receive event and the gateway callback does not revert into the cross-chain failure path. The unpaid tokens remain in GatewaySend custody and the route has no automatic refund signal. Current evidence proves the code-level silent-success condition but not an in-scope supported-token trigger.

**Mitigation**  
Replace the raw payout with TransferHelper.safeTransfer(toToken, evmWalletAddress, outputAmount) or SafeERC20.safeTransfer. Keep the native branch separately checked, and for swapped payouts consider verifying the expected toToken balance delta before paying the receiver.

### [I-20] GatewaySend onCall accepts false-return ERC20 intake as successful custody

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:358; omni-chain-contracts/contracts/GatewaySend.sol:359; omni-chain-contracts/contracts/GatewaySend.sol:363; omni-chain-contracts/contracts/GatewaySend.sol:372; omni-chain-contracts/contracts/libraries/TransferHelper.sol:18

**Summary/Description**  
GatewaySend.onCall pulls non-ETH callback assets with a raw IERC20(fromToken).transferFrom(msg.sender, address(this), amount) call and never checks the returned bool. A token that returns false without reverting can leave GatewaySend with no newly received input while the function continues into same-token settlement or destination swap accounting. In the same-token branch, outputAmount is set directly to the decoded amount and the later raw transfer can spend pre-existing GatewaySend token balance or emit a successful EddyCrossChainReceive event even though the inbound pull failed.

**Root Cause**  
The callback treats a non-reverting ERC20 transferFrom call as proof of custody instead of requiring true returndata or checking a balance delta. The scoped TransferHelper.safeTransferFrom helper rejects false returndata, but GatewaySend.onCall does not use it for inbound custody.

**Pre_conditions**  
A gateway-authenticated call reaches GatewaySend.onCall with fromToken set to an ERC20 whose transferFrom can return false without moving funds. For direct value loss, GatewaySend already holds that token as residual or pending value and the route uses fromToken == toToken or otherwise reaches a payout path that can consume the residual balance. Reportability as a stronger finding depends on proving such false-return behavior for a supported ZetaChain destination asset.

**Impact**  
The local accounting can drift from actual custody: the callback records and settles amount as if the token was received even when no transfer occurred. With residual token balance, a crafted callback can pay the receiver from existing GatewaySend funds; without residual balance, the callback can still emit success or proceed into later logic without the promised input. Because the current supported-token set is expected to contain regular ERC20 assets and no concrete whitelisted false-return token was proven here, this is recorded as Info rather than a contest-strength Medium/High.

**Mitigation**  
Use TransferHelper.safeTransferFrom or OpenZeppelin SafeERC20.safeTransferFrom for the inbound pull and TransferHelper.safeTransfer or SafeERC20.safeTransfer for ERC20 payouts. For stronger accounting, verify the expected token balance delta before setting outputAmount or paying the receiver, and reject no-asset gateway calls that declare an ERC20 input amount.

### [I-21] GatewayCrossChain same-gas withdrawals leave residual GatewayZEVM allowance

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:370; omni-chain-contracts/contracts/GatewayCrossChain.sol:389; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:106

**Summary/Description**  
GatewayCrossChain same-gas Bitcoin and EVM/Solana outbound branches approve GatewayZEVM for outputAmount + gasFee, then call GatewayZEVM with amount outputAmount - gasFee. GatewayZEVM pulls the gas fee and the passed withdrawal amount from msg.sender, totaling outputAmount, so a successful same-gas withdrawal leaves gasFee of allowance from GatewayCrossChain to GatewayZEVM.

**Root Cause**  
The same-gas approval is computed as if GatewayZEVM would pull outputAmount plus a separate gas fee, but the amount passed to GatewayZEVM is already net of that gas fee. The approval therefore exceeds the actual transferFrom total.

**Pre_conditions**  
A GatewayCrossChain onCall route reaches the Bitcoin branch or the EVM/Solana branch where decoded.targetZRC20 equals its gas ZRC20, gasFee is positive, and the GatewayZEVM withdrawal succeeds.

**Impact**  
The current in-scope GatewayCrossChain wrappers are internal and only reached through onlyGateway onCall, so no public caller-controlled path was found that spends the leftover allowance directly. The stale allowance is still incorrect approval hygiene and widens dependency risk if the gateway interface or configured gateway can later exercise arbitrary pulls.

**Mitigation**  
Approve GatewayZEVM for the exact total it will pull in same-gas branches, which is outputAmount, or reset the GatewayZEVM allowance to zero after the withdrawal. Keep GatewayCrossChain's raw withdraw wrappers internal.

### [I-22] ZEVM refund callbacks panic on underlength revert messages

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:517; omni-chain-contracts/contracts/GatewayCrossChain.sol:551; omni-chain-contracts/contracts/GatewayTransferNative.sol:612; omni-chain-contracts/contracts/GatewayTransferNative.sol:645

**Summary/Description**  
GatewayCrossChain.onRevert/onAbort, and the duplicated GatewayTransferNative callbacks, slice revertMessage[0:32] and revertMessage[32:] before checking the message length. A gateway-authenticated revert or abort context with revertMessage shorter than 32 bytes therefore panics before the contract can directly refund or store refundInfos. GatewayCrossChain's own outbound helpers always prepend a bytes32 externalId, so this is not reached by the normal in-contract withdrawal envelopes; it remains unsafe for malformed or arbitrary gateway-sourced callback contexts.

**Root Cause**  
The callbacks treat arbitrary RevertContext/AbortContext revertMessage bytes as if they always begin with a 32-byte externalId, and only classify the refund after performing bounds-checked calldata slices.

**Pre_conditions**  
GatewayZEVM invokes onRevert or onAbort on the ZEVM refund contract with revertMessage.length < 32, for example from a malformed direct gateway interaction or integration error. The current GatewayCrossChain withdraw and withdrawAndCall helpers do not generate underlength revert messages because they use externalId || receiver.

**Impact**  
The callback reverts before writing a refund record. If the upstream abort/revert flow has already credited assets to the target contract, those assets are not claimable through claimRefund and require manual reconciliation or owner recovery. Because normal project-generated GatewayCrossChain envelopes include the 32-byte prefix and no third-party extraction was proven, this is recorded as Info.

**Mitigation**  
Validate revertMessage.length before slicing. Require at least 32 bytes for stored refunds and exactly 52 bytes only for explicitly EVM-address refunds; handle underlength messages with a controlled fallback/manual-refund path or reject them before any asset can be stranded.

### [I-23] Real-ZRC20 native exits accept stray native value

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayTransferNative.sol:428; omni-chain-contracts/contracts/GatewayTransferNative.sol:438; omni-chain-contracts/contracts/GatewayTransferNative.sol:535; omni-chain-contracts/contracts/GatewayTransferNative.sol:540; omni-chain-contracts/contracts/GatewayTransferNative.sol:560; omni-chain-contracts/contracts/GatewayTransferNative.sol:687

**Summary/Description**  
GatewayTransferNative.withdrawToNativeChain is payable for both native-sentinel and real-ZRC20 routes. In the real-ZRC20 branch it transfers amount of zrc20 from the caller, but it never rejects nonzero msg.value. Empty swapDataZ leaves the attached native value in GatewayTransferNative, while non-empty swapDataZ forwards the entire msg.value to DODORouteProxy.mixSwap.

**Root Cause**  
The real-ZRC20 exit path lacks a msg.value == 0 gate, and the shared DODO helper blindly uses msg.value as the call value instead of deriving call value from the selected route asset.

**Pre_conditions**  
A caller invokes withdrawToNativeChain with zrc20 set to a real ZRC20, sufficient allowance/balance for amount, and nonzero msg.value.

**Impact**  
The attached native value is not part of the fee, swap output, or GatewayZEVM withdrawal accounting. It can remain as residual native balance or be sent to DODO during a ZRC20 route. The confirmed impact is caller-funded value loss or route-local value drift; no standalone third-party asset-loss path was proven beyond the already reported DODO route-binding findings.

**Mitigation**  
For real-ZRC20 withdrawToNativeChain routes, require msg.value == 0 before fee, DODO, or withdrawal logic. Pass zero value to DODO for ZRC20-source swaps, and only pass native value in an explicit native route that validates exact backing and refund/accounting semantics.

### [I-24] GatewayTransferNative WZETA unwrap reverts for native-rejecting receivers

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayTransferNative.sol:398; omni-chain-contracts/contracts/GatewayTransferNative.sol:400; omni-chain-contracts/contracts/GatewayTransferNative.sol:402; omni-chain-contracts/contracts/GatewayTransferNative.sol:404; omni-chain-contracts/contracts/libraries/TransferHelper.sol:24; omni-chain-contracts/contracts/libraries/TransferHelper.sol:25; omni-chain-contracts/contracts/libraries/TransferHelper.sol:26; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:374; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:375

**Summary/Description**  
GatewayTransferNative.onCall unwraps WZETA outputs to native ZETA and immediately pushes the native value to the decoded receiver through TransferHelper.safeTransferETH. That helper performs a low-level value call and requires success, so a receiver contract that rejects native value or has no payable receive/fallback makes the gateway callback revert after the route has selected native settlement.

**Root Cause**  
The WZETA/native settlement branch only supports push-based native delivery. It does not provide a wrapped-WZETA fallback, pull-payment escrow, or recorded refund path when the selected receiver cannot accept native value.

**Pre_conditions**  
GatewayZEVM calls GatewayTransferNative.onCall with a decoded local settlement where decoded.targetZRC20 == WZETA and the branch reaches the unwrap path. The decoded receiver projects to a contract address whose native-value call reverts or is non-payable.

**Impact**  
The selected native-output route cannot complete for those receivers: TransferHelper.safeTransferETH reverts, GatewayZEVM does not catch the UniversalContract.onCall revert in the scoped Solidity path, and settlement must rely on upstream retry/revert/manual recovery behavior. The receiver is route-selected and no third-party extraction path was proven, so this is recorded as Info rather than Medium/High.

**Mitigation**  
For WZETA native settlement, either reject contract receivers before unwrapping, allow recipients to claim/pull native value, or fall back to transferring WZETA when the native call fails. If native delivery is intended only for EOAs, enforce and document that constraint before DODO swap and unwrap side effects.

### [I-25] GatewayTransferNative feePercent is not capped below the routed amount

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayTransferNative.sol:109; omni-chain-contracts/contracts/GatewayTransferNative.sol:143; omni-chain-contracts/contracts/GatewayTransferNative.sol:343; omni-chain-contracts/contracts/GatewayTransferNative.sol:373; omni-chain-contracts/contracts/GatewayTransferNative.sol:381; omni-chain-contracts/contracts/GatewayTransferNative.sol:556; omni-chain-contracts/contracts/GatewayTransferNative.sol:557

**Summary/Description**  
GatewayTransferNative accepts an owner-configured feePercent without an upper bound, while _handleFeeTransfer computes platformFeesForTx = amount * feePercent / 1000. If feePercent reaches 1000, the fee can consume the full positive route amount; if it exceeds 1000, later post-fee subtraction can revert or the fee transfer can attempt to spend more than the route-local amount.

**Root Cause**  
initialize and setFeePercent store the supplied fee rate directly, and the fee-taking routes rely on amount - platformFeesForTx without enforcing platformFeesForTx < amount for positive-value routes.

**Pre_conditions**  
The trusted Owner initializes or updates GatewayTransferNative with feePercent >= 1000, then a positive-value onCall or withdrawToNativeChain route reaches platform fee logic.

**Impact**  
Same-token local settlement and public native-chain exits can leave zero routed value or revert from uint256 underflow/failed token transfer. In the different-token onCall branch, an excessive fee also increases the residual-balance exposure already reported in H-04 because the branch transfers the fee but still asks DODO to spend the pre-fee amount. The direct trigger is trusted-owner configuration, so this is recorded as Info rather than a primary Medium/High issue.

**Mitigation**  
Validate feePercent during initialize and setFeePercent. For positive-value routes, require platformFeesForTx < amount before transferring fees or routing funds; for example, cap feePercent below 1000 or use an explicit max fee constant matching the intended denominator.

### [I-26] GatewayTransferNative outbound swap events emit post-fee source amounts

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayTransferNative.sol:60; omni-chain-contracts/contracts/GatewayTransferNative.sol:556; omni-chain-contracts/contracts/GatewayTransferNative.sol:557; omni-chain-contracts/contracts/GatewayTransferNative.sol:572; omni-chain-contracts/contracts/GatewayTransferNative.sol:578; omni-chain-contracts/contracts/GatewayTransferNative.sol:592; omni-chain-contracts/contracts/GatewayTransferNative.sol:598

**Summary/Description**  
GatewayTransferNative.withdrawToNativeChain stores the caller-supplied amount in a mutable local variable, transfers the platform fee, then subtracts platformFeesForTx before emitting EddyCrossChainSwap. As a result, the outbound event's amount field records the post-fee source amount, while fees separately records the deducted platform fee. This differs from the natural reading of amount as the gross user input and from other lifecycle logs that use amount as the pre-fee or pre-settlement value.

**Root Cause**  
The outbound path reuses the mutated amount variable in the event instead of preserving and emitting an explicit gross input amount or naming the field as postFeeAmount.

**Pre_conditions**  
A successful GatewayTransferNative.withdrawToNativeChain call executes with feePercent greater than zero and reaches either the Bitcoin or EVM/Solana outbound event emission.

**Impact**  
Off-chain accounting that treats EddyCrossChainSwap.amount as the gross source amount will undercount the user input by the platform fee and can misinterpret fee rates or source volume. The same event includes fees, so gross input can be reconstructed as amount + fees; no direct on-chain asset loss, withdrawal underfunding, or receiver shortfall was verified from this logging ambiguity alone.

**Mitigation**  
Preserve originalAmount before fee subtraction and emit that value as amount, or change the event schema/naming to make postFeeAmount and platformFees explicit. Keep the outbound message/output fields tied to the actual post-fee routed amount.

### [I-27] Bot-claimed 20-byte refunds emit the bot as beneficiary

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:551; omni-chain-contracts/contracts/GatewayCrossChain.sol:572; omni-chain-contracts/contracts/GatewayTransferNative.sol:645; omni-chain-contracts/contracts/GatewayTransferNative.sol:666

**Summary/Description**  
GatewayCrossChain and GatewayTransferNative can store a 20-byte refund walletAddress in onAbort. When a configured bot claims that refund for a different receiver, claimRefund decodes the 20-byte walletAddress and transfers the tokens to that receiver, but EddyCrossChainRefundClaimed emits abi.encodePacked(msg.sender). The claim event therefore records the bot claimant in the walletAddress field instead of the actual payout beneficiary.

**Root Cause**  
claimRefund does not preserve the resolved payout receiver for the claim event and unconditionally emits msg.sender as walletAddress.

**Pre_conditions**  
A gateway abort stores refundInfo with walletAddress.length == 20, and an approved bot rather than the receiver calls claimRefund(externalId).

**Impact**  
Funds are not redirected because the token transfer uses the decoded 20-byte receiver. Off-chain refund accounting or monitoring that treats EddyCrossChainRefundClaimed.walletAddress as the beneficiary will attribute the claim to the bot instead of the user.

**Mitigation**  
Emit refundInfo.walletAddress or abi.encodePacked(receiver) for 20-byte refund claims, and use a separate claimant field if bot/operator attribution is needed.

### [I-28] Refund claim events emit zero token and amount after storage deletion

**Severity**: Info  
**Likelihood**: High  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:572; omni-chain-contracts/contracts/GatewayCrossChain.sol:582; omni-chain-contracts/contracts/GatewayCrossChain.sol:583; omni-chain-contracts/contracts/GatewayCrossChain.sol:585; omni-chain-contracts/contracts/GatewayTransferNative.sol:666; omni-chain-contracts/contracts/GatewayTransferNative.sol:676; omni-chain-contracts/contracts/GatewayTransferNative.sol:677; omni-chain-contracts/contracts/GatewayTransferNative.sol:679

**Summary/Description**  
GatewayCrossChain.claimRefund and GatewayTransferNative.claimRefund bind RefundInfo as a storage reference, transfer refundInfo.token/refundInfo.amount, delete refundInfos[externalId], then emit EddyCrossChainRefundClaimed using refundInfo.token and refundInfo.amount from that same storage reference. After the delete, those fields read as address(0) and 0, so every successful claim log misreports the claimed token and amount.

**Root Cause**  
The claim functions emit event fields from a storage alias after deleting the mapped RefundInfo instead of preserving token and amount in local variables before deletion.

**Pre_conditions**  
A stored refund exists in GatewayCrossChain or GatewayTransferNative, the authorized receiver or bot calls claimRefund, and the token transfer succeeds.

**Impact**  
The on-chain refund transfer is not reduced because it uses the stored values before deletion, but off-chain indexers, reconciliation jobs, support tooling, or liability closeout processes that rely on EddyCrossChainRefundClaimed cannot identify the claimed asset or amount from the claim event and may record a zero-value claim or leave the refund unresolved.

**Mitigation**  
Copy token, amount, and the intended refund wallet/receiver to local variables before transfer and deletion, then emit EddyCrossChainRefundClaimed from those locals. Consider emitting both the original refund wallet and the actual claimant/recipient when bots process manual refunds.

### [I-29] Zero externalId refund records cannot be claimed

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:519; omni-chain-contracts/contracts/GatewayCrossChain.sol:534; omni-chain-contracts/contracts/GatewayCrossChain.sol:553; omni-chain-contracts/contracts/GatewayCrossChain.sol:556; omni-chain-contracts/contracts/GatewayCrossChain.sol:580; omni-chain-contracts/contracts/GatewayTransferNative.sol:614; omni-chain-contracts/contracts/GatewayTransferNative.sol:628; omni-chain-contracts/contracts/GatewayTransferNative.sol:647; omni-chain-contracts/contracts/GatewayTransferNative.sol:650; omni-chain-contracts/contracts/GatewayTransferNative.sol:674

**Summary/Description**  
GatewayCrossChain and GatewayTransferNative store refundInfos under the externalId parsed from the first 32 bytes of revertMessage, including bytes32(0). The claim path then uses refundInfo.externalId != bytes32(0) as the existence check. A stored refund whose key and embedded externalId are zero therefore exists in storage with token, amount, and walletAddress, but claimRefund(bytes32(0)) always reverts with REFUND_NOT_EXIST.

**Root Cause**  
The refund mapping uses the externalId value both as the key/correlation id and as the existence sentinel. The callback writers do not reject externalId == bytes32(0), while claimRefund treats bytes32(0) as nonexistent instead of tracking existence with a separate status flag.

**Pre_conditions**  
GatewayZEVM invokes onRevert or onAbort on GatewayCrossChain or GatewayTransferNative with revertMessage.length >= 32 and the first 32 bytes equal to bytes32(0). For onRevert the message must enter the stored-refund branch, for example by not being exactly 52 bytes. Canonical GatewaySend and GatewayTransferNative-generated ids are hashes and are not expected to be zero; this is reachable through malformed or foreign gateway-sourced recovery messages, or through routes that import an arbitrary externalId into GatewayCrossChain.

**Impact**  
The returned asset is recorded in refundInfos[bytes32(0)] but neither the receiver nor an approved bot can claim it through claimRefund because the existence check fails before transfer. Recovery would require owner/manual intervention such as superWithdraw, so this is weaker than the existing duplicate-externalId overwrite issue and is classified as Info.

**Mitigation**  
Reject externalId == bytes32(0) before storing refund records, or replace the sentinel check with an explicit exists/status field that is set on write and cleared on claim. Prefer binding callbacks to pending records with expected token, amount, and recipient before accepting any refund write.

### [I-30] Manual refund claim events omit the original wallet bytes

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:572; omni-chain-contracts/contracts/GatewayCrossChain.sol:585; omni-chain-contracts/contracts/GatewayTransferNative.sol:666; omni-chain-contracts/contracts/GatewayTransferNative.sol:679

**Summary/Description**  
GatewayCrossChain and GatewayTransferNative store non-20-byte refund walletAddress values for manual refund handling, matching the README model where refund bots collect tokens and process those refunds manually. When claimRefund succeeds, however, EddyCrossChainRefundClaimed emits abi.encodePacked(msg.sender) in the walletAddress field instead of the stored refundInfo.walletAddress. For non-EVM or otherwise non-20-byte refunds, the claim log therefore loses the original beneficiary bytes needed to identify the external-chain recipient.

**Root Cause**  
claimRefund derives a local EVM receiver only for 20-byte walletAddress values and then unconditionally emits msg.sender as the walletAddress field, rather than preserving the stored arbitrary walletAddress bytes in the claim event.

**Pre_conditions**  
A gateway revert or abort stores refundInfo with walletAddress.length != 20, and the refund is later claimed through claimRefund. Under the intended manual-refund model this caller is a configured refund bot; in the current code H-01 also allows any caller to reach the same event path.

**Impact**  
The token transfer path is already covered by the stronger non-20-byte claim authorization issue. This finding is limited to observability: off-chain refund accounting, support tooling, or bot reconciliation that relies on EddyCrossChainRefundClaimed cannot recover the original external-chain wallet bytes from the claim event and may attribute the manual refund release to the claimant address instead of the intended beneficiary.

**Mitigation**  
Copy refundInfo.walletAddress, token, and amount into local variables before deletion and emit the original walletAddress bytes in EddyCrossChainRefundClaimed. If claimant/operator attribution is needed, add a separate claimant field instead of replacing the beneficiary wallet bytes.

### [I-31] ZEVM onCall trusts self-reported sender bytes

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:442; omni-chain-contracts/contracts/GatewayCrossChain.sol:453; omni-chain-contracts/contracts/GatewayCrossChain.sol:477; omni-chain-contracts/contracts/GatewayCrossChain.sol:497; omni-chain-contracts/contracts/GatewayTransferNative.sol:359; omni-chain-contracts/contracts/GatewayTransferNative.sol:369; omni-chain-contracts/contracts/GatewayTransferNative.sol:384; omni-chain-contracts/contracts/GatewayTransferNative.sol:413; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:21; omni-chain-contracts/contracts/libraries/SwapDataHelperLib.sol:28; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/interfaces/UniversalContract.sol:26

**Summary/Description**  
GatewayCrossChain.onCall and GatewayTransferNative.onCall receive an authenticated Zeta MessageContext containing origin, sender, and chainID, but neither handler binds the route payload's sender field to context.origin or context.sender. The handlers decode DecodedMessage.sender or DecodedNativeMessage.sender from arbitrary route bytes and emit it as the source sender, while context is only used for chainID in events. A direct Gateway-originated caller or malformed upstream route can therefore make settlement events attribute the route to arbitrary sender bytes.

**Root Cause**  
The ZEVM handlers treat the route payload as the source-of-truth for sender identity and use only msg.sender == gateway for callback authentication. They do not define whether the canonical source identity is MessageContext.origin, MessageContext.sender, or payload sender bytes, and they do not require equality or a source-app allowlist before side effects and events.

**Pre_conditions**  
GatewayZEVM calls GatewayCrossChain.onCall or GatewayTransferNative.onCall with a payload whose sender bytes differ from the gateway-supplied MessageContext identity. This is reachable for any path that can supply route bytes to the universal contract; meaningful impact requires downstream accounting, monitoring, or support processes to rely on the emitted sender field.

**Impact**  
The directly proven impact is attribution and reconciliation drift: events can claim an arbitrary source sender even though the gateway context identifies a different caller/source. I did not prove a standalone fund-loss path from the sender field alone because value routing uses amount, receiver, target token, and swap fields. The stronger compositional risk from not binding externalId and callback provenance is already reported as H-07.

**Mitigation**  
Define one canonical authenticated source identity per source domain. For EVM-originated GatewaySend routes, require context.sender to equal the deployed GatewaySend for context.chainID and bind decoded.sender to context.origin or remove decoded.sender from trusted telemetry. Include context.chainID, context.sender, zrc20, amount, and a route hash in externalId/refund lifecycle checks.

### [I-32] Hardcoded Bitcoin route id only recognizes mainnet

**Severity**: Info  
**Likelihood**: Medium  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:24; omni-chain-contracts/contracts/GatewayCrossChain.sol:468; omni-chain-contracts/contracts/GatewayTransferNative.sol:25; omni-chain-contracts/contracts/GatewayTransferNative.sol:563

**Summary/Description**  
GatewayCrossChain and GatewayTransferNative classify Bitcoin routes only by decoded.dstChainId == 8332. Upstream ZetaChain metadata uses 8332 for Bitcoin mainnet, but Bitcoin testnet/signet/testnet4 use separate ids, while the scoped deployment config targets ZetaChain testnet. A Bitcoin route encoded with the canonical non-mainnet Bitcoin id will fall into the EVM/Solana withdrawAndCall branch instead of the plain Bitcoin withdraw branch.

**Root Cause**  
The route classifier is a single hardcoded Eddy DB value and is not derived from targetZRC20.CHAIN_ID, ZetaChain chain metadata, or a configurable set of Bitcoin-family chain ids.

**Pre_conditions**  
A route to a Bitcoin-family ZRC20 is encoded with a destination id other than 8332, such as ZetaChain Bitcoin testnet/signet/testnet4, or a future Bitcoin-family id added by the protocol, and the route otherwise expects plain Bitcoin withdrawal semantics.

**Impact**  
Plain Bitcoin withdrawals with empty contractAddress revert in GatewayZEVM.withdrawAndCall before the outbound withdrawal is emitted, causing route-local failure/refund handling instead of settlement. If a route tries to force the non-Bitcoin branch with a non-empty contractAddress, it requests contract-call semantics on a no-VM Bitcoin destination and relies on protocol failure handling. No standalone third-party fund extraction was proven.

**Mitigation**  
Replace the single constant comparison with an explicit configurable mapping or helper for Bitcoin-family chain ids, ideally bound to the targetZRC20 chain metadata rather than an off-chain route id. Reject inconsistent dstChainId/targetZRC20 combinations before fee, swap, approval, or gateway withdrawal side effects.

### [I-33] Zero treasury address can burn WZETA fees and native recoveries

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewayCrossChain.sol:108; omni-chain-contracts/contracts/GatewayCrossChain.sol:152; omni-chain-contracts/contracts/GatewayCrossChain.sol:161; omni-chain-contracts/contracts/GatewayCrossChain.sol:297; omni-chain-contracts/contracts/GatewayTransferNative.sol:109; omni-chain-contracts/contracts/GatewayTransferNative.sol:157; omni-chain-contracts/contracts/GatewayTransferNative.sol:166; omni-chain-contracts/contracts/GatewayTransferNative.sol:343; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/WZETA.sol:43

**Summary/Description**  
GatewayCrossChain and GatewayTransferNative accept address(0) as EddyTreasurySafe during initialization and through setEddyTreasurySafe. Fee and recovery paths then use that address directly. Standard ZRC20 fee transfers revert on a zero recipient, blocking affected settlements while misconfigured, but WZETA uses WETH-style transfer accounting and native recovery uses a raw value call, so WZETA platform fees or native residual withdrawals can be sent to address(0).

**Root Cause**  
The treasury recipient is stored without a nonzero validation in initialize or setEddyTreasurySafe, and fee/recovery transfers do not guard EddyTreasurySafe before sending value.

**Pre_conditions**  
The trusted owner/deployer initializes or updates EddyTreasurySafe to address(0), feePercent is positive for fee paths or superWithdraw is called for native recovery, and the transferred asset path is WZETA or native value. For standard ZRC20 fee paths, the transfer reverts instead of burning because ZRC20 rejects a zero recipient.

**Impact**  
WZETA-denominated platform fees can be credited to address(0), and native residual value recovered through superWithdraw can be sent to address(0). Standard ZRC20 routes are temporarily blocked until the treasury is reconfigured. The direct trigger is trusted-owner configuration, so this is recorded as Info rather than Medium/High.

**Mitigation**  
Reject address(0) in initialize and setEddyTreasurySafe. Also require EddyTreasurySafe != address(0) before fee and superWithdraw transfers, and consider adding a getter or monitoring hook for treasury configuration.

### [I-34] Unbounded gasLimit can disable or overcharge callback routes

**Severity**: Info  
**Likelihood**: Low  
**Impact**: Low  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:68; omni-chain-contracts/contracts/GatewaySend.sol:96; omni-chain-contracts/contracts/GatewaySend.sol:249; omni-chain-contracts/contracts/GatewaySend.sol:299; omni-chain-contracts/contracts/GatewayCrossChain.sol:108; omni-chain-contracts/contracts/GatewayCrossChain.sol:244; omni-chain-contracts/contracts/GatewayCrossChain.sol:385; omni-chain-contracts/contracts/GatewayTransferNative.sol:109; omni-chain-contracts/contracts/GatewayTransferNative.sol:148; omni-chain-contracts/contracts/GatewayTransferNative.sol:253; omni-chain-contracts/contracts/GatewayTransferNative.sol:478; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:184; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/GatewayZEVM.sol:187; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/ZRC20.sol:280; omni-chain-contracts/node_modules/@zetachain/protocol-contracts/contracts/zevm/ZRC20.sol:288

**Summary/Description**  
GatewaySend, GatewayCrossChain, and GatewayTransferNative store gasLimit directly from initialization or owner setters and reuse it for callback and outbound call gas settings. In the ZEVM wrappers, the same value is passed to withdrawGasFeeWithGasLimit before every EVM/Solana withdrawAndCall route. ZRC20 computes gasFee as gasPrice * gasLimit + PROTOCOL_FLAT_FEE, while GatewayZEVM only rejects gasLimit == 0 and does not enforce an upper bound. A zero setting disables withdrawAndCall, and an excessive nonzero setting can make gasFee exceed the routed output or consume most of it as gas.

**Root Cause**  
The scoped contracts do not validate gasLimit against the GatewayZEVM nonzero requirement, destination-chain practical gas bounds, or a maximum fee budget before using it for CallOptions and ZRC20 gas-fee quotes.

**Pre_conditions**  
The trusted owner/deployer initializes GatewayCrossChain, GatewayTransferNative, or GatewaySend with an invalid gasLimit, or updates GatewaySend/GatewayTransferNative through setGasLimit. A later route reaches GatewaySend revert handling or a GatewayCrossChain/GatewayTransferNative EVM/Solana withdrawAndCall path.

**Impact**  
For ZEVM outbound routes, gasLimit == 0 reverts at GatewayZEVM.withdrawAndCall. Excessive gasLimit increases gasFee linearly; same-gas-token paths revert when gasFee >= outputAmount or forward only outputAmount - gasFee, and different-gas-token paths must swap a larger amount into gas tokens before withdrawal. For GatewaySend source-chain routes, the unbounded value is forwarded as onRevertGasLimit and can make revert callback execution operationally mismatched. The direct trigger is trusted configuration, so this is recorded as Info rather than a primary Medium/High issue.

**Mitigation**  
Validate gasLimit during initialization and setters. Require a nonzero minimum for withdrawAndCall, cap it to a chain-specific maximum below practical block and protocol limits, and optionally enforce a per-route gas-fee budget such as gasFee < outputAmount with a configurable safety margin before approving or dispatching withdrawals.

### [I-35] Zero gateway configuration locks gateway callbacks

**Severity**: Info  
**Likelihood**: Low  
**Impact**: High  
**Location**: omni-chain-contracts/contracts/GatewaySend.sol:53; omni-chain-contracts/contracts/GatewaySend.sol:68; omni-chain-contracts/contracts/GatewaySend.sol:76; omni-chain-contracts/contracts/GatewaySend.sol:91; omni-chain-contracts/contracts/GatewayCrossChain.sol:89; omni-chain-contracts/contracts/GatewayCrossChain.sol:108; omni-chain-contracts/contracts/GatewayCrossChain.sol:119; omni-chain-contracts/contracts/GatewayCrossChain.sol:147; omni-chain-contracts/contracts/GatewayTransferNative.sol:90; omni-chain-contracts/contracts/GatewayTransferNative.sol:109; omni-chain-contracts/contracts/GatewayTransferNative.sol:120; omni-chain-contracts/contracts/GatewayTransferNative.sol:152

**Summary/Description**  
GatewaySend, GatewayCrossChain, and GatewayTransferNative store the configured gateway address during initialization and owner rotation without rejecting address(0). Their onlyGateway modifiers then require msg.sender to equal the stored gateway. If the stored gateway is zero, no real GatewayEVM or GatewayZEVM callback can satisfy the modifier, so onCall, onRevert, and onAbort are locked out and revert with Unauthorized.

**Root Cause**  
The gateway setters and initializers accept address(0), while callback authorization is a strict equality check against that mutable storage value.

**Pre_conditions**  
A trusted deployer initializes a proxy with a zero gateway, or the trusted owner calls setGateway(address(0)). Current checked deployment configs use nonzero gateway addresses, so this requires configuration error or later owner rotation.

**Impact**  
While the zero value is configured, inbound settlement callbacks and refund/abort callbacks from the actual gateway cannot execute. Live cross-chain routes can fail to settle or fail to enter the intended refund bookkeeping path until the configuration is corrected; any already-failed callback may require protocol/manual recovery. Because the trigger is solely trusted admin/deployment input under the README trust model, this is classified as Info rather than a contest-strength Medium/High issue.

**Mitigation**  
Reject address(0) in initialize and setGateway for all gateway-bound contracts. Prefer also checking _gateway.code.length > 0 and validating the expected GatewayEVM/GatewayZEVM address in deployment scripts before broadcasting.

