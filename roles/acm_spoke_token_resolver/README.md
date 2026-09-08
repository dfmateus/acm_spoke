<!-- Copyright (C) 2025-2026 Diego Felipe Mateus <dfmateus@hotmail.com> -->
<!-- SPDX-License-Identifier: GPL-3.0-or-later -->

# acm_spoke_token_resolver

Resolves a temporary bearer token for any spoke/managed cluster registered on an RHACM
Hub, using the ManagedServiceAccount (MSA) API. Works with any Kubernetes distribution:
OpenShift, AKS, EKS, GKE, IKS, ROSA, ARO, or vanilla K8s, as long as the cluster is
imported as a ManagedCluster on the Hub with the klusterlet running. Instead of maintaining
a static credential per cluster, the automation authenticates once against the ACM Hub and
the role returns a `spoke_token` + `spoke_api_url` pair that any downstream automation can
consume. The role runs four stages: Hub login, preflight validation (cluster availability,
addon status, ManifestWork applied), MSA Secret read, and token validation against the
spoke API.

## Requirements

- Ansible >= 2.16
- `kubernetes.core` collection
- The MSA infrastructure must already exist on the Hub for the target cluster (provisioned
  by the [acm_spoke_token_setup](../acm_spoke_token_setup) role)
- The Hub token must belong to a read-only SA with get/list on ManagedCluster,
  ManagedClusterAddOn, ManagedServiceAccount, Secret, and ManifestWork (provisioned by the
  [acm_spoke_hub_rbac_setup](../acm_spoke_hub_rbac_setup) role)

## Role Variables

All variables are declared in [defaults/main.yml](defaults/main.yml).

| Variable | Required | Default | Description |
|---|---|---|---|
| `acm_spoke_token_resolver_hub_url` | Yes | `""` | API URL of the ACM Hub (e.g., `https://api.hub.example.com:6443`). In AAP, inject via a custom credential type. |
| `acm_spoke_token_resolver_hub_token` | Yes | `""` | Bearer token for Hub authentication. Token from a SA with read-only RBAC. The SA needs only get/list on five resource types: ManagedCluster, ManagedClusterAddOn, ManagedServiceAccount, Secret, and ManifestWork. |
| `acm_spoke_token_resolver_validate_certs` | No | `true` | Validate TLS certificates when connecting to the Hub API. |
| `acm_spoke_token_resolver_spoke_validate_certs` | No | `"auto"` | Validate TLS certificates when connecting to the spoke API. `"auto"` (default): auto-detects from the ManagedCluster CR `vendor` label. OpenShift (ARO, ROSA, OCP) = `true`; others (AKS, EKS, GKE, IKS, vanilla K8s) = `false`. Non-OpenShift platforms use internal CAs that are not in the EE trust store. Accepts manual override with `true` or `false`. |
| `acm_spoke_token_resolver_target_cluster` | Yes | `""` | ManagedCluster name on the Hub (OpenShift, AKS, EKS, GKE, IKS, etc.). Corresponds to the Hub namespace where the MSA Secret and ManifestWork reside. |
| `acm_spoke_token_resolver_msa_name` | No | `"acm-spoke-automation"` | ManagedServiceAccount CR name on the Hub. All spokes use the same MSA name; the namespace (`target_cluster`) differentiates them. Change only if your organization uses a different naming convention. |
| `acm_spoke_token_resolver_manifestwork_name` | No | `"acm-spoke-automation-rbac"` | ManifestWork CR name that deploys RBAC (ClusterRoleBinding) on the spoke. The preflight check validates that this ManifestWork exists and has Applied status before attempting to use the token. |
| `acm_spoke_token_resolver_no_log` | No | `true` | Mask sensitive data (tokens, secrets) in job output. Falls back to `var_no_log` if defined. Set to `false` only for Hub login troubleshooting. |
| `acm_spoke_token_resolver_separator_char` | No | `"*"` | Character used in visual separator blocks in log output. |
| `acm_spoke_token_resolver_default_message_padding` | No | `4` | Padding added to visual separator blocks for readability. |

## Output Facts

The role registers the following facts for downstream use:

| Fact | Description |
|---|---|
| `acm_spoke_token_resolver_spoke_token` | Bearer token for the spoke cluster. Use with `kubernetes.core.k8s` or `k8s_info` as the `api_key` parameter. |
| `acm_spoke_token_resolver_spoke_api_url` | API URL of the spoke cluster, resolved from the ManagedCluster CR on the Hub. Use as the `host` parameter. |
| `acm_spoke_token_resolver_spoke_validate_certs` | Resolved TLS validation setting (bool). When `spoke_validate_certs` is set to `"auto"`, this fact contains the auto-detected value based on the spoke vendor label. Use as the `validate_certs` parameter. |

## Example Playbook

```yaml
- name: "Resolve spoke token and run downstream tasks"
  hosts: localhost
  connection: local
  tasks:
    - name: "Resolve token for spoke cluster"
      ansible.builtin.include_role:
        name: dfmateus.acm_spoke.acm_spoke_token_resolver
      vars:
        acm_spoke_token_resolver_hub_url: "https://api.hub.example.com:6443"
        acm_spoke_token_resolver_hub_token: "{{ hub_reader_token }}"
        acm_spoke_token_resolver_target_cluster: "aro-lab-spoke-01"

    - name: "Use the resolved token to list namespaces on the spoke"
      kubernetes.core.k8s_info:
        api_version: v1
        kind: Namespace
        host: "{{ acm_spoke_token_resolver_spoke_api_url }}"
        api_key: "{{ acm_spoke_token_resolver_spoke_token }}"
        validate_certs: "{{ acm_spoke_token_resolver_spoke_validate_certs }}"
      register: spoke_namespaces
```

## License

GPL-3.0-or-later

## Author

Diego Felipe Mateus
