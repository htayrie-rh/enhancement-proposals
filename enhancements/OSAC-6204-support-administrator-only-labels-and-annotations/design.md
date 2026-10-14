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

OSAC reserves `admin.osac.openshift.io/` for metadata that administrators manage through the private API, CLI, and console. Existing whole-map updates replace administrator maps and preserve stored reserved entries in tenant maps. See [PRD](prd.md) for detailed requirements.

## Motivation

The public API currently returns all labels and annotations. Simply hiding reserved entries would create two update problems: replacing a tenant's visible map could delete hidden entries, and the CLI could silently ignore a forbidden removal because the fetched object does not contain the key. [Codebase: fulfillment-service/internal/servers/generic_mapper.go; fulfillment-service/internal/servers/generic_server.go; fulfillment-service/internal/cmd/cli/annotate/annotate_cmd.go]

The server preserves reserved entries during tenant updates and rejects supplied reserved keys. CLI and console commands reject named forbidden removals before constructing the update map. The same privacy boundary covers event payloads and query results, including result counts. [Codebase: fulfillment-service/internal/servers/events_server.go; fulfillment-service/internal/database/dao/generic_dao_list.go] [User]

### Goals

- Keep existing metadata maps, database columns, versions, and events.
- Apply one reserved-key rule to API requests, responses, events, and filters.
- Reuse existing Update requests, field masks, and version locking across the API, CLI, and console.
- Preserve ordinary updates and use existing authorization and transactions.

### Non-Goals

- Changing ordinary metadata behavior or adding public networking Update methods.
- Replacing metadata with typed resource fields or configurable prefix policies.
- Adding in-place upgrades, production services, or provisioning workflows.

## Proposal

Cloud Provider Admin and Cloud Infrastructure Admin have identical reserved-metadata permissions on every covered resource through the private API, CLI, and console. Tenant views omit reserved entries. A tenant request that names a reserved entry fails in full, including any ordinary changes in that request. [Locked: D3, D4, D5]

The server checks writes before processing them and filters responses before returning them. Tenant updates always retain reserved entries from the locked stored object. Authorized private administrators replace selected maps, including deletion by omission. Clients use the existing `update_mask` to select maps; no new per-key mask is added. [Locked: D6] [User]

### Workflow Description

An administrator starts with private API access and an existing object:

1. Select the object through private Get/List, `osac login --private`, or the console's administrator metadata page.
2. Read its metadata and current version.
3. Edit a copy of the complete private map, retaining untouched entries. Submit that map, its existing `update_mask`, the object identifier, and the version with `lock=true`.
4. The server checks permissions, locks the stored row, validates the proposed changes, and saves them in one transaction.
5. Receive the full private object. If the version is stale, reload before retrying.

Tenants continue editing ordinary metadata. A supplied reserved entry or an accepted explicit reserved-key mask path returns `PermissionDenied` and saves nothing. Updated CLI/console controls reject a named reserved removal locally before sending any Update. Map omission always preserves hidden entries; it does not tell the server whether removal was intended. [Locked: D4, D6]

![Metadata update flow: authorization, protected-key rejection, validation, persistence, and full or filtered responses.](assets/metadata-update-flow.png)

The diagram shows how the server applies an existing map update, preserves reserved entries for tenants, and chooses the response view. Client checks run before Update; early server rejection prevents side effects, and response filtering changes only a copy.

### API Extensions

`IS-1` through `IS-5` refer to the PRD's five In Scope bullets, in order. These local anchors and the interface IDs below connect the design to [the test plan](testplan.md).

| ID | Change | Requirements |
|---|---|---|
| IC-1 | Reserve an exact prefix in both metadata maps; retain existing key/value validation | IS-1, IS-2, IS-3, IS-4, IS-5 |
| IC-2 | Administrator reads and writes across all 35 persistent private types; preserve supported metadata updates | IS-1, IS-2, IS-4 |
| IC-3 | Reuse existing map replacement and `update_mask`; preserve tenant reserved entries from the locked row and use version locking | IS-1, IS-3, IS-4 |
| IC-4 | Reject forbidden Create/Update input with `PermissionDenied`; reject named tenant operations in clients before Update; commit no changes | IS-3, IS-4 |
| IC-5 | Filter public and unprivileged private responses, nested Metadata, and Events.Watch | IS-2, IS-4 |
| IC-6 | Reject protected-key filters; evaluate other supported filters against visible metadata and return matching counts | IS-2, IS-4 |
| IC-7 | Reuse platform-admin authority for both personas; retain tenant and restricted-client permissions | IS-1, IS-2, IS-3 |
| IC-8 | Update CLI editing/output and server compatibility handling | IS-1, IS-2, IS-3, IS-4 |
| IC-9 | Add the console metadata page/editor; protect tenant display/search and preserve edits after errors | IS-1, IS-2, IS-3, IS-4 |
| IC-10 | Document the prefix, operations, errors, and API/CLI/console examples | IS-5 |
| IC-11 | Advertise reserved-namespace enforcement through Capabilities.Get before clients submit reserved writes | IS-1, IS-3, IS-4, IS-5 |

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

#### Creating objects with metadata

Create uses the existing labels and annotations fields. Authorized private administrators may supply reserved entries alongside ordinary metadata. Public and non-admin private requests supplying a reserved entry fail before object creation or dependency processing; no ordinary subset is saved. Create has no stored map to merge. Existing resource validation and ownership rules still apply. [Locked: D4, D5]

#### Updating metadata maps

Keep the existing Update request:

```protobuf
message ComputeInstancesUpdateRequest {
  ComputeInstance object = 1;
  google.protobuf.FieldMask update_mask = 2;
  bool lock = 3;
}
```

Clients select `metadata.labels`, `metadata.annotations`, or both with the existing mask. They send the complete intended contents of each selected map. Qualified keys remain ordinary map entries, avoiding per-key FieldMask encoding through REST and Connect. Supplied reserved input is rejected for unprivileged callers even when its map is excluded by the mask. [Codebase: fulfillment-service/internal/servers/generic_server.go]

| Input selecting a metadata map | Tenant/public caller | Authorized private administrator |
|---|---|---|
| Ordinary entries only | Replace ordinary entries; restore stored reserved entries | Replace the entire selected map |
| Reserved entry supplied | Reject the whole request, even if unchanged or outside the mask | Store it alongside the other supplied entries |
| Empty or omitted selected map | Clear ordinary entries; retain reserved entries | Clear the selected map |
| Map excluded by the mask | Leave it unchanged | Leave it unchanged |

An empty string remains a value. Deleting an authorized key means omitting it from the submitted replacement map. Existing parent-metadata and unmasked updates keep their resource-specific behavior; all tenant forms restore reserved entries after applying the existing update semantics. Unmasked updates do not acquire a new implied mask. Existing accepted per-key paths remain supported, but unprivileged explicit reserved targets are denied. Invalid paths remain errors. [Locked: D6] [Codebase: fulfillment-service/internal/servers/generic_server.go]

The client reads the complete map available to its API view, applies all requested edits locally, and sends only the selected maps plus identifier/version. Updated metadata clients use `lock=true`; a stale map fails with `Aborted` and must be reloaded before resubmission. The server restores tenant reserved entries from its latest locked row even for existing unlocked clients.

REST keeps `body: "object"` and the generated gateway's existing body-derived mask. Explicit map-level query masks also work. For example, replacing labels with this map removes an omitted reserved key for a private administrator, while a tenant replacement retains it:

```text
PATCH /api/private/v1/compute_instances/<id>?update_mask=metadata.labels&lock=true
Body: {"metadata":{"version":7,"labels":{"example.org/environment":"prod"}}}
```

Sending an empty labels map with the same mask clears the map under the rules above. The explicit mask keeps version as a control input. No custom query parser is needed. Connect, the console's RPC transport, and native gRPC send the same maps in the full Update request with `update_mask.paths = ["metadata.labels"]` and `lock=true`. [Codebase: proto/gen/osac/private/v1/compute_instances_service.pb.gw.go]

Add `MetadataCapabilities metadata = 3` to CapabilitiesGetResponse, with `string reserved_prefix = 1`. Advertise `admin.osac.openshift.io/` only when reserved-write protection and response/event/query privacy are implemented. Updated clients require the matching prefix before reserved Create/Update operations: older servers can accept these maps but expose their values. The advertisement grants no permission. [Codebase: proto/private/osac/private/v1/capabilities_service.proto]

#### Permissions, validation, and response filtering

Carry OPA's evaluated `is_admin` boolean in AuthzDecision and Subject; default it to false. OPA is the existing permission policy engine. Retain the original external API method when a public handler delegates to private code. [Codebase: fulfillment-service/internal/auth/authorization_evaluator.go; fulfillment-service/internal/auth/auth_subject.go]

| Original API call | Reserved reads and writes |
|---|---|
| Private, authorized administrator | Read/add/change/remove |
| Public, any caller | Hidden; writes rejected |
| Private, non-admin caller | Hidden; writes rejected |

Existing method/object authorization still applies. Broad tenant visibility does not establish administrator authority.

Use a shared request/response interceptor after authorization and before automatic dependency creation or reference processing. It rejects protected input early, then removes reserved entries from a copy of each unprivileged response. Filtering covers nested public/private Metadata messages and streamed events, without touching unrelated maps. Internal objects remain complete. Events.Watch filters metadata before evaluating tenant event predicates. [Codebase: fulfillment-service/internal/cmd/service/start/grpcserver/start_grpc_server_cmd.go; fulfillment-service/internal/servers/events_server.go]

GenericServer reads the current row under its existing lock, checks the requested version, and applies the existing field mask or unmasked replacement. It then restores the stored reserved portion of both maps for unprivileged callers before candidate/resource validation and persistence. Raw-input rejection runs before this merge; restored entries are trusted stored data. Do not restore ordinary entries removed by the request or change identity/ownership. Public handlers keep forwarding the existing mask and `lock`. [Locked: D4, D6] [Codebase: fulfillment-service/internal/servers/generic_server.go]

TxInterceptor calls `tx.ReportError(&err)` before ending the transaction, so a handler error rolls back earlier writes too. Use the existing transaction and row read. [Locked: D4] [Codebase: fulfillment-service/internal/database/database_tx_interceptor.go]

For unprivileged List/watch filters, inspect the parsed, type-checked CEL expression. Explicit protected-key references—including membership checks and mixed predicates—return `PermissionDenied` before item/count queries, regardless of whether the key exists. Ordinary literal-key filters retain existing indexed SQL. Supported whole-map filters use JSONB with reserved entries excluded; unsupported operators keep their errors. Authorized private administrator filters use full metadata. [Research: §5] [Codebase: fulfillment-service/internal/database/dao/filter_translator.go]

#### Editing networking metadata

Reuse existing private Update methods for NetworkClass, VirtualNetwork, Subnet, SecurityGroup, ExternalIPPool, ExternalIP, ExternalIPAttachment, and NATGateway. All eight current paths accept map-level metadata edits, subject to existing resource validation. Both administrators use the same map replacement through the API, CLI, and console. Preserve supported ordinary and mixed metadata edits. [Locked: D3, D5] [User] [Codebase: fulfillment-service/internal/servers/private_network_classes_server.go; fulfillment-service/internal/servers/private_virtual_networks_server.go; fulfillment-service/internal/servers/external_ip_lifecycle.go]

ExternalIP, ExternalIPAttachment, and NATGateway require a nonempty existing `update_mask` and permit metadata paths; clients supply the map-level mask. Keep specification/status/ownership checks and system-default-label restrictions. Editing a fetched map retains untouched operational labels. No reserved-only mode is added. Existing controller status operations continue, and public Update/PATCH remains absent. [Codebase: fulfillment-service/internal/servers/external_ip_lifecycle.go; fulfillment-service/internal/servers/default_networking_validation.go]

The [networking design](../OSAC-1433-unified-networking/design.md#api-operation-constraint) declares metadata immutable. The remaining review question reconciles that document with the existing private metadata paths and this feature's required administrator access. Integration fixtures must verify the allowed edits and lifecycle restrictions for every type.

#### CLI, console, and storage

The CLI keeps `label`/`annotate` syntax, including `key=value` and `key-`. Before changing the fetched map, it checks every named operation against the caller's API view and authority. A tenant reserved set/removal fails locally, including an unseen or absent key, and sends no Update for a mixed command. Allowed edits replace the fetched selected map using its existing mask and `lock=true`; errors produce a nonzero exit. Private mode selects the API; it does not grant permission, and existing server method authorization still applies. Reserved writes on unsupported servers fail before mutation. Get/describe uses the server's filtered view.

The console adds `/admin/metadata` with a type selector, object list, and shared editor for all 35 private services. Existing detail pages link to it. Both admins can edit either complete private map, retaining untouched entries. The editor submits changed maps with the existing mask and version locking and retains unsaved changes after errors. Tenant pages edit visible ordinary maps, reject named reserved operations locally, and use filtered data for display/search. Private-only types stay in administrator navigation. [Locked: D3, D5, D6]

This feature reuses existing platform-admin authorization. Both administrator personas have the same reserved-metadata permissions. The console derives `isPlatformAdmin` from the existing `admins` group claim and sends the user's token for every operation. [Locked: D5] [User] [Codebase: osac-ui/apps/app-frontend/src/hooks/oidc-login.tsx; osac-ui/proxy/middleware/auth.go]

Keep existing metadata columns/indexes, versions, and events; no database migration or Helm setting is added. Regenerate private/public protos and UI types, and update API/CLI/user guides and component testing maps.

### Security Considerations

Enforce protection on the server. Headers, UI role selection, schema-reflection credentials, and cross-tenant scope cannot grant administrator authority. Errors and logs never include stored metadata values. Existing tenant/project checks and tenant/owner isolation annotations remain in place. [Research: §4, §5]

The Container Storage Interface (CSI) driver is a restricted private client with current cross-tenant scope. Its permitted Volume reads must still omit reserved entries. Include it in authorization regression coverage. [Codebase: fulfillment-service/internal/auth/policies/authz.rego]

### Failure Handling and Recovery

| Failure | Result and recovery |
|---|---|
| Protected input or query | `PermissionDenied`; no writes or existence disclosure. Remove the prohibited input. |
| Named reserved CLI/console operation from an unprivileged view | Local permission error; no Update is sent and no ordinary subset is applied. |
| Invalid key/mask or prohibited resource edit | Existing validation error, including `InvalidArgument`; no changes. Correct the request. |
| Stale version with locking | `Aborted`; no changes. Reload before retrying. |
| Object outside tenant/project scope | Existing denial/not-found behavior. |
| Database/transaction failure | Existing `Internal`/`Aborted` mapping and rollback. Reload before retrying. |
| Unsupported server | Clients block reserved writes before Create/Update; ordinary legacy operations continue. |
| Console proxy/API unavailable | Show the connection error and allow retry; never report success. |

Replacing a map with identical contents is a no-op. An authorized absent-key removal is a no-op when the client retains all other entries. Supplied or explicitly named forbidden targets fail regardless of existence. A tenant replacement omitting reserved entries succeeds for its ordinary changes and retains those entries; the server cannot recover a removed target from an older client's map. No provisioning recovery state is added. [Locked: D6]

### RBAC / Tenancy

Both human admin personas have the same resource and interface coverage. Tenant users retain existing tenant/project and shared-object access. Restricted private clients remain non-admin; recognized administrator/controller identities retain trusted private operations. No new roles or resources are added. [Locked: D5]

The authorization table above defines reserved-metadata access. Ordinary metadata retains existing permissions and resource immutability rules.

### Observability and Monitoring

Use existing RPC status metrics and request logs for permission, validation, concurrency, and database failures. Logs may identify the method and supplied key, never metadata values. No new metrics or Kubernetes events are added.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| A service drops the existing mask or exposes metadata | Cover all 35 types, nested Metadata, and gRPC/REST/Connect routes. |
| An older server stores reserved entries without protecting reads | Require namespace enforcement capabilities before reserved writes. |
| Tenant replacement loses a concurrent admin edit | Preserve reserved entries from the locked current row. |
| Administrator replacement loses untouched entries | Edit the complete private map; use its existing mask and version locking. |
| A client erases a forbidden removal target before submitting a map | Check named CLI/console operations first; always preserve tenant reserved entries on the server. |
| Filters reveal hidden data or slow ordinary queries | Check protected references, filter whole maps, retain ordinary index paths, and inspect whole-map query plans. |
| UI and API authority differ | Use the same `admins` claim and enforce permissions on every request. |
| Networking edits bypass existing restrictions | Keep existing validators and required masks; cover operational labels and lifecycle states. |

API/authentication and console maintainers review their boundaries and the open questions below.

### Drawbacks

Whole-map editing requires clients to retain untouched entries and reload after conflicts. The server cannot distinguish a forbidden removal from omission once a client has discarded the target. CLI/console checks provide the named-operation error; stored-entry preservation provides API protection. The console registry must track new resource types.

## Alternatives (Not Implemented)

| Alternative | Benefit | Reason for rejection |
|---|---|---|
| Reserve all `osac.openshift.io/` keys | Reuse one prefix | Changes existing isolation/lifecycle metadata behavior. |
| Add a per-key metadata mask | Server receives explicit set/removal targets | Adds fields and forwarding to every Update request; the confirmed whole-map preservation rule uses the existing contract. |
| Put qualified keys in FieldMask | Use one mask type | Installed JSON codecs reject the paths; custom codecs add transport maintenance. |
| JSON Merge Patch / JSON Patch | Explicit removal | Adds a separate format/parser across gRPC, REST, and Connect. |
| Ignore forbidden entries | Simple merging | Violates whole-request rejection. |
| Reject all tenant replacements containing hidden data | Simple validation | Breaks ordinary updates. |
| Filter only in the frontend or CR admission | Fewer API changes | Leaves direct API calls, events, and queries exposed. |
| Separate storage/maps or configurable matching | More policy flexibility | Adds storage/policy work beyond one fixed namespace. |

## Open Questions

### 1. Networking metadata contract

How should the networking document describe administrator metadata edits through the existing private Update methods while retaining its public immutability boundary? All eight paths accept map-level metadata in the current source; preserve their supported ordinary edits and validate the complete behavior with fixtures. Both admins must have the required reserved-metadata access on every covered networking resource. [User]

- **Owner:** Fulfillment Service API and networking maintainers.
- **Impact:** IC-2, IC-3, IC-8, IC-9 and cases TC-IS1-03 / TC-IS4-05. Resolve the documented private metadata contract and any validation mismatch found by the integration matrix without introducing public Update or removing supported ordinary behavior.

## Test Plan

The [test plan](testplan.md) contains 29 planned cases covering all five scope bullets and 11 interfaces. It records execution commands, real versus mocked dependencies, and remaining harness/fixture work. No implementation tests were run during drafting or this revision.

### Unit Tests

- Prefix matching, key validation, explicit target rejection, and admin/capability defaults.
- Selected-map replacement, empty maps/values, tenant reserved-entry preservation, stale versions, and rollback handling.
- Nested response/event filtering and protected versus ordinary queries.
- CLI/console named-operation checks, map/mask forwarding, registry, compatibility, and unsaved-error state.

### Integration Tests

- All 35 types and both admin personas through deployed API, Keycloak, PostgreSQL, REST, and CLI; extend label/annotation suites and add inventory/transport coverage.
- Real PostgreSQL rollback/concurrency and Kafka event/filter coverage in the component suites.
- Forbidden mixed requests leave the complete private object, version, and side effects unchanged.

### E2E Tests

Both admins manage reserved metadata through the deployed console/proxy/API. Both tenant personas see no reserved keys or values. Existing login/scratch Playwright coverage does not provide this regression suite; add a maintained runner under gap G-1 in the test plan. Full deployed resource/identity fixtures are gap G-2.

## Graduation Criteria

For 0.4 Tech Preview, resolve the networking contract question, document the contract, and pass all required coverage: 35 types, both admins, API/CLI/console parity, tenant privacy, atomic rejection, concurrency, and ordinary compatibility. Close G-1 and G-2 with maintained suites and recorded execution results.

## Upgrade / Downgrade Strategy

No in-place upgrades are added. Existing storage and field numbers remain intact. Check imported state for the new prefix. Reverting the server removes its privacy protection; remove provider-only values before exposing that state through an older deployment. No automatic cleanup is provided.

## Version Skew Strategy

Older ordinary clients keep working against the new server, which preserves reserved entries from its locked stored row on every tenant update. An older client that discards a named reserved removal cannot report the new local permission error; its map omission still cannot delete stored reserved data. Updated clients require advertised namespace enforcement before reserved writes, because older servers accept the existing request shape without providing privacy. Ordinary operations retain their existing older-server path. Console types/proxy schemas must match the deployed API; Update requests and CRDs gain no fields.

## Support Procedures

Use API status metrics/logs and an authorized private Get to inspect failures. Permission errors require corrected input/access; stale versions require reload. Compare gRPC, REST, and Connect requests for transport problems without logging metadata values.

To stop an account's reserved writes, remove its admin group membership. Console access can also be withdrawn during investigation. Server protection remains active; existing workloads continue. After repairing access/configuration, reload before retrying. No switch disables metadata privacy.

## Infrastructure Needed

Reuse existing Kind API/database/Keycloak and Kafka harnesses. Add the maintained browser runner and complete resource/persona fixtures described by G-1 and G-2. Their owners and missing follow-up tickets are recorded in the test plan. No new production infrastructure is needed.

---

## Provenance

Authored: revise @ design 0.11.5 - 1c6e8de, workspace main @ f0a8211
Phases: draft, revise, revise, revise, revise, revise, revise, revise, revise

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.5","ai_workflows":"1c6e8de","source_repo":"f0a8211","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["draft","revise","revise","revise","revise","revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":false,"origin_untracked":false} -->
