---
name: vite-plus
description: Use Vite+ (vp) as the frontend toolchain for the Family Money Manager repo - the Astro/Vite conflict that decides the split, the vite package-manager override hazard, and the three places in this repo that must NOT be touched by it (the root package.json test recursion trap, and the zero-dependency check_invariants_ts suite). Use when scaffolding or working on the frontend workspace, adding lint/format/test/task tooling, or when vp is already installed and you need to know what it owns.
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, WebFetch
---

# Vite+ for the FMM frontend

**Vite+** (`vp`) is VoidZero's unified toolchain: one CLI over **Vite, Vitest, Oxlint, Oxfmt, Rolldown, tsdown and Vite Task**. One `vite.config.ts` configures dev server, build, tests, lint, format and monorepo tasks.

MIT licensed. Docs: <https://viteplus.dev>. Source: <https://github.com/voidzero-dev/vite-plus>

## The decision this repo has to make first

**Vite+ is a toolchain. Astro is a framework. They are different layers, and this repo's locked stack is
Astro.** `.plan/core_brain/plan.md` §3 is one of sixteen decisions the user settled by explicit choice:
`frontend` = *AWS Blocks Hosting + Astro + React + a hand-rolled `@ag-ui/client`*.

**Vite+ cannot replace Astro** — it has no routing, no islands, no SSG. So the question is only whether
it may *sit underneath* Astro, and the answer is **yes for most of it and no for two commands.**

| Vite+ owns | Astro must keep |
|---|---|
| `vp check` · `vp lint` · `vp fmt` · `vp test` | `astro dev` · `astro build` — **the app build** |
| `vp install` · `vp env` · `vp run` · `vp staged` | `astro.config.mjs` — the framework config |
| `vp toolchain` · `vp add/rm/up/outdated` | |

**Never run `vp dev` or `vp build` to build the Astro app.** Those invoke Vite directly against
`vite.config.ts`, which is *not* how Astro compiles. See `references/astro-conflict.md` for why this
split is forced rather than stylistic.

## `vp` is already installed here, and it has taken over Node

```
vp -> C:\Users\julia\AppData\Local\vite-plus\bin\vp.exe     vite-plus@1.1.0
node -> C:\Users\julia\AppData\Local\vite-plus\bin\node.exe v24.21.0
npm  -> ...\npm.exe     pnpm -> ...\pnpm.exe     bun -> ...\bun.exe
```

⚠️ **`vp env` manages Node and the package managers globally.** `node`, `npm`, `pnpm` and `bun` on
`PATH` all resolve into Vite+'s bin directory. **Every Node command in this repo already runs under
Vite+'s runtime** whether or not any Vite+ config exists. That is why the repo's `node --test` suite
works, and why removing Vite+ would be a much larger change than adding a config file.

## Three places `vp` must NOT be pointed

These are the failure modes that would cost a session.

### 1. `vp run test` at the repo root — infinite recursion

The root `package.json` is:

```json
"scripts": { "test": "npm run test" }
```

**`npm test` at the root hangs forever**, and `vp run test` runs that same script. `vp test` is safe
(it invokes Vitest, not the script) but **`vp run test` is not.** The real commands are in
`.plan/core_brain/check_invariants_ts/` and are run with plain `node`.

### 2. `.plan/core_brain/check_invariants_ts/` — zero dependencies, on purpose

```
"dependencies": {}, "devDependencies": {}      engines: node >= 22.6
```

Its 275 tests import **`node:test`, `node:assert/strict`, `node:child_process`, `node:fs`,
`node:os`, `node:path`** and nothing else, and every import carries a **`.ts` extension**
(`"../src/actions.ts"`) because it runs on **Node's native type stripping**, with no build step.

⚠️ **Pointing `vp test` at it would break all 275 tests.** Vitest's `test`/`describe`/`expect` are not
`node:test`'s, and `node:assert/strict` is not `expect`. This suite is **32/32 mutation-tested and
part of the repo's verification gate** — it is the thing that proves the validators can reject
something. Leave it on `node --test`.

**This is not a Vite+ project and should never become one.** Zero-dependency is the design, not an
accident of timing.

### 3. The `vite` package-manager override — read before you add it

The documented manual install adds overrides so other packages use Vite+'s copies:

```jsonc
"overrides": { "vite": "npm:@voidzero-dev/vite-plus-core@<version>", "vitest": "<version>" }
```

⚠️ **In this repo that override swaps the Vite that Astro depends on.** Astro bundles Vite as its own
compiler; aliasing `vite` to `@voidzero-dev/vite-plus-core` is a change to Astro's build, not just to
the toolchain. **Prefer `vp install -D vite-plus` and the standard plugin approach, and treat the
override as an experiment requiring a green Astro build** — never as routine setup.

## Commands worth knowing

```
vp toolchain                 # versions of vp, vite, rolldown, oxc, tsdown — read this first
vp check                     # fmt + lint + typecheck in one pass          (the main one)
vp check --fix               # same, writing fixes
vp test                      # Vitest
vp run <task>                # package.json scripts + monorepo tasks, cached
vp staged                    # linters on staged files — needs `staged: { '*': 'vp check --fix' }`
vp migrate                   # merge .oxlintrc / .oxfmtrc / lint-staged into vite.config.ts
```

## Settling the toolchain in this repo

**There is no frontend code.** `AGENTS.md`: *"The three npm workspaces are planned in `plan.md` §3 but
do not exist on disk. Do not scaffold them; there is no app code in this repo."* So nothing should be
migrated yet — `vp migrate` has nothing to migrate, and `vp create` would create app code the repo has
explicitly refused to create.

**When the frontend workspace is built,** the shape is:

- `vite.config.ts` at the **workspace root** — Vite+ requires exactly this, at the root, and VoidZero
  has explicitly declined to support alternative config locations.
- `astro.config.mjs` alongside it, untouched. Astro does not read `vite.config.ts`.
- App scripts stay `astro dev` / `astro build` / `astro preview`; `vp` is invoked for `check`,
  `test`, `fmt`, `lint` and task running.
- `vp staged` is the natural pre-commit hook — this repo has **no CI and no build step**, so a
  `staged` hook is currently the *only* automated gate available.

## Sources

| Fact | Where |
|---|---|
| Astro/Vite config conflict, and VoidZero's refusal to fix it | [issue #912](https://github.com/voidzero-dev/vite-plus/issues/912), closed **as a refusal** by cpojer Mar 2026, re-litigated through Aug 2026 |
| `vp` command surface and `vite.config.ts` shape | [vite-plus README](https://github.com/voidzero-dev/vite-plus#readme) |
| Override + `pnpm`/`yarn` variants | README § *Manual Installation & Migration* |
| CI | [`voidzero-dev/setup-vp`](https://github.com/voidzero-dev/setup-vp) — pin an exact version, never `v1` |
