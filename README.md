# OCP Virtualization Snapshot Role

An Ansible role for creating KubeVirt VirtualMachine snapshots in OpenShift Virtualization (OCP Virt).

## Overview

This role creates snapshots of KubeVirt VirtualMachines in OpenShift/Kubernetes clusters. Snapshots are automatically named using a timestamp format (`aap-snapshot-YYYYMMDD-HHMMSS`) to ensure uniqueness.

## Features

- Automatic snapshot name generation (timestamp-based)
- Validates VM existence before snapshot creation
- Waits for snapshot completion with retry logic
- Error handling for common failure scenarios
- Works with AAP (Ansible Automation Platform)
- Supports all Kubernetes authentication methods

## Prerequisites

### 1. OpenShift Virtualization (KubeVirt) Requirements

- OpenShift Virtualization must be installed and configured
- KubeVirt snapshot API must be available (`snapshot.kubevirt.io/v1alpha1`)
- VirtualMachineSnapshot CRD must be installed

### 2. Required Permissions

The service account or user running this role needs the following permissions:

#### Minimum Required (Namespace-scoped)
For snapshotting VMs in a specific namespace, create a RoleBinding with the `edit` role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: snapshot-creator
  namespace: <target-namespace>
subjects:
- kind: ServiceAccount
  name: <service-account-name>
  namespace: <service-account-namespace>
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

**Required Permissions:**
- `get`, `list` on `virtualmachines.kubevirt.io` (to validate VM exists)
- `create`, `get`, `list` on `virtualmachinesnapshots.snapshot.kubevirt.io` (to create and check snapshot status)

#### Production Recommendation (Cluster-wide)
For production environments where snapshots may be created across multiple namespaces, use a ClusterRoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: snapshot-creator-cluster
subjects:
- kind: ServiceAccount
  name: <service-account-name>
  namespace: <service-account-namespace>
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

**Note:** The `edit` ClusterRole provides sufficient permissions for snapshot operations. For more restrictive access, create a custom ClusterRole with only the required permissions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kubevirt-snapshot-creator
rules:
- apiGroups:
  - kubevirt.io
  resources:
  - virtualmachines
  verbs:
  - get
  - list
- apiGroups:
  - snapshot.kubevirt.io
  resources:
  - virtualmachinesnapshots
  verbs:
  - create
  - get
  - list
```

#### Step-by-Step Setup Instructions

**Option 1: Namespace-Scoped Setup (Recommended for Testing)**

1. Create a Service Account in your target namespace:
   ```bash
   oc create serviceaccount snapshot-creator -n <target-namespace>
   ```

2. Create a RoleBinding to grant the `edit` role to the Service Account:
   ```bash
   oc create rolebinding snapshot-creator-binding \
     --clusterrole=edit \
     --serviceaccount=<target-namespace>:snapshot-creator \
     -n <target-namespace>
   ```

3. Verify the Service Account was created:
   ```bash
   oc get serviceaccount snapshot-creator -n <target-namespace>
   ```

4. Verify the RoleBinding was created:
   ```bash
   oc get rolebinding snapshot-creator-binding -n <target-namespace>
   ```

**Option 2: Cluster-Wide Setup (Recommended for Production)**

1. Create a Service Account in a dedicated namespace (e.g., `ansible-automation`):
   ```bash
   oc create namespace ansible-automation
   oc create serviceaccount snapshot-creator -n ansible-automation
   ```

2. Create a ClusterRoleBinding to grant cluster-wide `edit` permissions:
   ```bash
   oc create clusterrolebinding snapshot-creator-cluster-binding \
     --clusterrole=edit \
     --serviceaccount=ansible-automation:snapshot-creator
   ```

3. Verify the Service Account was created:
   ```bash
   oc get serviceaccount snapshot-creator -n ansible-automation
   ```

4. Verify the ClusterRoleBinding was created:
   ```bash
   oc get clusterrolebinding snapshot-creator-cluster-binding
   ```

**Option 3: Custom Minimal Permissions (Most Secure)**

1. Create a Service Account:
   ```bash
   oc create serviceaccount snapshot-creator -n <namespace>
   ```

2. Create the custom ClusterRole with minimal permissions using YAML:
   ```bash
   cat <<EOF | oc apply -f -
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: kubevirt-snapshot-creator
   rules:
   - apiGroups:
     - kubevirt.io
     resources:
     - virtualmachines
     verbs:
     - get
     - list
   - apiGroups:
     - snapshot.kubevirt.io
     resources:
     - virtualmachinesnapshots
     verbs:
     - create
     - get
     - list
   EOF
   ```

3. Create a ClusterRoleBinding:
   ```bash
   oc create clusterrolebinding snapshot-creator-custom-binding \
     --clusterrole=kubevirt-snapshot-creator \
     --serviceaccount=<namespace>:snapshot-creator
   ```

**Using the Service Account Token in AAP**

After creating the Service Account, you need to extract the token for use in AAP:

1. Get the Service Account token secret name:
   ```bash
   oc get serviceaccount snapshot-creator -n <namespace> -o jsonpath='{.secrets[0].name}'
   ```

2. Extract the token:
   ```bash
   oc get secret <secret-name> -n <namespace> -o jsonpath='{.data.token}' | base64 -d
   ```

3. Alternatively, create a new token (OpenShift 4.11+):
   ```bash
   oc create token snapshot-creator -n <namespace> --duration=8760h
   ```

4. Use this token in AAP by:
   - Creating a Kubernetes credential in AAP
   - Setting the API Key field to the token value
   - Setting the Host field to your OpenShift API server URL (e.g., `https://api.your-cluster.example.com:6443`)

**Verifying Permissions**

Test that the Service Account has the required permissions:

```bash
# Test VM access
oc auth can-i get virtualmachines --as=system:serviceaccount:<namespace>:snapshot-creator -n <target-namespace>

# Test snapshot creation
oc auth can-i create virtualmachinesnapshots --as=system:serviceaccount:<namespace>:snapshot-creator -n <target-namespace>
```

Both commands should return `yes` if permissions are correctly configured.

### 3. Security Considerations

This role handles sensitive authentication credentials (API keys, passwords, kubeconfig files). The following security measures are implemented:

#### Sensitive Data Protection

- **No Logging of Credentials**: All tasks that use sensitive credentials (`api_key`, `password`, `username`, `kubeconfig`, `host`) have `no_log: true` enabled to prevent credential exposure in Ansible logs
- **Protected Tasks**: The following tasks suppress credential logging:
  - `Get VirtualMachine info`
  - `Create VirtualMachineSnapshot`
  - `Wait for snapshot to be ready`

#### Best Practices

1. **Use AAP Credentials**: Store Kubernetes credentials in AAP's credential store rather than passing them as extra variables
2. **Service Account Tokens**: Prefer Service Account tokens over user credentials when possible
3. **Environment Variables**: When using environment variables, ensure they are not logged or exposed in shell history
4. **Kubeconfig Files**: If using kubeconfig files, ensure proper file permissions (e.g., `chmod 600 ~/.kube/config`)
5. **Avoid Verbose Logging**: Be cautious when using `-vvv` verbose mode, as it may expose more information than necessary
6. **Secure Storage**: Never commit credentials to version control - use `.gitignore` to exclude sensitive files

#### Credential Management in AAP

When configuring AAP Job Templates:

- **Use OpenShift/Kubernetes Credential Type**: This is the recommended method as AAP handles credential injection securely
- **Avoid Extra Variables for Credentials**: Do not pass `api_key`, `password`, or `kubeconfig` content via extra variables or surveys
- **Use Environment Variables**: AAP automatically injects `K8S_AUTH_*` environment variables when a Kubernetes credential is attached

#### Error Messages

Error messages are designed to avoid exposing sensitive information. They only reference:
- VM names
- Namespace names
- Snapshot names
- Generic error messages from the Kubernetes API

No credentials, tokens, or authentication details are included in error output.

### 4. Ansible Requirements

- Ansible 2.9 or higher
- `kubernetes.core` collection (version >= 2.0.0)
- Python with `kubernetes` library

Install the collection:
```bash
ansible-galaxy collection install -r requirements.yml
```

## Installation

1. Clone this repository or add it to your Ansible project
2. Install required collections:
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

## Usage

### Basic Playbook

```yaml
---
- name: Create KubeVirt VM Snapshot
  hosts: localhost
  gather_facts: yes
  
  tasks:
    - name: Create KubeVirt snapshot
      ansible.builtin.include_role:
        name: ocp_virt_snapshot
      vars:
        vm_name: "my-vm"
        vm_namespace: "my-namespace"
```

### Required Variables

| Variable | Type | Description |
|----------|------|-------------|
| `vm_name` | string | Name of the VirtualMachine to snapshot |
| `vm_namespace` | string | Kubernetes namespace containing the VM |

### Optional Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `kubeconfig` | string | `K8S_AUTH_KUBECONFIG` env | Path to kubeconfig file |
| `context` | string | `K8S_AUTH_CONTEXT` env | Kubernetes context to use |
| `host` | string | `K8S_AUTH_HOST` env | Kubernetes API server URL |
| `api_key` | string | `K8S_AUTH_API_KEY` env | API key for authentication |
| `username` | string | `K8S_AUTH_USERNAME` env | Username for authentication |
| `password` | string | `K8S_AUTH_PASSWORD` env | Password for authentication |
| `validate_certs` | bool | `true` | Validate SSL certificates |

**Note:** When using AAP (Ansible Automation Platform), Kubernetes credentials are automatically injected via `K8S_AUTH_*` environment variables. You typically only need to set `vm_name` and `vm_namespace`.

### Ansible Automation Platform (AAP) Integration

See `playbooks/aap-job-template-survey-example.md` for detailed AAP configuration instructions.

**Quick Setup:**
1. Create a Job Template pointing to `playbooks/create-snapshot.yml`
2. Add a Survey with:
   - `vm_name` (required, Text)
   - `vm_namespace` (required, Text)
3. Attach OpenShift/Kubernetes credentials
4. The snapshot name is automatically generated

## Return Values

The role sets the following facts:

### `snapshot_info`
A dictionary containing snapshot information:

```yaml
snapshot_info:
  vm_name: "my-vm"
  vm_namespace: "my-namespace"
  snapshot_name: "aap-snapshot-20251215-102235"
  snapshot_uid: "c935f368-e9c5-4ff4-a21f-e8015b7086c9"
  status: "success"
```

### `snapshot_created`
Boolean indicating if the snapshot was successfully created.

## Snapshot Naming

Snapshots are automatically named using the format:
```
aap-snapshot-YYYYMMDD-HHMMSS
```

Where:
- `YYYYMMDD` = Date (e.g., `20251215` for December 15, 2025)
- `HHMMSS` = Time in UTC (e.g., `102235` for 10:22:35)

**Note:** The timestamp uses UTC time from the execution environment.

## Examples

### Example 1: Basic Snapshot Creation

```yaml
---
- name: Create VM Snapshot
  hosts: localhost
  gather_facts: yes
  
  tasks:
    - name: Create snapshot
      ansible.builtin.include_role:
        name: ocp_virt_snapshot
      vars:
        vm_name: "rhel9-server-1"
        vm_namespace: "production"
    
    - name: Display snapshot info
      ansible.builtin.debug:
        var: snapshot_info
```

### Example 2: Using Custom Kubeconfig

```yaml
---
- name: Create VM Snapshot with Custom Kubeconfig
  hosts: localhost
  gather_facts: yes
  
  tasks:
    - name: Create snapshot
      ansible.builtin.include_role:
        name: ocp_virt_snapshot
      vars:
        vm_name: "my-vm"
        vm_namespace: "my-namespace"
        kubeconfig: "/path/to/kubeconfig"
```

## Error Handling

The role handles the following error conditions:

1. **VM Not Found**: Fails with a clear error message if the VM doesn't exist
2. **Snapshot Creation Failure**: Captures and reports Kubernetes API errors
3. **Snapshot Timeout**: Waits up to 150 seconds (30 retries × 5 seconds) for snapshot completion
4. **Snapshot Error Status**: Detects and reports snapshot errors from KubeVirt

## Limitations

- Each execution creates a new snapshot (not idempotent by design)
- Snapshot names are timestamp-based and cannot be customized
- Maximum wait time: 150 seconds for snapshot completion
- Requires KubeVirt snapshot API (`snapshot.kubevirt.io/v1alpha1`)

## Troubleshooting

### Error: "VM not found"
- Verify the VM name and namespace are correct
- Check that the service account has `get` and `list` permissions on `virtualmachines.kubevirt.io`
- Ensure you're connected to the correct cluster/context

### Error: "Failed to create object"
- Verify the service account has `create` permission on `virtualmachinesnapshots.snapshot.kubevirt.io`
- Check that the KubeVirt snapshot API is available
- Review Kubernetes API server logs for detailed error messages

### Error: "Snapshot failed"
- Check the snapshot status in Kubernetes: `kubectl get vmsnapshot <snapshot-name> -n <namespace> -o yaml`
- Review KubeVirt controller logs for snapshot processing errors
- Verify VM is in a state that allows snapshots (not paused, not migrating, etc.)

### Snapshot Name Issues
- Snapshot names must be valid Kubernetes resource names (lowercase alphanumeric, hyphens, dots)
- Names are automatically generated and should not conflict

## Development

### Project Structure

```
.
├── playbooks/
│   ├── create-snapshot.yml          # Main playbook
│   └── aap-job-template-survey-example.md  # AAP configuration guide
├── roles/
│   └── ocp_virt_snapshot/
│       ├── defaults/
│       │   └── main.yml             # Default variables
│       ├── meta/
│       │   └── main.yml             # Role metadata
│       └── tasks/
│           ├── main.yml             # Main entry point
│           └── create.yml           # Snapshot creation logic
├── requirements.yml                 # Ansible collection requirements
└── README.md                        # This file
```

## License

MIT

## Author

Jon Patterson

## Version History

- **2.0** (December 2025): Simplified for AAP, removed snapshot name survey, automatic timestamp-based naming
- **1.0**: Initial release

