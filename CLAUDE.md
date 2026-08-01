# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 1. What This Repository Is

**Vafþrúðnir** is the interrogator of the Norse Architecture — a git-native [Bruno](https://www.usebruno.com/) workspace that tests Yggdrasil's hosted API footprint across every realm and wire format. It contains **no logic of its own**: only requests, assertions, environments, and the docs that govern them. If a change here needs code, the code belongs in a realm, not here.

The repo root **is** the Bruno workspace (`workspace.yml`). Cloning it and opening the folder in the Bruno app is the entire setup. It rides beside the other realms as a Bifröst submodule; sessions start from Bifröst, so sibling paths (`../../Yggdrasil/...`, `../Glitnir/docs/...` from Bifröst root) resolve.

**File format is Bruno's OpenCollection YAML (`opencollection: 1.0.0`)** — `workspace.yml` at the root, one `opencollection.yml` per collection, requests as `{Name}.yml`, folder settings in `folder.yml`, environments in `environments/{Name}.yml`. The legacy `.bru` format is not used here. The Bruno app is the primary author of request files; hand edits are fine but must round-trip cleanly through the app — let `git diff` be the review.

## 2. Layout

One collection per realm under `collections/`; inside each collection, one folder per wire format. Disk paths are ASCII (matching `.gitmodules` convention: `Himinbjorg`, not `Himinbjörg`); the lore spelling plus function lives in the collection's display name (`info.name: Himinbjörg (Authentication)`).

```
Vafthrudnir/
├── workspace.yml                  # opencollection workspace — lists every collection
├── .gitignore                     # .env*, node_modules — secrets never land
├── collections/
│   └── Himinbjorg/                # display name: "Himinbjörg (Authentication)"
│       ├── opencollection.yml
│       ├── environments/
│       │   └── Local.yml          # url → Yggdrasil's Hosting.Web.Server
│       ├── gRPC/
│       │   ├── folder.yml
│       │   ├── Login.yml
│       │   ├── Logout.yml
│       │   └── Register.yml
│       └── REST/                  # third-party adapter surface — see §4
├── CLAUDE.md · README.md · LICENSE
```

- **New realm surface lands?** New collection folder + an entry in `workspace.yml`'s `collections:` list, in the same change.
- **One `.gitignore`, at the repo root.** Bruno drops a per-collection `.gitignore` when it scaffolds a collection as a standalone git repo — redundant inside this repo; delete it on sight.
- **MCP is a declared fast-follow, not a folder.** Bruno does not speak MCP yet; when it does, an `MCP/` folder lands beside `REST/` and `gRPC/` in each collection that needs one. Do not scaffold empty `MCP/` folders ahead of support.
- Environments are per-collection by Bruno's design; every realm collection carries its own `environments/Local.yml` with the same `url` variable. Accepted duplication — one variable, one host.

## 3. Commands

```sh
# Stand up the target platform (from Bifröst root — starts Postgres, migrations, web)
dotnet run --project src/Orchestration.AppHost

# CLI runner (REST only today — see caveat)
npm i -g @usebruno/cli
cd collections/Himinbjorg && bru run --env Local          # whole collection
cd collections/Himinbjorg && bru run REST --env Local     # one folder
```

- **`bru run` cannot execute gRPC requests yet** ([usebruno/bruno#6067](https://github.com/usebruno/bruno/issues/6067), [#6068](https://github.com/usebruno/bruno/issues/6068)) — gRPC riddles run from the Bruno app's runner only. CI on this repo is therefore REST-only until upstream lands; do not fake gRPC coverage in CI with curl reimplementations.
- There is no build and no lint; the verification gate for changes here is opening the workspace in Bruno and running the touched requests against a live AppHost.

## 4. The Target Surface (what the riddles interrogate)

Everything targets **Yggdrasil's `Hosting.Web.Server`** — the Aspire `web` resource, `https://yggdrasil.dev.localhost:5001` by launch profile. Facts that shape every request:

- **TLS is mandatory; there is no HTTP profile.** gRPC requests must have TLS explicitly enabled — plaintext h2c fails with a misleading `14 UNAVAILABLE`. This trap has been hit before (Postman); it is the most likely setup failure.
- The dev certificate is issued for `localhost` only, which is why `Local.yml` sets `url: localhost:5001` rather than the `yggdrasil.dev.localhost` name — the name would fail hostname validation unless cert verification is disabled.
- **gRPC is the native wire; REST is the third-party surface.** For platform consumers, business endpoints are gRPC-only by law (`../Glitnir/docs/Yggdrasil/specs/2026-05-20-yggdrasil-hosting-design.md`). REST exists for third parties who don't speak protobuf: they obtain a JWT from Yggdrasil via the OAuth `client_credentials` grant, then call REST adapters published over specific gRPC service functions — case by case, never wholesale (query surfaces are expected to publish by default early on). No adapters exist yet; today's REST surface is framework plumbing only: `GET /livez`, `GET /readyz` (anonymous, plain text), form-encoded antiforgery-gated `/Account/*` identity endpoints, and `/_auth/complete`. Each adapter that publishes earns riddles in its realm's `REST/` folder — including the unauthenticated `401`.
- **Contracts are code-first protobuf-net.Grpc — zero `.proto` files exist anywhere.** Method discovery uses gRPC **server reflection**, which Yggdrasil maps **in Development only**. If reflection returns nothing, check `ASPNETCORE_ENVIRONMENT` before suspecting Bruno.
- Wire identity comes from `[ServiceContract(Name = ...)]`, not CLR namespaces. Live today: `grpc.authentication.v1.AuthenticationService` (`Login` / `Register` / `Logout` — contract: `../../Heimdall/src/AuthN.Services/IAuthenticationService.cs`, implementation in Himinbjörg). `Logout` has no request DTO — send `{}`. Also mapped: `grpc.health.v1.Health`.
- **Future coverage is generator-driven:** any realm interface named `I{Context}Service` carrying `[ServiceContract]` with `Task<Outcome<T>>` methods is auto-mapped by `MapNorseGrpcServices()`. A new contract matching that shape means a new collection here.

## 5. Assertion Semantics (get these right or the head is forfeit)

The pipeline returns `Outcome<T>`; failures cross the wire as gRPC errors encoded by `OutcomeServerInterceptor` (`google.rpc.Status` + details on the `grpc-status-details-bin` trailer, status detail string = `ErrorCategory` name).

- **`Login` with wrong credentials is gRPC `OK` with `Succeeded: false`** — deliberately indistinguishable from unknown-user (anti-enumeration). Never assert a non-OK status for a failed login.
- **`Register` failures ARE gRPC errors**: duplicate email → `AlreadyExists` with `"Conflict"` detail; rejected password → validation error. Assert the trailer, not a `Succeeded` flag.
- **`DeferredCompletionUrl` is always absent over the wire** — it is populated only on the in-process Blazor Server path. Assert null/absent.
- All three AuthN methods are callable unauthenticated (`AuthN.Public` policy passes everything). A cookie-less `Logout` proves the RPC completes, not that a session was cleared.

## 6. Process

- **No automatic git commits.** Stage and show the diff; the human commits.
- **Secrets never enter the record.** `.env*` is gitignored; environments hold non-secret variables only. Seed test credentials (`user@norse.org`) are fixtures, not secrets — fine in request files.
- **README.md and CLAUDE.md stay in sync — boy-scout law.** README is the public narrative, this file the working law. A change to layout, collections, or conventions updates both in the same change, and both must match `workspace.yml`.
- Adding a collection also updates the realm table in README.md and the `collections:` list in `workspace.yml` — three files, one change.
