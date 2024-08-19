---
sponsor: "Canto"
slug: "2024-05-canto"
date: "2024-08-19"
title: "Canto"
findings: "https://github.com/code-423n4/2024-05-canto-findings/issues"
contest: 385
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Canto smart contract system written in Go. The audit took place between May 30 — June 20, 2024.

## Wardens

12 Wardens contributed reports to Canto:

  1. [0x1771](https://code4rena.com/@0x1771)
  2. [0xSergeantPepper](https://code4rena.com/@0xSergeantPepper)
  3. [zhaojie](https://code4rena.com/@zhaojie)
  4. [Dup1337](https://code4rena.com/@Dup1337) ([sorrynotsorry](https://code4rena.com/@sorrynotsorry) and [deliriusz](https://code4rena.com/@deliriusz))
  5. [forgebyola](https://code4rena.com/@forgebyola)
  6. [ladboy233](https://code4rena.com/@ladboy233)
  7. [3docSec](https://code4rena.com/@3docSec)
  8. [ABAIKUNANBAEV](https://code4rena.com/@ABAIKUNANBAEV)
  9. [Ocean\_Sky](https://code4rena.com/@Ocean_Sky)
  10. [honeymewn](https://code4rena.com/@honeymewn)
  11. [carrotsmuggler](https://code4rena.com/@carrotsmuggler)

This audit was judged by [3docSec](https://code4rena.com/@3docsec). The judge also competed in the audit as a warden, but forfeited their winnings.

Final report assembled by [thebrittfactor](https://twitter.com/brittfactorC4).

# Summary

The C4 analysis yielded an aggregated total of 4 unique vulnerabilities. Of these vulnerabilities, 4 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 11 reports detailing issues with a risk rating of LOW severity or non-critical.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 Canto repository](https://github.com/code-423n4/2024-05-canto), and is composed of 28 smart contracts written in the Go programming language and includes 5388 lines of Go code.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# Medium Risk Findings (4)

## [[M-01] An attacker can DoS a coinswap pool](https://github.com/code-423n4/2024-05-canto-findings/issues/28)
*Submitted by [0x1771](https://github.com/code-423n4/2024-05-canto-findings/issues/28), also found by [0xSergeantPepper](https://github.com/code-423n4/2024-05-canto-findings/issues/20) and [zhaojie](https://github.com/code-423n4/2024-05-canto-findings/issues/6)*

The balance calculations are initiated by calling `k.GetPoolBalances(ctx, pool.EscrowAddress)`, which internally calls the `k.bk.GetAllBalances` function. This function iterates through all token balances in a loop. If the array of tokens is excessively large, the function may fail due to insufficient gas.

In essence, if an attacker introduces a large number of tokens, for instance through the `AddLiquidity` process, and subsequently transfers these tokens to a target pool, it can lead to an exploit. The attacker can strategically overload the array, causing significant gas consumption and ultimately causing the function to fail.

The process is as follows:

When the `AddLiquidity` or `RemoveLiquidity` functions are called within the coinswap module, the `k.GetPoolBalances` function retrieves the balance of all tokens in the pool. This function, `k.GetPoolBalances`, calls `k.bk.GetAllBalances`, which iterates through and aggregates all token balances before sorting them into an array.

Specifically, `k.bk.GetAllBalances` utilizes the following approach:

```go
func (k BaseViewKeeper) GetAllBalances(ctx context.Context, addr sdk.AccAddress) sdk.Coins {
    balances := sdk.NewCoins()
    k.IterateAccountBalances(ctx, addr, func(balance sdk.Coin) bool {
        balances = balances.Add(balance)
        return false
    })
    return balances.Sort()
}
```

Here, `sdk.NewCoins()` returns an array of type `Coins`.

When an attacker exploits the `AddLiquidity` function in the coinswap module, they can create a pool using `k.CreatePool(ctx, msg.MaxToken.Denom)` if the pool does not already exist. By generating a large number of tokens and sending them to the target pool, the attacker causes the array of balances returned by `GetPoolBalances` to become excessively large. This leads to high gas consumption and potential transaction failure due to insufficient gas, thus disrupting the functionality of the coinswap module.

### Recommended Mitigation Steps

Get only 1 token balance instead of all.

**[poorphd (Canto) confirmed and commented via duplicate Issue #20](https://github.com/code-423n4/2024-05-canto-findings/issues/20#issuecomment-2191190618):**
> **Reasoning:** As raised in the issue, if an attacker sends tokens of various denoms to the reserved pool address, `k.GetPoolBalances(ctx, pool.EscrowAddress)` could invoke `k.bk.GetAllBalances` that internally uses iteration, leading to a situation where the operation could fail if the array becomes very large.
>     
> However, pool creation is only allowed for whitelisted denoms, so it is impossible to obtain new tokens through `AddLiquidity` as raised in the issue. (See [here](https://github.com/b-harvest/Canto/blob/liquidstaking-module/x/coinswap/keeper/keeper.go#L113-L116) and [here](https://github.com/b-harvest/Canto/blob/liquidstaking-module/x/coinswap/types/params.go#L19-L36)).
>
> **Severity:** Mid → Low.
>
> In the worst-case scenario, the swap or `RemoveLiquidity` in coinswap might fail, but this only affects the auto swap during onboarding and does not impact the essential functions of the chain.
>
> **Patch:**
> -  We will patch this before v0.50 upgrade.
> - Change `k.GetPoolBalances(ctx, pool.EscrowAddress)` so that it does not use `k.bk.GetAllBalances` and only queries and returns the balance of standard coin, counter party coin, and pool coin.
> - Make appropriate changes for `GetPoolBalances` callers.
>
> ```go
> // GetPoolBalances return the liquidity pool by the specified anotherCoinDenom
> func (k Keeper) GetPoolBalances(ctx sdk.Context, pool types.Pool) (coins sdk.Coins, err error) {
> 	address, err := sdk.AccAddressFromBech32(pool.EscrowAddress)
> 	if err != nil {
>		return coins, err
>	}
>	acc := k.ak.GetAccount(ctx, address)
>	if acc == nil {
>		return nil, errorsmod.Wrap(types.ErrReservePoolNotExists, pool.EscrowAddress)
>	}
>
>	balances := sdk.NewCoins()
>	balances.Add(k.bk.GetBalance(ctx, acc.GetAddress(), pool.StandardDenom))
>	balances.Add(k.bk.GetBalance(ctx, acc.GetAddress(), pool.CounterpartyDenom))
>	balances.Add(k.bk.GetBalance(ctx, acc.GetAddress(), pool.LptDenom))
>	
>	return balances, nil
> }       

**[3docSec (judge) decreased severity to Medium](https://github.com/code-423n4/2024-05-canto-findings/issues/28#issuecomment-2191558246)**

**[3docSec (judge) commented via duplicate Issue #20](https://github.com/code-423n4/2024-05-canto-findings/issues/20#issuecomment-2191557238):**
> I find Medium to be appropriate for this group.
>
> Because Canto is connected to other Cosmos networks via IBC, an arbitrary number of token denominations can coexist (and be donated) to an existing pool to DoS its liquidity operations, without any privilege required for an attacker.

***

## [[M-02] `MsgSwapOrder` will never work for Canto nodes](https://github.com/code-423n4/2024-05-canto-findings/issues/27)
*Submitted by [0x1771](https://github.com/code-423n4/2024-05-canto-findings/issues/27), also found by [3docSec](https://github.com/code-423n4/2024-05-canto-findings/issues/4)*

An oversight in the [`MsgSwapOrder`](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/coinswap/v1/tx.proto#L96-L104) where the tag is directed to nested. The input message lacks the necessary `cosmos.msg.v1.signer` to indirectly identify the signer.

```go
message Input {
  string address = 1;
  cosmos.base.v1beta1.Coin coin = 2 [ (gogoproto.nullable) = false ];
}
```

### Recommended Mitigation Steps

Add `DefineCustomGetSigners` call in `app.go` for the coinswap Input message like you did for `MsgConvertERC20`.

https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/app/app.go#L316

```go
signingOptions.DefineCustomGetSigners(protov2.MessageName(&erc20v1.MsgConvertERC20{}), erc20types.GetSignersFromMsgConvertERC20V2)
```

**[poorphd (Canto) confirmed and commented](https://github.com/code-423n4/2024-05-canto-findings/issues/27#issuecomment-2193977635):**
 > **Reasoning:** The liquidity pools used for onboarding are directly called by the keeper method in the IBC middleware, so there is no problem with the onboarding function because the swap occurs. However, since only the swap from the ibc voucher to canto takes place, if there is a price discrepancy, a mechanism is needed for the arbitrager to return to the appropriate price through MsgSwapOrder.
 >
> **Severity:** `Mid` → `QA`.
> 
>In order to abuse this for price manipulation, it is necessary to repeatedly make auto-swaps through IBC transfers, but the auto-swap function only works when the balance of canto is less than 4, making it difficult to manipulate prices. This issue is valid, but since it is not an issue that opposes the real risk of assets, the severity should be adjusted from `Mid` to `QA`.
> 
> **Patch:** We will patch this before the v0.50 production release.

**[3docSec (judge) commented](https://github.com/code-423n4/2024-05-canto-findings/issues/27#issuecomment-2194106946):**
 > As the sponsor said, the effect of this vulnerability is that the pools' price drifts won't be balanced by a necessary arbitraging force which is required for the swap to meet the slippage/[`maxSwapAmount` check](https://github.com/code-423n4/2024-05-canto/blob/main/canto-main/x/coinswap/keeper/swap.go#L207); hence, impacting the availability of the Onboarding functionality. For this reason, I find Medium an appropriate severity for this finding.

***

## [[M-03] `Govshuttle` module does not register its transaction `MsgServer`](https://github.com/code-423n4/2024-05-canto-findings/issues/5)
*Submitted by [3docSec](https://github.com/code-423n4/2024-05-canto-findings/issues/5)*

The `x/govshuttle` module in `canto-main` defines and handles two messages that can be emitted by a governance proposal:

- [`MsgLendingMarketProposal`](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/govshuttle/keeper/msg_server.go#L22)
- [`MsgTreasuryProposal`](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/govshuttle/keeper/msg_server.go#L38)

However, because the module only registers the `QueryServer` (and not its `MsgServer`) in its `RegisterServices` function, causing no message to be routed to its message server:

```go
func (am AppModule) RegisterServices(cfg module.Configurator) {
	types.RegisterQueryServer(cfg.QueryServer(), am.keeper)
}
```

If we compare this with another module that can handle messages, for example, CSR, we see that this is the place for registering the `MsgServer` where transactional messages are routed to:

```go
func (am AppModule) RegisterServices(cfg module.Configurator) {
	types.RegisterMsgServer(cfg.MsgServer(), am.keeper)
	types.RegisterQueryServer(cfg.QueryServer(), am.keeper)
}
```

### Impact

Successful governance actions that include a `LendingMarketProposal` or `TreasuryProposal` will fail to execute because no handler is provided for them.

### Proof of Concept

To reproduce the issue it is sufficient to create and approve a proposal among the affected ones.

### Recommended Mitigation Steps

Consider adding a `RegisterMsgServer` call in the `x/govshuttle` `RegisterService` callback.

**[dudong2 (Canto) confirmed and commented](https://github.com/code-423n4/2024-05-canto-findings/issues/5#issuecomment-2199206207):**
 > **Reasoning:** As your description, the `MsgServer` isn't not registered about govshuttle module. Even if gov proposal that include a `LendingMarketProposal` or `TreasuryProposal` is passed, the msgs are not executed because there is no handler registered.
 >
> **Severity:** `Mid`.
>
> **Patch:** We will patch this before the v0.50 production release.

***

## [[M-04] Incorrect names provided in `RegisterConcrete` calls break `LegacyAmino` signing method](https://github.com/code-423n4/2024-05-canto-findings/issues/2)
*Submitted by [3docSec](https://github.com/code-423n4/2024-05-canto-findings/issues/2)*

One of the breaking changes introduced with the Cosmos SDK v0.50.x upgrade is [a change in the codec used for Amino JSON
(de)serialization](https://github.com/cosmos/cosmos-sdk/blob/release/v0.50.x/UPGRADING.md#protobuf). To ensure the
new codec behaves as the abandoned one did, the team added `amino.name` tags to the `message` types defined in the Canto
modules' ".proto" files.

There are however many instances where these tags are inconsistent with the `RegisterConcrete` calls made by the
in-scope modules' `func (AppModuleBasic) RegisterInterfaces` functions, all summarized below:

**Module `coinswap`:**

- [MsgAddLiquidity's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/coinswap/v1/tx.proto#L40) (`canto/MsgAddLiquidity`) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/coinswap/types/codec.go#L24) (`coinswap/coinswap/MsgSwapOrder`)
- [MsgRemoveLiquidity's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/coinswap/v1/tx.proto#L68) (`canto/MsgRemoveLiquidity`) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/coinswap/types/codec.go#L25) (`coinswap/coinswap/MsgAddLiquidity`)
- [MsgSwapOrder's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/coinswap/v1/tx.proto#L98) (`canto/MsgSwapOrder`) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/coinswap/types/codec.go#L26) (`coinswap/coinswap/MsgRemoveLiquidity`)
- [MsgUpdateParams's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/coinswap/v1/tx.proto#L116) (`canto/MsgUpdateParams`) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/coinswap/types/codec.go#L27) (`coinswap/coinswap/MsgUpdateParams`)
- [Param's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/coinswap/v1/coinswap.proto#L36) (not set) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/coinswap/types/codec.go#L28) (`coinswap/coinswap/Params`)

**Module `csr`:**

- [MsgUpdateParams's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/csr/v1/tx.proto#L20) (`"canto/MsgUpdateParams"`) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/csr/types/codec.go#L33) (`canto/x/csr/MsgUpdateParams`)
- [Param's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/csr/v1/params.proto#L10) (not set) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/csr/types/codec.go#L34) (`canto/x/csr/Params`)

**Module `erc20`:**

- [MsgRegisterCoin's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/erc20/v1/tx.proto#L112) (`canto/MsgRegisterCoin`) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/types/codec.go#L65) (`"canto/RegisterCoinProposal"`)
- [MsgRegisterERC20's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/erc20/v1/tx.proto#L133) (`canto/MsgRegisterERC20`) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/types/codec.go#L65) (`"canto/RegisterERC20Proposal"`)
- [Params's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/erc20/v1/genesis.proto#L18) (not set) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/types/codec.go#L65) (`"canto/Params"`)

**Module `govshuttle`:**

Module govshuttle has no discrepancy thanks to the fact that the `RegisterConcrete` call was not made with the `Msg` types

**Module `inflation`:**

- [MsgUpdateParams's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/inflation/v1/tx.proto#L26) (`canto/MsgUpdateParams`) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/inflation/types/codec.go#L37) (`canto/x/inflation/MsgUpdateParams`)
- [Params's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/inflation/v1/genesis.proto#L25) (not set) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/inflation/types/codec.go#L37) (`canto/x/inflation/Params`)

**Module `onboarding`:**

- [MsgUpdateParams's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/onboarding/v1/tx.proto#L26) (`canto/MsgUpdateParams`) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/onboarding/types/codec.go#L32) (`canto/x/onboarding/MsgUpdateParams`)
- [Params's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/proto/canto/onboarding/v1/genesis.proto#L16) (not set) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/onboarding/types/codec.go#L32) (`canto/x/onboarding/Params`)

**Module `evm`:**

- [MsgUpdateParams's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/proto/ethermint/evm/v1/tx.proto#L173) (not set) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/x/evm/types/codec.go#L105) (`ethermint/MsgUpdateParams`)

**Module `feemarket`:**

- [MsgUpdateParams's tag in protobuf](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/proto/ethermint/feemarket/v1/tx.proto#L21) (not set) does not match [the name registered in code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/x/feemarket/types/codec.go#L41) (`ethermint/feemarket/MsgUpdateParams`)

### Impact

All the messages with inconsistent settings listed above, when signed with the `LegacyAmino` method on a v7 or compatible client, will not be recognized (and consequently rejected) by the Canto app v8 message routing.

### Proof of Concept

This finding can be proved by adapting [this generative test](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-beta.0/tests/integration/tx/aminojson/aminojson_test.go#L94)
(that is [the verification tool mentioned in the Cosmos SDK v0.50.x upgrade guide](https://github.com/cosmos/cosmos-sdk/blob/release/v0.50.x/UPGRADING.md#protobuf))
to check the messages defined in the Canto modules instead of those standard to the Cosmos SDK it was originally
written for.

Adapting this test requires a bit of workarounds because the test itself uses internal packages of the Canto SDK that
can't be imported directly, so to make a runnable PoC, I've created a Bash script that builds up the test environment,
and runs the failing test (note that Git and Go installations are a prerequisite for this script).

This Bash script can be found [here](https://gist.github.com/3docSec/2ce73d7321dd957a0dc8ee5c379cbc45) and its output (limited to the first of 14 failing tests) is:

    Expected :{"type":"coinswap/coinswap/MsgAddLiquidity","value":{"max_token":{"amount":"0"},"exact_standard_amt":"0","min_liquidity":"0"}}
    Actual   :{"type":"canto/MsgAddLiquidity","value":{"max_token":{"amount":"0"},"exact_standard_amt":"0","min_liquidity":"0"}}

### Recommended Mitigation Steps

Consider fixing the `RegisterConcrete` calls to match the `amino.name` flags of all the messages enumerated above, which fail the test provided as PoC.

### Assessed type

en/de-code

**[dudong2 (Canto) confirmed and commented](https://github.com/code-423n4/2024-05-canto-findings/issues/2#issuecomment-2199278565):**
 > **Reasoning:** Through your test code, we checked that several LegacyAmino has wrong type name. And it can cause AminoJson signing failing.
 >
> **Severity:** `Mid`.
>
> **Patch:** We will patch this before the v0.50 production release.

***

# Low Risk and Non-Critical Issues

For this audit, 11 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2024-05-canto-findings/issues/33) by **Dup1337** received the top score from the judge.

*The following wardens also submitted reports: [forgebyola](https://github.com/code-423n4/2024-05-canto-findings/issues/34), [ladboy233](https://github.com/code-423n4/2024-05-canto-findings/issues/35), [ABAIKUNANBAEV](https://github.com/code-423n4/2024-05-canto-findings/issues/36), [0xSergeantPepper](https://github.com/code-423n4/2024-05-canto-findings/issues/26), [Ocean\_Sky](https://github.com/code-423n4/2024-05-canto-findings/issues/24), [0x1771](https://github.com/code-423n4/2024-05-canto-findings/issues/19), [honeymewn](https://github.com/code-423n4/2024-05-canto-findings/issues/15), [carrotsmuggler](https://github.com/code-423n4/2024-05-canto-findings/issues/13), [3docSec](https://github.com/code-423n4/2024-05-canto-findings/issues/12), and [zhaojie](https://github.com/code-423n4/2024-05-canto-findings/issues/10).*

## [L-01] Usage of deprecated `WrapSDKContext()`

`WrapSDKContext()` has depreciation notice on it:

```go
// Deprecated: there is no need to wrap anymore as the Cosmos SDK context implements context.Context.
func WrapSDKContext(ctx Context) context.Context {
	return ctx
}
```

Canto still uses this function in multiple places.

<details>
<summary>List of occurrences</summary>

```
x/inflation/keeper/grpc_query_test.go|49 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/inflation/keeper/grpc_query_test.go|108 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/inflation/keeper/grpc_query_test.go|158 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/inflation/keeper/grpc_query_test.go|174 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/inflation/keeper/grpc_query_test.go|191 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/inflation/keeper/grpc_query_test.go|206 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/coinswap/keeper/grpc_query_test.go|10 col 40| resp, err := s.queryClient.Params(sdk.WrapSDKContext(s.ctx), &types.QueryParamsRequest{})
x/coinswap/keeper/grpc_query_test.go|20 col 47| resp, err := s.queryClient.LiquidityPool(sdk.WrapSDKContext(s.ctx), &types.QueryLiquidityPoolRequest{LptDenom: pool.LptDenom})
x/coinswap/keeper/grpc_query_test.go|39 col 48| resp, err := s.queryClient.LiquidityPools(sdk.WrapSDKContext(s.ctx), &types.QueryLiquidityPoolsRequest{})
x/coinswap/keeper/grpc_query_test.go|44 col 47| resp, err = s.queryClient.LiquidityPools(sdk.WrapSDKContext(s.ctx), &types.QueryLiquidityPoolsRequest{})
x/onboarding/keeper/grpc_query_test.go|10 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/onboarding/keeper/ibc_callbacks.go|137 col 44| if _, err = k.erc20Keeper.ConvertCoin(sdk.WrapSDKContext(ctx), convertMsg); err != nil {
x/epochs/keeper/grpc_query_test.go|185 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/evm.go|187 col 46| gasRes, err := k.evmKeeper.EstimateGas(sdk.WrapSDKContext(ctx), &evmtypes.EthCallRequest{
x/erc20/keeper/keeper_test.go|211 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/keeper_test.go|266 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/keeper_test.go|309 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/keeper_test.go|405 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/msg_server_test.go|343 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/msg_server_test.go|521 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/msg_server_test.go|531 col 14| ctx = sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/msg_server_test.go|1079 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/msg_server_test.go|1138 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/msg_server_test.go|1143 col 14| ctx = sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/msg_server_test.go|1293 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/msg_server_test.go|1471 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/msg_server_test.go|1481 col 14| ctx = sdk.WrapSDKContext(suite.ctx)
x/govshuttle/keeper/msg_server_test.go|15 col 44| //return keeper.NewMsgServerImpl(*k), sdk.WrapSDKContext(ctx)
x/erc20/keeper/evm_hooks_test.go|213 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/govshuttle/keeper/keeper_test.go|63 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/govshuttle/keeper/grpc_query_params_test.go|12 col 17| // wctx := sdk.WrapSDKContext(ctx)
x/erc20/keeper/grpc_query_test.go|69 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/grpc_query_test.go|149 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/erc20/keeper/grpc_query_test.go|164 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/csr/keeper/grpc_query_test.go|13 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/csr/keeper/grpc_query_test.go|117 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/csr/keeper/grpc_query_test.go|135 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/csr/keeper/grpc_query_test.go|193 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/csr/keeper/grpc_query_test.go|210 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/csr/keeper/grpc_query_test.go|287 col 15| ctx := sdk.WrapSDKContext(suite.ctx)
x/csr/keeper/grpc_query_test.go|303 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
x/csr/keeper/grpc_query_test.go|315 col 13| ctx := sdk.WrapSDKContext(suite.ctx)
```

</details>

## [L-02] Unnecessary event emission

According to the [docs](https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md#all-2):

> `EventTypeMessage` events, with `sdk.AttributeKeyModule` and `sdk.AttributeKeySender` are now emitted directly at message execution (in baseapp). 

This means that the following boilerplate should be removed from all your custom modules:

```
ctx.EventManager().EmitEvent(
    sdk.NewEvent(
        sdk.EventTypeMessage,
        sdk.NewAttribute(sdk.AttributeKeyModule, types.AttributeValueCategory),
        sdk.NewAttribute(sdk.AttributeKeySender, `signer/sender`),
    ),
)
```

However, Canto still uses it:

```go
canto-main/x/coinswap/keeper/msg_server.go
 59    ctx.EventManager().EmitEvent(
 60       sdk.NewEvent(
 61          sdk.EventTypeMessage,
 62          sdk.NewAttribute(sdk.AttributeKeyModule, types.AttributeValueCategory),
 63          sdk.NewAttribute(sdk.AttributeKeySender, msg.Sender),
 64       ),
```

## [L-03] Double order set of modules

According to the [docs](https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md#set-preblocker), upgrade module should be initialized only in `PreBlocker`; however, Canto initialized it both in pre blocker and begin blocker:

```go
ethermint-main/app/app.go

 639    // NOTE: upgrade module is required to be prioritized
 640    app.ModuleManager.SetOrderPreBlockers(
 641 @>    upgradetypes.ModuleName,
 642    )
 643    // During begin block slashing happens after distr.BeginBlocker so that
 644    // there is nothing left over in the validator fee pool, so as to keep the
 645    // CanWithdrawInvariant invariant.
 646    // NOTE: upgrade module must go first to handle software upgrades.
 647    // NOTE: staking module is required if HistoricalEntries param > 0
 648    // NOTE: capability module's beginblocker must come before any modules using capabilities (e.g. IBC)
 649    app.ModuleManager.SetOrderBeginBlockers(
//[...]
 668 @>    upgradetypes.ModuleName,
```

The same in `canto-main/app/app.go`.

`ModuleManager.SetOrder...` checks if it implements correct function; however, it leads to confusion and the behavious may change in the future.

```go
func (m *Manager) BeginBlock(ctx sdk.Context) (sdk.BeginBlock, error) {
    ctx = ctx.WithEventManager(sdk.NewEventManager())
    for _, moduleName := range m.OrderBeginBlockers {
        if module, ok := m.Modules[moduleName].(appmodule.HasBeginBlocker); ok {
            if err := module.BeginBlock(ctx); err != nil {
                return sdk.BeginBlock{}, err
            }
        }
    }

    return sdk.BeginBlock{
        Events: ctx.EventManager().ABCIEvents(),
    }, nil
}
```

## [L-04] Using deprecated `RandomizedParams`

As per the docs:

> Remove `RandomizedParams` from `AppModuleSimulation` interface. Previously, it used to generate random parameter changes during simulations; however, it does so through `ParamChangeProposal` which is now legacy. Since all modules were migrated, we can now safely remove this from `AppModuleSimulation` interface.

And the existing code has this interface:

```go
Contract: module_simulation.go

47: // RandomizedParams creates randomized  param changes for the simulator
48: func (am AppModule) RandomizedParams(_ *rand.Rand) []simtypes.LegacyParamChange {
49: 
50:     return []simtypes.LegacyParamChange{}
51: }
```

**List of occurrences:**

```
canto-main/x/govshuttle/module_simulation.go|47 col 4| // RandomizedParams creates randomized  param changes for the simulator
canto-main/x/govshuttle/module_simulation.go|48 col 21| func (am AppModule) RandomizedParams(_ *rand.Rand) []simtypes.LegacyParamChange {
canto-main/x/erc20/module.go|151 col 21| func (am AppModule) RandomizedParams(r *rand.Rand) []simtypes.LegacyParamChange {
canto-main/x/coinswap/module.go|155 col 4| // RandomizedParams creates randomized coinswap param changes for the simulator.
canto-main/x/coinswap/module.go|156 col 18| func (AppModule) RandomizedParams(r *rand.Rand) []simtypes.LegacyParamChange {
canto-main/x/inflation/module.go|175 col 4| // RandomizedParams creates randomized inflation param changes for the simulator.
canto-main/x/inflation/module.go|176 col 21| func (am AppModule) RandomizedParams(r *rand.Rand) []simtypes.LegacyParamChange {
canto-main/x/onboarding/module.go|143 col 18| func (AppModule) RandomizedParams(_ *rand.Rand) []simtypes.LegacyParamChange {
canto-main/x/epochs/module.go|168 col 4| // RandomizedParams creates randomizedepochs param changes for the simulator.
canto-main/x/epochs/module.go|169 col 18| func (AppModule) RandomizedParams(r *rand.Rand) []simtypes.LegacyParamChange {
```

## [L-05] Importing deprecated simapp package

As per the [docs](https://github.com/cosmos/cosmos-sdk/blob/release/v0.47.x/UPGRADING.md#simapp);

> The simapp package should not be imported in your own app. Instead, you should import the `runtime.AppI` interface, that defines an App, and use the simtestutil package for application testing.

But it´s imported and used in `canto-main/app`:

```javascript
Contract: app.go

37:     "cosmossdk.io/simapp"
```

```go
Contract: app.go

338:     // create and set dummy vote extension handler
339:     voteExtOp := func(bApp *baseapp.BaseApp) {
340:         voteExtHandler := simapp.NewVoteExtensionHandler()
```

```go
Contract: app.go

1067: // InitChainer updates at chain initialization
1068: func (app *Canto) InitChainer(ctx sdk.Context, req *abci.RequestInitChain) (*abci.ResponseInitChain, error) {
1069:     var genesisState simapp.GenesisState
```

## [L-06] Incorrect import of protobuf

As per the [docs](https://github.com/cosmos/cosmos-sdk/blob/release/v0.47.x/UPGRADING.md#protobuf);

> The SDK has migrated from gogo/protobuf (which is currently unmaintained), to our own maintained fork, cosmos/gogoproto.
> 
> This means you should replace all imports of `github.com/gogo/protobuf` to `github.com/cosmos/gogoproto`. This allows you to remove the replace directive replace `github.com/gogo/protobuf => github.com/regen-network/protobuf v1.3.3-alpha.regen.1` from your `go.mod file`.

While `go.mod` files for `canto-main` and `ethermint-main` imports protobuf as:

```go
Contract: go.mod

32:     github.com/gogo/protobuf v1.3.2
```

The `proto-tools-installer.sh` script installs the latest one which might cause dependency issues:

```go
Contract: proto-tools-installer.sh

86: f_install_protoc_gen_gocosmos() {
87:     f_print_installing_with_padding protoc-gen-gocosmos
88:     
89:     if ! grep "github.com/gogo/protobuf => github.com/regen-network/protobuf" go.mod &>/dev/null ; then
90:         echo -e "\tPlease run this command from somewhere inside the canto folder."
91:         return 1
92:     fi
93:     
94:     go get github.com/regen-network/cosmos-proto/protoc-gen-gocosmos 2>/dev/null
95:     f_print_done
96: }
```

Especially given that [this](https://github.com/regen-network/cosmos-proto) was deprecated in favor of `cosmos/gogoproto` and  v1.5.0 of [this](https://github.com/cosmos/gogoproto/releases/tag/v1.5.0) is the current one.

## [L-07] Usage of deprecated params module

As per the [docs](https://github.com/cosmos/cosmos-sdk/blob/release/v0.47.x/UPGRADING.md#xparams):

> The `params` module was deprecated since v0.46. The Cosmos SDK has migrated away from `x/params` for its own modules. Cosmos SDK modules now store their parameters directly in its respective modules. The `params` module will be removed in v0.48, as mentioned in v0.46 release. It is strongly encouraged to migrate away from `x/params` before v0.48.
> 
> When performing a chain migration, the `params` table must be initialized manually. This was done in the modules keepers in previous versions. Have a look at `simapp.RegisterUpgradeHandlers()` for an example.

The codebase still uses the module. In `canto-main/app/app.go`:

```javascript
Contract: app.go

 99:     "github.com/cosmos/cosmos-sdk/x/params"
100:     paramsclient "github.com/cosmos/cosmos-sdk/x/params/client"
101:     paramskeeper "github.com/cosmos/cosmos-sdk/x/params/keeper"
102:     paramstypes "github.com/cosmos/cosmos-sdk/x/params/types"
103:     paramproposal "github.com/cosmos/cosmos-sdk/x/params/types/proposal"
```

In `canto-main/app/upgrades/v8/upgrades.go`:

```javascript
Contract: upgrades.go

13:     paramstypes "github.com/cosmos/cosmos-sdk/x/params/types"
```

In `canto-main/ibc/testing/simapp/app.go`:

```javascript
Contract: app.go

92:     "github.com/cosmos/cosmos-sdk/x/params"
93:     paramsclient "github.com/cosmos/cosmos-sdk/x/params/client"
94:     paramskeeper "github.com/cosmos/cosmos-sdk/x/params/keeper"
95:     paramstypes "github.com/cosmos/cosmos-sdk/x/params/types"
96:     paramproposal "github.com/cosmos/cosmos-sdk/x/params/types/proposal"
```

Also, many more for canto-main. In `ethermint-main/app/app.go`:

```javascript
Contract: app.go

105:     "github.com/cosmos/cosmos-sdk/x/params"
106:     paramsclient "github.com/cosmos/cosmos-sdk/x/params/client"
107:     paramskeeper "github.com/cosmos/cosmos-sdk/x/params/keeper"
108:     paramstypes "github.com/cosmos/cosmos-sdk/x/params/types"
109:     paramproposal "github.com/cosmos/cosmos-sdk/x/params/types/proposal"
```

We're not sure if this is as designed, but [this](https://github.com/cosmos/cosmos-sdk/blob/v0.46.1/UPGRADING.md#xparams) mentions a pull request that performed migration differently [here](https://github.com/cosmos/cosmos-sdk/pull/12363/files), specifically via `x/mint/keeper/migrator.go`:

```go
// Migrate1to2 migrates the x/mint module state from the consensus version 1 to
// version 2. Specifically, it takes the parameters that are currently stored
// and managed by the x/params modules and stores them directly into the x/mint
// module state.
func (m Migrator) Migrate1to2(ctx sdk.Context) error {
    return v2.Migrate(ctx, ctx.KVStore(m.keeper.storeKey), m.legacySubspace, m.keeper.cdc)
}
```

## [L-08] Using deprecated `AppModuleBasic`

As per the [update](https://github.com/cosmos/cosmos-sdk/pull/19512):

> The notion of basic manager does not exist anymore (and all related helpers).
> 
>  - The module manager now can do everything that the basic manager was doing.
>  - `AppModuleBasic` has been deprecated for extension interfaces.
>  -  Modules can now implement `appmodule.HasRegisterInterfaces`, `module.HasGRPCGateway` and `module.HasAminoCodec` when relevant.
>  - SDK modules now directly implement those extension interfaces on `AppModule` instead of `AppModuleBasic`.

However, all the apps (`canto-main/app`, `canto-main/ibc/testing/simapp`, `ethermint-main/app`) use the removed manager, in which the initializations of the module managers will fail;

```go
Contract: app.go

779:     app.BasicModuleManager = module.NewBasicManagerFromManager(
780:         app.ModuleManager,
781:         map[string]module.AppModuleBasic{
782:             genutiltypes.ModuleName: genutil.NewAppModuleBasic(genutiltypes.DefaultMessageValidator),
783:             govtypes.ModuleName: gov.NewAppModuleBasic(
784:                 []govclient.ProposalHandler{
785:                     paramsclient.ProposalHandler,
786:                 },
787:             ),
788:         })
```

```go
Contract: app.go

607:     app.BasicModuleManager = module.NewBasicManagerFromManager(
608:         app.ModuleManager,
609:         map[string]module.AppModuleBasic{
610:             genutiltypes.ModuleName: genutil.NewAppModuleBasic(genutiltypes.DefaultMessageValidator),
611:             govtypes.ModuleName: gov.NewAppModuleBasic(
612:                 []govclient.ProposalHandler{
613:                     paramsclient.ProposalHandler,
614:                 },
615:             ),
616:         })
```

```go
Contract: app.go

626:     app.BasicModuleManager = module.NewBasicManagerFromManager(
627:         app.ModuleManager,
628:         map[string]module.AppModuleBasic{
629:             genutiltypes.ModuleName: genutil.NewAppModuleBasic(genutiltypes.DefaultMessageValidator),
630:             govtypes.ModuleName: gov.NewAppModuleBasic(
631:                 []govclient.ProposalHandler{
632:                     paramsclient.ProposalHandler,
633:                 },
634:             ),
635:         })
```

## [NC-01] Usage of legacy amino is no longer needed

According to the [docs](https://docs.cosmos.network/v0.50/build/building-modules/protobuf-annotations#amino):

> The amino codec was removed in v0.50+, this means there is not a need register `legacyAminoCodec`. To replace the amino codec, Amino protobuf annotations are used to provide information to the amino codec on how to encode and decode protobuf messages.

However, it's still used in Canto.

Exemplary proto file [here](https://github.com/code-423n4/2024-05-canto/blob/main/canto-main/proto/canto/coinswap/v1/coinswap.proto).

And auto generated `.pg.go` definition file from the proto [here](https://github.com/code-423n4/2024-05-canto/blob/main/canto-main/x/coinswap/types/coinswap.pb.go).

## [NC-02] Old way of passing `StoreKey`

Canto's custom module - `group` still uses old way of passing store to keeper:

```go
x/group/keeper/keeper.go

func NewKeeper(storeKey storetypes.StoreKey, cdc codec.Codec, router baseapp.MessageRouter, accKeeper group.AccountKeeper, config group.Config) Keeper {
    k := Keeper{
        key:       storeKey,
        router:    router,
        accKeeper: accKeeper,
        cdc:       cdc,
    }
```

While this is not a problem, it is against the design decisions that Cosmos SDK took, according to the [docs](https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md#module-wiring). 

> The following modules `NewKeeper` function now take a `KVStoreService` instead of a `StoreKey`.

## [NC-03] Missing checks for implementing all interfaces

According to the [docs](https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md#all-1):

> It is possible to ensure that a module implements the correct interfaces by using compiler assertions in your x/{moduleName}/module.go:
>
>```
> var (
>     _ module.AppModuleBasic      = (*AppModule)(nil)
>     _ module.AppModuleSimulation = (*AppModule)(nil)
>     _ module.HasGenesis          = (*AppModule)(nil)
> 
>     _ appmodule.AppModule        = (*AppModule)(nil)
>     _ appmodule.HasBeginBlocker  = (*AppModule)(nil)
>     _ appmodule.HasEndBlocker    = (*AppModule)(nil)
>     ...
> )
> ```

While the protocol implements the interfaces implicitly in modules like `epochs` or `feemarket`, it's not explicit and may pose problems in the future, in case that Cosmos SDK interfaces change again, then the code is not accommodated for it.

## [NC-04] Not upgrading to `[]byte` from `cometbft` types

According to the [docs](https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md#migration-to-cometbft-part-2):

> The usage of `github.com/cometbft/cometbft/libs/bytes.HexByte` has been replaced by `[]byte`.

While this is Cosmos related, it would be good for the protocol to align with those changes. Canto is still using it:

```go
canto-main/x/onboarding/types/interfaces.go

  3 import (
//[...]
 12    tmbytes "github.com/cometbft/cometbft/libs/bytes"
//[...]
 70 type TransferKeeper interface {
 71    GetDenomTrace(ctx sdk.Context, denomTraceHash tmbytes.HexBytes) (transfertypes.DenomTrace, bool)
 72 }
```
The same situation exists for `ethermint-main/x/evm/keeper/msg_server.go`.

***

# Disclosures

C4 is an open organization governed by participants in the community.

C4 audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and Go developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.