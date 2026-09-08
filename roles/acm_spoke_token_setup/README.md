<!-- Copyright (C) 2025-2026 Diego Felipe Mateus <dfmateus@hotmail.com> -->
<!-- SPDX-License-Identifier: GPL-3.0-or-later -->

# acm_spoke_token_setup

Provisions a ManagedServiceAccount (MSA) and a ManifestWork that deploys a
ClusterRoleBinding on a target spoke cluster, granting the MSA ServiceAccount the
specified ClusterRole. After setup, the
[acm_spoke_token_resolver](../acm_spoke_token_resolver) role can resolve temporary bearer
tokens for the provisioned spoke. Works with any Kubernetes distribution managed by ACM:
OpenShift, AKS, EKS, GKE, IKS, ROSA, ARO, or vanilla K8s, as long as the cluster is
imported as a ManagedCluster on the Hub with the klusterlet running. The role enables the
`managed-serviceaccount` addon, creates the MSA CR with automatic token rotation, and
deploys the RBAC ManifestWork. Idempotent: safe to re-run on clusters that already have
the CRs.

## Requirements

- Ansible >= 2.16
- `kubernetes.core` collection
- The target cluster must be imported as a ManagedCluster on the Hub
- The Hub token must belong to a SA with write permissions (create/update on
  ManagedServiceAccount, ManifestWork, and ManagedClusterAddOn). The read-only SA used for
  token resolution is not sufficient. Use the provisioner SA created by the
  [acm_spoke_hub_rbac_setup](../acm_spoke_hub_rbac_setup) role.

## Role Variables

All variables are declared in [defaults/main.yml](defaults/main.yml).

| Variable | Required | Default | Description |
|---|---|---|---|
| `acm_spoke_token_setup_hub_url` | Yes | `""` | API URL of the ACM Hub (e.g., `https://api.hub.example.com:6443`). In AAP, inject via a custom credential type. |
| `acm_spoke_token_setup_hub_token` | Yes | `""` | Bearer token for Hub authentication. Token from a SA with write RBAC (create/update on MSA, ManifestWork, ManagedClusterAddOn). |
| `acm_spoke_token_setup_validate_certs` | No | `true` | Validate TLS certificates when connecting to the Hub API. |
| `acm_spoke_token_setup_target_cluster` | Yes | `""` | ManagedCluster name on the Hub (OpenShift, AKS, EKS, GKE, IKS, etc.). Corresponds to the namespace on the Hub where the MSA CR and ManifestWork are created. |
| `acm_spoke_token_setup_msa_name` | No | `"acm-spoke-automation"` | ManagedServiceAccount CR name on the Hub. All spokes use the same MSA name; the namespace (`target_cluster`) differentiates them. Must match `acm_spoke_token_resolver_msa_name` so the resolver can find the resources created by setup. |
| `acm_spoke_token_setup_manifestwork_name` | No | `"acm-spoke-automation-rbac"` | ManifestWork CR name that deploys a ClusterRoleBinding on the spoke cluster. |
| `acm_spoke_token_setup_spoke_cluster_role` | No | `"cluster-admin"` | ClusterRole to bind to the MSA ServiceAccount on the spoke. Defaults to `cluster-admin` for broad compatibility. Override for least-privilege if downstream automation needs fewer permissions. |
| `acm_spoke_token_setup_msa_validity` | No | `"720h"` | Validity period for the MSA token (30 days). The klusterlet renews the token automatically before expiration. Shorter values reduce the blast radius if the Hub Secret is compromised; longer values reduce API churn. |
| `acm_spoke_token_setup_no_log` | No | `true` | Mask sensitive data (tokens, secrets) in job output. Falls back to `var_no_log` if defined. Set to `false` only for Hub login troubleshooting. |
| `acm_spoke_token_setup_separator_char` | No | `"*"` | Character used in visual separator blocks in log output. |
| `acm_spoke_token_setup_default_message_padding` | No | `4` | Padding added to visual separator blocks for readability. |

## Output Facts

This role does not register output facts. After setup completes, use the
[acm_spoke_token_resolver](../acm_spoke_token_resolver) role to resolve the temporary
bearer token for the provisioned cluster.

## Example Playbook

```yaml
- name: "Provision MSA and RBAC for a spoke cluster"
  hosts: localhost
  connection: local
  tasks:
    - name: "Setup spoke token infrastructure"
      ansible.builtin.include_role:
        name: dfmateus.acm_spoke.acm_spoke_token_setup
      vars:
        acm_spoke_token_setup_hub_url: "https://api.hub.example.com:6443"
        acm_spoke_token_setup_hub_token: "{{ hub_provisioner_token }}"
        acm_spoke_token_setup_target_cluster: "aro-lab-spoke-01"
        acm_spoke_token_setup_msa_validity: "720h"  # 30 days
        acm_spoke_token_setup_spoke_cluster_role: "cluster-admin"
```

## License

GPL-3.0-or-later

## Author

Diego Felipe Mateus
