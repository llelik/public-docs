# NetBox ONTAP Infrastructure Plugin

A NetBox plugin that brings full NetApp ONTAP storage topology into NetBox as a single source of truth — clusters, nodes, SVMs, volumes, qtrees, LUNs, interfaces, export policies, SnapMirror relationships, and their associated policies and schedules. Designed for automation-first workflows with a complete REST API that integrates directly with Ansible, Terraform, or any IaC toolchain.

## Why This Plugin

NetBox is the standard for network and infrastructure source of truth. ONTAP storage is typically managed separately — ONTAP System Manager, Active IQ, or custom scripts. This creates a gap:

- **No single pane** across compute, network, and storage
- **No programmatic source of truth** for storage topology that IaC tools can consume
- **No cross-domain relationships** — which SVM serves which tenant? Which volumes are on which aggregate? Which interfaces carry which services?
- **No SnapMirror topology visibility** — source/destination clusters, SVMs, volumes, policies, schedules all tracked as linked objects

This plugin closes that gap.

## Data Model

17 models covering the full ONTAP object hierarchy:

```
ONTAPCluster
├── ONTAPNode (1:N)
├── ONTAPTier / Aggregate (1:N)
├── ONTAPsvm (1:N)
│   ├── ONTAPVolume (1:N)
│   │   ├── ONTAPQtree (1:N)
│   │   │   └── ONTAPQuotaRule (1:N)
│   │   └── ONTAPLUN (1:N)
│   ├── ONTAPInterface (1:N)
│   ├── ONTAPExportPolicy (1:N)
│   │   └── ONTAPExportPolicyRule (1:N)
│   └── ONTAPSnapshotPolicy (1:N)
├── ONTAPSnapMirrorPolicy (1:N)
├── ONTAPJobSchedule (1:N)
└── ONTAPSnapMirror (N:N — cross-cluster relationships)
```

### Model Details

| Model | Key Fields | Unique Constraint | Notes |
|-------|-----------|-------------------|-------|
| **ONTAPCluster** | name, cluster_type, version, state, management_ip | name | Types: standalone, mcc_ip, mcc_fc, cvo, anf |
| **ONTAPNode** | name, serial_number, model, state, device (FK) | cluster + name | Links to NetBox Device for physical tracking |
| **ONTAPTier** | name, tier_type, node, size, state, snaplock_type | name | Types: aggregate, cloud. MCC-mirrored flag |
| **ONTAPsvm** | name, svm_type, subtype, admin_state, language, ipspace, enabled_services | cluster + name | Types: data/admin/system. Subtypes: default, dp_destination, sync_source, sync_destination. Auto-indexed per cluster |
| **ONTAPVolume** | name, voltype, size, volstate, encryption, security_style, language, snaplock_type, mount_path | svm + name | Types: RW/RO/DP. Autosize modes. Export policy + snapshot policy FKs |
| **ONTAPQtree** | name, security_style, export_policy, volume | volume + name | Links to parent volume and quota rules |
| **ONTAPQuotaRule** | type, volume, qtree, space hard/soft limits, file limits | — | Types: tree/user/group. Binary unit display |
| **ONTAPLUN** | name, os_type, size, volume, qtree | svm + name | 13 OS types. Links to volume and optional qtree |
| **ONTAPInterface** | name, ip_address, admin_state, home_node, home_port, ipspace, enabled_services | — | Up to 30 services. VLAN tag extraction. IPspace-aware IP uniqueness |
| **ONTAPExportPolicy** | name, svm | svm + name | Container for export rules |
| **ONTAPExportPolicyRule** | index, clientmatch, ro_rule, rw_rule, superuser, protocols | export_policy + index | Full NFS export ACL modeling |
| **ONTAPSnapshotPolicy** | name, scope, enabled, svm | — | Cluster-scoped (admin SVM) or SVM-scoped (data SVM) |
| **ONTAPSnapMirrorPolicy** | name, policy_type, sync_type, scope, identity_preservation, transfer_schedule | cluster + svm + name | Types: async/sync/continuous. Full `ontap_details` JSON storage |
| **ONTAPJobSchedule** | name, schedule_type, scope, cron_expression, interval_iso | cluster + svm + name | Types: cron/interval. Full `ontap_details` JSON storage |
| **ONTAPSnapMirror** | source/dest cluster, SVM, volumes (M2M), policy, schedule, protection_type | source_cluster + source_svm + dest_cluster + dest_svm | Types: SVMDR, VSM, CG. Cardinality enforced per type |

### Cross-Domain Relationships

Every model integrates with NetBox core:

- **Tenant** (FK) — Cluster, SVM, Volume, Interface, Export Policy
- **Site** (FK) — Cluster, Node, SVM, Interface
- **Device** (FK) — Node (physical chassis)
- **IPAddress** (FK) — Cluster (management), Interface
- **Owner** (FK) — All 17 models (NetBox 4.5+ owner assignment)
- **Tags** — All models (NetBox tagging system)
- **Custom Fields** — All models
- **Journal Entries** — All models
- **Change Logging** — Full audit trail on all models

## REST API

Base URL: `/api/plugins/ontap-nas/`

| Endpoint | Model | Methods |
|----------|-------|---------|
| `clusters/` | ONTAPCluster | GET, POST, PUT, PATCH, DELETE |
| `nodes/` | ONTAPNode | GET, POST, PUT, PATCH, DELETE |
| `tiers/` | ONTAPTier | GET, POST, PUT, PATCH, DELETE |
| `svms/` | ONTAPsvm | GET, POST, PUT, PATCH, DELETE |
| `volumes/` | ONTAPVolume | GET, POST, PUT, PATCH, DELETE |
| `qtrees/` | ONTAPQtree | GET, POST, PUT, PATCH, DELETE |
| `quota-rules/` | ONTAPQuotaRule | GET, POST, PUT, PATCH, DELETE |
| `luns/` | ONTAPLUN | GET, POST, PUT, PATCH, DELETE |
| `interfaces/` | ONTAPInterface | GET, POST, PUT, PATCH, DELETE |
| `export-policies/` | ONTAPExportPolicy | GET, POST, PUT, PATCH, DELETE |
| `export-policy-rules/` | ONTAPExportPolicyRule | GET, POST, PUT, PATCH, DELETE |
| `snapshot-policies/` | ONTAPSnapshotPolicy | GET, POST, PUT, PATCH, DELETE |
| `snapmirror-policies/` | ONTAPSnapMirrorPolicy | GET, POST, PUT, PATCH, DELETE |
| `job-schedules/` | ONTAPJobSchedule | GET, POST, PUT, PATCH, DELETE |
| `snapmirror-relationships/` | ONTAPSnapMirror | GET, POST, PUT, PATCH, DELETE |

All endpoints support:
- **Bulk create**: POST with JSON array
- **Filtering**: Full django-filter integration (see Filtering section)
- **Pagination**: Standard NetBox pagination
- **`?brief=1`**: Minimal response for dropdown population
- **Nested serializers**: GET responses include full nested objects (cluster, SVM, etc.)
- **OpenAPI schema**: Auto-generated at `/api/schema/`

### API Design Principles

**Server-side FK resolution** — pass names, get objects:

```bash
# Create a volume — pass SVM by name, server resolves to ID
curl -X POST /api/plugins/ontap-nas/volumes/ \
  -d '{
    "name": "vol_data01",
    "svm": {"name": "svm_prod", "cluster": {"name": "cluster01"}},
    "size": 107374182400,
    "voltype": "RW",
    "volstate": "online",
    "aggregate": {"name": "aggr1_node01"}
  }'
```

**Case-insensitive choice fields** — `rw`, `Rw`, `RW` all resolve to canonical `RW`:
```bash
curl -X POST /api/plugins/ontap-nas/volumes/ \
  -d '{"voltype": "rw", "volstate": "Online", "encryption": "enabled"}'
# All accepted, normalized to canonical values
```

**SnapMirror with policy resolution by name**:
```bash
curl -X POST /api/plugins/ontap-nas/snapmirror-relationships/ \
  -d '{
    "source_cluster": {"name": "cluster01"},
    "source_vserver": {"name": "svm_prod"},
    "source_volume": [1234],
    "destination_cluster": {"name": "cluster02"},
    "destination_vserver": {"name": "svm_dr"},
    "destination_volume": [5678],
    "snapmirror_policy": {"name": "MirrorAndVault"},
    "schedule": {"name": "hourly"},
    "protection_type": "VSM"
  }'
# Policy and schedule resolved from destination cluster context
```

**Export policy with ORM-style keys** (Ansible module compatibility):
```bash
curl -X POST /api/plugins/ontap-nas/volumes/ \
  -d '{
    "export_policy": {
      "name": "default",
      "svm__name": "svm_prod",
      "svm__cluster__name": "cluster01"
    }
  }'
```

### Filtering

Every FK field supports 3 filter patterns:

```
?cluster_id=5              # By ID (integer)
?cluster=jamaica           # By name (ModelChoiceFilter)
?cluster__name=jamaica     # By ORM traversal (MultiValueCharFilter)
```

Choice fields use the class reference directly (OpenAPI-safe):
```
?voltype=RW&volstate=online
?protection_type=SVMDR&protection_type=VSM   # Multi-value
?svm_type=data
```

Combined filtering example:
```
GET /api/plugins/ontap-nas/volumes/?cluster__name=cluster01&svm__name=svm_prod&voltype=RW&volstate=online
```

### Bulk Operations

All endpoints support bulk create via JSON array:

```bash
curl -X POST /api/plugins/ontap-nas/svms/ \
  -H "Content-Type: application/json" \
  -d '[
    {"name": "svm01", "cluster": 1, "svm_type": "data"},
    {"name": "svm02", "cluster": 1, "svm_type": "data"},
    {"name": "svm03", "cluster": 2, "svm_type": "data"}
  ]'
```

UI bulk import supports CSV, YAML, and JSON formats.

## Validation

### IP Address Uniqueness (IPspace-Aware)

Interfaces enforce IP uniqueness within IPspace boundaries, with exceptions for operational ONTAP patterns:

| Scenario | Result |
|----------|--------|
| Same IP, different IPspace | **Allowed** (IPspace isolation) |
| Same IP, same IPspace, same cluster | **Blocked** (network conflict) |
| Same IP, same IPspace, different cluster, admin/system SVM | **Allowed** (cluster management) |
| Same IP, same IPspace, different cluster, SVM stopped | **Allowed** (DR/migration) |
| Same IP, same IPspace, different cluster, SVMDR config (down/stopped/dp_destination) | **Allowed** (SVM-DR standby) |
| Same IP, same IPspace, different cluster, both running data SVMs | **Blocked** |

### SnapMirror Volume Cardinality

Enforced in both UI and API via centralized `validate_snapmirror_volumes()`:

| Protection Type | Source Volumes | Destination Volumes |
|----------------|----------------|---------------------|
| **SVMDR** | Must be empty | Must be empty |
| **VSM** | Exactly 1 (required) | 0 or 1 (dest DP type) |
| **CG** | 1+ (required) | 0 or equal count to source |

Additional checks: cluster membership, volume type (DP required for VSM/SVMDR dest, DP/RW for CG), no source/destination overlap.

### Scope Validation (Policies & Schedules)

Cluster-scoped objects automatically resolve to admin SVM. SVM-scoped objects require data SVM:

```
scope=cluster → svm must be admin type (auto-populated if missing)
scope=svm     → svm must be data type (required)
```

### Model-Level Validation

Enforced via Django `clean()` on all models:
- SVM ↔ Cluster consistency (SVM must belong to assigned cluster)
- Aggregate ↔ Cluster consistency
- Export policy ↔ SVM consistency
- Snapshot policy scope validation (cluster-scoped from admin SVM, SVM-scoped from data SVM)
- Node ↔ Cluster tenant matching
- DP-destination subtype restricted to standalone clusters
- Volume name: no dashes (ONTAP requirement), max 203 chars
- MCC-mirrored flag only on MCC-IP/MCC-FC clusters

## Ansible Integration

The `netapp_ps.ontap_netbox` Ansible collection provides three categories of modules:

### Single-Item Modules (CUD)

One module per object type for idempotent Create, Update, Delete operations (`state: present` / `state: absent`). Support nested FK resolution by name.

| Module | API Endpoint |
|--------|-------------|
| `ontap_netbox_cluster` | `clusters/` |
| `ontap_netbox_node` | `nodes/` |
| `ontap_netbox_tier` | `tiers/` |
| `ontap_netbox_svm` | `svms/` |
| `ontap_netbox_volume` | `volumes/` |
| `ontap_netbox_qtree` | `qtrees/` |
| `ontap_netbox_quota_rule` | `quota-rules/` |
| `ontap_netbox_lun` | `luns/` |
| `ontap_netbox_interface` | `interfaces/` |
| `ontap_netbox_export_policy` | `export-policies/` |
| `ontap_netbox_export_policy_rule` | `export-policy-rules/` |
| `ontap_netbox_snapshot_policy` | `snapshot-policies/` |
| `ontap_netbox_snapmirror_policy` | `snapmirror-policies/` |
| `ontap_netbox_job_schedule` | `job-schedules/` |
| `ontap_netbox_snapmirror` | `snapmirror-relationships/` |
| `ontap_netbox_nas_share` | `shares/` |
| `ontap_netbox_nas_mount_point` | `mount-points/` |

### Bulk Modules

High-performance bulk operations for large-scale imports (10–15x faster than individual calls). Batch size configurable (default: 100).

| Module | Description |
|--------|-------------|
| `ontap_netbox_svm_bulk` | Bulk SVMs |
| `ontap_netbox_volume_bulk` | Bulk volumes |
| `ontap_netbox_qtree_bulk` | Bulk qtrees |
| `ontap_netbox_quota_rule_bulk` | Bulk quota rules |
| `ontap_netbox_lun_bulk` | Bulk LUNs |
| `ontap_netbox_interface_bulk` | Bulk interfaces |
| `ontap_netbox_export_policy_bulk` | Bulk export policies |
| `ontap_netbox_export_policy_rule_bulk` | Bulk export policy rules |
| `ontap_netbox_snapshot_policy_bulk` | Bulk snapshot policies |
| `ontap_netbox_snapmirror_policy_bulk` | Bulk SnapMirror policies |
| `ontap_netbox_job_schedule_bulk` | Bulk job schedules |

### Lookup Plugin

A single lookup plugin `ontap_lookup` queries any ONTAP object type from NetBox. The object type is specified as the first argument using dot-prefixed terms (`ontap.*` for infrastructure, `nastenant.*` for services).

Supported terms: `ontap.clusters`, `ontap.svms`, `ontap.volumes`, `ontap.qtrees`, `ontap.quota-rules`, `ontap.luns`, `ontap.interfaces`, `ontap.export-policies`, `ontap.export-policy-rules`, `ontap.snapshot-policies`, `ontap.snapmirror-policies`, `ontap.job-schedules`, `ontap.snapmirror-relationships`, `ontap.nodes`, `ontap.tiers`, `nastenant.shares`, `nastenant.mount-points`.

### CUD Module Example

```yaml
- name: Ensure volume exists in NetBox
  netapp_ps.ontap_netbox.ontap_netbox_volume:
    netbox_url: "https://{{ netbox_host }}"
    netbox_token: "{{ netbox_token }}"
    state: present
    data:
      name: "vol_data01"
      svm:
        name: "svm_prod"
        cluster:
          name: "cluster01"
      size: 107374182400
      voltype: RW
      volstate: online
      aggregate:
        name: "aggr1_node01"
      export_policy:
        name: "default"
        svm__name: "svm_prod"
        svm__cluster__name: "cluster01"
```

### Lookup Plugin Example

```yaml
- name: Get all DP volumes on svm_dr
  ansible.builtin.set_fact:
    dp_volumes: "{{ query('netapp_ps.ontap_netbox.ontap_lookup', 'ontap.volumes',
                          api_endpoint=netbox_url,
                          api_filter='svm__name=svm_dr voltype=DP',
                          token=netbox_token) }}"

- name: Show volume names
  ansible.builtin.debug:
    msg: "{{ dp_volumes | map(attribute='value') | map(attribute='name') | list }}"
```

### Sync SnapMirror Policies from ONTAP

```yaml
- name: Sync SnapMirror policy to NetBox
  netapp_ps.ontap_netbox.ontap_netbox_snapmirror_policy:
    netbox_url: "https://{{ netbox_host }}"
    netbox_token: "{{ netbox_token }}"
    state: present
    data:
      name: "{{ policy.name }}"
      cluster:
        name: "{{ cluster_name }}"
      svm:
        name: "{{ policy.svm.name }}"
      policy_type: "{{ policy.type }}"
      scope: "{{ policy.scope }}"
      ontap_details: "{{ policy }}"
```

## Tenant Services (Preview)

An optional services layer sits on top of the infrastructure models to track NAS share provisioning requests. This is specific to environments that need order-based provisioning workflows.

| Model | Purpose |
|-------|---------|
| **TenantNASShare** | Provisioning order tied to a NetBox Tenant with structured `automation_spec` |
| **TenantNASMountPoint** | Per-VM mount point tracking (path, protocol, mount_opts, NAS path) |

The services layer is independent — the infrastructure models work standalone without it.

## ONTAP Data Import & Sync

### Initial Import (Fresh Installation)

For a new NetBox deployment with no existing ONTAP data, use the **bulk import** playbook:

```bash
ansible-playbook netbox_ontap_import_bulk.yml -i inventory.yml
```

The import runs against all ONTAP clusters in the inventory and populates NetBox in dependency order:

1. **Clusters & Nodes** — registers cluster identity and node inventory
2. **Tiers/Aggregates** — physical storage tiers per node
3. **SVMs** — all data, admin, and system SVMs with enabled services
4. **Volumes** — all volumes with export policies, snapshot policies, encryption state
5. **Qtrees** — nested under their parent volumes
6. **Quota Rules** — tree/user/group quotas linked to volumes and qtrees
7. **LUNs** — SAN objects linked to volumes and optional qtrees
8. **Interfaces** — IP addresses, home ports, services, IPspace awareness
9. **Export Policies & Rules** — NFS access control configuration
10. **Snapshot Policies** — cluster-scoped and SVM-scoped policies
11. **Job Schedules** — cron and interval schedules
12. **SnapMirror Policies** — async/sync/continuous policy definitions

Each object type uses bulk API calls (batches of 100) for performance. Existing objects are matched by their unique constraint (e.g., cluster+name for SVMs) and updated if found.

Every object type is backed by a dedicated Ansible role (e.g., `bulk_import_svm`, `bulk_import_volume`, `sync_svm`, `sync_volume`). The playbooks orchestrate these roles in dependency order, but each role can also be invoked independently for targeted import or sync of a single object type.

### Ongoing Sync (Existing Installation)

For environments where NetBox already has ONTAP data, use the **sync** playbook:

```bash
ansible-playbook netbox_ontap_sync.yml -i inventory.yml
```

Sync differs from import in three key ways:

- **Orphan detection** — objects present in NetBox but no longer on ONTAP are identified. Controlled by `orphan_action`: `mark` (tags as orphaned) or `delete` (removes from NetBox).
- **Last-seen tracking** — every synced object gets a `last_seen` timestamp. Objects not seen across sync cycles are candidates for orphan handling.
- **Configuration drift detection** — each sync role compares the live ONTAP state against what's recorded in NetBox. Field-level differences (e.g., volume size changed, SVM services modified, export policy rules updated) are detected and logged. Drifted objects are updated in NetBox to reflect the current ONTAP state, ensuring NetBox remains the accurate source of truth.

Sync runs the same dependency-ordered collection from ONTAP, then compares against NetBox state per cluster. It creates new objects, updates changed ones, and flags or removes stale entries.

### Sync Reporting

When `generate_report: true` (default), each sync run produces a Markdown report in `report_dir` (`/tmp/netbox_sync_reports` by default) with per-cluster summaries:

- Objects created, updated, unchanged, and orphaned — broken down by type
- Configuration drift details — which fields changed and their before/after values
- Orphan inventory — objects marked or deleted with last-seen timestamps
- Error summary — any failed API calls or object resolution issues

Both playbooks use `nolog: true` for ONTAP credential tasks and support `--limit` to target specific clusters.

## Requirements

| Component | Version |
|-----------|---------|
| NetBox | 4.5.0+ |
| Python | 3.12+ |
| Django | 5.0+ |
| PostgreSQL | Via NetBox |

## Installation

```bash
# Editable install (development)
cd /path/to/plugin
pip install -e .

# Add to NetBox config
# PLUGINS = ['netbox_ontap_nas']

# Run migrations
cd /opt/netbox/netbox
python3 manage.py migrate netbox_ontap_nas

# Restart
sudo systemctl restart netbox netbox-rq
```

## Architecture

```
netbox_ontap_nas/
├── infrastructure/          # ONTAP object models, forms, API, views
│   ├── models.py            #   17 models with full ONTAP hierarchy
│   ├── api/serializers.py   #   Nested serializers, server-side FK resolution
│   ├── api/views.py         #   ViewSets with filtering
│   ├── filtersets.py        #   3-filter pattern per FK (id, name, __name)
│   ├── forms.py             #   Create/edit, filter, bulk import, bulk edit
│   ├── tables.py            #   List views with linkified FKs
│   └── views.py             #   UI views with object actions
│
├── services/                # Optional tenant provisioning layer
│
├── validators/
│   └── infrastructure/      # Centralized validation
│       ├── existence.py     #   FK existence + scoped lookup
│       ├── format.py        #   Name format, size format, mount path
│       ├── uniqueness.py    #   SVM+name, volume+name uniqueness
│       └── snapmirror.py    #   Cross-field volume cardinality
│
├── api/urls.py              # 17 registered API endpoints
├── form_fields.py           # Dynamic dropdown fields with custom labels
└── navigation.py            # Menu: Infrastructure, Policies, Services
```

## License

TBD
