# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| 1.0.x   | Yes       |

## Reporting a Vulnerability

If you discover a security vulnerability in this collection, **do not open a
public issue**. Instead, report it privately:

* **Email:** [dfmateus@hotmail.com](mailto:dfmateus@hotmail.com)
* **Subject line:** `[SECURITY] dfmateus.acm_spoke - <brief description>`

Include the following in your report:

1. Description of the vulnerability and its potential impact.
2. Steps to reproduce the issue.
3. Affected versions and roles (if known).
4. Any suggested fix or mitigation (optional).

## Response Timeline

* **Acknowledgment:** within 48 hours of receipt.
* **Assessment and fix:** within 14 days for confirmed vulnerabilities.
* **Disclosure:** coordinated with the reporter once a fix is available.

## Scope

This policy covers the Ansible roles, playbooks, plugins, templates, and
example scripts shipped in the `dfmateus.acm_spoke` collection. It does not cover
the upstream dependencies listed in `galaxy.yml` (report those to their
respective maintainers).

## Credential Handling

This collection never stores, logs, or transmits credentials in plain text.
All sensitive values are handled with `no_log: true` and read from environment
variables, Ansible Vault, or AAP credential injection. If you find a case where
a secret could be exposed, report it using the process above.
