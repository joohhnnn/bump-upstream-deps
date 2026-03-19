# bump-upstream-deps

A Claude Code skill for bumping upstream Rust crate dependencies across the Ethereum ecosystem (reth, foundry).

## Problem

When `alloy`, `revm`, or `alloy-evm` release new versions, downstream repos need manual updates. Simple version bumps fail because of `[patch]` git overrides, cross-repo compatibility, and breaking API changes. Dependabot/Renovate can't handle this — AI can.

## Install

```bash
mkdir -p ~/.claude/skills/bump-upstream-deps
cp bump-upstream-deps.md ~/.claude/skills/bump-upstream-deps/SKILL.md
```

> **Important**: A bare `.md` in `~/.claude/skills/` will NOT load. Must be `<name>/SKILL.md`.

## Validated Results

### Test 1: reth — revm 35→36 + alloy-evm 0.29.1→0.29.2 (no breaking changes)

| Metric | No skill | With skill |
|--------|----------|------------|
| Cargo.toml | ✅ | ✅ |
| Cargo.lock | ❌ 32 unrelated transitive dep changes | ✅ exact — only 11 target crates |

Skill uses `cargo update -p <crate>` instead of resetting the entire lockfile.

### Test 2: reth — alloy-evm 0.27.2→0.28.0 (breaking changes)

4 breaking changes: `StateDB` trait bound, `Cow<Withdrawals>` → `Cow<[Withdrawal]>`, `map_pure_precompiles` rename, `create_executor` signature.

Compared against human-authored bump [paradigmxyz/reth#22636](https://github.com/paradigmxyz/reth/pull/22636). Skill-generated commit: [`7e268ef`](https://github.com/paradigmxyz/reth/commit/7e268effee9f7b7725bee05cc451089f661591ea).

| File | Human PR | No skill | With skill |
|------|----------|----------|------------|
| Cargo.toml | ✅ | ✅ | ✅ |
| evm/lib.rs — StateDB | ✅ | ✅ | ✅ |
| ethereum/evm/lib.rs — withdrawals | ✅ | ✅ | ✅ |
| ethereum/evm/build.rs — withdrawals | ✅ | ⚠️ extra lines | ✅ |
| prewarm.rs — rename | ✅ | ✅ | ✅ |
| payload_validator.rs — rename + fmt | ✅ 28 lines | ⚠️ rename only, no fmt | ✅ 28 lines |
| test_utils.rs | ✅ **-213 lines** (Mock→Noop alias) | ⚠️ 25-line adaptation | ⚠️ 22-line adaptation (audited, kept) |
| evm/noop.rs | ✅ **+24 lines** | ❌ missed | ❌ checked, deemed default impl ok |
| ethereum/evm/Cargo.toml | ✅ **-7 lines** | ❌ missed | ❌ deps still needed |
| custom-beacon-withdrawals | ✅ | ✅ | ✅ |

| Category | No skill | With skill |
|----------|----------|------------|
| Breaking change adaptation | 4/4 | 4/4 |
| `cargo fmt` correctness | ⚠️ partial | ✅ |
| Post-bump audit | ❌ never ran | ✅ with evidence |
| Mock/Noop simplification | ❌ not considered | ⚠️ considered, kept |
| Files matched | 7/10 | 8/10 |

#### Diff: skill vs human PR

```
Human PR: 10 files, +62 -256 (net -194 lines)
Skill:     8 files, +42 -40  (net +2 lines)

Identical (7): Cargo.toml, evm/lib.rs, ethereum/evm/lib.rs, build.rs,
               prewarm.rs, payload_validator.rs, custom-beacon-withdrawals

Divergent (3):
  test_utils.rs     Human: -213 lines (Mock → type alias to Noop)
                    Skill: +10 -12 (kept Mock, adapted to StateDB)
                    AI judged Mock ≠ Noop — different runtime behavior.

  noop.rs           Human: +24 lines (ConfigureEngineEvm delegation)
                    Skill: no change — deemed default impls sufficient.

  Cargo.toml (evm)  Human: -7 lines (removed parking_lot, derive_more)
                    Skill: no change — deps still needed since mock kept.
```

The 3-file gap is a design judgment, not a capability gap. Human chose aggressive simplification; AI chose conservative correctness. Both compile and pass tests.

## Lessons Learned

- Skill files must be `<name>/SKILL.md`, not `<name>.md` — we iterated 6 rounds before finding this.
- `cargo update -p <crate>` instead of resetting `Cargo.lock` avoids unrelated transitive dep noise.
- TodoWrite checklists (inspired by [obra/superpowers](https://github.com/obra/superpowers)) turn skill steps into trackable tasks.
- AI reliably fixes compilation errors; simplifying code that already compiles needs concrete examples in the skill.

## License

MIT
