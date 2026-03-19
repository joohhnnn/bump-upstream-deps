# bump-upstream-deps

A Claude Code skill that automatically bumps upstream Rust crate dependencies across the Ethereum ecosystem.

## Problem

When upstream libraries like `alloy`, `revm`, or `alloy-core` release new versions, downstream repos (`reth`, `foundry`, etc.) need manual updates. This is repetitive work that AI can handle.

## Supported Dependency Graph

```
alloy / alloy-core
  ├── reth (paradigmxyz/reth)
  ├── foundry (foundry-rs/foundry)
  └── revm
        ├── reth
        └── foundry
```

## Install as Claude Code Skill

```bash
cp bump-upstream-deps.md ~/.claude/skills/
```

Then in any Rust project, ask Claude Code:
```
bump alloy to latest
```

## What It Does

1. Checks latest upstream version (crates.io / GitHub releases)
2. Scans all `Cargo.toml` files in the target repo
3. Updates workspace and member crate versions
4. Runs `cargo check` / `cargo test` to verify
5. Reports breaking changes from upstream CHANGELOG
6. Generates a clean commit message

## License

MIT
