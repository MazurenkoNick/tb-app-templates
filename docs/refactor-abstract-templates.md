# Refactor: Abstract App Templates + Dynamic Version Wiring

**Status:** Proposed / plan for review
**Scope:** `agent-app-templates` (data), `tb-updates` (version graph), `thingsboard-pe` (consumer)
**Author:** design notes

---

## 1. Problem

Today every ThingsBoard Edge release ships a full copy of its docker-compose "template"
(`template-EDGE-DOCKER_COMPOSE-<version>.json`) plus a full copy of its compose set
(`compose/edge/<version>/{in_memory,kafka,hybrid}.yml`). Diffing these copies across
versions, the **only** things that actually change are:

| Varies per version | Example | Equivalent field in `edge-update.json` |
|---|---|---|
| template `id` | `88290000-…` (deterministic per version) | — (artifact of per-version storage) |
| `nextVersion` | `"4.3.1.2EDGEPE"` / `null` | `nextEdgeVersion` |
| compose paths | `compose/edge/4.2.2/*.yml` | derived from `edgeVersion` |
| migration job `image` | `thingsboard/tb-edge-pe:4.2.2EDGEPE` | derived from `edgeVersion` |
| presence of the 2 DB steps (`COMPOSE` postgres-up + `RUN_JOB` migrate) | present on minors, absent on patches | `requiresUpdateDb` |
| final start step `pullImages` | `false` (DB path) / `true` (patch path) | derived from `requiresUpdateDb` |

The compose YAMLs differ **only in the image tag** within a minor line (patch→patch),
though infra genuinely changes at major/minor boundaries (kafka image + listener config
changed at 3.9.1 → 4.0.0).

In other words: the per-version fan-out is just `edge-update.json`'s
`{edgeVersion, nextEdgeVersion, requiresUpdateDb}` re-encoded as static file copies. The
version graph already lives authoritatively in `tb-updates/edge-update.json`, which PE
already fetches via `installMapping` / `upgradeMapping` (`EdgeUpgradeInfo` literally has
`requiresUpdateDb` + `nextEdgeVersion`) — but the agent-app-template subsystem currently
has **zero linkage** to it.

Gateway and Generic do **not** have per-version fan-out today (single template each,
`upgradeSteps: []`, `nextVersion: null`), so the duplication problem is **Edge-only**.
They are still folded into the same model for consistency.

---

## 2. Goals / Non-goals

**Goals**
- One **abstract** structural template per app type (EDGE / GATEWAY / GENERIC); no
  per-version copies.
- The version graph (`edgeVersion`, `nextEdgeVersion`, `requiresUpdateDb`) is the single
  source of truth in `tb-updates` (`edge-update.json`, and a new `gw-update.json`).
- DB-migration steps run **only** when `requiresUpdateDb == true`.
- Store template definitions **in memory** (no DB table); applications & profiles
  reference templates by a **`version` string** (+ app type), not by UUID.
- Support both a **line-level** compose default and **concrete per-version** compose
  overrides.

**Non-goals (for now)**
- Reworking the Go agent (`tb-agent`) beyond optional `imageDigest` cleanup.
- Changing the git-sync transport (`GitSyncService`) itself.
- Generic app versioning semantics (Generic stays verbatim / unversioned).

---

## 3. Core design decision — materialize at fetch, not at resolve

Rather than keep placeholders in the stored template and resolve them on every application
action, we **expand the abstract template into concrete per-version templates once, at
git-sync / fetch time**, and store those.

This keeps the runtime path (step resolution for an application action) essentially
unchanged, and confines all the "dynamic wiring" to a single materialization pass.

Timing of the two placeholder families is deliberately different:

| Placeholder | Resolved at | Why |
|---|---|---|
| `${var.<name>}`, `${var.!<name>}` (version vars) | **fetch / materialization** | Depends only on the version graph, known at fetch. |
| `${compose.svcImgRegex(...)....}` (compose-node refs) | **runtime** (existing `AgentComposeRefUtils` in `RunJobStep`) | Depends on the app's *actual* compose after user edits + credential injection. |

Likewise, `condition` on a step is a **fetch-time** construct: the materializer decides
whether to include the step and re-stitches the linked list. The runtime resolver never
sees a condition.

### Consequences

- The in-memory registry holds **N concrete templates** (one per version per app type),
  **not** 3. It looks like today's DB content, except **generated** from *1 abstract
  template + the version graph* instead of hand-copied.
- **No runtime version-graph bean.** The graph is consumed only during materialization to
  (a) enumerate versions, (b) set each concrete template's `nextVersion`, (c) decide
  `requiresUpdateDb`. At runtime each concrete template already carries its own
  `nextVersion`, so upgrade-eligibility reads it straight from the registry entry.
- The runtime resolver (`DefaultAgentAppEventStepsResolver`) needs **no** condition/variable
  logic — only its existing `templateOnly` filter.

---

## 4. Repository layout (`agent-app-templates`)

### Templates — one abstract file per app type
```
templates/
  template-EDGE-DOCKER_COMPOSE.json       # abstract (no version in name)
  template-GATEWAY-DOCKER_COMPOSE.json    # abstract
  template-GENERIC-DOCKER_COMPOSE.json    # abstract (verbatim, no versioning)
  since/
    4.4.0.1/template-EDGE-DOCKER_COMPOSE.json # replaces the base edge template on TB 4.4.0.1+
```

Every TB server syncs this branch, so a base template may only use what the oldest supported
server understands (e.g. an unknown `${var.*}` is left as a literal). A template that needs
something newer goes to `since/<TB version>/` under the same file name: a server uses, per file,
the newest `since` folder whose version is not above its own, else the base file. Folders don't
have to contain every template, and servers that predate `since/` never list it.

Example: `since/4.4.0.1` uses `${var.nextEdgeRepo}` (`tb-edge-pe` up to 4.4.0, `tb-edge` from
4.4.0.1) for the DB-migration job image, since the 4.4.0 -> 4.4.0.1 upgrade switches image repos.
A TB 4.4.1 server with no `since/4.4.1` folder keeps using `since/4.4.0.1`.

### Compose — version folders, flat
Compose sets live in **version folders side by side** (no nesting): one per infra "line"
(major.minor) plus a folder for any later version whose compose differs. Within a folder only
the image tag varies, so parametrize it.

```
compose/
  edge/
    3.9/{in_memory,kafka,hybrid}.yml            # line default; image: thingsboard/tb-edge-pe:${var.edgeVersion}
    4.0/{in_memory,kafka,hybrid}.yml
    4.2/{in_memory,kafka,hybrid}.yml
    4.3/{in_memory,kafka,hybrid}.yml
    4.4/{in_memory,kafka,hybrid}.yml
    4.4.0.1/{in_memory,kafka,hybrid}.yml        # 4.4.0.1+ CE-style image: thingsboard/tb-edge:${var.edgeVersion}
  gateway/
    3.8/default.yml                             # image: thingsboard/tb-gateway:${var.gatewayVersion}
  generic/
    default.yml
```

**Compose lookup during materialization** for version `V`, per compose file:

> the file from the newest version folder that is not above `V`, falling back to older folders
> when that folder doesn't have the file. For `4.4.1`: `4.4.0.1/` -> `4.4/` -> `4.3/` -> …

So a folder applies from its version on (e.g. `4.4.0.1/` also serves 4.4.0.2, 4.4.1, 4.5.0, …)
until a newer folder exists; add a folder only when the compose actually changes. Template
paths keep the `${var.composeLine}` folder (`compose/edge/${var.composeLine}/kafka.yml`): newer
servers only use it to find the parent folder to look in.

Servers that predate flat folders (TB <= 4.4.0) look up `compose/edge/<line>/` directly and never
materialize versions above their own, so they keep reading the line folders. Never rename or
remove a line folder, and don't nest version folders inside one.

Concrete override files may hardcode their tag or keep `${var.edgeVersion}` — the
substitutor runs over whichever file is loaded, so both work.

This collapses ~17 template files → 3 and ~17 compose dirs → ~5 (one per minor line, plus
occasional concrete overrides). Adding a new **patch** release becomes: edit
`edge-update.json` only — no change in this repo. Adding a **minor** with infra changes:
add one compose line dir.

---

## 5. Version graph (source of truth in `tb-updates`)

### Edge — `edge-update.json` (existing)
```jsonc
{
  "versionMapping": [ { "tbVersion": "...", "edgeVersion": "..." }, ... ],
  "edgeVersions": {
    "3.6.0": { "requiresUpdateDb": true, "nextEdgeVersion": "3.6.1" },
    ...
    "<latest>": { "requiresUpdateDb": true, "nextEdgeVersion": null }
  }
}
```
`edgeVersions[v].requiresUpdateDb` and `.nextEdgeVersion` drive materialization directly.

### Gateway — `gw-update.json` (new, same shape, managed the same way)
Gateway's `nextVersion` policy — **jump-to-latest vs chained** — is decided **here**, in
`gw-update.json`, exactly like edge. No `requiresUpdateDb` for gateway (no DB migration).

```jsonc
{
  "gatewayVersions": {
    "3.8.0": { "nextGatewayVersion": "3.9.0" },     // chained example
    "3.9.0": { "nextGatewayVersion": null }
    // — or — jump-to-latest: every entry's next points straight at the latest known version
  }
}
```

The update server exposes this the same way it exposes the edge mapping (a mapping
endpoint), so PE fetches it with no new client type. Because the `next` policy lives in the
data file, switching gateway between "chained" and "jump-to-latest" is a data edit, not a
code change.

### Generic
No graph. Generic has no version progression; its template is used verbatim.

---

## 6. Abstract template format

Two additions vs. today: `${var.*}` placeholders and an optional `condition` on steps.
No `id`, no `currentVersion`, no `nextVersion`, no `imageDigest` in the abstract file —
those are either generated at materialization or removed entirely.

### Abstract edge template (illustrative)
```jsonc
{
  "appType": "EDGE",
  "configType": "DOCKER_COMPOSE",

  "startSteps": [
    { "id": "…00", "nextId": "…01", "type": "COMPOSE_TEMPLATE", "templateOnly": true,
      "composeTemplates": {
        "in_memory": "compose/edge/${var.composeLine}/in_memory.yml",
        "kafka":     "compose/edge/${var.composeLine}/kafka.yml",
        "hybrid":    "compose/edge/${var.composeLine}/hybrid.yml" } },
    { "id": "…01", "type": "COMPOSE",
      "state": { "pullImages": { "value": true, "userChoice": true } } }
  ],

  "upgradeSteps": [
    { "id": "…01", "nextId": "…02", "type": "COMPOSE_DOWN", "title": "Remove Current Edge containers" },
    { "id": "…02", "nextId": "…13", "type": "BACKUP_VOLUME",
      "state": { "backupVolumes": { "value": [], "userChoice": true } } },

    // ── conditional DB block: included only when requiresUpdateDb == true ──
    { "id": "…13", "nextId": "…14", "type": "COMPOSE", "condition": "requiresUpdateDb",
      "state": { "serviceImageRegexPatterns": { "value": ["postgres:.+"] },
                 "pullImages": { "value": true, "userChoice": true } } },
    { "id": "…14", "nextId": "…04", "type": "RUN_JOB", "condition": "requiresUpdateDb",
      "state": {
        "image":      { "value": "thingsboard/tb-edge-pe:${var.edgeVersion}" },
        "binds":      { "value": ["${compose.svcImgRegex(thingsboard/tb-edge-pe:.+).volumes}"] },
        "env":        { "value": ["${compose.svcImgRegex(thingsboard/tb-edge-pe:.+).environment}"] },
        "entrypoint": { "value": ["upgrade-tb-edge.sh"] },
        "pullImages": { "value": false },
        "retries":    { "value": 10 },
        "networkFromServiceImageRegexPattern": { "value": "postgres:.+" } } },
    // ──────────────────────────────────────────────────────────────────────

    { "id": "…04", "nextId": "…05", "type": "COMPOSE",
      "state": { "pullImages": { "value": "${var.!requiresUpdateDb}",
                                 "userChoice": "${var.!requiresUpdateDb}" } } },
    { "id": "…05", "type": "BACKUP_VOLUME_REMOVE" }
  ],

  "deleteSteps":   [ /* unchanged; no version refs */ ],
  "rollbackSteps": [ /* unchanged */ ],
  "restartSteps":  [ /* unchanged */ ]
}
```

### The `pullImages` / `userChoice` rule (why `${var.!requiresUpdateDb}`)

The upgrade has two concrete shapes:
```
requiresUpdateDb = true :  down → backup → up[postgres only, pull=true] → RUN_JOB(migrate) → up[all, pull=false] → cleanup
requiresUpdateDb = false:  down → backup →                                                   up[all, pull=true]  → cleanup
```
The final "up[all]" step pulls (`true`) in the no-DB path but not (`false`) in the DB path,
because the DB path already pulled earlier. Hence its `pullImages.value` **and**
`pullImages.userChoice` are both `!requiresUpdateDb`.

We deliberately **keep this structure** (call it option "a") rather than restructuring into
a single unconditional pull + one conditional `RUN_JOB` (option "b"). Option "b" would
collapse the intentional "start postgres-only → migrate → start everything" ordering into
"start everything once", changing upgrade semantics. Because we materialize at fetch,
`${var.!requiresUpdateDb}` is resolved to a concrete boolean per version, so keeping the
structure costs nothing at runtime.

---

## 7. Materialization algorithm

Run on **every fetch** (see triggers, §11). Pseudocode per app type:

```
for each abstractTemplate in git(templates/):
    appType = abstractTemplate.appType
    graph   = versionGraphFor(appType)          // edge-update.json / gw-update.json / none

    if graph == none:                            // GENERIC
        registry.put(appType, unversioned, substituteVars(abstractTemplate, {}))
        continue

    for each version V in graph:
        vars = {
          edgeVersion|gatewayVersion : V,
          nextEdgeVersion|nextGatewayVersion : graph.next(V),
          requiresUpdateDb : graph.requiresUpdateDb(V)   // false for gateway
          composeLine : lineOf(V)                        // e.g. "4.2" from "4.2.1"
        }
        concrete = deepCopy(abstractTemplate)
        concrete.nextVersion = graph.next(V)

        for each stepList in concrete:
            // 1. drop conditional steps whose condition var is false, re-stitch nextId
            StepLinkedListUtils.removeAndRestitch(stepList,
                step -> step.condition != null && !bool(vars, step.condition))
            // 2. substitute ${var.*} / ${var.!*} in remaining step state + compose paths
            substituteVars(stepList, vars)
            // 3. resolve compose sets: concrete override subfolder else line default,
            //    load yml, convert to JSON (as today), tag already substituted
            resolveComposeTemplates(stepList, appType, V, vars.composeLine)

        registry.put(appType, V, concrete)
```

Notes:
- `${compose.*}` refs are **left intact** (resolved at runtime).
- `removeAndRestitch` handles head removal and consecutive removals.
- Materialization is pure/deterministic given (abstract template, graph) — an atomic
  registry swap makes refresh safe under concurrency.

---

## 8. In-memory registry

```java
// key: app type -> (version -> concrete template)
Map<AgentApplicationType, NavigableMap<String, AgentAppTemplate>>
```
- `NavigableMap` yields `latest(type)` (install target), ordering, and `next(type, version)`
  for free.
- Generic = a single unversioned entry.
- Refresh = build a new map, then atomic reference swap.

Lookups used by the rest of the system:
- `get(appType, version)` → concrete template for step resolution.
- `latest(appType)` → install version.
- `next(appType, version)` → upgrade eligibility (edge: strict next; gateway: whatever
  `gw-update.json` encodes).

---

## 9. Runtime resolution (mostly unchanged)

`DefaultAgentAppEventStepsResolver`:
- Look up the concrete template by `(appType, application.version)` (was: `templateId`).
- Map action type → step list (unchanged).
- Apply the existing `templateOnly` filter (unchanged).
- **No** condition or `${var.*}` handling (already materialized).
- `${compose.*}` refs still resolved by `RunJobStep.getCommandMetadata` via
  `AgentComposeRefUtils` (unchanged).

---

## 10. Storage model — in-memory, referenced by version string

Template **definitions** live only in memory (the registry). There is **no**
`agent_app_template` table. Applications & profiles reference a template by
**`(appType, version)`** — a string — not a UUID.

| | Today | Target |
|---|---|---|
| Template store | `agent_app_template` table (N rows/version) | in-memory `AppTemplateRegistry` |
| App → template ref | `template_id` / `desired_template_id` (UUID) | `version` / `desired_version` (String) |
| Profile → template ref | `template_id` (UUID) | `version` (String) |
| "What's next version" | `currentTemplate.nextVersion == desiredTemplate.currentVersion` across rows | `registry.next(appType, app.version)` |
| Add a version | new template file + compose dir + DB sync | edit `edge-update.json` / `gw-update.json` |

Rollout is a **single migration straight to the target** (no interim phase keeping the
table).

---

## 11. Update management / triggers

Re-materialize the registry when **either** input changes:
1. **git-sync** of `agent-app-templates` (abstract template or compose set changed) — via
   the existing `GitSyncService` registration in `AgentAppTemplateSyncService`.
2. **version-graph poll** — the hourly fetch of `edge-update.json` / `gw-update.json` in
   `DefaultUpdateService`.

Operational outcomes:
- New **edge patch** release → edit `edge-update.json` (`requiresUpdateDb: false`, set
  `nextEdgeVersion`). No `agent-app-templates` change.
- New **edge minor** with infra changes → add one `compose/edge/<line>/` set; set that
  hop's `requiresUpdateDb: true` in `edge-update.json`.
- New **gateway** release → edit `gw-update.json` (chained or jump-to-latest, as configured
  there).
- **Structural** change (rare) → edit the single abstract template; git-sync picks it up.

---

## 12. PE class changes

### New
- `AppTemplateMaterializer` (`application/.../service/agent/template/`) — expands an abstract
  template + version graph into concrete per-version templates (substitute `${var.*}`,
  evaluate `condition` + re-stitch, resolve compose sets, set `nextVersion`).
- `AppTemplateRegistry` (bean) — the in-memory map; atomic swap; `get/latest/next`.
- `TemplateVarSubstitutor` (`common/data`) — resolves `${var.<name>}` / `${var.!<name>}`
  against a `Map<String,Object>`; leaves `${compose.*}` untouched.
- `StepLinkedListUtils.removeAndRestitch(steps, predicate)` — drop matching nodes, link
  predecessor `nextId` → removed `nextId`; handle head + consecutive removals.
- (`tb-updates`) `gw-update.json` + a gateway mapping endpoint mirroring the edge one
  (minus `requiresUpdateDb`).

### Changed
- `AgentAppStep` — add optional `String condition` (consumed only by the materializer).
- `AgentAppTemplateSyncService` — version-less filename pattern
  `template-<APP>-<CONFIG>.json`; drop folder-version gating (`FOLDER_TB_VERSION`,
  `isValidFolderVersion`, `compareVersions`); on sync load abstract template + compose sets,
  call the materializer, publish to the registry (no DB save); re-run on graph updates.
- `DefaultUpdateService` — feed the already-fetched edge `upgradeMapping` (versions +
  `requiresUpdateDb` + `nextEdgeVersion`) and the new gateway graph into the materialization
  trigger (no new HTTP client).
- `DefaultAgentAppEventStepsResolver` — look up by `(appType, version)`; drop UUID/template
  logic.
- `MergeUpgradeImageRule` / `MergeTemplateComposeRule` — operate on the concrete template's
  compose (tag already correct); simplifies.
- Lifecycle → version strings: `AgentApplication.version` / `desiredVersion`,
  `AgentAppProfile.version`; `AgentApplicationDataValidator` eligibility via `registry.next`
  (edge strict-next / gateway per `gw-update.json` / generic n/a); `UpgradeActionHandler`
  sets `desiredVersion`; `DefaultAgentEventProcessor` promotes `desiredVersion → version`;
  `AgentEventErrorHandler` clears it; `DefaultAgentBulkActionProcessingService` eligibility
  via registry.

### Removed
- `agent_app_template` table + `AgentAppTemplateEntity` + `AgentAppTemplateService` /
  repository / `AgentAppTemplateId`.
- `ImageDigestChecker` + `AgentAppTemplate.imageDigest` + the `pullRequired` attribute path
  in `ComposeProjectSyncMessageHandler` (see §13).
- All per-version template files (already trimmed; keep abstract only).

---

## 13. `imageDigest` removal

`imageDigest` powers `ImageDigestChecker`, which compares the template's recorded digest
against the running container's digest and writes a **`pullRequired`** server attribute
("your tag was re-pushed → re-pull"). **Removing `imageDigest` removes that
re-pushed-image detection.** Confirmed acceptable.

Removal scope:
- `common/data`: remove `AgentAppTemplate.imageDigest`.
- `dao`: remove `AgentAppTemplateEntity.imageDigest` (+ column) — folded into the migration.
- `application`: delete `ImageDigestChecker`; strip the digest call + `pullRequired` write
  from `ComposeProjectSyncMessageHandler`; drop digest population in
  `AgentAppTemplateSyncService`; update `ComposeProjectSyncMessageHandlerTest`,
  `AbstractAgentTest`.
- `tb-agent` (Go, optional): `ContainerInfo.imageDigest` in the proto + `inspect.go` /
  `dockersync/sync.go` may keep emitting harmlessly or be cleaned up later — not required
  for PE to compile.

---

## 14. DB migration (single, straight to target)

- Add `version` / `desired_version` (varchar) to `agent_application`.
- Add `version` (varchar) to `agent_app_profile`.
- **Backfill** `version` from each row's current template `currentVersion`, and
  `desired_version` from the desired template (for in-flight upgrades) — see open question
  in §16.
- Drop `template_id` / `desired_template_id` from `agent_application` and
  `agent_app_profile`.
- Drop `agent_app_template` table (incl. `image_digest`).

---

## 15. Gateway & Generic specifics

- **Gateway** — materialized per known version (image tag substituted), **no DB block**.
  `next` policy (chained vs jump-to-latest) is encoded in `gw-update.json`, so eligibility
  = `registry.next(GATEWAY, version)` and the data file decides whether that is the next
  hop or the latest. `upgradeSteps` (empty today) would be `COMPOSE_DOWN → COMPOSE(pull+start)`
  when added — never a `RUN_JOB`.
- **Generic** — single unversioned registry entry; no graph, no `${var.*}`, no `condition`.
  The materializer is effectively a no-op (verbatim copy). This is "how generic is pulled":
  loaded like the others, but with an empty var map and no version enumeration. `version`
  for a generic app is a free-text label.

---

## 16. Open questions / decisions log

**Decided**
- Version graph source: reuse `tb-updates/edge-update.json` (edge) + new `gw-update.json`
  (gateway); generic has none. ✔
- Storage: in-memory registry, reference by `(appType, version)` string; single migration
  straight to target (no interim phase). ✔
- Compose: line default + optional per-version subfolder override. ✔
- Keep the upgrade step structure ("a"); drive final-step `pullImages.value` **and**
  `.userChoice` via `${var.!requiresUpdateDb}`. ✔
- Drop `imageDigest` + `ImageDigestChecker` + `pullRequired` (accepting loss of
  re-pushed-image detection). ✔
- Gateway `nextVersion` policy (jump-to-latest vs chained) lives in `gw-update.json`. ✔
- Materialize at fetch; no runtime version-graph bean; `TemplateVarContext` is a
  `Map<String,Object>`. ✔

**Open**
1. **Materialization trigger** — re-materialize on *both* git-sync of `agent-app-templates`
   and the version-graph poll. (Assumed yes.)
2. **Backfill of in-flight upgrades** — must migration preserve rows with a non-null
   `desired_template_id`, or is a maintenance-window cutover acceptable?
3. **Gateway mapping endpoint** — expose `gw-update.json` via a dedicated endpoint mirroring
   the edge mapping, or fold gateway into the existing update response.

---

## 17. Appendix — materialized example (edge 4.2.2, `requiresUpdateDb=true`)

Given abstract §6 + `edge-update.json` entry
`"4.2.2": { requiresUpdateDb: true, nextEdgeVersion: "4.3.1.2EDGEPE" }`, the materializer
emits (upgrade steps shown):

```jsonc
{
  "appType": "EDGE", "configType": "DOCKER_COMPOSE",
  "nextVersion": "4.3.1.2EDGEPE",
  "upgradeSteps": [
    { "id":"…01","nextId":"…02","type":"COMPOSE_DOWN" },
    { "id":"…02","nextId":"…13","type":"BACKUP_VOLUME","state":{"backupVolumes":{"value":[],"userChoice":true}} },
    { "id":"…13","nextId":"…14","type":"COMPOSE",
      "state":{"serviceImageRegexPatterns":{"value":["postgres:.+"]},"pullImages":{"value":true,"userChoice":true}} },
    { "id":"…14","nextId":"…04","type":"RUN_JOB",
      "state":{"image":{"value":"thingsboard/tb-edge-pe:4.2.2EDGEPE"},
               "binds":{"value":["${compose.svcImgRegex(thingsboard/tb-edge-pe:.+).volumes}"]},
               "env":{"value":["${compose.svcImgRegex(thingsboard/tb-edge-pe:.+).environment}"]},
               "entrypoint":{"value":["upgrade-tb-edge.sh"]},"pullImages":{"value":false},
               "retries":{"value":10},"networkFromServiceImageRegexPattern":{"value":"postgres:.+"}} },
    { "id":"…04","nextId":"…05","type":"COMPOSE","state":{"pullImages":{"value":false,"userChoice":false}} },
    { "id":"…05","type":"BACKUP_VOLUME_REMOVE" }
  ]
}
```

For a **patch** version with `requiresUpdateDb=false`, the `…13` and `…14` steps are
dropped and `…02.nextId` is re-stitched to `…04`, whose `pullImages` becomes
`{value:true, userChoice:true}`:

```jsonc
"upgradeSteps": [
  { "id":"…01","nextId":"…02","type":"COMPOSE_DOWN" },
  { "id":"…02","nextId":"…04","type":"BACKUP_VOLUME","state":{"backupVolumes":{"value":[],"userChoice":true}} },
  { "id":"…04","nextId":"…05","type":"COMPOSE","state":{"pullImages":{"value":true,"userChoice":true}} },
  { "id":"…05","type":"BACKUP_VOLUME_REMOVE" }
]
```
