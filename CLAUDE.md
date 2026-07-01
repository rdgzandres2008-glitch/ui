# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is the **shadcn/ui** monorepo. It publishes two independent npm packages and hosts the documentation website plus the component registry that powers the CLI.

- **pnpm** workspaces + **Turborepo** for builds, **Changesets** for releases.
- Node `>=20.18.1` (`.nvmrc` pins `v20.5.1`), package manager `pnpm@10.33.4`.
- The registry build step uses **Bun** (`bun run ./scripts/build-registry.mts`).

## Workspaces

| Path              | Name            | What it is                                                                 |
| ----------------- | --------------- | ------------------------------------------------------------------------- |
| `packages/shadcn` | `shadcn`        | The CLI (`init`, `add`, `apply`, `build`, `mcp`, `preset`, `registry`, …). Published to npm. |
| `packages/react`  | `@shadcn/react` | Headless/unstyled React primitives (e.g. `message-scroller`). Published to npm. |
| `apps/v4`         | `v4`            | The Next.js docs site (`ui.shadcn.com`) **and** the source-of-truth component registry. Not published. |
| `packages/tests`  | `tests`         | Integration tests that run the real CLI against fixtures. Not published.   |
| `templates/*`     | —               | Starter templates the CLI scaffolds (`next-app`, `vite-app`, `astro-app`, monorepo variants, …). |

The two published packages **version on their own lines** — a change to one never bumps the other unless a changeset says so.

## Common commands

Run from the repo root unless noted. Turbo fans tasks out across workspaces; use `--filter=<workspace>` to scope.

```bash
pnpm install              # install all workspace deps

pnpm dev                  # run every workspace's dev task
pnpm --filter=v4 dev      # run only the docs site on http://localhost:4000
pnpm --filter=shadcn dev  # rebuild the CLI on change (tsup --watch)

pnpm build                # build everything
pnpm build:packages       # build ONLY packages/* (shadcn + @shadcn/react), never apps/v4

pnpm check                # lint + typecheck + format:check (the pre-commit gate)
pnpm lint                 # eslint across workspaces
pnpm lint:fix
pnpm typecheck            # tsc --noEmit across workspaces
pnpm format:write         # prettier write
```

### Running the CLI locally against the local registry

```bash
pnpm dev            # terminal 1: starts the docs site + serves the registry at :4000
pnpm shadcn         # terminal 2: runs the built CLI with REGISTRY_URL=http://localhost:4000/r
pnpm shadcn add button -c ~/path/to/test-app   # test against a scratch project
```

`pnpm shadcn` maps to `packages/shadcn`'s `start:dev`, which points `REGISTRY_URL` at localhost and `SHADCN_TEMPLATE_DIR` at `../../templates`. Use `pnpm shadcn:prod` to hit the live registry instead.

## Testing

There are three distinct test layers — know which one you need:

- **CLI unit tests** (`packages/shadcn`, colocated `*.test.ts`, MSW-mocked):
  ```bash
  pnpm --filter=shadcn test                        # all CLI unit tests
  pnpm --filter=shadcn exec vitest run src/commands/add.test.ts   # a single file
  pnpm --filter=shadcn exec vitest -t "some test name"            # a single test by name
  ```
- **CLI integration tests** (`packages/tests`, runs the real CLI against fixtures — needs the registry running):
  ```bash
  pnpm test        # root: builds the registry, boots v4 on :4000, then runs integration tests
  ```
  `pnpm test` is the full end-to-end suite; it does `registry:build` then `start-server-and-test`. The `tests:test` script referenced in `packages/tests/README.md` is stale — use the root `pnpm test`.
- **`@shadcn/react` tests** (`packages/react`): `pnpm --filter=@shadcn/react test`, plus `test:browser` for the Vitest browser (Playwright) runner.

## The registry (most important architecture to understand)

`apps/v4/registry` is the **source of truth** for every component the CLI can install. See `apps/v4/registry/README.md` and `apps/v4/registry/bases/README.md` for the full model. The key mental model:

- **Authored by hand** (edit these):
  - `registry/bases/base/` and `registry/bases/radix/` — two parallel base registries (Base UI vs Radix), each with `ui/`, `lib/`, `hooks/`, `blocks/`, `examples/`, `internal/`.
  - `registry/styles/style-*.css` — design-token files per style (`nova`, `vega`, `sera`, …).
  - `registry/new-york-v4/` — the legacy source registry, authored and committed directly.
  - `apps/v4/examples/{base,radix}` — component demos.
- **Generated (do NOT hand-edit)**: `__index__.tsx`, `__blocks__.json`, per-combination folders like `base-nova/`, `apps/v4/styles/<combo>/ui`, and the installable JSON under `apps/v4/public/r/`.

Every base × style produces a **combination** (`base-nova`, `radix-sera`, …), generated from the authored bases plus the style CSS.

### Building the registry

```bash
# From apps/v4 (or `pnpm registry:build` from root):
pnpm registry:build                 # canonical full build — formats output; run before committing
pnpm registry:build --style base-nova   # fast targeted build of one combination (skips formatting)
pnpm registry:build --indexes            # just the runtime indexes
pnpm registry:build --examples           # just examples/__index__.tsx
```

Targeted builds skip formatting for speed and can leave large-but-harmless diffs. **Always run the full `pnpm registry:build` before committing** so generated output is canonicalized. The root `pnpm registry:build` also runs `lint:fix` and `format:write` afterward.

### Base ↔ Radix parity (enforced convention — see `.cursor/rules/registry-bases-parity.mdc`)

`bases/base` and `bases/radix` are parallel trees. **Any behavioral or visual change to a file under one must be mirrored in the matching path under the other**, diverging only where the underlying APIs differ (import paths, Base UI vs Radix props). Do not update only one side unless the user explicitly asks for a single-base change, and confirm both trees were updated when done.

When adding or modifying components: make the change for every style, update the docs (`apps/v4/content/docs`, MDX), and run `pnpm registry:build`.

## CLI package (`packages/shadcn`) layout

- `src/index.ts` — commander entry; registers all subcommands.
- `src/commands/*` — one file per command (`add`, `apply`, `build`, `diff`, `docs`, `eject`, `info`, `init`, `mcp`, `migrate`, `preset`, `search`, `view`, plus `commands/registry/*`).
- `src/preflights/*` — pre-run validation for each command.
- `src/migrations/*` — codemods (`migrate-icons`, `migrate-radix`, `migrate-rtl`).
- `src/registry/`, `src/schema/`, `src/preset/`, `src/mcp/`, `src/utils/` — supporting subsystems. `src/preset/` decodes/resolves the base62 preset codes; never build or decode preset URLs by hand.
- Built with **tsup**; multiple entry points are exposed via `package.json#exports` (`./registry`, `./schema`, `./mcp`, `./utils`, `./icons`, `./preset`).

## Conventions

- **Commits follow Conventional Commits**, enforced by commitlint: `category(scope): message` with categories `feat`, `fix`, `refactor`, `docs`, `build`, `test`, `ci`, `chore` (e.g. `feat(components): add new prop to avatar`). Commits must be signed (see `signed-commits.yml`).
- **Every publishable change needs a changeset**: run `pnpm changeset`, select the affected package(s) and bump. A PR with no changeset publishes nothing. See `RELEASING.md` for stable releases, per-PR snapshots (`release: beta` / `release: rc` labels), and prerelease trains.
- **Formatting**: Prettier with `@ianvs/prettier-plugin-sort-imports` and `prettier-plugin-tailwindcss`. No semicolons, double quotes, 2-space indent, `es5` trailing commas. `apps/v4` defines a custom import order and treats `cn`/`cva` as Tailwind functions — respect it rather than reordering imports by hand.
- CI runs on every PR: `code-check.yml` (lint/typecheck/format), `test.yml`, `browser-tests.yml`, `validate-registries.yml`, `templates.yml`. Match them locally with `pnpm check` and the relevant test command before pushing.

## Working with shadcn components as a consumer

When *using* shadcn/ui components (as opposed to developing this repo), the `skills/shadcn/` skill is the authoritative guide — `SKILL.md` plus `rules/{styling,forms,composition,icons,chat,base-vs-radix}.md`, `cli.md`, `registry.md`, and `customization.md`. It covers the critical composition/styling rules (e.g. `FieldGroup`+`Field` for forms, `data-icon` for button icons, semantic color tokens, `gap-*` over `space-*`). Consult it before hand-rolling component markup.
