# Versiqii

**Universal Version‑Aware Configuration Management (VACM)** for any component that ships with a version number — micro‑services, DB schemas, ML models, front‑end bundles, you name it.

> *Shift versions without drift* — map, validate & migrate every config so it always matches the exact version (and dependencies) it is deployed with.

---

## Table of Contents

* [Why Versiqii?](#why-versiqii)
* [Key Features](#key-features)
* [Core Concepts](#core-concepts)
* [Architecture Overview](#architecture-overview)
* [Quick Start](#quick-start)
* [CLI Usage](#cli-usage)
* [API Reference](#api-reference)
* [Configuration Examples](#configuration-examples)
* [Roadmap](#roadmap)
* [Contributing](#contributing)
* [License](#license)

---

## Why Versiqii?

Modern platforms are a **matrix of versions**: micro‑services, databases, function runtimes, ML models… each moving at its own cadence. A single upgrade can break configs, schema contracts, or downstream dependencies.

**Versiqii** acts as the **control‑plane** that normalises this chaos:

1. **Discovers** every *Target* (anything with a `version`) across all environments.
2. **Normalises** each Target into a *Target Family* with a JSON‑Schema (or Avro/Protobuf) contract.
3. **Tracks** a *VersionGraph* to understand cross‑component dependencies.
4. **Binds & Validates** configs at deploy time — blocking drifts before they hit prod.
5. **Migrates** configs automatically (WASM/Lua scripts) when versions shift.
6. **Audits & Rolls Back** in one call when things go south.

> From Terraform plans to feature flags, Versiqii keeps every file compatible with the version that will actually run.

---

## Key Features

| Category                   | What it does                                                                              |
| -------------------------- | ----------------------------------------------------------------------------------------- |
| **Schema Registry**        | Draft‑2020‑12 JSON‑Schema (plus Avro/Proto) with evolution metadata.                      |
| **VersionGraph**           | DAG of Target Versions and their dependency constraints (`requires`, `accepts`).          |
| **Immutable Config Store** | Write‑once, append‑only history with Git‑style diff.                                      |
| **Drift Matrix**           | Multi‑dimensional view (Target, Schema, Dependency, Environment) with Prometheus metrics. |
| **WASM Migration Runtime** | Sandboxed scripts to upgrade/downgrade configs on the fly.                                |
| **GitOps Plugins**         | FluxCD / Argo CD integrations: `versiqii-validate` gate in your pipeline.                 |
| **Typed SDKs**             | Rust · Go · Node · Python clients auto‑generated from OpenAPI/gRPC.                       |
| **Audit & Policy**         | OPA/Casbin rules, event log, one‑click rollback.                                          |

---

## Core Concepts

| Term                | Description                                  | Example                                                    |
| ------------------- | -------------------------------------------- | ---------------------------------------------------------- |
| **Target**          | Anything that has a `version` & a schema     | `payment‑service`, `feature‑flag‑api`, `ml‑model@resnet50` |
| **Target Family**   | Group of Targets sharing one schema language | `OpenAPI‑Payments`, `Avro‑OrderFeed`                       |
| **Target Version**  | Concrete version (`semver`, `git‑sha`, date) | `api@3.2.1`, `db@14`, `frontend@2025‑07‑03`                |
| **Config Instance** | Immutable JSON/YAML/Proto payload            | `payment‑service.yaml`                                     |
| **Binding**         | Links Config ▶ Environment ▶ TargetVersion   | `payment‑service.yaml` → `prod/eu‑fr` @ `api@3.2.1`        |
| **VersionGraph**    | DAG of TargetVersion dependencies            | `auth@1.5 → api@3.2`                                       |

---

## Architecture Overview

```mermaid
flowchart TB
    subgraph Control‑Plane
        SR[Schema Registry]
        CS[Config Store]
        VG[VersionGraph Service]
        MS[Migration Service]
        AG[API Gateway]
        SR --> AG
        CS --> AG
        VG --> AG
        MS --> CS
    end

    subgraph Env[Environment / Cluster]
        DV[Deployment Validator]
    end

    AG -- stream --> DV
    DV -- validated config --> Deployer
    DV -- drift event --> MS
```

* **Control‑Plane** (single writer, multi‑region CockroachDB) — source‑of‑truth.
* **Deployment Validator** runs as a sidecar or admission webhook in each cluster.

More diagrams in [docs/architecture.md](docs/architecture.md).

---

## Quick Start

\### 1. Clone & Build

```bash
git clone https://github.com/your‑org/versiqii.git
cd versiqii
make dev  # spins up CockroachDB, NATS & the service
```

> Requires **Rust 1.79+**, **Docker 24+**, and **make**.

\### 2. Hello World

```bash
# Register a Target Family schema
versiqii schemas add --family OpenAPI‑Payments --version 2025‑07 ./examples/openapi_payments.json

# Declare a Target (micro‑service) & its version
authTarget=$(versiqii targets create --name payment‑api --family OpenAPI‑Payments --version 3.2.1)

# Store a config instance
cfgId=$(cat examples/payment_service.yaml | versiqii configs create --target $authTarget -)

# Bind to environment "prod‑eu" (auto‑validates & migrates)
versiqii bindings create --config $cfgId --env prod‑eu
```

\### 3. See It in Action

Open Swagger UI at [http://localhost:8080/docs](http://localhost:8080/docs) after `make dev`.

---

## CLI Usage

| Command           | Purpose                                          |
| ----------------- | ------------------------------------------------ |
| `schemas add`     | Register / update a schema (JSON, Avro, Proto)   |
| `targets create`  | Declare a new Target & version                   |
| `graphs link`     | Add dependency edge in VersionGraph              |
| `configs create`  | Store immutable config instance                  |
| `bindings create` | Bind + auto‑migrate to environment               |
| `drifts list`     | Show drift matrix (all dimensions)               |
| `env set‑target`  | Change deployed TargetVersion for an environment |

Run `versiqii --help` for full options.

---

## API Reference

OpenAPI + gRPC. Full spec lives in [`spec/openapi.yaml`](spec/openapi.yaml).

```http
POST /targets
PUT  /envs/{env}/targets/{target}/version
GET  /drifts?env=prod‑us&target=api
```

---

## Configuration Examples

```yaml
# payment_service.yaml
scan_depth: 3
timeout_ms: 5000
features:
  use_heuristics: true
  max_rules: 32
```

Validate locally:

```bash
versiqii validate --file payment_service.yaml --target payment‑api@3.2.1
```

---

## Roadmap

| Phase                | Milestone                                    | Target Date |
| -------------------- | -------------------------------------------- | ----------- |
| **α (Core)**         | Schema & Target registry, single‑env binding | 2025‑08     |
| **β (Drift Matrix)** | VersionGraph, Prom metrics                   | 2025‑09     |
| **γ (Migration)**    | WASM runtime, upgrade scripts                | 2025‑10     |
| **δ (GitOps)**       | FluxCD/Argo plugins, policy engine           | 2025‑11     |
| **GA**               | Multi‑tenant SaaS, SLA 99.9%                 | 2026‑Q1     |

---

## Contributing

We ❤️ pull requests! Please read [CONTRIBUTING.md](CONTRIBUTING.md) first.

```bash
git checkout -b feat/amazing
cargo test
cargo fmt --check
```

Open an issue to discuss big changes before coding.

---

## License

Versiqii is licensed under the **Apache License 2.0**. See [LICENSE](LICENSE) for details.

---

> *Made with ☕, Rust & a splash of version‑control magic by the Versiqii team — shift safely, ship faster.*
