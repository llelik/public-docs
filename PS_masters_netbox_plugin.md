# NetBox ONTAP Plugin: ONTAP as a Structured, Automatable Source of Truth

---

## Slide 1: Title

**NetBox ONTAP NAS Plugin: Automation Solution Overview**

- Object model, metadata, drift detection, and automation integration
- NetApp Professional Services

---

## Slide 2: What Is NetBox, and Why Do Customers Choose It?

- Open-source infrastructure source of truth, built around **DCIM** and **IPAM**
- Core hierarchy: Regions > Sites > Locations > Racks > Devices
- Also covers tenants, IP addresses, prefixes, VLANs, VRFs, cables, and power
- Plugin framework adds custom models, views, and APIs
- Full REST API for every object type

**Why customers choose it:**
- **Operations:** structured documentation reduces outage risk, speeds incident response, and supports AI agents
- **Automation:** configuration generation and workflow orchestration replace manual processes
- **Observability:** discovery keeps the source of truth synchronized with reality
- **Availability:** CMDB context is available to other consumers via API

**Positioning:**
- An **open-source alternative to ServiceNow for infrastructure CMDB use cases**, now with an ONTAP integration plugin
- ServiceNow also appears in our architecture as a request source, so the two can coexist
- Not a full ITSM replacement

*Speaker note: this is vocabulary, not a NetBox training. Most of the audience is hearing the name for the first time.*

---

## Slide 3: NetBox Features the Plugin Builds On

| NetBox capability | How the plugin uses it |
|---|---|
| Plugin framework | ONTAP objects are first-class NetBox models, views, and API endpoints |
| REST API | Automation reads and writes ONTAP objects and `automation_spec` |
| **Tenancy** | Tenant association on core objects |
| **Custom Fields** | Ownership, lifecycle, and protection metadata that drive decisions |
| RBAC | Object-level permissions on storage objects |
| Bulk import/export | CSV, JSON, and YAML for mass onboarding |
| DCIM / IPAM | Links storage to racks, devices, cabling, IPs, VLANs, and VRFs |
| Event rules / webhooks | Create, Update, and Delete events trigger provisioning |
| Change logging | Audit trail of object changes |

---

## Slide 4: Plugin Architecture: Two-Layer Design

**Infrastructure layer (ONTAP objects)**
- Clusters, nodes, SVMs, volumes, qtrees, LUNs, interfaces, export policies
- SnapMirror and snapshot policies
- ONTAP S3 buckets (planned, StorageGRID still an open question)
- Add-ons driven by customer requests

**Tenant Services layer**
- Storage requests organized as application-aware **Orders**
- Order types: `new_app`, `new_volume`, `new_qtree`, and others
- Logic keyed on `app_type`
- Self-service request workflow produces an **`automation_spec`** (IaC-ready JSON)
- Optional OS mount points

**PS value-add:** application business logic, workflows, and automation integration

---

## Slide 5: NetBox ONTAP Plugin: Solution Features

- Auto-import from live ONTAP clusters
- Cross-cluster SnapMirror tracking
- Full REST API
- Bulk import/export (CSV, JSON, YAML)
- Ansible modules covering all objects
- Create / Update / Delete event support (Python, AWX, APP, or others)
- Every object can be automated **without** the NAS Services plugin
- Customer-driven playground: other resources by request

**Visual:** Discover > Document > Compare > Automate > Report

---

## Slide 6: NetBox ONTAP Plugin: ONTAP Objects

- **Cluster:** cluster, HA pairs, node details, aggregates, storage tiers
- **SVM:** interfaces (LIFs), volumes > qtrees and LUNs, quotas
- **Block access:** iGroups, LUN mappings
- **NAS access:** export policies > export policy rules
- **Protection:** SnapMirror relationships, policies, schedules; snapshot policies
- **Other resources** by request

*Visual: show as a hierarchy, with protection and access relationships alongside rather than forced into one tree.*

---

## Slide 7: Import, Sync & Drift Detection

| Operation | Purpose | Direction |
|---|---|---|
| **Import** | Read ONTAP instance configuration to build the initial inventory | ONTAP > NetBox |
| **Sync** | Compare ONTAP data with NetBox, then write updates to NetBox | ONTAP > compare > NetBox |
| **Reports** | Read NetBox CMDB data and compute the delta (configuration drift) | NetBox vs ONTAP |
| **IaC export** | Export configuration for infrastructure-as-code use | NetBox > files |

**Design principles:**
- Define which system owns each field
- Preserve CMDB-only metadata (Custom Fields, tenancy) during refresh
- Treat a missing object as something to investigate, not as an approved deletion
- A drift report is information; it does not change ONTAP by itself

---

## Slide 8: Integration Architecture

```text
Request sources: self-service portals / custom scripts / ServiceNow (ITSM)
                    │
                    ▼
        NetBox (Web UI, REST API)
        Objects • Relationships • Custom Fields • Tenants
                    │
        Tenant Services: receive order, validate, store automation_spec
        Generic objects: trigger events
                    │
                    ▼
        AWX / Ansible Tower
        Retrieves automation_spec via REST API
                    │
                    ▼
        Ansible storage frameworks
        NetApp PS Ansible collections, pre-built roles
        (playbooks = customer project scope, NOT in plugin)
                    │
                    ▼
                  ONTAP
                    │
        Provisioning status / discovery ──► NetBox
```

**End-to-end flow:** Request > Validate > Store spec > Event > Playbook > Create validated objects > Reconcile

**Integration context:** ONTAP objects link to NetBox core data (racks, cabling, ports, IPs, VLANs, VRFs) and to virtualization and application plugins.

*Key point: NetBox holds context and intent. The automation backend performs execution.*

---

## Slide 9: Custom Fields: The Decision Layer

**Examples:** `service_owner`, `application_id`, `environment`, `data_classification`, `protection_tier`, `automation_policy`, `lifecycle_state`, `decommission_approved`

```text
IF environment = production
AND protection_tier = critical
THEN select the approved protection workflow
AND require additional approval for destructive changes
```

- Typed, structured fields are easier to validate than free-form labels
- Automation uses Custom Fields to choose workflows and parameters
- Metadata sits next to the objects it describes
- ONTAP tags alone do not provide structured CMDB context or governance

*Custom Fields hold the decision inputs; automation implements the decisions.*

---

## Slide 10: Tenancy and Ownership: What ONTAP Metadata Misses

| ONTAP configuration provides | CMDB operations also need |
|---|---|
| Cluster, SVM, volume, and LUN configuration | Business owner and accountable support team |
| Capacity and placement | Application, service, and cost-center association |
| Export and access configuration | Approved consumers and business purpose |
| Protection settings and relationships | Protection tier, RPO/RTO, policy rationale |
| Current object existence | Lifecycle state, approval history, decommission intent |

- **Tenancy** is applied on NetBox core objects (sites, racks, devices, IPs, VLANs, and so on)
- **Ownership** is captured on every plugin object, using Custom Fields
- Together they answer: who owns this, why does it exist, and who pays for it?

*Speaker note: ONTAP knows how an object is configured. The CMDB explains why it exists and who is accountable.*

---

## Slide 11: Design Balance: ONTAP Fidelity vs CMDB Practice

**The plugin tries to hold two things at once:**

| Keep ONTAP design | Keep the CMDB way |
|---|---|
| ONTAP object names, hierarchy, and relationships stay recognizable | Objects link to sites, racks, devices, IPs, VLANs, and tenants |
| Cluster > SVM > Volume > Qtree/LUN structure is preserved | Ownership, lifecycle, and business context live in Custom Fields |
| SnapMirror, export policies, and snapshot policies are modeled as in ONTAP | Records are stable, reportable, and queryable through the API |

- NetBox is not another storage GUI: ONTAP stays the system that executes
- NetBox is the system of record for context, ownership, and intent
- Observed state (import), intended state (NetBox), and execution (automation) are kept separate

*Speaker note: an ONTAP expert should recognize every object. A CMDB expert should recognize every workflow.*

---

## Slide 12: Safe by Design: Decommissioning and Accidental Removal

**Target safeguards:**
- Decommissioning is an explicit lifecycle step, not a plain record deletion
- Dependency checks before removal (for example, a volume with SnapMirror relationships, mappings, or exports)
- Accidental removals are blocked or require confirmation
- An object missing during import or sync is flagged for review, not auto-deleted
- Destructive operations require an approval field (for example, `decommission_approved`)

**Lifecycle:**
`Active > Decommission requested > Approved > Executed > Retired`

**Key principle:**
> Deleting a CMDB record must never silently delete live storage.

*Speaker note: customer-facing wording is "safe by design" rather than "idiot-proof". Confirm which safeguards are implemented today and which are planned, and label them on the slide.*

---

## Slide 13: Ransomware Protection Configuration in Custom Fields

**Idea:** record the protection requirement as structured metadata, and let automation act on it.

**Suggested fields:**
- `ransomware_protection_required`
- `protection_profile`
- `data_classification`
- `snapshot_policy_class`
- `required_retention`
- `security_owner`
- `exception_reference`

**Flow:**
`Custom Field intent > policy selection > automation (Ansible/AWX) > ONTAP configuration > drift report`

- Custom Fields record the **desired** protection and governance context
- Automation translates that intent into supported ONTAP settings
- Import and drift detection compare actual ONTAP configuration against the intent
- Custom Fields do not provide protection themselves; ONTAP features do

*Speaker note: the field names are examples. Align them with the ONTAP protection features you want to cover.*

---

## Slide 14: Roadmap

**Protection and topology (requested priorities):**
- **SnapMirror S3:** model S3 replication relationships (ONTAP S3 buckets are already listed as planned; StorageGRID is an open question)
- **SnapMirror topology:** visualize replication relationships across clusters
- **Storage topology:** show the chain from physical infrastructure to logical storage and consumers

**Day-0 operations (from the deck):**
- Cluster configuration
- Node join / unjoin

**Lowering the entry barrier: turnkey appliance (from the deck):**
- VMware / KVM appliance
- NetBox + ONTAP NAS plugin
- ONTAP cluster import service
- AnsibleForms (simple GUI)
- Pre-configured AWX (optional)
- Benefits: deploy in minutes, no NetBox expertise required, ideal for PoC and smaller deployments, upgrade path to full enterprise

*Label on slide: roadmap items are plans, not committed delivery dates.*

---

## Slide 15: Available for MUCCBC Users

**Instance:** https://ipam02.muccbc.hq.netapp.com

**Suggested starting path:**
1. Browse the ONTAP object hierarchy
2. Inspect tenant, ownership, and Custom Field metadata
3. Review an import and a drift report
4. Pick one workflow to automate

*Speaker note: confirm access requirements and which operations are permitted on the shared instance.*

---

## Slide 16: Demo Call

**See it live: from inventory to controlled automation**

**Proposed demo flow:**
1. Browse cluster > SVM > volume > qtree/LUN
2. Show tenant and ownership context
3. Show the Custom Fields that drive decisions
4. Run an import or sync and review the result
5. Show a drift report
6. Submit an order and show the `automation_spec` it produces
7. Show a decommission safeguard (only if implemented and safe to demo)

**Discussion question:** which workflow would you automate first: provisioning, protection validation, or decommissioning?

*Use a sandbox or recorded clip for any destructive scenario.*

---

## Slide 17: Thank You

**NetBox ONTAP NAS Plugin: Automation Solution Overview**
NetApp Professional Services

- Questions
- Demo call details
- MUCCBC instance: https://ipam02.muccbc.hq.netapp.com
