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

| File | Human PR | No skill (baseline) | With skill |
|------|----------|---------------------|------------|
| Cargo.toml — version bump | ✅ | ✅ | ✅ |
| evm/lib.rs — StateDB bound | ✅ 9 lines | ✅ (different approach: added StateDB import + changed trait bounds directly) | ✅ |
| ethereum/evm/lib.rs — withdrawals | ✅ 6 lines | ✅ | ✅ |
| ethereum/evm/build.rs — withdrawals | ✅ 4 lines | ⚠️ slightly different (extra lines) | ✅ |
| prewarm.rs — rename | ✅ 2 lines | ✅ | ✅ |
| payload_validator.rs — rename + fmt | ✅ 28 lines (rename + closure reformat) | ⚠️ 2 lines (rename only, no fmt reformat) | ✅ 28 lines (rename + reformat) |
| test_utils.rs | ✅ **-213 lines** (replaced 200-line MockEvmConfig with 1-line `type = NoopEvmConfig`) | ⚠️ 25-line minimal adaptation (kept full mock) | ⚠️ 22-line adaptation (audited Mock vs Noop, decided to keep) |
| evm/noop.rs | ✅ **+24 lines** (added `ConfigureEngineEvm` impl) | ❌ missed entirely | ❌ (checked trait impls, deemed default impl sufficient) |
| ethereum/evm/Cargo.toml | ✅ **-7 lines** (removed parking_lot, derive_more) | ❌ missed entirely | ❌ (deps still in use since mock was kept) |
| custom-beacon-withdrawals example | ✅ 17 lines | ✅ | ✅ |

#### Score summary

| Category | No skill (baseline) | With skill |
|----------|---------------------|------------|
| Breaking change adaptation | 4/4 (100%) | 4/4 (100%) |
| `cargo fmt` correctness | ⚠️ partial (missed closure reformat) | ✅ full |
| Post-bump audit executed | ❌ never ran | ✅ ran Step 5A/5B/5C with evidence |
| Mock/Noop simplification | ❌ not considered | ⚠️ considered, decided to keep |
| Trait impl completeness | ❌ not checked | ⚠️ checked, deemed default impls ok |
| Overall files matched | 7/10 | 8/10 |

#### Diff between skill output ([`7e268ef`](https://github.com/paradigmxyz/reth/commit/7e268effee9f7b7725bee05cc451089f661591ea)) and human PR ([`0df9791`](https://github.com/paradigmxyz/reth/commit/0df9791bea))

```
 Human PR: 10 files, +62 -256 (net -194 lines)
 Skill:     8 files, +42 -40  (net +2 lines)

 Files identical or equivalent (7):
   Cargo.toml, evm/lib.rs, ethereum/evm/lib.rs, build.rs,
   prewarm.rs, payload_validator.rs, custom-beacon-withdrawals

 Files divergent (3):
   test_utils.rs    Human: -213 lines (replaced MockEvmConfig with type alias to NoopEvmConfig)
                    Skill: +10 -12 lines (kept MockEvmConfig, adapted to new StateDB bounds)
                    Why different: AI audited both types, judged Mock ≠ Noop (different functionality).
                    MockEvmConfig provides mock execution results via Arc<Mutex<Vec<ExecutionOutcome>>>,
                    while NoopEvmConfig panics on every call. Conservative but defensible.

   noop.rs          Human: +24 lines (added ConfigureEngineEvm trait impl with 3 delegation methods)
                    Skill: no change
                    Why different: AI checked trait methods, noted all have default impls.
                    Human author added explicit impls for completeness; AI deemed defaults sufficient.

   Cargo.toml (evm) Human: -7 lines (removed parking_lot, derive_more from test-utils feature)
                    Skill: no change
                    Why different: Cascading from test_utils.rs — since mock code was kept,
                    its dependencies are still needed.
```

#### Key insight

The skill reliably handles all compilation-breaking changes (100%). The remaining 3-file gap stems from a **design judgment**: the human maintainer replaced a 200-line mock with a 1-line type alias, while the AI correctly noted they have different runtime behavior and conservatively kept the mock. Both approaches compile, pass tests, and are defensible engineering choices.

The real difference the skill makes is not in the final file count, but in the **process quality**: without the skill, the AI never considers whether Mock types could be simplified. With the skill, it explicitly audits them, makes a reasoned decision, and documents why.

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
