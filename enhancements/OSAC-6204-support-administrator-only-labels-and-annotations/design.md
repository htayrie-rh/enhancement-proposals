---
title: support-administrator-only-labels-and-annotations
authors:
  - Haim Tayrie <htayrie@redhat.com>
creation-date: 2026-10-13
last-updated: 2026-10-14
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-6204
prd:
  - prd.md
see-also:
  - /enhancements/OSAC-1433-unified-networking
replaces: []
superseded-by: []
---

# Support Administrator-Only Labels and Annotations

## Summary

OSAC reserves `admin.osac.openshift.io/` for metadata that administrators manage through the private API, CLI, and console. The server hides these entries from tenants and rejects tenant changes, using existing metadata maps and an added mask for exact key edits. See [PRD](prd.md) for detailed requirements.

## Motivation

The public API currently returns all labels and annotations. Simply hiding reserved entries would create two update problems: replacing a tenant's visible map could delete hidden entries, and the CLI could silently ignore a forbidden removal because the fetched object does not contain the key. [Codebase: fulfillment-service/internal/servers/generic_mapper.go; fulfillment-service/internal/servers/generic_server.go; fulfillment-service/internal/cmd/cli/annotate/annotate_cmd.go]

The server therefore needs explicit edit/removal targets and rules that preserve hidden entries. It must also protect event payloads and query results, including result counts. [Codebase: fulfillment-service/internal/servers/events_server.go; fulfillment-service/internal/database/dao/generic_dao_list.go]

### Goals

- Keep existing metadata maps, database columns, versions, and events.
- Apply one reserved-key rule to API requests, responses, events, and filters.
- Support exact edits and removals through the API, CLI, and console.
- Preserve ordinary updates and use existing authorization and transactions.

### Non-Goals

- Changing ordinary metadata behavior or adding public networking Update methods.
- Replacing metadata with typed resource fields or configurable prefix policies.
- Adding in-place upgrades, production services, or provisioning workflows.

## Proposal

Cloud Provider Admin and Cloud Infrastructure Admin have identical reserved-metadata permissions on every covered resource through the private API, CLI, and console. Tenant views omit reserved entries. A tenant request that names a reserved entry fails in full, including any ordinary changes in that request. [Locked: D3, D4, D5]

The server checks writes before processing them and filters responses before returning them. Updates preserve hidden stored entries. Clients use an exact-key mask to distinguish a deliberate removal from an entry omitted because it was hidden. [Research: §2, §4]

### Workflow Description

An administrator starts with private API access and an existing object:

1. Select the object through private Get/List, `osac login --private`, or the console's administrator metadata page.
2. Read its metadata and current version.
3. Submit the keys to set or remove, the object identifier, and the version with `lock=true`.
4. The server checks permissions, locks the stored row, validates the proposed changes, and saves them in one transaction.
5. Receive the full private object. If the version is stale, reload before retrying.

Tenants continue editing ordinary metadata. Supplying a reserved entry or explicitly targeting one for removal returns `PermissionDenied` and saves nothing. Omitting a hidden entry during an ordinary map replacement preserves it. [Locked: D4]

![Metadata update flow: authorization, protected-key rejection, validation, persistence, and full or filtered responses.](assets/metadata-update-flow.png)

The diagram shows where an Update request can fail and which metadata view it returns. Early rejection prevents side effects; response filtering changes a copy, leaving stored metadata intact.

### API Extensions

`IS-1` through `IS-5` refer to the PRD's five In Scope bullets, in order. These local anchors and the interface IDs below connect the design to [the test plan](testplan.md).

| ID | Change | Requirements |
|---|---|---|
| IC-1 | Reserve an exact prefix in both metadata maps; retain existing key/value validation | IS-1, IS-2, IS-3, IS-4, IS-5 |
| IC-2 | Administrator reads and writes across all 35 persistent private types; preserve supported metadata updates | IS-1, IS-2, IS-4 |
| IC-3 | Add `metadata_update_mask` to Update requests for exact set/clear operations, with mask validation and version locking | IS-1, IS-3, IS-4 |
| IC-4 | Reject forbidden Create/Update input with `PermissionDenied`; commit no changes | IS-3, IS-4 |
| IC-5 | Filter public and unprivileged private responses, nested Metadata, and Events.Watch | IS-2, IS-4 |
| IC-6 | Reject protected-key filters; evaluate other supported filters against visible metadata and return matching counts | IS-2, IS-4 |
| IC-7 | Reuse platform-admin authority for both personas; retain tenant and restricted-client permissions | IS-1, IS-2, IS-3 |
| IC-8 | Update CLI editing/output and server compatibility handling | IS-1, IS-2, IS-3, IS-4 |
| IC-9 | Add the console metadata page/editor; protect tenant display/search and preserve edits after errors | IS-1, IS-2, IS-3, IS-4 |
| IC-10 | Document the prefix, operations, errors, and API/CLI/console examples | IS-5 |
| IC-11 | Advertise metadata support through Capabilities.Get | IS-1, IS-3, IS-4, IS-5 |

No CRDs, finalizers, webhooks, or AAP jobs change. Metadata edits use the API and database without waiting for provisioning controllers.

### Implementation Details/Notes/Constraints

#### Namespace and scope

A valid label or annotation is reserved when the part before `/` is exactly `admin.osac.openshift.io`. Matching is case-sensitive, without wildcard, case conversion, or subdomain inheritance. Existing key/value validation still applies, including uppercase name segments. [Research: §1]

| Key | Classification |
|---|---|
| `admin.osac.openshift.io/tracking-id` | Reserved |
| `child.admin.osac.openshift.io/tracking-id` | Ordinary |
| `admin.osac.openshift.io.example/tracking-id` | Ordinary |
| `osac.openshift.io/tenant`, `osac.openshift.io/owner-reference`, `admin`, `system` | Ordinary |

The prefix has no matches in the proto, service, or frontend source searched during drafting. Existing isolation and lifecycle keys keep their meaning. Imported data using the prefix must be checked before use.

All 35 persistent types are covered:

| Area | Types |
|---|---|
| Clusters | Cluster, ClusterTemplate, ClusterCatalogItem, ClusterVersion |
| Compute | ComputeInstance, ComputeInstanceTemplate, ComputeInstanceCatalogItem, InstanceType, DiskImage |
| Bare metal | BareMetalInstance, BareMetalInstanceTemplate, BareMetalInstanceCatalogItem, BareMetalInstanceType |
| Add-ons | AddOnOperator |
| Networking | NetworkClass, VirtualNetwork, Subnet, SecurityGroup, ExternalIPPool, ExternalIP, ExternalIPAttachment, NATGateway, FabricDomain |
| Storage | StorageBackend, StorageTier, Volume |
| Administration | Hub, Tenant, User, IdentityProvider, Project, ProjectMembership, Role, RoleBinding, Secret |

Each already defines private Update. Deployments still choose which services to enable. Hub, NetworkClass, StorageBackend, and Volume are private-only.

SelfSubjectAccessReview also carries Metadata, so its input and output follow the protection rules. It only evaluates a permission check and returns the result; it is not stored and gains no Get/List/Update/Delete methods. [Codebase: proto/private/osac/private/v1/*_type.proto; proto/private/osac/private/v1/*_service.proto]

#### Editing individual metadata keys

Add this helper to every existing Update request at its next unused field number:

```protobuf
message MetadataUpdateMask {
  repeated string labels = 1;
  repeated string annotations = 2;
}

// Example Update request:
message ComputeInstancesUpdateRequest {
  ComputeInstance object = 1;
  google.protobuf.FieldMask update_mask = 2;
  bool lock = 3;
  MetadataUpdateMask metadata_update_mask = 4;
}
```

Generate the public form where Update exists; networking Update stays private. Exact string keys avoid the installed protobuf JSON codec's rejection of `/` and quoting in FieldMask paths. [Research: §2] [Codebase: fulfillment-service/go.mod; google.golang.org/protobuf/encoding/protojson/well_known_types.go]

With the helper present:

| Input | Result |
|---|---|
| Target key is present in its object map | Set that value; an empty string is a value |
| Target key is absent from its object map | Remove it; an absent authorized key is a no-op |
| Key is not targeted | Leave it unchanged |
| Helper has no targets, duplicate targets within a map, or invalid keys | `InvalidArgument` |

An ordinary `update_mask` may select other fields or the other map. It cannot select `metadata`, a map selected by the helper, or descendants of that map; overlap returns `InvalidArgument`. Without an ordinary mask, only the helper's targets change. Identifier and version remain control inputs.

Without the helper, existing ordinary update behavior continues. Tenant map, parent-metadata, and unmasked replacements retain reserved entries from the locked current row. Candidate preparation cannot change identity or ownership.

Unprivileged requests reject every supplied reserved entry or explicit target, even if unchanged, absent, or outside the mask. Legacy explicit protected-key paths are also rejected; invalid paths remain errors.

REST keeps `body: "object"` and carries helper targets in repeated query parameters. For example, this clears a label:

```text
PATCH /api/private/v1/compute_instances/<id>?metadata_update_mask.labels=admin.osac.openshift.io%2Ftracking-id&lock=true
Body: {"metadata":{"version":7}}
```

The gateway currently derives an ordinary mask from PATCH bodies. Add a QueryParameterParser wrapper: run DefaultQueryParser, then clear the derived mask when the helper is present and no explicit `update_mask` query parameter exists. Keep explicit masks and check overlap. Generated handlers remain generated. [Codebase: proto/gen/osac/private/v1/compute_instances_service.pb.gw.go; grpc-gateway/runtime/query.go]

Connect and native gRPC use the full request: object ID, version 7, `lock=true`, and the same helper target, with the removed key absent from the map.

Add `MetadataCapabilities metadata = 3` to CapabilitiesGetResponse, containing `reserved_prefix` (string) and `supports_key_masks` (bool). The new server advertises `admin.osac.openshift.io/` and `true`. Updated clients require this advertisement before sending helper masks; it grants no permission. [Codebase: proto/private/osac/private/v1/capabilities_service.proto]

#### Permissions, validation, and response filtering

Carry OPA's evaluated `is_admin` boolean in AuthzDecision and Subject; default it to false. OPA is the existing permission policy engine. Retain the original external API method when a public handler delegates to private code. [Codebase: fulfillment-service/internal/auth/authorization_evaluator.go; fulfillment-service/internal/auth/auth_subject.go]

| Original API call | Reserved reads and writes |
|---|---|
| Private, authorized administrator | Read/add/change/remove |
| Public, any caller | Hidden; writes rejected |
| Private, non-admin caller | Hidden; writes rejected |

Existing method/object authorization still applies. Broad tenant visibility does not establish administrator authority.

Use a shared request/response interceptor after authorization and before automatic dependency creation or reference processing. It rejects protected input early, then removes reserved entries from a copy of each unprivileged response. Filtering covers nested public/private Metadata messages and streamed events, without touching unrelated maps. Internal objects remain complete. Events.Watch filters metadata before evaluating tenant event predicates. [Codebase: fulfillment-service/internal/cmd/service/start/grpcserver/start_grpc_server_cmd.go; fulfillment-service/internal/servers/events_server.go]

GenericServer builds the proposed object (the candidate) from the locked current row and both masks, then validates it before saving. Public handlers forward both masks and `lock`. TxInterceptor calls `tx.ReportError(&err)` before ending the transaction, so a handler error rolls back earlier writes too. Use the existing transaction and row read. [Locked: D4] [Codebase: fulfillment-service/internal/servers/generic_server.go; fulfillment-service/internal/database/database_tx_interceptor.go]

For unprivileged List/watch filters, inspect the parsed, type-checked CEL expression. Explicit protected-key references—including membership checks and mixed predicates—return `PermissionDenied` before item/count queries, regardless of whether the key exists. Ordinary literal-key filters retain existing indexed SQL. Supported whole-map filters use JSONB with reserved entries excluded; unsupported operators keep their errors. Authorized private administrator filters use full metadata. [Research: §5] [Codebase: fulfillment-service/internal/database/dao/filter_translator.go]

#### Editing networking metadata

Reuse existing private Update methods for NetworkClass, VirtualNetwork, Subnet, SecurityGroup, ExternalIPPool, ExternalIP, ExternalIPAttachment, and NATGateway. Both administrators must be able to set, change, and remove reserved entries through the API, CLI, and console. Where those methods already support ordinary metadata updates, preserve them, including legacy requests and mixed ordinary/reserved edits allowed by existing rules. [Locked: D3, D5] [User] [Codebase: fulfillment-service/internal/servers/private_virtual_networks_server.go; fulfillment-service/internal/servers/private_subnets_server.go]

Keep existing checks on specification, status, ownership, and operational labels. Add a narrowly scoped reserved-metadata mode only for types whose validators block the required edits. In that mode, use a nonempty reserved-key mask with no ordinary mask and reject other changes with `InvalidArgument`; do not apply this restriction to already-supported ordinary metadata updates. Validate metadata without repeating unrelated creation-time provider/readiness checks. Existing controller status operations continue, and public Update/PATCH remains absent.

The [networking design](../OSAC-1433-unified-networking/design.md#api-operation-constraint) declares metadata immutable. Review each private service's actual behavior and reconcile that contract with supported metadata editing; question 2 identifies the remaining validation work. [Codebase: proto/private/osac/private/v1/*_service.proto]

#### CLI, console, and storage

The CLI keeps `label`/`annotate` syntax, including `key=value` and `key-`. On supported servers it sends every explicit target, including removal of an unseen key, with version locking. Mixed edits use one request; errors produce a nonzero exit. Ordinary operations retain the older-server path; reserved edits on unsupported servers fail locally before mutation. Get/describe uses the server's filtered view.

The console adds `/admin/metadata` with a type selector, object list, and shared editor for all 35 private services. Existing detail pages link to it. Both admins can edit either map, including explicit removals. The editor uses version locking and retains unsaved changes after errors. Tenant pages retain ordinary editing and use filtered data for display/search. Private-only types stay in administrator navigation. [Locked: D3, D5]

This feature reuses existing platform-admin authorization. Both administrator personas have the same reserved-metadata permissions. The console derives `isPlatformAdmin` from the existing `admins` group claim and sends the user's token for every operation. [Locked: D5] [User] [Codebase: osac-ui/apps/app-frontend/src/hooks/oidc-login.tsx; osac-ui/proxy/middleware/auth.go]

Keep existing metadata columns/indexes, versions, and events; no database migration or Helm setting is added. Regenerate private/public protos and UI types, and update API/CLI/user guides and component testing maps.

### Security Considerations

Enforce protection on the server. Headers, UI role selection, schema-reflection credentials, and cross-tenant scope cannot grant administrator authority. Errors and logs never include stored metadata values. Existing tenant/project checks and tenant/owner isolation annotations remain in place. [Research: §4, §5]

The Container Storage Interface (CSI) driver is a restricted private client with current cross-tenant scope. Its permitted Volume reads must still omit reserved entries. Include it in authorization regression coverage. [Codebase: fulfillment-service/internal/auth/policies/authz.rego]

### Failure Handling and Recovery

| Failure | Result and recovery |
|---|---|
| Protected input or query | `PermissionDenied`; no writes or existence disclosure. Remove the prohibited input. |
| Invalid key, duplicate target, overlapping masks, or prohibited networking edit | `InvalidArgument`; no changes. Correct the request. |
| Stale version with locking | `Aborted`; no changes. Reload before retrying. |
| Object outside tenant/project scope | Existing denial/not-found behavior. |
| Database/transaction failure | Existing `Internal`/`Aborted` mapping and rollback. Reload before retrying. |
| Unsupported server | Clients block reserved edits before sending Update; ordinary legacy operations continue. |
| Console proxy/API unavailable | Show the connection error and allow retry; never report success. |

Repeated sets to the same value and authorized absent-key clears are no-ops. A forbidden target always fails, even when absent. No provisioning recovery state is added.

### RBAC / Tenancy

Both human admin personas have the same resource and interface coverage. Tenant users retain existing tenant/project and shared-object access. Restricted private clients remain non-admin; recognized administrator/controller identities retain trusted private operations. No new roles or resources are added. [Locked: D5]

The authorization table above defines reserved-metadata access. Ordinary metadata retains existing permissions and resource immutability rules.

### Observability and Monitoring

Use existing RPC status metrics and request logs for permission, validation, concurrency, and database failures. Logs may identify the method and supplied key, never metadata values. No new metrics or Kubernetes events are added.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| A service drops the mask or exposes metadata | Cover all 35 types, nested Metadata, and gRPC/REST/Connect routes. |
| An older server ignores the new helper | Require capabilities; never fall back for reserved edits. |
| Tenant replacement loses a concurrent admin edit | Preserve reserved entries from the locked current row. |
| Filters reveal hidden data or slow ordinary queries | Check protected references, filter whole maps, retain ordinary index paths, and inspect whole-map query plans. |
| UI and API authority differ | Use the same `admins` claim and enforce permissions on every request. |
| Networking edits bypass existing restrictions | Keep existing validators. Compare nonreserved fields with stored data only in a reserved-only mode needed to enable otherwise blocked edits. |

API/authentication and console maintainers review their boundaries and the open questions below.

### Drawbacks

The helper requires changes to every Update handler and generated client. The console registry must track new resource types. Supporting both legacy map replacement and exact-key editing adds maintenance, while preserving existing ordinary clients.

## Alternatives (Not Implemented)

| Alternative | Benefit | Reason for rejection |
|---|---|---|
| Reserve all `osac.openshift.io/` keys | Reuse one prefix | Changes existing isolation/lifecycle metadata behavior. |
| Put qualified keys in FieldMask | Use one mask type | Installed JSON codecs reject the paths; custom codecs add transport maintenance. |
| JSON Merge Patch / JSON Patch | Explicit removal | Adds a separate format/parser across gRPC, REST, and Connect. |
| Ignore forbidden entries | Simple merging | Violates whole-request rejection. |
| Reject all tenant replacements containing hidden data | Simple validation | Breaks ordinary updates. |
| Filter only in the frontend or CR admission | Fewer API changes | Leaves direct API calls, events, and queries exposed. |
| Separate storage/maps or configurable matching | More policy flexibility | Adds storage/policy work beyond one fixed namespace. |

## Open Questions

### 1. Generic Update extension

Do API maintainers accept `MetadataUpdateMask` as the generic exact-key update mechanism? The API guidelines prefer standard request fields; the JSON FieldMask limitation motivates this extension.

- **Owner:** Fulfillment Service API maintainers.
- **Impact:** IC-3, IC-8, IC-9, IC-11 and generated clients. Another representation changes the examples and transport tests.

### 2. Networking metadata update capability

Which networking types, if any, need validation changes to support reserved metadata through their existing private Update methods? Inventory current ordinary and reserved-key edit/removal behavior, preserve supported ordinary updates, and add a reserved-only mode only where necessary. Reconcile the networking document's metadata-immutability rule with the supported behavior. Support for both admins on all covered networking resources is required. [User]

- **Owner:** Fulfillment Service API and networking maintainers.
- **Impact:** IC-2, IC-3, IC-8, IC-9 and cases TC-IS1-03 / TC-IS4-05. The inventory determines which validators need changes and which existing behaviors require compatibility coverage.

## Test Plan

The [test plan](testplan.md) contains 30 planned cases covering all five scope bullets and 11 interfaces. It records execution commands, real versus mocked dependencies, and remaining harness/fixture work. No implementation tests were run during drafting.

### Unit Tests

- Prefix matching, key validation, explicit target rejection, and admin/capability defaults.
- Exact set/clear, legacy hidden-entry preservation, overlapping masks, stale versions, and rollback handling.
- Nested response/event filtering and protected versus ordinary queries.
- CLI target retention and console registry, compatibility, and unsaved-error state.

### Integration Tests

- All 35 types and both admin personas through deployed API, Keycloak, PostgreSQL, REST, and CLI; extend label/annotation suites and add inventory/transport coverage.
- Real PostgreSQL rollback/concurrency and Kafka event/filter coverage in the component suites.
- Forbidden mixed requests leave the complete private object, version, and side effects unchanged.

### E2E Tests

Both admins manage reserved metadata through the deployed console/proxy/API. Both tenant personas see no reserved keys or values. Existing login/scratch Playwright coverage does not provide this regression suite; add a maintained runner under gap G-1 in the test plan. Full deployed resource/identity fixtures are gap G-2.

## Graduation Criteria

For 0.4 Tech Preview, resolve both open questions, document the contract, and pass all required coverage: 35 types, both admins, API/CLI/console parity, tenant privacy, atomic rejection, concurrency, and ordinary compatibility. Close G-1 and G-2 with maintained suites and recorded execution results.

## Upgrade / Downgrade Strategy

No in-place upgrades are added. Existing storage and field numbers remain intact. Check imported state for the new prefix. Reverting the server removes its privacy protection; remove provider-only values before exposing that state through an older deployment. No automatic cleanup is provided.

## Version Skew Strategy

Older ordinary clients keep working against the new server. Updated clients require advertised support before key-mask edits, including native gRPC clients whose unknown fields could otherwise be ignored. Console types/proxy schemas must match the deployed API. Existing controller clients and CRDs need no new fields.

## Support Procedures

Use API status metrics/logs and an authorized private Get to inspect failures. Permission errors require corrected input/access; stale versions require reload. Compare gRPC, REST, and Connect requests for transport problems without logging metadata values.

To stop an account's reserved writes, remove its admin group membership. Console access can also be withdrawn during investigation. Server protection remains active; existing workloads continue. After repairing access/configuration, reload before retrying. No switch disables metadata privacy.

## Infrastructure Needed

Reuse existing Kind API/database/Keycloak and Kafka harnesses. Add the maintained browser runner and complete resource/persona fixtures described by G-1 and G-2. Their owners and missing follow-up tickets are recorded in the test plan. No new production infrastructure is needed.

---

## Provenance

Authored: revise @ design 0.11.5 - 1c6e8de, workspace main @ f0a8211
Phases: draft, revise, revise, revise, revise, revise, revise, revise

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.5","ai_workflows":"1c6e8de","source_repo":"f0a8211","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["draft","revise","revise","revise","revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":false,"origin_untracked":false} -->
