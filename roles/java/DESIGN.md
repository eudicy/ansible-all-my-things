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
the `source` builtin. Specifying `executable: /bin/bash` ensures bash
is used, and prefixing `source ~/.sdkman/bin/sdkman-init.sh &&` makes
the `sdk` function available to the subsequent command. When
`become_user` is set, `~` resolves to the target user's home directory.

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

## Code Conventions

- All YAML files begin with `#SPDX-License-Identifier: MIT-0`.
- Task-level `become: true` is not used (play-level `become` is inherited).
- FQCN (`ansible.builtin.*`) is used throughout.
