# NetBox ONTAP NAS Plugin

**ONTAP CMDB Automated — powered by NetBox**

A NetBox plugin that models the complete ONTAP NAS stack as a structured, API-first CMDB — from clusters and nodes down to qtrees, quota rules, and export policy rules. Built for storage automation teams who need a single source of truth for ONTAP infrastructure and tenant provisioning workflows.

**Version:** 0.0.18  (WIP)
**NetBox:** 4.5.x  
**Python:** 3.12+  
**License:** TBD (internal distribution)

---

## Why This Plugin Exists

If you manage ONTAP at scale, you already know the pain:

- **Inventory drift.** Cluster configs live in spreadsheets, wiki pages, and tribal knowledge. Nobody knows the actual state across 20+ clusters until someone runs a `volume show` on each.
- **No IaC foundation.** Ansible can provision volumes, but where does the desired-state definition live? YAML files in Git? Which version is deployed? What about qtree quotas, export policies, SnapMirror relationships?
- **Tenant isolation is manual.** Multi-tenant ONTAP environments need per-tenant views, per-SVM ownership, and audit trails — none of which ONTAP System Manager provides.
- **No cross-domain correlation.** A volume is useless without knowing which VM mounts it, what tenant owns it, and which export policy rules grant access. ONTAP APIs don't model these relationships.
- **Provisioning is a black box.** A tenant requests "Oracle storage with 6 data types, NFS export policies, quota rules, and mount points." Today that request travels through tickets, emails, and manual CLI sessions. There's no machine-readable order format, no status tracking, no audit trail of what was actually requested vs. what was provisioned.
- **Application storage is complex.** An Oracle database needs arch, redo, data, temp, base, and app volumes — each with different snapshot policies, retention periods, export rules, and quota limits. Assembling this from scratch for every SID is error-prone and time-consuming. There's no template system that captures the full storage layout as a single atomic document.
- **Mount point tracking doesn't exist.** After provisioning, someone manually updates a wiki with NAS paths and mount points. Nobody knows if a mount point is orphaned (VM deleted but storage remains), stale (volume decommissioned but fstab entry persists), or simply undocumented.

This plugin solves all of the above by giving ONTAP a proper CMDB layer with:

1. **Full REST API** for every ONTAP object type (17 endpoints, CRUD + bulk operations)
2. **Structured relationships** — cluster → node → SVM → volume → qtree → quota rule, all with foreign keys
3. **NetBox-native multi-tenancy** — every object belongs to a tenant, inherits RBAC, tags, custom fields
4. **Automation specs** — declarative JSON orders that Ansible playbooks consume directly. A single `automation_spec` document captures the full storage layout for an application (volumes, qtrees, quotas, export policies, mount points) as an immutable, auditable desired-state record. The Tenant Services API (`/api/plugins/ontap-nas/shares/`) accepts these specs, validates them against the infrastructure layer, and stores them for consumption by provisioning playbooks.
5. **Mount point lifecycle** — `TenantNASMountPoint` tracks every VM-to-storage mapping with NAS path, mount options, and protocol. Computed properties (`is_orphaned`, `is_active`, `fstab_entry`) give you instant visibility into mount hygiene without running `df -h` on every host.
6. **Order-driven provisioning workflow** — submit orders with `provisioning_status=requested`, let Ansible poll for pending work, provision on ONTAP, update status to `completed`, and create mount point records — all through the API. The full lifecycle is tracked with NetBox change logging.
7. **Bulk import/export** — CSV, YAML, JSON for mass onboarding of existing infrastructure

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      TENANT SERVICES LAYER                       │
│                                                                  │
│   TenantNASShare ──── automation_spec (JSON)                     │
│        │                  • order_type: new_volume/new_qtree/    │
│        │                                new_app                  │
│        │                  • storage[]: volumes, qtrees, quotas,  │
│        │                    export policies, mount points        │
│        │                  • discovery mode (volume placement)    │
│        │                                                         │
│   TenantNASMountPoint ── VM ↔ Volume/Qtree mapping              │
│        │                  • mount_path, nas_path, mount_opts     │
│        │                  • fstab_entry generation               │
│        │                  • orphan detection                     │
├──────────────────────────────────────────────────────────────────┤
│                     INFRASTRUCTURE LAYER                         │
│                                                                  │
│   ONTAPCluster ─── ONTAPNode                                     │
│        │               │                                         │
│        ├── ONTAPsvm ───┤── ONTAPVolume ── ONTAPQtree             │
│        │   (data/admin) │       │              │                 │
│        │               │       ├── ONTAPLUN    ├── ONTAPQuotaRule │
│        │               │       │               │                 │
│        ├── ONTAPTier    │       └── ONTAPExportPolicy             │
│        │   (aggr/cloud) │            └── ONTAPExportPolicyRule    │
│        │               │                                         │
│        ├── ONTAPInterface (data LIFs, with IP from NetBox IPAM)  │
│        │                                                         │
│        ├── ONTAPSnapshotPolicy                                   │
│        ├── ONTAPSnapMirror ── ONTAPSnapMirrorPolicy               │
│        └── ONTAPJobSchedule                                      │
└──────────────────────────────────────────────────────────────────┘
```

**Infrastructure layer** — 15 models that map 1:1 to ONTAP REST API resources. Every field name matches what you'd see in `ontap_rest_cli` or the ONTAP REST API docs. No abstraction, no translation layer — `volume.size` is bytes, `security_style` is `unix|ntfs|mixed`, `snaplock_type` is `non_snaplock|enterprise|compliance`.

**Tenant services layer** — 2 models that add business context. A `TenantNASShare` is a provisioning order that carries a full `automation_spec` JSON — think of it as the desired-state document that Ansible consumes. A `TenantNASMountPoint` tracks where each volume/qtree is actually mounted on VMs.

---

## Data Model Reference

### Infrastructure Models (15)

| Model | ONTAP Equivalent | Key Fields | Relationships |
|-------|-----------------|------------|---------------|
| **ONTAPCluster** | `cluster` | name, management_ip, version, cluster_type (standalone/mcc_ip/mcc_fc/cvo/anf) | → Site, Tenant, IPAddress |
| **ONTAPNode** | `cluster/nodes` | name, serial_number, model, state | → Cluster, Site, Device |
| **ONTAPTier** | `storage/aggregates` | name, tier_type (aggregate/cloud), size, state, snaplock_type, is_mcc_mirrored | → Cluster, Node |
| **ONTAPsvm** | `svm/svms` | name, svm_type, subtype (default/dp_destination/sync_source/sync_destination), admin_state, enabled_services[], ipspace | → Cluster, Site, Tenant, Tiers (M2M) |
| **ONTAPVolume** | `storage/volumes` | name, size, voltype (RW/RO/DP), volstate, encryption, snaplock_type, mount_path, space_reservation, autosize_mode, security_style, language | → Cluster, SVM, Tier (M2M), ExportPolicy, SnapshotPolicy, ParentVolume (clone) |
| **ONTAPQtree** | `storage/qtrees` | name, security_style | → Cluster, SVM, Volume, ExportPolicy, QuotaRule |
| **ONTAPLUN** | `storage/luns` | name (auto-built path), logical_unit, os_type, size | → Cluster, SVM, Volume, Qtree |
| **ONTAPInterface** | `network/ip/interfaces` | name, ip_address, netmask, admin_state, home_node, home_port_name, enabled_services[] | → Cluster, SVM, IPAddress, Node |
| **ONTAPExportPolicy** | `protocols/nfs/export-policies` | name | → Cluster, SVM |
| **ONTAPExportPolicyRule** | `export-policies/{id}/rules` | rule_index, access_proto[], clientmatch, rorule[], rwrule[], superuser[], allow_suid, allow_dev | → ExportPolicy, Cluster, SVM |
| **ONTAPQuotaRule** | `storage/quota/rules` | type (tree/user/group), qtree_name, space_hard_limit, space_soft_limit, files_hard_limit | → Cluster, SVM, Volume |
| **ONTAPSnapshotPolicy** | `storage/snapshot-policies` | name, scope (svm/cluster), enabled, ontap_details (JSON) | → Cluster, SVM |
| **ONTAPSnapMirror** | `snapmirror/relationships` | protection_type (SVMDR/VSM/CG) | → source/dest Cluster, SVM, Volumes (M2M), SnapMirrorPolicy, JobSchedule |
| **ONTAPSnapMirrorPolicy** | `snapmirror/policies` | policy_type (async/sync/continuous), sync_type, identity_preservation, ontap_details (JSON) | → Cluster, SVM, JobSchedule |
| **ONTAPJobSchedule** | `cluster/schedules` | schedule_type (cron/interval), cron_expression, interval_iso | → Cluster, SVM |

### Services Models (2)

| Model | Purpose | Key Fields |
|-------|---------|------------|
| **TenantNASShare** | Provisioning order / desired-state document | order_type (new_volume/new_qtree/new_app), app_type, protocol, availability_class, provisioning_status, automation_spec (JSON, **immutable after creation**) |
| **TenantNASMountPoint** | VM-to-storage mapping | mount_path, nas_path, mount_opts, resource_type (volume/qtree), protocol + computed properties: `fstab_entry`, `is_orphaned`, `is_active` |

### Uniqueness Constraints

Constraints follow ONTAP's actual uniqueness rules:

| Object | Unique Within | Fields |
|--------|--------------|--------|
| Cluster | Global | name |
| Node | Cluster | name |
| Tier | Cluster | name |
| SVM | Cluster | name |
| Volume | SVM | name (within same cluster+SVM) |
| Qtree | Volume | name (within same cluster+SVM+volume) |
| Interface | SVM | name |
| ExportPolicy | SVM | name |
| SnapshotPolicy | SVM | name |
| MountPoint | VM | mount_path |

---

## REST API

Every model has a full CRUD endpoint under `/api/plugins/ontap-nas/`. Standard DRF with NetBox extensions (filtering, pagination, nested serializers).

### Endpoints

| Endpoint | Methods | Bulk Support |
|----------|---------|-------------|
| `/api/plugins/ontap-nas/clusters/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/nodes/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/tiers/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/svms/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/volumes/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/qtrees/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/luns/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/interfaces/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/export-policies/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/export-policy-rules/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/quota-rules/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/snapshot-policies/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/snapmirror-relationships/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/snapmirror-policies/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/job-schedules/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/shares/` | GET, POST, PUT, PATCH, DELETE | Yes |
| `/api/plugins/ontap-nas/mount-points/` | GET, POST, PUT, PATCH, DELETE | Yes |

### Authentication

Standard NetBox API token:

```bash
curl -k -H "Authorization: Token $NETBOX_TOKEN" \
  https://netbox.example.com/api/plugins/ontap-nas/volumes/
```

### Filtering

Every FK field supports three filter styles — the same ones you'd use in NetBox core:

```bash
# By ID
GET /api/plugins/ontap-nas/volumes/?cluster_id=5

# By name
GET /api/plugins/ontap-nas/volumes/?cluster=tpc1y110cl

# By ORM traversal
GET /api/plugins/ontap-nas/volumes/?cluster__name=tpc1y110cl

# Combined
GET /api/plugins/ontap-nas/volumes/?svm=b-tcdepsh-on-01&voltype=RW&volstate=online
```

### Bulk Create

POST an array instead of a single object — NetBox handles it natively:

```bash
curl -k -X POST \
  -H "Authorization: Token $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[
    {"name": "vol_001", "cluster": 1, "svm": 5, "size": 107374182400, ...},
    {"name": "vol_002", "cluster": 1, "svm": 5, "size": 214748364800, ...}
  ]' \
  https://netbox.example.com/api/plugins/ontap-nas/volumes/
```

### Bulk Import (UI)

The plugin supports CSV, YAML, and JSON bulk import through the NetBox UI for all major object types. Upload files at `/plugins/ontap-nas/<model>/import/`.

---

## Automation Spec — The IaC Heart

The `automation_spec` field on `TenantNASShare` is a structured JSON document that fully describes a storage provisioning request. Think of it as an Ansible playbook variable file that also happens to be stored in a CMDB with full audit trail.

### Order Types

#### `new_volume` — Single Volume

```json
{
  "tenant": 42,
  "virtual_machine": 7,
  "automation_spec": {
    "order_meta": {
      "order_type": "new_volume",
      "protocol": "nfs"
    },
    "storage": [
      {
        "meta": {
          "resource_type": "volume",
          "mount_point": "/mnt/appdata"
        },
        "volume": {
          "name": "app_data_vol_01",
          "size": "500GiB",
          "svm": {"name": "prod-svm-01"}
        },
        "export_policy": {
          "name": "ep-appserver-nosuid",
          "rules": [
            {
              "clients": [{"match": "appserver.prod.example.com"}],
              "protocols": "nfs",
              "ro_rule": "sys",
              "rw_rule": "sys",
              "superuser": "sys",
              "index": 1
            }
          ]
        }
      }
    ]
  }
}
```

#### `new_qtree` — Qtree in Existing Volume

```json
{
  "tenant": 42,
  "automation_spec": {
    "order_meta": {
      "order_type": "new_qtree",
      "protocol": "nfs"
    },
    "storage": [
      {
        "meta": {
          "resource_type": "qtree",
          "mount_point": "/mnt/project_x"
        },
        "volume": {"name": "shared_vol_01", "svm": {"name": "prod-svm-01"}},
        "qtree": {"name": "project_x"},
        "quota": {"space": {"hard_limit": "100GiB"}}
      }
    ]
  }
}
```

#### `new_app` — Multi-Resource Application-Aware Storage

This is where it gets interesting. A `new_app` order defines an entire application's storage layout in one document — volumes, qtrees, quotas, export policies, mount points — everything Ansible needs to provision the complete stack.

**Example: Oracle database with 6 storage items (arch, redo, data, temp, base, app):**

```json
{
  "tenant": 1,
  "virtual_machine": 1148,
  "automation_spec": {
    "order_meta": {
      "app_type": "oracle_qtree",
      "availability_class": "enhanced",
      "order_type": "new_app",
      "protocol": "nfs"
    },
    "storage": [
      {
        "meta": {
          "resource_type": "volume",
          "retention_period": "arch_3d_22d",
          "data_type": "arch"
        },
        "volume": {
          "qtree": [
            {
              "name": "MYDB_arch_1",
              "quota": {"space": {"hard_limit": "15GiB"}},
              "export_policy": {
                "name": "ep-mydb-nosuid",
                "rules": [
                  {
                    "clients": [{"match": "db-host.prod.example.com"}],
                    "protocols": "nfs",
                    "ro_rule": "sys", "rw_rule": "sys", "superuser": "sys",
                    "allow_suid": false, "allow_device_creation": true,
                    "anonymous_user": 65534, "index": 1
                  }
                ]
              },
              "meta": {
                "mount_point": "/oracle/arch_1",
                "resource_type": "qtree"
              }
            }
          ]
        }
      }
      // ... redo, data, temp, base, app storage items
    ]
  }
}
```

### Volume Discovery Mode

For `new_app` orders, you can omit `volume.name` and instead provide `meta.retention_period` + `meta.data_type`. The provisioning engine resolves the target volume automatically based on retention policy and data classification — no need for operators to know specific volume names.

| Mode | When to use | Key fields |
|------|------------|------------|
| **Explicit** | You know the volume name | `volume.name` |
| **Discovery** | Provisioning finds the right volume | `meta.retention_period` + `meta.data_type` |

Both modes can coexist in the same order.

### Immutability

`automation_spec` is **immutable after creation**. The API rejects any attempt to modify it on an existing share. This is by design — the spec is the audit trail of what was requested. Operational fields (`provisioning_status`, `comments`, FK assignments) can be updated freely.

---

## Validation Engine

The plugin implements a layered validation architecture with shared validators — inline validation is prohibited by design.

```
validators/
├── infrastructure/          # ONTAP object validation
│   ├── messages.py          # All error message templates (single source of truth)
│   ├── uniqueness.py        # Volume/qtree/interface/policy uniqueness
│   ├── existence.py         # FK existence checks (SVM exists? Volume is RW?)
│   ├── relationships.py     # Cross-FK consistency (volume belongs to SVM?)
│   ├── format.py            # Name format, size format, mount path validation
│   ├── constants.py         # Regex patterns, length limits
│   └── snapmirror.py        # SnapMirror-specific cross-field rules
└── services/                # Tenant service validation
    ├── messages.py          # Error templates for orders
    ├── schema.py            # automation_spec JSON schema validation
    ├── business.py          # Order-type-specific business rules
    ├── format.py            # Protocol, availability class validation
    └── uniqueness.py        # Batch duplicate detection
```

Every validation rule flows through `model.clean()` → shared validator function → formatted error message. The same rule fires whether the data comes from the UI, CSV import, or REST API.

---

## IaC Value Proposition

### What This Plugin Enables for Ansible

**Without this plugin:**
```
Spreadsheet → human reads → writes Ansible vars → runs playbook → hopes it's correct
```

**With this plugin:**
```
NetBox API → Ansible reads automation_spec → provisions ONTAP → updates status in NetBox
```

The plugin is the **desired-state store**. An Ansible playbook can:

1. **Query** `GET /api/plugins/ontap-nas/shares/?provisioning_status=requested` to find pending orders
2. **Read** the full `automation_spec` — volumes, qtrees, quotas, export policies, mount points
3. **Provision** on ONTAP using `na_ontap_*` modules with values taken directly from the spec
4. **Update** `PATCH /api/plugins/ontap-nas/shares/{id}/` with `provisioning_status=completed`
5. **Create** mount point records via `POST /api/plugins/ontap-nas/mount-points/`

The spec is always stored, always versioned (NetBox change log), always queryable. No more YAML files scattered across Git repos.

### CMDB + Automation Feedback Loop

```
   ┌────────────┐      ┌──────────────────┐      ┌───────────┐
   │  Operator  │──────│  NetBox Plugin   │──────│  Ansible   │
   │  (UI/API)  │ POST │  (CMDB + Spec)   │ GET  │  Playbook  │
   └────────────┘      └──────────────────┘      └─────┬─────┘
                              ▲                        │
                              │  PATCH status          │ na_ontap_*
                              │  POST mount_points     │ modules
                              │                        ▼
                       ┌──────┴────────┐        ┌──────────────┐
                       │  NetBox API   │        │  ONTAP REST  │
                       │  (feedback)   │        │  API         │
                       └───────────────┘        └──────────────┘
```

### Sync Engine (ONTAP → NetBox)

The `netapp_ps.ontap_netbox` collection ships **sync roles** for every ONTAP object type. These aren't simple importers — they implement a full **detect → compare → apply** cycle with drift detection, orphan handling, and structured reporting.

#### Sync Flow (per object type)

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌─────────────┐
│  1. Collect   │────▶│  2. Compare      │────▶│  3. Apply        │────▶│ 4. Report   │
│  ONTAP State  │     │  ONTAP vs NetBox │     │  Create/Update/  │     │ Summary +   │
│  (REST API)   │     │  (Jinja2 delta)  │     │  Orphan (Bulk)   │     │ Markdown    │
└──────────────┘     └──────────────────┘     └──────────────────┘     └─────────────┘
```

Each sync role follows the same architecture:

1. **Collect** — Gather live state from ONTAP via `na_ontap_rest_info` / REST API
2. **Query NetBox** — Fetch current records via `ontap_lookup` plugin with `cluster__name` filter
3. **Compute delta** — Jinja2 comparison template builds four lists:
   - `to_create` — In ONTAP but not in NetBox
   - `to_update` — In both, but fields differ (drift detected)
   - `to_orphan` — In NetBox but not in ONTAP (stale records)
   - `unchanged` — No action needed
4. **Apply** — Bulk create/update via `ontap_netbox_*_bulk` modules (10–15x faster than individual calls)
5. **Report** — Structured Markdown report with counts, item lists, and changed field details

#### Sync Coverage

| ONTAP Object | Sync Role | Delta Key | Drift Fields Compared |
|---|---|---|---|
| Cluster + Nodes + Aggregates | `import_cluster` | name | Full discovery (initial import) |
| SVMs | `sync_svm` | `name` within cluster | state, enabled_services, subtype, ipspace, aggregates |
| Volumes | `sync_volume` | `svm:volume` | size, state, aggregates, security_style, junction_path, encryption, export_policy, snapshot_policy, autosize_mode, is_clone, parent_volume, is_sm_protected |
| Qtrees | `sync_qtree` | `svm:volume:qtree` | security_style, export_policy |
| LUNs | `sync_lun` | `svm:volume:lun` | size, os_type, state |
| Interfaces (LIFs) | `sync_interface` | `svm:name` | ip_address, admin_state, home_node, home_port, enabled_services |
| Export Policies | `sync_export_policy` | `svm:name` | — (existence only) |
| Export Policy Rules | `sync_export_policy_rule` | `policy:index` | clientmatch, protocols, ro/rw/superuser, allow_suid |
| Snapshot Policies | `sync_snapshot_policy` | `svm:name` | scope, enabled, ontap_details |
| Quota Rules | `sync_quota_rule` | `svm:volume:type:target` | space_hard_limit, space_soft_limit, files_hard_limit |
| Job Schedules | `sync_job_schedule` | `svm:name` | schedule_type, cron_expression, interval_iso |
| SnapMirror Policies | `sync_snapmirror_policy` | `name` | policy_type, sync_type, identity_preservation |
| SnapMirror Relationships | `sync_snapmirror` | `source:dest` | protection_type, state, policy, schedule |

#### Orphan Handling

Objects that exist in NetBox but no longer exist on ONTAP are handled via configurable `orphan_action`:

- **`mark`** (default) — Sets `ontap_sync_status=Orphaned` custom field. The record stays in NetBox for audit, but is clearly flagged.
- **`delete`** — Removes the record from NetBox entirely.

Objects with `ontap_sync_exclude=true` are never touched by sync — use this for manually-managed records or records under migration.

#### Sync Report Output

Every sync run generates a Markdown report:

```markdown
# ONTAP-NetBox Volume Sync Report

**Cluster:** tpc1y110cl
**Generated:** 2026-04-18T14:30:00+00:00
**Mode:** APPLIED
**Orphan Action:** mark

## Summary
| Metric         | Count |
|----------------|------:|
| ONTAP Volumes  |   816 |
| NetBox Volumes |   814 |
| **To Create**  |     2 |
| **To Update**  |     5 |
| **To Orphan**  |     0 |
| Unchanged      |   809 |
| Excluded       |     3 |

## Objects to Update in NetBox
| Volume               | Changed Fields                          |
|----------------------|-----------------------------------------|
| ora_data_3d_22d_1001 | size: 536870912000 → 1073741824000       |
| ora_arch_3d_22d_1004 | junction_path: /arch_old → /arch_1004    |
```

This runs idempotently on a schedule (AWX job template, cron, etc.) to keep NetBox in sync with reality. After initial import, typical sync runs complete in under 10 minutes per cluster with the bulk modules and all ONTAP objects sync enabled. Exact time depends on number of instances per object.

#### Custom Fields Managed by Sync

The `sync_common` role automatically creates these custom fields on first run:

| Custom Field | Type | Applied To | Purpose |
|---|---|---|---|
| `ontap_sync_status` | Selection | SVM, Volume, LUN, Qtree | Lifecycle state: `Active`, `Orphaned`, `Decommissioned` |
| `ontap_sync_exclude` | Boolean | SVM, Volume, LUN, Qtree | Skip this object during sync (manual override) |
| `ontap_last_seen` | Datetime | SVM, Volume, LUN, Qtree | Last ONTAP discovery timestamp (ISO 8601) |

### Custom Fields for Operational Metadata

The plugin leverages NetBox custom fields for operational tracking:

| Custom Field | Type | Applied To | Purpose |
|-------------|------|------------|---------|
| `cf_ontap_provisioning_status` | Selection | Volume | Track provisioning lifecycle (requested → provisioning → completed → failed) |
| `cf_ontap_datatype` | Selection | Volume | Data classification (ora_data, ora_arch, ora_base, linux_nfs, ...) |
| `cf_ontap_retention_period` | Selection | Volume | Snapshot retention policy mapping |

---

## Bulk Import Workflows

### Infrastructure Onboarding

For initial population of an existing ONTAP environment, the plugin supports CSV/YAML/JSON bulk import for all major object types.

**Example: 816 Oracle NFS volumes across 34 SVMs (CSV)**

```csv
name,cluster,svm,aggregate,tenant,size,voltype,volstate,encryption,...
ora_data_3d_22d_1001,tpc1y110cl,b-tcdepsh-on-01,tpc1y110_aggr_m_01,Toll Collect,536870912000,RW,online,Disabled,...
ora_arch_3d_22d_1004,tpc1y110cl,b-tcdepsh-on-01,tpc1y110_aggr_m_01,Toll Collect,536870912000,RW,online,Disabled,...
```

**Naming convention**: `ora_<datatype>_<retention_suffix>_<location><index>`

- `location`: 1 (site A), 2 (site B)
- `index`: continuous per cluster, 3-digit
- `retention_suffix`: extracted from snapshot policy (e.g., `prim_3d_22d` → `3d_22d`)

### Tenant Share Orders (Application Provisioning)

Full application storage layouts submitted via API or UI bulk import:

```bash
# Submit 3 Oracle application orders in one API call
curl -k -X POST \
  -H "Authorization: Token $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d @oracle_orders.json \
  https://netbox.example.com/api/plugins/ontap-nas/shares/
```

Each order carries the complete `automation_spec` with all volumes, qtrees, quotas, export policies, and mount points needed for one Oracle SID.

---

## Ansible Integration

The plugin's Ansible ecosystem is split across three purpose-built collections. Everything is production-ready — no "planned" placeholders.

### Collection Architecture

```
netapp_ps.ontap_netbox          NetBox CMDB layer — modules, bulk modules, sync roles,
│                               lookup plugin. Talks to NetBox API.
│
netapp_ps.ontap                 *MAF* ONTAP execution layer — 44 roles for direct cluster
│                               management (volumes, SVMs, export policies, SnapMirror,
│                               CIFS, SAN, security, QoS). Talks to ONTAP REST API.
│
netapp_ps.tollcollect           Orchestration layer — finder_svm, finder_qtree,
                                finder_vault, netbox_share_status. Bridges NetBox
                                data model with ONTAP provisioning decisions.
```

### `netapp_ps.ontap_netbox` — NetBox CMDB Modules

#### Single-Object Modules (18)

Full CRUD for every ONTAP object type in NetBox. Idempotent, with state=present/absent semantics.

| Module | Purpose |
|--------|--------|
| `ontap_netbox_cluster` | Manage ONTAP cluster records |
| `ontap_netbox_node` | Manage node records (serial, model, state) |
| `ontap_netbox_tier` | Manage aggregates and cloud tiers |
| `ontap_netbox_svm` | Manage SVMs (data/admin, services, subtype) |
| `ontap_netbox_volume` | Manage volumes (RW/RO/DP, FlexGroup, clones) |
| `ontap_netbox_qtree` | Manage qtrees |
| `ontap_netbox_lun` | Manage LUNs (SAN block storage) |
| `ontap_netbox_interface` | Manage LIFs (data/mgmt, IPv4/IPv6, services) |
| `ontap_netbox_export_policy` | Manage NFS export policies |
| `ontap_netbox_export_policy_rule` | Manage individual export policy rules |
| `ontap_netbox_quota_rule` | Manage quota rules (user/group/tree) |
| `ontap_netbox_qos_policy_group` | Manage QoS policy groups |
| `ontap_netbox_snapshot_policy` | Manage snapshot policies |
| `ontap_netbox_job_schedule` | Manage job schedules (cron/interval) |
| `ontap_netbox_snapmirror` | Manage SnapMirror relationships (VSM/SVMDR/CG) |
| `ontap_netbox_snapmirror_policy` | Manage SnapMirror policies |
| `ontap_netbox_nas_share` | Manage tenant share orders (automation_spec) |
| `ontap_netbox_nas_mount_point` | Manage VM-to-storage mount point records |

**Example — register a volume in NetBox:**

```yaml
- name: Register volume in NetBox
  netapp_ps.ontap_netbox.ontap_netbox_volume:
    netbox_url: "{{ netbox_url }}"
    netbox_token: "{{ netbox_token }}"
    state: present
    data:
      name: ora_data_3d_22d_1001
      cluster: { name: prod-cluster-01 }
      svm: { name: prod-svm-01, cluster: { name: prod-cluster-01 } }
      size: "500GiB"
      voltype: RW
      volstate: online
      security_style: unix
      aggregates: [aggr_ssd_01]
      export_policy: { name: default }
```

#### Bulk Modules (12) — 10–15x Faster

For large-scale import and sync operations. Process hundreds of objects per API call with batched transactions, partial success handling, and detailed error reporting.

| Module | Batch Default |
|--------|------:|
| `ontap_netbox_svm_bulk` | 100 |
| `ontap_netbox_volume_bulk` | 100 |
| `ontap_netbox_qtree_bulk` | 100 |
| `ontap_netbox_lun_bulk` | 100 |
| `ontap_netbox_interface_bulk` | 100 |
| `ontap_netbox_export_policy_bulk` | 100 |
| `ontap_netbox_export_policy_rule_bulk` | 100 |
| `ontap_netbox_quota_rule_bulk` | 100 |
| `ontap_netbox_snapshot_policy_bulk` | 100 |
| `ontap_netbox_snapmirror_bulk` | 100 |
| `ontap_netbox_snapmirror_policy_bulk` | 100 |
| `ontap_netbox_job_schedule_bulk` | 100 |

**Bulk module return structure:**

```yaml
created: [{id: 101, name: "vol_001"}, ...]    # Successfully created
updated: [{id: 101, name: "vol_001"}, ...]    # Successfully updated
deleted: ["101", "102"]                        # Deleted IDs
errors:  [{index: 2, object: {...}, error: "SVM not found"}]
batches_processed: 5
total_objects: 498
success_rate: 99.6
msg: "Processed 498/500 volumes in 5 batches"
```

Key features:
- **Partial success** — one failed object doesn't block the rest (`fail_on_error=false`)
- **Smart size parsing** — accepts `"100GB"`, `"1TiB"`, `"500GiB"` or raw bytes
- **Nested FK resolution** — `svm: {name: "svm_prod", cluster: {name: "cluster1"}}` resolves automatically
- **FlexGroup support** — `aggregates` list for M2M tier mapping

#### Lookup Plugin — `ontap_lookup`

Query any NetBox ONTAP object from playbooks, roles, or Jinja2 templates. Supports 15+ object types with aliases, Django-style filters, and multiple output formats.

```yaml
# Query SVMs by purpose and state
- set_fact:
    target_svms: "{{ query('netapp_ps.ontap_netbox.ontap_lookup', 'ontap.svms',
                          api_endpoint=netbox_url, token=netbox_token,
                          validate_certs=false,
                          api_filter='cf_ontap_svm_purpose=oracle_nfs admin_state=running') }}"

# Query volumes for a specific SVM
- set_fact:
    svm_volumes: "{{ query('netapp_ps.ontap_netbox.ontap_lookup', 'ontap.volumes',
                          api_endpoint=netbox_url, token=netbox_token,
                          api_filter='cluster__name=prod-cluster-01 svm=prod-svm-01') }}"

# Query pending share orders
- set_fact:
    pending: "{{ query('netapp_ps.ontap_netbox.ontap_lookup', 'nastenant.shares',
                       api_endpoint=netbox_url, token=netbox_token,
                       api_filter='provisioning_status=requested') }}"
```

**Supported terms** (with underscore/hyphen/singular/plural aliases):

| Category | Terms |
|---|---|
| Infrastructure | `ontap.clusters`, `ontap.nodes`, `ontap.tiers`, `ontap.svms`, `ontap.volumes`, `ontap.qtrees`, `ontap.luns`, `ontap.interfaces` |
| Policies | `ontap.export-policies`, `ontap.export-policy-rules`, `ontap.snapshot-policies`, `ontap.quota-rules`, `ontap.job-schedules` |
| Replication | `ontap.snapmirror-relationships`, `ontap.snapmirror-policies` |
| Tenant Services | `nastenant.shares`, `nastenant.mount-points` |

**Filter syntax:** space-separated `key=value` pairs. Same key repeated = OR logic; different keys = AND logic.

**Return format** (default):
```yaml
[{key: <object_id>, value: {id: 42, name: "svm_prod", cluster: {...}, ...}}, ...]
```

Set `raw_data=true` to get flat API response objects instead.

#### Sync & Import Roles (37 roles)

Three tiers of data ingestion, each for a different use case:

| Tier | Roles | Use Case | Performance |
|---|---|---|---|
| **Import** (13 roles) | `import_cluster`, `import_svm`, `import_volume`, `import_qtree`, `import_lun`, `import_interface`, `import_export_policy`, `import_export_policy_rule`, `import_snapshot_policy`, `import_quota_rule`, `import_snapmirror`, `import_snapmirror_policy`, `import_job_schedule` | Initial population, one-time imports | Single-object modules |
| **Bulk Import** (12 roles) | `bulk_import_svm` through `bulk_import_snapmirror` | High-volume initial load with drift detection | Bulk modules, 10–15x faster |
| **Sync** (11 roles) | `sync_svm` through `sync_snapmirror` | Continuous scheduled sync with full delta detection, orphan handling, and reporting | Bulk modules + Jinja2 comparison |
| **Support** | `sync_common`, `error_collector`, `facts` | Custom field setup, report generation, error aggregation | — |

See the [Sync Engine](#sync-engine-ontap--netbox) section above for the detailed sync flow.

---

### `netapp_ps.ontap` — ONTAP Execution Roles (44 roles) - MAF based

Direct ONTAP cluster management via REST API. These roles are what actually provisions, modifies, and configures ONTAP resources.

#### Core Roles (MAF)

| Category | Roles |
|---|---|
| **Cluster & Network** | `cluster`, `broadcast_domain`, `interface`, `subnet`, `dns`, `vlan` |
| **SVM & Storage** | `svm`, `volume`, `volume_autosize`, `volume_clone`, `volume_efficiency`, `qtree`, `quota`, `quota_policy` |
| **NFS** | `nfs`, `export_policy`, `export_policy_rule`, `name_mapping` |
| **CIFS/SMB** | `cifs`, `cifs_share`, `cifs_acl`, `cifs_local_user`, `cifs_local_group`, `cifs_privilege` |
| **SAN** | `lun`, `lun_map`, `igroup`, `iscsi` |
| **Snapshots & Replication** | `snapshot`, `snapshot_policy`, `snapmirror`, `snapmirror_policy` |
| **Security** | `security_certificate`, `user`, `unix_user`, `unix_group`, `vserver_peer`, `cluster_peer` |
| **File & Audit** | `file_security_permissions`, `file_security_permissions_acl`, `vserver_audit` |
| **Advanced** | `qos_policy_group`, `fpolicy_policy`, `fpolicy_event`, `fpolicy_scope`, `fpolicy_ext_engine`, `fpolicy_status`, `software_update`, `vscan_scanner_pool`, `facts` |

---

### `netapp_ps.tollcollect` — Orchestration Roles

These roles bridge the gap between the NetBox CMDB and ONTAP execution. They consume `automation_spec` data from share orders and make intelligent placement decisions.

#### `finder_svm` — Target SVM Discovery

Given a VM and a share order, automatically discovers the correct target SVM by correlating VM network location with ONTAP infrastructure in NetBox:

```
VM (NetBox)                        ONTAP (NetBox)
┌─────────────┐                    ┌──────────────────┐
│ Interfaces  │──── IPv6 prefix ──▶│ SVM LIFs         │
│ Site/DC     │──── location ─────▶│ SVM site         │
│ Tenant      │──── ownership ────▶│ SVM tenant       │
└─────────────┘                    │ SVM purpose      │
                                   │ SVM protocols    │
                                   │ MCC-mirrored aggr│
                                   └──────────────────┘
```

**Discovery flow:**
1. Resolve VM interfaces and IPv6 address from NetBox
2. Determine VM's network prefix and primary datacenter
3. Query SVMs matching: `svm_purpose` + `tenant` + `site` + `protocol` + `mirrored_aggr` requirement
4. Cross-reference SVM LIF IPs against VM prefix — only SVMs reachable from VM's network qualify
5. Assert exactly one unique SVM found (fail-fast on ambiguity)
6. Return SVM name, cluster management IP, and volume/aggregate placement details

**Supports order-type routing** — different validation and placement logic for `new_volume`, `new_app/default`, and `new_app/oracle_qtree` flows.

**Volume discovery mode** — for `new_app` orders without explicit volume names, queries NetBox volumes by `cf_ontap_data_type` + `cf_ontap_retention_period` + `cf_ontap_provisioning_status=provisioned` and selects the volume with the lowest `qtree_count`.

**Return value:**

```yaml
finder_svm_result:
  result: "success"          # or "failed" with diagnostic msg
  msg: "SVM found successfully"
  new_app:                   # key matches order_type
    svm:
      name: "prod-svm-01"
    cluster:
      name: "prod-cluster-01"
      management_ip: "10.0.0.1"
    volume:                  # list of volume placements
      - name: "ora_data_3d_22d_1001"
        aggregate_name: "aggr_ssd_01"
        lif_ip: "10.0.1.100"
        junction_path: "/ora_data_3d_22d_1001"
```

#### `finder_qtree` — Qtree Target Resolution

Validates and populates qtree data for `new_qtree` orders. Ensures the target volume exists, has capacity, and the qtree name is available.

#### `finder_vault` — SnapMirror Destination Discovery

For protected volumes (`is_protected: true`), discovers the correct vault/destination SVM and generates secondary volume specifications. Validates SVM peering and SnapMirror policy availability.

#### `netbox_share_status` — Provisioning Lifecycle

Updates share order status in NetBox based on ONTAP provisioning results. Validates provisioned resources against the original `automation_spec`, generates mount point specifications from templates, and transitions the order through: `requested → provisioning → completed` (or `failed` with diagnostics).

---

### End-to-End Provisioning Flow

How the three collections work together for a `new_app/oracle_qtree` order:

```
                     NetBox                    AWX/Ansible                    ONTAP
                       │                           │                           │
  Operator submits ───▶│ POST /shares/             │                           │
  automation_spec      │ (validation + store)      │                           │
                       │                           │                           │
  AWX polls ──────────▶│ GET /shares/?status=req   │                           │
                       │◀──────────────────────────│                           │
                       │                           │                           │
                       │  ontap_lookup queries ────▶│ finder_svm               │
                       │  (VM, SVMs, LIFs,         │ (discover target SVM,    │
                       │   volumes, prefixes)       │  resolve volumes)         │
                       │                           │                           │
                       │                           │ netapp_ps.ontap roles ───▶│
                       │                           │ • export_policy           │ create EP
                       │                           │ • export_policy_rule      │ create rules
                       │                           │ • qtree                   │ create qtrees
                       │                           │ • quota                   │ set quotas
                       │                           │ • snapmirror (if protect) │ create SM
                       │                           │                           │
                       │◀──── ontap_netbox_* ──────│ Register in NetBox:       │
                       │  • ontap_netbox_qtree      │ qtrees, export policies, │
                       │  • ontap_netbox_quota_rule │ quota rules, mount points│
                       │  • ontap_netbox_export_*   │                           │
                       │  • ontap_netbox_nas_mount  │                           │
                       │                           │                           │
                       │◀── PATCH /shares/{id}/ ───│ netbox_share_status       │
                       │  provisioning_status:      │ (validate + finalize)     │
                       │  completed                 │                           │
```

---

## Provisioning Architecture

The provisioning engine lives in the `automation_core` repository and orchestrates three collections to turn NetBox `automation_spec` orders into running ONTAP resources. It follows the **MAF pattern** (Mirko Ansible Framework): each ONTAP object type has a dedicated `netapp_ps.ontap` role for create/update/delete, and a matching `netapp_ps.ontap_netbox` module to register the result in NetBox.

### AWX Entry Points

The engine exposes three AWX job template endpoints:

| Endpoint Playbook | Trigger | Purpose |
|---|---|---|
| `netbox_share_endpoint.yaml` | Share order (`new_app`, `new_volume`, `new_qtree`) | Application-aware provisioning with SVM discovery, volume placement, and full storage stack creation |
| `netbox_generic_endpoint.yml` | Individual NetBox object (`ontapsvm`, `ontapvolume`, `ontapsnapmirror`, `ontapquotarule`) | Single-object provisioning — create one SVM, volume, or SnapMirror relationship on demand |
| `trunk/netbox_provision.yml` | Legacy NetBox→MAF transformation | Direct volume/SVM creation with size conversion and template matching |

### Current Provisioning Capabilities

| Operation | ONTAP Role (MAF) | NetBox Registration | Status |
|---|---|---|---|
| SVM create (full stack: NFS/CIFS/iSCSI services, DNS, root volume) | `svm` | `ontap_netbox_svm` | Done |
| Volume create (size conversion, aggregate, junction path, security style) | `volume` | `ontap_netbox_volume` | Done |
| Qtree create (security style, unix permissions) | `qtree` | `ontap_netbox_qtree` | Done |
| Export Policy create | `export_policy` | `ontap_netbox_export_policy` | Done |
| Export Policy Rule create (clients, ro/rw/su, SUID, protocols) | `export_policy_rule` | `ontap_netbox_export_policy_rule` | Done |
| Quota Rule create/update (tree quotas, space/file limits) + quota enable | `quota` | `ontap_netbox_quota_rule` | Done |
| SnapMirror create (async/sync, via `finder_vault` discovery) | `snapmirror` | `ontap_netbox_snapmirror` | Done |
| Mount Point registration | — | `ontap_netbox_nas_mount_point` | Done |
| Share status lifecycle (`requested` → `provisioning` → `completed`/`failed`) | — | `ontap_netbox_nas_share` | Done |

### Order Processing Flow

The provisioning engine is event-driven via AWX/AAP:

1. **Order ingestion** — Operator or upstream system submits a `TenantNASShare` with `automation_spec` via NetBox API
2. **Order pickup** — AWX job template polls for `provisioning_status=requested` orders
3. **SVM discovery** — `finder_svm` role resolves the target SVM based on VM location, tenant, protocol, and network reachability
4. **Volume resolution** — Explicit volume names are validated against NetBox; discovery-mode volumes are resolved by `(data_type, retention_period)` matching
5. **Payload transformation** — `share_to_maf` filter converts the `automation_spec` into MAF `vars_local` format (size unit conversion GiB→MiB, cluster/SVM context injection, template selection)
6. **ONTAP execution loop** — For each storage item, `netapp_ps.ontap` roles execute in dependency order:
   - Export policy + rules → Volume (if new) → Qtree → Quota → Enable quotas on volume → SnapMirror (if `is_protected`)
7. **NetBox registration** — `netapp_ps.ontap_netbox` modules register each created object back into the CMDB
8. **Finalization** — `netbox_share_status` role validates the provisioned state against the spec, creates mount point records, and transitions the order to `completed`

All provisioning tasks use block/rescue patterns — on failure, the share status is set to `failed` with error details written to the NetBox comments field.

### Supported Order Types

| Order Type | Entry Task | What Gets Created |
|---|---|---|
| `new_app` (default) | `provision_new_app.yml` | Volumes + export policies + rules. SVM discovery via `finder_svm`. Supports explicit and discovery-mode volume placement. |
| `new_app` (oracle_qtree) | `provision_new_app.yml` | Discovers existing volumes by `(data_type, retention_period)`, creates qtrees + quotas + export policies on each. Full Oracle SID provisioning (arch, redo, data, temp, base, app). |
| `new_volume` | `provision_new_volume.yml` | Single or multiple volumes with aggregate placement. SVM discovery or explicit. |
| `new_qtree` | `provision_new_qtree.yml` | Qtrees in existing volumes. Uses `finder_qtree` for target resolution. Creates quotas and export policies per qtree. |
| Generic SVM | `provision_generic_ontapsvm.yml` | Full SVM with protocol stack (NFS/CIFS/iSCSI/FCP), DNS, root volume, aggregate list, ipspace. |
| Generic SnapMirror | `provision_generic_ontapsnapmirror.yml` | SnapMirror relationship via `finder_vault` (destination SVM/cluster discovery, peering validation). |

The design goal is full coverage: for every ONTAP object type modeled in the plugin, there should be a matching provisioning playbook that can create, update, or delete it — and register the outcome in NetBox.

---

## Roadmap

### Data Model Enhancements

- [ ] **`ontap_details` field on all infrastructure models** — A JSON field storing the raw ONTAP REST API response for each object. This enables full IaC replay: given a NetBox export, you can reconstruct any ONTAP object exactly as it exists on the cluster. Currently available on `ONTAPSnapshotPolicy` and `ONTAPSnapMirrorPolicy`; planned for all 15 infrastructure models.
- [ ] **ONTAP S3 Bucket model** — `ONTAPBucket` to track S3 buckets per SVM (name, size, versioning, policy, lifecycle rules)
- [ ] **Vserver Peering model** — `ONTAPVserverPeer` to track SVM peering relationships (source SVM, destination SVM, peer cluster, applications, state)

### Provisioning Enhancements

- [ ] **Volume delete/decommission** — Reverse provisioning lifecycle: delete qtrees, quotas, export policies, SnapMirror relationships; update NetBox records; transition share to `decommissioned`
- [ ] **LIF provisioning** — Network interface creation as part of SVM provisioning
- [ ] **Execution log persistence to Git** — Provisioning runs save structured execution logs (order ID, timestamps, tasks executed, ONTAP responses, success/failure) to a Git repository for full audit trail and replay capability
- [ ] **`new_qtree` order type in share endpoint** — End-to-end qtree provisioning through the share order workflow (currently available via generic endpoint)

### Appliance

- [ ] Pre-built VM image (Ubuntu 24.04 LTS) with all components pre-installed
- [ ] **AWX** deployed in the appliance for playbook execution, job scheduling, webhook-driven provisioning, and credential management
- [ ] **AnsibleForms** deployed in the appliance as the operator-facing UI for maintenance operations, ad-hoc provisioning, and parameter-driven form submissions
- [ ] First-boot configuration wizard (admin password, site URL, ONTAP credentials)
- [ ] Cron job templates for scheduled sync
- [ ] VMware OVA / KVM qcow2 / Docker Compose deployment options

---

## Quick Start

### 1. Install the Plugin

```bash
cd /opt/netbox
source venv/bin/activate
cd myplugins && pip install -e .
```

Add to NetBox configuration (`/opt/netbox/netbox/netbox/configuration.py`):

```python
PLUGINS = ['netbox_ontap_nas']
```

### 2. Run Migrations

```bash
cd /opt/netbox/netbox
sudo /opt/netbox/venv/bin/python3 manage.py migrate netbox_ontap_nas
```

### 3. Restart NetBox

```bash
sudo systemctl restart netbox netbox-rq
```

### 4. Install the Ansible Collections

```bash
# Install all three collections
ansible-galaxy collection install netapp_ps.ontap_netbox
ansible-galaxy collection install netapp_ps.ontap
ansible-galaxy collection install netapp_ps.tollcollect

# Also need the official NetApp collection for ONTAP REST API access
ansible-galaxy collection install netapp.ontap
```

### 5. Import Your First Cluster

Register the cluster and discover nodes + aggregates in one step:

```yaml
# import_cluster.yml
- name: Import ONTAP cluster into NetBox
  hosts: localhost
  gather_facts: false
  vars:
    netbox:
      netbox_url: "https://netbox.example.com"
      netbox_token_secret: "{{ vault_netbox_token }}"
    ontap:
      hostname: "10.0.0.1"          # Cluster management IP
      username: admin
      password: "{{ vault_ontap_password }}"

  tasks:
    - name: Import cluster, nodes, and aggregates
      ansible.builtin.include_role:
        name: netapp_ps.ontap_netbox.import_cluster
```

This discovers the cluster name, all nodes (serial numbers, models), and all aggregates (sizes, types, MCC mirror status) — and registers everything in NetBox.

### 6. Sync SVMs and Volumes

```yaml
# sync_cluster.yml
- name: Sync ONTAP to NetBox
  hosts: localhost
  gather_facts: false
  vars:
    _cluster_name: "prod-cluster-01"
    dry_run: false                    # Set true to preview changes
    generate_report: true             # Generate Markdown sync report
    orphan_action: mark               # 'mark' or 'delete'
    netbox:
      netbox_url: "https://netbox.example.com"
      netbox_token_secret: "{{ vault_netbox_token }}"
    ontap:
      hostname: "10.0.0.1"
      username: admin
      password: "{{ vault_ontap_password }}"

  tasks:
    - name: Sync SVMs
      ansible.builtin.include_role:
        name: netapp_ps.ontap_netbox.sync_svm

    - name: Sync export policies
      ansible.builtin.include_role:
        name: netapp_ps.ontap_netbox.sync_export_policy

    - name: Sync export policy rules
      ansible.builtin.include_role:
        name: netapp_ps.ontap_netbox.sync_export_policy_rule

    - name: Sync volumes
      ansible.builtin.include_role:
        name: netapp_ps.ontap_netbox.sync_volume

    - name: Sync interfaces
      ansible.builtin.include_role:
        name: netapp_ps.ontap_netbox.sync_interface

    - name: Sync SnapMirror relationships
      ansible.builtin.include_role:
        name: netapp_ps.ontap_netbox.sync_snapmirror
```

Run with `dry_run: true` first to see the delta report without making changes. When satisfied, set `dry_run: false` and let the sync apply.

### 7. Query NetBox from Playbooks

```yaml
# Use the lookup plugin to query ONTAP objects
- name: Find all running SVMs on a cluster
  ansible.builtin.debug:
    msg: "SVM: {{ item.value.name }} — services: {{ item.value.enabled_services }}"
  loop: "{{ query('netapp_ps.ontap_netbox.ontap_lookup', 'ontap.svms',
                  api_endpoint=netbox.netbox_url,
                  token=netbox.netbox_token_secret,
                  validate_certs=false,
                  api_filter='cluster__name=prod-cluster-01 admin_state=running') }}"
```

### 8. Schedule Continuous Sync

Create an AWX/AAP job template with the sync playbook and set a recurring schedule (e.g., every 4 hours). Each run:
- Detects new objects (creates in NetBox)
- Detects changed fields (updates in NetBox with `changed_fields` in report)
- Detects removed objects (marks as orphaned)
- Generates a Markdown report with full diff summary

---

## Technical Notes

- **NetBox change logging** — every create/update/delete on any plugin object is tracked in NetBox's change log with user attribution and timestamps. No additional audit configuration needed.
- **RBAC** — all objects inherit NetBox's permission system. You can restrict users to specific tenants, object types, or even individual fields.
- **Tags and custom fields** — all plugin objects support NetBox tags and custom fields out of the box.
- **GraphQL** — NetBox 4.5 provides GraphQL for all plugin objects automatically.
- **Webhooks** — NetBox webhooks fire on any plugin object change, enabling event-driven automation.
- **OpenAPI schema** — full OpenAPI 3.0 spec auto-generated at `/api/schema/`, compatible with code generators for any language.
