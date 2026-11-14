# The Astro / Vite+ conflict, and why the split is forced

## The short version

**Vite+ requires a root `vite.config.ts`. Astro wraps Vite and discourages a separate Vite config.
VoidZero has explicitly declined to resolve this.** So the two cannot both be in charge of the build,
and someone has to yield. In this repo, **Astro yields on the app build and Vite+ takes everything
else.**

## What issue #912 actually says

[voidzero-dev/vite-plus#912](https://github.com/voidzero-dev/vite-plus/issues/912), *"Use
`nuxt.config.ts` / `astro.config.mjs` as Vite+ config for Nuxt/Astro"*, opened 15 Mar 2026:

> *"Right now, a `vite.config.ts` is required. Nuxt and Astro are the two meta frameworks that do work
> slightly different as they **wrap** the Vite config and discourage using a separate one."*

Closed 23 Mar 2026 — by **cpojer (VoidZero member), as a refusal**:

> *"At this time, we are **not planning on supporting other configuration file names or systems**
> because it would raise complexity across various tools, require resolution rules (what if both are
> present?), and come with mental overhead to know where to look for humans and AI, especially in
> larger monorepos. If Nuxt or Astro want to simplify and use `vite.config.ts`, they could support
> something like `export const nuxtConfig = …` in `vite.config.ts`. **I'll close this issue, but feel
> free to keep discussing it.**"*

**The issue state is `closed: completed`, but the resolution is a decision not to build the feature.**
That distinction matters: reading the state alone suggests support exists. It does not.

### Re-litigated, still unresolved

- **7 Apr 2026** — tobiasdiez (Nuxt core) reports Nuxt now *warns*:
  `[nuxt] WARN Using vite.config.ts is not supported together with Nuxt. Use options.vite instead.`
  and proposes `vp.config.ts` or a generated `.nuxt/vite.config.ts`.
- **23–24 Jul 2026** — danielroe re-asks about the generated path; cpojer: *"Generating is fine of
  course, but I don't think we are in a position to support a nested folder for the base of a vite
  config file."*
- **12 Aug 2026** — still open in the thread, no maintainer answer.

## Support status, stated precisely

| Framework | Status |
|---|---|
| React, Vue, Svelte, Solid, Angular | Officially supported |
| Qwik | Experimental |
| **Astro, Nuxt** | **Works, but the awkward case** — the two frameworks Vite+ cannot cleanly configure |

VoidZero's own announcement says Vite+ *"ships across Vue, React, Svelte, Angular, Solid, Qwik, and
Astro. Vite+ is stable, but not yet complete."* Third-party review echoes it: *"Void officially
supports React, Vue, Svelte, and Solid, integration with Astro and Qwik is still experimental."*

**So: not broken, but not smooth.** That is a materially different claim from "unsupported," and the
right response to it is a **split**, not an abandonment.

## The three resolutions, and why this one

### (a) Vite+ replaces Astro entirely

**Rejected — it isn't possible.** Vite+ is a toolchain with no routing, no islands, no SSG, no
content collections. Astro is a framework. Dropping Astro means re-deciding
`.design/frontend/02-surfaces.md`'s *"Astro — static, no hydration" for `/signup`, `/confirm` and
`/lobby`*, plus *"ONE React island"* for `/`. That architecture maps onto Astro's island model almost
exactly, and discarding it costs a locked decision to gain nothing.

### (b) Astro keeps the build, Vite+ takes the toolchain ← **chosen**

| Command | Owner | Why |
|---|---|---|
| `astro dev` / `astro build` / `astro preview` | **Astro** | Astro is the only thing that knows how to compile an island |
| `vp check` / `lint` / `fmt` / `test` | **Vite+** | These never touch Astro's Vite config |
| `vp install` / `env` / `run` / `staged` | **Vite+** | Package-manager and task concerns, orthogonal to the framework |
| `vite.config.ts` | **Vite+** | Required at the root, and VoidZero will not move it |

**Why this works:** Oxlint, Oxfmt, Vitest and Vite Task do not read the bundler configuration. They are
happy beside an Astro project. The conflict is confined to `dev` and `build`, and those two stay with
Astro.

### (c) The `vite` override — Vite+ everywhere

```jsonc
"overrides": { "vite": "npm:@voidzero-dev/vite-plus-core@1.1.0" }
```

⚠️ **Rejected as routine setup.** This aliases `vite` **globally in the dependency graph** — which
includes Astro's own compiler. It is not a toolchain change; it is a change to how Astro builds.
Documented as the path for manual installs, so it will appear in tutorials, and it is the thing most
likely to break silently.

**If it is ever tried here, the bar is a green `astro build` plus the full validation suite — not "it
compiled".**

## Verification, when the frontend exists

None of this has been run in this repo, because **there is no frontend code**. `AGENTS.md`: *"Do not
scaffold them; there is no app code in this repo."* So everything above is documentation-grade, not
tested-grade, and the skill says so.

**The check to run first, whenever that changes** — cheap, and it answers the only question that
matters:

```bash
vp migrate            # merges .oxlintrc / .oxfmtrc / lint-staged into vite.config.ts
vp check              # does Oxlint + Oxfmt + typecheck coexist with astro.config.mjs?
astro build           # does Astro still compile with a vite.config.ts sitting beside it?
```

If `vp check` works and `astro build` is unchanged, the split holds. If `astro build` changes
behaviour, **stop** — that is the override leaking, not a tuning problem.

## A note on what "same Vite" even means

Vite+ bundles its own `vite@8.3.3` inside `@voidzero-dev/vite-plus-core`. Astro has its own Vite
dependency. **Two Vite copies can legitimately coexist** — Vite+ uses its copy for
`vp check`/`vp test`/`vp run`, Astro uses its own for compiling the app. That is not a duplicate
dependency problem to be solved; it is the shape of the chosen split.

⚠️ **It becomes a problem only if you add the `vite` override**, which collapses the two into one and
forces Vite+'s copy onto Astro.
