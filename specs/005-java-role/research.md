# Research: Java Role

**Branch**: `005-java-role` | **Date**: 2026-04-07

This document resolves the technical unknowns identified during planning.
All findings feed directly into `plan.md`.

## Finding 1: SDKMAN Installer and Idempotency Guard

The standard SDKMAN installation command is:

```bash
curl -s "https://get.sdkman.io" | bash
```

The installer creates `~/.sdkman/bin/sdkman-init.sh` as its final step.
Using `creates: /home/{{ item }}/.sdkman/bin/sdkman-init.sh` in the
`ansible.builtin.shell` task prevents re-running the installer when SDKMAN
is already present. The `creates:` parameter is evaluated on the remote host
in the context of `become_user`, so `~` correctly resolves to the target
user's home directory.

**Selected approach**: `creates:` guard. No `stat` module check needed — the
`creates:` parameter provides the same behaviour with less code.

## Finding 2: Suppressing SDKMAN Interactive Prompts

`sdk install java` prompts interactively when:

- Installing a candidate for the first time ("Do you want java X to be set
  as default? (Y/n):").

**Resolution**: Set the `SDKMAN_AUTO_ANSWER=true` environment variable before
calling any `sdk` subcommand. SDKMAN reads this variable and answers "yes" to
all prompts without user input. The variable is set in the Ansible task's
`environment:` block, so it does not persist to the user's shell environment.

This is preferred over writing `sdkman_auto_answer=true` to
`~/.sdkman/etc/config` because it avoids a separate file-modification task and
leaves the user's SDKMAN configuration untouched.

## Finding 3: Java Install Idempotency Guard

`sdk install java` is not fully idempotent on its own: its exit behaviour when
the candidate is already installed is implementation-dependent and has changed
across SDKMAN versions. Relying on it is fragile.

**Selected approach**: two-task pattern:

1. **Check task** — source SDKMAN init script, run `java -version 2>&1`,
   register result with `changed_when: false` and `failed_when: false`. If
   Java is not installed (or SDKMAN not yet initialised), the command exits
   non-zero with a "command not found" message; "Temurin" will not appear in
   stdout.
2. **Install task** — run only `when: '"Temurin" not in item.stdout'`. This
   satisfies FR-003 (idempotency) and SC-002 (zero changed tasks on re-run).

The `2>&1` redirect is required because the JVM writes `java -version` output
to **stderr** (by specification). Without the redirect, Ansible captures an
empty `stdout` and the "Temurin" check always fails.

## Finding 4: Sourcing SDKMAN in Ansible Shell Tasks

The `sdk` command is a shell function, not an executable. It is only available
after sourcing `~/.sdkman/bin/sdkman-init.sh`. `ansible.builtin.shell`
defaults to `/bin/sh`, which does not support the `source` builtin.

**Selected approach**: specify `executable: /bin/bash` on every task that
sources the init script, and prefix the command with `source
~/.sdkman/bin/sdkman-init.sh &&`. When `become_user` is set, `~` resolves
to the target user's home directory, so no absolute-path substitution is
required.

## Finding 5: Loop Result Access Pattern

Ansible registers loop results under `.results` as a list. Each element
retains the original loop variable as `.item`. The pattern used by
`android_studio` (and adopted here) is:

```yaml
# Task 1 — loop over usernames, register check results
- name: Check Java for each user
  ansible.builtin.shell: ...
  loop: "{{ desktop_user_names }}"
  register: java_version_check
  ...

# Task 2 — loop over results, access username via item.item
- name: Install Java for each user if needed
  ansible.builtin.shell: ...
  become_user: "{{ item.item }}"
  loop: "{{ java_version_check.results }}"
  when: '"Temurin" not in item.stdout'
  ...
```

This eliminates a second `loop_control` index variable and keeps the condition
expression self-contained.

## Finding 6: `become_user` Without Task-Level `become: true`

`configure-linux-roles.yml` sets `become: true` at the play level. Role tasks
that set only `become_user: "{{ item }}"` inherit the play-level `become: true`
and switch to the named user without requiring a task-level `become:` key.
This matches the pattern in `android_studio/tasks/main.yml`.

## Finding 7: AMD64 Tag in `configure-linux-roles.yml`

The java role targets AMD64 Ubuntu only (CO-001). It must be added to
`configure-linux-roles.yml` with the `not-supported-on-vagrant-arm64` tag,
consistent with `android_studio`, `google_chrome`, and `flutter`. The tag is
applied at the role entry level in the playbook, not inside the role's tasks.

## Finding 8: `curl` Prerequisite

The SDKMAN installer requires `curl`. Ubuntu 22.04+ includes `curl` in the
default install. No explicit package installation step is needed, but the
`README.md` must list `curl` as a prerequisite.

## Open Issues

None. All key technical challenges are resolved:

- SDKMAN non-interactive mode → `SDKMAN_AUTO_ANSWER=true` env var.
- Java-version stderr output → `2>&1` redirect in the check command.
- `sdk` as a shell function → `executable: /bin/bash` + `source` prefix.
- Idempotency without reliable exit codes from SDKMAN → pre-check + `when:`
  guard.
