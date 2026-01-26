# NetApp ONTAP SVM Disaster Recovery Automation
## Technical Reference Guide

---

## Table of Contents

1. [Executive Overview](#executive-overview)
2. [SVMDR Standard Workflow](#svmdr-standard-workflow)
3. [SVMDR Pseudo (Custom) Workflow](#svmdr-pseudo-custom-workflow)
4. [QuickDR Workflow](#quickdr-workflow)
5. [SnapMirror Operations Tracking](#snapmirror-operations-tracking)
6. [The svm_clone Role Deep Dive](#the-svm_clone-role-deep-dive)
7. [Customization Options](#customization-options)
8. [CIFS Server Management](#cifs-server-management)
9. [Pre-flight Validation](#pre-flight-validation)
10. [Integration and Deployment](#integration-and-deployment)

---

## Executive Overview

### The Challenge

Manual disaster recovery testing and failover operations are:
- **Time-consuming and error-prone** - Hours of manual steps with high risk of mistakes
- **Rarely tested** - DR plans remain unvalidated until real disasters strike
- **Inconsistent** - Different procedures across environments and teams
- **Difficult to audit** - Limited visibility into what was done and when

### The Solution

This automation framework provides three distinct DR workflows built on Ansible, each addressing specific use cases:

| Workflow | Technology | Primary Use Case | SnapLock Support |
|----------|------------|------------------|------------------|
| **SVMDR Standard** | Native NetApp SVMDR | Production DR with identity preservation | ❌ Not supported |
| **SVMDR Pseudo** | SVM Clone + SnapMirror | Production DR including SnapLock volumes | ✅ Full support |
| **QuickDR** | FlexClone technology | Rapid testing without SnapMirror | ⚠️ Testing only |

### Key Benefits

- **Reduced Recovery Time Objective (RTO)**: Minutes instead of hours
- **Consistency**: Same validated process every time
- **Auditability**: Complete logs and dry-run capability
- **Compliance Support**: Full SnapLock Enterprise and Compliance support via Pseudo workflow
- **Flexibility**: Extensive customization options for different customer requirements
- **Modern API**: 100% REST API implementation for future-proof automation

### Compatibility

> **✅ Tested and Validated:** All workflows have been thoroughly tested on **ONTAP 9.14** and are compatible with higher versions. The entire codebase uses **REST API only** strategy, ensuring modern, scalable, and future-proof automation that aligns with NetApp's strategic direction.

---

## SVMDR Standard Workflow

### Overview

The **SVMDR Standard** workflow leverages NetApp's native SVM Disaster Recovery (SVMDR) technology with identity preservation. This is the simplest and most straightforward DR solution for environments without SnapLock requirements.

### Technology Foundation

**Native NetApp SVMDR** provides:
- **Identity Preservation**: IP addresses, CIFS server names, and network identities maintained
- **Configuration Replication**: Automatic replication of SVM configuration elements
- **Single Relationship**: One SVMDR relationship manages the entire SVM
- **Minimal Overhead**: Most efficient in terms of management and resources

### Architecture

```
    PRODUCTION SITE                         DR SITE
    ┌─────────────────┐                    ┌─────────────────┐
    │  Source SVM     │                    │  Destination    │
    │  (Running)      │                    │  SVM (Standby)  │
    │                 │   SVMDR            │  subtype:       │
    │  ┌──────────┐  │   Relationship     │  dp-destination │
    │  │ Volumes  │  │   ═══════════►     │                 │
    │  │ Config   │  │   Identity         │  ┌──────────┐   │
    │  │ Network  │  │   Preserved        │  │ Replica  │   │
    │  └──────────┘  │                    │  └──────────┘   │
    └─────────────────┘                    └─────────────────┘
```

### Prerequisites

- SVMDR relationship **already established** between source and destination SVMs
- Identity preservation enabled (`identity-preserve: true`)
- Vserver peering configured
- Destination SVM must be in `dp-destination` subtype
- Network connectivity and proper licensing

### Supported Operations

#### Failover

Activates the DR site in a controlled manner:

1. **Validate States** - Verifies source SVM is running and destination is in standby
2. **Stop Source SVM** - Gracefully stops production SVM
3. **Final Sync** - Performs last SnapMirror update to minimize data loss
4. **Break Relationship** - Breaks SVMDR relationship
5. **Activate Destination** - Starts destination SVM with production identity
6. **Network Activation** - All network identities automatically preserved

**Use Cases:**
- Planned maintenance windows
- Datacenter migrations
- DR testing exercises
- Disaster recovery activation

#### Failback

Returns operations to the original production site:

1. **Create Reverse Relationship** - Establishes reverse SVMDR relationship
2. **Reverse Resync** - Synchronizes data back to original source
3. **Stop DR SVM** - Stops the currently active DR SVM
4. **Break Reverse** - Breaks the reverse relationship
5. **Activate Production** - Restarts original production SVM
6. **Re-establish Forward** - Recreates original SVMDR relationship

**Use Cases:**
- Return after planned maintenance
- Restoration after successful DR test
- Return to production after disaster resolution

### Important Limitations

| Feature | Support Level | Notes |
|---------|---------------|-------|
| **SnapLock Volumes** | ❌ **Not Supported** | Native SVMDR limitation - use Pseudo workflow |
| SnapLock Compliance | ❌ Not Supported | Cannot be included in SVMDR relationships |
| SnapLock Enterprise | ❌ Not Supported | Cannot be included in SVMDR relationships |
| FlexGroup Volumes | ⚠️ Limited | Check ONTAP version compatibility |
| SAN (iSCSI/FC) | ⚠️ Requires Config | Additional igroup and LUN mapping setup |
| NAS (NFS/CIFS) | ✅ Full Support | Complete support with identity preservation |

> **Critical:** If your environment includes **any** SnapLock volumes (Enterprise or Compliance), you **must** use the **SVMDR Pseudo workflow** instead. Native SVMDR will fail if SnapLock volumes are present.

### Execution Example

```bash
# Failover to DR site
ansible-playbook svmdr_standard.yml -e "
  input_prod_cluster=prod-cluster.example.com 
  input_prod_svm=svm_production 
  input_username=admin 
  input_password=SecurePassword123 
  input_operation=failover
  input_dryrun=False"

# Failback to production
ansible-playbook svmdr_standard.yml -e "
  input_prod_cluster=prod-cluster.example.com 
  input_prod_svm=svm_production 
  input_username=admin 
  input_password=SecurePassword123 
  input_operation=failback
  input_dryrun=False"

# Always test first with dry-run
ansible-playbook svmdr_standard.yml -e "
  input_prod_cluster=prod-cluster.example.com 
  input_prod_svm=svm_production 
  input_username=admin 
  input_password=SecurePassword123 
  input_operation=failover
  input_dryrun=True"
```

### When to Use SVMDR Standard

**✅ Use SVMDR Standard when:**
- You have no SnapLock volumes
- You need the simplest DR solution
- Identity preservation is critical
- You want minimal management overhead
- Native NetApp features are sufficient

**❌ Do not use SVMDR Standard when:**
- Any SnapLock volumes exist (Enterprise or Compliance)
- You need granular per-volume failover control
- You require features not yet in native SVMDR
- You need to test individual volume failover

---

## SVMDR Pseudo (Custom) Workflow

### Overview

The **SVMDR Pseudo** workflow (also called "Custom SVMDR") provides DR capabilities for environments with **SnapLock volumes** or when **granular control** over individual volumes is required. It uses SVM cloning combined with individual SnapMirror relationships to achieve SVMDR-like functionality with greater flexibility.

### Why Pseudo-SVMDR Exists

Native NetApp SVMDR has a fundamental limitation: **it does not support SnapLock volumes**. Many regulated industries (finance, healthcare, legal) require SnapLock for compliance data retention, making standard SVMDR unusable.

**Pseudo-SVMDR solves this by:**
- Cloning SVM configuration instead of using native SVMDR
- Creating individual SnapMirror relationships for each volume
- Supporting SnapLock volumes (both Enterprise and Compliance)
- Providing granular per-volume failover capabilities
- Enabling custom workflows not possible with native SVMDR

### Architecture

```
    PRODUCTION SITE                         DR SITE
    ┌─────────────────┐                    ┌─────────────────┐
    │   Source SVM    │                    │  Cloned SVM     │
    │                 │   SVM Config       │  (Configuration │
    │  ┌──────────┐  │   Cloning          │   Replicated)   │
    │  │ Vol A    │  │   ════════════►    │                 │
    │  │ (RW)     │  │─┐                  │  ┌──────────┐   │
    │  └──────────┘  │ │ Individual SM    │  │ Vol A    │   │
    │  ┌──────────┐  │ └─────────────────►│  │ (DP)     │   │
    │  │ Vol B    │  │─┐                  │  └──────────┘   │
    │  │ (RW)     │  │ │ Individual SM    │  ┌──────────┐   │
    │  └──────────┘  │ └─────────────────►│  │ Vol B    │   │
    │  ┌──────────┐  │─┐                  │  │ (DP)     │   │
    │  │SnapLock  │  │ │ Individual SM    │  └──────────┘   │
    │  │Volume    │  │ │ (Supports SL!)   │  ┌──────────┐   │
    │  │(Compl)   │  │ └─────────────────►│  │SnapLock  │   │
    │  └──────────┘  │                    │  │Volume    │   │
    └─────────────────┘                    │  │(Compl)   │   │
                                           │  └──────────┘   │
                                           └─────────────────┘
    
    Each volume has its own SnapMirror relationship
    SnapLock volumes replicate to SnapLock destinations
```

### Technology Foundation

**SVM Clone + SnapMirror Combination:**
- **svm_clone role**: Replicates SVM configuration (see detailed section below)
- **Individual SnapMirror**: Per-volume relationships for data replication
- **SnapLock Support**: Full support for Enterprise and Compliance volumes
- **Granular Control**: Volumes can be failed over individually or selectively

### Supported Operations

#### create_and_sm

Establishes the complete DR environment from scratch:

1. **Clone SVM Configuration**
   - Reads source SVM configuration via REST API
   - Creates destination SVM with identical configuration
   - Applies configurable subset of features (see svm_clone section)
   
2. **Create Destination Volumes**
   - Creates DP (data protection) volumes on destination
   - Preserves volume types including SnapLock
   - Sets appropriate security styles and export policies
   
3. **Establish SnapMirror Relationships**
   - Creates individual relationship for each volume
   - Supports SnapLock volume replication
   - Configurable SnapMirror policies
   
4. **Initialize Replication**
   - Performs baseline transfer for all volumes
   - Can be run asynchronously for faster completion
   
5. **Configure Network**
   - Creates network interfaces (initially in down state)
   - Prepares CIFS configuration with DR-specific names
   - Sets up auxiliary LIFs for CIFS management

6. **Mount Volumes**
   - Mounts destination volumes after initialization
   - Applies proper junction paths

> **💡 Important:** The `create_and_sm` operation can be **re-executed periodically** to synchronize SVM configuration changes (new export policies, CIFS shares, users, etc.) without affecting established SnapMirror relationships. This keeps the DR site configuration in sync with production.

#### failover

Activates the DR site:

1. **Validation**
   - Verifies SnapMirror relationships are healthy
   - Checks SVM states
   - Confirms network configuration readiness
   
2. **Stop Source SVM**
   - Gracefully stops production SVM
   
3. **Break SnapMirror**
   - Updates and breaks all SnapMirror relationships
   - Converts DP volumes to RW
   
4. **Activate Destination**
   - Starts destination SVM
   - Brings up network interfaces
   
5. **CIFS Server Rename** (optional, see CIFS section)
   - Renames DR CIFS server to production name
   - Enables seamless client reconnection

#### failback

Returns to production site:

1. **Create Reverse Relationships**
   - Establishes SnapMirror from DR back to production
   - Reverse replication for all volumes
   
2. **Reverse Resync**
   - Synchronizes changed data back to production
   - Preserves SnapLock compliance data
   
3. **Stop DR SVM**
   - Stops currently active DR SVM
   
4. **Break Reverse**
   - Breaks reverse SnapMirror relationships
   
5. **Activate Production**
   - Restarts original production SVM
   - Brings network interfaces back online
   
6. **Re-establish Forward**
   - Recreates forward SnapMirror relationships
   - Returns to normal replication state
   
7. **CIFS Server Rename** (optional)
   - Restores original CIFS server names

#### cleanup

Removes reverse SnapMirror relationships created during failback without affecting the SVM or forward relationships.

#### remove_dest

Complete teardown of DR environment:

1. **Delete SnapMirror Relationships** - Removes all volume relationships
2. **Remove Vserver Peering** (optional) - Can be preserved with `safe_remove_dest: true`
3. **Delete Destination SVM** - Complete removal or safe cleanup preserving auxiliary LIFs

### Granular Volume Failover

One of the key advantages of Pseudo-SVMDR is the ability to **failover volumes individually or selectively** based on criteria:

**Example: Conditional Failover by Volume Name Pattern**
```yaml
# Failover only volumes matching a pattern
volumes_to_failover: "{{ vars_local.volumes | selectattr('name', 'match', '^prod_.*') | list }}"
```

**Example: Failover by SnapLock Type**
```yaml
# Failover only SnapLock Compliance volumes
volumes_to_failover: "{{ vars_local.volumes | selectattr('snaplock.type', 'equalto', 'compliance') | list }}"
```

**Example: Exclude Specific Volumes**
```yaml
# Failover all except test volumes
volumes_to_failover: "{{ vars_local.volumes | rejectattr('name', 'match', '.*_test$') | list }}"
```

This granularity enables:
- **Phased Migrations**: Move applications one at a time
- **Partial DR**: Only critical volumes during limited DR scenarios
- **Testing**: Validate specific volume failover without affecting others
- **Compliance Isolation**: Separate handling of SnapLock vs regular volumes

### Execution Examples

```bash
# Create Pseudo-SVMDR environment with SnapLock support
ansible-playbook svmdr_snaplock.yml -e "
  input_username=admin 
  input_password=SecurePassword123 
  input_prod_cluster=prod-cluster.example.com 
  input_dr_cluster=dr-cluster.example.com 
  input_prod_svm=svm_production 
  input_dr_svm=svm_dr 
  input_prod_cifs_name=PRODCIFS 
  input_dr_cifs_name=DRCIFS 
  input_operation=create_and_sm 
  input_dryrun=False
  input_ipspace=Default"

# Perform failover with automatic CIFS rename
ansible-playbook svmdr_snaplock.yml -e "
  input_username=admin 
  input_password=SecurePassword123 
  input_prod_cluster=prod-cluster.example.com 
  input_dr_cluster=dr-cluster.example.com 
  input_prod_svm=svm_production 
  input_dr_svm=svm_dr 
  input_prod_cifs_name=PRODCIFS 
  input_dr_cifs_name=DRCIFS 
  input_operation=failover
  input_skip_cifs_rename=False"

# Perform failback
ansible-playbook svmdr_snaplock.yml -e "
  input_username=admin 
  input_password=SecurePassword123 
  input_prod_cluster=prod-cluster.example.com 
  input_dr_cluster=dr-cluster.example.com 
  input_prod_svm=svm_production 
  input_dr_svm=svm_dr 
  input_prod_cifs_name=PRODCIFS 
  input_dr_cifs_name=DRCIFS 
  input_operation=failback"

# Cleanup reverse relationships after failback
ansible-playbook svmdr_snaplock.yml -e "
  input_username=admin 
  input_password=SecurePassword123 
  input_prod_cluster=prod-cluster.example.com 
  input_dr_cluster=dr-cluster.example.com 
  input_prod_svm=svm_production 
  input_dr_svm=svm_dr 
  input_operation=cleanup"

# Remove entire DR environment (safe mode - preserves peering and aux LIFs)
ansible-playbook svmdr_snaplock.yml -e "
  input_username=admin 
  input_password=SecurePassword123 
  input_prod_cluster=prod-cluster.example.com 
  input_dr_cluster=dr-cluster.example.com 
  input_prod_svm=svm_production 
  input_dr_svm=svm_dr 
  input_operation=remove_dest"
```

### When to Use SVMDR Pseudo

**✅ Use SVMDR Pseudo when:**
- You have SnapLock volumes (Enterprise or Compliance)
- You need granular per-volume failover control
- You want to failover subsets of volumes (by pattern, type, etc.)
- You need features not yet supported in native SVMDR
- Configuration sync flexibility is important
- You want to test partial DR scenarios

**❌ Consider Standard SVMDR instead when:**
- No SnapLock volumes exist
- You prefer simpler native SVMDR
- Identity preservation is handled natively
- Minimal management overhead is priority

---

## QuickDR Workflow

### Overview

**QuickDR** is a rapid testing workflow designed for non-disruptive validation of SVM configurations and data accessibility. It uses NetApp FlexClone technology to create **instant, space-efficient clones** of SVMs and volumes on the **same cluster** without requiring SnapMirror replication.

### Technology Foundation

**FlexClone** provides:
- **Instant Clones**: Created in seconds regardless of size
- **Space Efficient**: Only stores changed blocks
- **Zero Impact**: No performance impact on source SVM
- **Writable**: Cloned volumes are fully read-write
- **SnapLock Testing**: Can clone SnapLock volumes as regular RW volumes for testing purposes

### Architecture

```
    PRODUCTION CLUSTER
    ┌─────────────────────────────────────────────────────┐
    │                                                     │
    │  Source SVM              Cloned Test SVM           │
    │  ┌──────────────┐       ┌──────────────┐          │
    │  │ Production   │       │ Test Clone   │          │
    │  │ (Running)    │       │ (Isolated)   │          │
    │  │              │ Clone │              │          │
    │  │ ┌──────────┐ │ ════►│ ┌──────────┐ │          │
    │  │ │ Volumes  │ │       │ │FlexClone │ │          │
    │  │ │          │ │       │ │Volumes   │ │          │
    │  │ └──────────┘ │       │ └──────────┘ │          │
    │  │ Config Data  │       │ Config Copy  │          │
    │  └──────────────┘       └──────────────┘          │
    │                                                     │
    └─────────────────────────────────────────────────────┘
    
    Local cluster - no remote replication required
```

### Use Cases

- **Pre-Migration Testing**: Validate application behavior before actual migration
- **Development/Test Environments**: Provide production-like data instantly
- **Configuration Validation**: Test export policies, CIFS shares, permissions
- **Application Testing**: Connect test applications to cloned environment
- **Training**: Create training environments with production-like data
- **SnapLock Data Testing**: Test access to SnapLock data in writable form

### SnapLock Support in QuickDR

QuickDR provides special handling for SnapLock volumes:

**Clone Modes:**
- `exclude_slc`: Exclude SnapLock volumes from cloning (default for safety)
- `reduce_if_slc`: Clone SnapLock volumes as regular RW volumes (removes SnapLock attributes)

**Use Case:** Test applications that need to access SnapLock data without compliance restrictions, or validate data structure and accessibility.

> ⚠️ **Important:** SnapLock clones in QuickDR are **regular RW volumes** without SnapLock protection. This is intentional for testing purposes.

### Supported Operations

#### create

Creates complete test clone:

1. **Clone SVM Configuration** (using svm_clone role)
2. **Clone Volumes** (FlexClone - instant, space-efficient)
3. **Apply Network Configuration** (LIFs in down state initially)
4. **Configure Services** (CIFS with test-specific name, NFS, etc.)
5. **Start SVM** (ready for testing)

#### clean

Complete removal:
- Deletes all cloned volumes
- Removes cloned SVM
- No impact on source

#### safe_clean

Conservative cleanup:
- Removes cloned configuration
- Preserves SVM object
- Keeps auxiliary network interfaces (pattern: `*AUX*`)
- Useful for repeated testing

### Configuration

QuickDR uses the same svm_clone role with specific options:

```yaml
svm_cloning_opts:
  svm_clone_destination: 'local'      # Same cluster
  clone_volumes:         True         # Enable FlexClone
  cifs:
    name:                "TESTCIFS"   # Different name for testing
  network:
    lif_state:           'down'       # Start with LIFs down
    lif_port:            "a0a"        # Target port for LIFs
    ipspace:             "Default"
  svm:
    admin_state:         'started'    # Start immediately

vol_cloning_opts:
  vol_options:
    type:              'rw'           # Read-write volumes
    space_guarantee:   'none'         # Space efficient
    split:             false          # Keep as clone (space efficient)
    snaplock:
      type:            'non_snaplock' # Clone as regular volumes
      clone_mode:      'exclude_slc'  # Exclude SnapLock or 'reduce_if_slc'
```

### Execution Examples

```bash
# Create QuickDR test environment
ansible-playbook quick_dr.yml -e "
  input_username=admin 
  input_password=SecurePassword123 
  input_prod_cluster=prod-cluster.example.com 
  input_prod_svm=svm_production 
  input_dr_svm=svm_test_clone 
  input_operation=create 
  input_cifs_name=TESTCIFS
  input_ipspace=test_ipspace"

# Clean up test environment completely
ansible-playbook quick_dr.yml -e "
  input_username=admin 
  input_password=SecurePassword123 
  input_prod_cluster=prod-cluster.example.com 
  input_prod_svm=svm_production 
  input_dr_svm=svm_test_clone 
  input_operation=clean"

# Safe cleanup (preserves SVM and auxiliary LIFs)
ansible-playbook quick_dr.yml -e "
  input_username=admin 
  input_password=SecurePassword123 
  input_prod_cluster=prod-cluster.example.com 
  input_prod_svm=svm_production 
  input_dr_svm=svm_test_clone 
  input_operation=safe_clean"
```

### When to Use QuickDR

**✅ Use QuickDR for:**
- Rapid testing without remote replication
- Development and test environments
- Pre-migration validation
- Application compatibility testing
- Training and demonstrations
- Testing access to SnapLock data

**❌ Do not use QuickDR for:**
- Actual disaster recovery (use SVMDR workflows)
- Production failover scenarios
- Remote site DR (requires SnapMirror)
- Long-term data retention

---

## SnapMirror Operations Tracking

### Overview

During failover and failback operations, the automation must carefully orchestrate and monitor multiple SnapMirror relationships to ensure data consistency and successful DR operations. This section explains how the framework tracks SnapMirror operations throughout the DR lifecycle.

### SnapMirror State Management

The automation uses a **state-based tracking system** that continuously monitors SnapMirror relationship status through REST API calls, ensuring each operation completes successfully before proceeding to the next step.

### Key SnapMirror States

| State | Description | Automation Action |
|-------|-------------|-------------------|
| **snapmirrored** | Healthy, idle relationship | Ready for operations |
| **transferring** | Active data transfer in progress | Wait until complete |
| **uninitialized** | Newly created, not yet baselined | Initialize transfer |
| **broken-off** | Relationship broken, destination RW | Expected after failover |
| **paused** | Relationship paused | Resume or investigate |
| **out-of-sync** | Source and destination not in sync | Resync required |

### Tracking During Failover (SVMDR Pseudo)

#### Phase 1: Pre-Failover Validation

```yaml
- name: Validate SnapMirror relationships before failover
  netapp.ontap.na_ontap_rest_info:
    hostname: "{{ dest_cluster }}"
    username: "{{ username }}"
    password: "{{ password }}"
    use_rest: Always
    gather_subset: "snapmirror/relationships"
    parameters:
      destination.svm.name: "{{ dest_svm }}"
      source.svm.name: "{{ source_svm }}"
  register: sm_relationships

- name: Check each relationship health
  assert:
    that:
      - item.state == "snapmirrored"
      - item.healthy == true
      - item.transfer.state != "transferring"
    fail_msg: "Relationship {{ item.uuid }} not ready: state={{ item.state }}"
  loop: "{{ sm_relationships.ontap_info['snapmirror/relationships'].records }}"
  loop_control:
    label: "{{ item.source.path }} -> {{ item.destination.path }}"
```

**What's Tracked:**
- Relationship state (must be `snapmirrored`)
- Health status (must be `true`)
- Transfer state (must not be `transferring`)
- Last transfer errors (must be none)

#### Phase 2: Final Update and Break

The automation performs a final SnapMirror update, waits for completion, then breaks relationships:

```yaml
# 1. Trigger final update for all relationships
- name: Perform final SnapMirror update
  netapp.ontap.na_ontap_snapmirror:
    state: present
    source_path: "{{ item.source.path }}"
    destination_path: "{{ item.destination.path }}"
  loop: "{{ sm_relationships_list }}"

# 2. Monitor until transfer completes
- name: Wait for update completion
  netapp.ontap.na_ontap_rest_info:
    gather_subset: "snapmirror/relationships"
  register: sm_status
  until: sm_status.records[0].transfer.state != 'transferring'
  retries: "{{ sm_state_retries | default(300) }}"
  delay: "{{ sm_state_delay | default(10) }}"

# 3. Break relationships
- name: Break SnapMirror relationships
  netapp.ontap.na_ontap_snapmirror:
    state: broken
    source_path: "{{ item.source.path }}"
    destination_path: "{{ item.destination.path }}"
  loop: "{{ sm_relationships_list }}"

# 4. Verify state is 'broken-off'
- name: Verify break completed
  netapp.ontap.na_ontap_rest_info:
    gather_subset: "snapmirror/relationships"
  register: sm_verify
  until: sm_verify.records[0].state == 'broken-off'
```

**Key Tracking Points:**
- Poll transfer state every 10 seconds (configurable via `sm_state_delay`)
- Default timeout: 3000 seconds / 50 minutes (configurable via `sm_state_retries`)
- Verify final state is `broken-off` with destination volumes converted to RW type

### Tracking During Failback (SVMDR Pseudo)

#### Phase 1: Reverse SnapMirror Creation

During failback, reverse relationships are created from DR back to production:

```yaml
# 1. Create reverse SnapMirror relationships
- name: Create reverse relationships
  netapp.ontap.na_ontap_snapmirror:
    state: present
    source_path: "{{ dest_svm }}:{{ item.volume }}"
    destination_path: "{{ source_svm }}:{{ item.volume }}"
    policy: "{{ sm_policy | default('DPDefault') }}"
    relationship_type: "extended_data_protection"
  loop: "{{ volumes_list }}"

# 2. Monitor initialization
- name: Track reverse relationship initialization
  netapp.ontap.na_ontap_rest_info:
    gather_subset: "snapmirror/relationships"
  register: reverse_sm_status
  until:
    - reverse_sm_status.records | length > 0
    - reverse_sm_status.records[0].state in ['snapmirrored', 'transferring']
  retries: 60
  delay: 5
```

**Key Points:**
- Creates reverse relationships for all volumes
- Monitors initialization and baseline transfer start
- Tracks state transitions from `uninitialized` to `snapmirrored`

#### Phase 2: Resync Operation

Resync synchronizes changed data from DR back to production:

```yaml
# 1. Trigger resync
- name: Perform resync
  netapp.ontap.na_ontap_snapmirror:
    state: present
    source_path: "{{ dest_svm }}:{{ item.volume }}"
    destination_path: "{{ source_svm }}:{{ item.volume }}"
  loop: "{{ volumes_list }}"

# 2. Monitor completion
- name: Monitor resync progress
  netapp.ontap.na_ontap_rest_info:
    gather_subset: "snapmirror/relationships"
  register: resync_status
  until:
    - resync_status.records[0].transfer.state is not defined
    - resync_status.records[0].state == 'snapmirrored'
  retries: "{{ sm_state_retries | default(300) }}"
  delay: "{{ sm_state_delay | default(10) }}"
```

**Key Points:**
- Monitors transfer state, bytes transferred, and duration
- Comprehensive error detection and reporting

### Tracking During Standard SVMDR Operations

For **Standard SVMDR**, the automation tracks the SVM-level relationship:

```yaml
# 1. Get SVMDR relationship
- name: Get SVMDR relationship status
  netapp.ontap.na_ontap_rest_info:
    gather_subset: "snapmirror/relationships"
    parameters:
      destination.svm.name: "{{ dest_svm }}"
      relationship_type: "svm_dr"
  register: svmdr_relationship

# 2. Update and monitor
- name: Update SVMDR relationship
  netapp.ontap.na_ontap_snapmirror:
    state: present
    source_vserver: "{{ source_svm }}"
    destination_vserver: "{{ dest_svm }}"

- name: Wait for update completion
  netapp.ontap.na_ontap_rest_info:
    gather_subset: "snapmirror/relationships"
  register: svmdr_status
  until:
    - svmdr_status.records[0].transfer.state != 'transferring'
    - svmdr_status.records[0].healthy == true
  retries: 300
  delay: 10

# 3. Break relationship
- name: Break SVMDR relationship
  netapp.ontap.na_ontap_snapmirror:
    state: broken
    source_vserver: "{{ source_svm }}"
    destination_vserver: "{{ dest_svm }}"
```

### Retry and Timeout Configuration

The tracking system uses **configurable retry and timeout values**:

```yaml
# defaults_svmdr_sl.yml
sm_state_retries: 300        # Maximum retry attempts (default 300)
sm_state_delay: 10           # Seconds between checks (default 10)
                             # Total timeout: 300 × 10 = 3000s (50 minutes)

# For slower WAN links
sm_state_retries: 600        # 100 minutes total
sm_state_delay: 60           # Check every minute

# For fast local links
sm_state_retries: 60         # 10 minutes total
sm_state_delay: 10           # Check every 10 seconds
```

### Error Handling and Recovery

The automation includes comprehensive error handling:

```yaml
- name: SnapMirror operation with error handling
  block:
    - name: Perform SnapMirror break
      netapp.ontap.na_ontap_snapmirror:
        hostname: "{{ dest_cluster }}"
        use_rest: Always
        state: broken
        source_path: "{{ item.source.path }}"
        destination_path: "{{ item.destination.path }}"
      register: sm_break
      failed_when: false  # Don't fail immediately
    
    - name: Check for errors
      debug:
        msg: "Warning: SnapMirror break failed for {{ item.destination.path }}: {{ sm_break.msg }}"
      when: sm_break.failed
    
    - name: Attempt recovery if needed
      block:
        - name: Check if relationship already broken
          netapp.ontap.na_ontap_rest_info:
            hostname: "{{ dest_cluster }}"
            use_rest: Always
            gather_subset: "snapmirror/relationships"
            parameters:
              destination.path: "{{ item.destination.path }}"
          register: sm_check
        
        - name: Continue if already in desired state
          debug:
            msg: "Relationship already in broken-off state"
          when: 
            - sm_check.ontap_info['snapmirror/relationships'].records[0].state == 'broken-off'
      when: sm_break.failed
  rescue:
    - name: Log critical error and stop
      fail:
        msg: "Critical error breaking SnapMirror for {{ item.destination.path }}"
```

### Tracking Asynchronous Operations

For **large environments with many volumes**, the framework supports **asynchronous tracking**:

```yaml
- name: Break SnapMirror relationships asynchronously
  netapp.ontap.na_ontap_snapmirror:
    hostname: "{{ dest_cluster }}"
    use_rest: Always
    state: broken
    source_path: "{{ item.source.path }}"
    destination_path: "{{ item.destination.path }}"
  loop: "{{ sm_relationships_list }}"
  async: 7200              # 2 hour timeout per operation
  poll: 0                  # Don't wait, return immediately
  register: async_break_jobs

- name: Track async job completion
  async_status:
    jid: "{{ item.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 720             # Check for up to 2 hours
  delay: 10
  loop: "{{ async_break_jobs.results }}"
  loop_control:
    label: "Job {{ item.ansible_job_id }}"

- name: Verify all operations succeeded
  assert:
    that:
      - item.rc == 0
    fail_msg: "Async operation failed: {{ item.stderr | default('Unknown error') }}"
  loop: "{{ job_result.results }}"
```

### Output Control and Logging

#### Console Output Suppression

The `nolog` parameter controls sensitive output visibility in Ansible console logs:

```yaml
# defaults_svmdr_sl.yml
nolog: false   # Show all output (default for debugging)
nolog: true    # Suppress sensitive output (credentials, auth details)
```

**What is suppressed when `nolog: true`:**
- Cluster authentication credentials (username, password)
- Connection details (hostname, validate_certs, use_rest flags)
- AD administrator credentials (for CIFS rename operations)

**Usage in playbook tasks:**
```yaml
- name: Set connection details
  ansible.builtin.set_fact:
    src_auth:
      username: "{{ admin_user }}"
      password: "{{ admin_pass }}"
      hostname: "{{ cluster_mgmt }}"
  no_log: "{{ nolog | default(true) }}"
```

#### File-Based Logging

The framework includes a custom `do_log` filter plugin for writing detailed operation logs to files:

**Filter Plugin:** `netapp_ps.maf.plugins.filter.log.logToFile()`

**Log File Creation:**
```yaml
- name: Log operation details to file
  set_fact:
    log_result: "{{ operation_data | to_nice_yaml | do_log(
      title='SnapMirror Failover Operation',
      name='Relationship Status',
      logdir='/var/log/ansible',
      logname='svmdr_failover'
    ) }}"
```

**Log File Naming Convention:**
- Without timestamp: `/var/log/ansible/svmdr_failover.log` (append mode)
- With timestamp: `/var/log/ansible/20260126143022_svmdr_failover.log` (when logname doesn't end with .log)

**What Gets Logged:**
- Operation titles with underlined headers
- Section names for different phases
- Full YAML-formatted data structures (SnapMirror states, SVM configurations, volume details)
- Appended to existing log file for continuous operation tracking

**Example Log Output:**
```
SnapMirror Failover Operation
------------------------------
Relationship Status:
state: snapmirrored
healthy: true
source_path: svm_prod:vol_data
destination_path: svm_dr:vol_data
```

#### Best Practices

1. **Development/Testing:**
   - Set `nolog: false` for full visibility
   - Enable verbose Ansible output (`-vvv`)
   
2. **Production:**
   - Set `nolog: true` to protect sensitive data
   - Use file-based logging for audit trails
   
3. **Troubleshooting:**
   - Temporarily set `nolog: false` for specific tasks
   - Review timestamped log files for historical operations

### Troubleshooting SnapMirror Operations

| Issue | Symptoms | Resolution |
|-------|----------|------------|
| **Stuck Transfer** | `transfer.state = 'transferring'` for extended time | Check network, storage, CPU; may need to abort and retry |
| **Relationship Unhealthy** | `healthy = false` | Check error messages; verify source and destination accessibility |
| **Break Fails** | Cannot break relationship | Verify no active transfers; check for source accessibility issues |
| **Resync Timeout** | Reverse resync takes too long | Check available bandwidth; consider increasing timeout |
| **Initialization Fails** | New relationships won't initialize | Verify peering, network connectivity, and sufficient space |

---

## The svm_clone Role Deep Dive

### Overview

The **svm_clone** role is the core technology behind both **Pseudo-SVMDR** and **QuickDR** workflows. It provides comprehensive SVM configuration cloning capabilities, reading configuration from a source SVM via REST API and recreating it on a destination (either remote cluster or local).

### Architecture and Design

**Modular Configuration Cloning:**

The role operates on a **dependency-based level system** where configuration items are applied in order based on their dependencies:

```
Level 0 (No Dependencies)
├── dns, route, export_policy
├── name_mapping, job_schedule  
├── snapmirror_policy, lif
└── iscsi

Level 1 (Basic Dependencies)
└── snapshot_policy

Level 5 (Volume Dependencies)
└── volume (if cloning)

Level 10 (User/Group Dependencies)
├── cifs_local_group, unix_group
├── nfs
└── cifs

Level 15 (User Dependencies)
├── cifs_local_user, unix_user
├── fpolicy
└── cifs_privilege

Level 100 (Volume-dependent)
├── cifs_share (depends on volumes and CIFS)
├── quota_rule (depends on volumes)
├── qtree
└── audit (depends on cloned volumes)
```

### Comprehensive Configuration Items

The role can clone the following SVM configuration elements:

| Category | Configuration Items | Notes |
|----------|---------------------|-------|
| **Network** | DNS, Routes, LIFs, IPspaces | Can specify target ports and IP spaces |
| **NFS** | NFS service config, Export policies, Export rules | Full NFS configuration |
| **CIFS** | CIFS server, Shares, Local users, Local groups, Privileges | AD-joined CIFS servers supported |
| **Users/Groups** | Unix users, Unix groups, Name mappings | Complete identity management |
| **Storage** | Volumes (cloning or DP), Qtrees, Quotas | FlexClone for local, DP for remote |
| **Policies** | Snapshot policies, SnapMirror policies | Policy definitions and schedules |
| **Security** | FPolicy engines/events/policies, Audit config | Compliance and security features |
| **Scheduling** | Job schedules | Cron-based schedules |
| **Protocols** | iSCSI service configuration | SAN protocol support |

### How svm_clone Works

#### 1. Configuration Reading (REST API)

The role uses **ONTAP REST API exclusively** to read source SVM configuration. No ONTAPI (ZAPI) calls are used, ensuring long-term supportability:

```yaml
# Example: Reading SVM info via REST API
netapp.ontap.na_ontap_rest_info:
  hostname: "{{ source_cluster }}"
  username: "{{ admin_user }}"
  password: "{{ admin_pass }}"
  use_rest: Always          # REST API only - no ONTAPI fallback
  gather_subset: "svm/svms"
  fields: "name,uuid,subtype,nfs.enabled,cifs.enabled,..."
  parameters:
    name: "{{ source_svm_name }}"
```

**Subset-based Configuration Retrieval:**

Each configuration item (e.g., `export_policy`, `cifs_share`) has a defined REST API path and fields:

```yaml
config_api_paths:
  export_policy:
    path: "protocols/nfs/export-policies"
    fields:
      - name
      - rules
  cifs_share:
    path: "protocols/cifs/shares"
    fields:
      - name
      - path
      - volume.name
      - acls
      - browsable
      - oplocks
      # ... 20+ more fields
```

#### 2. Configuration Transformation

After reading, the role:

1. **Removes non-transferable data**: UUIDs, internal links, read-only fields
2. **Applies transformations**: Adapts network settings, volume types, CIFS names
3. **Resolves dependencies**: Ensures prerequisite objects exist
4. **Validates compatibility**: Checks for supported features

```yaml
# Example: Transform volume for DP destination
volume_config:
  name: "{{ source_volume.name }}"
  type: "DP"  # Changed from RW
  size: "{{ source_volume.size }}"
  export_policy: "{{ source_volume.export_policy.name }}"
  # Remove: uuid, snaplock (if not supported on destination)
```

#### 3. Configuration Application

The role applies configuration in **dependency-ordered levels**:

**Level 0 - No Dependencies:**
```yaml
- name: Apply level 0 configs
  include_role:
    name: "netapp_ps.ontap.{{ config_item }}"
  vars:
    qtask: create
  loop:
    - dns
    - route
    - export_policy
```

**Level 10 - Service Dependencies:**
```yaml
- name: Apply CIFS (depends on network and AD)
  include_role:
    name: netapp_ps.ontap.cifs
  vars:
    qtask: create
    cifs_name: "{{ dr_cifs_name }}"  # Different name for DR
```

**Level 100 - Volume Dependencies:**
```yaml
- name: Apply CIFS shares (depends on volumes and CIFS server)
  include_role:
    name: netapp_ps.ontap.cifs_share
  vars:
    qtask: create_multi
  when: 
    - volumes_cloned or volumes_created
    - cifs_enabled
```

#### 4. Volume Cloning (Local) or Creation (Remote)

**For Local Clones (QuickDR):**
```yaml
clone_volumes: True
vol_clone_options:
  vol_options:
    type: 'rw'              # Read-write FlexClone
    space_guarantee: 'none'  # Space efficient
    split: false            # Keep as clone (not split)
    snaplock:
      type: 'non_snaplock'  # Remove SnapLock attributes
      clone_mode: 'exclude_slc'  # or 'reduce_if_slc'
```

**For Remote DP Volumes (Pseudo-SVMDR):**
```yaml
clone_volumes: False  # Don't use FlexClone
# Volumes created separately as type DP
# SnapMirror relationships established afterward
```

### Configuration Subset Customization

You can control exactly what gets cloned by specifying the `svm_clone_config` list:

**Minimal Configuration (Network + NFS only):**
```yaml
svm_clone_config:
  - dns
  - route
  - export_policy
  - nfs
  - lif
```

**Full Configuration (Everything):**
```yaml
svm_clone_config:
  - dns
  - route
  - export_policy
  - nfs
  - name_mapping
  - unix_group
  - unix_user
  - snapshot_policy
  - snapmirror_policy
  - job_schedule
  - quota_rule
  - lif
  - fpolicy
  - audit
  - cifs
  - cifs_share
  - cifs_local_user
  - cifs_local_group
  - cifs_privilege
  - qtree
```

**CIFS-Only Configuration:**
```yaml
svm_clone_config:
  - dns
  - route
  - name_mapping
  - unix_group
  - unix_user
  - cifs
  - cifs_share
  - cifs_local_user
  - cifs_local_group
  - cifs_privilege
  - lif
```

### Extending svm_clone

The modular design allows **easy extension** to support new configuration items or match evolving native SVMDR features:

**Adding a New Configuration Item:**

1. **Define REST API path and fields** in `defaults/main.yml`:
```yaml
config_api_paths:
  new_feature:
    path: "protocols/new-feature"
    fields:
      - name
      - enabled
      - custom_setting
```

2. **Add to dependency level** in `defaults/main.yml`:
```yaml
multi_subset_items:
  10:  # Appropriate dependency level
    - new_feature
```

3. **Create Ansible role** for the feature:
```yaml
# roles/new_feature/tasks/create.yml
- name: Create new feature config
  netapp.ontap.na_ontap_rest_cli:
    command: 'new-feature/create'
    params:
      svm: "{{ dest_svm_name }}"
      config: "{{ item }}"
  loop: "{{ vars_local.new_feature }}"
```

4. **Add to clone_config list** when needed:
```yaml
svm_clone_config:
  - dns
  - route
  - new_feature  # Now available
```

This extensibility means that as NetApp adds features to ONTAP or you need custom configuration items, you can **easily extend the framework** without rewriting core logic.

### svm_clone Operations

The role supports multiple operations through the `qtask` parameter:

| Operation | Description | Use Case |
|-----------|-------------|----------|
| `create` | Full SVM configuration cloning | Initial DR setup, QuickDR creation |
| `create_finals` | Apply level 100 dependencies | CIFS shares, auditing after volumes exist |
| `state` | Change SVM state (started/stopped) | Control SVM availability |
| `delete` | Remove cloned SVM and config | Cleanup operations |
| `readonly` | Read configuration without changes | Validation, reporting |

### Network Configuration Options

The role provides flexible network configuration:

```yaml
svm_clone_options:
  network:
    lif_state: 'down'        # Start LIFs in down state
    lif_port: 'e0d'          # Target port for LIFs
    ipspace: 'DR_IPspace'    # Different IPspace
  svm:
    admin_state: 'stopped'   # Start SVM in stopped state
```

**Common Patterns:**

**Pattern 1: DR Site with Different Network**
```yaml
network:
  lif_state: 'down'          # Start down, bring up after failover
  lif_port: 'a0a-200'        # VLAN for DR network
  ipspace: 'DR_IPspace'      # Isolated IPspace
```

**Pattern 2: Test Environment on Same Network**
```yaml
network:
  lif_state: 'down'          # Prevent IP conflicts
  lif_port: 'a0a-100'        # Test VLAN
  ipspace: 'Test_IPspace'    # Isolated test network
```

---

## Customization Options

### Overview

The automation framework is designed for extensive customization to meet diverse customer requirements. Every aspect from which configuration items are cloned to how failover behaves can be tailored.

### Inventory Customization

Inventory files define the topology and parameters for each workflow:

**Structure:**
```
vars/
└── customer/
    └── <customer_name>/
        ├── defaults.yml          # Common settings
        ├── defaults_svmdr_sl.yml # Pseudo-SVMDR settings
        └── inventory.yml         # Cluster and SVM definitions
```

**Example Inventory Definition:**
```yaml
# inventory.yml
ontap_clusters:
  production:
    hostname: "prod-cluster.example.com"
    username: "admin"
    password: "{{ vault_prod_password }}"
  
  dr:
    hostname: "dr-cluster.example.com"
    username: "admin"
    password: "{{ vault_dr_password }}"

svms:
  production_svm:
    name: "svm_prod"
    cluster: "production"
    cifs_server: "PRODCIFS"
    volumes:
      - name: "vol_data"
        snaplock_type: "enterprise"
      - name: "vol_logs"
        snaplock_type: "non_snaplock"
  
  dr_svm:
    name: "svm_dr"
    cluster: "dr"
    cifs_server: "DRCIFS"
```

### Clone Scope Customization

Control which configuration elements are cloned:

**Scenario 1: NFS-Only Environment**
```yaml
svm_clone_config:
  - dns
  - route
  - export_policy
  - nfs
  - lif
  - snapshot_policy
  - unix_user
  - unix_group
  # Omit all CIFS-related items
```

**Scenario 2: CIFS-Only Environment**
```yaml
svm_clone_config:
  - dns
  - route
  - name_mapping
  - cifs
  - cifs_share
  - cifs_local_user
  - cifs_local_group
  - cifs_privilege
  - lif
  # Omit NFS-related items
```

**Scenario 3: Minimal Configuration (Network + Protocol Only)**
```yaml
svm_clone_config:
  - dns
  - route
  - nfs
  - lif
  # Everything else managed separately
```

### Network Customization

Adapt network configuration for different environments:

**Scenario 1: Different IPspace for DR**
```yaml
svm_clone_options:
  network:
    ipspace: "DR_IPspace"
    lif_port: "a0a-200"        # VLAN 200 for DR
    lif_state: "down"          # Start down
```

**Scenario 2: Preserve Some LIFs, Modify Others**
```yaml
# Custom logic in playbook
lifs_to_create: "{{ all_lifs | rejectattr('name', 'match', '.*_mgmt$') | list }}"
# Only create data LIFs, exclude management LIFs
```

**Scenario 3: Auxiliary LIFs for CIFS Management**
```yaml
# Pre-create auxiliary LIFs with naming pattern
# These are preserved during cleanup
lif_name_pattern: "*AUX*"
# Used for AD communication during CIFS rename operations
```

### SnapMirror Customization

Control SnapMirror behavior:

```yaml
# defaults_svmdr_sl.yml
sm_policy_name: "DPDefault"          # Or custom policy
sm_state_retries: 300                # Max retries for SM operations
sm_state_delay: 10                   # Seconds between status checks
sm_schedule: "hourly"                # Transfer schedule

# Custom: Different policies for different volumes
sm_policies_by_volume:
  snaplock_volumes: "SnapLockPolicy"  # Longer retention
  standard_volumes: "DPDefault"       # Standard policy
```

### Timing and Retry Customization

Adjust for different performance environments:

```yaml
# Slow WAN link - longer timeouts
sm_state_retries: 600      # 10 minutes
sm_state_delay: 60         # Check every minute

# Fast local network - shorter timeouts
sm_state_retries: 60       # 10 minutes
sm_state_delay: 10         # Check every 10 seconds
```

### CIFS Behavior Customization

Control CIFS server management:

```yaml
# Automatic CIFS rename (default)
input_skip_cifs_rename: False
# Playbook handles rename via auxiliary LIFs

# Manual CIFS management
input_skip_cifs_rename: True
# You handle CIFS rename/AD rejoin separately
```

### Cleanup Behavior Customization

Control what happens during removal:

```yaml
# Safe removal - preserve peering and auxiliary LIFs
safe_remove_dest: True
keep_lifs: ['*AUX*']

# Complete removal - delete everything
safe_remove_dest: False
```

### Volume Selection Customization (Pseudo-SVMDR)

**Example 1: Failover Only Specific Volume Types**
```yaml
- name: Filter volumes by SnapLock type
  set_fact:
    filtered_volumes: "{{ vars_local.volumes | selectattr('snaplock.type', 'equalto', 'compliance') | list }}"

- name: Failover only compliance volumes
  include_role:
    name: netapp_ps.ontap.snapmirror
  vars:
    volumes: "{{ filtered_volumes }}"
    qtask: break_multi
```

**Example 2: Exclude Volumes by Pattern**
```yaml
- name: Exclude test volumes from failover
  set_fact:
    production_volumes: "{{ vars_local.volumes | rejectattr('name', 'match', '.*_test$') | list }}"
```

**Example 3: Failover by Aggregate Location**
```yaml
- name: Failover only volumes on specific aggregate
  set_fact:
    aggr_volumes: "{{ vars_local.volumes | selectattr('aggregate.name', 'equalto', 'aggr_ssd_01') | list }}"
```

### Pre-flight Check Customization

Add or modify validation steps:

```yaml
# Custom pre-flight check in playbook
- name: Custom validation
  block:
    - name: Check available space on destination
      netapp.ontap.na_ontap_rest_info:
        hostname: "{{ dr_cluster }}"
        gather_subset: "storage/aggregates"
      register: aggr_info
    
    - name: Validate sufficient space
      assert:
        that:
          - aggr_info.ontap_info['storage/aggregates'].records[0].space.available > required_space
        fail_msg: "Insufficient space on destination aggregate"
```

### Integration Customization

Adapt for different automation platforms:

**AWX/Ansible Tower:**
```yaml
# Use Tower credentials
username: "{{ tower_username }}"
password: "{{ tower_password }}"

# Use Tower surveys for input
input_operation: "{{ tower_operation }}"
input_dryrun: "{{ tower_dryrun | default(True) }}"
```

**CI/CD Pipeline:**
```yaml
# Environment variable-based configuration
username: "{{ lookup('env', 'ONTAP_USERNAME') }}"
password: "{{ lookup('env', 'ONTAP_PASSWORD') }}"

# Pipeline stage control
input_operation: "{{ pipeline_stage }}"  # from pipeline variable
```

**Custom API Integration:**
```yaml
# Receive parameters from custom API
- name: Read parameters from API
  uri:
    url: "https://api.company.com/dr-params"
    return_content: yes
  register: api_params

- name: Use API parameters
  set_fact:
    input_prod_svm: "{{ api_params.json.source_svm }}"
    input_dr_svm: "{{ api_params.json.dest_svm }}"
```

---

## CIFS Server Management

### Overview

CIFS server management in DR scenarios is complex due to Active Directory domain integration. The automation provides **automated CIFS server renaming** to enable seamless client reconnection during failover and failback.

### The CIFS DR Challenge

**Problem:** When failing over to a DR site, the DR CIFS server has a **different name** than production:
- Production: `PRODCIFS`
- DR: `DRCIFS`

Clients trying to access `\\PRODCIFS\share` will fail because that name now points to the wrong server.

**Solution:** Automatically **rename the DR CIFS server** to the production name during failover.

### Auxiliary LIF Architecture

CIFS server rename operations require Active Directory communication, but during failover the SVM's primary LIFs may be in transition. **Auxiliary LIFs** solve this:

```
    PRODUCTION SVM                         DR SVM
    ┌─────────────────┐                    ┌─────────────────┐
    │ PRODCIFS        │                    │ DRCIFS          │
    │                 │                    │                 │
    │ Data LIFs:      │                    │ Data LIFs:      │
    │ ├─ 10.1.1.10    │                    │ ├─ 10.2.1.10    │
    │ ├─ 10.1.1.11    │                    │ ├─ 10.2.1.11    │
    │ └─ 10.1.1.12    │                    │ └─ 10.2.1.12    │
    │                 │                    │                 │
    │ Auxiliary LIFs: │                    │ Auxiliary LIFs: │
    │ └─ PROD_AUX ───────► AD Domain ◄──────── DR_AUX        │
    │    10.1.1.99    │                    │    10.2.1.99    │
    └─────────────────┘                    └─────────────────┘
    
    Auxiliary LIFs provide stable AD connectivity
    during CIFS server rename operations
```

### Prerequisites

1. **Different CIFS Server Names**
   - Production and DR must have unique CIFS server names
   - Both joined to the same AD domain

2. **Auxiliary LIFs Pre-configured**
   - Must exist on both source and DR SVMs
   - Naming pattern: `*AUX*` (e.g., `PROD_AUX`, `DR_AUX`)
   - Must have connectivity to AD domain controllers
   - Preserved during all operations (not deleted during cleanup)

3. **AD Credentials**
   - Automation requires AD admin credentials for rename operations
   - Credentials stored securely (Ansible vault recommended)

### Automated CIFS Rename Workflow

#### During Failover (Production → DR)

```
Before Failover:
  Production: PRODCIFS (active)  │  DR: DRCIFS (standby)
  Clients access: \\PRODCIFS     │  

Failover Process:
  1. Stop production SVM         │  PRODCIFS offline
  2. Break SnapMirror            │  
  3. Start DR SVM                │  DRCIFS online (wrong name!)
  4. Rename via Auxiliary LIF ──►│  DRCIFS → PRODCIFS
  5. Bring up data LIFs          │  

After Failover:
  Production: PRODCIFS (stopped) │  DR: PRODCIFS (active)
  Clients access: \\PRODCIFS ────────► Now reaches DR site!
```

#### During Failback (DR → Production)

```
Before Failback:
  Production: PRODCIFS (stopped) │  DR: PRODCIFS (active)
  Clients access: \\PRODCIFS ────────► DR site

Failback Process:
  1. Create reverse SnapMirror   │  
  2. Resync data                 │  
  3. Stop DR SVM                 │  PRODCIFS offline
  4. Rename via Auxiliary LIF ──►│  PRODCIFS → DRCIFS
  5. Start production SVM        │  PRODCIFS online
  6. Rename via Auxiliary LIF ──►│  (if needed)

After Failback:
  Production: PRODCIFS (active)  │  DR: DRCIFS (standby)
  Clients access: \\PRODCIFS ────────► Back to production!
```

### Configuration Options

**Automatic Rename (Default):**
```yaml
input_skip_cifs_rename: False
input_prod_cifs_name: "PRODCIFS"
input_dr_cifs_name: "DRCIFS"
```

The playbook automatically:
1. Identifies auxiliary LIFs (matching `*AUX*` pattern)
2. Performs AD-based CIFS server rename
3. Handles rejoin to AD domain
4. Verifies rename completion

**Manual Rename (Skip Automation):**
```yaml
input_skip_cifs_rename: True
```

Use when:
- Custom AD organizational units require special handling
- Manual AD administrator intervention is required
- Complex AD trust relationships exist
- Testing rename procedures separately

### Troubleshooting CIFS Rename

**Common Issues:**

1. **AD Communication Failure**
   - **Symptom**: Rename operation times out or fails
   - **Cause**: Auxiliary LIF cannot reach domain controllers
   - **Solution**: Verify auxiliary LIF routing and DNS resolution

2. **Authentication Failure**
   - **Symptom**: "Access denied" during rename
   - **Cause**: Insufficient AD privileges
   - **Solution**: Ensure AD credentials have computer object rename rights

3. **Duplicate Computer Object**
   - **Symptom**: Rename fails with "object already exists"
   - **Cause**: Previous cleanup incomplete in AD
   - **Solution**: Remove stale computer object from AD manually

4. **DNS Update Delay**
   - **Symptom**: Clients cannot resolve CIFS server name after rename
   - **Cause**: DNS replication delay
   - **Solution**: Wait for DNS propagation or force DNS update

**Manual CIFS Rename Procedure (When Skipping Automation):**

```bash
# On ONTAP CLI - failover rename
vserver cifs modify -vserver svm_dr -cifs-server PRODCIFS

# On ONTAP CLI - failback rename
vserver cifs modify -vserver svm_dr -cifs-server DRCIFS
vserver cifs modify -vserver svm_production -cifs-server PRODCIFS
```

---

## Pre-flight Validation

### Overview

All workflows include comprehensive pre-flight validation to catch configuration issues before execution. This **"fail fast"** approach prevents partial failures and provides clear error messages.

### Validation Categories

#### 1. Connectivity Validation

Verifies network reachability to all involved clusters:

```yaml
- name: Test production cluster connectivity
  netapp.ontap.na_ontap_rest_info:
    hostname: "{{ input_prod_cluster }}"
    username: "{{ input_username }}"
    password: "{{ input_password }}"
    gather_subset: "cluster/nodes"
  register: connectivity_test
  failed_when: connectivity_test.failed
```

**Validates:**
- Cluster hostname resolution
- Network path to management LIF
- HTTPS connectivity
- TLS certificate handling

#### 2. Credential Validation

Tests authentication before starting operations:

```yaml
- name: Validate credentials
  netapp.ontap.na_ontap_rest_info:
    hostname: "{{ input_prod_cluster }}"
    username: "{{ input_username }}"
    password: "{{ input_password }}"
    gather_subset: "cluster/software"
  register: auth_test
  failed_when: 
    - auth_test.failed
    - "'401' in auth_test.msg or 'Unauthorized' in auth_test.msg"
  msg: "Authentication failed. Check username and password."
```

**Validates:**
- Username and password correctness
- User has sufficient privileges
- Account is not locked

#### 3. State Validation

Checks SVM and relationship states before operations:

**Failover Pre-flight:**
```yaml
- name: Validate source SVM is running
  assert:
    that:
      - source_svm.state == "running"
    fail_msg: "Source SVM must be in running state for failover"

- name: Validate destination SVM is stopped
  assert:
    that:
      - dest_svm.state == "stopped"
    fail_msg: "Destination SVM must be stopped before failover"

- name: Validate SnapMirror relationships are healthy
  assert:
    that:
      - item.state == "snapmirrored"
      - item.healthy == true
    fail_msg: "SnapMirror relationship {{ item.uuid }} is not healthy"
  loop: "{{ snapmirror_relationships }}"
```

**Failback Pre-flight:**
```yaml
- name: Validate destination SVM is running
  assert:
    that:
      - dest_svm.state == "running"
    fail_msg: "Destination SVM must be running for failback"

- name: Validate no reverse relationships exist
  assert:
    that:
      - reverse_relationships | length == 0
    fail_msg: "Reverse SnapMirror relationships already exist. Run cleanup first."
```

#### 4. Resource Validation

Confirms required resources exist:

```yaml
- name: Validate vserver peering exists
  netapp.ontap.na_ontap_rest_info:
    hostname: "{{ input_prod_cluster }}"
    gather_subset: "svm/peers"
    parameters:
      svm.name: "{{ input_prod_svm }}"
      peer.svm.name: "{{ input_dr_svm }}"
  register: peering_info
  failed_when: 
    - peering_info.ontap_info['svm/peers'].num_records == 0
  msg: "Vserver peering does not exist between {{ input_prod_svm }} and {{ input_dr_svm }}"
```

**Validates:**
- Vserver peering exists
- SnapMirror relationships configured
- Required aggregates available
- Sufficient space for operations

#### 5. Configuration Validation

Checks input parameters and configuration:

```yaml
- name: Validate required parameters
  assert:
    that:
      - input_operation is defined
      - input_operation in ['create_and_sm', 'failover', 'failback', 'cleanup', 'remove_dest']
      - input_prod_svm is defined
      - input_dr_svm is defined
      - input_prod_cluster is defined
      - input_dr_cluster is defined
    fail_msg: "Missing or invalid required parameters"

- name: Validate CIFS parameters when CIFS enabled
  assert:
    that:
      - input_prod_cifs_name is defined
      - input_dr_cifs_name is defined
      - input_prod_cifs_name != input_dr_cifs_name
    fail_msg: "CIFS server names must be defined and different"
  when: 
    - cifs_enabled
    - not input_skip_cifs_rename | default(false)
```

#### 6. Relationship Health Validation

Detailed SnapMirror health checks:

```yaml
- name: Check SnapMirror transfer state
  assert:
    that:
      - item.state != "transferring"
    fail_msg: "SnapMirror relationship {{ item.uuid }} is currently transferring. Wait for completion."
  loop: "{{ snapmirror_relationships }}"
```

### Dry-Run Mode

Every workflow supports `input_dryrun=True` for complete validation without changes:

```bash
ansible-playbook svmdr_snaplock.yml -e "
  input_operation=failover
  input_dryrun=True
  ..."
```

**Dry-run provides:**
- Complete pre-flight validation
- Display of operations that would be performed
- No actual changes to ONTAP
- Safe testing of parameters and configuration

**Example Output:**
```
TASK [Dryrun execution]
ok: [localhost] => {
    "msg": "Dryrun mode enabled. No changes will be made."
}

TASK [debug]
ok: [localhost] => {
    "svmdr": {
        "source": {
            "svm": {"name": "svm_production", "state": "running"},
            "cluster": {"hostname": "prod-cluster.example.com"}
        },
        "destination": {
            "svm": {"name": "svm_dr", "state": "stopped"},
            "cluster": {"hostname": "dr-cluster.example.com"}
        },
        "operations": [
            "Stop source SVM: svm_production",
            "Break SnapMirror: 12 relationships",
            "Start destination SVM: svm_dr",
            "Rename CIFS: DRCIFS -> PRODCIFS"
        ]
    }
}
```

### Custom Validation

Add customer-specific validation:

```yaml
# In customer-specific logic role
- name: Custom business rule validation
  block:
    - name: Check maintenance window
      assert:
        that:
          - current_time | int > maintenance_start | int
          - current_time | int < maintenance_end | int
        fail_msg: "Operation not allowed outside maintenance window"
    
    - name: Verify change ticket exists
      uri:
        url: "https://cmdb.company.com/api/tickets/{{ change_ticket_id }}"
        return_content: yes
      register: ticket_info
      failed_when: ticket_info.json.status != "approved"
      
    - name: Check for active alerts
      uri:
        url: "https://monitoring.company.com/api/alerts"
        return_content: yes
      register: alerts
      failed_when: alerts.json.critical_count > 0
      msg: "Cannot proceed with active critical alerts"
```

---

## Integration and Deployment

### Deployment Models

The automation can be deployed in various configurations:

#### 1. Direct Ansible CLI

Simplest deployment for administrators:

```bash
# Install collections
ansible-galaxy collection install -r requirements.yml

# Customize inventory
cp vars/customer/template vars/customer/mycompany -r
vi vars/customer/mycompany/inventory.yml

# Execute operations
ansible-playbook svmdr_snaplock.yml -e "@vars/customer/mycompany/params.yml"
```

#### 2. AWX / Ansible Automation Platform

Enterprise deployment with role-based access:
  - suppotred as per AAP/Tower configuration guidelines

#### 3. CI/CD Pipeline Integration

Automated DR testing in GitOps workflow:

```yaml
# GitLab CI/CD Pipeline
stages:
  - validate
  - test_dr
  - cleanup

variables:
  ONTAP_USERNAME: "${VAULT_ONTAP_USER}"
  ONTAP_PASSWORD: "${VAULT_ONTAP_PASS}"

validate_dr_config:
  stage: validate
  script:
    - ansible-playbook svmdr_snaplock.yml -e "input_operation=create_and_sm input_dryrun=True" --syntax-check
    - ansible-lint svmdr_snaplock.yml
  only:
    - main

monthly_dr_test:
  stage: test_dr
  script:
    # Perform complete DR test
    - ansible-playbook svmdr_snaplock.yml -e "input_operation=failover input_dryrun=False"
    - sleep 300  # 5 minutes of DR operation
    - ansible-playbook svmdr_snaplock.yml -e "input_operation=failback input_dryrun=False"
  only:
    - schedules
  when: manual

cleanup_test:
  stage: cleanup
  script:
    - ansible-playbook svmdr_snaplock.yml -e "input_operation=cleanup input_dryrun=False"
  when: always
```

#### 4. AnsibleForms (Self-Service Portal)

Optional web UI for non-CLI users:

```yaml
# AnsibleForms form definition
---
name: SVMDR Stadard Ops *Planned Failover*
type: ansible
playbook: customers-playbooks/svmdr.yml
description: PoC
roles:
  - public
categories:
  - Customer Demo
tileClass: has-background-info-light
icon: bullseye
fields:
  - name: input_prod_cluster
    type: enum
    required: true
    label: Select PROD Cluster
    values:
      - hostname: trinidad.muccbc.hq.netapp.com
        clsname: Trinidad
        mgmt_ip: 10.65.59.220
    valueColumn: hostname
    columns:
      - clsname
    previewColumn: clsname
    model: input_prod_cluster
    group: Clusters
  - name: input_prod_svm
    type: enum
    label: Select PROD Vserver
    help: from what SVM configuration is read
    expression: fn.fnRestBasic('get','https://$(input_prod_cluster)/api/snapmirror/relationships?list_destinations_only=true&source.path=*:&fields=source.path,source.svm.name,destination.cluster.name,destination.svm.name','','muccbc_ontap_cluster','[.records[]
      | {svm_name:.source.svm.name,dr_name:.destination.svm.name}]')
    model: input_prod_svm
    required: true
    group: SVM
    valueColumn: name
  - name: prod_volumes
    type: enum
    label: Volumes list (informational)
    help: Informational output only
    expression: fn.fnRestBasic('get','https://$(input_prod_cluster)/api/storage/volumes?svm.name=$(input_prod_svm)&fields=name,state,snapmirror.is_protected','','muccbc_ontap_cluster','[.records[]
      | {name:.name,state:.state,is_protected:.snapmirror.is_protected}]')
    model: input_prod_svm
    group: SVM
    valueColumn: name
  - name: svm_state
    type: expression
    label: PROD SVM state
    expression: fn.fnRestBasic('get','https://$(input_prod_cluster)/api/svm/svms?name=$(input_prod_svm)&fields=state','','muccbc_ontap_cluster','.records[0].state')
    group: SVM
  - name: input_username
    type: text
    group: Credentials
    required: true
    label: ONTAP Cluster username
    help: Admin permissions required
  - name: input_password
    type: password
    group: Credentials
    required: true
    label: Password for username
  - name: input_dryrun
    type: checkbox
    label: Dryrun execution
    help: Mark here if you want only data collection/print
    group: Operation
  - name: no_verbose
    type: checkbox
    label: No verbose output
    default: true
    group: Operation
  - name: input_operation
    type: expression
    label: Operation available
    expression: '((state) => {return state === "running" ? "failover" : state ===
      "stopped" ? "failback" : null; })("$(svm_state)")'
    runLocal: true
    group: Operation
  - name: sm_state_retries
    label: Retries
    help: Number of transfer completion check retries
    type: number
    default: 100
    group: Operation
  - name: sm_state_delay
    label: Delay
    help: Delay in sec between retries
    type: number
    default: 5
    group: Operation

```

#### Monitoring Integration

Send metrics to monitoring systems:

```yaml
- name: Send operation metrics to Prometheus
  uri:
    url: "https://pushgateway.company.com/metrics/job/svmdr"
    method: POST
    body: |
      # TYPE svmdr_operation_duration_seconds gauge
      svmdr_operation_duration_seconds{operation="{{ input_operation }}",svm="{{ input_prod_svm }}"} {{ operation_duration }}
      # TYPE svmdr_operation_status gauge
      svmdr_operation_status{operation="{{ input_operation }}",svm="{{ input_prod_svm }}"} {{ operation_status }}
    headers:
      Content-Type: "text/plain"

- name: Send Slack notification
  slack:
    token: "{{ slack_token }}"
    channel: "#storage-ops"
    msg: |
      ✅ SVMDR {{ input_operation }} completed successfully
      SVM: {{ input_prod_svm }}
      Duration: {{ operation_duration }}s
      Operator: {{ operator_name }}
```

### Security Considerations

#### Credential Management

**Use Ansible Vault:**
```bash
# Encrypt credentials
ansible-vault encrypt vars/customer/mycompany/secrets.yml

# Run with vault
ansible-playbook svmdr_snaplock.yml --ask-vault-pass

# Use vault password file
ansible-playbook svmdr_snaplock.yml --vault-password-file ~/.vault_pass
```

**Use External Secret Management:**
```yaml
# HashiCorp Vault integration
- name: Read credentials from Vault
  community.hashi_vault.vault_kv2_get:
    url: "https://vault.company.com"
    path: "secret/ontap/{{ input_prod_cluster }}"
    auth_method: "token"
    token: "{{ lookup('env', 'VAULT_TOKEN') }}"
  register: vault_secret

- name: Use credentials
  set_fact:
    ontap_username: "{{ vault_secret.secret.username }}"
    ontap_password: "{{ vault_secret.secret.password }}"
```

#### Access Control

**Role-Based Access in Ansible Forms:**
   - supported with MS AD
   - supported with local user setup


### Best Practices

1. **Always Test with Dry-Run First**
   ```bash
   ansible-playbook <playbook> -e "input_dryrun=True ..."
   ```

2. **Use Version Control for Inventory**
   - Store inventory in Git
   - Review changes via pull requests
   - Tag releases for production use

3. **Schedule Regular DR Tests**
   ```yaml
   # Cron schedule: Monthly DR test
   0 2 1 * * ansible-playbook svmdr_snaplock.yml -e "input_operation=failover ..." && \
             sleep 3600 && \
             ansible-playbook svmdr_snaplock.yml -e "input_operation=failback ..."
   ```

4. **Monitor SnapMirror Health**
   - Monitor transfer errors
   - Track relationship status

5. **Document Customizations**
   - Maintain README in customer directory
   - Document any custom pre-flight checks
   - Record deviation from standard configuration

6. **Maintain Auxiliary LIFs**
   - Regularly test AD connectivity
   - Keep LIF naming consistent
   - Preserve during cleanup operations

7. **Backup Configuration**
   ```bash
   # Backup before major changes
   ansible-playbook svmdr_snaplock.yml -e "input_operation=create_and_sm" --check --diff > backup_$(date +%Y%m%d).txt
   ```

---

## Conclusion

This automation framework provides enterprise-grade SVM disaster recovery capabilities with flexibility to meet diverse customer requirements:

- **SVMDR Standard**: Simple, native SVMDR for environments without SnapLock
- **SVMDR Pseudo**: Advanced DR with SnapLock support and granular control
- **QuickDR**: Rapid testing without SnapMirror overhead

The **svm_clone role** at the heart of the framework enables comprehensive SVM configuration replication and can be easily extended to support new features as NetApp ONTAP evolves.

### Key Takeaways

✅ **SnapLock Support**: Pseudo-SVMDR enables DR for compliance environments  
✅ **Granular Control**: Failover individual volumes or subsets as needed  
✅ **Extensibility**: Easy to add new configuration items and features  
✅ **Flexibility**: Extensive customization for diverse requirements  
✅ **Automation**: Reduce RTO from hours to minutes  
✅ **Auditability**: Complete logging and dry-run capabilities  

### Getting Started

1. Review your environment and SnapLock requirements
2. Choose appropriate workflow (Standard vs Pseudo)
3. Customize inventory for your topology
4. Test with dry-run mode
5. Schedule regular DR tests
6. Integrate with your automation platform

For questions or support, consult your NetApp Professional Services representative.

---
*Document Version: 1.0*  
*Last Updated: January 26, 2026*  
*Tested on: ONTAP 9.14.1*  
*Compatible with: ONTAP 9.14 and higher*  
*API Strategy: REST API Only (ONTAPI-free)*
