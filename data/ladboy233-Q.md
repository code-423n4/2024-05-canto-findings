# Vulnerable package version are used.

the code goes 

go-ethereum v1.10.19 to v1.10.26

> Cosmos SDK v0.45.9 to v0.50.6
> CometBFT (Tendermint) v0.34.25 to v0.38.6
> ibc-go v3.2.0 to v8.2.1
> go-ethereum v1.10.19 to v1.10.26

then the code is vulnerable to these issues

https://www.cvedetails.com/cve/CVE-2023-40591/

> go-ethereum (geth) is a golang execution layer implementation of the Ethereum protocol. A vulnerable node, can be made to consume unbounded amounts of memory when handling specially crafted p2p messages sent from an attacker node. The fix is included in geth version `1.12.1-stable`, i.e, `1.12.2-unstable` and onwards. Users are advised to upgrade. There are no known workarounds for this vulnerability.

https://www.cvedetails.com/cve/CVE-2023-42319/

> Geth (aka go-ethereum) through 1.13.4, when --http --graphql is used, allows remote attackers to cause a denial of service (memory consumption and daemon hang) via a crafted GraphQL query. NOTE: the vendor's position is that the "graphql endpoint [is not] designed to withstand attacks by hostile clients, nor handle huge amounts of clients/traffic.

Also, the code version use v0.50.6, but the newly released v0.50.7 has a few bug fix from v0.50.6 that needs attention.

https://github.com/cosmos/cosmos-sdk/blob/v0.50.7/CHANGELOG.md

* (baseapp) [#20346](https://github.com/cosmos/cosmos-sdk/pull/20346) Correctly assign `execModeSimulate` to context for `simulateTx`.
* (baseapp) [#20144](https://github.com/cosmos/cosmos-sdk/pull/20144) Remove txs from mempool when AnteHandler fails in recheck.
* (baseapp) [#20107](https://github.com/cosmos/cosmos-sdk/pull/20107) Avoid header height overwrite block height.
* (cli) [#20020](https://github.com/cosmos/cosmos-sdk/pull/20020) Make bootstrap-state command support both new and legacy genesis format.
* (testutil/sims) [#20151](https://github.com/cosmos/cosmos-sdk/pull/20151) Set all signatures and don't overwrite the previous one in `GenSignedMockTx`.

it is recommended to upgraded to the newly version.

# use string() in state-machine code can cause panic

https://github.com/cosmos/cosmos-sdk/blob/v0.50.6/UPGRADING.md

> The gogoproto.goproto_stringer = false annotation has been removed from most proto files. This means that the String() method is being generated for types that previously had this annotation. The generated String() method uses proto.CompactTextString for stringifying structs. Verify the usage of the modified String() methods and double-check that they are not used in state-machine code.

More detailed discussion is here

https://github.com/cosmos/cosmos-sdk/pull/13850#issuecomment-1328889651

However, the string() is still used in state-machine code such as :

```solidity
// BlockedAddrs returns all the app's module account addresses that are not
// allowed to receive external tokens.
func (app *Canto) BlockedAddrs() map[string]bool {
	blockedAddrs := make(map[string]bool)
	for acc := range maccPerms {
		blockedAddrs[authtypes.NewModuleAddress(acc).String()] = true
	}

	return blockedAddrs
}
```

# Governance voting change is not addressed.

https://github.com/cosmos/cosmos-sdk/blob/v0.50.6/UPGRADING.md#xgov

according to the change log:

> The gov v1 module now supports expedited governance proposals. When a proposal is expedited, the voting period will be shortened to ExpeditedVotingPeriod parameter. An expedited proposal must have an higher voting threshold than a classic proposal, that threshold is defined with the ExpeditedThreshold parameter.

However, the parameter ExpeditedVotingPeriod is not set, result in non-functional  expedited governance proposals.

and 

> The gov module now supports cancelling governance proposals. When a proposal is canceled, all the deposits of the proposal are either burnt or sent to ProposalCancelDest address. The deposits burn rate will be determined by a new parameter called ProposalCancelRatio parameter.

and

deposits * proposal_cancel_ratio will be burned or sent to `ProposalCancelDest` address , if `ProposalCancelDest` is empty then deposits will be burned.

and deposits * (1 - proposal_cancel_ratio) will be sent to depositors.

because the setting ProposalCancelRatio and ProposalCancelDest  are not set, 

all deposit will be burnt when a governance proposal is canneled.
