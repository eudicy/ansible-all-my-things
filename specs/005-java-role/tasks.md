# Tasks: Java Role

**Input**: Design documents from `specs/005-java-role/`
**Branch**: `005-java-role`

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to
- Exact file paths are included in all descriptions

---

## Phase 1: Setup

**Purpose**: Create the role directory skeleton

- [ ] T001 [P] Create directory `roles/java/tasks/`
- [ ] T002 [P] Create directory `roles/java/defaults/` *(ceremonial skeleton step — implicit when creating files in that directory)*
- [ ] T003 [P] Create directory `roles/java/meta/` *(ceremonial skeleton step — implicit when creating files in that directory)*

---

## Phase 2: Foundational

No foundational blocking prerequisites beyond Phase 1 setup.

---

## Phase 3: User Story 1 — Java Available for All Desktop Users (Priority: P1)

**Goal**: Every user in `desktop_user_names` can run `java -version` and see Temurin.

**Independent Test**: Apply the role to a `vagrant_docker` VM with at least one
desktop user. Log in as that user and run `java -version`; output must contain
"Temurin".

### Implementation for User Story 1

- [ ] T004 [US1] Create `roles/java/tasks/main.yml` with `#SPDX-License-Identifier: MIT-0` as line 1,
  then add Install-SDKMAN task (`ansible.builtin.shell`, `creates:` guard,
  `become_user`, loop over `desktop_user_names`)
- [ ] T005 [US1] Add Check-Temurin task to `roles/java/tasks/main.yml`
  (`ansible.builtin.shell`, `executable: /bin/bash`, `source` prefix,
  `register: java_version_check`, `changed_when: false`, `failed_when: false`)
- [ ] T006 [US1] Add Install-Java task to `roles/java/tasks/main.yml`
  (`ansible.builtin.shell`, `SDKMAN_AUTO_ANSWER: "true"` env, loop over
  `java_version_check.results`, `when: '"Temurin" not in item.stdout'`)
- [ ] T007 [P] [US1] Create `roles/java/defaults/main.yml` with `#SPDX-License-Identifier: MIT-0` as line 1 (empty defaults file)
- [ ] T008 [P] [US1] Create `roles/java/meta/main.yml` with `#SPDX-License-Identifier: MIT-0` as line 1
  (Galaxy metadata, AMD64 Ubuntu, MIT licence, min_ansible_version 2.19)
- [ ] T009 [P] [US1] Create `roles/java/README.md` (requirements, variables,
  prerequisites including `curl`, example playbook)
- [ ] T010 [P] [US1] Create `roles/java/DESIGN.md` (non-obvious decisions:
  `SDKMAN_AUTO_ANSWER`, `2>&1` redirect, `executable: /bin/bash`, loop result
  access via `item.item`, non-existent user behavior, partial SDKMAN install behavior)
- [ ] T011 [US1] Add java role entry to `configure-linux-roles.yml` with
  `tags: not-supported-on-vagrant-arm64` (after the flutter role entry)
- [ ] T011a [US1] Run the role on the `vagrant_docker` VM and confirm that
  `java -version` output contains "Temurin" for each configured desktop user

**Checkpoint**: User Story 1 fully functional — apply role and verify
`java -version` output contains "Temurin" for each desktop user.

---

## Phase 4: User Story 2 — Idempotent Re-application (Priority: P2)

**Goal**: A second role run reports zero changed tasks.

**Independent Test**: Apply the role twice consecutively; the second run must
show zero changed and zero failed tasks.

Idempotency is delivered by the `creates:` guard (T004) and the `when:`
condition on the install task (T006) — no additional implementation tasks are
required. Verify by re-running the playbook after Phase 3 is complete.

- [ ] T012 [US2] Re-apply the role on the `vagrant_docker` VM and confirm
  zero changed tasks and zero failed tasks
- [ ] T012a [US2] Run the role with `desktop_user_names: []` and confirm
  zero changed tasks and zero failed tasks (FR-007)
- [ ] T012b [US2] Run the role with a non-existent username in
  `desktop_user_names` and confirm the role fails with a clear error message
  and does not silently skip the user (FR-008)

**Checkpoint**: Idempotency confirmed — User Story 2 acceptance criteria met.

---

## Final Phase: Polish

**Purpose**: Verify markdown quality compliance per constitution Principle VI

- [ ] T013 [P] Verify `roles/java/README.md` complies with constitution VI
  (ATX headings, blank-line list rules, no trailing whitespace, single trailing
  newline)
- [ ] T014 [P] Verify `roles/java/DESIGN.md` complies with constitution VI
- [ ] T015 Commit with message `feat: add java role` (Principle V)
