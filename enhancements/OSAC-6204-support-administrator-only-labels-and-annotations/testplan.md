# OSAC-6204 — Administrator-only metadata test plan

## Overview

- **Source:** [PRD](prd.md), its co-located clarifications, and [design draft](design.md).
- **Status:** Planned behavioral coverage; no implementation tests were executed during this draft phase.
- **Coverage:** 30 cases, five of five In Scope bullets, and 11 of 11 interface changes. Twenty-eight cases are planned for automation; two are manual documentation reviews. Browser execution and exhaustive deployed fixtures still require the work recorded under G-1 and G-2.
- **Target:** 0.4 Tech Preview. Both owned design questions must be resolved before implementing their affected contracts.

The PRD uses In Scope bullets rather than FR/NFR identifiers. IS-1 through IS-5 below are local traceability anchors to those exact bullets, not new requirements. Implementation details such as the key mask, capability preflight, transaction rollback, and filter projection are testable design mechanisms for those requirements.

## Requirements and interface coverage

| Anchor | PRD In Scope requirement | Primary cases | Design interface coverage |
|---|---|---|---|
| IS-1 | Both admin personas read/add/change/remove reserved metadata through private API, console, and CLI on all metadata-bearing resources | TC-IS1-01–09 | IC-1, IC-2, IC-3, IC-7, IC-8, IC-9, IC-11 |
| IS-2 | Tenant API responses and console omit reserved keys | TC-IS2-01–06 | IC-1, IC-2, IC-5, IC-6, IC-7, IC-8, IC-9 |
| IS-3 | Tenant add/change/remove attempts fail the whole request clearly | TC-IS3-01–08 | IC-1, IC-3, IC-4, IC-7, IC-8, IC-9 |
| IS-4 | Ordinary metadata keeps its existing behavior | TC-IS4-01–05 | IC-1, IC-2, IC-3, IC-4, IC-5, IC-6, IC-8, IC-9, IC-11 |
| IS-5 | Document exact reserved prefix and matching behavior | TC-IS5-01–02 | IC-1, IC-10, IC-11 |

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
| Unit / DEV | Existing fulfillment auth, validation, mapper, mask/filter, CLI, and transaction tests; add reserved predicate, capability, explicit target, reflection, parser, and rollback-order cases | `osac/fulfillment-service`: `ginkgo run -r internal` (use workspace-local GOCACHE/GOTMPDIR; heavy suites sequentially) | Policy/parser/transport fixtures and test doubles. This broad command also reaches the component suites below; report their real dependencies separately. |
| Component integration / DEV | Existing `internal/servers/servers_suite_test.go` and server tests; add full candidate, networking metadata compatibility/scoped exceptions, inventory, concurrency, and error rollback cases | `osac/fulfillment-service`: `ginkgo run --timeout 10m internal/servers` | Real PostgreSQL from database.Container; in-process RPC and mocked tenancy/attribution. This does not establish deployed Keycloak, REST, proxy, or provider behavior. |
| Events component integration / DEV | Existing `internal/servers/events_server_test.go`; add nested projection and filter privacy cases | `osac/fulfillment-service`: same server-suite command, executed once for the combined suite | Real Kafka via Kafka.Container, bufconn RPC, and mocked tenancy. Record broker startup/subscription readiness. Deployed identity coverage requires the next tier. |
| Deployed Fulfillment integration / DEV | Extend `it/it_labels_test.go` and `it/it_annotations_test.go`; proposed `it/it_metadata_inventory_test.go` and `it/it_metadata_transport_test.go` | `osac/fulfillment-service`: `make -C ../osac-installer test PLATFORM=kind PROFILE=dev NS=osac SUITE=fulfillment` | Real deployed API, PostgreSQL, Keycloak, REST gateway, and CLI. Seed metadata fixtures; this does not prove infrastructure-provider provisioning. G-2: all 35 valid fixtures, both admin accounts, both tenant accounts, and restricted-client coverage must be added. |
| UI unit / DEV | Existing Vitest suites/mock Connect transport; add editor, all-type registry, capabilities, tenant rendering/search, and error retention cases | `osac/osac-ui`: `pnpm test` | Mock transport/session data. Does not prove deployed proxy/API redaction or real login mapping. |
| Proxy unit / DEV | Existing bridge/middleware tests; add helper forwarding and permission/concurrency error preservation | `osac/osac-ui/proxy`: `go test ./bridge ./middleware` with workspace-local Go cache/temp | Transport/schema fixtures. Real Connect forwarding remains a deployed browser/API assertion. |
| Deployed browser E2E / QE | Proposed maintained metadata journey suite under the mono-repo cross-component E2E structure `osac/tests/e2e/core/`; editor, all-resource registry, four personas, network response privacy, and stale/error behavior | **Runner/command must be added as part of G-1.** Existing `pnpm playwright` in `osac/osac-ui` provides scratch live-cluster investigation, not a maintained metadata suite. | Real browser, console, proxy, API, and Keycloak. A tracked login-only spec does not supply metadata coverage; current package scripts/configuration do not establish this runner. No test command is invented here. |
| Documentation / DEV | API schema/reference, CLI help, admin and tenant guides | Manual review of TC-IS5-01 and TC-IS5-02 | Compare documented examples with the implemented contract and transport evidence. |

Remote integration or E2E execution must follow `~/.codex/skills/remote-integration-tests/SKILL.md`, including fresh-cluster, detached-run, resource-limit, routing, result-recovery, and cleanup requirements. This draft does not initiate an execution environment.

## Test Cases

### IS-1: Equal administrator metadata management

#### TC-IS1-01: Both administrator personas manage every persistent metadata type

| Interface Change | Priority | Automation |
|---|---|---|
| IC-1, IC-2, IC-7 | critical | automated |

**Tier:** Component integration; deployed Fulfillment

##### Preconditions

For each of the 35 persistent types in the inventory below, seed valid dependencies and an object with an ordinary label and annotation. Both administrator accounts have the reviewed platform-admin authority.

##### Steps

1. For each administrator, type, and map, Create an object containing a reserved entry where Create is supported, then Get and List it through its private service.
2. Update the reserved value, add a second reserved key, and remove each reserved key using the typed key mask.
3. Use an authorized private read to compare values, ordinary entries, identifier, and version after every mutation.

##### Expected Results

- Each supported Create/Update succeeds for both personas and both maps; Get/List include the exact reserved keys and values.
- All 35 persistent types have tested private Update coverage; private-only types are included. Ordinary entries remain unchanged.

#### TC-IS1-02: Exact key-mask set, empty value, and clear semantics

| Interface Change | Priority | Automation |
|---|---|---|
| IC-1, IC-3 | high | automated |

**Tier:** Unit; component integration

##### Preconditions

An administrator owns a private object with two ordinary and two reserved entries in each map.

##### Steps

1. Submit a mask naming one label and one annotation with values present in the corresponding object maps; include an empty-string value in a separate iteration.
2. Submit the same targets with those keys absent from their maps to clear them.
3. Repeat with no ordinary update_mask, then with an explicit ordinary mask for a separate non-metadata field on a mutable type.

##### Expected Results

- Only named keys change; an empty string remains a stored value and absence clears a target.
- With no ordinary mask only the selected keys change. An allowed disjoint ordinary field update is applied atomically with the key edit. Untargeted map entries remain intact.

#### TC-IS1-03: Networking metadata reuses supported private Update behavior

| Interface Change | Priority | Automation |
|---|---|---|
| IC-2, IC-3 | critical | automated |

**Tier:** Component integration; deployed Fulfillment

##### Preconditions

Fixtures cover NetworkClass, VirtualNetwork, Subnet, SecurityGroup, ExternalIPPool, ExternalIP, ExternalIPAttachment, and NATGateway in valid stored states. Record each existing private service's allowed ordinary metadata operations and immutable-field checks before adding the feature. Identify any type requiring a reserved-only validation mode.

##### Steps

1. For both administrators, set, change, and clear reserved entries through each existing private Update service using exact key-mask targets.
2. Where ordinary metadata editing is already supported, exercise legacy map updates and mixed ordinary/reserved key edits permitted by that service's existing rules.
3. Attempt changes to fields rejected by each service's recorded baseline, such as Subnet CIDR or VirtualNetwork NetworkClass, together with a reserved edit; inspect the full object and version after rejection.
4. For types needing the new reserved-only mode, submit a nonempty reserved-key mask without an ordinary mask, then attempt an ordinary-map or other nonreserved change in that mode.
5. Verify public Update remains absent and existing controller status operations retain their behavior.

##### Expected Results

- Reserved set/change/clear succeeds for both admins on all eight types without a new provisioning cycle or unrelated provider-readiness checks.
- Already-supported ordinary metadata operations continue to succeed; allowed mixed edits apply atomically. The feature does not impose a new ordinary-metadata ban on these services.
- Existing immutable-field errors remain errors and save no subset of the mixed request. In a newly required reserved-only mode, nonreserved changes return InvalidArgument and leave the object/version unchanged.
- Public networking Update is not introduced, and existing controller status operations remain supported.

#### TC-IS1-04: REST query key masks survive gateway canonicalization

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3 | critical | automated |

**Tier:** Gateway unit; deployed Fulfillment

##### Preconditions

A mutable private object has known labels/annotations and version; the updated gateway and custom query parser are deployed.

##### Steps

1. PATCH with an encoded slash in metadata_update_mask.labels and a body supplying its new value, without an explicit ordinary update_mask query parameter.
2. PATCH again with the same query target and a body containing only metadata.version to clear it.
3. Send a disjoint explicit ordinary update_mask, then an overlapping metadata.labels mask. Repeat for annotation targets.

##### Expected Results

- The typed target is decoded exactly; the gateway body-derived ordinary mask is cleared only when the helper is present and no explicit ordinary mask was supplied.
- Set and clear have the same behavior as native gRPC. Explicit disjoint masks remain effective; overlap returns InvalidArgument without changing either field or version.

#### TC-IS1-05: CLI private metadata operations retain all explicit targets

| Interface Change | Priority | Automation |
|---|---|---|
| IC-2, IC-3, IC-8 | high | automated |

**Tier:** CLI unit; deployed Fulfillment

##### Preconditions

The updated CLI advertises supported metadata editing, has private administrator credentials, and can resolve every inventory type through its existing names/aliases.

##### Steps

1. For both administrators and every type, run label and annotate to set, change, and remove a reserved key using the existing key=value and key- forms.
2. Submit multiple targets in one command, including an empty value and removal of an already-absent key.
3. Read the object in structured and human-readable output; inspect the outgoing typed mask in a protocol unit test.

##### Expected Results

- The CLI sends exact map-specific targets, retains absent removal intent, and reports success on supported private services.
- Reads show the administrator values. Untargeted fields remain unchanged and version locking is requested. Unsupported aliases are caught by the registry coverage check rather than silently omitted.

#### TC-IS1-06: Console registry and editor cover all persistent types for both admins

| Interface Change | Priority | Automation |
|---|---|---|
| IC-2, IC-7, IC-9 | critical | automated |

**Tier:** UI unit; deployed browser E2E

##### Preconditions

Both administrators can sign in to the deployed console. Valid fixtures exist for all 35 types; private schema/proxy support matches the API.

##### Steps

1. Open /admin/metadata with each persona and enumerate every inventory type, including private-only types and types without a dedicated detail page.
2. For every type, load an object and use both map editors to add, change, and remove a reserved key.
3. Follow metadata-editor links from existing detail pages and reload changed objects through the API.

##### Expected Results

- Both personas have identical resource and action coverage. Registry completeness is enforced against the explicit inventory.
- Editors preserve separate map targets, display stored private values, and persist successful changes through Connect/proxy/API. Existing detail-page links reach the same editor.

#### TC-IS1-07: Clients require metadata capabilities before exact-key writes

| Interface Change | Priority | Automation |
|---|---|---|
| IC-8, IC-9, IC-11 | critical | automated |

**Tier:** CLI/UI unit; deployed Fulfillment

##### Preconditions

Use new-server, capability-absent old-server, and unsupported-capability transport fixtures. Record every write invocation.

##### Steps

1. Read Capabilities.Get from the new server and verify the metadata reserved_prefix and supports_key_masks fields.
2. Attempt reserved metadata writes from updated CLI and console with capability absent or unsupported, including a native gRPC client preflight.
3. Perform an ordinary legacy operation against the older-server fixture.

##### Expected Results

- The new server advertises admin.osac.openshift.io/ and key-mask support; support advertisement conveys no identity authority.
- Updated clients prevent reserved/key-mask writes before sending Update to an unsupported server, show a clear compatibility error, and never downgrade the reserved operation to a broad replacement.
- Existing ordinary older-server flows retain their legacy request representation.

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

#### TC-IS1-09: Version locking protects administrator edits

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-8, IC-9 | high | automated |

**Tier:** Component integration; deployed Fulfillment; UI unit

##### Preconditions

An administrator reads version v; a second authorized update advances the same object to v+1.

##### Steps

1. Submit a typed-mask edit with metadata.version=v and lock=true through gRPC, REST, CLI, and Connect.
2. Read the private object after each rejection.
3. Reload its new version and resubmit the intended edit.

##### Expected Results

- Every stale operation returns Aborted, commits no changes, and leaves the current version intact.
- CLI reports failure; the editor retains the unsaved edit and requires reload/resubmission. A fresh-version operation succeeds once.

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

#### TC-IS3-03: Explicit reserved clear is denied even when the key is absent

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-4 | critical | automated |

**Tier:** Input unit; deployed Fulfillment

##### Preconditions

Prepare otherwise identical objects with a reserved key present and absent. Both tenant personas can edit ordinary metadata.

##### Steps

1. Submit metadata_update_mask naming the reserved key while omitting its value, together with a disjoint ordinary edit.
2. Repeat with the key present, absent, and in each map.
3. Compare errors and private stored snapshots.

##### Expected Results

- All requests return PermissionDenied with equivalent error semantics independent of key existence.
- Absence of a visible/stored key never turns explicit removal into success or a silent no-op. Ordinary edits and versions remain unchanged.

#### TC-IS3-04: Legacy explicit mask intent cannot bypass protected targets

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-4 | high | automated |

**Tier:** Input/mask unit; REST/gRPC component integration

##### Preconditions

Legacy clients use ordinary update_mask without the helper. Capture the private pre-request snapshot.

##### Steps

1. Exercise any legacy mask syntax accepted by the server that explicitly identifies a reserved key, including direct gRPC paths.
2. Submit REST/Connect encodings with invalid quoted/slash-containing FieldMask forms.
3. Submit a valid ordinary map replacement omitting hidden keys as a separate compatibility control.

##### Expected Results

- Accepted explicit protected targets are denied before mutation; malformed protobuf/FieldMask encodings return InvalidArgument during validation/decoding with no stored change.
- A decoding error is not evidence of successful protected-target authorization. The valid omission control preserves hidden entries under the documented legacy semantics.

#### TC-IS3-05: CLI mixed forbidden operations fail without partial success

| Interface Change | Priority | Automation |
|---|---|---|
| IC-4, IC-8 | critical | automated |

**Tier:** CLI unit; deployed Fulfillment

##### Preconditions

Tenant CLI credentials can edit ordinary metadata; reserved entries are hidden by Get. Capture the full stored object and version.

##### Steps

1. Run label with an ordinary set plus a reserved key- removal; repeat for annotate and for an absent reserved key.
2. Run a mixed ordinary and reserved set operation.
3. Inspect CLI exit status, stderr, outgoing mask targets, and the private stored object.

##### Expected Results

- Every command exits nonzero and reports the permission failure; the reserved removal target remains in the request despite being absent from the fetched public object.
- No requested ordinary change or version increment persists. The CLI does not report a forbidden absent-key removal as a successful no-op.

#### TC-IS3-06: REST and Connect report atomic failure and preserve editor state

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-4, IC-9 | critical | automated |

**Tier:** UI/proxy unit; deployed Fulfillment; deployed browser E2E

##### Preconditions

A tenant mutation fixture has pending ordinary edits and an explicitly injected protected target; keep a full private stored snapshot.

##### Steps

1. Send mixed forbidden requests through REST and Connect/proxy, including set and clear targets.
2. Drive the editor mutation/error component with the resulting permission error, including a request crafted outside normal tenant controls.
3. Reload through an authorized private read and inspect the unsaved form.

##### Expected Results

- REST and Connect propagate PermissionDenied using their established transport mappings; neither strips the protected target and retries an ordinary subset.
- The database and version are unchanged. The editor displays a clear failure and retains the unsaved ordinary edits for correction. Server enforcement does not depend on whether a tenant control was rendered.

#### TC-IS3-07: Invalid helper and overlapping masks fail before persistence

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-4 | high | automated |

**Tier:** Mask/candidate unit; component integration

##### Preconditions

An authorized administrator can update the target mutable object; capture a private snapshot.

##### Steps

1. Submit a present-but-empty helper, duplicate targets in one map, and invalid keys.
2. Submit a helper plus ordinary update_mask covering metadata, the same map, or a descendant of the selected map.
3. Repeat with labels and annotations, and compare a disjoint-map request as a valid control.

##### Expected Results

- Invalid helper syntax and mask overlap return InvalidArgument; object, version, and side effects are unchanged.
- The valid disjoint-map control updates only its selected targets and ordinary mask. The same key string in different maps is treated as two map-specific targets, not an invalid duplicate across maps.

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

#### TC-IS4-01: Legacy map replacement retains hidden stored entries

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-4 | critical | automated |

**Tier:** Candidate unit; deployed Fulfillment

##### Preconditions

A tenant-visible mutable object contains ordinary and reserved entries in both maps. Snapshot the full private object.

##### Steps

1. Replace metadata.labels, metadata.annotations, and metadata through the existing ordinary mask forms; include the legacy no-mask update path.
2. Clear ordinary keys by omission using each accepted legacy representation.
3. Read the private object and compare ordinary replacement results with the existing baseline.

##### Expected Results

- Every successful ordinary update retains the protected portion from the locked current stored row.
- Ordinary set/replacement/clear behavior and version advancement remain consistent with existing semantics; missing hidden entries in a tenant payload are not treated as explicit deletion intent.

#### TC-IS4-02: Prefix boundary and operational metadata remain ordinary

| Interface Change | Priority | Automation |
|---|---|---|
| IC-1, IC-5 | high | automated |

**Tier:** Predicate/validation unit; deployed Fulfillment

##### Preconditions

Use valid ordinary keys including osac.openshift.io/tenant and osac.openshift.io/owner-reference where the existing API permits them, plus near-prefix strings accepted by existing syntax validation.

##### Steps

1. Evaluate exact admin.osac.openshift.io/key and a different valid DNS prefix such as x.admin.osac.openshift.io/key.
2. Exercise ordinary map handling and visibility for operational and near-prefix keys; separately test invalid-case/syntax inputs against the existing validator.
3. Verify values, including empty strings, retain their established validation/storage behavior.

##### Expected Results

- Only exact case-sensitive admin.osac.openshift.io/ matching invokes the reserved rule; there is no subdomain inheritance, wildcard, or case normalization.
- Operational isolation/lifecycle keys are not reclassified as reserved. Existing validation and resource-specific ownership restrictions still apply to ordinary keys.
- Invalid keys fail under the existing syntax contract; this feature adds no broader acceptance.

#### TC-IS4-03: Concurrent tenant updates preserve the latest hidden metadata

| Interface Change | Priority | Automation |
|---|---|---|
| IC-3, IC-4 | critical | automated |

**Tier:** Real-PostgreSQL component integration; deployed Fulfillment

##### Preconditions

The tenant reads version v; an administrator changes a reserved entry and advances the row before the tenant submits its ordinary update.

##### Steps

1. Submit the stale ordinary tenant update with lock=true and inspect the row.
2. Repeat with the API legacy unlocked-update path where supported, controlling row acquisition order in the integration fixture.
3. Inspect both ordinary changes and the latest reserved values using an authorized private read.

##### Expected Results

- The locked stale request returns Aborted and changes nothing.
- The allowed unlocked ordinary update preserves the latest protected entries from the row read under lock, rather than restoring the tenant snapshot or an earlier administrator value.
- Concurrent requests never lose a committed reserved change through public-map replacement.

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

#### TC-IS4-05: Ordinary clients and existing service mutation boundaries remain compatible

| Interface Change | Priority | Automation |
|---|---|---|
| IC-2, IC-8, IC-9, IC-11 | high | automated |

**Tier:** CLI/UI unit; deployed Fulfillment

##### Preconditions

Use an older ordinary CLI fixture against the new server, updated clients against a legacy support fixture, existing ordinary console controls, and public networking services. Include private networking fixtures with a recorded baseline of ordinary label/annotation edits that their existing services permit.

##### Steps

1. Perform established ordinary label/annotation operations through each supported legacy client path.
2. Repeat ordinary console edits on mutable public resources and inspect projected responses.
3. Attempt public networking Update through the existing service boundary and compare the baseline failure.
4. Replay the recorded supported private networking metadata operations using legacy masks and replacement requests; compare ordinary entries and hidden reserved entries through an authorized private read.

##### Expected Results

- Existing ordinary supported operations retain their behavior and do not require reserved edit affordances.
- Hidden entries survive ordinary updates on the new server. Public networking Update remains absent/Unimplemented under the existing service contract.
- Capabilities neither grant authorization nor broaden mutability.
- Existing supported private networking metadata edits retain their behavior, while unprivileged map replacements preserve hidden reserved entries. Existing operational-label and immutable-field restrictions remain enforced.

### IS-5: Prefix and behavior documentation

#### TC-IS5-01: Documentation defines the exact prefix and matching examples

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

- Every guide names admin.osac.openshift.io/ exactly and explains case-sensitive prefix matching without inheritance or configurable variants.
- Examples agree with executable contract cases and preserve the distinction between reserved matching and existing syntax validation.

#### TC-IS5-02: Documentation covers persona parity, transports, errors, and compatibility

| Interface Change | Priority | Automation |
|---|---|---|
| IC-10, IC-11 | medium | manual |

**Tier:** Documentation review

##### Preconditions

Guide examples, CLI help, and API schema are available for the proposed implementation.

##### Steps

1. Review both admin persona journeys and both tenant persona expectations across API, CLI, and console.
2. Check gRPC, encoded REST query, and Connect set/clear examples; verify capability preflight, stale-version handling, and whole-request rejection instructions.
3. Check the all-resource inventory, ordinary-map preservation explanation, and absence of an in-place upgrade promise.

##### Expected Results

- Documentation describes identical admin actions on every metadata-bearing persistent type and clearly states tenant omission and atomic denial.
- Examples use the reviewed generic key-mask contract, include explicit removal intent and version handling, and do not recommend legacy fallback for a reserved write.
- Support guidance explains permission/compatibility/concurrency failures without logging protected values.

## Coverage summary and remaining work

- **30 cases:** 19 critical, 9 high, 2 medium, 0 low.
- **Automation:** 28 planned automated cases and two manual documentation cases. A case spanning tiers requires evidence at every listed relevant boundary.
- **Requirement coverage:** 5/5 local In Scope anchors mapped; no PRD scope bullet is omitted.
- **Interface coverage:** 11/11 design interface IDs mapped. The table below provides an independent index.

| Interface | Cases |
|---|---|
| IC-1 | TC-IS1-01, TC-IS1-02, TC-IS2-01, TC-IS2-06, TC-IS3-01, TC-IS4-02, TC-IS5-01 |
| IC-2 | TC-IS1-01, TC-IS1-03, TC-IS1-05, TC-IS1-06, TC-IS2-01, TC-IS4-05 |
| IC-3 | TC-IS1-02, TC-IS1-03, TC-IS1-04, TC-IS1-05, TC-IS1-09, TC-IS3-02, TC-IS3-03, TC-IS3-04, TC-IS3-06, TC-IS3-07, TC-IS4-01, TC-IS4-03 |
| IC-4 | TC-IS3-01, TC-IS3-02, TC-IS3-03, TC-IS3-04, TC-IS3-05, TC-IS3-06, TC-IS3-07, TC-IS3-08, TC-IS4-01, TC-IS4-03 |
| IC-5 | TC-IS2-01, TC-IS2-02, TC-IS2-04, TC-IS2-05, TC-IS2-06, TC-IS4-02, TC-IS4-04 |
| IC-6 | TC-IS2-02, TC-IS2-03, TC-IS4-04 |
| IC-7 | TC-IS1-01, TC-IS1-06, TC-IS1-08, TC-IS2-05, TC-IS3-01, TC-IS3-02 |
| IC-8 | TC-IS1-05, TC-IS1-07, TC-IS1-09, TC-IS2-04, TC-IS3-05, TC-IS4-05 |
| IC-9 | TC-IS1-06, TC-IS1-07, TC-IS1-09, TC-IS2-04, TC-IS3-06, TC-IS4-05 |
| IC-10 | TC-IS5-01, TC-IS5-02 |
| IC-11 | TC-IS1-07, TC-IS4-05, TC-IS5-02 |

### Execution gaps

| Gap | Required work / owner | Affected coverage | Ticket status and closure evidence |
|---|---|---|---|
| G-1 | QE with console maintainers: establish a maintained browser runner, correct discovery/configuration, add deployed metadata journeys, and document the execution command | TC-IS1-06, TC-IS2-04, TC-IS3-06; browser portions of stale/error cases | Work belongs to OSAC-6204; no separate follow-up ticket has been created. Close with committed suite/runner, reproducible command, deployed four-persona results, and saved failure artifacts. |
| G-2 | Fulfillment DEV with QE: create valid inventory fixtures and deployed identity credentials/seed setup for all 35 persistent types, both maps, both admins, both tenants, and restricted CSI identity | TC-IS1-01, TC-IS1-03, TC-IS1-05, TC-IS1-08, TC-IS2-01, TC-IS2-05, TC-IS3-01 and exhaustive regression matrix | Work belongs to OSAC-6204; no separate follow-up ticket has been created. Close with executable fixtures, explicit seeded lifecycle states, inventory completeness assertions, and real deployed transport/auth evidence. |

The generic Update representation and per-type networking validation work remain owned review items in the design. Question 2 determines which networking types need a reserved-only mode; TC-IS1-03 and TC-IS4-05 require preserving supported existing metadata updates. Once resolved, update the affected examples, cases, and generated-client expectations together. These are delivery prerequisites, not a reason to reduce the all-resource or interface-parity scope.

### Acceptance evidence

Completion requires real results for the exhaustive inventory and required boundary suites, not only a case count or mocked success. Record the executed command/cwd, commit, suite/tier, account setup, real dependencies and doubles, result artifacts, and any remaining gap with its owner. Reserved requests must leave the complete object/version and side effects unchanged; tenant response/filter evidence must contain no protected values. Both administrators must demonstrate the same API, CLI, and deployed console actions. Ordinary compatibility cases and the documentation reviews must pass before milestone graduation.
