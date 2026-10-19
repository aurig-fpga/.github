# 0001 — Project manifest contract

- Status: Accepted
- Date: 2026-10-19
- Repositories: aurig-schema (new), aurig-core, aurig-build, aurig-lint, aurig-doc, aurig-sentinel, .github

## Context

All AURIG tools work from the same project manifest (`project.yaml`), but they handle it differently:

- aurig-core ships a JSON Schema (`manifest-v1.json`) and a Tcl pipeline that reads, normalises and validates it (`schema::load_manifest`), then resolves sources and the top-level entity (`schema::scan_project`).
- aurig-build ships a copy of the same schema with its own Python pipeline.
- aurig-lint and aurig-doc read the raw YAML through `util::readYaml` and `util::collect_project_files`. They do not validate against the schema.
- aurig-sentinel passes the manifest path to the lint and doc runners and does not read the manifest itself.

The result is one file format with several behaviours:

- The Tcl and Python validators accept and reject different inputs. For example, `top: true`, a numeric `schema_version` and a multi-word scalar `tool` are accepted by the Tcl validator and rejected by the Python one. YAML-to-Tcl conversion loses type information: null, empty string, empty list and empty mapping become the same Tcl value, and some lists, mappings and strings become indistinguishable.
- `.local.yaml` overlays are applied by the aurig-core schema pipeline and by aurig-build. aurig-lint and aurig-doc do not apply them; their collector only stops with an error when the overlay alone sets `project_root`. The two pipelines that apply them merge differently: the Tcl merge skips empty overlay values, while the Python merge replaces with empty scalars and lists and merges empty mappings recursively.
- `scan_project` and aurig-build inventory only the `rtl` and `sim` file sets. The util collector used by lint and doc inventories every group.
- aurig-lint and aurig-doc do not consistently enforce the contract. They accept, for example, a missing `schema_version`, a scalar `tool`, a `top_file` that does not exist, and a top-level entity declared in more than one file.
- The two schema copies differ in line endings, and parity between them is checked by hand.

The schema also requires `top`, which excludes a legitimate lint/doc use case: a library or flat file list with no top-level entity.

## Decision

### 1. One contract, owned by aurig-schema

A dedicated repository, `aurig-fpga/aurig-schema`, holds the single source of the manifest contract: `manifest-v1.json` and the conformance fixtures (section 3).

The contract lives outside the tools because aurig-build does not depend on aurig-core, and because the schema is versioned on its own: a schema revision is published by tagging aurig-schema.

aurig-core and aurig-build each vendor a copy from an aurig-schema tag. A tool adopts a schema revision in one of its own releases by updating its vendored copy. The CI of each tool compares its copy against the tag it declares, JSON-normalised so that formatting differences do not count.

All copies are stored with LF line endings, enforced through `.gitattributes`.

### 2. Every tool reads manifests through the schema pipeline

- aurig-lint and aurig-doc use `schema::load_manifest` / `schema::scan_project` from aurig-core.
- aurig-build uses its Python pipeline.
- aurig-sentinel passes the manifest path through and does not interpret its contents.

aurig-core preserves YAML node types as far as its YAML reader allows, so that the Tcl and Python pipelines see the same values. The rules of this decision are chosen so that they do not depend on distinctions the Tcl reader cannot make (for example an empty mapping versus an empty string, see section 8). Where the two pipelines still cannot agree, the specification in aurig-schema lists the inputs concerned explicitly.

### 3. Conformance in three layers

Shared conformance applies to canonical manifests: manifests that use only the keys and shapes defined by the schema revision a tool declares. Tool-specific checks are separate acceptance layers on top of it: the validation of a tool's own namespace (section 9), the transition behaviour of aurig-lint and aurig-doc (sections 6, 7, 9) and the synthesis readiness check of aurig-build (section 4).

The fixtures in aurig-schema are organised in three layers, each with its own reference:

1. **Schema validation.** Input manifest → accept or reject. The CI of aurig-schema runs every fixture through a standard JSON Schema Draft 2020-12 validator, which is the reference. aurig-core and aurig-build must give the same outcome.
2. **Overlay and normalisation.** Base manifest, optional overlay → normalised manifest and warnings. The expected output is part of the fixture. aurig-core and aurig-build must produce the same normalised manifest, compared as JSON.
3. **Project resolution.** Manifest and file tree → source inventory and top resolution result (section 5). aurig-core is the implementation, used by aurig-lint and aurig-doc.

aurig-core runs all three layers in CI; aurig-build runs layers 1 and 2. Any divergence from a fixture is a defect, except on the inputs the specification lists under section 2.

The transition behaviour of aurig-lint and aurig-doc is a mode of the aurig-core pipeline that those tools select when they call it. It runs after the overlay merge and before validation: it removes the legacy keys (section 9) and reports missing required keys (section 6) as warnings. Other callers, and fixture layers 1 and 2, get strict validation. aurig-lint and aurig-doc 0.3.0 stop selecting the mode; aurig-core removes it only after both have been released.

### 4. `top` is optional

- **`top` absent:** no-top mode. Every file in the canonical file sets is analysed. This is the normal mode for libraries.
- **`top` present:** hierarchy mode, with top resolution as in section 5.

Schema validity does not imply synthesis readiness. Before synthesis, aurig-build checks that `top` is present, that a given `top_file` exists, and its other synthesis prerequisites, outside schema validation. It does not scan HDL for the entity declaration: the vendor tool resolves the top-level entity. A manifest without `top` remains valid for tools and flows that do not need one.

This is a validation relaxation within schema version 1. `schema_version` stays `"1"`.

### 5. Top resolution

When `top` is present, the tools that analyse HDL (aurig-lint and aurig-doc, through aurig-core's `scan_project`) resolve it as follows:

- A relative `top_file` is anchored at the resolved `project_root`.
- **`top_file` given:** the file must exist; otherwise it is an error. If it is a VHDL file (`.vhd`, `.vhdl`), it must declare `top`; otherwise it is an error. Other file types (for example Verilog or generated IP) are accepted without a declaration check. aurig-lint and aurig-doc report such a top as an unresolved hierarchy root.
- **No `top_file`:** `top` must be declared in exactly one VHDL file of the `rtl` set. No declaration is an error. Several declarations are an error that names the files.
- **`top_file` without `top`:** error.

This is stricter than `scan_project` today, which warns when no file declares `top` and accepts an explicit `top_file` it cannot verify.

### 6. Required keys

`schema_version` and `project_name` are required. aurig-build already enforces both. In aurig-lint and aurig-doc a missing key is a warning in the transition releases and an error afterwards (section 11).

### 7. File sets

The canonical groups are `rtl` and `sim`. The existing normalisations stay: `ip` is folded into `rtl`, and `constraints` is routed to the board constraint lists.

Any other group is a warning in the transition releases and an error afterwards. During the transition, aurig-lint and aurig-doc keep analysing the files of such groups, so no file silently drops out of analysis.

### 8. Local overlay

Every tool applies the sibling overlay `<stem>.local.<ext>` before normalisation and validation. The merge is defined as follows:

- The overlay root must be a mapping.
- Where the overlay holds a non-empty mapping and the base holds a mapping for the same key, the two mappings merge recursively, key by key.
- Any other overlay value replaces the base value for that key entirely: scalars, lists and every empty value (null, empty string, empty list, empty mapping). Lists are never merged.

Treating every empty value the same way keeps the rule identical in Tcl and Python, whose YAML readers do not all distinguish an empty mapping from an empty string.
- Keys absent from the overlay keep their base value.

The merged result is then normalised and validated.

### 9. Unknown keys, tool namespaces and legacy keys

Tool-specific settings live under a namespace key declared in the schema (`lint:`, `doc:`). The schema allows any content there; the owning tool validates it.

Any other unknown top-level key is a warning that names the closest known key.

The legacy aurig-doc metadata keys `fpga_device`, `fpga_family`, `tool_version` and a scalar `tool` keep working in the transition releases. aurig-doc removes them before validation, uses their values for display only, and emits a warning that shows the equivalent `device:` / `tool:` block. They are not migrated automatically, because a partial `device:` block would itself be invalid. After the transition they are errors.

### 10. Formats

Manifests are YAML. JSON is accepted as a YAML subset. The format is decided by parsing the content, not by the file extension.

INI manifests are deprecated starting with the releases that implement this decision:

- aurig-doc stops accepting INI manifests in its first transition release.
- aurig-core's `readIni`, `ini2yaml` and `yaml2ini` are deprecated in the same release and removed in a later one.

### 11. Transition releases

The transition is tied to named releases of each Tcl and Python tool (aurig-core, aurig-build, aurig-lint, aurig-doc):

- **0.2.0** implements this decision. Everything this document marks as "a warning in the transition releases" is a warning.
- **0.3.0** turns those warnings into errors.

A 0.3.0 release is published only after the corresponding 0.2.0 has been released.

### 12. Failures

Validation errors and top-resolution errors are fatal. In aurig-lint and aurig-doc, an empty source inventory after expansion is also fatal; aurig-lint's explicit `-allow_empty` option keeps turning that one failure into a warning.

Library functions and public procedures raise errors. Only the command-line runners map them to exit codes.

### 13. Detailed specification

This decision fixes the principles. The detailed rules live in a specification document in aurig-schema, written together with the conformance fixtures during implementation and reviewed with each change. The fixtures are its executable form. They cover at least:

- the accepted YAML profile: duplicate keys, multiple documents, empty document versus null root;
- warning comparison between the Tcl and Python pipelines in layer 2;
- top resolution edge cases: empty `top_file`, VHDL files that fail to parse, case-insensitive matching of VHDL names, a `top_file` outside the inventory;
- inventory edge cases: generated or IP-only projects, unmatched patterns, excluded files, zero recognised design units;
- precedence between legacy and canonical keys during the transition, and the content of the "equivalent block" shown in legacy-key warnings (no invented values);
- any input on which the Tcl and Python pipelines cannot agree.

This decision is amended only if a principle changes.

## Consequences

For users of aurig-lint and aurig-doc:

- Manifests are validated against the schema, which aurig-lint and aurig-doc do not do today. Apart from the transition cases below, invalid manifests are rejected.
- A `top_file` that does not exist, a VHDL `top_file` that does not declare `top`, a `top_file` without `top`, a top-level entity declared in several files, and a `top` that no file declares become errors.
- An empty source inventory is an error. aurig-doc today writes empty documentation and exits 0 when `top` is absent; aurig-lint already rejects it unless `-allow_empty` is given.
- `.local.yaml` overlays, ignored until now, are applied and can change paths, names, settings and validity. An empty overlay value now replaces the base value, where aurig-core's merge used to skip it.
- The normalisations of aurig-core apply: deprecated aliases are rewritten with a warning, and removed values are rejected (for example removed synthesis tools).
- A missing `schema_version` or `project_name`, non-canonical file-set groups and legacy aurig-doc keys produce warnings in 0.2.0 and errors in 0.3.0.
- A JSON manifest is parsed as YAML. aurig-doc previously sent non-YAML extensions to its INI reader.
- aurig-doc no longer accepts INI manifests.
- aurig-doc gains a runtime dependency on tcllib `json`, needed by the schema validator.

For users of aurig-build:

- Manifests without `top` pass schema validation. Synthesis still requires `top`, now through an explicit readiness check that also verifies a given `top_file` exists.
- Overlay handling follows section 8: an empty mapping in an overlay now replaces the base value instead of merging.

For maintainers:

- aurig-core and aurig-build accept and reject the same canonical manifests and normalise them identically, apart from the inputs the specification lists under section 2, and overlays mean the same thing everywhere, as pinned by the conformance fixtures. Tool-specific acceptance layers are documented separately.
- The Tcl validator rejects values it used to accept loosely (for example a numeric `schema_version`) where its reader can tell them apart. aurig-build already rejects them.
- Schema revisions are published in aurig-schema and adopted by each tool in its own release.

## Implementation order

Each step is one or more atomic PRs. Existing tests are updated in the same PR as the behaviour they cover. A step starts after the previous step is merged.

1. **aurig-schema**
   - Create the repository with the standard governance files.
   - Move `manifest-v1.json` from aurig-core, with LF line endings.
   - Make `top` optional and declare the tool namespaces.
   - Add the three fixture layers, starting from the existing aurig-core and aurig-build manifest fixtures, and the reference-validator CI for layer 1.
   - Start the specification document (section 13).
   - Tag the first release.
2. **aurig-core 0.2.0**
   - Preserve YAML node types as far as the reader allows (section 2); record remaining limits in the specification.
   - Vendor the schema from the aurig-schema tag; add the parity check.
   - Implement the overlay merge (section 8), top resolution (section 5), closest-key warnings, and the transition mode of section 3, applied after the overlay merge and before validation.
   - Run all three fixture layers in CI, including them in the CI test list.
   - Correct comments that describe the schema copies or the lint/doc consumers inaccurately.
   - Deprecate the INI readers and converters.
   - Release 0.2.0.
3. **aurig-build 0.2.0**
   - Vendor the schema from the aurig-schema tag; add the parity check.
   - Align the overlay merge with section 8.
   - Name the closest known key in unknown top-level key warnings (section 9).
   - Raise `ManifestError` instead of exiting when a manifest or overlay root is not a mapping (sections 8, 12).
   - Add the synthesis readiness check (`top` present, given `top_file` exists).
   - Run fixture layers 1 and 2 in CI.
   - Add `schema_version` to the README and configuration guide examples.
   - Release 0.2.0.
4. **aurig-lint 0.2.0**
   - Read manifests through the aurig-core pipeline, with the transition behaviour of sections 6, 7 and 9.
   - Validate the content of the `lint:` namespace (section 9). `lint.excludes` is already under `lint:`.
   - Release 0.2.0.
5. **aurig-doc 0.2.0**
   - Read manifests through the aurig-core pipeline, with the transition behaviour of sections 6, 7 and 9, including the legacy-key handling.
   - Fail on an empty source inventory (section 12).
   - Remove INI manifest support.
   - Add `schema_version` to the test fixtures.
   - Release 0.2.0.
6. **.github**
   - Update the organisation profile's description of the shared manifest.
7. **0.3.0 releases** of each tool, turning the transition warnings into errors.

Defects found during the audit that are not part of this contract, such as exit codes in the build simulation wrappers and the collector's format detection, are tracked as separate issues in their repositories.