# AAP Job Template Configuration Example

## Job Template: Create VM Snapshot

### Basic Information
- **Name**: `OCP Virtualization - Create VM Snapshot`
- **Description**: `Create a snapshot of a KubeVirt VM (handles single and multi-volume VMs automatically)`
- **Job Type**: `Run`
- **Inventory**: `localhost` (or your localhost inventory)
- **Project**: Select your project containing the playbook
- **Playbook**: `examples/aap-job-template-create-snapshot.yml`
- **Credentials**: 
  - Add OpenShift/Kubernetes credentials (OpenShift credential type)
  - Or use Machine credential type with kubeconfig content

### Tags
- **Tags to run**: `create,ocp_virt_snapshot`
- **Tags to skip**: (optional) `debug` to reduce output

### Extra Variables
```yaml
# Required - Set via Survey or Extra Variables
vm_name: "{{ vm_name }}"
vm_namespace: "{{ vm_namespace }}"
kubeconfig: "{{ kubeconfig_path }}"
snapshot_name: "{{ snapshot_name | default('aap-snapshot-' + ansible_date_time.epoch) }}"
```

### Survey Configuration

Create a Survey for interactive variable input:

#### Question 1: VM Name
- **Variable**: `vm_name`
- **Type**: `Text`
- **Required**: Yes
- **Description**: `Name of the KubeVirt VirtualMachine to snapshot`

#### Question 2: VM Namespace
- **Variable**: `vm_namespace`
- **Type**: `Text`
- **Required**: Yes
- **Description**: `Kubernetes namespace where the VM exists`
- **Default**: `default`

#### Question 3: Snapshot Name
- **Variable**: `snapshot_name`
- **Type**: `Text`
- **Required**: No
- **Description**: `Name for the snapshot (auto-generated if not provided)`
- **Default**: `aap-snapshot-{{ ansible_date_time.epoch }}`

### Options
- **Enable Privilege Escalation**: No
- **Verbosity**: 1 (Normal) or 2 (More Verbose)
- **Show Changes**: Yes
- **Limit**: (leave empty)
- **Forks**: (default)

### Execution Environment
- Use an execution environment that includes:
  - `kubernetes.core` collection
  - Python with `kubernetes` library

### Notes on Multi-Volume VMs

KubeVirt snapshots automatically include all disks/volumes attached to a VM. No special configuration is needed.

### Workflow Integration

This job template can be used in workflows:

**Example Workflow: Backup and Verify**
1. **Job Template 1**: Create Snapshot (this template)
   - On Success → Job Template 2
   - On Failure → Notify
2. **Job Template 2**: Verify Snapshot (kubectl or verification script)
   - Use `snapshot_info` from Job Template 1
   - On Success → Notify Success
   - On Failure → Notify Failure

### Notification Templates

Use the following variables in notification templates:

```
Snapshot Created Successfully

VM: {{ snapshot_info.vm_name }}
Namespace: {{ snapshot_info.vm_namespace }}
Snapshot: {{ snapshot_info.snapshot_name }}
Snapshot UID: {{ snapshot_info.snapshot_uid }}
Status: {{ snapshot_info.snapshot_phase }}
Disks Included: {{ snapshot_info.vm_disks_count }}
```

### Testing

- Single disk VM: Verify `vm_disks_count` is 1
- Multi-disk VM: Verify `vm_disks_count` matches actual disk count
- Check mode: Validation runs, snapshot creation is skipped
