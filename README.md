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
- `get`, `list` on `virtualmachines.kubevirt.io` (to validate VM exists)
- `create`, `get`, `list` on `virtualmachinesnapshots.snapshot.kubevirt.io` (to create and check snapshot status)

#### Service Account Configuration Options

You can configure the service account to work with snapshots in one namespace, multiple specific namespaces, or all namespaces in the cluster. Choose the option that best fits your security requirements:

**Configuration Comparison:**

| Configuration | RBAC Type | Security Level | Use Case | Setup Complexity |
|--------------|-----------|----------------|----------|------------------|
| **Single Namespace** | RoleBinding | Highest (most restrictive) | Testing, single project | Simple |
| **Multiple Namespaces** | Multiple RoleBindings | High (moderate restriction) | Production with limited scope | Moderate |
| **All Namespaces** | ClusterRoleBinding | Lower (least restrictive) | Production with broad scope | Simple |

**Quick Decision Guide:**
- **Testing or single project?** → Use **Single Namespace** (Option 1)
- **Multiple specific projects?** → Use **Multiple Namespaces** (Option 2)
- **All projects or dynamic namespaces?** → Use **All Namespaces** (Option 3)
- **Maximum security required?** → Use **Custom Minimal Permissions** (Option 4) with any of the above

---

#### Option 1: Single Namespace Configuration

Use this when you only need to create snapshots in one specific namespace.

**Step 1: Create Service Account**

Create the service account in your target namespace or a dedicated namespace:

```bash
# Option A: Create in the target namespace
oc create serviceaccount snapshot-creator -n <target-namespace>

# Option B: Create in a dedicated namespace (recommended)
oc create namespace ansible-automation
oc create serviceaccount snapshot-creator -n ansible-automation
```

**Step 2: Create RoleBinding**

Bind the `edit` ClusterRole to the service account in the target namespace:

```bash
# If service account is in the same namespace as target VMs
oc create rolebinding snapshot-creator-binding \
  --clusterrole=edit \
  --serviceaccount=<target-namespace>:snapshot-creator \
  -n <target-namespace>

# If service account is in a different namespace (e.g., ansible-automation)
oc create rolebinding snapshot-creator-binding \
  --clusterrole=edit \
  --serviceaccount=ansible-automation:snapshot-creator \
  -n <target-namespace>
```

**Example:**
```bash
# Create service account in dedicated namespace
oc create namespace ansible-automation
oc create serviceaccount snapshot-creator -n ansible-automation

# Grant permissions in vm-postbuild-config namespace
oc create rolebinding snapshot-creator-binding \
  --clusterrole=edit \
  --serviceaccount=ansible-automation:snapshot-creator \
  -n vm-postbuild-config
```

**YAML Alternative:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: snapshot-creator-binding
  namespace: <target-namespace>
subjects:
- kind: ServiceAccount
  name: snapshot-creator
  namespace: ansible-automation
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

**Verify:**
```bash
# Check RoleBinding exists
oc get rolebinding snapshot-creator-binding -n <target-namespace>

# Test permissions
oc auth can-i get virtualmachines \
  --as=system:serviceaccount:ansible-automation:snapshot-creator \
  -n <target-namespace>

oc auth can-i create virtualmachinesnapshots \
  --as=system:serviceaccount:ansible-automation:snapshot-creator \
  -n <target-namespace>
```

---

#### Option 2: Multiple Namespaces Configuration

Use this when you need to create snapshots in multiple specific namespaces but not all namespaces.

**Step 1: Create Service Account**

Create the service account in a dedicated namespace:

```bash
oc create namespace ansible-automation
oc create serviceaccount snapshot-creator -n ansible-automation
```

**Step 2: Create RoleBindings for Each Namespace**

Create a RoleBinding for each namespace where you need snapshot permissions:

```bash
# Grant permissions in namespace 1
oc create rolebinding snapshot-creator-binding-ns1 \
  --clusterrole=edit \
  --serviceaccount=ansible-automation:snapshot-creator \
  -n <namespace-1>

# Grant permissions in namespace 2
oc create rolebinding snapshot-creator-binding-ns2 \
  --clusterrole=edit \
  --serviceaccount=ansible-automation:snapshot-creator \
  -n <namespace-2>

# Grant permissions in namespace 3
oc create rolebinding snapshot-creator-binding-ns3 \
  --clusterrole=edit \
  --serviceaccount=ansible-automation:snapshot-creator \
  -n <namespace-3>
```

**Example:**
```bash
# Create service account
oc create namespace ansible-automation
oc create serviceaccount snapshot-creator -n ansible-automation

# Grant permissions in multiple namespaces
oc create rolebinding snapshot-creator-vm-postbuild \
  --clusterrole=edit \
  --serviceaccount=ansible-automation:snapshot-creator \
  -n vm-postbuild-config

oc create rolebinding snapshot-creator-vm-production \
  --clusterrole=edit \
  --serviceaccount=ansible-automation:snapshot-creator \
  -n vm-production

oc create rolebinding snapshot-creator-vm-staging \
  --clusterrole=edit \
  --serviceaccount=ansible-automation:snapshot-creator \
  -n vm-staging
```

**Bulk Creation Script:**
```bash
#!/bin/bash
SERVICE_ACCOUNT_NS="ansible-automation"
SERVICE_ACCOUNT="snapshot-creator"
NAMESPACES=("vm-postbuild-config" "vm-production" "vm-staging" "vm-development")

for namespace in "${NAMESPACES[@]}"; do
  oc create rolebinding snapshot-creator-${namespace//-/_} \
    --clusterrole=edit \
    --serviceaccount=${SERVICE_ACCOUNT_NS}:${SERVICE_ACCOUNT} \
    -n ${namespace}
done
```

**Verify:**
```bash
# List all RoleBindings for the service account
oc get rolebinding -A -o json | \
  jq -r '.items[] | select(.subjects[]?.name=="snapshot-creator") | "\(.metadata.namespace) \(.metadata.name)"'

# Test permissions in each namespace
for ns in vm-postbuild-config vm-production vm-staging; do
  echo "Testing $ns:"
  oc auth can-i create virtualmachinesnapshots \
    --as=system:serviceaccount:ansible-automation:snapshot-creator \
    -n $ns
done
```

---

#### Option 3: All Namespaces Configuration (Cluster-Wide)

Use this when you need to create snapshots in any namespace across the cluster. This is the simplest setup but provides the broadest access.

**Step 1: Create Service Account**

Create the service account in a dedicated namespace:

```bash
oc create namespace ansible-automation
oc create serviceaccount snapshot-creator -n ansible-automation
```

**Step 2: Create ClusterRoleBinding**

Create a single ClusterRoleBinding to grant permissions across all namespaces:

```bash
oc create clusterrolebinding snapshot-creator-cluster-binding \
  --clusterrole=edit \
  --serviceaccount=ansible-automation:snapshot-creator
```

**Example:**
```bash
# Create service account
oc create namespace ansible-automation
oc create serviceaccount snapshot-creator -n ansible-automation

# Grant cluster-wide permissions
oc create clusterrolebinding snapshot-creator-cluster-binding \
  --clusterrole=edit \
  --serviceaccount=ansible-automation:snapshot-creator
```

**YAML Alternative:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: snapshot-creator-cluster-binding
subjects:
- kind: ServiceAccount
  name: snapshot-creator
  namespace: ansible-automation
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

**Verify:**
```bash
# Check ClusterRoleBinding exists
oc get clusterrolebinding snapshot-creator-cluster-binding

# Test permissions in multiple namespaces
oc auth can-i get virtualmachines \
  --as=system:serviceaccount:ansible-automation:snapshot-creator \
  -n <any-namespace>

oc auth can-i create virtualmachinesnapshots \
  --as=system:serviceaccount:ansible-automation:snapshot-creator \
  -n <any-namespace>
```

---

#### Option 4: Custom Minimal Permissions (Most Secure)

For enhanced security, create a custom ClusterRole with only the minimum required permissions instead of using the `edit` role.

**Step 1: Create Service Account**

```bash
oc create namespace ansible-automation
oc create serviceaccount snapshot-creator -n ansible-automation
```

**Step 2: Create Custom ClusterRole**

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

**Step 3: Create ClusterRoleBinding or RoleBinding**

For all namespaces:
```bash
oc create clusterrolebinding snapshot-creator-custom-binding \
  --clusterrole=kubevirt-snapshot-creator \
  --serviceaccount=ansible-automation:snapshot-creator
```

For a single namespace:
```bash
oc create rolebinding snapshot-creator-custom-binding \
  --clusterrole=kubevirt-snapshot-creator \
  --serviceaccount=ansible-automation:snapshot-creator \
  -n <target-namespace>
```

For multiple namespaces, create a RoleBinding in each namespace using the custom ClusterRole.

---

#### Migrating Between Configurations

**From Single Namespace to Multiple Namespaces:**

Add additional RoleBindings for each new namespace:

```bash
# Existing RoleBinding in namespace-1 (keep this)
# Add new RoleBinding for namespace-2
oc create rolebinding snapshot-creator-binding-ns2 \
  --clusterrole=edit \
  --serviceaccount=ansible-automation:snapshot-creator \
  -n <namespace-2>
```

**From Single/Multiple Namespaces to All Namespaces:**

1. Create the ClusterRoleBinding:
   ```bash
   oc create clusterrolebinding snapshot-creator-cluster-binding \
     --clusterrole=edit \
     --serviceaccount=ansible-automation:snapshot-creator
   ```

2. Verify it works across all namespaces:
   ```bash
   oc auth can-i create virtualmachinesnapshots \
     --as=system:serviceaccount:ansible-automation:snapshot-creator \
     -n <test-namespace>
   ```

3. (Optional) Remove old namespace-scoped RoleBindings:
   ```bash
   # List all RoleBindings for the service account
   oc get rolebinding -A -o json | \
     jq -r '.items[] | select(.subjects[]?.name=="snapshot-creator") | "\(.metadata.namespace) \(.metadata.name)"'
   
   # Delete each one (replace with actual namespace and name)
   oc delete rolebinding <rolebinding-name> -n <namespace>
   ```

**From All Namespaces to Multiple Namespaces:**

1. Delete the ClusterRoleBinding:
   ```bash
   oc delete clusterrolebinding snapshot-creator-cluster-binding
   ```

2. Create RoleBindings for each required namespace:
   ```bash
   for ns in namespace-1 namespace-2 namespace-3; do
     oc create rolebinding snapshot-creator-${ns} \
       --clusterrole=edit \
       --serviceaccount=ansible-automation:snapshot-creator \
       -n ${ns}
   done
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

After configuring your service account, verify that it has the required permissions in the target namespace(s).

**For Single Namespace Configuration:**

```bash
# Replace with your actual service account namespace and target namespace
SERVICE_ACCOUNT_NS="ansible-automation"
TARGET_NS="vm-postbuild-config"

# Test VM access
oc auth can-i get virtualmachines \
  --as=system:serviceaccount:${SERVICE_ACCOUNT_NS}:snapshot-creator \
  -n ${TARGET_NS}

# Test snapshot creation
oc auth can-i create virtualmachinesnapshots \
  --as=system:serviceaccount:${SERVICE_ACCOUNT_NS}:snapshot-creator \
  -n ${TARGET_NS}

# Both commands should return `yes`
```

**For Multiple Namespaces Configuration:**

```bash
SERVICE_ACCOUNT_NS="ansible-automation"
NAMESPACES=("vm-postbuild-config" "vm-production" "vm-staging")

for ns in "${NAMESPACES[@]}"; do
  echo "Testing permissions in namespace: ${ns}"
  
  # Test VM access
  oc auth can-i get virtualmachines \
    --as=system:serviceaccount:${SERVICE_ACCOUNT_NS}:snapshot-creator \
    -n ${ns}
  
  # Test snapshot creation
  oc auth can-i create virtualmachinesnapshots \
    --as=system:serviceaccount:${SERVICE_ACCOUNT_NS}:snapshot-creator \
    -n ${ns}
  
  echo "---"
done
```

**For All Namespaces (Cluster-Wide) Configuration:**

```bash
SERVICE_ACCOUNT_NS="ansible-automation"

# Test in a specific namespace
oc auth can-i get virtualmachines \
  --as=system:serviceaccount:${SERVICE_ACCOUNT_NS}:snapshot-creator \
  -n <any-namespace>

oc auth can-i create virtualmachinesnapshots \
  --as=system:serviceaccount:${SERVICE_ACCOUNT_NS}:snapshot-creator \
  -n <any-namespace>

# Verify ClusterRoleBinding exists
oc get clusterrolebinding snapshot-creator-cluster-binding

# List all permissions granted to the service account
oc describe clusterrolebinding snapshot-creator-cluster-binding
```

**Troubleshooting Permission Issues:**

If the `oc auth can-i` commands return `no`, check the following:

1. **Verify RoleBinding/ClusterRoleBinding exists:**
   ```bash
   # For namespace-scoped
   oc get rolebinding -n <namespace> | grep snapshot-creator
   
   # For cluster-wide
   oc get clusterrolebinding | grep snapshot-creator
   ```

2. **Check RoleBinding/ClusterRoleBinding details:**
   ```bash
   # For namespace-scoped
   oc get rolebinding <binding-name> -n <namespace> -o yaml
   
   # For cluster-wide
   oc get clusterrolebinding <binding-name> -o yaml
   ```

3. **Verify Service Account exists:**
   ```bash
   oc get serviceaccount snapshot-creator -n <service-account-namespace>
   ```

4. **Check for typos in namespace or service account name:**
   - Ensure the service account namespace matches in all commands
   - Ensure the target namespace is correct
   - Verify the service account name is exactly `snapshot-creator` (or your custom name)

5. **For multiple namespaces, verify RoleBinding exists in each namespace:**
   ```bash
   # List all RoleBindings for the service account across all namespaces
   oc get rolebinding -A -o json | \
     jq -r '.items[] | select(.subjects[]?.name=="snapshot-creator") | "\(.metadata.namespace) \(.metadata.name)"'
   ```

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

