# Java

Install SDKMAN and Temurin LTS Java for all desktop users.

## Requirements

- AMD64 Ubuntu Linux.
- `desktop_user_names` variable defined (list of users to receive Java).
- Internet access on the first provisioning run (SDKMAN and JDK downloads).
- `curl` installed on the target host (present by default on Ubuntu 22.04+).
- `ansible.builtin` modules only; no additional collections required.

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `desktop_user_names` | _(required)_ | List of usernames that receive SDKMAN and Temurin Java. |

## Dependencies

none

## Example Playbook

```yaml
- hosts: servers
  roles:
    - java
```

## License

MIT

## Author Information

Stefan Boos <kontakt@boos.systems>
