### [L-01] Defense in depth: fail-open check in x/erc20 GetSignersFromMsgConvertERC20V2

[Link to affected code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/app/app.go#L316)

The `canto-main/app/app.go` startup configuration, sets a custom getter for signers of the `MsgConvertERC20` message.

In this custom getter, that is the `GetSignersFromMsgConvertERC20V2` function in the `x/erc20` module, there is a 
checked cast from `protov2.Message` to  `erc20v1.MsgConvertERC20`.

```go
func GetSignersFromMsgConvertERC20V2(msg protov2.Message) ([][]byte, error) {
	msgv2, ok := msg.(*erc20v1.MsgConvertERC20)
	if !ok {
		return nil, nil
	}
	// ...
}
```

When this cast fails (`ok == false`), the function incorrectly returns "no signers, no error", effectively making the
`MsgConvertERC20` permissionless in that case.

While there isn't a clear path for entering this function without having a proper `erc20v1.MsgConvertERC20` in argument,
best practices require this check to fail closed instead of fail open:

```go
func GetSignersFromMsgConvertERC20V2(msg protov2.Message) ([][]byte, error) {
	msgv2, ok := msg.(*erc20v1.MsgConvertERC20)
	if !ok {
		return nil, fmt.Errorf("invalid input message")
	}
	// ...
}
```

### [L-02] Possible panic when extracting signers from MsgEthereumTx messages

[Link to affected code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/app/app.go#L315)

The `canto-main/app/app.go` startup configuration, sets a custom getter for signers of the `MsgEthereumTx` message.

This custom getter contains a nested call to the `GetMsgEthereumTxFromMsgV2` function:
```go
func GetSignersFromMsgEthereumTxV2(msg protov2.Message) ([][]byte, error) {
    msgv1, err := GetMsgEthereumTxFromMsgV2(msg)
    // ...
}	
```

which in turn uses `ModuleCdc.MustUnmarshal` calls that panic instead of returning the encountered errors:

```go
func GetMsgEthereumTxFromMsgV2(msg protov2.Message) (MsgEthereumTx, error) {
	msgv2, ok := msg.(*evmapi.MsgEthereumTx)
	if !ok {
		return MsgEthereumTx{}, fmt.Errorf("invalid x/evm/MsgEthereumTx msg v2: %v", msg)
	}

	var dataAny *codectypes.Any
	var err error
	switch msgv2.Data.TypeUrl {
	case "/ethermint.evm.v1.LegacyTx":
		legacyTx := LegacyTx{}
		ModuleCdc.MustUnmarshal(msgv2.Data.Value, &legacyTx)
		dataAny, err = PackTxData(&legacyTx)
		if err != nil {
			return MsgEthereumTx{}, err
		}
```

While some risk comes from the fact that the `MsgEthereumTx` messages come from users, it is also worth noting that 
there is no obvious way to exploit panics generated this way, because Canto is a `BaseApp` 
[and therefore benefits from its default panic handler](https://github.com/cosmos/cosmos-sdk/blob/ea16396f71c21ed30edb8ce88872ee3b35672fae/baseapp/baseapp.go#L243).

It is however recommended to instead make use of `ModuleCdc.Unmarshal` calls instead and bubbling up errors when this function returns some.

### [L-03] Usage of abandoned HexByte type

[Link to affected code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/x/evm/keeper/msg_server.go#L105)

With the release of v0.50.x, the Cosmos SDK [dropped its use of the bytes.HexBytes type](https://github.com/cosmos/cosmos-sdk/blob/release/v0.50.x/UPGRADING.md#migration-to-cometbft-part-2) 
in favor of the `[]byte` type. Consider adapting [ethermint-main/x/evm/keeper/msg_server.go](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/x/evm/keeper/msg_server.go#L105)
accordingly.


### [L-03] Incorrect return values returned by `MsgRemoveLiquidityResponse` response message

[Link to affected code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/coinswap/keeper/msg_server.go#L111-L118)

After the coinswap module's `msgServer` processed `MsgRemoveLiquidity` messages, it builds a `MsgRemoveLiquidityResponse`
response by looping through the `coins` that were withdrawn:

```go
	var coins = make([]*sdk.Coin, 0, withdrawCoins.Len())
	for _, coin := range withdrawCoins {
		coins = append(coins, &coin)
	}

	return &types.MsgRemoveLiquidityResponse{
		WithdrawCoins: coins,
	}, nil
```

As we can see in the second and third line of the above snippet, it's the `coin` reference (because of the `&` prefix)
that is added to the coins to be returned.

This leads to an incorrect output, because up to Go 1.21 (used by this project) included, `for` loops use the same
memory address for loop items, only updating its value at every cycle.

The result of this is that if coins `A, B, C` were removed, the Response message will incorrectly report removing coins
`C, C, C`.

Consider fixing the issue by forcing a new memory allocation for each iteration:
```go
	var coins = make([]*sdk.Coin, 0, withdrawCoins.Len())
	for _, coin := range withdrawCoins {
		coin := coin // @audit this fixes the issue
		coins = append(coins, &coin)
	}

	return &types.MsgRemoveLiquidityResponse{
		WithdrawCoins: coins,
	}, nil
```

Also, upgrading to Go 1.22 should fix the issue because starting from this version, the Go compiler allocates new memory
at each iteration.

### [L-04] Missing error check in v8 upgrade code

[Link to affected code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/app/upgrades/v8/upgrades.go#L68)

In the V8 upgrade call, the canto V8 custom code includes a call to `stakingKeeper.GetParams` and one to `stakingKeeper.SetParams`:

```go
		// canto v8 custom
		{
			params, err := stakingKeeper.GetParams(ctx)
			if err != nil {
				return vm, err
			}
			params.MinCommissionRate = MinCommissionRate
			stakingKeeper.SetParams(ctx, params)
		}
```

while both return an `error`, only the error returned in `GetParams` is checked.

Consider checking the return value of `SetParams` too.

### [L-05] Duplicate call to baseapp.MigrateParams in ethermint-main/app/upgrades.go

[Link to affected code](https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/app/upgrades.go#L107-L123)

In `ethermint-main/app/upgrades.go`, function `RegisterUpgradeHandlers`, there is an unnecessarily repeated call to
`baseapp.MigrateParams` for migrating the consensus parameters:

```go
			legacyBaseAppSubspace := paramsKeeper.Subspace(baseapp.Paramspace).WithKeyTable(paramstypes.ConsensusParamsKeyTable())
			if err := baseapp.MigrateParams(sdkCtx, legacyBaseAppSubspace, &consensusParamsKeeper.ParamsStore); err != nil { // <@
				return fromVM, err
			}

			// ibc v7.1
			// explicitly update the IBC 02-client params, adding the localhost client type
			params := clientKeeper.GetParams(sdkCtx)
			params.AllowedClients = append(params.AllowedClients, exported.Localhost)
			clientKeeper.SetParams(sdkCtx, params)

			// cosmos-sdk v047
			// Migrate Tendermint consensus parameters from x/params module to a dedicated x/consensus module.
			err := baseapp.MigrateParams(sdkCtx, baseAppLegacySS, consensusParamsKeeper.ParamsStore) // <@
			if err != nil {
				return fromVM, err
			}
```

While this does not pose an immediate risk because the `baseapp.MigrateParams` function is idempotent, consider removing
the second, unnecessary call.