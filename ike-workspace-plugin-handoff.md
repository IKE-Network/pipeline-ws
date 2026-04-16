# IKE Workspace Plugin — Design and Implementation Handoff

## Date: 2026-04-02

## Purpose

This document specifies the design for `ike-workspace-maven-plugin`, the
Maven plugin that manages the IKE multi-repository workspace. A Claude Code
session will implement this. Every design decision below was reached through
explicit discussion and should be treated as a requirement, not a suggestion.

---

## 1. Foundational Principles

### 1.1 The Triangle

A release process is an irreducible interaction between three concerns:
a VCS, a build engine, and a body of sources. The version string is declared
in sources, consumed by the build engine, and labeled in VCS. All three must
agree, and at least two must be mutated in coordination during any release or
branch operation. This coupling cannot be eliminated—only managed explicitly.

### 1.2 The Workspace Model

The IKE workspace is a collection of git repositories (called "components")
that are developed, branched, and released as a coordinated unit. The
workspace is described by a `workspace.yaml` manifest checked into its own
git repository (the "aggregator repository"). The aggregator also contains a
Maven aggregator POM with `<subprojects>` listing all components.

**Lifecycle phases:**

1. **Specification time.** An architect authors or updates `workspace.yaml`.
   This is a design act.
2. **Realization time.** `ws:init` reads the YAML and clones every component
   at the declared repo URL and branch. After this, POMs exist on disk.
3. **Validation time.** `ws:verify` checks that the realized state matches
   the specification.

The YAML comes first. The POMs don't exist on disk until the YAML has been
processed. "Derive" is not a bootstrapping concept—it is an internal function
used by verify after realization.

### 1.3 No Bash

All workspace tooling is implemented in Java. No bash scripts, no shell
wrappers, no thin-Java-over-bash patterns. Cross-platform (macOS, Windows,
Linux) is a hard requirement. The JVM startup cost for workspace operations
that run a few times a day is irrelevant.

### 1.4 Maven 4.1.0

All POMs use the `http://maven.apache.org/POM/4.1.0` namespace. Full element
names only—never abbreviated (`n`, `v`). `<subprojects>` not `<modules>`.
Read `CLAUDE.md` and `IKE-MAVEN.md` at any project root before modifying POMs.

Never use `${revision}` or CI-friendly variables. Keith requires reproducible
builds from SCM alone, no external `-D` flags. The POM is self-describing;
the checkout is the truth.

---

## 2. Invariants

These are enforced preconditions, not conventions.

### 2.1 Single Version Per Reactor

Each workspace component is a single-version reactor. All submodules inherit
version from the root POM via Maven 4.1.0 implicit inheritance. **Submodules
must not declare `<version>`.** This invariant is what makes a component a
coherent unit for release, checkpoint, and cascade operations.

`ws:verify` enforces this: for each cloned component, walk its
`<subprojects>` list recursively. Any submodule POM that declares an explicit
`<version>` element (rather than inheriting) is a FAIL.

If a repository has submodules that genuinely need independent versions, it
cannot be modeled as a single workspace component. It must be split into
multiple components, each with its own entry in `workspace.yaml`.

### 2.2 Aggregator Participates in Branching

The aggregator repository is a participant, not an observer. When
`ws:feature-start` runs, it branches the aggregator itself alongside every
component repository. The `workspace.yaml` on the feature branch carries
the feature-branch version strings. The `workspace.yaml` on main carries
main-branch version strings. This ensures that cloning the aggregator at
any branch and running `ws:init` produces a self-consistent workspace.

### 2.3 Workspace Operations Are Whole-Workspace

All SCM-related goals (feature-start, feature-finish, release, post-release)
operate on the entire workspace as a unit. If you want to branch a single
component independently, do it outside the workspace tooling. The workspace
plugin enforces coordinated state transitions.

---

## 3. workspace.yaml Schema

### 3.1 Field Classification

Fields in `workspace.yaml` component entries fall into two categories:

**Novel fields** (can't live anywhere else—YAML is authoritative):
- `repo` — git remote URL
- `branch` — which branch to clone for this coordinated state
- `type` — component classification (software, document, knowledge-source,
  infrastructure, template)
- `depends-on` edges with `relationship: content` or `relationship: tooling`
- `groups` — named subsets for partial operations
- `notes` — rationale, migration annotations, architectural commentary

**Denormalized fields** (also in POMs—kept for readability, verified for
consistency):
- `groupId` — primary groupId of the component (documentary, not structural)
- `version` — root POM version string
- `depends-on` edges with `relationship: build`

Denormalized fields have a different update cadence than novel fields. Novel
fields change when the architect redesigns the workspace (rare, deliberate).
Denormalized fields change whenever a version bumps (frequent, mechanical).

### 3.2 groupId Is Documentary

The `groupId` field in workspace.yaml is a human-readable annotation. The
tool does NOT use it as a match key for dependency discovery. A component
may publish artifacts under multiple groupIds (e.g., `dev.ikm.tinkar` and
`dev.ikm.tinkar.provider` within tinkar-core). The tool discovers edges by
scanning the published artifact set (see section 5.2), not by matching a
single declared groupId.

`ws:verify` can check that the declared groupId is still accurate as a
courtesy warning, but it is not load-bearing for any algorithm.

---

## 4. Plugin Architecture

### 4.1 Module Structure

All modules live in the `ike-pipeline` reactor:

```
ike-plugin-common/                — shared library
ike-maven-plugin/                 — build-time utilities (existing)
ike-workspace-maven-plugin/       — workspace operations (new)
```

**`ike-plugin-common`** (`network.ike:ike-plugin-common`):
- Java model classes for workspace.yaml (components, edges, groups, types)
- YAML parsing (SnakeYAML)
- POM analysis utilities:
  - Extract published artifact set from a reactor
  - Extract dependency list from all submodule POMs
  - Validate single-version invariant
  - Version comparison
- Git operations (JGit or ProcessBuilder to `git` CLI)
- Workspace model I/O (read/write workspace.yaml)

**`ike-maven-plugin`** (`network.ike:ike-maven-plugin`, goal prefix `ike:`):
- Existing goals: `ike:help`, `ike:adocstudio`, `ike:semantic-linebreak`
- Future single-module build-time utilities
- Does NOT need workspace context

**`ike-workspace-maven-plugin`** (`network.ike:ike-workspace-maven-plugin`,
goal prefix `ws:`):
- All workspace-level goals (see section 5)
- Most goals are `aggregator = true`
- Depends on `ike-plugin-common`

### 4.2 Dependency Management

Both plugins depend on `ike-plugin-common`. Standard Maven dependency,
not parent inheritance. `ike-plugin-common` is a JAR, not a plugin.

Plugin dependencies:
- `maven-plugin-api` (provided)
- `maven-plugin-annotations` (provided)
- `snakeyaml` (for workspace.yaml parsing)
- JGit or direct `git` CLI invocation (TBD—JGit is pure Java and
  cross-platform; CLI is simpler but requires git on PATH)

---

## 5. Goal Specifications

### 5.1 ws:create

**Purpose:** Create a new empty workspace.

**Parameters:**
- `workspaceDir` (required) — directory for the workspace
- `artifactId` (optional, default: directory name) — aggregator artifactId

**Behavior:**
1. Create directory if absent.
2. Initialize git repository.
3. Write skeleton `workspace.yaml` (schema-version, empty components/groups).
4. Write skeleton aggregator `pom.xml` (4.1.0, `local.aggregate` groupId,
   `<packaging>pom</packaging>`, empty `<subprojects>`).
5. Write `.gitignore` (ignores child component directories).
6. Initial commit.

**Postconditions:**
- Directory contains a valid git repo with workspace.yaml, pom.xml, .gitignore.
- `ws:verify` passes (trivially—no components).

---

### 5.2 ws:init

**Purpose:** Realize the workspace from the specification. Clone all
components declared in workspace.yaml.

**Parameters:**
- `group` (optional) — named group to clone (default: all components)

**Behavior:**
1. Read workspace.yaml.
2. For each component (or each component in the specified group):
   a. If directory already exists with `.git/`, skip (idempotent).
   b. Clone repo at the declared branch into the workspace directory.
   c. Verify the clone is on the expected branch.
3. After all clones complete, run `ws:verify` to confirm consistency.

**Idempotency:** `ws:init` can be run repeatedly. It skips already-cloned
components. This supports incremental workspace population and recovery
from partial failures.

**Bootstrap note:** `ws:init` runs from the aggregator repository, which
is already cloned. The aggregator POM exists. Components listed in
`<subprojects>` may not exist on disk yet—Maven 4 validates subproject
existence, so `ws:init` must either: (a) run as `requiresProject = false`
before Maven validates subprojects, (b) use a profile to conditionally
include subprojects, or (c) be invoked via `exec:java` from
`ike-plugin-common` rather than as a Mojo. This is an open design question
(see section 10).

**Postconditions:**
- All components (or group members) are cloned at their declared branches.
- `ws:verify` passes.

---

### 5.3 ws:add

**Purpose:** Add a component to the workspace.

**Parameters:**
- `repo` (required) — git remote URL
- `branch` (optional, default: main) — branch to clone
- `type` (required) — component type (software | document | knowledge-source |
  infrastructure | template)
- Content/tooling relationship edges (optional) — manually specified, since
  these can't be inferred

**Behavior:**
1. Clone the repo at the specified branch into the workspace directory.
2. Parse the root POM: extract groupId, artifactId, version, packaging.
3. **Validate single-version invariant**: walk `<subprojects>` recursively.
   Any submodule with an explicit `<version>` → reject with error. The
   component cannot be added until this is fixed.
4. **Build the published artifact set**: scan all submodule POMs in the
   reactor. Collect every groupId:artifactId pair. This is the set of
   artifacts this component publishes.
5. **Discover build-relationship edges**: for each already-registered
   component in workspace.yaml, retrieve its published artifact set. For
   the new component, collect all `<dependency>` declarations across all
   its submodule POMs. For each dependency, check if its groupId:artifactId
   matches any published artifact from any registered component. Each match
   is a candidate `relationship: build` edge.
6. **Handle version mismatches**: if a dependency declares version X but the
   workspace component is at version Y, report this to the user. The
   workspace version is assumed to be correct (equal or later). This is
   informational—the reactor resolves it correctly at build time.
7. **Present discovered edges for confirmation**: "komet depends on
   tinkar-core (build). komet depends on rocks-kb (build). Confirm?"
   The architect may also specify content or tooling edges.
8. **Write the component entry to workspace.yaml**: repo, branch, type,
   groupId (derived), version (derived), confirmed edges, any
   architect-supplied notes.
9. **Update aggregator POM**: add `<subproject>` entry in dependency order
   (topological sort of the workspace graph).
10. **Commit**: workspace.yaml and pom.xml changes on the aggregator repo.

**Postconditions:**
- Component is cloned at correct branch.
- workspace.yaml has new entry with derived + novel fields.
- Aggregator POM lists new subproject in dependency order.
- `ws:verify` passes for the new component.

**POM parsing notes:**
- Use Maven's `ProjectBuilder` component (inject via `@Component`) to build
  a fully resolved `MavenProject` from the cloned POM. This handles parent
  inheritance, property interpolation, and profile activation correctly.
  Do NOT write custom XML parsing—reuse Maven's infrastructure.
- Two POM namespace versions in play: `http://maven.apache.org/POM/4.0.0`
  (komet, legacy) and `http://maven.apache.org/POM/4.1.0` (everything else).
  Maven's `ProjectBuilder` handles both transparently.
- For version changes during feature-start/release, use `mvn versions:set`
  via subprocess. For consumer POM workarounds (Maven 4 bugs requiring
  literal version replacement), use direct XML manipulation as already
  established in the codebase.

---

### 5.4 ws:remove

**Purpose:** Remove a component from the workspace.

**Parameters:**
- `component` (required) — component name to remove

**Behavior:**
1. Check for downstream dependents in workspace graph. If any exist, FAIL
   with message listing dependents (unless `--force`).
2. Remove component entry from workspace.yaml.
3. Remove `<subproject>` from aggregator POM.
4. Optionally remove cloned directory (prompt user, default: no).
5. Commit workspace.yaml and pom.xml.

---

### 5.5 ws:verify

**Purpose:** Compare realized workspace state against the specification.

**Parameters:**
- `--update` (optional flag) — auto-fix POM-derivable fields in YAML

**Behavior:**

For each component listed in workspace.yaml that is cloned on disk:

**Strict checks (FAIL):**
- YAML `groupId` must match POM's resolved groupId.
- YAML `version` must match POM's `<version>`.
- Single-version invariant: no submodule declares explicit `<version>`.
- Build-relationship edges: every YAML `relationship: build` edge must
  correspond to an actual POM dependency. Specifically: if YAML says
  component A depends on component B (build), then A's reactor must
  contain at least one dependency whose groupId:artifactId is in B's
  published artifact set.

**Reverse derivation (FAIL):**
- Walk every cloned component's POM dependency list. For each dependency,
  check if its groupId:artifactId matches any workspace component's
  published artifact set. If yes and the YAML has no `depends-on` edge
  for that pair → **missing edge**. This catches dependencies added in
  POMs without updating the manifest.

**Advisory checks (WARN):**
- YAML `branch` should match `git branch --show-current` in the component
  directory. Drift means someone switched branches without updating the
  manifest.
- Cross-component version references: if component A's POM declares a
  dependency on component B's artifact at version X, but B's root POM is
  at version Y, report the mismatch. The reactor masks it, but standalone
  builds would use the stale version.
- Content/tooling dependency targets not cloned on disk.
- Component `type` heuristic check (e.g., software type but no JAR
  packaging → WARN).

**Informational (INFO):**
- External dependencies (groupId doesn't match any workspace component).
  Listed for boundary awareness.

**Output format:**
```
ws:verify — 2026-04-02T10:30:00

COMPONENT: tinkar-core
  groupId:          PASS  (yaml: dev.ikm.tinkar, pom: dev.ikm.tinkar)
  version:          FAIL  (yaml: 0.6.0-SNAPSHOT, pom: 0.7.0-SNAPSHOT)
  single-version:   PASS  (27 submodules, 0 explicit versions)
  branch:           WARN  (yaml: feature/kec-jan-24, actual: feature/kec-march-15)
  edges:            PASS  (2 declared, 2 verified, 0 missing)

COMPONENT: komet
  groupId:          PASS
  version:          PASS
  single-version:   PASS
  branch:           PASS
  edges:            WARN  (missing edge: komet → rocks-kb found in POM)

SUMMARY: 10 components, 1 failure, 2 warnings
```

Exit code nonzero on any FAIL. Usable as git pre-commit hook or CI gate.

**--update mode:**
When `--update` is specified, for POM-derivable fields only (groupId,
version, build-relationship edges), update workspace.yaml to match what's
on disk. Novel fields (repo, branch, type, content edges, groups, notes)
are NEVER touched by --update. The result is a modified workspace.yaml
that the architect reviews and commits.

---

### 5.6 ws:fix (alias for ws:verify --update)

Convenience goal. Identical to `ws:verify --update`.

---

### 5.7 ws:status

**Purpose:** Cross-repo git status summary.

**Behavior:**
For each cloned component, run `git status --porcelain` and `git log
--oneline -1`. Report:
- Component name, current branch, version
- Clean / dirty / uncommitted / unpushed
- Commits ahead/behind remote

Compact tabular output.

---

### 5.8 ws:cascade

**Purpose:** Given a changed component, compute the transitive set of
downstream components that must be reverified.

**Parameters:**
- `component` (required) — the changed component
- `--type` (optional) — filter by relationship type (build | content | all)

**Behavior:**
Build reverse adjacency list from workspace.yaml edges. BFS from the
specified component. Report the propagation set with relationship types.

**Output:**
```
ws:cascade from tinkar-core

  tinkar-core
    → rocks-kb (build)
    → komet (build)
      → komet-desktop (build)
    → ike-lab-documents (content)
```

---

### 5.9 ws:feature-start

**Purpose:** Create a coordinated feature branch across the entire workspace.

**Parameters:**
- `featureName` (required) — e.g., "kec-april-refactor"

**Behavior:**
1. Verify workspace is clean (`ws:verify` passes, no uncommitted changes
   in any component).
2. **Branch the aggregator repository** to `feature/<featureName>`.
3. **Branch every component repository** to `feature/<featureName>`.
4. **Update version strings** in every component root POM. The version
   convention is: `<currentBaseVersion>-<featureName>-SNAPSHOT`. Use
   `mvn versions:set -DnewVersion=<new> -DgenerateBackupPoms=false` via
   subprocess. Example: if tinkar-core is at `0.7.0-SNAPSHOT` and
   featureName is `kec-april`, the new version is
   `0.7.0-kec-april-SNAPSHOT`.
5. **Update workspace.yaml** denormalized version fields to match the
   new component versions. Update branch fields to `feature/<featureName>`.
6. **Commit** workspace.yaml and aggregator POM on the aggregator's
   feature branch.
7. **Commit** version changes on each component's feature branch.
8. **Push** all branches (aggregator + all components).

**Postconditions:**
- Every repository (aggregator + components) is on `feature/<featureName>`.
- Every component POM version contains the feature qualifier.
- workspace.yaml on the feature branch describes the feature-branch state.
- workspace.yaml on main still describes main-branch state (unchanged).
- `ws:verify` passes on the feature branch.

**Components with no code changes:** The branch exists with just the
version bump commit. This is structurally correct—the version matches
the coordinated state. When the feature finishes, the merge for that
component is trivial.

---

### 5.10 ws:feature-finish

**Purpose:** Merge a feature branch back to its base branch.

**Parameters:**
- `featureName` (required)

**Behavior:**
1. Verify all components are on `feature/<featureName>` and clean.
2. For each component (in reverse dependency order):
   a. Checkout base branch (typically main).
   b. Merge `feature/<featureName>`.
   c. Update version back to base version (strip feature qualifier).
   d. Commit version change.
   e. Delete feature branch (local + remote).
3. Same for aggregator repository:
   a. Checkout main.
   b. Merge `feature/<featureName>`.
   c. Update workspace.yaml versions and branches back to main state.
   d. Commit.
   e. Delete feature branch.
4. Push all.

---

### 5.11 ws:release

**Purpose:** Release the entire workspace as a coordinated unit.

**Parameters:**
- `releaseVersion` (required) — e.g., "3" (monotonic) or "1.2.0"
- `nextVersion` (required) — e.g., "4-SNAPSHOT"

**Behavior:**
1. Verify workspace is clean and `ws:verify` passes.
2. Create `release/<releaseVersion>` branch on aggregator and all
   components.
3. Set versions to `releaseVersion` (strip SNAPSHOT) in all component
   root POMs.
4. Update workspace.yaml versions.
5. Build and verify entire workspace (`mvn clean verify` on aggregator).
6. Commit and tag `v<releaseVersion>` on each component and aggregator.
7. Deploy artifacts (`mvn deploy -DskipTests` on aggregator).
8. Push tags and release branches.
9. Merge release branches back to base branch.
10. Set versions to `nextVersion` on base branch.
11. Update workspace.yaml versions on base branch.
12. Commit and push.
13. Delete release branches.

**Thread safety:** If running with `-T` (parallel), the release mojo must
detect this and either fail fast or force `-T 1` for subprocess invocations
(particularly site plugin, which is not thread-safe). See existing
`ReleaseMojo.java` for the pattern.

---

### 5.12 ws:post-release

**Purpose:** Bump to next development version after a release. May be
folded into `ws:release` as a final step, or kept separate for manual
workflows.

**Parameters:**
- `nextVersion` (required)

**Behavior:**
1. Set version to `nextVersion` in all component root POMs.
2. Update workspace.yaml versions.
3. Commit and push.

---

### 5.13 ws:align

**Purpose:** Update cross-component dependency version references in POMs
to match current workspace component versions.

**Behavior:**
For each component, for each POM dependency that resolves to another
workspace component's published artifact set: if the declared version
doesn't match the target component's current root POM version, update it.

This is the mechanical cascade at the POM level—the workspace graph tells
you which POMs to touch, and the version to write comes from the upstream
component's root POM.

Commit changes per component.

---

## 6. Published Artifact Set — The Core Algorithm

This is the key data structure. For a given component (a cloned reactor
directory), the published artifact set is the set of all
`groupId:artifactId` pairs produced by the reactor.

**Construction:**
1. Parse root POM. Record its groupId:artifactId.
2. Walk `<subprojects>` recursively.
3. For each submodule POM, resolve its groupId (may be inherited from
   parent) and artifactId. Add to the set.
4. The set includes the root artifact.

**Usage:**
- `ws:add` uses it to discover build edges.
- `ws:verify` uses it to validate edges and detect missing edges.
- `ws:align` uses it to identify which POM dependencies are cross-component.

**Implementation note:** In a Mojo with access to `MavenSession`, this is
trivial: `session.getProjects()` for the component's reactor gives you
every `MavenProject` with fully resolved coordinates. Outside a reactor
build (e.g., during `ws:add` before the new component is in the reactor),
you may need to invoke Maven programmatically or parse POMs with the
`MavenProjectBuilder`.

---

## 7. Existing Code to Build On

### 7.1 ike-maven-plugin (existing)

Located at `ike-pipeline/ike-maven-plugin/`. Contains:
- `IkeHelpMojo.java` — lists available goals
- `AdocStudioMojo.java` — generates Adoc Studio sidecar files
- `ReleaseMojo.java` — current release implementation (subprocess-based,
  operates on single reactor). Contains thread-safety guard pattern.
- `ReleaseSupport.java` — shared release utilities
- `CheckpointMojo.java` — checkpoint creation

The workspace plugin supersedes `ReleaseMojo` for workspace-level releases.
The existing `ReleaseMojo` can be retained for single-reactor releases
outside workspace context.

### 7.2 Bash scripts (to be eliminated)

Located at `ike-pipeline/ike-build-tools/src/main/resources/scripts/`:
- `prepare-release.sh` — partially implemented
- `post-release.sh` — partially implemented
- `release-from-feature.sh` — implemented

All logic moves into Java Mojos. These scripts are deprecated.

### 7.3 ws bash script (to be eliminated)

Located at `~/ike-dev/ike-workspace/ws`. Bash 5 CLI with:
- `ws init` — clone from workspace.yaml
- `ws verify` — consistency checks
- `ws graph` — dependency graph visualization
- `ws cascade` — propagation set computation
- `ws status` — cross-repo git status
- `ws build` — reactor build by group

All logic moves into the workspace Maven plugin. The bash script is
deprecated.

---

## 8. Build/Test Strategy

### 8.1 Unit Tests

- Workspace model parsing (YAML → Java model → YAML round-trip)
- Published artifact set construction from mock POM structures
- Dependency edge discovery algorithm
- Single-version invariant validation
- Cascade/propagation BFS
- Version manipulation (add/strip feature qualifier, SNAPSHOT handling)

### 8.2 Integration Tests

- `ws:create` produces valid workspace
- `ws:add` with a test component discovers correct edges
- `ws:verify` detects planted version drift, missing edges, version
  invariant violations
- `ws:feature-start` / `ws:feature-finish` round-trip on test repos

Use `maven-invoker-plugin` or temporary git repos created in test setup.

---

## 9. Resolved Design Questions

1. **JGit vs CLI git:** Start with CLI git via ProcessBuilder. Simpler,
   matches what developers use, no large dependency. Migrate to JGit later
   if cross-platform issues arise. Git on PATH is a reasonable prerequisite.

2. **POM parsing outside reactor:** Use Maven's own `ProjectBuilder`
   component (injectable via `@Component`). It can build a fully resolved
   `MavenProject` from a POM file outside a reactor session—handles parent
   inheritance, property interpolation, everything. Do NOT write custom XML
   parsing. Reuse Maven's infrastructure.

3. **POM manipulation strategy:** Hybrid approach.
   - **Version changes:** Use `mvn versions:set` via subprocess. Well-tested,
     handles submodule walk correctly.
   - **Consumer POM workarounds:** Direct XML manipulation where needed to
     work around Maven 4 bugs (e.g., replacing property references with
     literal versions in consumer POMs). This pattern is already established
     in the existing codebase.
   - **Rule:** Don't invent new direct-manipulation paths where a subprocess
     invocation works.

4. **Interactive confirmation:** `ws:add` presents discovered edges for
   confirmation. Use System.in prompt for interactive use, with a
   `--non-interactive` flag that accepts all discovered edges by default
   (verify catches errors later). Recommendation: (a) for interactive use,
   with `--non-interactive` for CI.

5. **Aggregator POM ordering:** The `<subprojects>` list should be in
   dependency order. `ws:verify` warns if out of order. `ws:add`,
   `ws:remove`, and `ws:fix` update the ordering.

## 10. Open Questions for Implementor

1. **Specific Maven 4 consumer POM workarounds:** The existing codebase has
   direct POM XML manipulation for replacing variables with literal versions
   in consumer POMs. The implementor should examine `ReleaseMojo.java` and
   `ReleaseSupport.java` to understand which workarounds are in play and
   whether they apply to workspace-level operations.

2. **ws:init as a Mojo vs. standalone:** `ws:init` runs before any
   component is cloned, which means there's no reactor to run in. It may
   need to be a standalone Java CLI entry point (main method in
   `ike-plugin-common`) invoked via `java -jar` or a Maven exec:java
   invocation, rather than a Mojo. Alternatively, run it from the
   aggregator's reactor (which exists even without components cloned) using
   `requiresProject = false` or `aggregator = true` semantics.

---

## 11. Relationship to CLAUDE.md and IKE-MAVEN.md

Before modifying any POM in the ike-pipeline reactor, read:
- `CLAUDE.md` at the project root
- `.claude/standards/IKE-MAVEN.md` (unpacked by `mvn validate`)

These contain binding conventions for POM structure, element naming,
namespace, and version management that apply to all IKE projects.

---

## 12. Implementation Priority

Recommended order:

1. `ike-plugin-common` — model classes, YAML I/O, POM analysis utilities
2. `ws:create` — simplest goal, proves the module structure works
3. `ws:init` — realization from spec, exercises YAML parsing and git clone
4. `ws:add` — core algorithm (published artifact set, edge discovery)
5. `ws:verify` — validates everything, exercises all analysis code
6. `ws:status` — simple, immediately useful
7. `ws:cascade` — graph traversal, depends on model being solid
8. `ws:feature-start` — first VCS-mutating goal, highest risk
9. `ws:feature-finish` — inverse of feature-start
10. `ws:release` / `ws:post-release` — combines version manipulation + VCS
11. `ws:align` — mechanical POM updates
12. `ws:fix` — trivial once verify works
13. `ws:remove` — rarely used, low priority

---

## 13. workspace.yaml JSON Schema

This schema is the contract for `ike-plugin-common`'s YAML parser. It
defines the structure that all workspace goals depend on.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "IKE Workspace Manifest",
  "type": "object",
  "required": ["schema-version", "components"],
  "properties": {
    "schema-version": {
      "type": "string",
      "const": "1.0"
    },
    "generated": {
      "type": "string",
      "description": "Date the manifest was last generated or updated"
    },
    "defaults": {
      "type": "object",
      "properties": {
        "org": {
          "type": "string",
          "format": "uri",
          "description": "Default GitHub org URL"
        },
        "branch": {
          "type": "string",
          "description": "Default branch for components that don't specify one"
        }
      }
    },
    "component-types": {
      "type": "object",
      "additionalProperties": {
        "type": "object",
        "properties": {
          "description": { "type": "string" },
          "build-command": { "type": "string" },
          "checkpoint-mechanism": {
            "type": "string",
            "enum": ["git-tag", "composite"]
          }
        }
      }
    },
    "components": {
      "type": "object",
      "additionalProperties": {
        "$ref": "#/$defs/component"
      }
    },
    "groups": {
      "type": "object",
      "additionalProperties": {
        "type": "array",
        "items": { "type": "string" },
        "description": "List of component names or group names (recursive)"
      }
    }
  },
  "$defs": {
    "component": {
      "type": "object",
      "required": ["type", "repo"],
      "properties": {
        "type": {
          "type": "string",
          "enum": ["software", "document", "knowledge-source",
                   "infrastructure", "template"]
        },
        "description": { "type": "string" },
        "repo": {
          "type": "string",
          "format": "uri",
          "description": "Git remote URL (NOVEL — YAML authoritative)"
        },
        "branch": {
          "type": "string",
          "description": "Branch to clone (NOVEL — YAML authoritative)"
        },
        "version": {
          "type": ["string", "null"],
          "description": "Root POM version (DENORMALIZED — verified against POM)"
        },
        "groupId": {
          "type": ["string", "null"],
          "description": "Primary groupId (DENORMALIZED — documentary only, not used as match key)"
        },
        "depends-on": {
          "type": "array",
          "items": { "$ref": "#/$defs/dependency-edge" }
        },
        "notes": {
          "type": "string",
          "description": "Rationale, migration annotations (NOVEL)"
        }
      }
    },
    "dependency-edge": {
      "type": "object",
      "required": ["component", "relationship"],
      "properties": {
        "component": {
          "type": "string",
          "description": "Target component name"
        },
        "relationship": {
          "type": "string",
          "enum": ["build", "content", "tooling"],
          "description": "build = POM dependency (DENORMALIZED, verified). content/tooling = architectural assertion (NOVEL, not mechanically verifiable)."
        }
      }
    }
  }
}
```

The schema annotations (`NOVEL`, `DENORMALIZED`) indicate which fields
`ws:verify` checks against POMs and which it treats as architect-authored
truth. `ws:fix` / `ws:verify --update` only modifies DENORMALIZED fields.

---

## 14. Summary of Key Decisions

| Decision | Resolution | Rationale |
|----------|-----------|-----------|
| Bash vs Java | Java only | Cross-platform requirement |
| Plugin count | 2 plugins + 1 shared lib | ws: for workspace, ike: for build-time |
| SCM goals in workspace plugin | Yes | All SCM ops need workspace graph |
| Aggregator branched with components | Yes | Spec must match realization on every branch |
| groupId as match key | No — use published artifact set | Components publish under multiple groupIds |
| Denormalized YAML fields | Allowed, verified | Readability justifies sync cost if mechanically checked |
| Single-version invariant | Enforced by verify, required by add | Fundamental to component-as-unit abstraction |
| ${revision} | Never | Builds must reproduce from SCM alone |
| Feature branch version convention | `<base>-<featureName>-SNAPSHOT` | Matches established convention |
| Feature branches on no-change components | Version-bump commit only | Structurally correct, trivial merge |
| Version mismatch during ws:add | Report, workspace version wins | Reactor resolves correctly; POM version is stale |
| ws:verify exit code | Nonzero on FAIL | Usable as CI gate or pre-commit hook |
| ws:verify --update scope | POM-derivable fields only | Novel fields are architect-authored |
| POM parsing | Maven's ProjectBuilder | Reuse Maven's infrastructure, don't write custom parsers |
| Version manipulation | `mvn versions:set` subprocess | Well-tested, handles submodule walk |
| Direct POM XML manipulation | Only for Maven 4 consumer POM workarounds | Established pattern, don't extend to new cases |
| Git operations | CLI git via ProcessBuilder | Simpler, matches developer tooling, JGit later if needed |
