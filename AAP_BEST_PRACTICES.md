# Ansible Automation Platform Best Practices

This role follows Ansible Automation Platform (AAP) / Ansible Tower best practices for OpenShift Virtualization snapshot creation.

## Tagging Strategy

The role uses tagging for selective task execution in AAP job templates. Tags are organized hierarchically:

### Primary Action Tags
- `create` - Create snapshot operations

### Functional Tags
- `validation` - Input validation tasks
- `facts` - Fact gathering and setting
- `check` - Pre-operation checks
- `debug` - Debug output tasks
- `snapshot` - Snapshot-specific operations
- `ocp_virt_snapshot` - All role tasks (catch-all)

### Special Tags
- `always` - Always run (validation, fact setting)
- `never` - Skip in check mode (destructive operations)

## Using Tags in AAP Job Templates

- Run only validation: Set tags to `validation`
- Dry run (check mode): Enable "Check Mode", tags with `never` are skipped
- Create snapshot (skip debug): Set tags to `create`, skip tags `debug`

## Return Values

The role sets `snapshot_created` (boolean) and `snapshot_info` (dictionary) facts. The `snapshot_info` dictionary contains:
- `vm_name`, `vm_namespace`, `snapshot_name`, `snapshot_uid`
- `snapshot_phase`, `snapshot_ready`, `created_at`
- `action`, `status`, `vm_disks_count`

These can be used in AAP workflow job templates for conditional logic.

## Check Mode Support

Tasks that modify state are tagged with `never` to prevent execution in check mode:
- Snapshot creation

Tasks that are safe in check mode:
- Validation
- Fact gathering
- Debug output

## Variable Management

**AAP Credentials**: Store Kubernetes credentials in AAP Credentials (OpenShift/Kubernetes credential type)

**Extra Variables**: Set role variables in Job Template Extra Variables

**Surveys**: Create surveys for interactive input:
- `vm_name` (text, required)
- `vm_namespace` (text, required)
- `snapshot_name` (text, optional)
- `snapshot_force` (multiple choice: true/false)

## Job Template Configuration

**Recommended Settings**:
- Inventory: Static inventory with localhost
- Playbook: `examples/aap-job-template-create-snapshot.yml`
- Tags: `create,ocp_virt_snapshot`
- Verbosity: 1-2 for normal operations

**Workflow Example**: Create snapshot → Verify snapshot → Notify on completion

## Notification Integration

Use `snapshot_info` facts in notification templates: `{{ snapshot_info.vm_name }}`, `{{ snapshot_info.snapshot_name }}`, `{{ snapshot_info.status }}`, etc.

## Multi-Volume VM Support

KubeVirt snapshots automatically include all disks/volumes attached to a VM. The role handles single and multi-disk VMs without additional configuration. The `snapshot_info` fact includes `vm_disks_count` to show how many disks were included.

## Job Template Setup

See `examples/aap-job-template-survey-example.md` for detailed job template configuration.

**Quick Setup**:
- Playbook: `examples/aap-job-template-create-snapshot.yml`
- Inventory: `localhost`
- Tags: `create,ocp_virt_snapshot`
- Credentials: OpenShift/Kubernetes credential type

## Workflow Integration

The role's structured return values enable workflow integration. Use `snapshot_info` from the snapshot creation job template in subsequent workflow steps.

## Security & Performance

- Store credentials in AAP Credentials, never in playbooks
- Use Ansible Vault for sensitive variables
- Use AAP RBAC to control access
- Run snapshot operations in parallel for multiple VMs using workflows
- Use tags to skip unnecessary tasks (e.g., skip debug tags in production)

## Testing

1. **Single Volume VM**: Test with a VM that has one disk, verify `vm_disks_count` is 1
2. **Multi-Volume VM**: Test with a VM that has multiple disks, verify `vm_disks_count` matches actual count
3. **Check Mode**: Enable check mode, verify validation runs but snapshot creation is skipped
4. **Error Handling**: Test with invalid VM name, invalid namespace, and existing snapshot name
