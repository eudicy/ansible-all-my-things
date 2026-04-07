# Java Role — Design

This document captures the non-obvious design decisions and technical
constraints of the `java` role. For requirements and user stories, see
`specs/005-java-role/spec.md`.

## Overview

The role installs SDKMAN into each desktop user's home directory and then
uses SDKMAN to install the current default Temurin LTS Java distribution.

AMD64 Ubuntu only; ARM64 hosts are skipped via the
`not-supported-on-vagrant-arm64` tag applied in
`configure-linux-roles.yml`.

## Non-Obvious Technical Constraints

### SDKMAN_AUTO_ANSWER Environment Variable

`sdk install java` prompts interactively ("Do you want java X to be set
as default? (Y/n):") when installing a candidate for the first time.
Setting `SDKMAN_AUTO_ANSWER: "true"` in the task's `environment:` block
answers "yes" to all prompts without user input. This approach is
preferred over writing `sdkman_auto_answer=true` to
`~/.sdkman/etc/config` because it leaves the user's SDKMAN configuration
untouched and requires no additional file-modification task.

### `2>&1` Redirect in the Check Task

The JVM writes `java -version` output to **stderr** by specification.
Without the `2>&1` redirect, Ansible captures an empty `stdout` and the
`"Temurin" not in item.stdout` guard always evaluates to `true`,
causing a redundant reinstall attempt on every run.

### `executable: /bin/bash` and the `source` Prefix

The `sdk` command is a shell function, not a standalone executable. It is
defined only after sourcing `~/.sdkman/bin/sdkman-init.sh`.
`ansible.builtin.shell` defaults to `/bin/sh`, which does not support
the `source` builtin. Specifying `executable: /bin/bash` is sufficient:
Ansible passes the `cmd` string directly to that interpreter, so `source`
(a bash builtin) works without an additional `bash -c '...'` wrapper.
Prefixing `source ~/.sdkman/bin/sdkman-init.sh &&` makes the `sdk`
function available to the subsequent command. When `become_user` is set,
`~` resolves to the target user's home directory.

### `failed_when: false` in the Check Task

`failed_when: false` is a defensive guard for the edge case where
`/bin/bash` is absent on the target host. In that scenario,
`executable: /bin/bash` would cause the shell module to fail before the
command body runs. The guard ensures the check task never aborts the
play; the resulting empty `stdout` triggers the install task, which then
fails with a clearer error. In practice, bash is always present on
Ubuntu 22.04+.

### Loop Result Access via `item.item`

The check task loops over `desktop_user_names` and registers
`java_version_check`. Ansible stores results under `.results` as a list;
each element retains the original loop variable as `.item`. The install
task loops over `java_version_check.results` and accesses the username
via `item.item` and the command output via `item.stdout`. This eliminates
an explicit `loop_control: index_var:` entry and keeps the `when:`
condition self-contained.

### Non-Existent User Behavior

If a username in `desktop_user_names` does not exist on the target host,
`become_user` fails visibly with an Ansible error identifying the
offending username. The role does not silently skip non-existent users.

### Partial SDKMAN Install Behavior

The `creates:` guard on the SDKMAN install task checks for
`~/.sdkman/bin/sdkman-init.sh`. If SDKMAN is partially installed (init
script present but corrupted), the install task is skipped. A subsequent
`sdk install java` call then fails visibly, making the corruption
detectable without masking it.

## SDKMAN Installer Integrity

SDKMAN's only supported install path is `curl -s "https://get.sdkman.io" | bash`.
The maintainers do not publish installer checksums, so there is no supported
way to verify the script before execution.

The `curl|bash` approach is accepted with the same rationale as TD-001
(Claude Code installer) and TD-007 (Google signing key): HTTPS transport
authenticates the server endpoint via TLS and prevents in-transit
modification, which is the same trust model used by Homebrew, rustup, and
the official Node.js installer. The blast radius is additionally limited to
the desktop user's home directory — SDKMAN is installed entirely under
`~/.sdkman/` and does not require root privileges.

This risk is documented as TD-010 in the technical debt register.

## Code Conventions

- All YAML files begin with `#SPDX-License-Identifier: MIT-0`.
- Task-level `become: true` is not used (play-level `become` is inherited).
- FQCN (`ansible.builtin.*`) is used throughout.
