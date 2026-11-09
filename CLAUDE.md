# CLAUDE.md

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.

## What this is

`@sudobility/raidr_cli` (BUSL-1.1) — the `raidr` binary that turns an raidr
capture bundle into a runnable project, plus the `raidr-reconstruct` agent skill
(`skills/reconstruct/`) that drives it. The CLI does every deterministic stage;
the skill does only judgment work (components, naming, wiring).

Ships as TypeScript: `bin.raidr` is `./src/cli.ts`, `files` is `src` + `skills`.
There is **no build step and no `dist/`**; users need Bun ≥ 1.2 (`engines.bun`).

Published to **GitHub Packages** (`npm.pkg.github.com`), not public npm — the
`@sudobility` scope is pinned there in `.npmrc`; installing needs a GitHub token
with `read:packages`.

### Place in the raidr family

| Repo | Role |
|---|---|
| raidr_processor | Pure bundle format / redaction / analysis / codegen. **This repo's core dependency.** |
| raidr_extension | Chrome MV3 capture (produces the bundles this CLI reads) |
| **raidr_cli** | Reconstruction CLI + `raidr-reconstruct` skill (this repo) |
| raidr_crawler | Headless crawl + `raidr-publish` skill |
| raidr_types → raidr_client → raidr_lib → raidr_app | Catalog types, client, logic, UI |
| raidr_api | Catalog + hosted MCP |
| raidr_web | Landing page |

Nothing in the family depends on raidr_cli; its consumers are end users (`bun add
-g`), coding agents via the skill, and the MCP skills raidr_crawler generates
(which run `raidr token` through `bunx`).

## Commands

Verified on this checkout (Bun 1.4.2, 2026-11-08):

| Command | What it does | Result |
|---|---|---|
| `bun install --frozen-lockfile` | Install deps | pass |
| `bun run typecheck` | `tsc --noEmit` over src, tests, scripts, fixtures/api | pass |
| `bun run test:unit` | `bun test` on the listed dirs (excludes `tests/capture`) | pass — ~65 tests / 10 files without the network-bound `tests/roundTrip.test.ts`, which `test:unit` also runs |
| `bun test tests/capture` | Real-browser capture harness | **fails here**: needs Playwright Chromium + `fixtures:build` first |
| `bun run fixtures:build` | `bun install && bun run build` in each `fixtures/apps/*` | needs network; writes fixture `node_modules`/`dist` |
| `bun run fixtures:capture` | Regenerates `fixtures/bundles/*.zip` via Playwright | overwrites tracked fixtures; needs a browser |

No `lint`/`build` script. Plain `bun test` also runs `tests/capture` (needs a browser) — use `bun run test:unit`.

## Architecture

```
src/
  cli.ts                  bin entry; dispatches reconstruct | token | install | uninstall (lazy imports)
  commands/token.ts       `raidr token <apiHost>`: sign in in a headed browser, save the site token
  token/watcher.ts        CredentialWatcher: when a request's token counts as signed in
  commands/reconstruct.ts the pipeline: stages 01–07, mode choice, project + replay/proxy codegen, report.md
  commands/install.ts     symlinks skills/reconstruct into ~/.claude|.codex|.agents/skills/raidr-reconstruct
  bundle/load.ts          zip-or-dir file I/O → raidr_processor's readBundle/unzipBundle (LoadedBundle)
  stages/unpack.ts        stage 3: prettier-beautify JS chunks, split webpack modules without an AST
  stages/mirror.ts        stage 5b: write served bytes to public/, snapshot fallback for client routes
  emit.ts                 write files, prettier-format when possible (never fails on format errors)
  paths.ts                safeRelativePath: shortens path segments over 200 bytes (ENAMETOOLONG)
  introspect/probes.ts    page-evaluated probes — SHARED FILE, see Invariants
  capture/harness.ts      test/fixture-only Playwright capture → bundle zip (not used by the CLI)
skills/reconstruct/       SKILL.md (the agent procedure) + INSTALL.md
fixtures/apps/            react-sample, vue-sample (vite apps; dist/ and node_modules/ gitignored)
fixtures/api/server.ts    Hono fixture API on :8123 the sample apps call
fixtures/bundles/*.zip    committed capture bundles — the input for most tests
scripts/                  fixtures:build and fixtures:capture
```

### `raidr reconstruct <bundle> --out <dir> [--replay]` data flow

All analysis/codegen comes from `@sudobility/raidr_processor`; this repo sequences
it and does I/O. Artifacts go to `<dir>/.raidr/`:

| Stage | Writes | Notes |
|---|---|---|
| 1 | `01-bundle.json` | manifest, gaps, redaction table |
| 2 | `02-sources/` | original sources from source maps (`recoverBundleSources`) |
| 3 | `03-chunks/` | only when recovery ratio < 80 (`unpackChunks`) |
| 4 | `04-api-model.json`, `recordings.json` | XHR/Fetch only, OPTIONS excluded; keyed by `endpointKey` |
| 5 | `05-route-model.json` | runtime router table, or derived from Document requests (`source` field) |
| 5b | `06-mirror.json` + `<dir>/public/` | always written |
| 5c | `07-link-audit.json` | mirrored-page links that resolve to nothing |
| 6–7 | project files in `<dir>` | `generateProject` or `mirrorProject`; `src/api/{types,client}.ts`, `server/replay.ts`, `server/recordings.json`. `02-recovery.json` is written **after** 07 (mode depends on the mirror) |
| — | `report.md` | human summary; the skill starts here |

Mode: `recovery` if ratio ≥ 80 (`RECOVERY_THRESHOLD`), else `mirror` if any page
was mirrored, else `inference`. In mirror mode `server/replay.ts` is a live
**proxy** to the real origin by default; `--replay` switches to serving
`recordings.json`.

### `raidr token <apiHost> [--print]`

Gets a site's auth token without copying it from DevTools. Lives here (not in
raidr_crawler) so generated MCP skills never name the crawler; they call
`bunx --package @sudobility/raidr_cli raidr token <apiHost> --print`.

- **Target** from raidr_api `GET /api/v1/apis/<apiHost>` (via `apiKey` in
  `~/.raidr/config.json`; `--api-url`/`RAIDR_API_URL`/`apiUrl`/`https://api.raidr.app`),
  giving auth style, header/cookie name, `tokenPrefix`, loginUrl, and `auth:'user'`
  paths as `userPaths`. `--login <url> --style bearer|header|cookie`
  (+ `--header-name`/`--cookie-name`/`--user-path`) skips the doc; style `none` exits 0.
- **Browser**: Playwright `launchPersistentContext`, headed, profile
  `~/.raidr/browser` (`RAIDR_BROWSER_PROFILE`). Tries Chrome → Edge → Playwright
  Chromium; `--channel` pins one with no fallback.
- **Verification** (`CredentialWatcher`, same rule as raidr_extension's
  `TokenCapture`): token accepted once a request carrying it to a `userPaths` path
  answers 2xx (or any 2xx when `userPaths` is empty), then the browser closes.
  Window closed first / `--timeout-ms` (default 600000) → null → exit **2** (unless
  `userPaths` empty: last token returns unverified). `saveSiteToken` merges
  `siteTokens[apiHost]` into `~/.raidr/config.json` (mode 0600); `--print` writes
  the token alone to stdout, every message to stderr.

## Invariants (easy to break)

- **`src/introspect/probes.ts` must stay byte-identical** to
  `raidr_extension/src/introspect/probes.ts` (and raidr_crawler's copy). Enforced
  by `tests/introspect/probesParity.test.ts`, *skipped* when the sibling checkout
  is absent (i.e. in CI). Change all three together; don't reformat it here alone.
  Can't move into raidr_processor (DOM).
- **Probe functions are serialized with `.toString()`** and evaluated in the page
  (`PROBE_SOURCES`). They must not reference anything outside their own body.
- **raidr_processor must stay I/O-free**; anything needing `fs`/`path`/zip goes
  here (README "Why this is a separate repository").
- **`mirrorReplayServer` patches `generateReplayServer` output with exact-string
  `.replace()`**. If raidr_processor's `src/codegen/replay.ts` template text
  changes, these silently no-op. Only `--replay` mirror mode is affected, and no
  unit test covers it (round-trip tests use a recovery-mode bundle).
- **Never invent data.** Unrecorded endpoints return 501 `RAIDR-GAP` (asserted in
  `tests/roundTrip.test.ts`); gaps become `RAIDR-GAPS.md`.
- **Mirror safety**: paths with `..` are dropped; served bytes always win over DOM
  snapshots (`seen` set in `stages/mirror.ts`).
- **`install()` takes `home` as a parameter** — `os.homedir()` ignores runtime
  `HOME` changes, so tests would write into the real home. It refuses to delete a
  non-symlink at the target.
- **Skill text is tested** (`tests/skills/skillFormat.test.ts`): frontmatter
  (`name` letters/digits/hyphens, `description` starts with "Use when", no
  " then/first/step/stage ", < 1024 chars), every `.raidr/` artifact name, the
  four completion-report headings, no `jq`, and INSTALL.md commands/paths.
- **`raidr token` must not accept a guest token.** Keep the verification rule
  identical to raidr_extension's `src/background/tokenCapture.ts` (both use
  raidr_types `extractCredential` + `matchesPathTemplate`).
- **`raidr token --print` keeps stdout to the token alone.** Skills capture it
  with `$(…)`; everything else is stderr.
- Redaction is raidr_processor's job (it deliberately leaves `x-api-key` alone);
  this repo never re-redacts.

## Testing

- `tests/` mirrors `src/`. `tests/roundTrip.test.ts` reconstructs
  `fixtures/bundles/{react,vue}-sample.zip` into `.tmp/`, runs `bun install`,
  typecheck, build, and boots the replay server on ports 8899/8900 — needs
  network and takes most of the run.
- Tests write to `.tmp/` (gitignored); leave it alone, tests recreate it.
  `tests/bundle/fixtures/badversion/` is a manifest the loader must reject.
- CI (`.github/workflows/ci-cd.yml` → `johnqh/workflows` unified workflow,
  `publish-target: github`) runs typecheck, lint if present (none), `test:unit`,
  build if present (none). CI **cannot** run the probes parity check (no sibling)
  or `tests/capture` (needs Chromium + built fixtures).

## Common changes

- **New bundle field / change redaction**: both start in raidr_processor — add
  the field (or edit `src/redaction/`), publish, bump `@sudobility/raidr_processor`
  here, then consume in `src/commands/reconstruct.ts` (and `src/capture/harness.ts`
  if the harness should emit it). Never redact here.
- **Add a reconstruct stage/artifact**: `src/stages/<name>.ts` → call from
  `reconstruct()` + `writeJson('NN-name.json')` → add to `report.md`, SKILL.md
  Quick Reference, and the artifact list in `skillFormat.test.ts` → `tests/stages/`.
- **Change what counts as a site token**: `extractCredential` is in raidr_types
  (shared). The signed-in rule is `src/token/watcher.ts` here and `tokenCapture.ts`
  in the extension — change both. Tests: `tests/token/`.
- **Add a CLI command**: `src/commands/<name>.ts` exporting `run<Name>(argv)` → a
  `case` + usage string in `src/cli.ts` → `tests/commands/` → README Usage.
- **Change the skill**: edit `skills/reconstruct/SKILL.md` / `INSTALL.md` → `bun
  run test:unit` → keep the frontmatter description a trigger, not a procedure.

## Versioning and publishing

CI publishes when `package.json` `version` is new on push to `main`. Never
publish by hand or bump versions unless asked; releases go through
`raidr_app/scripts/push_all.sh` after raidr_processor.

## Gotchas

- `src/cli.ts`'s top-level usage omits `--replay` (the `runReconstruct`
  subcommand usage has it); `reconstruct()` does not clear `--out`, so stale files
  from a previous run survive.
- `emitFiles` swallows prettier errors on purpose (unformatted output is not a
  bug); `recoveryRatio` is an integer percent (0–100), not a fraction.
- `playwright` is a runtime dependency because `raidr token` launches a browser;
  with no Chrome/Edge it needs `bunx playwright install chromium`. With
  `--login/--style`, pass `--token-prefix` when a `header`-style site prefixes the
  token (otherwise `header` strips nothing, `bearer` strips `Bearer `).
