# Feature Specification: Java Role

**Feature Branch**: `005-install-java-lts`
**Created**: 2026-04-07
**Status**: Clarified
**Input**: User description: "Create a java role: Install sdkman. Use sdkman to
install the current LTS Temurin JDK. Works on AMD64 and ARM64 Ubuntu Linux."

## User Scenarios & Testing

### User Story 1 - Install Java LTS on AMD64 (Priority: P1)

A developer provisions an AMD64 Ubuntu Linux machine using this Ansible project.
After the `java` role runs, the user can open a terminal and run `java -version`,
which shows the installed Temurin JDK version with the word "Temurin" in the
output.

**Why this priority**: This is the primary use case. A machine that cannot run
Java after provisioning has not delivered the core value of the role.

**Independent Test**: Provision a fresh AMD64 VM with the `java` role applied and
run `java -version` as the provisioned user. Confirm the output contains the
Java version number and "Temurin".

**Acceptance Scenarios**:

1. **Given** a freshly provisioned AMD64 machine with the `java` role applied,
   **When** the provisioned user runs `java -version`, **Then** the output
   contains the installed version number and the word "Temurin".
2. **Given** the `java` role is applied, **When** the provisioned user opens a
   new shell, **Then** `java` is on the user's PATH without any manual
   configuration.

---

### User Story 2 - Idempotent Re-runs (Priority: P2)

A developer runs the provisioning playbook multiple times against the same
machine. The `java` role must not break subsequent runs or re-install an
already-present sdkman or Java SDK installation.

**Why this priority**: Idempotency is a core Ansible principle and is required
for the role to be safely usable in recurring provisioning workflows.

**Independent Test**: Run the playbook twice on the same VM and confirm the
second run reports no `changed` tasks for the `java` role.

**Acceptance Scenarios**:

1. **Given** sdkman and Java are already installed by a previous playbook run,
   **When** the playbook runs again, **Then** all tasks in the `java` role
   report `ok` or `skipped`, never `changed`.

---

### User Story 3 - Works on ARM64 (Priority: P3)

A developer runs the provisioning playbook against an ARM64 Ubuntu Linux machine
(e.g., a Vagrant VM on Apple Silicon). The `java` role installs Java
successfully without errors.

**Why this priority**: The project's local test VMs run on ARM64 (Apple
Silicon). Unlike `android_studio`, sdkman and Temurin both provide native
ARM64 binaries, so this role can and should work on both architectures.

**Independent Test**: Run the playbook against an ARM64 Vagrant VM. Verify that
`java -version` shows Temurin after the role completes.

**Acceptance Scenarios**:

1. **Given** the target machine is ARM64, **When** the playbook runs, **Then**
   the `java` role installs sdkman and Java LTS without errors.
2. **Given** the target machine is ARM64, **When** the provisioned user runs
   `java -version`, **Then** the output contains the version number and
   "Temurin".

---

### Edge Cases

- What happens when `desktop_user_names` contains a user that does not yet
  exist on the system? The role documents that target users must already exist
  (prerequisite, not enforced).
- What happens if the sdkman installer URL is temporarily unreachable? The
  install task will fail with a network error. This is expected and acceptable.

## Requirements

- **R01**: The role MUST install sdkman into `/home/<user>/.sdkman` for each
  user in `desktop_user_names` by running
  `curl -s "https://get.sdkman.io" | bash`.
- **R02**: The sdkman installation MUST be idempotent: use `creates:` pointing
  to `/home/<user>/.sdkman/bin/sdkman-init.sh` so the installer does not run
  again.
- **R03**: The role MUST install the Temurin JDK via sdkman using a
  configurable version variable (default: `21.0.10-tem`).
- **R04**: The Java installation MUST be idempotent: use `creates:` pointing to
  `/home/<user>/.sdkman/candidates/java/<version>/` so `sdk install java` does
  not run again.
- **R05**: All tasks that invoke sdkman MUST source
  `/home/<user>/.sdkman/bin/sdkman-init.sh` in the same shell invocation and
  use `executable: /bin/bash`.
- **R06**: The role MUST set `SDKMAN_DIR` to `/home/{{ item }}/.sdkman`
  (explicit path, not `~`) in the environment for all sdkman tasks.
- **R07**: The role MUST work on both AMD64 and ARM64 Ubuntu Linux without
  architecture-specific guards.
- **R08**: The role MUST follow the file structure of `roles/android_studio`
  (tasks/main.yml, defaults/main.yml, meta/main.yml, README.md).
- **R09**: After the role runs, the provisioned user MUST have `java` on `PATH`
  in any new interactive shell (sdkman appends init lines to `~/.bashrc`
  automatically).
- **R10**: The role MUST include `#SPDX-License-Identifier: MIT-0` at the top
  of each YAML file.
- **R11**: The Java SDK version default MUST be documented in
  `defaults/main.yml` with a comment explaining how to update it.

## Technical Context

- **Platform**: Ubuntu Linux (AMD64 + ARM64)
- **Ansible version**: 2.19+
- **Collections**: `community.general` (already in `requirements.yml`)
- **sdkman install**: `shell: curl -s "https://get.sdkman.io" | bash` with
  `creates:` guard
- **Idempotency guard (sdkman)**: `creates: /home/{{ item }}/.sdkman/bin/sdkman-init.sh`
- **Idempotency guard (Java)**:
  `creates: /home/{{ item }}/.sdkman/candidates/java/{{ java_sdk_version }}/`
- **Java module**: `ansible.builtin.shell` with `executable: /bin/bash`
- **Java version default**: `21.0.10-tem` (Java 21 LTS Temurin)
- **Per-user pattern**: loop over `desktop_user_names`, same as
  `android_studio` role

## Out of Scope

- Installing multiple Java versions via sdkman
- Setting a system-wide JAVA_HOME (sdkman manages per-user via shell init)
- Configuring sdkman auto-updates or other sdkman settings
- Installing any sdkman candidates other than Java

## Open Questions

None.
