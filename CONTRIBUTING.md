# Contributing

Thanks for considering a contribution. This document covers the development setup, code conventions, and the pull request workflow.

## Development setup

Clone the repository and install the toolchain:

```bash
# Python dependencies (use a virtualenv or pipx, never sudo pip)
pip install ansible-core ansible-lint yamllint molecule molecule-plugins

# Ansible collection dependencies
ansible-galaxy collection install -r collections/requirements.yml

# Markdown linter (requires Node.js)
npm install -g markdownlint-cli2

# Shell script static analysis (Fedora/RHEL)
sudo dnf install ShellCheck
```

## Code conventions

### FQCN on all modules

Every task uses fully qualified collection names. No bare module names, ever.

```yaml
# correct
- name: Get the ManagedCluster list
  kubernetes.core.k8s_info:
    kind: ManagedCluster

# wrong
- name: Get the ManagedCluster list
  k8s_info:
    kind: ManagedCluster
```

### Task naming

Tasks follow the **`"[STAGE NN - Name] | N.N.N) Description"`** pattern so that **`-v`** output and AAP job logs are scannable stage by stage:

```yaml
- name: "[STAGE 00 - Preflight] | 0.1.2) Validate required variables"
```

Name every task, block, and handler.

### Variable naming

Role variables are prefixed with the role name (**`acm_spoke_token_resolver_*`**, **`acm_spoke_token_setup_*`**). Numeric values carry an inline comment with the unit and purpose:

```yaml
acm_spoke_token_setup_msa_validity: "720h"  # 30 days, MSA token validity before klusterlet renewal
acm_spoke_hub_rbac_setup_default_message_padding: 4  # characters, padding in visual separator blocks
```

### Assert messages

Every **`ansible.builtin.assert`** uses both **`fail_msg`** and **`success_msg`** with the standard prefixes:

```yaml
fail_msg: "❌ FAILED! Oh. No! => k8s_target_context is empty; pass it with -e"
success_msg: "✅ PASSED! Oh. Yes! => target context resolved: {{ k8s_target_context }}"
```

Never omit **`success_msg`**.

### Credential handling

Set **`no_log: true`** on any task that touches tokens, client secrets, PATs, or kubeconfig content. This includes **`debug`** tasks used for troubleshooting.

### Status emojis

These are a project convention, not decoration:

| Symbol | Meaning |
|--------|---------|
| ✅ | Success |
| ❌ | Hard failure |
| ⚠️ | Non-blocking warning |
| 🔎 | Dry-run notice |
| 🛡️ | Safety-guard check |

## Running tests

At minimum, verify that:

- `ansible-lint` and `yamllint` pass with zero errors.
- `ansible-playbook --syntax-check` passes for all playbooks under `playbooks/`.
- The bootstrap playbook completes successfully against a Hub with at least one ManagedCluster.
- The resolver playbook resolves a spoke token and validates access.
- Re-running the bootstrap is idempotent (no unexpected changes).

## Commit messages

Use [Conventional Commits](https://www.conventionalcommits.org/) format:

```
feat|fix|docs|refactor|chore|test(scope): imperative summary
```

The body explains *why* the change was made, not a restatement of the diff. One logical change per commit.

Examples:

```
feat(token_resolver): add EKS distribution auto-detection

fix(token_setup): resolve race condition on MSA secret creation

docs(hub_rbac_setup): add Azure AD integration notes to README

chore: update kubernetes.core collection to 3.1.0
```

## Pull request process

1. One logical change per PR. Keep diffs small and reviewable.
2. All checks must pass (`ansible-lint`, `yamllint`, `--syntax-check`).
3. If you add new variables, declare them in the role's **`defaults/main.yml`** with an inline comment (unit + purpose) and document them in the role's **`README.md`**.
4. If you add stages or change behavior, update the role's README.
5. Never commit secrets, **`.env`** files, kubeconfig content, `*.tfstate`, or `*.retry` files.

## License

This project is licensed under GPL-3.0-or-later. By submitting a contribution, you agree to license your work under the same terms.
