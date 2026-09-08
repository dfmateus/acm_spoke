<!-- Copyright (C) 2025-2026 Diego Felipe Mateus <dfmateus@hotmail.com> -->
<!-- SPDX-License-Identifier: GPL-3.0-or-later -->

# acm_spoke_hub_rbac_setup

Provisions two ServiceAccounts and their RBAC on the ACM Hub required by the
[acm_spoke_token_setup](../acm_spoke_token_setup) (write) and
[acm_spoke_token_resolver](../acm_spoke_token_resolver) (read-only) roles. The
provisioner SA can create ManagedServiceAccount, ManifestWork, and enable addons. The
reader SA can only read MSA Secrets and validate preflight checks. Both SAs get long-lived
token Secrets (`kubernetes.io/service-account-token`). The role also discovers
ManagedClusters imported into the Hub and exposes them as an output fact. Idempotent: safe
to re-run on a Hub that already has the SAs provisioned.

## Requirements

- Ansible >= 2.16
- `kubernetes.core` collection
- The token provided in `acm_spoke_hub_rbac_setup_hub_token` must belong to a user or SA
  with `cluster-admin` on the Hub
- RHACM must be installed on the Hub (the role asserts the ACM namespace exists)

## Role Variables

All variables are declared in [defaults/main.yml](defaults/main.yml).

| Variable | Required | Default | Description |
|---|---|---|---|
| `acm_spoke_hub_rbac_setup_hub_url` | Yes | `""` | API URL of the ACM Hub (e.g., `https://api.hub.example.com:6443`). |
| `acm_spoke_hub_rbac_setup_hub_token` | Yes | `""` | Bearer token for a user with `cluster-admin` on the Hub. Can be kubeadmin, an existing SA with `cluster-admin`, or any identity with permission to create SA, ClusterRole, and ClusterRoleBinding. |
| `acm_spoke_hub_rbac_setup_validate_certs` | No | `true` | Validate TLS certificates when connecting to the Hub API. |
| `acm_spoke_hub_rbac_setup_namespace` | No | `"open-cluster-management"` | Namespace where the SAs are created. Must be the primary ACM Hub namespace where the MCE/ACM CRDs are installed. |
| `acm_spoke_hub_rbac_setup_provisioner_sa_name` | No | `"acm-spoke-provisioner"` | Name of the write SA used by the `acm_spoke_token_setup` role. |
| `acm_spoke_hub_rbac_setup_provisioner_clusterrole_name` | No | `"acm-spoke-provisioner"` | Name of the ClusterRole granting write permissions (create/update on MSA, ManifestWork, ManagedClusterAddOn). |
| `acm_spoke_hub_rbac_setup_provisioner_token_secret_name` | No | `"acm-spoke-provisioner-token"` | Name of the long-lived token Secret for the provisioner SA. |
| `acm_spoke_hub_rbac_setup_reader_sa_name` | No | `"acm-spoke-reader"` | Name of the read-only SA used by the `acm_spoke_token_resolver` role. |
| `acm_spoke_hub_rbac_setup_reader_clusterrole_name` | No | `"acm-spoke-reader"` | Name of the ClusterRole granting read-only permissions (get/list on ManagedCluster, ManagedClusterAddOn, ManagedServiceAccount, Secret, ManifestWork). |
| `acm_spoke_hub_rbac_setup_reader_token_secret_name` | No | `"acm-spoke-reader-token"` | Name of the long-lived token Secret for the reader SA. |
| `acm_spoke_hub_rbac_setup_msa_secret_name` | No | `"acm-spoke-automation"` | Name of the MSA Secret the reader SA is granted permission to read. Must match the `msa_name` used by `acm_spoke_token_resolver` and `acm_spoke_token_setup`. |
| `acm_spoke_hub_rbac_setup_no_log` | No | `true` | Mask sensitive data (tokens) in job output. Falls back to `var_no_log` if defined. Set to `false` only for Hub login troubleshooting. |
| `acm_spoke_hub_rbac_setup_separator_char` | No | `"*"` | Character used in visual separator blocks in log output. |
| `acm_spoke_hub_rbac_setup_default_message_padding` | No | `4` | Padding added to visual separator blocks for readability. |

## Output Facts

The role registers the following facts for downstream use:

| Fact | Description |
|---|---|
| `acm_spoke_hub_rbac_setup_provisioner_token` | Bearer token for the provisioner SA. Use this token with the `acm_spoke_token_setup` role. |
| `acm_spoke_hub_rbac_setup_reader_token` | Bearer token for the reader SA. Use this token with the `acm_spoke_token_resolver` role. |
| `acm_spoke_hub_rbac_setup_discovered_clusters` | List of ManagedCluster names imported into the Hub (excludes `local-cluster`). |

## Example Playbook

```yaml
- name: "Provision RBAC on ACM Hub"
  hosts: localhost
  connection: local
  tasks:
    - name: "Create SAs and RBAC for spoke automation"
      ansible.builtin.include_role:
        name: dfmateus.acm_spoke.acm_spoke_hub_rbac_setup
      vars:
        acm_spoke_hub_rbac_setup_hub_url: "https://api.hub.example.com:6443"
        acm_spoke_hub_rbac_setup_hub_token: "{{ hub_cluster_admin_token }}"
        acm_spoke_hub_rbac_setup_validate_certs: true
```

## License

GPL-3.0-or-later

## Author

Diego Felipe Mateus
