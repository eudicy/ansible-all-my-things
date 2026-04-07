# Java

Install the Java LTS Temurin JDK for all desktop users via sdkman.

## Requirements

- Ubuntu Linux (AMD64 or ARM64) with `curl` and `bash` pre-installed.
- `desktop_user_names` variable defined (list of users to receive the JDK).
- Internet access on the first provisioning run (sdkman and JDK downloads).
- Target users must already exist on the system.

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `java_sdk_version` | `21.0.10-tem` | sdkman version identifier for the Temurin JDK. |

To find available Temurin versions, run `sdk list java` after sdkman is
installed and look for entries ending in `-tem`.

## Dependencies

none

## Example Playbook

```yaml
- hosts: servers
  roles:
    - java
```

## Security

The sdkman project does not publish a checksum for its installer script.
This role therefore uses the canonical `curl -s "https://get.sdkman.io" | bash`
installation method. The HTTPS transport prevents network-level interception;
a compromise of the `get.sdkman.io` origin server cannot be detected. This
risk is accepted because no checksum alternative is available upstream.

The JDK archive itself is downloaded and verified by sdkman using the
checksums it maintains for each candidate version.

## License

MIT

## Author Information

Stefan Boos <kontakt@boos.systems>
