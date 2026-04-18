# NetBox ONTAP NAS Plugin

**ONTAP Infrastructure as Code — powered by NetBox**

A NetBox plugin that models the complete ONTAP NAS stack as a structured, API-first CMDB — from clusters and nodes down to qtrees, quota rules, and export policy rules. Built for storage automation teams who need a single source of truth for ONTAP infrastructure and tenant provisioning workflows.

**Version:** 0.0.18  
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

#### `new_app` — Multi-Resource Application Storage

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

### Sync Playbook (ONTAP → NetBox)

An import/sync playbook collects live ONTAP state and pushes it into NetBox:

```
ONTAP Cluster ──(REST API)──→ Ansible ──(NetBox API)──→ Plugin
  cluster show                             POST /clusters/
  node show                                POST /nodes/
  aggr show                                POST /tiers/
  vserver show                             POST /svms/
  volume show                              POST /volumes/
  qtree show                               POST /qtrees/
  export-policy show                       POST /export-policies/
  export-policy rule show                  POST /export-policy-rules/
  network interface show                   POST /interfaces/
  snapshot policy show                     POST /snapshot-policies/
  snapmirror show                          POST /snapmirror-relationships/
```

This runs idempotently on a schedule (e.g., every 4 hours) to keep NetBox in sync with reality. Drift detection becomes trivial — compare NetBox records against live ONTAP state.

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

> **This section will be expanded as modules and playbooks are developed.**

### Planned Ansible Modules

| Module | Purpose | Status |
|--------|---------|--------|
| `netbox_ontap_cluster` | Sync ONTAP cluster info to NetBox | Planned |
| `netbox_ontap_svm` | Sync SVM inventory | Planned |
| `netbox_ontap_volume` | Sync/create volume records | Planned |
| `netbox_ontap_qtree` | Sync/create qtree records | Planned |
| `netbox_ontap_interface` | Sync LIF inventory | Planned |
| `netbox_ontap_export_policy` | Sync export policies + rules | Planned |
| `netbox_ontap_snapshot_policy` | Sync snapshot policies | Planned |
| `netbox_ontap_snapmirror` | Sync SnapMirror relationships | Planned |
| `netbox_ontap_share` | Create/update tenant share orders | Planned |
| `netbox_ontap_mount_point` | Create/update mount point records | Planned |

### Planned Playbooks

#### Sync / Import Playbooks

| Playbook | Purpose |
|----------|---------|
| `ontap_full_sync.yml` | Full cluster-to-NetBox sync (all object types) |
| `ontap_incremental_sync.yml` | Delta sync based on last-modified timestamps |
| `ontap_volume_sync.yml` | Volume-only sync with custom field population |
| `ontap_dr_sync.yml` | SnapMirror relationship and policy sync |

#### Provisioning Playbooks

| Playbook | Purpose |
|----------|---------|
| `provision_share_order.yml` | Process pending share orders (status=requested) |
| `provision_oracle_app.yml` | Oracle-specific provisioning with discovery mode |
| `provision_mount_points.yml` | Mount NFS/CIFS shares on target VMs |
| `decommission_share.yml` | Reverse provisioning with cleanup |

#### Workflow Example

```yaml
# provision_share_order.yml (conceptual)
- name: Process pending storage orders
  hosts: localhost
  vars:
    netbox_url: "https://netbox.example.com"
    netbox_token: "{{ vault_netbox_token }}"

  tasks:
    - name: Get pending orders
      uri:
        url: "{{ netbox_url }}/api/plugins/ontap-nas/shares/?provisioning_status=requested"
        headers:
          Authorization: "Token {{ netbox_token }}"
      register: pending_orders

    - name: Process each order
      include_tasks: process_order.yml
      loop: "{{ pending_orders.json.results }}"
      loop_control:
        loop_var: order

# process_order.yml
- name: Extract automation spec
  set_fact:
    spec: "{{ order.automation_spec.automation_spec }}"
    order_type: "{{ order.order_type }}"

- name: Provision storage items
  include_tasks: "provision_{{ order_type }}.yml"
  loop: "{{ spec.storage }}"
  loop_control:
    loop_var: storage_item

- name: Update order status
  uri:
    url: "{{ netbox_url }}/api/plugins/ontap-nas/shares/{{ order.id }}/"
    method: PATCH
    headers:
      Authorization: "Token {{ netbox_token }}"
    body_format: json
    body:
      provisioning_status: completed
```

---

## Appliance Roadmap

### Vision

Package this plugin as a **ready-to-deploy appliance** — a VM image (VMware OVA / KVM qcow2) with everything preinstalled. Download, deploy, point at your ONTAP clusters, run the sync playbook, done.

### Phase 1: Core Appliance

- [ ] Pre-built VM image (Ubuntu 24.04 LTS)
- [ ] NetBox 4.5.x installed and configured
- [ ] Plugin pre-installed in editable mode
- [ ] PostgreSQL + Redis configured
- [ ] nginx reverse proxy with self-signed cert
- [ ] systemd units for netbox + netbox-rq
- [ ] First-boot configuration wizard (admin password, site URL)

### Phase 2: Ansible Integration Built-In

- [ ] Ansible installed in a dedicated venv
- [ ] `netapp.ontap` collection pre-installed
- [ ] Custom `netbox_ontap_*` Ansible modules packaged
- [ ] Sync playbooks in `/opt/ontap-nas/playbooks/sync/`
- [ ] Provisioning playbooks in `/opt/ontap-nas/playbooks/provision/`
- [ ] Credential management (Ansible Vault or HashiCorp Vault integration)
- [ ] Cron job templates for scheduled sync

### Phase 3: Operational Features

- [ ] Built-in drift detection (NetBox vs live ONTAP state)
- [ ] Dashboard with cluster health, provisioning queue, orphaned mount points
- [ ] Webhook integration for order-driven provisioning (order created → Ansible runs automatically)
- [ ] Grafana dashboard templates for storage utilization
- [ ] Backup/restore scripts for the appliance

### Phase 4: Multi-Platform

- [ ] VMware OVA export with OVF properties
- [ ] KVM/libvirt qcow2 image
- [ ] Docker Compose deployment option
- [ ] Kubernetes Helm chart
- [ ] Terraform provider for NetBox plugin resources

### Appliance Architecture

```
┌─────────────────────────────────────────────────────┐
│              ONTAP NAS Appliance VM                  │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  nginx   │  │  NetBox  │  │  Ansible Engine  │  │
│  │  :443    ├──│  :8001   │  │  (sync + prov)   │  │
│  └──────────┘  └────┬─────┘  └────────┬─────────┘  │
│                     │                  │            │
│              ┌──────┴─────┐    ┌───────┴────────┐   │
│              │ PostgreSQL │    │ ONTAP Clusters  │   │
│              │ + Redis    │    │ (REST API)      │   │
│              └────────────┘    └────────────────┘   │
│                                                     │
│  Pre-installed:                                     │
│  • netbox-ontap-nas plugin                          │
│  • na_ontap Ansible collection                     │
│  • Sync & provisioning playbooks                   │
│  • Custom netbox_ontap_* Ansible modules           │
└─────────────────────────────────────────────────────┘
```

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

### 4. Create Your First Cluster

```bash
curl -k -X POST \
  -H "Authorization: Token $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "prod-cluster-01",
    "cluster_type": "mcc_ip",
    "state": "up"
  }' \
  https://netbox.example.com/api/plugins/ontap-nas/clusters/
```

### 5. Import SVM Inventory

Prepare a CSV and upload via UI at `/plugins/ontap-nas/svms/import/`, or POST via API.

---

## Technical Notes

- **NetBox change logging** — every create/update/delete on any plugin object is tracked in NetBox's change log with user attribution and timestamps. No additional audit configuration needed.
- **RBAC** — all objects inherit NetBox's permission system. You can restrict users to specific tenants, object types, or even individual fields.
- **Tags and custom fields** — all plugin objects support NetBox tags and custom fields out of the box.
- **GraphQL** — NetBox 4.5 provides GraphQL for all plugin objects automatically.
- **Webhooks** — NetBox webhooks fire on any plugin object change, enabling event-driven automation.
- **OpenAPI schema** — full OpenAPI 3.0 spec auto-generated at `/api/schema/`, compatible with code generators for any language.
