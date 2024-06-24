# CANTO QA REPORT

## Summary

| No | Title |
| --- | --- |
| [L-01] | If enough epochs pass without the period being increased, the accounting can be affected |
| [L-02] | GetQueryCmd and GetTxCmd are not implemented in Ethermint::feemarket |
| [L-03] | Lack of validation of Token Pair Denom in `convertERC20` |
| [L-04] | StandardDeposit can surpass maxStandardCoinPerPool  in CoinSwap module |
| [L-05] | Unimplemented TODO in code |
| [L-06] | The previous method of KVStore using store keys is still being implemented |

## Low Findings

### [L-01]  If enough epochs pass before the calculation for the period, the accounting of Epochs can be affected

- Lines of Code: 
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/inflation/keeper/hooks.go#L73

#### Impact
`BeforeEpochStarts` and `AfterEpochEnd` are hooks called after a successful epoch begins or ends in `inflation/keeper::hooks.go`.  When `AfterEpochEnd`, there is a calculation of the current epoch which checks if enough epochs have passed to start a new period. If enough epochs pass, the period is incremented.
```
@> if epochNumber-epochsPerPeriod*int64(period)-int64(skippedEpochs) > epochsPerPeriod {
		period++
		k.SetPeriod(ctx, period)
		period = k.GetPeriod(ctx)
		bondedRatio := k.BondedRatio(ctx)
		newProvision = types.CalculateEpochMintProvision(
			params,
			period,
			epochsPerPeriod,
			bondedRatio,
		)
		k.SetEpochMintProvision(ctx, newProvision)
	}
```
However, there's a slim possibility that multiple epochs pass without this calculation being carried out. This is due to a check earlier in the hook which doesn't throw and err but ends the hook call.
```
if epochIdentifier != epochstypes.DayEpochID {
			return
	}
```
This causes the hook to end without this calculation occuring, and the epoch still gets incremented. 
The calculation checks if the currentEpoch, period and skippedEpochs are > epochPerPeriod and the period is incremented by 1, to start a new period. 
What happens if enough epochs pass without this calculation being completed due to the early return, for example if epochPerPeriod is 10, and epochs goes from 10 --> 21 before this calculation is eventually carried out. The period now should be 3. However, due to the calculation only ever incrementing by 1, the period gets set to 2. 
This affects the entire accounting and functionality of the epochs .
Currently the epochPerPeriod is set to 365, therefore the edge case of this happening is relatively low. But if the epochPerPeriod is set to a lower value, for example 10, it is possible for this to occur. 

#### Recommendation
1. Before incrementing the period, take into account the current epoch and epochsPerPeriod, or perform a check before incrementing period. For example if epochPerPeriod is 10, and currentEpoch is 23, period should be 3. Therefore a check should be performed to confirm the correct value of period being setting it to period.

### [L-02] GetQueryCmd and GetTxCmd are not implemented in Ethermint::feemarket

- Lines of Code: 
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/x/feemarket/module.go

#### Impact
`GetQueryCmd` and `GetTxCmd` are necessary for providing Cli interfaces for interaction with the Blockchain within cosmos. It is required to be implemented within modules to ensure clients have an interface exposed for carrying out several queries and commands. However, it is not implemented within feemarket module on Ethermint. This would impact the functionality of this module.


#### Recommendation 
1. Implement `GetQueryCmd` and `GetTxCmd`within `feemarket/module.go`

### [L-03]  

- Lines of Code: 
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/keeper/msg_server.go#L85
#### Impact
Token pair Denom is not validated in `erc20::convertERC20`. This may result in the conversion of an incorrect token pair in the convertERC20 operation.

#### Recommendation
Validate the Denom of the provided token pair in `erc20::convertERC20` as it is validated within `convertCoin`

### [L-04] StandardDeposit can surpass maxStandardCoinPerPool  in CoinSwap module
- Lines of Code:  https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/coinswap/keeper/keeper.go#L193
#### Impact
The keeper imposes a limit on max amount of standard coins deposit in a pool. This ensures the size of pools remain within controllable limits. However, while not a strict invariant, this limit can be surpassed or bypassed. It is not clear what the impact is, but it may affect the accounting of the liquidity of the pool.
 There are 2 ways it can be bypassed
1. By direct deposits into the pool.
2. By coin swaps and ibc transfers

#### Recommendation
1. No recommendation but worth noting.

### [L-05] Unimplemented TODO in code 

#### Impact
There are unimplemented TODO in some parts of the code of inscope contracts. 
These instances include
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/coinswap/module.go#L75
```
/ GetQueryCmd returns no root query command for the coinswap module.
func (AppModuleBasic) GetQueryCmd() *cobra.Command {
@> 	// TODO: add cmd
	return cli.GetQueryCmd()
}
```
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/x/feemarket/keeper/keeper.go#L129
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/x/feemarket/keeper/keeper.go#L31
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/go.mod#L253

Why not of considerable impact, this indicates there may be lack of functionality which can impact the functioning of the app.

#### Recommendation
1. Complete implementation of functionality in TODO snippets 

| [L-06] | The previous method of KVStore using store keys is still being implemented |
- Lines of Code:  https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/coinswap/keeper/keeper.go#L193
#### Impact
One of the main changes from cosmos 0.4.x to cosmos 0.50.x is the updated way of using KVStore which involves use of kvStoreservice instead of store keys.

https://github.com/cosmos/cosmos-sdk/blob/v0.50.6/UPGRADING.md#module-wiring
This is also noted within the contest README

``
keeper
KVStore has been changed to use KVStoreService as opposed to the prev way of accessing KVStore via store key.(ref. refactor(x/staking)!: KVStoreService, return errors and use context.Context cosmos/cosmos-sdk#16324, ...)```

However, there are some places the previous method is still used in app.go
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/app/app.go#L361
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/app/app.go#L941


#### Recommendation
1. Implement KVStoreService completely to ensure complete compatibility with Cosmos 0.5.x