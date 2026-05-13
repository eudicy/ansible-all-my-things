# CLAUDE.md

This project's authoritative guidance lives in **`AGENTS.md`** (agent
runtime) and **`.specify/memory/constitution.md`** (project rules). Read
both.

Do not duplicate rules here. Update AGENTS.md or the constitution instead.

## Active Technologies
- Ansible-core >= 2.19.0 + `ansible.builtin.uri`, `ansible.builtin.replace`, `ansible.builtin.set_fact`, `ansible.builtin.debug` (built-in) + `community.general.version_sort` (already in `requirements.yml`) (007-version-update-playbooks)
- Local filesystem — role `defaults/main.yml` files modified in-place (007-version-update-playbooks)
