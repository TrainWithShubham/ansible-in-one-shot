# Docker Role

Installs and configures Docker across Ubuntu, RHEL, and Amazon Linux.

## What it does
- Installs Docker using the appropriate method per OS family
- Starts and enables the Docker service
- Optionally adds users to the docker group

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `docker_users` | `[]` | Users to add to the docker group |
| `docker_service_enabled` | `true` | Enable Docker on boot |

## Usage

```yaml
- hosts: all
  become: true
  roles:
    - role: docker
      docker_users:
        - ubuntu
```

## Supported OS
- Ubuntu (docker.io via apt)
- RHEL (docker-ce from Docker's repo via dnf). The course targets RHEL 9; the kernel-module handling below was tested on RHEL 10.2
- Amazon Linux 2023 (docker via dnf)

## RHEL: kernel modules and the reboot

Docker's bridge network needs the `xt_addrtype` kernel module, which ships in
`kernel-modules-extra`. A plain `dnf update` installs the newest kernel but does not boot into it,
so the running kernel may lack the module and `dockerd` fails with:

```
Extension addrtype revision 0 not supported, missing kernel module?
```

On RHEL the role therefore:
1. installs `kernel-modules-extra`
2. checks the running kernel with `modinfo xt_addrtype`
3. reboots the host only if the module is missing (a node that is already on the right kernel is not rebooted)
4. writes `{"iptables": false}` to `/etc/docker/daemon.json` only as a last-resort fallback, if the module is still missing after the reboot. With iptables disabled Docker no longer manages NAT, so published ports and outbound container traffic stop working

Plan for the reboot: don't run this role against production hosts during working hours.
