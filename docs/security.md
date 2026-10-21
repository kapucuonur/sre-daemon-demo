# Security and privacy design

SRE Daemon reads logs and source code and can restart services. That makes it a target. This document lists what it can and cannot do, how that is enforced, and the limits we know about. Status: beta.

## Principles

1. **The operating system enforces permissions, not the model.** Everything the daemon may do is fixed by the installer through users, polkit, systemd sandboxing and a filtering Docker proxy.
2. **Model output never becomes a command.** Actions come from deterministic policy. The only automatic action is one restart of a watched service that is actually down.
3. **Evidence stays on the server by default.** Nothing is sent to an AI provider unless the customer adds their own key.

## Privilege boundaries

| Boundary | Enforcement | Verified by the end-to-end suite |
|---|---|---|
| Not root | Dedicated `sre-daemon` system user, hardened systemd unit (`NoNewPrivileges`, read-only system) | Runs as `sre-daemon` |
| Not in the `docker` group | Docker access only through the proxy socket | Not in `docker` group; real Docker socket denied |
| Restart only watched services | polkit rule listing the units chosen at install | Can restart a watched unit, cannot restart an unrelated one |
| Read code, not secrets | Project directories are mounted read-only into the service; the rest of `/home` is hidden; `.env` and key files are not readable | Can read project source, cannot read `.env` |
| Cannot modify itself | Code directory is root-owned | Cannot write to the install directory |

## Docker proxy

Membership in the `docker` group is equivalent to root: anyone who can create a container can mount the host filesystem. The daemon only needs to look at containers and restart them, so it talks to a small proxy instead of the Docker socket.

- **Allowed:** reading containers (list, inspect, logs, stats, events) and start / stop / restart of existing containers.
- **Refused:** creating containers, `exec`, builds, volumes, networks, plugins, swarm, and anything not on the allow list.
- **Strict request handling:** one request per connection, strict HTTP parsing, no path encoding tricks, minimum API version, no host configuration smuggled into start or update calls.
- **No secret leakage through inspect:** container inspect responses carry environment variable names only; values are stripped, because they are often passwords.

## Prompt injection

Logs and container output are attacker-controllable: a request path, a user agent or a form field can end up in a log line and then in a prompt.

- Evidence is passed to models as fenced data, explicitly marked as not instructions.
- Text that addresses an AI rather than describing an error is flagged.
- Traceback resolution only reads source files, so a log line cannot pull configuration files into a prompt.
- The decisive control is structural: model output is a message to a human, never a command. The worst outcome of a successful injection is a misleading diagnosis.

## Supply chain and updates

- **Licensed download.** The installer and updater require a valid license key.
- **Signed releases.** The checksum file of every release is signed with an ed25519 key whose private half never leaves the release machine. The public key is pinned at install time and carried from version to version; it is never taken from a download, so a compromised download server cannot sign its own update.
- **Hashed, pinned dependencies.** Python dependencies are installed from a lock file with hash checking and binary wheels only.
- **Safe unpacking.** Release archives may contain only regular files and directories inside the target directory.
- **Managed daily updates with rollback.** A new version is unpacked next to the current one and swapped in; if the daemon is not healthy shortly after, the previous version is restored. Customers can opt out.

## Audit log

Every action is appended to a JSON-lines log in which each entry includes the hash of the previous one. Editing, removing or reordering an entry breaks the chain and is detected on verification.

Known limit: the chain is not keyed. Someone with the daemon user's access could rewrite the whole chain or truncate its tail. To make that visible, the current head hash is also reported off the box in the daily digest.

## Privacy: what leaves the server

| Destination | What | When |
|---|---|---|
| Customer's own Telegram bot | Alerts, diagnoses, daily digest | Always (configured at install) |
| Hosted dashboard | Incident metadata (service, title, status), host metrics, usage counts | Always on a paid plan; logs and diagnosis details only if the customer opts in |
| Anthropic API | Evidence with secrets redacted | Only if the customer adds their own key for the Claude add-on |
| Any other AI provider | Nothing | Never |

The authoritative statement is the Privacy Policy at [sre.trihonor.com](https://sre.trihonor.com).

## Known limits

- The local model is small and can be wrong. This is mitigated by design (no model-driven actions), not eliminated.
- Secret redaction is pattern-based. It covers common formats but cannot guarantee that every secret in free-form log text is recognised. This matters only if the Claude add-on or dashboard sharing is enabled.
- The optional draft pull request feature includes a check that flags risky diffs (new dependencies, build or CI changes, shell or network code). It is a tripwire, not a guarantee; the real control is that a human reviews and merges. The feature is off by default.
- Kubernetes is not supported yet.

## Reporting a vulnerability

Please report security issues privately via the contact details on [sre.trihonor.com](https://sre.trihonor.com) rather than opening a public issue.
