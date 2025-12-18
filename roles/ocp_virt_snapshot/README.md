# OpenShift Virtualization Snapshot Creation Role

An Ansible role for creating snapshots of KubeVirt virtual machines in OpenShift Container Platform environments. This role is for use with Ansible Automation Platform (AAP) job templates and handles VMs with single or multiple volumes/disks.

## Features

- **Create Snapshots**: Create VirtualMachineSnapshot CRDs for KubeVirt VMs
- **Multi-Volume Support**: Handles VMs with single or multiple disks/volumes - all disks are included in snapshots by default
- **Idempotent Operations**: All operations are idempotent and safe to run multiple times
- **Validation**: Validates inputs and provides clear error messages
- **AAP Ready**: Built for Ansible Automation Platform with tagging and structured return values

## Requirements

- Ansible 2.9 or higher
- Python 3.6 or higher
- `kubernetes.core` collection (version >= 2.0.0)
- Access to OpenShift/Kubernetes cluster with KubeVirt installed
- Valid kubeconfig file or cluster credentials
- Permissions to create VirtualMachineSnapshot resources

## Installation

### Install Required Collections

```bash
ansible-galaxy collection install kubernetes.core
```

### Install the Role

```bash
ansible-galaxy install -r requirements.yml
```

Or manually place this role in your `roles/` directory.

## Role Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `vm_name` | VirtualMachine name | `my-vm-name` |
| `vm_namespace` | Kubernetes namespace where VM exists | `default` |

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `kubeconfig` | Path to kubeconfig file | `null` |
| `context` | Kubernetes context to use | `null` |
| `host` | Kubernetes API server URL | `null` |
| `api_key` | API key for authentication | `null` |
| `username` | Username for authentication | `null` |
| `password` | Password for authentication | `null` |
| `validate_certs` | Validate SSL certificates | `true` |
| `snapshot_name` | Snapshot name | Auto-generated timestamp |
| `snapshot_force` | Force overwrite if snapshot exists | `false` |

**Note**: For authentication, provide either `kubeconfig`, `context`, or `host` with `api_key`/`username`+`password`.

## Usage Examples

### Create a Snapshot

```yaml
- name: Create a snapshot
  hosts: localhost
  gather_facts: yes
  tasks:
    - name: Create snapshot
      ansible.builtin.include_role:
        name: ocp_virt_snapshot
      vars:
        vm_name: web-server-01
        vm_namespace: production
        kubeconfig: ~/.kube/config
        snapshot_name: pre-upgrade-backup
```

**Note**: KubeVirt snapshots automatically include ALL disks/volumes attached to the VM. For VMs with multiple volumes, no special configuration is needed - all disks are captured in a single snapshot operation.

### Create a Snapshot Using Context

```yaml
- name: Create snapshot using context
  hosts: localhost
  tasks:
    - name: Create snapshot
      ansible.builtin.include_role:
        name: ocp_virt_snapshot
      vars:
        vm_name: database-server
        vm_namespace: database
        context: my-cluster-context
        snapshot_name: backup-2024
```

### Create Snapshots for Multiple VMs

```yaml
---
- name: Create snapshots for multiple VMs
  hosts: localhost
  gather_facts: yes
  vars:
    kubeconfig: ~/.kube/config
    target_vms:
      - name: web-server-01
        namespace: production
      - name: database-server
        namespace: database
      - name: app-server-01
        namespace: production

  tasks:
    - name: Create snapshots for all VMs
      ansible.builtin.include_role:
        name: ocp_virt_snapshot
      vars:
        vm_name: "{{ item.name }}"
        vm_namespace: "{{ item.namespace }}"
        snapshot_name: "backup-{{ ansible_date_time.epoch }}"
      loop: "{{ target_vms }}"
```

## Return Values

After snapshot creation, the role sets the following facts that can be used in subsequent tasks:

- `snapshot_created`: Boolean indicating success
- `snapshot_info`: Dictionary with snapshot details containing:
  - `vm_name`: Name of the VM
  - `vm_namespace`: Namespace of the VM
  - `snapshot_name`: Name of the created snapshot
  - `snapshot_uid`: UID of the snapshot resource
  - `snapshot_phase`: Current phase of the snapshot
  - `snapshot_ready`: Boolean indicating if snapshot is ready
  - `created_at`: Timestamp when snapshot was created
  - `action`: "created"
  - `status`: "success"
  - `vm_disks_count`: Number of disks included in the snapshot
  - `note`: Information about multi-volume support

Example usage:
```yaml
- name: Display snapshot information
  ansible.builtin.debug:
    msg:
      - "Snapshot created successfully!"
      - "VM: {{ snapshot_info.vm_name }}"
      - "Namespace: {{ snapshot_info.vm_namespace }}"
      - "Snapshot Name: {{ snapshot_info.snapshot_name }}"
      - "Status: {{ snapshot_info.snapshot_phase }}"
      - "Disks Included: {{ snapshot_info.vm_disks_count }}"
```

## Tagging

Tags for selective task execution:
- `create`, `snapshot` - Snapshot operations
- `validation`, `facts`, `check`, `debug` - Functional tags
- `ocp_virt_snapshot` - All role tasks (catch-all)
- `always`, `never` - Special tags (always run / skip in check mode)

For detailed AAP integration guidance, see [AAP_BEST_PRACTICES.md](../../AAP_BEST_PRACTICES.md).

## Multi-Volume VM Support

KubeVirt snapshots automatically include all disks/volumes attached to a VM. The role handles single and multi-disk VMs without additional configuration. The `snapshot_info.vm_disks_count` field reports how many disks were included.

## Best Practices

- Store Kubernetes credentials in AAP Credentials
- Use descriptive snapshot names with timestamps
- Test in non-production environments first
- Ensure proper RBAC permissions for the namespace

## Troubleshooting

### Common Issues

1. **VM Not Found**
   - Verify VM name and namespace are correct
   - Ensure user has permissions to view the VM
   - Check that KubeVirt is installed in the cluster

2. **Permission Denied**
   - Verify user has appropriate Kubernetes permissions
   - Required permissions: `get`, `list` on `virtualmachines.kubevirt.io`
   - Required permissions: `create`, `get`, `delete` on `virtualmachinesnapshots.snapshot.kubevirt.io`

3. **SSL Certificate Errors**
   - Set `validate_certs: false` for self-signed certificates (not recommended for production)
   - Or install proper CA certificates

4. **Snapshot Creation Fails**
   - Check VM is not being deleted or migrated
   - Verify sufficient storage space in the cluster
   - Check VirtualMachineSnapshot CRD is installed

5. **Snapshot Not Ready**
   - Snapshots are created asynchronously
   - The role waits for snapshot to be ready (up to 30 retries with 5 second delays)
   - Check snapshot status in Kubernetes: `kubectl get vmsnapshot -n <namespace>`

## Testing

- Single disk VM: Verify `vm_disks_count` is 1
- Multi-disk VM: Verify `vm_disks_count` matches actual disk count
- Check mode: Validation runs, snapshot creation is skipped

## License

MIT

## Author Information

This role provides KubeVirt snapshot creation capabilities for Ansible Automation Platform integration with OpenShift Virtualization.

## Support

For issues, questions, or contributions, please open an issue in the repository.
