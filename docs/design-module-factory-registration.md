# Design: External Module Factory Registration (方案 A)

## Problem

`RuntimeOptions.Modules` only accepts `[]host.ModuleBootstrapper` (signature: `func(chi.Router, any, *zap.Logger) error`).
External modules like Post have a `NewPostModule` factory (`core.ModuleFactory` signature: `func(logging.Logger, *Dependencies) Module`) that can receive full `Dependencies`.
There's no way to register external `ModuleFactory` functions into the `BootstrapModules` pipeline.

## Solution

Add `ModuleFactories []core.ModuleFactory` field to `RuntimeOptions`, wire it through to `BootstrapModules`.

## Changes

### 1. `core/runtime.go` (leeforge/core repo)

Add import:
```go
corecore "github.com/leeforge/core/core"
```

Add field to `RuntimeOptions`:
```go
type RuntimeOptions struct {
    ConfigPath       string
    Modules          []host.ModuleBootstrapper   // Keep for backward compat
    ModuleFactories  []corecore.ModuleFactory    // NEW: typed module factories
    ResourceProvider ResourceProvider
    SkipPlugins      bool
    SkipMigrate      bool
    BasePath         string
    Logger           *zap.Logger
    PluginRegistrar  PluginRegistrar
}
```

Modify `BuildRuntime` line 153:
```go
coreModules := buildModuleBootstrapper(opts.Modules, opts.ModuleFactories)
```

Modify `buildModuleBootstrapper`:
```go
func buildModuleBootstrapper(extra []host.ModuleBootstrapper, factories []corecore.ModuleFactory) host.ModuleBootstrapper {
    return func(router chi.Router, cfg any, logger *zap.Logger) error {
        if err := modules.BootstrapWithExtras(router, cfg, logger, factories); err != nil {
            return err
        }
        for i, bootstrap := range extra {
            if bootstrap == nil {
                continue
            }
            target := router
            if scoped, ok := any(router).(interface{ WithOwner(string) chi.Router }); ok {
                target = scoped.WithOwner(fmt.Sprintf("module[%d]", i))
            }
            if err := bootstrap(target, cfg, logger); err != nil {
                return err
            }
        }
        return nil
    }
}
```

### 2. `modules/bootstrap.go` (leeforge/core repo)

Add new exported function:
```go
func BootstrapWithExtras(router chi.Router, rawConfig any, logger *zap.Logger, extras []corecore.ModuleFactory) error {
    return registerBusinessModules(router, rawConfig, logger, extras...)
}
```

Keep existing `Bootstrap` function unchanged for backward compatibility:
```go
func Bootstrap(router chi.Router, rawConfig any, logger *zap.Logger) error {
    return registerBusinessModules(router, rawConfig, logger)
}
```

### 3. `modules/runtime_bootstrap.go` (leeforge/core repo)

Change `registerBusinessModules` signature to accept variadic extra factories:
```go
func registerBusinessModules(router chi.Router, rawConfig any, logger *zap.Logger, extraFactories ...corecore.ModuleFactory) error {
```

At `BootstrapModules` call site (around line 113), append external factories:
```go
allFactories := []corecore.ModuleFactory{
    initmod.NewInitModule,
    user.NewUserModule,
    role.NewRoleModule,
    permission.NewPermissionModule,
    menu.NewMenuModule,
    dictionary.NewDictionaryModule,
    domainmod.NewDomainModule,
    media.NewMediaModule,
    apikey.NewAPIKeyModule,
    schema.NewSchemaModule,
    mcp.NewMCPModule,
    auditlog.NewAuditLogModule,
    operationlog.NewOperationLogModule,
    systemerror.NewSystemErrorModule,
    func(frameLogging.Logger, *corecore.Dependencies) corecore.Module {
        return captchaModule
    },
}
allFactories = append(allFactories, extraFactories...)
corecore.BootstrapModules(api, private, baseLogger, deps, allFactories...)
```

### 4. `server/bootstrap/app.go` (leeforge/cms repo)

```go
import (
    corecore "github.com/leeforge/core/core"
    // Remove: "github.com/leeforge/core/host"
)

func NewApp() (*App, error) {
    return newApp(zap.NewNop(), core.RuntimeOptions{
        ConfigPath:      resolveConfigPath(),
        PluginRegistrar: registerExamplePlugins,
        ModuleFactories: []corecore.ModuleFactory{
            postmodule.NewPostModule,
        },
    })
}
```

## Testing

1. `make dev` should compile and start successfully
2. Post CRUD APIs should work at `/api/v1/posts`
3. Existing core modules should be unaffected
4. `NewPostModuleBootstrapper` can be kept for backward compat or removed
