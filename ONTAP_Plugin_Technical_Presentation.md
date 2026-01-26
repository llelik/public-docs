# NetBox ONTAP NAS Plugin - Technical Presentation

## Target Audience
- NetApp Professional Services Engineers
- ONTAP Sales/Pre-Sales Engineers
- Customer Technical Teams

---

## Slide 1: Title

**NetBox ONTAP NAS Plugin**
*Infrastructure Management & Tenant Self-Service for NetApp ONTAP*

- Open Source Foundation + Professional Services Value-Add
- Version 0.0.10 | NetBox 4.4.7+

---

## Slide 2: What is NetBox?

**Network Source of Truth (NSoT)**

NetBox is an open-source infrastructure resource modeling (IRM) application designed to empower network automation.

**Core Concepts:**
- **Regions** → Sites → Locations → Racks → Devices
- **Tenants** → Business units, customers, or project owners
- **IPAM** → IP addresses, prefixes, VLANs, VRFs
- **DCIM** → Racks, devices, cables, power

**Plugin Architecture:**
- Extends NetBox with custom models, views, and APIs
- Full REST API for every object type
- Integrates with NetBox's tenant model and RBAC

**Why NetBox for ONTAP?**
- Industry-standard infrastructure documentation
- Built-in multi-tenancy support
- REST API enables automation workflows
- Extensible via plugins

**NetApp Device Library:**
- NetApp provides ONTAP device type definitions (device types, module types)
- Enables modeling of ONTAP controller ports and connections in NetBox DCIM
- Delivered as part of per-customer solution
- Integrates physical infrastructure with logical ONTAP objects

---

## Slide 3: Plugin Architecture Overview

**Two-Layer Design**

```
┌─────────────────────────────────────────────────────────┐
│                    TENANT SERVICES                       │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Tenant NAS Shares (Orders)                     │    │
│  │  • Order Types: new_volume, new_qtree, etc.     │    │
│  │  • Self-service request workflow                │    │
│  │  • automation_spec (IaC-ready JSON)             │    │
│  └─────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────┤
│                  INFRASTRUCTURE LAYER                    │
│  ┌─────────────────────────────────────────────────┐    │
│  │  ONTAP Objects (Open Source)                    │    │
│  │  • Clusters, Nodes, SVMs, Volumes, Qtrees       │    │
│  │  • Tiers, LUNs, Interfaces, Export Policies     │    │
│  │  • SnapMirror Relationships, Snapshot Policies  │    │
│  │  • S3 Buckets (Planned)                         │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Infrastructure Layer:** Open source - models ONTAP objects in NetBox
**Tenant Services Layer:** PS value-add - business logic, workflows, automation

---

## Slide 4: Infrastructure Layer - ONTAP Object Hierarchy

**Complete ONTAP Modeling**

```
ONTAPCluster
├── ONTAPNode (HA pairs, node details)
├── ONTAPTier (aggregates, storage tiers)
├── ONTAPsvm (Storage Virtual Machines)
│   ├── ONTAPInterface (LIFs - data, mgmt)
│   ├── ONTAPVolume
│   │   ├── ONTAPQtree
│   │   └── ONTAPLun
│   ├── ONTAPExportPolicy
│   │   └── ONTAPExportPolicyRule
│   └── ONTAPSnapmirror (DP relationships)
└── ONTAPSnapshotPolicy (cluster-level policies)
```

**Planned Additions:**
- ONTAPBucket (S3 object storage)
- S3 user/policy management

**Key Features:**
- Automatic import from live ONTAP clusters
- Cross-cluster SnapMirror relationship tracking
- Capacity tracking and reporting
- Full REST API for all objects
- Bulk import/export (CSV, JSON, YAML)

---

## Slide 5: Tenant Services Layer

**NAS Share Request Workflow**

**Order Types:**
| Order Type | Creates | Use Case |
|------------|---------|----------|
| `new_volume` | Volume only | Dedicated storage |
| `new_qtree` | Qtree in existing volume | Shared volume, quota control |
| `new_volume_with_qtree` | Both | Best of both |

**Request Entry Points:**
1. NetBox Web UI (direct form entry)
2. REST API (JSON payload)
3. ServiceNow Integration (ITSM workflow)
4. Custom Portals (tenant self-service)

**Planned Additions:**
- S3 bucket provisioning orders
- Object storage quota management

**automation_spec Field:**
- Complete provisioning specification stored as JSON
- Includes: volume, qtree, quota, export_policy definitions
- **IaC Ready:** Can be exported and used by DevOps teams for:
  - Infrastructure reproduction
  - Version-controlled storage definitions
  - Audit trails and compliance documentation
  - Terraform/Ansible variable sources

---

## Slide 6: automation_spec Structure

**Example: new_volume Order**

```json
{
  "order_meta": {
    "order_type": "new_volume",
    "mount_point": "/mnt/project_data",
    "tenant_name": "Engineering"
  },
  "volume": {
    "svm": {"name": "svm_prod_01", "cluster": {"name": "cluster_nyc"}},
    "name": "vol_eng_data_01",
    "size": 1099511627776,
    "tiering_policy": "auto",
    "snapshot_policy": {"name": "default"}
  },
  "export_policy": {
    "name": "eng_data_export",
    "rules": [
      {
        "ro_rule": ["sys"],
        "rw_rule": ["sys"],
        "clients": [{"match": "10.0.0.0/8"}]
      }
    ]
  }
}
```

**DevOps Value:**
- Stored in database for audit and reproduction
- JSON format compatible with Ansible, Terraform, Python scripts
- Complete specification eliminates manual documentation

---

## Slide 7: Integration Architecture

**End-to-End Automation Flow**

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  ServiceNow  │     │  Custom      │     │  NetBox      │
│  (ITSM)      │     │  Portal      │     │  Web UI      │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │   NetBox REST API     │
                │   (Plugin Endpoints)  │
                └───────────┬───────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │   AWX / Ansible       │
                │   Tower               │
                │   ─────────────────   │
                │   MAF / AMF           │
                │   (NetApp PS Ansible  │
                │    Frameworks)        │
                └───────────┬───────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │   ONTAP Clusters      │
                │   (REST API)          │
                └───────────────────────┘
```

**AWX/Ansible Integration:**
- AWX monitors NetBox for new orders (status: pending)
- Retrieves automation_spec via REST API
- Executes playbooks to provision resources
- Updates NetBox with provisioning status

**MAF / AMF Frameworks:**
- NetApp Professional Services proprietary Ansible collections
- Pre-built roles and modules for ONTAP automation
- Handles complex provisioning logic (SnapMirror, tiering, QoS)
- **Note:** Playbooks are NOT part of the plugin solution
- Playbooks are part of customer project integration efforts

---

## Slide 8: Plugin vs. Integration Boundary

**What's in the Plugin (Open Source + PS Enhancements):**
- ONTAP object models (Infrastructure layer)
- Tenant NAS Share models (Tenant Services layer)
- REST API endpoints
- Bulk import/export
- Validation logic
- ONTAP cluster import service

**What's Customer Project Integration:**
- AWX/Ansible Tower deployment
- Playbook development (using MAF/AMF)
- ServiceNow workflow configuration
- Custom portal development
- RBAC and tenant configuration
- Network and firewall configuration

**Clear Separation:**
- Plugin = Reusable product
- Integration = Customer-specific implementation

---

## Slide 9: Delivery Model

**Support Contract:**
- Plugin license with ongoing support
- Bug fixes and feature updates
- Integration guidance

**Time & Materials (T&M):**
- Custom playbook development
- ServiceNow/portal integration
- Customer-specific workflows
- Training and knowledge transfer

**What Customers Get:**
- Standardized NAS provisioning workflow
- Self-service capabilities for tenant teams
- Audit trail and compliance documentation
- IaC-ready automation specifications
- Reduced provisioning time (hours → minutes)
- ONTAP clusters network awareness in NetBox (ports, cables, IPs)
- Network teams visibility into storage NAS connections

**For Existing NetBox Customers:**
- Smooth integration with existing network source of truth
- ONTAP plugin extends current infrastructure documentation
- No disruption to existing workflows

**For New NetBox Customers:**
- Production-grade Network and VirtualMachine management included
- Complete infrastructure documentation platform
- Industry-standard IRM solution with ONTAP integration

---

## Slide 10: Roadmap - Turnkey Appliance

**Lowering Entry Barrier**

Planned VMware/KVM appliance including:

```
┌─────────────────────────────────────────────┐
│           NetBox ONTAP Appliance            │
├─────────────────────────────────────────────┤
│  ┌─────────────────────────────────────┐    │
│  │  NetBox + ONTAP NAS Plugin          │    │
│  └─────────────────────────────────────┘    │
│  ┌─────────────────────────────────────┐    │
│  │  ONTAP Cluster Import Service       │    │
│  │  (Automatic discovery & sync)       │    │
│  └─────────────────────────────────────┘    │
│  ┌─────────────────────────────────────┐    │
│  │  AnsibleForms                       │    │
│  │  (Simple GUI for NAS requests)      │    │
│  └─────────────────────────────────────┘    │
│  ┌─────────────────────────────────────┐    │
│  │  Pre-configured AWX (optional)      │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

**Benefits:**
- Deploy in minutes, not days
- No NetBox expertise required for basic usage
- AnsibleForms provides simplified request interface
- Ideal for proof-of-concept and smaller deployments
- Upgrade path to full enterprise integration

---

## Slide 11: Feature Summary

| Feature | Status | Layer |
|---------|--------|-------|
| ONTAP Clusters, Nodes | ✅ Complete | Infrastructure |
| SVMs, Volumes, Qtrees | ✅ Complete | Infrastructure |
| LUNs, Interfaces | ✅ Complete | Infrastructure |
| Export Policies & Rules | ✅ Complete | Infrastructure |
| SnapMirror Relationships | ✅ Complete | Infrastructure |
| Snapshot Policies | ✅ Complete | Infrastructure |
| S3 Buckets | 🔜 Planned | Infrastructure |
| Tenant NAS Shares | ✅ Complete | Tenant Services |
| automation_spec (IaC) | ✅ Complete | Tenant Services |
| S3 Bucket Orders | 🔜 Planned | Tenant Services |
| REST API | ✅ Complete | Both |
| Bulk Import/Export | ✅ Complete | Both |
| ONTAP Auto-Import | ✅ Complete | Infrastructure |
| Turnkey Appliance | 🔜 Roadmap | Deployment |

---

## Slide 12: Summary

**NetBox ONTAP NAS Plugin**

1. **Infrastructure Layer** - Complete ONTAP object modeling
   - Open source foundation
   - Auto-import from live clusters
   - Full REST API

2. **Tenant Services Layer** - Self-service NAS provisioning
   - Order workflows (volume, qtree, combined)
   - IaC-ready automation_spec
   - Multi-entry point support

3. **Integration Ready** - Works with enterprise tooling
   - AWX/Ansible with MAF/AMF frameworks
   - ServiceNow, custom portals
   - Playbooks = customer project scope

4. **Flexible Delivery** - Support contract or T&M

5. **Future** - S3 buckets, turnkey appliance

---

## Contact & Resources

- Plugin Repository: [Internal GitLab/GitHub]
- Documentation: `/opt/netbox/myplugins/netbox_ontap_nas/docs/`
- Demo Environment: [Lab URL]

---

*Document Version: 1.0 | January 2026*

---

## Appendix: Plugin Domain Coupling Analysis

### A.1 Architecture Overview - Two Distinct Domains

| Domain | Models | Purpose |
|--------|--------|---------|
| **ONTAP Infrastructure** | `ONTAPCluster`, `ONTAPNode`, `ONTAPsvm`, `ONTAPVolume`, `ONTAPQtree`, `ONTAPSnapshotPolicy`, `ONTAPExportPolicy`, `ONTAPTier`, `ONTAPInterface`, `ONTAPSnapMirror`, `ONTAPQuotaRule`, `ONTAPLUN` | Represents actual NetApp ONTAP resources (discovered/synced from ONTAP) |
| **Tenant NAS Services** | `TenantNASShare` | Represents tenant storage orders/requests (order management layer) |

---

### A.2 Validators Classification by Domain Dependency

#### Pure Format Validators (ZERO ONTAP dependency) ✅

| File | Validators | Dependencies |
|------|------------|--------------|
| `validators/tenant/format.py` | `validate_size_format`, `validate_mount_point`, `validate_order_type`, `validate_availability_class`, `validate_protocol` | Only constants + Django RegexValidator |
| `validators/ontap/format.py` | `validate_volume_name`, `validate_qtree_name` | Only ONTAP naming constants |
| `validators/*/constants.py` | `VALID_ORDER_TYPES`, `VALID_AVAILABILITY_CLASSES`, `VALID_PROTOCOLS`, `VOLUME_NAME_REGEX` | None |
| `validators/*/messages.py` | `ONTAPValidationMessages`, `TenantValidationMessages`, `format_error` | None |

**These can be completely reused if you detach Tenant NAS Services.**

---

#### ONTAP-Coupled Validators (Import ONTAP models) ⚠️

| File | Validators | ONTAP Models Used |
|------|------------|-------------------|
| `validators/ontap/existence.py` | `validate_svm_exists` | `ONTAPsvm` |
| `validators/ontap/existence.py` | `validate_volume_exists`, `validate_volume_is_rw`, `validate_volume_not_dp_destination` | `ONTAPVolume` |
| `validators/ontap/existence.py` | `validate_snapshot_policy_exists` | `ONTAPSnapshotPolicy` |
| `validators/ontap/uniqueness.py` | `validate_volume_unique_in_svm` | `ONTAPVolume` |
| `validators/ontap/uniqueness.py` | `validate_qtree_unique_in_volume` | `ONTAPQtree` |

---

#### Business Validators (Indirect ONTAP dependency via existence.py) ⚠️

| File | Validators | What They Do |
|------|------------|--------------|
| `validators/tenant/business.py` | `validate_new_qtree_order` | Calls `validate_svm_exists`, `validate_volume_exists` when `check_db=True` |
| `validators/tenant/business.py` | `validate_automation_spec` | Dispatches to order-type validators |

---

### A.3 Coupling Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                        VALIDATORS MODULE                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────────────┐                                             │
│  │  validators/tenant/ │◄── Tenant-specific (order types, sizes)     │
│  │  format.py          │    Can import from validators/ontap/        │
│  │  constants.py       │                                             │
│  │  messages.py        │                                             │
│  └─────────────────────┘                                             │
│           │                                                           │
│           ▼                                                           │
│  ┌─────────────────────┐      ┌────────────────────────────────┐    │
│  │  validators/tenant/ │─────►│  validators/ontap/             │    │
│  │  business.py        │      │  existence.py + uniqueness.py  │    │
│  │  (when check_db=    │      │                                │    │
│  │   True)             │      │  IMPORTS:                      │    │
│  └─────────────────────┘      │  - ONTAPsvm                    │    │
│                               │  - ONTAPVolume                  │    │
│                               │  - ONTAPQtree                   │    │
│                               │  - ONTAPSnapshotPolicy          │    │
│                               │  - ONTAPCluster                 │    │
│                               └────────────────────────────────┘    │
│                                           │                          │
└───────────────────────────────────────────│──────────────────────────┘
                                            │
                                            ▼
                              ┌─────────────────────────────┐
                              │     ONTAP INFRASTRUCTURE    │
                              │         MODELS              │
                              │  (Discovered from ONTAP)    │
                              └─────────────────────────────┘
```

---

### A.4 TenantNASShare Model Dependencies on ONTAP

```python
class TenantNASShare(NetBoxModel):
    # Core Tenant field (NOT ONTAP)
    tenant = ForeignKey('tenancy.Tenant')      # ← NetBox core, NOT ONTAP
    
    # ONTAP FK relationships (COUPLING POINTS):
    svm = ForeignKey('ONTAPsvm')               # ← ONTAP dependency
    volumes = ManyToManyField('ONTAPVolume')   # ← ONTAP dependency
    qtrees = ManyToManyField('ONTAPQtree')     # ← ONTAP dependency
    export_policy = ForeignKey('ONTAPExportPolicy')  # ← ONTAP dependency
    
    # Pure service fields (NO ONTAP dependency):
    order_type, status, protocol, nas_path, mount_point,
    availability_class, automation_spec, app_type, ...
```

**TenantNASShare has 4 direct FK/M2M relationships to ONTAP models.**

---

### A.5 Decoupling Options

#### Option A: Soft Decoupling (Keep validators, skip DB checks)

The validators already support this via `check_db=False` parameter:

```python
# validators/tenant/business.py
def validate_new_qtree_order(automation_spec, record_num=None, check_db=True):
    # When check_db=False, skips all ONTAP model lookups
    if svm_name and volume_name and check_db:  # ← Only queries DB when True
        error, svm = validate_svm_exists(svm_name)
```

**Impact:** Just pass `check_db=False` to skip ONTAP validation. Format validation still works.

---

#### Option B: Hard Decoupling (Separate plugins)

| Plugin | Models | Validators Needed |
|--------|--------|-------------------|
| `netbox-ontap-infrastructure` | All ONTAP* models | `validators/ontap/` (keep as-is) |
| `netbox-tenant-nas-services` | `TenantNASShare` | `validators/tenant/` (imports from ontap plugin) |

**Changes Required:**

1. **TenantNASShare model changes:**
   - Remove `svm`, `volumes`, `qtrees`, `export_policy` FKs
   - Store SVM/volume names in `automation_spec` only (already there)
   - Add `svm_name`, `volume_names`, `qtree_names` as CharField/JSONField

2. **Validators changes:**
   - Change imports: `from netbox_ontap_infrastructure.validators import ...`
   - Tenant plugin depends on ONTAP infrastructure plugin

3. **API changes:**
   - Tenant service accepts SVM/volume names as strings
   - Validation against ONTAP infrastructure optional
   - Full validation happens at execution time (external automation)

---

### A.6 Decoupling Effort Estimate

| Approach | Effort | Risk |
|----------|--------|------|
| **A: check_db=False** | 1 day | Low - Just change flag, existing data works |
| **B: Soft FK removal** | 3-5 days | Medium - Migrations needed, some views/forms/serializers changes |
| **C: Full plugin split** | 2-3 weeks | High - Two codebases, separate releases, complex testing |

---

### A.7 Recommendation

**The current validators architecture already supports decoupling via `check_db=False`.**

For stronger decoupling:

1. **Short-term:** Use `check_db=False` for Tenant NAS validation when ONTAP infrastructure isn't populated
2. **Medium-term:** Make the FKs in TenantNASShare nullable (they already are!) and stop populating them
3. **Long-term:** If you truly need separate plugins, extract `validators/ontap/` first (zero risk), then `validators/tenant/` imports from ONTAP plugin

The validators module is structured with this separation in mind:
- `validators/ontap/` - Standalone ONTAP validators (no tenant dependencies)
- `validators/tenant/` - Tenant validators (can import from ontap, but ontap cannot import from tenant)