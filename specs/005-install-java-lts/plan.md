# Implementation Plan: Java Ansible Role

**Branch**: `005-install-java-lts` | **Date**: 2026-04-07 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/005-install-java-lts/spec.md`

## Summary

Create a new Ansible role `java` that installs sdkman per-user into
`/home/<user>/.sdkman` via the official installer script, then uses sdkman
to install the Temurin JDK (default `21.0.10-tem`) for every user in
`desktop_user_names`. The role works on AMD64 and ARM64 Ubuntu Linux without
architecture guards because both sdkman and Temurin ship native binaries for
both architectures.

## Technical Approach

1. **Install sdkman** — run `curl -s "https://get.sdkman.io" | bash` via
   `ansible.builtin.shell` with
   `creates: /home/{{ item }}/.sdkman/bin/sdkman-init.sh` for idempotency.
   Set `SDKMAN_DIR` explicitly to avoid sdkman defaulting to `~/.sdkman`.

2. **Install Java** — run
   `source .../sdkman-init.sh && sdk install java {{ java_sdk_version }}`
   in the same shell (required for `sdk` to be on `PATH`). Guard with
   `creates: /home/{{ item }}/.sdkman/candidates/java/{{ java_sdk_version }}`.

3. **PATH** — sdkman's `.bashrc` hook (added automatically by the installer)
   puts `~/.sdkman/candidates/java/current/bin` on `PATH`; no extra
   `blockinfile` task is needed.

Both tasks loop over `desktop_user_names` and use `become_user: "{{ item }}"`.
`become: true` is at play level, not in the role.

## Role Structure

```text
roles/java/
├── defaults/main.yml   # java_sdk_version: "21.0.10-tem"
├── meta/main.yml       # galaxy_info; dependencies: []
├── tasks/main.yml      # install sdkman + install Java via sdkman
└── README.md

configure-linux-roles.yml  # add java entry after flutter
```

## Constitution Check

| Principle | Status | Notes |
| --- | --- | --- |
| I. Idempotency | PASS | `creates:` guards on both tasks prevent re-runs from reporting `changed` |
| II. Role-First Organisation | PASS | All logic in `roles/java/`; single entry added to `configure-linux-roles.yml` |
| IV. Simplicity (YAGNI) | PASS | Two tasks only; no abstraction beyond what the spec requires |
