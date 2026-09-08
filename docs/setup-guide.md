# 📋 Setup guide

Everything that must be ready before using the `acm_spoke_token_resolver` role. Organized by phase (when) and responsible party (who).

---

## 👥 Responsibilities

| Actor                       | Responsibility                                                       |
| --------------------------- | -------------------------------------------------------------------- |
| 🛡️ ACM admin              | Hub SA + RBAC, MSA CRs, ManifestWork CRs                             |
| 🌐 Network / security admin | Firewall rules (Hub + spokes)                                        |
| 🔧 AAP admin                | Project, credential types, credentials, inventory, EE, job templates |

---

## 🚀 Automated approach (recommended)

The `bootstrap_hub_spoke.yml` playbook automates Phases 1 and 2 of this guide in a single run:

```bash
# Local (ansible-navigator):
ansible-navigator run playbooks/bootstrap_hub_spoke.yml \
  --mode stdout \
  --eei <registry>/acm-spoke-token-resolver-ee:1.0.0 \
  -e @.local/hub_vars_full.yml
```

> 🛡️ **Security:** Use the `.local/hub_vars_full.yml` variable file (gitignored) instead of `-e key=value` inline. Tokens on the command line are visible in shell history and the process list. See [usage.md section &#34;Prepare the variables file&#34;](usage.md#-prepare-the-variables-file) for the file format.

The bootstrap chains automatically: creates SAs + RBAC on the Hub (Phase 1.2-1.3), discovers all ManagedClusters, provisions MSA + ManifestWork on each spoke (Phase 2.1-2.2), and validates access to each spoke (Phase 2.3). Idempotent: re-running detects new spokes and provisions only the missing ones.

In AAP, create a JT with the Credential Type `ACM Spoke Bootstrap - Hub` that injects `acm_spoke_hub_rbac_setup_hub_url`, `acm_spoke_hub_rbac_setup_hub_token` and `acm_spoke_hub_rbac_setup_validate_certs`. See [usage.md section &#34;Credential Type: ACM Spoke Bootstrap - Hub&#34;](usage.md#credential-type-acm-spoke-bootstrap---hub) for the full definition.

> ⚠️ **Note:** Prerequisites 1.1 (MSA addon), 1.4 (firewall) and 1.5 (EE) remain manual and must be completed before bootstrap. The sections below detail each step for those who need to run them manually or understand what the bootstrap does.

---

## 🔧 Phase 1: Global setup (once per Hub)

### 1.1 🔌 Enable the managed-serviceaccount addon on the Hub

The MSA addon must be running before creating any MSA CR.

#### Option A: per cluster

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: managed-serviceaccount
  namespace: <managed-cluster-name>
spec:
  installNamespace: open-cluster-management-agent-addon
```

```bash
oc apply -f msa-addon.yml --context <hub-context>
```

#### Option B: global via MultiClusterEngine (recommended)

```yaml
apiVersion: multicluster.openshift.io/v1
kind: MultiClusterEngine
metadata:
  name: multiclusterengine
spec:
  overrides:
    components:
      - name: managed-serviceaccount
        enabled: true
```

Verification:

```bash
oc get ManagedClusterAddOn managed-serviceaccount -A --context <hub-context>
# Expected: Available = True for all target clusters
```

> ✅ **Result:** `Available = True` for all target clusters.

**Who:** ACM admin.

---

### 1.2 🛡️ Create the read-only ServiceAccount on the Hub

AAP authenticates to the Hub using a dedicated, read-only ServiceAccount. This SA does not have `cluster-admin` on the Hub. It can read exactly 5 resource types.

> 🔎 **Context: 3 layered SAs.** The collection uses three distinct SAs: **`acm-spoke-provisioner`** (Hub, write, setup), **`acm-spoke-reader`** (Hub, read-only, daily operation) and **`acm-spoke-automation`** (spoke, `cluster-admin`, temporary token). The SA created in this section is the **`acm-spoke-reader`**. It does NOT execute anything on spokes. The token it reads belongs to **`acm-spoke-automation`**, a different SA that lives on the spoke and is created automatically by the klusterlet in Phase 2. See [architecture.md](architecture.md) for the full access chain diagram.

Permissions for `acm-spoke-reader`:

| Resource                                            | API Group                                     | Verbs             | Reason                                           |
| --------------------------------------------------- | --------------------------------------------- | ----------------- | ------------------------------------------------ |
| 🔎`ManagedCluster`                                | `cluster.open-cluster-management.io`        | `get`, `list` | Spoke API URL and Available status               |
| 🔎`ManagedClusterAddOn`                           | `addon.open-cluster-management.io`          | `get`           | Verify that the MSA addon is active on the spoke |
| 🔎`ManagedServiceAccount`                         | `authentication.open-cluster-management.io` | `get`           | Verify that the MSA CR exists and is provisioned |
| 🔎`ManifestWork`                                  | `work.open-cluster-management.io`           | `get`           | Verify that spoke RBAC is applied                |
| 🛡️`Secret` (name=`acm-spoke-automation` only) | core                                          | `get`           | Read the MSA token with automatic rotation       |

**Read-only SA manifest (apply once on the Hub):**

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: acm-spoke-reader
  namespace: open-cluster-management
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: acm-spoke-reader
rules:
  # ManagedCluster is cluster-scoped. Required for API URL and status.
  - apiGroups: ["cluster.open-cluster-management.io"]
    resources: ["managedclusters"]
    verbs: ["get", "list"]

  # ManagedClusterAddOn: verify MSA addon is active.
  - apiGroups: ["addon.open-cluster-management.io"]
    resources: ["managedclusteraddons"]
    verbs: ["get"]

  # ManagedServiceAccount: verify the MSA CR exists.
  - apiGroups: ["authentication.open-cluster-management.io"]
    resources: ["managedserviceaccounts"]
    verbs: ["get"]

  # ManifestWork: verify spoke RBAC is applied.
  - apiGroups: ["work.open-cluster-management.io"]
    resources: ["manifestworks"]
    verbs: ["get"]

  # Secret: read the MSA token. Scoped by resourceNames to the MSA Secret only.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["acm-spoke-automation"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: acm-spoke-reader
subjects:
  - kind: ServiceAccount
    name: acm-spoke-reader
    namespace: open-cluster-management
roleRef:
  kind: ClusterRole
  name: acm-spoke-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
oc apply -f hub-sa-reader.yml --context <hub-context>
```

> 🔎 **Note:** If the MSA has a custom name (different from `acm-spoke-automation`), adjust the `resourceNames` field in the ClusterRole to match.

**Generate a long-lived token:**

```bash
# Create a token Secret (Kubernetes 1.24+)
cat <<EOF | oc apply -f - --context <hub-context>
apiVersion: v1
kind: Secret
metadata:
  name: acm-spoke-reader-token
  namespace: open-cluster-management
  annotations:
    kubernetes.io/service-account.name: acm-spoke-reader
type: kubernetes.io/service-account-token
EOF

# Extract the token (store in the AAP credential)
oc get secret acm-spoke-reader-token \
  -n open-cluster-management \
  --context <hub-context> \
  -o jsonpath='{.data.token}' | base64 -d && echo
```

**Verify that the SA has exactly the correct access:**

```bash
SA="system:serviceaccount:open-cluster-management:acm-spoke-reader"

# Should return "yes"
oc auth can-i get managedclusters --as="$SA"
oc auth can-i get secrets/acm-spoke-automation --as="$SA" -n <spoke-namespace>

# Should return "no"
oc auth can-i get secrets --as="$SA" -n <spoke-namespace>
oc auth can-i '*' '*' --as="$SA"
```

> ✅ **Result:** The first two commands return `yes`; the last two return `no`.

**Who:** ACM admin (Hub `cluster-admin` required only for the initial RBAC setup).

---

### 1.3 🔧 Provisioner SA (for the `acm_spoke_token_setup` role)

Used only by the `acm_spoke_token_setup` role to create MSA CRs, ManifestWorks and ManagedClusterAddOns. Does not need to be permanently registered in AAP. Use only during new cluster onboarding.

The full RBAC manifest is at [examples/rbac/acm-spoke-provisioner-rbac.yaml](../examples/rbac/acm-spoke-provisioner-rbac.yaml).

| Resource                    | API Group                                     | Verbs                                      |
| --------------------------- | --------------------------------------------- | ------------------------------------------ |
| 🔎`ManagedCluster`        | `cluster.open-cluster-management.io`        | `get`                                    |
| 🔧`ManagedClusterAddOn`   | `addon.open-cluster-management.io`          | `get`, `create`, `update`, `patch` |
| 🔧`ManagedServiceAccount` | `authentication.open-cluster-management.io` | `get`, `create`, `update`, `patch` |
| 🔧`ManifestWork`          | `work.open-cluster-management.io`           | `get`, `create`, `update`, `patch` |
| 🔎`Secret`                | core                                          | `get`, `list`                          |

```bash
oc apply -f examples/rbac/acm-spoke-provisioner-rbac.yaml --context <hub-context>

# Extract the token (store in the AAP credential or in .local/hub_vars_admin.yml)
oc get secret acm-spoke-provisioner-token \
  -n open-cluster-management \
  --context <hub-context> \
  -o jsonpath='{.data.token}' | base64 -d && echo
```

Alternatively, use the Hub with `cluster-admin` for the initial setup (simpler, but with more permissions than necessary). The setup is a one-time operation per cluster, so the risk is limited.

**Who:** ACM admin.

---

### 1.4 🌐 Open firewall rules

The role requires network access from the AAP EE to the Hub and spokes. Internet access is not required.

| # | Source    | Destination      | Port | Purpose                 |
| - | --------- | ---------------- | ---- | ----------------------- |
| 1 | 🔧 AAP EE | 🛡️ ACM Hub API | 6443 | Hub login + MSA reading |
| 2 | 🔧 AAP EE | 🌐 OCP spokes    | 6443 | Validation + automation |
| 3 | 🔧 AAP EE | 🌐 xKS spokes    | 443  | Validation + automation |

**Who:** Network / security admin.

See [architecture.md](architecture.md) for the full port matrix by platform.

---

### 1.5 📦 Build and register the Execution Environment

The EE must contain:

- `oc` CLI (OpenShift client)
- `kubernetes.core` Ansible collection (>= 3.0.0)

```bash
# Build from the repository root
ansible-builder build \
  -t acm-spoke-token-resolver-ee:1.0.0 \
  -f examples/ee/execution-environment.yml \
  --context .local/tmp/ee-build

# Clean up build artifacts
rm -rf .local/tmp/ee-build

# Push to registry (adjust the URL)
podman push acm-spoke-token-resolver-ee:1.0.0 <registry-url>/acm-spoke-token-resolver-ee:1.0.0
```

The EE definition is at [examples/ee/execution-environment.yml](../examples/ee/execution-environment.yml). It includes `kubernetes.core`, Python `kubernetes` and the `oc` CLI.

Register it in AAP under Administration > Execution Environments.

**Who:** AAP admin.

---

## 🔄 Phase 2: Per-cluster onboarding

Repeat these steps for each cluster that should be accessible by automation. All steps run on the Hub. Direct access to the spoke is not required.

> 🔑 **What happens in this phase:** the MSA CR (section 2.1) instructs the klusterlet to create the **`acm-spoke-automation`** SA inside the spoke, in the `open-cluster-management-agent-addon` namespace. The ManifestWork (section 2.2) grants permissions to that SA on the spoke (default: `cluster-admin`, configurable). After these two steps, the klusterlet reports the `acm-spoke-automation` token back to the Hub as a Secret. That Secret is what the **`acm-spoke-reader`** (created in Phase 1) reads during daily operation.

### 2.1 🔧 Create the ManagedServiceAccount CR on the Hub

```yaml
apiVersion: authentication.open-cluster-management.io/v1beta1
kind: ManagedServiceAccount
metadata:
  name: acm-spoke-automation
  namespace: <managed-cluster-name>
spec:
  rotation:
    enabled: true
    validity: 720h    # 30 days - adjust per security policy
```

```bash
oc apply -f msa-cr.yml --context <hub-context>
```

**TTL considerations:**

| TTL                 | Rotation frequency | Use case                                   |
| ------------------- | ------------------ | ------------------------------------------ |
| `168h` (7 days)   | Weekly             | 🛡️ High-security environments            |
| `720h` (30 days)  | Monthly            | ✅ Balance between security and operations |
| `2160h` (90 days) | Quarterly          | 🔄 Lower-risk environments                 |
| `8766h` (~1 year) | Annually           | ⚠️ Minimal management overhead           |

The klusterlet renews the token before expiration, so a valid token always exists in the Hub Secret.

**Who:** ACM admin.

---

### 2.2 🛡️ Create the ManifestWork RBAC on the Hub

The MSA creates the ServiceAccount on the spoke, but does NOT grant permissions. The ManifestWork deploys the required RBAC via the klusterlet.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: acm-spoke-automation-rbac
  namespace: <managed-cluster-name>
spec:
  workload:
    manifests:
      - apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: acm-spoke-automation-cluster-admin
        subjects:
          - kind: ServiceAccount
            name: acm-spoke-automation
            namespace: open-cluster-management-agent-addon
        roleRef:
          kind: ClusterRole
          name: cluster-admin
          apiGroup: rbac.authorization.k8s.io
```

```bash
oc apply -f manifestwork-rbac.yml --context <hub-context>
```

> 🛡️ **Security:** The SA has **`cluster-admin`** on the spoke, but the token is limited by the MSA TTL. If it leaks, it expires automatically. An infinite token, by comparison, requires manual revocation.

> 🔎 **Note:** If downstream automation does not need **`cluster-admin`**, replace it with a more restrictive ClusterRole (e.g., `view`, `edit` or a custom ClusterRole). Adjust the `acm_spoke_token_setup_spoke_cluster_role` variable in the setup role.

**Who:** ACM admin.

---

### 2.3 ✅ Verify that the spoke is ready

```bash
# MSA Secret exists on the Hub
oc get secret acm-spoke-automation \
  -n <managed-cluster-name> --context <hub-context>

# MSA reported the token
oc get managedserviceaccount acm-spoke-automation \
  -n <managed-cluster-name> --context <hub-context> \
  -o jsonpath='{.status.conditions}' | python3 -m json.tool

# ManifestWork is Applied
oc get manifestwork acm-spoke-automation-rbac \
  -n <managed-cluster-name> --context <hub-context> \
  -o jsonpath='{.status.conditions}' | python3 -m json.tool

# Verify spoke login with the MSA token
SPOKE_TOKEN=$(oc get secret acm-spoke-automation \
  -n <managed-cluster-name> --context <hub-context> \
  -o jsonpath='{.data.token}' | base64 -d)

SPOKE_URL=$(oc get managedcluster <managed-cluster-name> \
  --context <hub-context> \
  -o jsonpath='{.spec.managedClusterClientConfigs[0].url}')

oc login --token="${SPOKE_TOKEN}" --server="${SPOKE_URL}"
oc whoami
# Expected: system:serviceaccount:open-cluster-management-agent-addon:acm-spoke-automation

oc auth can-i '*' '*' --all-namespaces
# Expected: yes (cluster-admin via ManifestWork)
```

> ✅ **Result:** Spoke login works and `oc auth can-i '*' '*'` returns `yes`.

**Automated alternative:** Use `playbooks/bootstrap_hub_spoke.yml` to automate the full setup (Phase 1 + 2) including auto-discovery of all ManagedClusters. Or use `playbooks/setup_spoke_cluster.yml` for a single spoke. See [usage and examples](usage.md) for details.

**Who:** ACM admin.

---

### 2.4 🔄 Scale to multiple clusters

```bash
for CLUSTER in $(oc get managedclusters --context <hub-context> \
  -o jsonpath='{.items[*].metadata.name}'); do
  echo "Creating MSA + ManifestWork for ${CLUSTER}..."
  cat <<EOF | oc apply -f - --context <hub-context>
apiVersion: authentication.open-cluster-management.io/v1beta1
kind: ManagedServiceAccount
metadata:
  name: acm-spoke-automation
  namespace: ${CLUSTER}
spec:
  rotation:
    enabled: true
    validity: 720h
---
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: acm-spoke-automation-rbac
  namespace: ${CLUSTER}
spec:
  workload:
    manifests:
      - apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: acm-spoke-automation-cluster-admin
        subjects:
          - kind: ServiceAccount
            name: acm-spoke-automation
            namespace: open-cluster-management-agent-addon
        roleRef:
          kind: ClusterRole
          name: cluster-admin
          apiGroup: rbac.authorization.k8s.io
EOF
done
```

---

### 2.5 ❌ Access revocation

To revoke access for a specific cluster:

```bash
# Remove the MSA (klusterlet deletes the SA on the spoke)
oc delete managedserviceaccount acm-spoke-automation \
  -n <cluster-name> --context <hub-context>

# Remove the ManifestWork RBAC (klusterlet removes the ClusterRoleBinding)
oc delete manifestwork acm-spoke-automation-rbac \
  -n <cluster-name> --context <hub-context>
```

> 🛡️ **Security:** Both operations are centralized on the Hub. Revocation does not require direct access to the spoke.

---

## 🔄 Phase 3: Ongoing responsibilities

| Item                                | Frequency                                           | Automatic?                                                                                                          | Responsible              |
| ----------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| 🔄 MSA token rotation on each spoke | Per TTL (default 30 days)                           | ✅ Yes. The klusterlet renews before expiration. No intervention needed unless the klusterlet is degraded.          | ACM platform (automatic) |
| 🛡️ Hub SA token                   | Long-lived (SA token Secret, no default expiration) | ❌ No. Rotate per security policy. Delete and recreate the Secret; update the credential in AAP.                    | ACM admin                |
| 🔎 MSA addon health                 | Continuous                                          | ✅ No manual action for healthy clusters. If a cluster goes offline, the MSA rotates automatically on reconnection. | ACM platform             |

---

## ✅ Quick checklist

### 🔧 Global setup (Hub) - once

- [ ] 🔌 `managed-serviceaccount` addon enabled on the Hub (global or per cluster)
- [ ] 🛡️ SAs (`acm-spoke-reader` + `acm-spoke-provisioner`) created on the Hub (manually or via bootstrap)
- [ ] 🛡️ Hub SA bearer token generated and stored securely
- [ ] 🌐 Network path confirmed: AAP EE -> Hub API `:6443`
- [ ] 🌐 Network path confirmed: AAP EE -> OCP spokes `:6443`
- [ ] 🌐 Network path confirmed: AAP EE -> xKS spokes `:443`
- [ ] 📦 AAP EE built, pushed to registry and registered in AAP
- [ ] 🔧 AAP Project created and synced
- [ ] 🔧 AAP Credential Type `ACM Spoke Bootstrap - Hub` created (**`cluster-admin`**, for bootstrap)
- [ ] 🔧 AAP Credential Type `ACM Spoke Resolver - Hub` created (**`acm-spoke-reader`**, for daily operation)
- [ ] 🔧 AAP Credential Type `ACM Spoke Setup - Hub` created (**`acm-spoke-provisioner`**, for individual setup)
- [ ] 🔧 AAP Credential instances created (URL + SA token, one per type and environment)
- [ ] 🔧 AAP Inventory created
- [ ] 🔧 AAP Job Templates created (`JT - Bootstrap Hub + Spoke`, `JT - Resolve Spoke Access`, `JT - Setup Spoke`). See [usage.md](usage.md#job-templates) for the full definition.

### 🔄 Per cluster (automatic via bootstrap)

The `bootstrap_hub_spoke.yml` discovers ManagedClusters automatically and runs the steps below.
For manual verification:

- [ ] ✅ `ManagedServiceAccount` CR `acm-spoke-automation` created in the `<cluster-name>` namespace on the Hub
- [ ] ✅ `ManifestWork` CR `acm-spoke-automation-rbac` created in the `<cluster-name>` namespace on the Hub
- [ ] ✅ MSA Secret `acm-spoke-automation` visible in the `<cluster-name>` namespace on the Hub
- [ ] ✅ MSA conditions: `TokenReported = True`
- [ ] ✅ ManifestWork conditions: `Applied = True`
- [ ] ✅ Resolver runs successfully (TLS auto-detected via `vendor` label)

---

## 🔍 Troubleshooting

### ❌ MSA Secret does not appear on the Hub

1. Check klusterlet health: `oc get managedcluster <name> -o jsonpath='{.status.conditions}'`
2. Check addon status: `oc get ManagedClusterAddOn managed-serviceaccount -n <name>`
3. Check MSA conditions: `oc get managedserviceaccount acm-spoke-automation -n <name> -o yaml`

### ⚠️ Expired or invalid token

1. Check MSA rotation status. The klusterlet should renew automatically.
2. Delete and recreate the MSA if the klusterlet is stuck.
3. Check network connectivity between Hub and spoke (klusterlet).
