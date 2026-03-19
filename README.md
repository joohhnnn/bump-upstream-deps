# bump-upstream-deps

A Claude Code skill for bumping upstream Rust crate dependencies across the Ethereum ecosystem (reth, foundry).

## Problem

When upstream libraries like `alloy`, `revm`, or `alloy-evm` release new versions, downstream repos need manual updates. Simple version bumps often fail because of `[patch]` git overrides, cross-repo compatibility constraints, and breaking API changes. Dependabot/Renovate can't handle this — AI can.

## Install

```bash
# Claude Code requires skills in a directory with SKILL.md
mkdir -p ~/.claude/skills/bump-upstream-deps
cp bump-upstream-deps.md ~/.claude/skills/bump-upstream-deps/SKILL.md
```

> **Important**: A bare `.md` file in `~/.claude/skills/` will NOT be loaded. It must be `<skill-name>/SKILL.md`.

Then in any Rust project:
```
bump alloy-evm to 0.28.0
```

## Validated Results

### Test 1: reth — revm 35→36 + alloy-evm 0.29.1→0.29.2 (no breaking changes)

Pure version bump, no API changes. Tests lockfile precision.

| Metric | Without skill | With skill (v1) | With skill (v2) |
|--------|--------------|-----------------|-----------------|
| Cargo.toml | ✅ | ✅ | ✅ |
| Cargo.lock | ❌ 32 unrelated transitive dep changes (windows-sys, syn, socket2) | ❌ same (used `git checkout -- Cargo.lock`) | ✅ exact — only 11 target crates |

**Fix**: `cargo update -p <crate>` instead of resetting entire lockfile.

### Test 2: reth — alloy-evm 0.27.2→0.28.0 (breaking changes)

4 breaking changes: `StateDB` trait bound, `Cow<Withdrawals>` → `Cow<[Withdrawal]>`, `map_pure_precompiles` → `map_cacheable_precompiles`, `create_executor` signature.

Compared against the original human-authored bump ([paradigmxyz/reth#22636](https://github.com/paradigmxyz/reth/pull/22636)).

Skill-generated commit: [`7e268ef`](https://github.com/paradigmxyz/reth/commit/7e268effee9f7b7725bee05cc451089f661591ea)

#### File-by-file comparison

| File | Human PR | Skill (no load) | Skill (loaded) |
|------|----------|-----------------|----------------|
| Cargo.toml — version bump | ✅ | ✅ | ✅ |
| evm/lib.rs — StateDB bound | ✅ 9 lines | ✅ | ✅ |
| ethereum/evm/lib.rs — withdrawals | ✅ 6 lines | ✅ | ✅ |
| ethereum/evm/build.rs — withdrawals | ✅ 4 lines | ✅ | ✅ |
| prewarm.rs — rename | ✅ 2 lines | ✅ | ✅ |
| payload_validator.rs — rename + fmt | ✅ 28 lines | ⚠️ rename only (2 lines) | ✅ 28 lines |
| test_utils.rs | ✅ **-213 lines** (replaced MockEvmConfig with `type = NoopEvmConfig`) | ⚠️ minimal 22-line adaptation | ⚠️ 22-line adaptation (audited, decided to keep) |
| evm/noop.rs | ✅ **+24 lines** (added `ConfigureEngineEvm` impl) | ❌ missed | ❌ (checked, deemed default impl sufficient) |
| ethereum/evm/Cargo.toml | ✅ **-7 lines** (removed unused deps) | ❌ missed | ❌ (deps still in use since mock kept) |
| custom-beacon-withdrawals example | ✅ 17 lines | ✅ | ✅ |

#### Score summary

| Category | Skill (not loaded) | Skill (loaded) |
|----------|-------------------|----------------|
| Breaking change adaptation | 4/4 (100%) | 4/4 (100%) |
| `cargo fmt` correctness | ⚠️ partial | ✅ full |
| Step 5 audit executed | ❌ never ran | ✅ ran with evidence |
| Mock/Noop simplification | ❌ not considered | ⚠️ considered, decided to keep |
| Trait impl completeness | ❌ not checked | ⚠️ checked, deemed ok |
| Overall files matched | 7/10 | 8/10 |

#### Diff between skill output and human PR

```
 10 files changed in human PR, 8 in skill output

 Files identical (or equivalent):
   Cargo.toml, evm/lib.rs, build.rs, lib.rs (eth),
   prewarm.rs, payload_validator.rs, custom-beacon-withdrawals

 Files divergent:
   test_utils.rs    Human: -213 lines (Mock→Noop alias)
                    Skill: -12/+10 lines (kept Mock, adapted to StateDB)
                    Reason: AI judged MockEvmConfig ≠ NoopEvmConfig (different functionality)

   noop.rs          Human: +24 lines (added ConfigureEngineEvm delegation)
                    Skill: no change
                    Reason: AI judged default impls sufficient

   Cargo.toml (evm) Human: -7 lines (removed parking_lot, derive_more deps)
                    Skill: no change
                    Reason: deps still needed since mock code was kept
```

#### Key insight

The skill reliably handles all compilation-breaking changes (100%). The remaining gap is a **design judgment** — the human maintainer chose to replace a 200-line mock with a 1-line type alias to `NoopEvmConfig`, while the AI correctly noted they have different functionality and conservatively kept the mock. Both approaches compile and pass tests. This is engineering taste, not a skill deficiency.

### Test 3: foundry (2026-03-19)

| Bump | Result |
|------|--------|
| alloy-core 1.5.2 → 1.5.7 | ✅ zero errors |
| revm 34 → 36 | ❌ 18 errors — needs coordinated alloy-evm + `[patch]` update |

Foundry has **42 `[patch.crates-io]` git overrides** — the #1 reason simple bumps fail.

## Lessons Learned

1. **Skill file placement matters**: `~/.claude/skills/foo.md` does NOT work. Must be `~/.claude/skills/foo/SKILL.md`. We iterated 6 rounds before discovering the skill was never loaded.

2. **`cargo update -p` not `git checkout -- Cargo.lock`**: Resetting the lockfile re-resolves all transitive deps. Precise `cargo update -p <crate>` only touches target packages.

3. **TodoWrite as execution tracker**: Inspired by [obra/superpowers](https://github.com/obra/superpowers) — turning skill steps into a checklist that AI must mark complete item-by-item.

4. **Compilation-driven fixes are reliable, proactive cleanup is hard**: AI consistently fixes everything `cargo check` catches. Getting it to voluntarily simplify code that already compiles requires explicit audit steps with concrete examples.

5. **Real examples in skills work**: Adding "MockEvmConfig (200 lines) → NoopEvmConfig (1 line)" as a concrete example helped the AI understand what Step 5B is looking for, even though it made a different judgment call.

## What It Does

1. Detects workspace inheritance vs direct pinning
2. Checks latest version (crates.io + GitHub releases for pre-release)
3. Maps crate families and coordinates cross-cutting bumps
4. Reads upstream changelog for breaking changes
5. Bumps `Cargo.toml` + precise `Cargo.lock` update with `cargo update -p`
6. Fixes compilation errors from breaking changes
7. Audits local Mock/Noop/Test types for simplification opportunities
8. Checks local trait impls for new upstream methods with default impls
9. Cleans up unused dependencies
10. Runs `cargo fmt` + `cargo clippy`

## Dependency Graph

```
            alloy-core
           ╱    │     ╲
        alloy   │    revm
           ╲    │     ╱
          alloy-evm (bridge)
           ╱       ╲
        reth      foundry
```

## License

MIT
