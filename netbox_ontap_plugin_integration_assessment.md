# NetBox ONTAP NAS Plugin — Security, Access Control & Integration Assessment

**Audience:** Customer DC/infrastructure, security, and storage teams evaluating whether to integrate `netbox-ontap-nas` into an existing, security-hardened vanilla NetBox deployment.

**Scope:** This document assesses (1) the access-control/RBAC/ownership model the plugin exposes on top of vanilla NetBox, and (2) the plugin's data-model coupling to native NetBox objects — what it depends on, how it can affect native object operations, and what the operational consequences are.

**Method:** Static review of the plugin source (`myplugins/netbox_ontap_nas/`) — models, views, API viewsets, forms, validators, navigation, and plugin configuration — cross-referenced against NetBox core (`netbox/netbox/models/`, `netbox/users/`) to confirm which behaviors are inherited from the platform vs. implemented by the plugin.

**Plugin version reviewed:** 0.1.5 (`min_version = 4.5.0`)

---

## 1. Executive Summary

- The plugin introduces **no new authentication, authorization, or permission mechanism**. Every model, UI view, and REST endpoint uses NetBox's stock building blocks (`NetBoxModel`, `NetBoxModelViewSet`, `netbox.views.generic.*`). Access control is 100% governed by NetBox's existing Users/Groups/Tokens + `ObjectPermission` framework — there is nothing plugin-specific to separately secure, audit, or bypass.
- All 19 plugin models participate in NetBox's standard governance features automatically: permissions, object-level permission **constraints**, custom fields, tags, change logging, journaling, webhooks/event rules, and bookmarks.
- All 17 `infrastructure/` models also inherit NetBox's native `OwnerMixin` (**ownership**), giving the customer a ready-made per-object "owner" (a `users.Owner`, itself mappable to Users, Groups, or an `OwnerGroup`) that can be used to drive self-service RBAC scoping — this is a core NetBox 4.5 feature, not something the plugin invented. The 2 `services/` models use `tenant`-based scoping instead (see §3.3).
- The plugin is a **passive data/inventory layer**: it stores structured records about ONTAP infrastructure and tenant NAS provisioning requests. It contains **no code that calls out to ONTAP clusters, no stored credentials/secrets, no `required_settings`/`default_settings`, and no custom middleware or authentication backends**. It does not create or mutate native NetBox objects — it only references pre-existing ones selected by the user.
- The plugin **depends on** (has foreign keys into) five native apps: `tenancy.Tenant`, `dcim.Site`, `dcim.Device`, `ipam.IPAddress`, and `virtualization.VirtualMachine`. Several of these relationships use `on_delete=PROTECT`, meaning **deleting those native objects can be blocked** once storage/tenant data references them. This is the main operational consequence DC/network teams need to plan for (see §4).
- No signals, template extensions, or view overrides touch native object detail pages or native model lifecycle — the coupling is one-directional (plugin → core) and confined to the plugin's own views/serializers.
- **The plugin is safe to install and, later, safe to remove.** Because native NetBox objects never reference plugin data (only the reverse), installing the plugin cannot alter or restructure any existing NetBox data, and uninstalling it later cleanly removes only the plugin's own storage records — it cannot orphan or corrupt Tenants, Sites, Devices, IP Addresses, or Virtual Machines (see §4.5).

**Bottom line for the decision:** integrating this plugin does not weaken or bypass any existing NetBox security control, and does not require the customer to design a new RBAC model — it reuses the one already in place. The primary integration planning item is coordinating with teams that own Tenants/Sites/Devices/IP addresses, since those objects can become "in use" (and protected from deletion) once storage data references them.

---

## 2. Plugin Architecture Overview

```
netbox_ontap_nas/
├── infrastructure/   # ONTAP infra objects (17 models): Cluster, Node, SVM, Volume, Interface,
│                      # ExportPolicy(+Rule), Qtree, QuotaRule, SnapshotPolicy, SnapMirror(+Policy),
│                      # Tier, JobSchedule, igroup, LUN, LUNMap
├── services/          # Tenant-facing NAS provisioning (2 models): TenantNASShare, TenantNASMountPoint
├── validators/        # Shared business-rule validation (format, uniqueness, immutability, relationships)
├── api/                          + infrastructure/api/, services/api/   # DRF serializers & viewsets
└── forms/, tables/, templates/, navigation.py, urls.py, views.py
```

All 19 models inherit `NetBoxModel` (`netbox/netbox/models/features.py` stack), which is what provides:

| Feature | Inherited automatically? |
|---|---|
| Django model permissions (`add_x`/`change_x`/`delete_x`/`view_x`) | Yes |
| `ObjectPermission` row-level constraints | Yes |
| Custom fields | Yes |
| Tags | Yes |
| Change logging (audit trail) | Yes |
| Journal entries | Yes |
| Webhooks / event rules | Yes |
| Bookmarks | Yes |
| GraphQL exposure | Yes (auto-generated by NetBox for plugin models) |

No plugin code re-implements or overrides any of the above.

---

## 3. Access Control, RBAC, and Ownership

### 3.1 Authentication & API surface

- REST API: every viewset (`ONTAPClusterViewSet`, `TenantNASShareViewSet`, etc.) subclasses `netbox.api.viewsets.NetBoxModelViewSet` with **no overrides** of `permission_classes` or `authentication_classes`. Confirmed by source review — there are zero occurrences of `permission_classes`, `AllowAny`, `IsAuthenticated`, `csrf_exempt`, or `@api_view` in the plugin. This means the plugin uses exactly the same Token/Session authentication and `TokenPermissions`/object-permission enforcement as every core NetBox endpoint.
- UI: every view subclasses `netbox.views.generic.{ObjectView, ObjectListView, ObjectEditView, ObjectDeleteView, BulkEditView, BulkImportView, BulkDeleteView}`, which enforce NetBox's standard `ObjectPermissionRequiredMixin` — a logged-in user without the corresponding `view_x`/`add_x`/`change_x`/`delete_x` permission (or a failing `ObjectPermission` constraint) is denied exactly as with any built-in NetBox object.
- Navigation (`navigation.py`): every menu item and button declares an explicit `permissions=[...]` gate (e.g. `netbox_ontap_nas.view_ontapcluster`, `netbox_ontap_nas.add_ontapnode`), so unauthorized users don't even see menu entries for objects they can't access — standard NetBox convention, not a plugin-specific control (menu hiding is cosmetic; the view-level permission check is the actual enforcement).

### 3.2 Object-level permissions (RBAC)

Because every plugin model is a standard `NetBoxModel`, the customer's security/admin team can create native `users.ObjectPermission` records against plugin content types exactly as they would for `dcim.Device` or `ipam.IPAddress` today:

- Scope by **action** (`view`/`add`/`change`/`delete`) per model.
- Scope by **constraint** (a JSON queryset filter), e.g. restrict a storage-operator group to only `TenantNASShare` objects where `tenant__in=[...]`, or to `ONTAPVolume` objects in a given `svm__cluster__site`.
- Combine with **Groups** for team-based access (e.g. "Storage-Operators", "Storage-Read-Only", "Tenant-Self-Service").

No plugin-specific admin screen is needed to configure this — it is done entirely through NetBox's existing **Users → Permissions** admin UI.

### 3.3 Ownership

Every one of the 17 infrastructure objects (clusters, SVMs, volumes, LUNs, export policies, snapshot/SnapMirror policies, etc.) has a built-in **Owner** field, using a feature that already exists in NetBox itself — the plugin doesn't add or change anything here, it simply turns the feature on for its own objects.

In practice this means every infrastructure object can be assigned an accountable owner — a specific person, or a group of people/team — the same way NetBox lets you assign an owner to a device or a circuit today. That gives two practical benefits:

- **Accountability**: it's always possible to see, and report on, who is responsible for a given cluster, SVM, or volume.
- **Self-service access control**: a security administrator can configure permissions so that a storage team, tenant, or individual can only view/edit the objects they own, without needing to build anything custom — this is done through NetBox's existing permissions screens.

The 2 tenant-facing objects (`TenantNASShare`, `TenantNASMountPoint` — the actual NAS shares/mount points requested by and delivered to tenants) don't use the Owner field. Instead, they are always tied to a required **Tenant**, which is the natural way to scope access/visibility for those — e.g. "tenant X's team can only see/manage tenant X's shares."

### 3.4 What the plugin does *not* add

- No custom or weakened permission logic of any kind — access checks always go through the same NetBox permission engine used everywhere else in the product.
- No additional login methods, no bypass of NetBox's login/CSRF protections, and no stored secrets or credentials of any kind associated with the plugin.
- No changes to how Sites, Tenants, Devices, Virtual Machines, or IP Addresses look or behave in the rest of NetBox — their pages, forms, and workflows are untouched.
- **Roadmap item, not yet built**: the plugin maintainer has a documented design (not yet implemented) for an extra safety step on deletes — requiring an operator to type the object's name before a delete is allowed, similar to safeguards used by some cloud consoles. Today, deleting a plugin object uses NetBox's normal single confirmation screen, which already lists any dependent/protected objects (see §4) before allowing the delete to proceed.

---

## 4. Dependencies on Native NetBox Objects & Consequences

The plugin has foreign keys into five native apps. No native app has any FK or dependency back into the plugin (coupling is one-directional: plugin → core). The plugin never programmatically creates, updates, or deletes native objects — every FK is populated by a user selecting an **existing** Tenant/Site/Device/IPAddress/VirtualMachine from a dropdown (`DynamicModelChoiceField` against `Model.objects.all()`), confirmed by source review (no `Tenant.objects.create(...)`, `Device.objects.create(...)`, `IPAddress.objects.create(...)`, etc. anywhere in the plugin).

### 4.1 Relationship map and delete behavior

| Native object | Referenced by (plugin model.field) | `on_delete` | Consequence of deleting the native object |
|---|---|---|---|
| `tenancy.Tenant` | `ONTAPCluster.tenant`, `ONTAPVolume.tenant`, `ONTAPsvm.tenant`, `ONTAPExportPolicy.tenant` (implied via chain), `TenantNASShare.tenant` | `PROTECT` | **Deletion is blocked** while any plugin object references the tenant. NetBox's standard delete view catches `ProtectedError` and shows the normal "cannot delete — referenced by" dependent-objects list, so this fails safely and visibly (not a crash) — but Tenant lifecycle/offboarding processes must account for storage data. |
| `dcim.Site` | `ONTAPCluster.site`, `ONTAPsvm.site` | `PROTECT` | Same as above — Site deletion blocked while clusters/SVMs are assigned to it. |
| `dcim.Site` | `ONTAPNode.site` | `SET_NULL` | Site deletion **succeeds**; the Node's `site` is silently cleared. No error is raised — DCIM teams should be aware that decommissioning a Site does not warn about ONTAP Nodes still pointing at it before the FK is nulled. |
| `dcim.Device` | `ONTAPNode.device` | `SET_NULL` | Device deletion (e.g. hardware decommission in DCIM) **succeeds**; the storage Node record silently loses its link to the physical device. |
| `ipam.IPAddress` | `ONTAPCluster.management_ip`, `ONTAPInterface.ip_address` | `PROTECT` | **Deletion is blocked** while a cluster management IP or a data LIF references the address. Network/IPAM teams need visibility into which IPs are "claimed" by storage before attempting cleanup/reclamation. |
| `virtualization.VirtualMachine` | `TenantNASShare.virtual_machine`, `TenantNASMountPoint` (VM reference) | `SET_NULL` | VM deletion **succeeds**; the NAS share/mount-point record silently loses its VM association (share itself is not deleted). |

Additional notes:
- All parent/child relationships **within** the plugin's own data (e.g. `ONTAPVolume → ONTAPQtree`, `ONTAPsvm → ONTAPVolume`, `ONTAPCluster → ONTAPNode`) use `CASCADE` or `PROTECT` as appropriate — this is internal to the plugin and does not touch native NetBox tables.
- `TenantNASShare.volumes` / `.qtrees` are `ManyToMany` fields to the plugin's own `ONTAPVolume`/`ONTAPQtree` models, not to native objects.

### 4.2 Extra safety checks before deleting storage objects

Beyond the table above, the plugin adds one further safeguard specifically for storage volumes: before a volume can be deleted, the plugin checks whether it is still protected by an active replication relationship (SnapMirror), or whether it still contains child objects (qtrees or LUNs). If either condition is true, the deletion is refused with a clear on-screen explanation of what needs to be removed or unwound first.

This check applies **regardless of the user's permission level** — even an administrator with full delete rights cannot accidentally remove a volume that is still actively in use. They must first remove its child objects or break the replication relationship, mirroring how safe storage changes are normally carried out on the ONTAP system itself. The check runs entirely inside NetBox and does not involve any communication with the actual storage array.

### 4.3 No hidden network egress or credential storage

- The plugin does not make any outbound connections to ONTAP clusters or any other external system, and it does not store any passwords, API keys, or other credentials. There is nothing in the plugin configuration that requires credentials to be entered or held anywhere.
- Fields like `ONTAPCluster.ontap_details` (JSON) and `TenantNASShare.automation_spec` (JSON) are **data containers only** — they are populated by whatever external automation/orchestration process the customer runs (e.g., Ansible/Terraform reading NetBox as source-of-truth and separately calling the ONTAP REST API). The plugin itself never reaches out to storage hardware.
- Practical implication: the plugin adds **no new network egress requirement or credential-management burden** to the NetBox server itself. Any ONTAP-side automation the customer builds around this data is a separate system, and its own credentials, network path to the storage cluster, and access controls should be assessed independently by the customer when that automation is designed.

### 4.4 Bulk import validation path (technical note for future code reviewers)

When records are loaded in bulk via JSON/YAML (rather than entered one at a time through the web form), some validation rules run at a different stage of processing than they do for single-record entry. This does **not** change who is allowed to import data — the same "add" permission is still required either way — it only affects *where in the code* the data gets checked. Anyone doing a deeper code-level security review of the plugin later should be aware of this and check both the form-level and bulk-import-level validation logic, rather than assuming one covers the other.

### 4.5 Safe to install now, and safe to remove later

The relationship between the plugin and native NetBox data only ever runs in one direction: plugin records can point at existing Tenants, Sites, Devices, IP Addresses, and Virtual Machines, but none of those native NetBox objects are ever changed to point back at the plugin. Practically, that has two consequences the customer can rely on:

- **Installing the plugin is non-destructive.** It adds new database tables and new menu items for storage objects; it does not modify, re-tag, migrate, or restructure any pre-existing Tenant, Site, Device, IP Address, or Virtual Machine data. Existing NetBox workflows continue to work exactly as before.
- **Removing the plugin later is clean.** Uninstalling simply drops the plugin's own tables (clusters, SVMs, volumes, shares, etc.) and removes its menus and permissions. Because no native NetBox table has a reference pointing into plugin data, this cannot orphan, corrupt, or cascade-delete anything in Tenants, Sites, Devices, IP Addresses, or Virtual Machines — those records are left exactly as they were, simply without the (now-removed) storage-side references to them.

In short: the customer can trial or adopt the plugin with confidence that it can be backed out later without any cleanup work on their existing NetBox data.

### 4.6 How data gets into NetBox, and what ONTAP access that requires

The plugin itself is passive — as noted in §4.3, it does not talk to ONTAP. Getting data into the plugin (or provisioning new storage) is handled by a separate, companion set of Ansible playbooks and modules (`netbox_ontap`) built specifically for this purpose. Since these are the tools customers will actually run against their ONTAP clusters, it's useful to call out the access they need:

- **Import / sync / reporting playbooks** — these connect to ONTAP purely to *read* its configuration (clusters, SVMs, volumes, LUNs, export policies, etc.) and then create or update the matching records in NetBox. They do not make any configuration changes on the storage system. The ONTAP account used for these playbooks only needs **read-only** access — no ONTAP write privileges are required for import, sync, or reporting.
- **Provisioning playbooks** — a separate Ansible-based framework handles the reverse direction: taking a provisioning request recorded in NetBox (e.g., a new NAS share) and creating it on the ONTAP cluster. Because this framework actually changes storage configuration, it requires an ONTAP account with **write access**.
- **Access can be scoped down using ONTAP's own controls.** ONTAP's built-in role-based access control (RBAC) can be used to create a dedicated automation account with permissions limited to exactly the operations the provisioning playbooks perform (e.g., creating volumes/shares), rather than granting broad cluster-admin rights. This lets the customer's storage team keep the same least-privilege model they'd apply to any other automation account.
- **The Ansible collection is the recommended way to automate against the plugin, but not the only way.** Every plugin object is also exposed through NetBox's standard REST API with full create/read/update/delete support, so a customer's existing tooling (scripts, other automation platforms, CI/CD pipelines, etc.) can integrate directly against the API using the same NetBox token/permission model described in §3, without needing to adopt Ansible specifically.

In short: read-only ONTAP access is sufficient to keep NetBox synchronized with reality, and any write access needed for provisioning automation can be scoped narrowly through ONTAP's native RBAC rather than requiring full administrative credentials.

---

## 5. Summary Risk Assessment

| Area | Assessment |
|---|---|
| New attack surface (auth/authz) | None — reuses NetBox's existing user/token auth and `ObjectPermission` engine verbatim. |
| Privilege escalation risk | None identified — no `AllowAny`, no permission bypass, no custom middleware. |
| Secrets / credential handling | None — plugin stores no credentials and makes no outbound calls to ONTAP. |
| Data integrity impact on native objects | Low — plugin never creates/mutates native objects; only references existing ones. |
| Operational impact on native object lifecycle | Moderate, and predictable — `PROTECT` FKs (Tenant, Site via Cluster/SVM, IPAddress) can block deletions; `SET_NULL` FKs (Site/Device via Node, VirtualMachine) silently clear links. Both are standard, visible Django/NetBox behaviors, not plugin bugs. |
| Auditability | Full — change logging, journaling, and (where applicable) `owner` attribution are available on every plugin object out of the box. |

---

*Prepared from static source review of `netbox_ontap_nas` v0.1.5 against NetBox 4.5.9 core. No dynamic/runtime penetration testing was performed as part of this assessment.*
