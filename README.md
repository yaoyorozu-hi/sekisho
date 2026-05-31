# sekisho

**関所** — the hardened release gate for the [tsukai-mcp](https://github.com/tsukai-mcp) Rust crates.

A *sekisho* was an Edo-era inspection checkpoint on the highways: everything passing through was examined at a single, controlled, well-watched point. This repo is that checkpoint for publishing — the one audited path through which tsukai-mcp crates reach crates.io.

## What this is

`sekisho` provides a **reusable GitHub Actions workflow** that publishes crates to [crates.io](https://crates.io) via **OIDC Trusted Publishing** (no long-lived registry token). Sibling repos call it through a SHA-pinned [`workflow_call`](https://docs.github.com/en/actions/using-workflows/reusing-workflows):

- [`tsukai-manifest`](https://github.com/tsukai-mcp/tsukai-manifest)
- [`tsukai`](https://github.com/tsukai-mcp/tsukai)
- [`tsukai-derive`](https://github.com/tsukai-mcp/tsukai-derive)

The reusable workflow lives at `.github/workflows/publish.yml` (added separately, after the crates.io Trusted Publishing details are verified).

## Why a separate, gated repo

The publish path is the highest-privilege operation in the supply chain — it mints artifacts that downstream users trust. Concentrating it here, minimal and branch-protected, makes it **auditable and change-gated**:

- **Privilege separation.** The OIDC publish credential is exercised only by this audited workflow, isolated from third-party release tooling (e.g. release-plz) running in the consuming repos. Compromise or a malicious update of that tooling does not directly hold the publish capability.
- **SHA-pinning.** Consumers pin this reusable workflow (and it pins its own actions) by commit SHA, not a mutable tag — so the gate cannot be silently swapped underneath callers.
- **Environment gating.** Publishing runs under a protected GitHub Environment, so the privileged step is reviewable and restricted to the intended ref.
- **Branch protection.** `main` requires a reviewed PR; the high-privilege path cannot change without a second look.

## Scope

Deliberately minimal. This repo holds the gate and nothing else. It may grow into a `sekisho-*` family of gates (other audited, privilege-separated CI paths) if the need arises.

## License

Dual-licensed under either of:

- MIT license ([LICENSE-MIT](LICENSE-MIT))
- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE))

at your option.
