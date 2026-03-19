---
name: bump-upstream-deps
description: Use when an upstream Rust crate (alloy, revm, alloy-core) releases a new version and downstream repos (reth, foundry) need to adopt the update. Also use when asked to check for outdated dependencies across Ethereum Rust ecosystem repos.
---

# Bump Upstream Dependencies

Automatically detect and apply upstream dependency updates across Ethereum Rust ecosystem repos.

## Supported Dependency Graph

```
alloy / alloy-core
  ├── reth (paradigmxyz/reth)
  ├── foundry (foundry-rs/foundry)
  └── revm
        ├── reth
        └── foundry
```

## Workflow

1. **Detect**: Check latest version of the upstream crate on crates.io
2. **Scan**: Find all `Cargo.toml` files in the target repo that reference the crate
3. **Diff**: Compare current pinned version vs latest
4. **Update**: Bump version in all `Cargo.toml` files (workspace root + member crates)
5. **Verify**: Run `cargo check` (and `cargo test` if requested) to confirm compatibility
6. **Report**: List breaking changes from the upstream CHANGELOG if available

## Steps

### 1. Check latest upstream version

```bash
# Check latest version on crates.io
cargo search alloy --limit 1
# Or fetch from GitHub releases
gh api repos/alloy-rs/alloy/releases/latest --jq '.tag_name'
```

### 2. Find current version in target repo

```bash
# Find all Cargo.toml files referencing the crate
grep -r 'alloy' --include='Cargo.toml' -l .
# Check workspace-level version
grep -A2 'alloy' Cargo.toml | head -20
```

### 3. Update versions

- Update workspace `Cargo.toml` dependency table first
- For workspace-inherited deps, only the root needs changing
- For pinned versions in member crates, update each one
- Handle renamed crates (e.g., `alloy-primitives` → check if re-exported)

### 4. Handle breaking changes

```bash
# Check upstream changelog
gh api repos/alloy-rs/alloy/releases/latest --jq '.body' | head -50

# After bumping, run cargo check to find breakage
cargo check 2>&1 | head -100

# Common fixes:
# - Renamed types/functions → search-replace
# - Removed re-exports → add direct dependency
# - Changed trait bounds → update impl blocks
```

### 5. Verify

```bash
cargo check
cargo test --workspace --no-fail-fast 2>&1 | tail -20
cargo clippy --workspace 2>&1 | tail -20
```

## Common Mistakes

- Forgetting to update `Cargo.lock` (run `cargo update -p <crate>`)
- Missing workspace member crates that pin their own version
- Not checking for feature flag changes in new version
- Bumping only one crate from a family (e.g., bumping `alloy` but not `alloy-core`)

## Output

Generate a commit message like:
```
chore(deps): bump alloy from 0.x.y to 0.x.z

Updated alloy dependency across workspace.
Changes: [brief summary from changelog]
```
