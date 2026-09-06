# Fuuz Integration Accelerator — modular

Cross-tenant and external connections: the orchestrator that moves data between Fuuz and other systems.

**`Integration Orchestrator@0.0.9`** · platform `2026.7.0` · spec `2.0.0`

This is the **modular** publication: the package unpacked into its components so
each module can be read, reviewed and diffed. The monolithic package is in
[`package/`](package/) and is what you install.

## Modules (1)

| Module | | Components |
|---|---|---|
| `integration` | Integration | 4 dataFlows · 5 screens |

## Components without a module

Only `dataFlows` and `screens` carry `moduleId`. These types carry none, so they
are filed by type rather than guessed at:

| Path | Count |
|---|---:|
| `dataModels/` | 6 |
| `data/` | 6 |

## Layout

```
manifest.json                 name, version, platformVersion
definition.json               package definition + selections
modules/<module>/<type>/*.json
dataModels/ documentDesigns/ data/
package/                      the installable .fuuz
docs/                         guides and technical overview
```

## Installing

Install `package/Integration Orchestrator@0.0.9.fuuz` through the Fuuz platform's package
installer. The exploded tree is for reading and reviewing; it is not the install
artifact.
