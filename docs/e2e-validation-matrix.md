# 🧪 E2E Validation Matrix

End-to-end validation results for the `acm_spoke_token_setup` and `acm_spoke_token_resolver` roles against a real RHACM environment.

## ⚙️ Environment

| Item | Value |
|---|---|
| Hub cluster | `hub-01` (ARO, East US) |
| Hub API URL | `https://api.hub-01.example.com:6443` |
| ACM version | RHACM 2.12 |
| OCP version | 4.20 (Hub) |
| Ansible core | 2.16.x (EE), 2.18.x (CLI) |
| EE image | `acm-spoke-token-resolver-ee:1.0.0` |
| Date | 2026-09-03 |

### 🖥️ Spoke clusters

| Cluster | Platform | API port | Kubernetes distribution | Vendor label |
|---|---|---|---|---|
| `spoke-aks-01` | AKS (Azure) | 443 | Managed Kubernetes | `Kubernetes` |
| `spoke-aro-classic-01` | ARO Classic (Azure) | 6443 | OpenShift | `OpenShift` |
| `spoke-aro-hcp-01` | ARO HCP (Azure) | 443 | OpenShift | `OpenShift` |

### 🔑 Service Accounts (dedicated to this project)

| SA | Where it lives | Namespace | Purpose | Used by |
|---|---|---|---|---|
| `acm-spoke-provisioner` | Hub | `open-cluster-management` | Write: create MSA, ManifestWork, Addon | `acm_spoke_token_setup` |
| `acm-spoke-reader` | Hub | `open-cluster-management` | Read-only: get MSA Secret, MC, Addon. **Has no permissions on the spokes.** | `acm_spoke_token_resolver` |
| `acm-spoke-automation` | Spoke | `open-cluster-management-agent-addon` | Temporary token with `cluster-admin` on the spoke (configurable). Created by the klusterlet via MSA. | Token read by `acm_spoke_token_resolver` |

## 📊 Test Results

### 🔧 Setup role (`acm_spoke_token_setup`)

| # | Test | Cluster | Exit | Changed | Result | Notes |
|---|---|---|---|---|---|---|
| 1 | Setup (first run) | `spoke-aks-01` | 0 | 3 | ✅ PASS | Login + MSA CR + ManifestWork created |
| 2 | Setup (first run) | `spoke-aro-classic-01` | 0 | 3 | ✅ PASS | Login + MSA CR + ManifestWork created |
| 3 | Idempotency (re-run) | `spoke-aks-01` | 0 | 1 | ✅ PASS | Only hub login changed (token refresh) |
| 4 | Idempotency (re-run) | `spoke-aro-classic-01` | 0 | 1 | ✅ PASS | Only hub login changed (token refresh) |
| 5 | Multi-cluster | Both (AKS + ARO) | 0 | 2 | ✅ PASS | Both processed in loop, 2 hub logins |

### 🔎 Resolver role (`acm_spoke_token_resolver`)

| # | Test | Cluster | Exit | Changed | Result | Notes |
|---|---|---|---|---|---|---|
| 6 | Resolve token | `spoke-aks-01` | 0 | 1 | ✅ PASS | Requires `validate_certs: false` (AKS uses internal CA) |
| 7 | Resolve token | `spoke-aro-classic-01` | 0 | 1 | ✅ PASS | Works with `validate_certs: true` (ARO uses public CA) |
| 8 | Negative (no MSA) | `local-cluster` | 0 | 1 | ✅ PASS | Fails at preflight 02.4 with actionable message |

### 🔗 Integration test: must-gather via ACM resolver

Real-world validation using `infra.support_assist.ocp_must_gather` as downstream automation. The resolver provides the spoke token, and the must-gather role uses it to collect diagnostic data from the spoke cluster without needing direct cluster credentials.

| # | Test | Cluster | Exit | Changed | Result | Notes |
|---|---|---|---|---|---|---|
| 9 | Must-gather via resolver | `spoke-aro-classic-01` | 0 | 6 | ✅ PASS | 230 MB archive, OCP 4.20.15, 98 tasks ok |

**Downstream collection:** [infra.support_assist](https://github.com/redhat-cop/infra.support_assist) (role `ocp_must_gather`)

**Flow:**

1. `acm_spoke_token_resolver` authenticates to the Hub and resolves the spoke token
2. The resolved facts (`spoke_token` + `spoke_api_url`) are mapped to `ocp_must_gather_token` and `ocp_must_gather_server_url`
3. `infra.support_assist.ocp_must_gather` runs `oc adm must-gather` against the spoke
4. Output: `must-gather-NoneDEFAULT_aroclassic01_2026-09-03-000724.tar.gz` (230 MB)

> ✅ **Result:** This test validates the core value proposition: any automation that needs spoke access can consume the resolver's output facts without managing its own cluster credentials.

---

## ▶️ Phase 2: Execution via AAP

End-to-end validation via AAP Controller hosted on `spoke-aro-classic-01`.

### ⚙️ AAP environment

| Item | Value |
|---|---|
| AAP version | 2.7 (Gateway architecture) |
| AAP URL | `https://aap.apps.hub-01.example.com` |
| Organization | ACM Spoke |
| EE | EE - ACM Spoke 1.0.0 (internal OCP registry) |
| Project | ACM Spoke Collection (main), bare repo sync |

### 📦 AAP resources created

| Resource | Name | ID |
|---|---|---|
| Organization | ACM Spoke | 3 |
| Inventory | ACM Spoke - LAB | 4 |
| EE | EE - ACM Spoke 1.0.0 | 6 |
| Credential Type | ACM Spoke - Hub Admin | 44 |
| Credential Type | ACM Spoke - Hub Reader | 45 |
| Credential | ACM Spoke - Hub Admin (kubeadmin) | 37 |
| Credential | ACM Spoke - Hub Reader (acm-spoke-reader) | 39 |
| Project | ACM Spoke Collection (main) | 31 |
| JT | JT - ACM Spoke - LAB - Bootstrap Hub + Spoke | 32 |
| JT | JT - ACM Spoke - LAB - Resolve Spoke Access | 33 |

### 🚀 Bootstrap via AAP

| # | Test | AAP Job | Status | ok/changed | Result | Notes |
|---|---|---|---|---|---|---|
| 10 | Bootstrap (clean Hub) | 404 | successful | 219/17 | ✅ PASS | 3 spokes auto-discovered and provisioned |
| 11 | Bootstrap (idempotent re-run) | 415 | successful | 225/7 | ✅ PASS | Only oc login changed; TLS auto-detect correct |

### 🔎 Resolver via AAP (with TLS auto-detection)

| # | Test | Cluster | AAP Job | Vendor | TLS auto | Result | Notes |
|---|---|---|---|---|---|---|---|
| 12 | Resolve token | `spoke-aks-01` | 411 | `Kubernetes` | `disabled` | ✅ PASS | Auto-detected: AKS -> false |
| 13 | Resolve token | `spoke-aro-classic-01` | 413 | `OpenShift` | `enabled` | ✅ PASS | Auto-detected: ARO -> true |
| 14 | Resolve token | `spoke-aro-hcp-01` | 414 | `OpenShift` | `enabled` | ✅ PASS | Auto-detected: ARO HCP -> true |

### ❌ Error and override tests

| # | Test | AAP Job | Result | Notes |
|---|---|---|---|---|
| 15 | Spoke not found | 416 | ✅ PASS (expected failure) | Preflight 02.1 fails with actionable message in AAP output |
| 16 | Manual override (force TLS=true on AKS) | 417 | ✅ PASS (expected failure) | Override respected: `mode=True`, fails with HTTP -1 as expected |

---

## 📋 Observations

### 🛡️ TLS auto-detection from ManagedCluster vendor label

The resolver auto-detects spoke TLS validation from the `vendor` label on the ManagedCluster CR:

| Vendor label | Platform examples | TLS validation | Reason |
|---|---|---|---|
| `OpenShift` | OCP, ARO, ROSA | `true` | Publicly trusted CA certificates |
| `Kubernetes` | AKS, EKS, GKE, IKS | `false` | Per-cluster internal CA, not present in the EE trust store |

> 🔎 **Note:** The `acm_spoke_token_resolver_spoke_validate_certs` variable defaults to `"auto"`. Override with `true` or `false` for special cases. The resolved value is exported as the output fact `acm_spoke_token_resolver_spoke_validate_certs` for downstream automation.

### 🔄 Idempotency

All three roles are fully idempotent:

- 🔧 **Bootstrap re-run:** `changed=7` (only `oc login` token refreshes). MSA, ManifestWork, SA, and RBAC resources are skipped when they already exist.
- ➕ **New spoke added:** When a new ManagedCluster is imported on the Hub, re-running the bootstrap auto-discovers and provisions only the new spoke. Existing spokes remain untouched.
- 🔎 **Resolver re-run:** `changed=1` (only Hub login). Read-only, no side effects.

### ⚠️ Negative tests

The resolver produces actionable error messages at each preflight stage:

- ❌ **02.1 (MC not found):** Names the cluster and suggests `oc get managedclusters` to list available ones.
- ❌ **02.4 (MSA not provisioned):** Directs the user to run the bootstrap or the equivalent JT in AAP.
- ❌ **02.6 (ManifestWork missing):** Same pattern, context-agnostic for both local and AAP execution.

### 🔍 Auto-discovery

The `acm_spoke_hub_rbac_setup` role queries all ManagedCluster CRs on the Hub (excluding `local-cluster`), sorts them alphabetically, and exports the list as `acm_spoke_hub_rbac_setup_discovered_clusters`. When `target_clusters` is not provided, the `bootstrap_hub_spoke.yml` playbook uses this list for automated onboarding of all spokes without manual enumeration.

## ✅ Summary

> ✅ **Result:** All 16 tests passed in both execution modes (CLI and AAP).

The collection handles three spoke platforms (AKS, ARO Classic, ARO HCP) with automatic TLS detection. The bootstrap provides end-to-end provisioning with auto-discovery, and the resolver delivers read-only token resolution without manual configuration. The three-layered SA model (provisioner for Hub setup, reader for daily Hub operation, and `acm-spoke-automation` with a temporary token on the spoke) ensures proper permission separation. The integration test with `infra.support_assist.ocp_must_gather` validates the real-world use case: downstream automation consuming resolver facts without managing its own cluster credentials.
