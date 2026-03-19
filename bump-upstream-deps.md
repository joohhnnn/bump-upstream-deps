---
name: bump-upstream-deps
description: Use when an upstream Rust crate (alloy, revm, alloy-core) releases a new version and downstream repos (reth, foundry) need to adopt the update. Also use when asked to check for outdated dependencies across Ethereum Rust ecosystem repos.
---

# Bump Upstream Dependencies

Automatically detect and apply upstream dependency updates across Ethereum Rust ecosystem repos.

## Dependency Graph

Dependencies are cross-cutting, not linear:

```
            alloy-core (primitives, sol-types, rlp, dyn-abi)
           ╱    │     ╲
        alloy   │    revm ←── uses alloy-core AND some alloy crates
           ╲    │     ╱
          alloy-evm (bridges alloy + revm)
           ╱       ╲
        reth      foundry
```

Who depends on what:

| Project | alloy-core | alloy | revm | alloy-evm |
|---------|-----------|-------|------|-----------|
| alloy | ✓ | — | ✗ | ✗ |
| revm | ✓ | ✓ partial | — | ✗ |
| alloy-evm | ✓ | ✓ | ✓ | — |
| reth | ✓ | ✓ | ✓ | ✓ |
| foundry | ✓ | ✓ | ✓ | ✓ |

**Key**: revm directly uses alloy crates (consensus, eips, provider), not just alloy-core. This means an alloy major bump can break revm too. Always check the full matrix before bumping.

## Step 0: Understand the Target Repo's Dep Structure

Before anything, determine the dep management pattern:

```bash
# Check if workspace inheritance is used (most Ethereum repos do this)
grep -c 'workspace = true' crates/*/Cargo.toml 2>/dev/null

# If high count → only root Cargo.toml needs version changes
# If zero → each member crate pins its own version, all need updating
```

**Foundry**: 215 workspace-inherited deps, only change root `Cargo.toml`
**Reth**: Similar workspace pattern

## Step 1: Check Versions

```bash
# Latest stable on crates.io
cargo search alloy --limit 1
cargo search revm --limit 1

# Latest pre-release (check GitHub — crates.io doesn't show rc/beta)
gh api repos/alloy-rs/alloy/releases --jq '.[0].tag_name'
gh api repos/bluealloy/revm/releases --jq '.[0].tag_name'

# Current version in target repo
grep -E '^alloy.*version' Cargo.toml | head -5
grep -E '^revm.*version' Cargo.toml | head -5
```

**Important**: If the repo uses pre-release versions (e.g. `2.0.0-rc.0`), track the pre-release channel, not the stable channel. Check GitHub releases, not just crates.io.

## Step 2: Read the Changelog

```bash
# Get release notes for version diff
gh api repos/alloy-rs/alloy/releases/latest --jq '.body' | head -50

# For breaking changes, look for:
# - "BREAKING" in release notes
# - Major version bumps in sub-crates
# - Renamed/removed types
# - Changed trait bounds
# - Feature flag renames
```

## Step 3: Update Versions

For workspace-inherited repos (most common):

```bash
# Find all alloy dep lines in root Cargo.toml
grep -n 'alloy.*version' Cargo.toml
```

**Caution with sed**: Cargo.toml has two dep formats:
```toml
alloy-dyn-abi = "1.5.2"                              # simple format
alloy-primitives = { version = "1.5.2", features = [] }  # inline table
```
Use `sed 's/= "OLD"/= "NEW"/g'` (not `version = "OLD"`) to catch both formats.

Update the version string for each group. The alloy crate families:

| Family | Sub-crates | Version tracks | Cross-deps |
|--------|-----------|---------------|------------|
| **alloy-core** | alloy-primitives, alloy-sol-types, alloy-dyn-abi, alloy-rlp | alloy-core version | none (root) |
| **alloy** | alloy-consensus, alloy-eips, alloy-network, alloy-provider, alloy-rpc-types, alloy-signer, alloy-transport, etc. | alloy main version | uses alloy-core |
| **revm** | revm, revm-interpreter, revm-context, revm-handler, revm-inspector, revm-precompile, etc. | revm main version | uses alloy-core + alloy (consensus, eips, provider) |
| **alloy-evm** | alloy-evm | alloy-evm version | uses alloy + revm (bridge layer) |
| **op-alloy** | op-alloy-consensus, op-alloy-rpc-types, op-revm, etc. | op-alloy version | uses alloy + revm |

## Step 4: Fix Breakage

```bash
# Run cargo check to find compile errors
cargo check 2>&1 | head -100

# Common fix patterns:
# 1. Renamed types → grep old name, replace with new
# 2. Removed re-exports → add direct dep on the sub-crate
# 3. Changed trait bounds → update impl blocks
# 4. Feature flag changes → check new Cargo.toml of upstream
# 5. Method signature changes → read the diff/changelog
```

For feature flag changes:
```bash
# Compare features between versions
cargo info alloy-consensus  # shows available features
# Cross-check with what the repo uses
grep -A5 'alloy-consensus' Cargo.toml
```

## Step 5: Verify

```bash
cargo check
cargo clippy --workspace 2>&1 | tail -20
cargo test --workspace --no-fail-fast 2>&1 | tail -30
```

## Step 6: Coordinated Multi-Crate Bumps

When alloy and revm need simultaneous bumps:

1. Bump alloy first (more foundational)
2. Then bump revm (may depend on new alloy types)
3. `cargo check` after each to isolate breakage source
4. If circular dependency issues, bump both at once

## Step 7: Handle `[patch]` Git Overrides (Critical!)

Many Ethereum repos (foundry, reth) use `[patch.crates-io]` to pin deps to git branches/commits instead of crates.io versions. **This is the #1 reason simple version bumps fail.**

```bash
# Check for patch overrides
grep -c 'git.*alloy\|git.*revm' Cargo.toml
# If > 0, you MUST update these too
```

Example from foundry:
```toml
[patch.crates-io]
alloy-consensus = { git = "https://github.com/alloy-rs/alloy", rev = "100a332..." }
alloy-evm = { git = "https://github.com/alloy-rs/evm.git", branch = "alloy-2.0" }
```

When bumping revm, you need to:
1. Update the `[dependencies]` version
2. Update the `[patch]` rev/branch to a commit that supports the new revm
3. Ensure `alloy-evm` git branch is compatible with both new revm AND current alloy

**This is the hard part that Dependabot/Renovate cannot do** — it requires understanding the cross-repo compatibility matrix.

## Common Mistakes

- **Stale `Cargo.lock`**: After a failed bump, `Cargo.lock` retains residual changes. Always `git checkout -- Cargo.lock` before retrying
- **`[patch]` silent ignore**: If `[dependencies]` says `0.36.1` but `[patch]` pins `0.36.0`, the patch is **silently ignored**, pulling incompatible transitive deps from crates.io. Versions must match exactly
- Bumping `alloy` but not `alloy-core` (or vice versa) when they released together
- Not checking pre-release channel when repo uses `rc`/`beta` versions
- Forgetting `cargo update -p <crate>` to sync `Cargo.lock`
- Missing feature flag renames in new version
- Not checking if `op-alloy` also needs bumping (for OP Stack repos)

## Output

```
chore(deps): bump alloy from X to Y

Updated alloy family deps across workspace:
- alloy-consensus, alloy-eips, ... : X → Y
- alloy-primitives: A → B (if changed)

Breaking changes addressed:
- [brief description]
```
