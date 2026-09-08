# Changelog

All notable changes to this collection will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this collection adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.1] - 2026-09-08

### Fixed

- README Quick Start code blocks rendering as inline text on Ansible Galaxy.
  Galaxy's Markdown renderer does not support fenced code blocks indented
  inside numbered lists.

### Added

- GitHub Actions CI workflow: yamllint, ansible-lint, ansible-test sanity
  (ansible-core 2.18 + 2.19), and playbook syntax-check on every PR.
- GitHub Actions release workflow: tag-triggered build, Galaxy publish, and
  GitHub Release creation.
- Branch protection ruleset on main: PR required, CI status checks required,
  force push and branch deletion blocked.

## [1.0.0] - 2026-09-03

### Added

- Role `acm_spoke_hub_rbac_setup` -- provisions ServiceAccounts and RBAC on the ACM Hub
  for the setup (write) and resolver (read-only) roles.
- Role `acm_spoke_token_setup` -- provisions ManagedServiceAccount CR, ManifestWork RBAC,
  and the MSA addon for each spoke cluster.
- Role `acm_spoke_token_resolver` -- resolves temporary spoke tokens from Hub MSA Secrets.
  Read-only, no side effects.
- Playbook `bootstrap_hub_spoke.yml` -- end-to-end orchestration: Hub RBAC setup,
  auto-discovery of ManagedClusters, spoke provisioning, and token validation.
- Playbook `resolve_spoke_access.yml` -- standalone spoke token resolution.
- Playbook `setup_hub_rbac.yml` -- standalone Hub RBAC setup.
- Playbook `setup_spoke_cluster.yml` -- standalone spoke provisioning.
- TLS auto-detection from ManagedCluster `vendor` label: OpenShift = `validate_certs: true`,
  Kubernetes = `validate_certs: false`.
- Three-layered SA model: `acm-spoke-provisioner` (write, setup) and `acm-spoke-reader`
  (read-only, resolver) on the Hub, with `acm-spoke-automation` (temporary token) on each spoke.
- Auto-discovery of ManagedClusters for fully automated spoke onboarding.
- Example RBAC manifests, Execution Environment definition, and configuration templates.
