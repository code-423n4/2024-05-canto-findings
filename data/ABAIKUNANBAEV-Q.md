## Finding Summary 

| ID | Description | Severity |
| - | - | :-: |
| [L-01](#l-01-avoid-directly-minting-follow-nfts-to-the-profile-owner-in-processblock) | `Protobuf` is deprecated and should be removed | Low | 
| [L-02](#l-01-avoid-directly-minting-follow-nfts-to-the-profile-owner-in-processblock) | Deviation from the spec when wiring their chain | Low |
| [L-03](#l-01-avoid-directly-minting-follow-nfts-to-the-profile-owner-in-processblock) | The module doesn't ensure that interface are correct | Low |



## [L-01] `Protobuf` is deprecated and should be removed

According to the docs:

https://github.com/cosmos/cosmos-sdk/blob/v0.47.11/UPGRADING.md
```
The SDK has migrated from gogo/protobuf (which is currently unmaintained), to our own maintained fork, cosmos/gogoproto.

This means you should replace all imports of github.com/gogo/protobuf to github.com/cosmos/gogoproto

This allows you to remove the replace directive replace github.com/gogo/protobuf => github.com/regen-network/protobuf v1.3.3-alpha.regen.1 from your go.mod file.
```

However, inside of the `go.mod` file, one of the directives is still not replaced:


https://github.com/code-423n4/2024-05-canto/blob/main/canto-main/go.mod#L32
```
github.com/gogo/protobuf v1.3.2
```


### Recommendation

Apply all the Protobuf changes to the current implementation.


## [L-02] Deviation from the spec when wiring the chain

According to the docs:

```
Users manually wiring their chain need to use the runtime.NewKVStoreService method to create a KVStoreService from a StoreKey:
```

```
app.ConsensusParamsKeeper = consensusparamkeeper.NewKeeper(
  appCodec,
- keys[consensusparamtypes.StoreKey]
+ runtime.NewKVStoreService(keys[consensusparamtypes.StoreKey]),
  authtypes.NewModuleAddress(govtypes.ModuleName).String(),
)

```

But in the current implementation, there is also an event that is absent in the spec:

https://github.com/code-423n4/2024-05-canto/blob/main/canto-main/app/app.go#L409
```
runtime.EventService{},
```

### Recommendation

Remove the event.




## [L-03] The module doesn't ensure that the interfaces are correct

According to the docs:

https://github.com/cosmos/cosmos-sdk/blob/v0.50.6/UPGRADING.md
```
It is possible to ensure that a module implements the correct interfaces by using compiler assertions in your x/{moduleName}/module.go:
```

```
var (
	_ module.AppModuleBasic      = (*AppModule)(nil)
	_ module.AppModuleSimulation = (*AppModule)(nil)
	_ module.HasGenesis          = (*AppModule)(nil)

	_ appmodule.AppModule        = (*AppModule)(nil)
	_ appmodule.HasBeginBlocker  = (*AppModule)(nil)
	_ appmodule.HasEndBlocker    = (*AppModule)(nil)
	...
)
```

The current implementation:

https://github.com/code-423n4/2024-05-canto/blob/main/canto-main/x/csr/module.go#L25-28
```
var (
	_ module.AppModule      = AppModule{}
	_ module.AppModuleBasic = AppModuleBasic{}
)

```

### Recommendation

Apply the checks.


