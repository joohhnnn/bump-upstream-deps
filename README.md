# bump-upstream-deps

A Claude Code skill that automatically bumps upstream Rust crate dependencies across the Ethereum ecosystem.

## Problem

When upstream libraries like `alloy`, `revm`, or `alloy-core` release new versions, downstream repos (`reth`, `foundry`, etc.) need manual updates. Simple version bumps often fail because of `[patch]` git overrides, cross-repo compatibility constraints, and breaking API changes. Dependabot/Renovate can't handle this — AI can.

## Dependency Graph

```
alloy-core (alloy-primitives, alloy-sol-types, alloy-rlp, alloy-dyn-abi)
  │
alloy (alloy-consensus, alloy-eips, alloy-provider, alloy-rpc-types, ...)
  │
revm ←── depends on alloy-core + some alloy crates
  │
alloy-evm ←── BRIDGES alloy + revm (critical middle layer)
  │
  ├── reth (paradigmxyz/reth)
  ├── foundry (foundry-rs/foundry)
  │
  └── Extensions:
        op-alloy / op-revm (OP Stack)
        tempo-primitives / tempo-alloy (Tempo)
```

**Key insight**: `alloy-evm` bridges alloy and revm. When either side bumps a major version, `alloy-evm` must be updated first, then downstream repos can follow.

## Validated Results

Tested on foundry (2026-03-19):

| Bump | Result |
|------|--------|
| alloy-core 1.5.2 → 1.5.7 | ✅ cargo check passed, zero errors |
| revm 34 → 36 | ❌ 18 errors — needs coordinated alloy-evm + `[patch]` update |

Foundry has **42 `[patch.crates-io]` git overrides** that silently override crates.io versions. This is the #1 reason simple version bumps fail, and why this skill exists.

## Install as Claude Code Skill

```bash
cp bump-upstream-deps.md ~/.claude/skills/
```

Then in any Rust project, ask Claude Code:
```
bump alloy to latest
```

## What It Does

1. Detects dep management pattern (workspace inheritance vs direct pinning)
2. Checks latest upstream version (crates.io + GitHub releases for pre-release)
3. Maps crate families (alloy / alloy-core / revm / op-alloy)
4. Reads upstream CHANGELOG for breaking changes
5. Updates versions in root `Cargo.toml` (handles both `= "ver"` and `{ version = "ver" }` formats)
6. Detects and updates `[patch.crates-io]` git overrides
7. Runs `cargo check` / `cargo test` to verify
8. Generates commit message with changelog summary

## License

MIT
