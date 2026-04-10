# CAP Upgrades — 45-Minute Session Outline

## 1. Why Upgrading Matters (~5 min)
- CAP follows a monthly release cadence — staying current means access to new features, bug fixes, and security patches
- Falling behind makes upgrades harder (multi-major jumps)
- Distinction between minor/patch upgrades (routine, safe) vs. major upgrades (requires review)

---

## 2. System Landscapes — Dev → Test → Production (~7 min)

### The typical landscape
- **Local / dev:** SQLite in-memory, `cds watch`, no auth, rapid iteration
- **Test / QA:** Cloud-hosted (CF or Kyma), real HANA or Postgres, full auth stack, mirrors production config
- **Production:** Same artefacts as test — only config/credentials differ, never code

### The golden rule: build once, promote the artefact
- `cds build --production` produces a deterministic build in `gen/`
- The same `.mtar` or container image travels from test to production — you do not rebuild per environment
- Environment-specific values (credentials, URLs, scaling) are injected externally, not baked in

### Environment-specific config without touching `mta.yaml`
- **CDS profiles** in `package.json` — `[development]` / `[production]` / custom profiles
  - e.g. `db.kind = sqlite` in development, `db.kind = hana` in production
- **MTA extension descriptors** (`.mtaext`) — overlay environment-specific parameters at deploy time
  - `cds up --overlay .deploy/eu10-prod.mtaext` (cds 9+)
  - Use separate `.mtaext` files per landscape: `dev.mtaext`, `test.mtaext`, `prod.mtaext`
- **`cds env --profile production`** — inspect the effective config that will apply in a given profile

### Rolling out a CAP upgrade stage by stage
1. **Upgrade locally** — bump versions, fix breaking changes, run `cds watch` and `npm test`
2. **Deploy to test** — `cds up --to cf` targeting the test subaccount; run integration/smoke tests
3. **Validate DB schema** — check that HDI delta or Postgres migration applied cleanly; redeploy if needed
4. **Deploy to production** — promote the same artefact (`.mtar` or image); no rebuild
5. **Post-deploy check** — `cds version` on the running app, smoke test critical flows, monitor logs

### What can go wrong when skipping a stage
- Breaking changes only surface with production-volume data or auth config differences
- Schema migrations that worked on clean test data can fail on production data (column widths, NULLs)
- Plugin-added internal tables (outbox, event queues) need to exist before the app boots in production

### CI/CD automation
- `cds add github-actions` (or `cds add gha`) scaffolds staging + production workflows out of the box
- Staging pipeline: triggered on push to `main` — deploys to test, runs tests
- Release pipeline: triggered on a GitHub Release — promotes to production
- Keep `@sap/cds-dk` version pinned in CI the same as local dev to avoid build surprises

---

## 3. Understanding the CAP Package Landscape (~5 min)
- Core packages: `@sap/cds`, `@sap/cds-dk` (CLI), `@sap/cds-compiler`
- Database plugins: `@cap-js/sqlite`, `@cap-js/hana`, `@cap-js/postgres`
- Optional capability plugins: `@cap-js/audit-logging`, `@cap-js/change-tracking`, `@cap-js/notifications`
- `@cap-js/*` plugins register themselves automatically via `cds-plugin` — no wiring needed; new capabilities unlocked just by adding a package
- `cds add` — scaffolding new capabilities into an existing project
- The rule: `@sap/cds-dk` (global) and `@sap/cds` (local) major versions must stay in sync
- Pinned vs. caret ranges in `package.json` — why `^` matters

---

## 4. Checking Your Current State (~5 min)
- `cds version` — what it tells you (global DK, local cds, compiler, Node.js)
- `npm outdated` — spot all outdated packages at once
- `npm view @sap/cds version` — query the latest published version

---

## 5. Running a Minor/Patch Upgrade (~5 min)
- `npm update` for local deps + `npm install -g @sap/cds-dk@latest` for the CLI
- When pinned versions block `npm update` — fix ranges first
- Verify: `cds watch` starts, `npm test` passes

---

## 6. Major Version Upgrades — Process (~8 min)
- Always upgrade one major at a time (7 → 8 → 9, not 7 → 9)
- Where to find breaking changes: the release notes migration section at `cap.cloud.sap/docs/releases/`
- `@sap/cds-attic` — the escape hatch for removed deprecated features
- Database redeployment: when and why it's needed after a major bump (HANA HDI, PostgreSQL)
- Clean install strategy: `rm -rf node_modules package-lock.json && npm install`

---

## 7. Key Breaking Changes by Major Version (~8 min)
- **cds 7 (2023):** `cds-serve` start script required; Node.js 16 minimum; new `@cap-js/sqlite` plugin
- **cds 8 (2024):** New database services (`@cap-js/sqlite`, `@cap-js/hana`) became default; Node.js 18 minimum; `@sap/cds-dk@8` requires `@sap/cds@7+`
- **cds 9 (2025):** Node.js 20 minimum; `@cap-js/cds-test` split out; Event Queues on by default; `req.params` structure changed; new CDS compiler 6 (no fallback); `hdbcds` deploy format removed

---

## 8. DB Schema Changes After an Upgrade (~5 min)

- CAP upgrades can introduce new CAP-managed tables — a redeploy is needed even if you changed no model
- **Persistent outbox** (default since cds 7): adds `cds_outbox` table
- **Event Queues** (default since cds 9): adds internal queue tables
- Enabling a new plugin (change tracking, audit logging) also adds tables — redeploy required
- **SQLite (dev):** `cds watch` recreates the DB automatically — nothing to do
- **HANA (HDI):** HDI calculates a delta; use `cds build --production` then redeploy the HDI container
- **PostgreSQL:** run `cds deploy --to postgres`; use `cds deploy --dry` to preview first

---

## 9. When Things Break — Diagnosis & Recovery (~5 min)

### Where breaks typically surface
- **`npm install` fails** — peer dependency conflicts between `@sap/cds` and `@sap/cds-dk` major versions; or a plugin not yet compatible with the new major
- **`cds watch` doesn't start** — module not found (removed/renamed package), CDS compiler error in a model file, incompatible Node.js version
- **App starts but requests fail** — runtime behaviour change (e.g. `req.params` structure, auth handling); a deprecated API that was silently removed
- **Tests fail but app runs** — test helper API changed (e.g. `cds.test` moved to `@cap-js/cds-test` in cds 9)
- **Silent data issues** — new default feature (e.g. Event Queues) changes how events are processed without an obvious error

### Diagnosis toolkit
- Read the error message fully — CAP errors usually name the exact module or API that changed
- `cds version` — confirm the versions that are actually loaded, not just what's in `package.json`
- `DEBUG=cds:* cds watch` — verbose runtime output to trace what CAP is loading and where it fails
- `cds env --profile production` — check if an unexpected config is active
- Check the release notes migration section for the exact major version you jumped to
- Use the `cap-troubleshooting` skill in OpenCode to walk through a structured diagnosis

### Recovery options
- `@sap/cds-attic` — re-enables removed deprecated features temporarily while you fix the root cause
- Clean reinstall: `rm -rf node_modules package-lock.json && npm install` — rules out stale dependency trees
- Pin back to the previous major temporarily: `npm install @sap/cds@8` while you work through breaking changes one by one

---

## 10. Testing — Your Safety Net (~5 min)


- Why tests are critical before (or immediately after) a major upgrade
- What to cover: server startup, CRUD on main entities, custom handlers, auth scenarios
- `cds.test()` — the built-in CAP test helper (now in `@cap-js/cds-test`)
- Undocumented/internal APIs can change even in minor releases — test coverage catches this

---

## 11. Live Demo / Walkthrough (~5 min)

```sh
git clone https://github.com/cap-js/incidents-app.git demo-app
cd demo-app
git checkout aeb0d09
nvm use 20
npm install
```

At this point the app is on `@sap/cds ^8`, `@sap/cds-dk ^8`, `@cap-js/sqlite ^1`, `@cap-js/hana ^1`.

**Step 1 — diagnose**
- Ask OpenCode: *"is my app up to date?"*
- Walk through the upgrade report: cds 8 → 9, `@cap-js/sqlite` 1 → 2, `@cap-js/hana` 1 → 2, Node.js requirement

**Step 2 — upgrade**
- Run the upgrade: `npm install @sap/cds@latest @sap/cds-dk@latest @cap-js/sqlite@latest @cap-js/hana@latest`
- Point out the cds 9 breaking changes flagged from the release notes: Node.js 20 minimum, Event Queues on by default, `req.params` structure, `hdbcds` format removed

**Step 3 — verify**
- `cds watch` — confirm server starts cleanly
- `npm test` — confirm tests still pass
- Show the new `cds version` output confirming cds 9
