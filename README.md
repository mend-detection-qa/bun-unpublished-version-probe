# bun-unpublished-version-probe

Probe #23 — Tier 5 (edge case), pattern E1 (`unpublished-version`).

## Pattern

**E1 (`unpublished-version`)** — The lockfile references a real npm package version that carries
a registry-level deprecation flag. The probe documents Mend's behavior when the npm registry
manifest response includes a `deprecated` field for the pinned version.

### Sub-case: deprecated-but-still-resolvable

This probe exercises the "deprecated-but-still-resolvable" sub-case of E1:

- The package (`request@2.88.2`) was **deprecated** — not unpublished.
- The tarball and manifest are still accessible from the npm registry.
- The npm registry manifest for this version carries:
  `"deprecated": "request has been deprecated, see https://github.com/request/request/issues/3142"`
- The lockfile constraint is declared as an **exact pin** (`"2.88.2"`) — no caret, no tilde —
  making version-substitution detectable by a simple equality check.

## Why standalone

The deprecated/unpublished version failure mode is its own code path in any registry-aware
dep-tree resolver:

1. A standard-registry dep triggers one code path.
2. A deprecated-registry dep triggers the same code path **plus** the registry's deprecation
   metadata in the manifest response. If the parser does not handle the `deprecated` field
   gracefully, it may drop the dep, substitute a version, or corrupt the metadata for that node.

Bundling this probe with any other pattern would obscure which failure caused the diff.
The probe must stand alone so that "request is missing" is unambiguously an E1 failure,
not an unrelated parser regression from a bundled pattern.

## Mend config

No `.whitesource` file is emitted with this probe.

Rationale: Bun is NOT in the Mend `install-tool` supported list. The `scanSettings.versioning`
block cannot pin a Bun toolchain version. Detection is lockfile-driven only — Mend reads
`bun.lock` (text JSONC format, Bun 1.1+) statically without executing any install step.
This limitation is tracked in `docs/BUN_COVERAGE_PLAN.md §4` and is the subject of
probe #24 (`bun-not-in-install-tool-probe`).

## Real package rationale

### Why `request@2.88.2`

`request` was chosen for the following reasons:

1. **Documented, verifiable deprecation.** The `request` package was formally deprecated on
   2020-02-11 via `npm deprecate`. The deprecation announcement is at
   [request/request#3142](https://github.com/request/request/issues/3142). The deprecation
   message is part of the registry response for every `request` version from that date forward.

2. **Still resolvable.** Unlike a truly unpublished package (e.g. `node-ipc@10.1.1` which was
   pulled during the 2022 protestware incident), `request` is deprecated-but-not-unpublished.
   This is the safer probe choice: the tarball and manifest remain accessible from the npm CDN,
   so the lockfile is reproducible and there is no risk of probe breakage due to registry
   unavailability.

3. **Exact version pin.** `request@2.88.2` is the last version published before deprecation
   was applied package-wide. Pinning the exact version (`"2.88.2"`) — not a range — makes
   Mend version substitution immediately detectable: any version other than `2.88.2` in the
   reported tree is a failure.

4. **Rich transitive graph.** `request@2.88.2` has 20 direct dependencies with further
   transitives. This exercises Mend's ability to resolve the full tree even when the root dep
   is deprecated — a shallow tree (one or two deps) would not catch failures where Mend
   partially resolves the tree before abandoning the deprecated root.

5. **Historically significant.** The `request` deprecation is well-documented in the npm
   ecosystem and commonly referenced in dependency audit tooling discussions. Its presence
   in a probe is immediately recognizable to reviewers.

### Why not `node-ipc@10.1.1`

`node-ipc@10.1.1` and `10.1.2` were unpublished after the 2022 protestware incident. While
this would target the "truly unpublished" sub-case of E1, the versions are no longer on the
registry at all — a lockfile referencing them cannot be installed and the probe itself would
fail in a way that obscures whether it was a Mend failure or a registry unavailability. The
`request@2.88.2` approach is reproducible and stable.

## Dependency graph

```
bun-unpublished-version-probe@0.1.0
  └── request@2.88.2 (DEPRECATED)
        ├── qs@6.5.3
        ├── aws4@1.13.2
        ├── uuid@3.4.0
        ├── extend@3.0.2
        ├── caseless@0.12.0
        ├── isstream@0.1.2
        ├── aws-sign2@0.7.0
        ├── form-data@2.3.3
        │     ├── asynckit@0.4.0
        │     ├── mime-types@2.1.35  (shared)
        │     └── combined-stream@1.0.8  (shared)
        ├── mime-types@2.1.35
        │     └── mime-db@1.52.0
        ├── oauth-sign@0.9.0
        ├── safe-buffer@5.2.1
        ├── tough-cookie@2.5.0
        │     ├── psl@1.15.0
        │     └── punycode@2.3.1
        ├── tunnel-agent@0.6.0
        ├── forever-agent@0.6.1
        ├── har-validator@5.1.5
        │     ├── ajv@6.12.6
        │     │     └── [uri-js, fast-deep-equal, json-schema-traverse, fast-json-stable-stringify]
        │     └── har-schema@2.0.0
        ├── is-typedarray@1.0.0
        ├── http-signature@1.2.0
        │     ├── assert-plus@1.0.0
        │     ├── jsprim@2.0.2
        │     │     └── [assert-plus, extsprintf, json-schema, verror]
        │     └── sshpk@1.18.0
        │           └── [asn1, assert-plus, bcrypt-pbkdf, dashdash, ecc-jsbn, getpass, jsbn, safer-buffer, tweetnacl]
        ├── combined-stream@1.0.8
        │     └── delayed-stream@1.0.0
        ├── performance-now@2.1.0
        └── json-stringify-safe@5.0.1
```

**Deps in lockfile:** 1 direct (request) + 20 direct-of-request + 12 first-level transitives
= 33 entries in `bun.lock`.

Deeper transitives (ajv's 4 deps, jsprim's 4 deps, sshpk's 9 deps) are documented in
`expected-tree.json::notes.transitive_depth_truncation` but are omitted from the lockfile
and `expected-tree.json::packages` for tractability. A production scan will resolve them;
the comparator should use `gte` tolerance with N-30% for total dep counts.

## Failure modes

| # | Failure | Observable symptom |
|---|---|---|
| 1 | Version substitution | Mend reports a version of `request` other than `2.88.2` (e.g. latest non-deprecated) |
| 2 | Dep dropped entirely | `request` is absent from the reported dep tree; all 20+ transitives also missing |
| 3 | Stale cache hit | Transitives resolved at wrong versions (e.g. `qs@6.5.2` instead of `6.5.3`) |
| 4 | Stale vulnerability data | Vulnerability findings for `request@2.88.2` omit post-2020 CVEs |
| 5 | Dep-type reclassification | `request` appears in tree but is excluded from vuln scanning |

## Resolver notes (UA analog)

The UA javascript resolver (npm fallback path) is the closest analog for Bun's detection.
Key behaviors relevant to this probe:

- The UA does NOT have native Bun support. It parses `bun.lock` only if the lockfile detection
  logic recognizes the JSONC format.
- The UA npm resolver fetches each dep's registry manifest to resolve versions and checksums.
  If the manifest includes a `deprecated` field, the UA resolver should handle it gracefully
  (preserve the tree entry, do not substitute). Historical behavior in npm-based resolvers
  has varied — some versions silently downgrade or skip deprecated entries.
- Lockfile-driven detection (Bun's only path): Mend reads resolved versions from `bun.lock`
  directly. It should NOT re-resolve versions from the registry during lockfile parsing — only
  the version recorded in the lockfile matters. If Mend does perform a registry re-resolution
  step (as some resolvers do for vulnerability metadata enrichment), the deprecated flag in
  the manifest response becomes relevant.

This probe is exploratory for the UA Bun path — the upstream resolver file does not document
Bun-specific handling of deprecated packages. See `docs/BUN_COVERAGE_PLAN.md §9` open questions.

## Probe metadata

- Pattern: E1 (`unpublished-version`)
- Target: `local`
- Bun version under test: `1.1.30` (text `bun.lock` format, `lockfileVersion: 1`)
- Lockfile format: `bun.lock` (JSONC, Bun 1.1+)
- Install-tool key: NOT in install-tool list — no `.whitesource` emitted
- `pm_version_tested` in `expected-tree.json`: `1.1.30`
- Real package: `request@2.88.2` — deprecated 2020-02-11, still resolvable
- Deprecation evidence: `npm view request@2.88.2 deprecated` → `"request has been deprecated, see https://github.com/request/request/issues/3142"`

Tracked in: `docs/BUN_COVERAGE_PLAN.md §11.5` entry #23