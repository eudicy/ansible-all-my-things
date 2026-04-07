---
description: "Task list for Java Ansible Role implementation"
---

# Tasks: Java Ansible Role

**Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

---

## T001 — Create `roles/java/defaults/main.yml`

**File**: `roles/java/defaults/main.yml`

```yaml
#SPDX-License-Identifier: MIT-0
---
java_sdk_version: "21.0.10-tem"
```

---

## T002 — Create `roles/java/meta/main.yml`

**File**: `roles/java/meta/main.yml`

```yaml
#SPDX-License-Identifier: MIT-0
galaxy_info:
  author: Stefan Boos
  description: Install Java LTS (Temurin) via sdkman on AMD64 and ARM64 Ubuntu Linux
  company: n.a.
  issue_tracker_url: https://github.com/wonderbird/ansible-all-my-things/issues
  license: MIT
  min_ansible_version: 2.19
  galaxy_tags: []

dependencies: []
```

---

## T003 — Create `roles/java/tasks/main.yml`

**File**: `roles/java/tasks/main.yml`

```yaml
#SPDX-License-Identifier: MIT-0
---
- name: Install sdkman
  ansible.builtin.shell:
    cmd: curl -s "https://get.sdkman.io" | bash
    executable: /bin/bash
    creates: /home/{{ item }}/.sdkman/bin/sdkman-init.sh
  environment:
    SDKMAN_DIR: /home/{{ item }}/.sdkman
  become_user: "{{ item }}"
  loop: "{{ desktop_user_names }}"

- name: Install Java via sdkman
  ansible.builtin.shell:
    cmd: |
      source /home/{{ item }}/.sdkman/bin/sdkman-init.sh &&
      sdk install java {{ java_sdk_version }}
    executable: /bin/bash
    creates: /home/{{ item }}/.sdkman/candidates/java/{{ java_sdk_version }}
  environment:
    SDKMAN_DIR: /home/{{ item }}/.sdkman
    SDKMAN_AUTO_ANSWER: "true"
  become_user: "{{ item }}"
  loop: "{{ desktop_user_names }}"
```

---

## T004 — Create `roles/java/README.md`

**File**: `roles/java/README.md`

Write a README following the `android_studio` role pattern with these
sections:

- **Title**: `Java`
- **Intro**: "Install the Java LTS Temurin JDK for all desktop users via
  sdkman."
- **Requirements**: Ubuntu Linux (AMD64 or ARM64) with `curl` and `bash`
  pre-installed; `desktop_user_names` variable defined; internet access on
  the first run; target users must already exist on the system.
- **Role Variables table**: one row — `java_sdk_version` | `21.0.10-tem` |
  sdkman version identifier for the Temurin JDK. Note that available
  versions can be listed with `sdk list java` after sdkman is installed.
- **Dependencies**: none
- **Example Playbook**: standard role invocation using `java`.
- **License**: MIT
- **Author Information**: Stefan Boos

---

## T005 — Add `java` entry to `configure-linux-roles.yml`

**File**: `configure-linux-roles.yml`

In the `roles:` list, add `- java` after the `flutter` role block and
before the `# Optional roles` comment:

```yaml
    - role: flutter
      tags: not-supported-on-vagrant-arm64
    - java

    # Optional roles - uncomment as needed
```
