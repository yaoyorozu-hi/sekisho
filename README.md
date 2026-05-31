# sekisho

**関所** — a hardened, reusable release gate for publishing Rust crates to crates.io.

A *sekisho* was an Edo-era inspection checkpoint on the highways: everything passing through was examined at a single, controlled, well-watched point. This repo is that checkpoint for publishing — one audited path through which a crate reaches crates.io.

It is **generic and reusable**: any Rust repo or workspace can consume it. It lives in the [`yaoyorozu-hi`](https://github.com/yaoyorozu-hi) org as shared release-gate tooling.

## What this is

`sekisho` provides a **composite GitHub Action** (`action.yml` at the repo root) that publishes crates to [crates.io](https://crates.io) via **OIDC Trusted Publishing** — no long-lived `CARGO_REGISTRY_TOKEN`. A consuming repo calls it from a SHA-pinned step inside its own `publish.yml`.

### Why a composite action and not a reusable workflow

crates.io Trusted Publishing matches the **caller** workflow's `workflow_ref` and `environment` claim — it does **not** look at `job_workflow_ref`. So the trust anchor registered on crates.io is the *consuming repo's own* `publish.yml` plus its `environment: release`; sekisho's location is invisible to Trusted Publishing.

A reusable-workflow (`uses:` at the job level via `workflow_call`) caller job **cannot** declare `environment:`. Trusted Publishing requires the environment claim on the job that mints the OIDC token. Therefore sekisho ships a **composite action**, invoked from a normal job in each consuming repo's `publish.yml` — a job that declares `environment: release` and `id-token: write` itself.

## What the action does

Inputs:

| Input | Required | Description |
|-------|----------|-------------|
| `packages` | yes | Space-separated crate names in dependency/publish order, e.g. `"my-crate my-crate-cli"`. The library must come before any crate that depends on it. |

Steps (composite):

1. **Checkout** (`actions/checkout`, `persist-credentials: false`).
2. **Install stable Rust** via stock `rustup` (no third-party toolchain action).
3. **Mint a crates.io OIDC token** via `rust-lang/crates-io-auth-action`. The token is short-lived and is **auto-revoked** by the action's post step when the job ends.
4. **Publish in order** — for each package, `cargo publish -p <pkg> --locked --token <minted>`, masking the token. Between packages it polls the crates.io sparse index for the previously published crate so the next package (which may depend on it) can resolve the new version before its publish runs.

## harden-runner placement (important)

[`step-security/harden-runner`](https://github.com/step-security/harden-runner) **must be the first step of the job**, and nesting it inside a composite action is unreliable. So this action does **not** run harden-runner. The consuming `publish.yml` must run it as the first step of the publish job, before the `uses: yaoyorozu-hi/sekisho@<sha>` step.

## To use sekisho in a repo

In the consuming repo, add `.github/workflows/publish.yml`. The tag (created by release-plz or by hand) is what triggers it:

```yaml
name: publish
on:
  push:
    tags:
      - "v*.*.*"
jobs:
  publish:
    runs-on: ubuntu-latest
    environment: release          # Trust anchor #1 — must match the crates.io TP config
    permissions:
      contents: read
      id-token: write             # required to mint the OIDC token
    steps:
      # harden-runner MUST be the first step of the job (not inside the action).
      - uses: step-security/harden-runner@<full-sha>  # v2.x
        with:
          egress-policy: block
          allowed-endpoints: >
            api.github.com:443
            github.com:443
            objects.githubusercontent.com:443
            token.actions.githubusercontent.com:443
            crates.io:443
            static.crates.io:443
            index.crates.io:443
            static.rust-lang.org:443
      - uses: yaoyorozu-hi/sekisho@<full-sha>   # pin to a full commit SHA, never a tag
        with:
          packages: "your-crate your-crate-cli"
```

Pin the `yaoyorozu-hi/sekisho@<sha>` reference to a **full commit SHA**, never a mutable tag — that is the gate, and it must not be swappable underneath callers.

## crates.io Trusted Publishing setup (required, out of band)

For **each** crate published through this action, configure Trusted Publishing on crates.io (crate Settings → Trusted Publishing → GitHub):

- **Repository owner:** the consuming repo's org (e.g. `yaoyorozu-hi`)
- **Repository name:** the consuming crate repo
- **Workflow filename:** `publish.yml` (the caller — *not* anything in sekisho)
- **Environment:** `release`

Also create a GitHub **Environment** named `release` in the consuming repo (protection rules / required reviewers optional but recommended) so the privileged publish is gated.

No `CARGO_REGISTRY_TOKEN` secret is needed anywhere; the OIDC token is minted per run and auto-revoked.

## Why a separate, gated repo

The publish path is the highest-privilege operation in the supply chain — it mints artifacts that downstream users trust. Concentrating it here, minimal and branch-protected, makes it **auditable and change-gated**:

- **Privilege separation.** The publish logic is isolated from third-party release tooling (e.g. release-plz) running in the consuming repos. A compromise or malicious update of that tooling does not directly hold the publish capability.
- **SHA-pinning.** Consumers pin this action (and it pins its own actions) by commit SHA, not a mutable tag — so the gate cannot be silently swapped underneath callers. `.github/dependabot.yml` opens reviewed bump PRs for the pinned actions.
- **Environment gating.** Publishing runs under a protected GitHub Environment in the caller, so the privileged step is reviewable and restricted to the intended ref.
- **Branch protection.** `main` requires a reviewed PR; the high-privilege path cannot change without a second look.

## Scope

Deliberately minimal. This repo holds the gate and nothing else. It may grow into a `sekisho-*` family of gates (other audited, privilege-separated CI paths) if the need arises.

## License

Dual-licensed under either of:

- MIT license ([LICENSE-MIT](LICENSE-MIT))
- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE))

at your option.
