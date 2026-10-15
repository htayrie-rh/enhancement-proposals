# OSAC-6204 — Administrator-only metadata test plan

## Overview

- **Source:** [PRD](prd.md), its co-located clarifications, and [design draft](design.md).
- **Status:** Planned behavioral coverage; no implementation tests were executed during drafting or this revision.
- **Coverage:** 29 cases, five of five In Scope bullets, and 11 of 11 interface changes. Twenty-seven cases are planned for automation; two are manual documentation reviews. Browser execution and exhaustive deployed fixtures still require the work recorded under G-1 and G-2.
- **Target:** 0.4 Tech Preview. The networking metadata contract question must be resolved before delivery.

The PRD uses In Scope bullets rather than FR/NFR identifiers. IS-1 through IS-5 below are local traceability anchors to those exact bullets, not new requirements. Implementation details such as selected-map replacement, reserved-entry preservation, capability preflight, transaction rollback, and filter projection are testable design mechanisms for those requirements.

## Requirements and interface coverage

| Anchor | PRD In Scope requirement | Primary cases | Design interface coverage |
|---|---|---|---|
| IS-1 | Both admin personas read/add/change/remove reserved metadata through private API, console, and CLI on all metadata-bearing resources | TC-IS1-01–09 | IC-1, IC-2, IC-3, IC-7, IC-8, IC-9, IC-11 |
| IS-2 | Tenant API responses and console omit reserved keys | TC-IS2-01–06 | IC-1, IC-2, IC-5, IC-6, IC-7, IC-8, IC-9 |
| IS-3 | Tenant add/change/remove attempts fail the whole request clearly | TC-IS3-01–06, TC-IS3-08 | IC-1, IC-3, IC-4, IC-7, IC-8, IC-9 |
| IS-4 | Ordinary metadata keeps its existing behavior | TC-IS4-01–05 | IC-1, IC-2, IC-3, IC-4, IC-5, IC-6, IC-8, IC-9, IC-11 |
| IS-5 | Document exact reserved prefix and matching behavior | TC-IS5-01–02 | IC-1, IC-10, IC-11 |

Under confirmed D6, tenant map omission always preserves stored reserved entries. Named forbidden CLI/console operations fail locally before any Update; supplied reserved entries and accepted explicit protected mask paths fail on the server. A whole-map request cannot reveal a removal target that a client has already discarded.

Cases can provide evidence for several bullets; their primary anchor determines numbering only. The case metadata below lists every tested interface change. IC-1 through IC-11 refer to the design's API Extensions table.

## Shared fixtures and oracles

Use two accounts representing Cloud Provider Admin and Cloud Infrastructure Admin with the existing platform-admin authorization, plus Tenant Admin and Tenant User accounts with their existing roles. No new persona roles or account-provisioning workflow are needed. Include a second tenant for isolation controls and the existing restricted private CSI identity. A role display string alone is not authority. Both admin personas must pass the same matrix; tenant cases run as both tenant personas where the method is authorized for ordinary use.

Seed valid stored dependencies for every object type. Use ordinary keys such as `example.org/environment` and reserved keys such as `admin.osac.openshift.io/tracking-id` in both maps, with unique marker values to detect disclosure. Include empty strings, multiple entries, and objects without protected entries. Do not rely on external provider provisioning to test a metadata edit; explicitly record seeded lifecycle/readiness states and the fact that no provider provisioning boundary is exercised.

Before each rejected or concurrent request, obtain an authorized private snapshot of the complete stored object, maps, and metadata.version. Afterward compare the complete object and relevant side-effect tables; checking only the public response cannot detect loss or partial writes. Pair public tenant reads with private administrator reads. Capture errors without copying protected values into logs or test reports.

### Persistent resource inventory

| Family | Types requiring both admin personas, both maps, and private Get/List/Update coverage |
|---|---|
| Add-ons | AddOnOperator |
| Clusters | Cluster, ClusterTemplate, ClusterCatalogItem, ClusterVersion |
| Compute | ComputeInstance, ComputeInstanceTemplate, ComputeInstanceCatalogItem, InstanceType, DiskImage |
| Bare metal | BareMetalInstance, BareMetalInstanceTemplate, BareMetalInstanceCatalogItem, BareMetalInstanceType |
| Networking | NetworkClass, VirtualNetwork, Subnet, SecurityGroup, ExternalIPPool, ExternalIP, ExternalIPAttachment, NATGateway, FabricDomain |
| Storage | StorageBackend, StorageTier, Volume |
| Platform / identity | Hub, Tenant, User, IdentityProvider, Project, ProjectMembership, Role, RoleBinding, Secret |

Total: 35 persistent types. Hub, NetworkClass, StorageBackend, and Volume are private-only; do not invent public methods for them. For each type, test supported Create responses where available and all existing private Update methods. SelfSubjectAccessReview is the additional ephemeral Metadata-bearing message, covered by TC-IS2-06 without persistent CRUD. Inventory completeness is an explicit test assertion, not a sample of familiar VM/network resources.

## Execution and evidence matrix

Commands below describe implementation-phase execution; none was run to validate this draft. Paths are relative to the workspace. Proposed files and runners are clearly identified. Real database, broker, authorization, gateway, proxy, and browser boundaries require their respective evidence; unit doubles do not substitute for them.

| Tier / owner | Suite and proposed additions | Working directory / execution | Real versus mocked dependencies; gaps |
|---|---|---|---|
| Unit / DEV | Existing fulfillment auth, validation, mapper, mask/filter, CLI, and transaction tests; add reserved predicate, capability, raw-input/explicit-path rejection, map selection/merge, client operation checks, and rollback-order cases | `osac/fulfillment-service`: `ginkgo run -r internal` (use workspace-local GOCACHE/GOTMPDIR; heavy suites sequentially) | Policy/parser/transport fixtures and test doubles. This broad command also reaches the component suites below; report their real dependencies separately. |
| Component integration / DEV | Existing `internal/servers/servers_suite_test.go` and server tests; add full candidate, networking metadata compatibility and lifecycle restrictions, inventory, concurrency, and error rollback cases | `osac/fulfillment-service`: `ginkgo run --timeout 10m internal/servers` | Real PostgreSQL from database.Container; in-process RPC and mocked tenancy/attribution. This does not establish deployed Keycloak, REST, proxy, or provider behavior. |
| Events component integration / DEV | Existing `internal/servers/events_server_test.go`; add nested projection and filter privacy cases | `osac/fulfillment-service`: same server-suite command, executed once for the combined suite | Real Kafka via Kafka.Container, bufconn RPC, and mocked tenancy. Record broker startup/subscription readiness. Deployed identity coverage requires the next tier. |
| Deployed Fulfillment integration / DEV | Extend `it/it_labels_test.go` and `it/it_annotations_test.go`; proposed `it/it_metadata_inventory_test.go` and `it/it_metadata_transport_test.go` | `osac/fulfillment-service`: `make -C ../osac-installer test PLATFORM=kind PROFILE=dev NS=osac SUITE=fulfillment` | Real deployed API, PostgreSQL, Keycloak, REST gateway, and CLI. Seed metadata fixtures; this does not prove infrastructure-provider provisioning. G-2: all 35 valid fixtures, both admin accounts, both tenant accounts, and restricted-client coverage must be added. |
| UI unit / DEV | Existing Vitest suites/mock Connect transport; add editor, all-type registry, capabilities, tenant rendering/search, and error retention cases | `osac/osac-ui`: `pnpm test` | Mock transport/session data. Does not prove deployed proxy/API redaction or real login mapping. |
| Proxy unit / DEV | Existing bridge/middleware tests; add existing map/mask/lock forwarding and permission/concurrency error preservation | `osac/osac-ui/proxy`: `go test ./bridge ./middleware` with workspace-local Go cache/temp | Transport/schema fixtures. Real Connect forwarding remains a deployed browser/API assertion. |
| Deployed browser E2E / QE | Proposed maintained metadata journey suite under the mono-repo cross-component E2E structure `osac/tests/e2e/core/`; editor, all-resource registry, four personas, network response privacy, and stale/error behavior | **Runner/command must be added as part of G-1.** Existing `pnpm playwright` in `osac/osac-ui` provides scratch live-cluster investigation, not a maintained metadata suite. | Real browser, console, proxy, API, and Keycloak. A tracked login-only spec does not supply metadata coverage; current package scripts/configuration do not establish this runner. No test command is invented here. |
| Documentation / DEV | API schema/reference, CLI help, admin and tenant guides | Manual review of TC-IS5-01 and TC-IS5-02 | Compare documented examples with the implemented contract and transport evidence. |

Remote integration or E2E execution must follow `~/.codex/skills/remote-integration-tests/SKILL.md`, including fresh-cluster, detached-run, resource-limit, routing, result-recovery, and cleanup requirements. This revision does not initiate an execution environment.

## Test Cases

### IS-1: Equal administrator metadata management

#### TC-IS1-01: Both administrator personas manage every persistent metadata type

| Interface Change | Priority | Automation |
|---|---|---|
| IC-1, IC-2, IC-7 | critical | automated |

**Tier:** Component integration; deployed Fulfillment

##### Preconditions

For each of the 35 persistent types, seed valid dependencies and an object with ordinary entries in both maps. Both administrator accounts have the reviewed platform-admin authority.

##### Steps

1. For each administrator, type, and map, Create with a reserved entry where supported, then Get/List through its private service.
2. Read the full map/version, change a reserved value and add a second reserved key, then remove each by omission from a replacement map. Select the edited map with the existing update_mask and set lock=true.
3. Compare ordinary entries, reserved values, identifier, and version after each mutation using an authorized private read.

##### Expected Results

- Every supported Create/Update succeeds for both personas and both maps; private Get/List include exact reserved keys and values.
- All 35 persistent types have private Update coverage, including private-only types. Clients retain untouched entries and fields.

#### TC-IS1-02: Administrator map replacement, empty values, and deletion by omission

| Interface Change | Priority | Automation |
|---|---|---|
| IC-1, IC-3 | high | automated |

**Tier:** Unit; component integration

##### Preconditions

An administrator reads a private object with two ordinary and two reserved entries in each map, plus an unrelated mutable field.

##### Steps

1. Select metadata.labels and submit a complete edited map with a reserved value changed, another key omitted, and a retained empty-string value. Leave annotations unselected.
2. Select both maps and replace their contents; separately select a map with an empty or omitted map payload to clear it.
3. Repeat with an allowed non-metadata field in the existing mask, and compare existing unmasked replacement on types that support it.
4. For an ordinary key supported by the existing native per-key path grammar, verify its previous update behavior remains intact.

##### Expected Results

- A selected private administrator map equals the submitted map: omitted ordinary and reserved entries are deleted; an empty string is stored as a value.
- A selected empty/omitted map clears completely. Unselected maps and fields remain intact. Allowed mixed field changes apply atomically.
- Existing unmasked/per-key behavior is retained where supported; no new request field or implied-mask convention is introduced.

#### TC-IS1-03: Networking metadata reuses supported private Update behavior

| Interface Change | Priority | Automation |
|---|---|---|
| IC-2, IC-3 | critical | automated |

**Tier:** Component integration; deployed Fulfillment

##### Preconditions

Fixtures cover NetworkClass, VirtualNetwork, Subnet, SecurityGroup, ExternalIPPool, ExternalIP, ExternalIPAttachment, and NATGateway in valid stored states. Record existing metadata operations, operational/default-label checks, and immutable-field restrictions.

##### Steps

1. For both admins, set, change, and remove reserved entries on all eight services using fetched replacement maps, existing map-level masks, version, and lock=true.
2. Exercise supported ordinary and mixed ordinary/reserved map edits, retaining untouched operational labels. ExternalIP, ExternalIPAttachment, and NATGateway use a nonempty ordinary mask.
3. Attempt each recorded forbidden field/default-label change together with a reserved edit. Also omit the mandatory mask on the three lifecycle services.
4. Verify public Update remains absent and existing controller status operations retain their behavior.

##### Expected Results

- All eight private services accept the required reserved metadata edits for both admins under their resource rules; editing metadata needs no new reserved-only mode.
- Supported ordinary and mixed map updates remain supported. Unselected spec/status and lifecycle state remain unchanged.
- Forbidden field/default-label changes and missing mandatory masks retain their existing errors; no subset or version change is saved.
- Public networking Update is not introduced; existing controller status operations remain supported.

#### TC-IS1-04: REST uses existing whole-map PATCH masks

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3 | critical | automated |

**Tier:** Gateway unit; deployed Fulfillment

##### Preconditions

A mutable private object has multiple entries in both maps and a known version. Use the existing generated gateway and default query parser.

##### Steps

1. PATCH an edited labels map containing qualified keys and version, first with update_mask=metadata.labels and lock=true, then without an explicit mask to exercise the body-derived mask.
2. With an explicit labels mask, send an empty map, then an omitted map, to clear labels. Repeat for annotations and for both selected maps.
3. Compare REST results with native gRPC map-level requests and verify unselected maps/fields through private reads.

##### Expected Results

- Qualified keys remain map entries and reach the service unchanged; body-derived masks select complete maps, not per-key paths.
- Administrator replacement/clearing agrees with native gRPC. Explicit masks retain selection; unselected fields remain intact. Version locking rejects stale input.
- No custom query parser or additional mask representation is required. Tenant map preservation is checked separately by TC-IS4-01.

#### TC-IS1-05: CLI private metadata operations submit complete selected maps

| Interface Change | Priority | Automation |
|---|---|---|
| IC-2, IC-3, IC-8 | high | automated |

**Tier:** CLI unit; deployed Fulfillment

##### Preconditions

The updated CLI has private administrator credentials, sees namespace enforcement capabilities, and resolves all inventory types through existing names/aliases.

##### Steps

1. For both admins and all types, label/annotate a fetched private map using key=value and key- to set, change, and remove reserved entries.
2. Combine several edits, including an empty value and removal of an already-absent key; include untouched ordinary/reserved entries in the fixture.
3. Inspect protocol-unit requests and structured/human output after the deployed command.

##### Expected Results

- The request contains the complete edited selected map, its existing map-level update_mask, the fetched version, and lock=true.
- Untouched map entries and unrelated fields remain intact; key- deletes by omission and an empty string remains a value. Authorized absent-key removal is a no-op.
- Private reads show reserved values; both personas have identical coverage. Alias/registry completeness failures cannot silently omit a type.

#### TC-IS1-06: Console registry and editor cover all persistent types for both admins

| Interface Change | Priority | Automation |
|---|---|---|
| IC-2, IC-7, IC-9 | critical | automated |

**Tier:** UI unit; deployed browser E2E

##### Preconditions

Both administrators can sign in to the deployed console. Valid fixtures exist for all 35 types; private schema/proxy support matches the API.

##### Steps

1. Open /admin/metadata with each persona and enumerate every type, including private-only types and types without a dedicated detail page.
2. For each type, use both map editors to add, change, and remove a reserved entry while retaining untouched ordinary/reserved entries.
3. Inspect Connect/proxy request forwarding of full selected maps, the existing mask, and lock=true; follow detail-page links and reload through the API.

##### Expected Results

- Both personas have identical resource/action coverage; registry completeness is checked against all 35 types.
- Editors display complete private maps and submit their edited replacements with version locking. Untouched entries and unselected maps/fields remain intact.
- Connect/proxy/API persistence and existing detail-page links reach the same shared editor.

#### TC-IS1-07: Clients establish namespace enforcement before reserved writes

| Interface Change | Priority | Automation |
|---|---|---|
| IC-8, IC-9, IC-11 | critical | automated |

**Tier:** CLI/UI unit; deployed Fulfillment

##### Preconditions

Use new-server, capability-absent old-server, and wrong-prefix fixtures. The legacy server accepts ordinary Update shapes without privacy enforcement; record every Create/Update invocation.

##### Steps

1. Read Capabilities.Get from the new server and verify metadata.reserved_prefix.
2. Attempt reserved Create/Update from updated CLI/console against absent or mismatched capabilities; cover native gRPC preflight.
3. Perform an ordinary operation against the older-server fixture and attempt an unauthorized reserved write against the supported server.

##### Expected Results

- The new server advertises admin.osac.openshift.io/ as an enforced namespace, independently of caller authority.
- Updated clients send no reserved Create/Update to an unsupported server and show a clear compatibility error; accepting whole-map requests alone is not privacy support.
- Ordinary older-server flows remain supported. Capabilities do not grant reserved access; unauthorized input still fails.

#### TC-IS1-08: Administrator authority comes from the evaluated OPA decision

| Interface Change | Priority | Automation |
|---|---|---|
| IC-7 | critical | automated |

**Tier:** Auth unit; deployed Fulfillment

##### Preconditions

Accounts represent both administrators, Tenant Admin, Tenant User, and the restricted private CSI client. OPA decision fixtures include true, false, and absent is_admin.

##### Steps

1. Evaluate the supported persona/group mapping and copy the explicit is_admin result into Subject.
2. Exercise private reserved reads/writes for each account; repeat a public read with an administrator account.
3. Give a fixture broad tenant visibility without administrator authority and repeat the private metadata read.

##### Expected Results

- Both supported administrators receive the same reviewed reserved-metadata authority. False or missing is_admin defaults to no authority.
- Tenant roles and CSI universal-tenant visibility do not become administrator authority. Public administrator calls still receive the tenant metadata projection.
- Private RPC access continues to require the existing method/object authorization independently of metadata projection.

#### TC-IS1-09: Version locking protects administrator replacement maps

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-8, IC-9 | high | automated |

**Tier:** Component integration; deployed Fulfillment; UI unit

##### Preconditions

An administrator reads a complete map at version v; a second authorized update changes an otherwise untouched entry and advances the row to v+1.

##### Steps

1. Submit the edited stale map with its ordinary map-level mask, metadata.version=v, and lock=true through gRPC, REST, CLI, and Connect.
2. Read the complete private object after each rejection and inspect CLI/editor state.
3. Reload the full current map/version, retain the concurrent entry, and resubmit the intended edit.

##### Expected Results

- Every stale operation returns Aborted, changes nothing, and retains the concurrent entry/current version.
- CLI reports failure; the editor retains unsaved edits and requires reload/resubmission. It does not automatically replay a stale complete map.
- A fresh-version replacement succeeds without losing the other administrator entry.

### IS-2: Tenant visibility

#### TC-IS2-01: Every external object response omits reserved entries

| Interface Change | Priority | Automation |
|---|---|---|
| IC-1, IC-2, IC-5 | critical | automated |

**Tier:** Projection unit; deployed Fulfillment

##### Preconditions

For each public metadata-bearing type, seed both maps with ordinary and reserved entries. Also seed nested Metadata and unprivileged private-service fixtures.

##### Steps

1. Get and List as both tenant personas; perform allowed ordinary Create/Update that returns an object containing stored reserved metadata.
2. Repeat public reads with administrator credentials and inspect nested response objects.
3. Compare external responses against the authorized private source and inspect whether projection mutated that source.

##### Expected Results

- No reserved key or value appears in public or unprivileged private object responses; ordinary entries, authorized resource scope, and versions retain their established shape.
- Administrators using public RPCs get the same projection. Nested actual Metadata messages are projected recursively, while unrelated payload maps are unchanged.
- Projection uses detached responses; stored and in-process administrator objects retain their full maps.

#### TC-IS2-02: Event payload and filter projection preserve privacy

| Interface Change | Priority | Automation |
|---|---|---|
| IC-5, IC-6 | critical | automated |

**Tier:** Events component integration; deployed Fulfillment

##### Preconditions

Real Kafka-backed EventsServer coverage is available. Tenant watches are ready before publication; seed an object with both map kinds and register an admin private watcher.

##### Steps

1. Publish a create/update event containing reserved entries, nested Metadata, and a known ordinary filter match.
2. Consume through public tenant and private administrator watches; compare event identifiers, timestamps, and visible entries.
3. Change a reserved value without changing ordinary matching entries and repeat; allow at most 30 seconds after publication in the test harness.

##### Expected Results

- Tenant events omit reserved entries before the tenant event filter runs; authorized private administrator events retain them.
- Ordinary matching behavior, identifiers, and timestamps remain unchanged. No stored or publisher object is mutated by projection.
- The 30-second bound is a bounded test wait, not a new product delivery SLA; publication/subscription readiness failures are diagnosed separately.

#### TC-IS2-03: List and watch filters cannot probe protected key existence

| Interface Change | Priority | Automation |
|---|---|---|
| IC-6 | critical | automated |

**Tier:** Filter unit; component integration; deployed Fulfillment

##### Preconditions

Prepare otherwise identical objects with the target reserved key present and absent, plus ordinary metadata matches and nonmatches.

##### Steps

1. Submit accepted CEL syntaxes that explicitly reference a reserved label or annotation, including membership, indexing, and mixed ordinary/reserved expressions.
2. Repeat for List and event-filter entry points with the reserved key present and absent.
3. Exercise already-supported whole-map predicates against the projected map and compare List items and totals.

##### Expected Results

- Every explicit protected-key reference returns PermissionDenied uniformly before item/count SQL or subscription filtering, independent of stored existence.
- Already-supported whole-map predicates evaluate only visible metadata and return matching counts; unsupported syntax retains its existing error rather than gaining a new query operator.
- Ordinary literal-key predicates retain the existing indexed query path.

#### TC-IS2-04: Tenant console display, search, and network responses contain no reserved values

| Interface Change | Priority | Automation |
|---|---|---|
| IC-5, IC-8, IC-9 | critical | automated |

**Tier:** UI unit; deployed browser E2E

##### Preconditions

Tenant Admin and Tenant User sessions have access to objects with unique reserved marker values and ordinary catalog/search metadata.

##### Steps

1. Open available catalog and resource detail views and search using ordinary keys and reserved marker values.
2. Inspect console network response bodies, rendered text, accessible labels, and structured CLI output for the same public objects.
3. Compare ordinary result membership with API projected results.

##### Expected Results

- Reserved keys and marker values are absent from tenant API/Connect response bodies, UI display/search results, and CLI structured/human output.
- Ordinary search and display behavior remains consistent with the existing authorized public view. Hiding a field in the frontend alone does not satisfy this case.

#### TC-IS2-05: Restricted private clients remain unprivileged metadata readers

| Interface Change | Priority | Automation |
|---|---|---|
| IC-5, IC-7 | critical | automated |

**Tier:** Auth/projection unit; deployed Fulfillment

##### Preconditions

The existing CSI identity has its documented restricted method access and universal tenant visibility; fixtures span two tenants.

##### Steps

1. Read an object using a private method allowed to the CSI identity.
2. Attempt a reserved metadata write and a private method outside its existing allowlist.
3. Compare the same allowed read using an authorized administrator.

##### Expected Results

- CSI reads omit reserved maps despite cross-tenant visibility. Reserved writes and disallowed methods do not succeed.
- The administrator private view retains reserved values. No method allowlist or tenant scope is widened by this feature.

#### TC-IS2-06: Ephemeral review Metadata uses the same boundary without persistent CRUD

| Interface Change | Priority | Automation |
|---|---|---|
| IC-1, IC-5 | high | automated |

**Tier:** Projection/input unit; component integration

##### Preconditions

SelfSubjectAccessReview is an ephemeral message carrying Metadata, not one of the 35 persistent objects.

##### Steps

1. Submit a review request with ordinary metadata through each supported external view.
2. Exercise response projection with a review response fixture containing reserved entries, including a nested review message.
3. Submit a review carrying a reserved map entry from an unprivileged view.

##### Expected Results

- Ordinary review behavior remains unchanged and tenant outputs omit reserved entries wherever Metadata occurs.
- Unprivileged reserved input is rejected before review processing. The test does not invent Get/List/Update or persistence for the ephemeral review.

### IS-3: Atomic rejection of tenant mutations

#### TC-IS3-01: Tenant Create with mixed metadata fails as a whole

| Interface Change | Priority | Automation |
|---|---|---|
| IC-1, IC-4, IC-7 | critical | automated |

**Tier:** Input unit; deployed Fulfillment

##### Preconditions

Both tenant personas have valid create permissions on representative mutable services and valid referenced dependencies; record object count and provisioning/JIT side effects.

##### Steps

1. Create with an ordinary entry plus a reserved label; repeat with a reserved annotation and both maps.
2. Repeat across the supported Create inventory, including requests where reserved entries are nested in actual Metadata.
3. Inspect the public error, authorized private inventory, and transaction-side-effect tables.

##### Expected Results

- Every forbidden request returns PermissionDenied and creates no object, ordinary metadata, version, reference, or provisioning side effect.
- The error identifies the protected-target rule without revealing stored reserved values. Ordinary create requests remain covered by compatibility cases.

#### TC-IS3-02: Tenant Update cannot add or overwrite a reserved entry

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-4, IC-7 | critical | automated |

**Tier:** Input/candidate unit; deployed Fulfillment

##### Preconditions

A tenant-visible object has ordinary metadata and stored reserved entries; take a complete authorized private snapshot including version.

##### Steps

1. Update with mixed ordinary changes and a reserved entry that adds a key, changes a value, or supplies the same stored value.
2. Repeat with the reserved supplied entry outside the ordinary mask and in each map.
3. Read the full private object after each attempt.

##### Expected Results

- All attempts return PermissionDenied based on raw supplied protected input, including unchanged values and entries outside the ordinary mask.
- The entire stored object and version equal the pre-request snapshot; no ordinary subset is committed.

#### TC-IS3-03: Named reserved removals fail in clients before map construction

| Interface Change | Priority | Automation |
|---|---|---|
| IC-4, IC-8, IC-9 | critical | automated |

**Tier:** CLI/UI unit; deployed CLI

##### Preconditions

Tenant clients see only ordinary maps. Prepare present/absent reserved-key fixtures, an authorized private snapshot, and a counter for outgoing Update calls.

##### Steps

1. Attempt a reserved key- removal in label and annotate, and inject a named removal into the console operation validator.
2. Repeat for both maps, with the reserved key present/absent, and with an ordinary edit in the same operation.
3. Inspect local errors, Update invocation counts, retained editor state, and the full private stored object.

##### Expected Results

- The client reports the reserved-operation permission error before discarding its target or constructing an Update map, regardless of stored existence.
- No Update is sent, no ordinary subset/version change is saved, and console edits remain available for correction.
- This local check supplies the named-removal error; server omission preservation is covered by TC-IS4-01.

#### TC-IS3-04: Existing explicit mask paths cannot bypass protected targets

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-4 | high | automated |

**Tier:** Input/mask unit; REST/gRPC component integration

##### Preconditions

Existing clients use ordinary update_mask; capture complete private snapshots for otherwise identical objects with a reserved key present and absent.

##### Steps

1. Exercise any existing mask syntax the server accepts that explicitly identifies a reserved key, including native gRPC paths, with its value omitted.
2. Submit REST/Connect encodings with invalid quoted/slash-containing FieldMask forms.
3. Submit a valid ordinary whole-map replacement omitting hidden keys as a separate control.

##### Expected Results

- Accepted explicit protected paths return PermissionDenied before mutation, regardless of key existence; ordinary changes and versions remain unchanged.
- Malformed encodings retain their validation/decoding errors with no mutation; such errors are not evidence of protected-target authorization.
- The valid omission control succeeds for ordinary edits and preserves hidden entries. No new qualified per-key grammar is added.

#### TC-IS3-05: CLI mixed forbidden operations fail without partial success

| Interface Change | Priority | Automation |
|---|---|---|
| IC-4, IC-8 | critical | automated |

**Tier:** CLI unit; deployed Fulfillment

##### Preconditions

Tenant CLI credentials can edit ordinary metadata; reserved entries are hidden by Get. Capture the full object/version and record Update invocations.

##### Steps

1. Run label with an ordinary set plus reserved key- removal; repeat for annotate and an absent reserved key.
2. Run a mixed ordinary/reserved set command, including a reserved entry with an unchanged value.
3. Inspect exit status, stderr, Update invocation count, and the complete private stored object.

##### Expected Results

- Every command exits nonzero with a clear permission error and sends no Update; the target is checked before local deletion could erase it.
- Neither ordinary changes nor version advancement persists. A forbidden absent-key removal is not reported as success.

#### TC-IS3-06: REST and Connect enforce input rejection and omission preservation

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-4, IC-9 | critical | automated |

**Tier:** UI/proxy unit; deployed Fulfillment; deployed browser E2E

##### Preconditions

A tenant has pending ordinary edits and an object with reserved entries. Capture its full private snapshot; UI unit fixtures can inject a named forbidden operation.

##### Steps

1. Send mixed ordinary/reserved map entries through REST and Connect/proxy, including requests crafted outside normal tenant controls.
2. Drive the editor error component with the returned server error; separately inject a named reserved removal and inspect the local client rejection.
3. Submit a tenant replacement containing only ordinary entries, omitting reserved entries, then read the full private object.

##### Expected Results

- Supplied reserved entries cause PermissionDenied with existing transport mappings and no object/version/side-effect changes. Neither transport strips them and retries the ordinary subset.
- The editor retains unsaved changes after server/local permission failures; a locally named forbidden removal sends no Update.
- Ordinary replacement succeeds and keeps stored reserved entries. Omission is not an explicit-removal request and does not produce a server permission error.

#### TC-IS3-08: Rejected requests and returned handler errors roll back transactions

| Interface Change | Priority | Automation |
|---|---|---|
| IC-4 | critical | automated |

**Tier:** Transaction unit; real-PostgreSQL component integration

##### Preconditions

A real PostgreSQL test transaction can observe candidate/object writes and auxiliary JIT/reference rows. Use a handler fault fixture; do not provision external infrastructure.

##### Steps

1. Submit forbidden raw input that would otherwise trigger JIT provisioning or reference processing.
2. In a separate integration fixture, make the handler write test rows and then return an error through TxInterceptor.
3. Read the object, version, and auxiliary tables from a separate transaction after completion.

##### Expected Results

- Forbidden input is rejected before JIT/reference mutation.
- The handler error is reported to the transaction before End; both test rows and object changes are rolled back. An error response never accompanies a committed partial change.
- Real database rollback evidence is required; a mocked transaction call-order test alone is insufficient.

### IS-4: Ordinary metadata compatibility

#### TC-IS4-01: Every tenant replacement preserves stored reserved entries

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-4 | critical | automated |

**Tier:** Candidate unit; deployed Fulfillment

##### Preconditions

A tenant-visible mutable object contains multiple ordinary/reserved entries in both maps. Capture the complete private object and its version.

##### Steps

1. Replace metadata.labels, metadata.annotations, and parent metadata with existing accepted masks, and exercise the existing no-mask path where supported.
2. Send ordinary edits, empty selected maps, and omitted selected maps; verify an unselected map remains unchanged.
3. Run an older CLI fixture that deletes an unseen reserved key locally before submitting its ordinary map, including a mixed ordinary change.
4. Compare complete private maps/versions with the baseline ordinary replacement behavior.

##### Expected Results

- Every successful tenant update restores all reserved entries from the locked current row, including parent-map and unmasked replacements.
- Ordinary omitted keys are removed as before; empty/omitted selected maps clear only ordinary entries. Unselected maps remain intact.
- The older client receives success for the valid ordinary request and reports no local reserved-removal error; stored reserved entries remain unchanged. The server does not infer an explicit removal from omission.
- Raw reserved entries still fail atomically under TC-IS3-02; merging does not silently accept a supplied forbidden key.

#### TC-IS4-02: Namespace boundaries and ordinary operational metadata

| Interface Change | Priority | Automation |
|---|---|---|
| IC-1, IC-5 | high | automated |

**Tier:** Predicate/validation unit; deployed Fulfillment

##### Preconditions

Use valid ordinary keys including osac.openshift.io/tenant and osac.openshift.io/owner-reference where the existing API permits them, plus near-prefix strings accepted by existing syntax validation.

##### Steps

1. Evaluate admin.osac.openshift.io/key, child.admin.osac.openshift.io/key, and a.b.admin.osac.openshift.io/key. Seed them through an authorized private write and inspect public response visibility in both maps.
2. Exercise ordinary handling and visibility for fakeadmin.osac.openshift.io/key, admin.osac.openshift.io.example/key, the slashless key admin.osac.openshift.io, and existing operational keys; separately test invalid-case/syntax inputs against the existing validator.
3. Verify values, including empty strings, retain their established validation/storage behavior.

##### Expected Results

- The base DNS namespace and every dot-delimited subdomain are reserved and omitted from public responses. Match case-sensitively without normalization; similarly named domains and slashless keys remain ordinary.
- Operational isolation/lifecycle keys are not reclassified as reserved. Existing validation and resource-specific ownership restrictions still apply to ordinary keys.
- Invalid keys fail under the existing syntax contract; this feature adds no broader acceptance.

#### TC-IS4-03: Concurrent tenant updates preserve the latest reserved metadata

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-4 | critical | automated |

**Tier:** Real-PostgreSQL component integration; deployed Fulfillment

##### Preconditions

A tenant reads its visible maps/version v; an administrator adds, changes, or removes a reserved entry and commits before the tenant Update acquires the row.

##### Steps

1. Submit the stale ordinary replacement with its map-level mask and lock=true; inspect the row.
2. Repeat with the existing unlocked path where supported, controlling row-acquisition order. Include empty map, parent-metadata, and unmasked forms.
3. Compare ordinary edits and the complete latest reserved portion through an authorized private read.

##### Expected Results

- The stale locked request returns Aborted and changes nothing.
- An allowed unlocked update retains exactly the latest reserved portion from the locked stored row: additions and new values survive, and an administrator removal is not resurrected.
- Ordinary replacement/deletion retains its baseline behavior; tenant snapshots or pre-lock reads are never the source of restored reserved entries.

#### TC-IS4-04: Ordinary filtering, totals, and watches retain existing behavior

| Interface Change | Priority | Automation |
|---|---|---|
| IC-5, IC-6 | high | automated |

**Tier:** Filter unit; Kafka/PostgreSQL component integration; deployed Fulfillment

##### Preconditions

Objects have ordinary match/nonmatch metadata and hidden reserved entries; existing accepted filter expressions and baseline counts are recorded.

##### Steps

1. Run existing ordinary literal-key and supported whole-map List filters and compare items/total.
2. Publish matching and nonmatching events and observe tenant watches using the same ordinary predicates.
3. Repeat after changing only a reserved entry.

##### Expected Results

- Ordinary literal-key results, counts, and watch membership match the existing baseline; SQL keeps the indexed literal-key path.
- Whole-map predicates observe the documented visible projection. Reserved-only changes do not alter an ordinary predicate result or expose a protected entry.

#### TC-IS4-05: Ordinary clients and service mutation boundaries remain compatible

| Interface Change | Priority | Automation |
|---|---|---|
| IC-2, IC-8, IC-9, IC-11 | high | automated |

**Tier:** CLI/UI unit; deployed Fulfillment

##### Preconditions

Use older ordinary CLI fixtures, updated clients against legacy server fixtures, existing ordinary console controls, public networking services, and private networking fixtures with recorded allowed metadata operations.

##### Steps

1. Perform established ordinary label/annotation operations through supported older/new clients; inspect the new clients' existing map masks and lock fields.
2. Repeat ordinary console editing on mutable public resources and inspect filtered responses.
3. Attempt public networking Update and compare the existing absence/Unimplemented boundary.
4. Replay supported private networking metadata updates with existing masks and full-map requests, retaining system/default labels.

##### Expected Results

- Supported ordinary operations remain available; updated clients select only edited maps and use version locking without replacing unrelated fields.
- The new server preserves reserved entries on tenant updates. Ordinary older-server operations remain supported, and capabilities do not grant permissions.
- Public networking Update remains absent. Existing private networking metadata, operational-label, immutable-field, and lifecycle-mask rules remain enforced.

### IS-5: Prefix and behavior documentation

#### TC-IS5-01: Documentation defines the namespace and subdomain matching

| Interface Change | Priority | Automation |
|---|---|---|
| IC-1, IC-10 | medium | manual |

**Tier:** Documentation review

##### Preconditions

API reference, administrator guide, and tenant guide drafts accompany the implementation.

##### Steps

1. Review the reserved namespace and examples in each guide.
2. Compare examples with the predicate tests, including subdomain, case/syntax handling, ordinary operational keys, and empty-value semantics.

##### Expected Results

- Every guide names admin.osac.openshift.io/ and explains case-sensitive matching of the base DNS namespace and all subdomains, including examples that reject similar non-subdomain names. No configurable namespace variants are introduced.
- Examples agree with executable contract cases and preserve the distinction between reserved matching and existing syntax validation.

#### TC-IS5-02: Documentation covers personas, map semantics, errors, and compatibility

| Interface Change | Priority | Automation |
|---|---|---|
| IC-10, IC-11 | medium | manual |

**Tier:** Documentation review

##### Preconditions

Guide examples, CLI help, and API schema are available for the proposed implementation.

##### Steps

1. Review both admin journeys and both tenant expectations across API, CLI, and console.
2. Check native gRPC, existing REST PATCH/query masks, and Connect examples for complete selected maps, omission-based deletion, empty maps/values, capability preflight, and stale versions.
3. Check the all-resource inventory, local named-removal error, raw-input server rejection, tenant omission preservation, and older-client/server limitations.

##### Expected Results

- Guides describe identical admin actions on all persistent types and private-only types, tenant privacy, and whole-request rejection for observable forbidden input.
- Examples use existing Update maps/masks/lock; administrator omission deletes while tenant omission preserves. Named CLI/console removal is checked before sending a map, and discarded targets cannot be inferred by the server.
- Support guidance requires namespace enforcement before reserved writes, documents older-client limitations, excludes in-place upgrades, and logs no protected values.

## Coverage summary and remaining work

- **29 cases:** 19 critical, 8 high, 2 medium, 0 low.
- **Automation:** 27 planned automated cases and two manual documentation cases. A case spanning tiers requires evidence at every listed relevant boundary.
- **Requirement coverage:** 5/5 local In Scope anchors mapped; no PRD scope bullet is omitted.
- **Interface coverage:** 11/11 design interface IDs mapped. The table below provides an independent index.

| Interface | Cases |
|---|---|
| IC-1 | TC-IS1-01, TC-IS1-02, TC-IS2-01, TC-IS2-06, TC-IS3-01, TC-IS4-02, TC-IS5-01 |
| IC-2 | TC-IS1-01, TC-IS1-03, TC-IS1-05, TC-IS1-06, TC-IS2-01, TC-IS4-05 |
| IC-3 | TC-IS1-02, TC-IS1-03, TC-IS1-04, TC-IS1-05, TC-IS1-09, TC-IS3-02, TC-IS3-04, TC-IS3-06, TC-IS4-01, TC-IS4-03 |
| IC-4 | TC-IS3-01, TC-IS3-02, TC-IS3-03, TC-IS3-04, TC-IS3-05, TC-IS3-06, TC-IS3-08, TC-IS4-01, TC-IS4-03 |
| IC-5 | TC-IS2-01, TC-IS2-02, TC-IS2-04, TC-IS2-05, TC-IS2-06, TC-IS4-02, TC-IS4-04 |
| IC-6 | TC-IS2-02, TC-IS2-03, TC-IS4-04 |
| IC-7 | TC-IS1-01, TC-IS1-06, TC-IS1-08, TC-IS2-05, TC-IS3-01, TC-IS3-02 |
| IC-8 | TC-IS1-05, TC-IS1-07, TC-IS1-09, TC-IS2-04, TC-IS3-03, TC-IS3-05, TC-IS4-05 |
| IC-9 | TC-IS1-06, TC-IS1-07, TC-IS1-09, TC-IS2-04, TC-IS3-03, TC-IS3-06, TC-IS4-05 |
| IC-10 | TC-IS5-01, TC-IS5-02 |
| IC-11 | TC-IS1-07, TC-IS4-05, TC-IS5-02 |

### Execution gaps

| Gap | Required work / owner | Affected coverage | Ticket status and closure evidence |
|---|---|---|---|
| G-1 | QE with console maintainers: establish a maintained browser runner, correct discovery/configuration, add deployed metadata journeys, and document the execution command | TC-IS1-06, TC-IS2-04, TC-IS3-06; browser portions of stale/error cases | Work belongs to OSAC-6204; no separate follow-up ticket has been created. Close with committed suite/runner, reproducible command, deployed four-persona results, and saved failure artifacts. |
| G-2 | Fulfillment DEV with QE: create valid inventory fixtures and deployed identity credentials/seed setup for all 35 persistent types, both maps, both admins, both tenants, and restricted CSI identity | TC-IS1-01, TC-IS1-03, TC-IS1-05, TC-IS1-08, TC-IS2-01, TC-IS2-05, TC-IS3-01 and exhaustive regression matrix | Work belongs to OSAC-6204; no separate follow-up ticket has been created. Close with executable fixtures, explicit seeded lifecycle states, inventory completeness assertions, and real deployed transport/auth evidence. |

The networking contract question reconciles the accepted metadata-immutability document with existing private metadata updates and required administrator access. TC-IS1-03 and TC-IS4-05 verify those updates with current masks and resource validation. Close any fixture-discovered mismatch without removing supported ordinary operations or adding public Update. G-1/G-2 remain delivery prerequisites for the full resource/interface scope.

### Acceptance evidence

Completion requires real results for the exhaustive inventory and required boundary suites, not only a case count or mocked success. Record the executed command/cwd, commit, suite/tier, account setup, real dependencies and doubles, result artifacts, and any remaining gap with its owner. Rejected reserved requests must leave the complete object/version and side effects unchanged; tenant response/filter evidence must contain no protected values. Both administrators must demonstrate the same API, CLI, and deployed console actions. Ordinary compatibility cases and the documentation reviews must pass before milestone graduation.
