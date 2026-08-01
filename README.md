# Vafþrúðnir

> *"Wise art thou, guest — go to the giant's bench, and let us speak together in the seat; here in the hall shall our heads be the wager, guest, over wisdom's strife."*
> — Vafþrúðnismál 19. The all-wise jötunn who staked his head on right answers.

Vafþrúðnir — the interrogator of the Norse Architecture. Git-native [Bruno](https://www.usebruno.com/) collections that question every API surface across MCP, REST, and gRPC. Each request is a riddle posed, each assertion a wager; the realm claims no logic of its own — it exists solely to demand correct answers, upon forfeit of the head.

## What This Is

This repository **is** a Bruno workspace — `workspace.yml` at the root, in Bruno's OpenCollection YAML format. Clone it, open the folder in the Bruno app, and every riddle the platform must answer is on the bench. No exports, no cloud sync, no shared logins: the collection *is* the git history, reviewed like any other code.

The subject under interrogation is [Yggdrasil](https://github.com/NorseArchitecture/Yggdrasil) (`Norse.Hosting.*`) — the world-tree host that serves every realm's wire surface. Vafþrúðnir cares not how a request got where it's going, only that it arrived at the right outcome in the right wire format.

## The Contest

1. Stand up the platform — from [Bifröst](https://github.com/NorseArchitecture/Bifrost): `dotnet run --project src/Orchestration.AppHost`
2. Open this repository's folder in Bruno (it is the workspace)
3. Select the **Local** environment
4. Pose the riddles. Wrong answers forfeit the head.

gRPC contracts are code-first ([protobuf-net.Grpc](https://github.com/protobuf-net/protobuf-net.Grpc)) — there are no `.proto` files to import. Bruno discovers methods via gRPC server reflection, which Yggdrasil serves in Development.

## Layout

One collection per realm; inside each collection, one folder per wire format.

```
Vafthrudnir/
├── workspace.yml                  # the Bruno workspace — lists every collection
└── collections/
    └── Himinbjorg/                # "Himinbjörg (Authentication)"
        ├── opencollection.yml
        ├── environments/
        │   └── Local.yml
        ├── gRPC/
        │   ├── Login.yml
        │   ├── Logout.yml
        │   └── Register.yml
        └── REST/
```

## The Riddles So Far

| Collection | Realm under interrogation | Surface | Riddles |
|---|---|---|---|
| Himinbjörg (Authentication) | [Himinbjörg](https://github.com/NorseArchitecture/Himinbjorg) · [Heimdall](https://github.com/NorseArchitecture/Heimdall) | gRPC | `Login` · `Register` · `Logout` on `grpc.authentication.v1.AuthenticationService` |

## The Full Spectrum

| Wire format | Status |
|---|---|
| **gRPC** | Live — runs in the Bruno app; CLI runner support is pending upstream ([usebruno/bruno#6067](https://github.com/usebruno/bruno/issues/6067)) |
| **REST** | Staged — the third-party surface: a JWT `client_credentials` grant from Yggdrasil, then REST adapters published case-by-case over gRPC service functions for consumers who don't speak protobuf. No adapters published yet; today's riddles cover health probes and identity plumbing |
| **MCP** | Fast follow — Bruno doesn't speak MCP yet; the folder lands the day it does |

## The cosmos

Vafþrúðnir is one realm of the [Norse Architecture](https://github.com/NorseArchitecture). The whole platform composes at [Bifröst](https://github.com/NorseArchitecture/Bifrost) — clone once, cross the bridge, and every session starts there so decisions get brainstormed across the entire landscape, not in isolation. Every design is tried in [Glitnir](https://github.com/NorseArchitecture/Glitnir), the design court, before a riddle is posed here.
