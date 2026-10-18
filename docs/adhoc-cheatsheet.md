# Ad-hoc commands cheat sheet

Ad-hoc commands are one-liners you run without writing a playbook. Run them from the control node.
They are great for checks and one-offs. For anything you want to repeat, write a playbook.

```
ansible <pattern> [-b] [-v] -m <module> -a "<arguments>"
```

If you leave out `-m`, Ansible uses the `command` module.

## Host patterns

Groups in this repo's inventory: `control`, `workers`, `ubuntu`, `redhat`, `amazon`, plus `all`.

| Pattern | Meaning | Example |
|---------|---------|---------|
| `all` | Every host | `ansible all -m ping` |
| `workers` | One group | `ansible workers -a "uptime"` |
| `worker-redhat` | One host by name | `ansible worker-redhat -m ping` |
| `ubuntu:redhat` | Union (OR): hosts in either group | `ansible ubuntu:redhat -m ping` |
| `'workers:&ubuntu'` | Intersection (AND): in both groups | `ansible 'workers:&ubuntu' -m ping` |
| `'workers:!amazon'` | Exclusion: in the first, not the second | `ansible 'workers:!amazon' -m ping` |
| `workers --limit worker-redhat` | Group, narrowed to one host | `ansible workers -m ping --limit worker-redhat` |

> The `:` trap: `servers:node-1` does not mean "node-1 inside servers". `:` is a union,
> so it targets every host in `servers` plus `node-1`. To target exactly one host, name it
> directly (`ansible node-1 ...`), use `'servers:&node-1'`, or use `--limit node-1`.
> Quote patterns that contain `&` or `!` so your shell doesn't interpret them.

Check what a pattern matches before you run anything:

```bash
ansible 'workers:&ubuntu' --list-hosts
ansible-inventory --graph
```

## Privilege escalation: `-b`

`ansible.cfg` sets `become = False`, so ad-hoc commands run as the login user (`ubuntu` or
`ec2-user`). Anything that needs root (package installs, services, files under `/etc`) needs `-b`:

```bash
ansible workers -b -a "systemctl status sshd --no-pager"
ansible redhat -b -a "dnf update -y"           # without -b, dnf refuses to run as a normal user
```

## Verbosity: `-v`

| Flag | Shows |
|------|-------|
| `-v` | Task results |
| `-vv` | Task input and more detail |
| `-vvv` | SSH connection details. Start here when a host is `UNREACHABLE` |
| `-vvvv` | Adds connection plugin and script details |

## Useful modules

```bash
ansible all -m ping                                   # SSH + Python reachable?
ansible workers -a "df -h"                            # command: no pipes or redirects
ansible workers -m shell -a "ps aux | grep sshd"      # shell: pipes and redirects work
ansible workers -m setup                              # all facts
ansible workers -m setup -a "filter=ansible_distribution*"   # filtered facts
ansible workers -m copy -a "src=./a.txt dest=/tmp/a.txt"
ansible workers -m file -a "path=/opt/app state=directory mode=0755" -b
ansible workers -m service -a "name=sshd state=started" -b
```

Use `-m setup` to see the facts. Remember that inside `ansible_facts` the keys have
no `ansible_` prefix: `ansible_facts['distribution']`, not `ansible_facts['ansible_distribution']`.

## Packages: simple commands

```bash
ansible ubuntu -b -a "apt-get update"
ansible ubuntu -b -a "apt-get install -y nginx"
ansible redhat -b -a "dnf install -y httpd"
ansible redhat -b -a "dnf update -y"
ansible workers -b -m package -a "name=tree state=present"   # works on every OS
```

Always pass `-y` to `apt-get install` and `dnf install` in an ad-hoc command. Without it the
package manager waits for a yes/no answer and the command hangs or fails.

The idempotent alternative uses the module instead of the command:

```bash
ansible redhat -b -m dnf -a "name=* state=latest update_cache=yes"
ansible ubuntu -b -m apt -a "name=nginx state=present update_cache=yes"
```

On RHEL 7 and older, use `yum` in place of `dnf`.

## Services

```bash
ansible ubuntu -b -a "systemctl enable --now nginx"
ansible ubuntu -b -m service -a "name=nginx state=started enabled=yes"
```

## Docker with the `command` module

Docker needs root unless the user is in the `docker` group, so add `-b`:

```bash
ansible workers -b -a "docker --version"
ansible workers -b -a "docker ps -a"
ansible workers -b -a "docker images"
ansible workers -b -a "docker run --rm hello-world"
ansible workers -b -a "docker run -d --name web -p 8080:80 nginx"
ansible workers -b -a "docker logs web"
ansible workers -b -a "docker stop web"
ansible workers -b -a "docker rm web"
```

`command` has no shell, so pipes and redirects do not work. Use `-m shell` for those.

In a playbook, prefer the idempotent `community.docker.docker_container` module over
`command: docker run ...`. Install it with `ansible-galaxy collection install community.docker`.

## Fact gathering across the whole fleet

```bash
ansible all -m setup                       # print every host's facts
ansible all -m setup --tree /tmp/facts     # save one JSON file per host
```

## Validate a playbook before you run it

```bash
ansible-playbook site.yml --syntax-check   # YAML and structure only, no connection
ansible-lint site.yml                      # style and best practices
ansible-playbook site.yml --list-hosts     # which hosts
ansible-playbook site.yml --list-tasks     # which tasks
ansible-playbook site.yml --check --diff   # dry run, shows file changes
```

There is no `--validate` flag. `--check` can be inaccurate for `command` and `shell` tasks,
because they are skipped in check mode.

## Beginner module map

| Module | Use it for |
|--------|-----------|
| `ping` | Check connectivity |
| `command` / `shell` | Run a command (`shell` allows pipes) |
| `copy` / `file` | Put files and directories on hosts |
| `package` / `apt` / `dnf` | Install packages |
| `service` / `systemd` | Manage services |
| `user` / `group` | Manage accounts |
| `setup` | Gather facts |
| `debug` | Print variables and messages |

Prefer a real module to `shell` or `command`. Modules are idempotent, so they only change
something when it differs. For documentation offline, run `ansible-doc <module>`.
