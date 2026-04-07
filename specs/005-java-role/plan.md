# Implementation Plan: Java Role

**Branch**: `005-java-role` | **Date**: 2026-04-07 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `specs/005-java-role/spec.md`

## Summary

Install SDKMAN per user into each home directory listed in `desktop_user_names`,
then use SDKMAN to install the current default Temurin LTS Java distribution.
The role follows the `android_studio` reference pattern: per-user iteration with
`become_user`, a `creates:` guard for SDKMAN installation idempotency, and a
shell-based Temurin detection check as the Java install idempotency guard.

## Technical Context

**Language/Version**: YAML (Ansible 2.19+)

**Primary Dependencies**: `community.general` collection (already in
`requirements.yml`); SDKMAN shell installer (fetched at runtime from
`https://get.sdkman.io`); Temurin JDK (fetched via SDKMAN at runtime)

**Storage**: N/A

**Testing**: Vagrant + Docker local VM (AMD64 Linux) per `CONTRIBUTING.md`;
`vagrant_docker` inventory; role isolated as the sole active role in
`configure-linux-roles.yml` during testing

**Target Platform**: AMD64 Ubuntu Linux

**Project Type**: Ansible role

**Performance Goals**: N/A

**Constraints**: AMD64 only; Bash only (SDKMAN modifies `~/.bashrc`); no Java
version pinning; no retry logic on network failure; interactive SDKMAN prompts
suppressed via the `SDKMAN_AUTO_ANSWER=true` environment variable

**Scale/Scope**: One role; five files; zero new collection dependencies

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Principle I — Idempotency (NON-NEGOTIABLE)

PASS. Two guards are applied:

- SDKMAN install: `creates: /home/{{ item }}/.sdkman/bin/sdkman-init.sh` — the
  task is skipped when the init script already exists.
- Java install: a preceding `ansible.builtin.shell` check task (with
  `changed_when: false` and `failed_when: false`) sources the SDKMAN init
  script and runs `java -version 2>&1`; the install task runs only `when:
  '"Temurin" not in item.stdout'`.

### Principle II — Role-First Organisation

PASS. New standalone role at `roles/java/`. The role has single responsibility
(install Java via SDKMAN). It will be added to `configure-linux-roles.yml`
with the `not-supported-on-vagrant-arm64` tag (same pattern as `android_studio`).

### Principle III — Test Locally Before Cloud

PASS. The role must be validated on a `vagrant_docker` VM (AMD64 Linux) with
the role isolated in `configure-linux-roles.yml` before any cloud deployment.

### Principle IV — Simplicity (YAGNI)

PASS. No version pinning, no retry logic, no ARM64 support, no multi-shell
support. The solution uses only `ansible.builtin.shell` tasks — no custom
modules or additional collections.

### Principle V — Conventional Commits

PASS. The implementation commit will use `feat: add java role`.

### Principle VI — Markdown Quality Standards

PASS. `README.md` and `DESIGN.md` will comply with ATX headings, blank-line
list rules, no trailing whitespace, single trailing newline.

### Post-design re-check

All principles still pass after the Phase 1 design below.

## Project Structure

### Documentation (this feature)

```text
specs/005-java-role/
├── plan.md       # This file
├── research.md   # Phase 0 output
├── spec.md       # Feature specification
└── tasks.md      # Phase 2 output (/speckit.tasks — NOT created here)
```

### Source Code (repository root)

```text
roles/java/
├── defaults/
│   └── main.yml     # Empty defaults (no version pins)
├── meta/
│   └── main.yml     # Galaxy metadata
├── tasks/
│   └── main.yml     # Install SDKMAN + Java per user
├── DESIGN.md        # Non-obvious decisions and constraints
└── README.md        # Requirements, variables, example playbook

configure-linux-roles.yml   # Add java role with not-supported-on-vagrant-arm64 tag
```

**Structure Decision**: Single Ansible role following the `android_studio`
reference pattern exactly. No sub-directories inside `tasks/` — the single
`main.yml` contains all three task groups (SDKMAN install, Java check, Java
install).

## Task Design

### Task Group 1 — Install SDKMAN per user

```yaml
- name: Install SDKMAN for {{ item }}
  ansible.builtin.shell:
    cmd: 'curl -s "https://get.sdkman.io" | bash'
    creates: "/home/{{ item }}/.sdkman/bin/sdkman-init.sh"
    executable: /bin/bash
  become_user: "{{ item }}"
  loop: "{{ desktop_user_names }}"
```

### Task Group 2 — Detect Temurin installation per user

```yaml
- name: Check if Temurin Java is installed for {{ item }}
  ansible.builtin.shell:
    cmd: 'source ~/.sdkman/bin/sdkman-init.sh && java -version 2>&1'
    executable: /bin/bash
  become_user: "{{ item }}"
  loop: "{{ desktop_user_names }}"
  register: java_version_check
  changed_when: false
  failed_when: false
```

### Task Group 3 — Install Temurin Java via SDKMAN

```yaml
- name: Install Temurin Java via SDKMAN for {{ item.item }}
  ansible.builtin.shell:
    cmd: 'source ~/.sdkman/bin/sdkman-init.sh && sdk install java'
    executable: /bin/bash
  environment:
    SDKMAN_AUTO_ANSWER: "true"
  become_user: "{{ item.item }}"
  loop: "{{ java_version_check.results }}"
  when: '"Temurin" not in item.stdout'
  changed_when: true
```

### Idempotency walkthrough

**First run (nothing installed)**:

1. SDKMAN install — init script absent → `creates:` not satisfied → task runs,
   SDKMAN installed.
2. Java check — init script now present; `java` not yet on PATH → `java -version`
   fails; rc ≠ 0 but `failed_when: false`; stdout does not contain "Temurin".
3. Java install — `when:` condition true → task runs, Temurin installed.

**Second run (SDKMAN and Temurin already present)**:

1. SDKMAN install — init script present → `creates:` satisfied → skipped, no
   change.
2. Java check — sources init script; `java -version 2>&1` succeeds; stdout
   contains "Temurin".
3. Java install — `when:` condition false → skipped, no change. Zero changed
   tasks.

**Empty `desktop_user_names`**:

All three tasks loop over an empty list → no iterations, no actions, role
completes successfully (FR-007).

## Complexity Tracking

No complexity violations. All constitution principles pass without exception.
