# Feature Specification: Java Role

**Feature Branch**: `005-java-role`
**Created**: 2026-04-07
**Status**: Draft
**Input**: User description: "Create a java role: Install sdkman. Use sdkman to install the current LTS version of the Java SDK (temurin). Installation is per-user, iterating over desktop_user_names. To verify the installation, java should show the installed version and the Temurin identifier."

## Clarifications

### Session 2026-04-07

- Q: Which shell(s) should SDKMAN shell initialisation target? → A: Bash only. SDKMAN's default installer modifies `~/.bashrc`, which is the standard login shell on Ubuntu. No additional multi-shell support is needed (YAGNI).
- Q: How should the role handle a network failure when downloading SDKMAN or Java? → A: No retry logic. A network failure causes the task to fail immediately through Ansible's standard error handling. Retries are not warranted for a single-developer project (YAGNI).
- Q: Should the role support ARM64 as well as AMD64? → A: AMD64 (x86\_64) Ubuntu Linux only, consistent with the `android_studio` reference role. ARM64 support is not in scope for this feature.
- Q: Should a specific Temurin LTS version be pinned, or should `sdk install java` (SDKMAN default) be used? → A: Use `sdk install java` without version pinning. SDKMAN's default selects the current Temurin LTS. The idempotency guard checks `java -version` output for the string "Temurin" so re-runs on an already-installed system report no changes.
- Q: How should the role handle a partially installed SDKMAN (e.g., init script present but corrupted)? → A: No automated cleanup. The idempotency guard uses `creates: ~/.sdkman/bin/sdkman-init.sh`. A partial install where the init script is present but unusable will cause `sdk install java` to fail with a visible Ansible error, requiring manual intervention. Silent corruption is impossible.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Java Available for All Desktop Users (Priority: P1)

A system administrator applies the java role to a fresh desktop machine. After the
role completes, every user listed in the desktop user configuration can open a
terminal, run the `java` command, and see that a Temurin LTS Java version is
installed and ready to use.

**Why this priority**: Delivering a working Java runtime to every desktop user is
the core value of this feature. Without it, no other outcome is meaningful.

**Independent Test**: Apply the role to a machine with at least one configured
desktop user. Log in as that user and execute `java -version`. The output must
reference the Temurin distribution and display a version number.

**Acceptance Scenarios**:

1. **Given** a machine with no Java installed and one or more entries in the
   desktop user list, **When** the java role is applied, **Then** each listed
   user can run `java -version` and the output contains the word "Temurin" and a
   valid version string.
2. **Given** the role has been applied, **When** a desktop user opens a new
   terminal session, **Then** `java` resolves without any additional manual
   configuration.

---

### User Story 2 - Idempotent Re-application (Priority: P2)

A system administrator runs the java role a second time on a machine where Java
is already installed. The role completes without errors and reports no changes.

**Why this priority**: Idempotency is essential for safe, repeated playbook
execution. Without it, re-runs risk breaking existing installations or producing
inconsistent state.

**Independent Test**: Apply the role twice consecutively. The second run must
finish with zero changed tasks and zero failed tasks.

**Acceptance Scenarios**:

1. **Given** Java is already installed via the role for all desktop users,
   **When** the java role is applied again, **Then** the role reports no changes
   and exits successfully.
2. **Given** Java is already installed, **When** the role is applied, **Then**
   no duplicate installations or configuration conflicts occur.

---

### Edge Cases

- What happens when `desktop_user_names` is empty? The role MUST complete
  without errors and install nothing.
- What happens when one user in the list does not exist on the system? The role
  MUST fail with a clear error message for that user and not silently skip them.
- What happens if the version manager is partially installed (e.g., interrupted
  mid-install)? The idempotency guard (`creates: ~/.sdkman/bin/sdkman-init.sh`)
  skips re-installation only when the init script is present. If the init script
  is present but the SDKMAN installation is corrupted, the subsequent `sdk
  install java` step will fail with a visible Ansible error. No automated
  cleanup is provided; manual intervention is required in that case.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The role MUST install a Java version manager in the home directory
  of each user listed in the desktop user configuration.
- **FR-002**: The role MUST install the current LTS release of the Temurin Java
  distribution for each listed desktop user via the version manager, using
  `sdk install java` without a pinned version string (SDKMAN's default selection).
- **FR-003**: The role MUST be idempotent: applying it multiple times MUST
  produce the same result as applying it once, with no changes reported on
  subsequent runs when the installation is already complete. (measured by SC-002)
- **FR-004**: After the role completes, each configured desktop user MUST be
  able to invoke `java` in a new bash shell session without any additional
  manual steps. (Bash is the target shell; SDKMAN initialisation is added to
  `~/.bashrc` by the SDKMAN installer.)
- **FR-005**: Running `java -version` as any configured desktop user MUST
  produce output that identifies both the version number and the Temurin
  distribution.
- **FR-006**: The role MUST support any number of desktop users by iterating
  over the `desktop_user_names` list variable.
- **FR-007**: When `desktop_user_names` is empty, the role MUST complete
  successfully without attempting any installation.
- **FR-008**: When a user listed in `desktop_user_names` does not exist on the
  target system, the role MUST fail with a clear error message for that user and
  MUST NOT silently skip them.

### Constraints

- **CO-001**: The role targets AMD64 (x86\_64) Ubuntu Linux only, consistent
  with the `android_studio` reference role. ARM64 is not in scope.
- **CO-002**: Bash is the only supported shell for SDKMAN initialisation.
  SDKMAN's default installer appends initialisation code to `~/.bashrc`.
- **CO-003**: No pinned Java version. `sdk install java` installs SDKMAN's
  current default, which is the Temurin LTS release at the time of role
  execution.

### Key Entities

- **Desktop User**: A local system account whose name appears in
  `desktop_user_names`; the target of per-user Java installation.
- **Java Version Manager**: A per-user tool that downloads and manages Java SDK
  distributions; installed once per user into their home directory.
- **Temurin LTS Java Distribution**: The specific open-source Java SDK variant
  that MUST be installed; identifiable by the "Temurin" label in `java -version`
  output.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For every user in `desktop_user_names`, running `java -version`
  exits with code 0 and the output contains the string "Temurin".
- **SC-002**: Applying the role a second time on a system where the installation
  is complete results in zero changed tasks. (tests FR-003)
- **SC-003**: The role completes without errors on a clean machine for a list of
  one or more desktop users.
