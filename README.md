# Ansible In One Shot — TrainWithShubham

Learn Ansible hands-on with real AWS infrastructure provisioned by Terraform.
Scratch → pro in a single ~4-hour live session.

**The setup mirrors the real world:** one **control node** runs Ansible and manages
three **worker** nodes over the private network. Your laptop is used only once, to
bootstrap the control node — then you SSH into the control node and run the whole
course from there.

```
your laptop ──(one-time bootstrap)──▶ control-node-ubuntu ──(private network)──▶ worker-ubuntu
                                          (runs Ansible)                          worker-redhat
                                                                                  worker-amazon
```

## Prerequisites (on your laptop)

- [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) (>= 1.6)
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) — used only for the one-time bootstrap
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) (v2), configured with credentials

(The control node itself gets a current ansible-core installed automatically during bootstrap.)

---

## Quick Start

### Phase 1 — from your laptop: provision + bootstrap the control node

```bash
# 1. Clone and enter the repo
git clone https://github.com/TrainWithShubham/ansible-in-one-shot.git
cd ansible-in-one-shot

# 2. Generate an SSH key pair (no .pem extension — matches the Terraform default)
mkdir -p ~/keys
ssh-keygen -t rsa -b 4096 -f ~/keys/terra-key-ansible -N ""
chmod 600 ~/keys/terra-key-ansible
cp ~/keys/terra-key-ansible.pub terraform/terra-key-ansible.pub

# 3. Provision the 4 EC2 instances (also writes inventories/dev/hosts.ini + bootstrap.ini)
cd terraform && terraform init && terraform apply -auto-approve && cd ..

# 4. Bootstrap the control node: installs Ansible there, pushes your key,
#    clones the repo, and drops the teaching inventory in place.
ansible-playbook -i inventories/dev/bootstrap.ini playbooks/bootstrap_control_node.yml
```

> The bootstrap clones this repo from GitHub onto the control node, so **push your
> changes first** if you've edited anything locally.

### Phase 2 — SSH into the control node and run the course

```bash
# Grab the control node's public IP
cd terraform && terraform output -raw control_node_public_ip && cd ..

# SSH in (use that IP) and go
ssh -i ~/keys/terra-key-ansible ubuntu@<CONTROL_NODE_PUBLIC_IP>
cd ansible-in-one-shot

# You're now on the control node — everything below runs from here
ansible all -m ping        # control (local) + 3 workers (private IPs)
ansible workers -m ping
```

Everything in the modules below is run **from the control node**.

---

## Course Modules

Live-paced timings for a single ~4-hour session (the last live run covered this in ~4.5h).
Each module has its own README with commands to run.

| # | Module | What You'll Learn | Live Time |
|---|--------|-------------------|-----------|
| 1 | [Basics](modules/01-basics/) | Ad-hoc commands, playbooks, packages, services, FQCN | 45m |
| 2 | [Variables & Facts](modules/02-variables-and-facts/) | vars, register, debug, group_vars, `ansible_facts[]`, when | 40m |
| 3 | [Templates & Handlers](modules/03-templates-and-handlers/) | Jinja2 templates, handlers, notify | 30m |
| 4 | [Loops & Conditionals](modules/04-loops-and-conditionals/) | loop, when, block/rescue/always, tags | 30m |
| 5 | [Roles](modules/05-roles/) | Role structure, site.yml, reusable automation | 50m |
| 6 | [Vault](modules/06-vault/) | ansible-vault, encrypting secrets, no_log | 25m |

**Total: ~4 hours** including ~20m setup and Q&A/buffer.

Learning on your own? Follow the [7-day Study Checklist](STUDY_CHECKLIST.md): a daily
assignment per module, with "break it on purpose" exercises and a capstone. For reference, see the
[ad-hoc commands cheat sheet](docs/adhoc-cheatsheet.md) and
[manual setup walkthrough](docs/manual-setup-under-the-hood.md).

---

## Infrastructure

Terraform creates **4 EC2 instances** in `us-west-2` (all `t3.micro`, free-tier eligible):

| Host | OS | SSH User | Role group |
|------|----|----------|------------|
| `control-node-ubuntu` | Ubuntu 24.04 | `ubuntu` | `control` |
| `worker-ubuntu` | Ubuntu 24.04 | `ubuntu` | `workers` |
| `worker-redhat` | RHEL 9 | `ec2-user` | `workers` |
| `worker-amazon` | Amazon Linux 2023 | `ec2-user` | `workers` |

AMI IDs are region-specific — update them in `terraform/variables.tf` if you change regions.

Terraform generates **two** inventories:

- `inventories/dev/bootstrap.ini` — used once from your laptop (control node by **public IP**) to run the bootstrap playbook.
- `inventories/dev/hosts.ini` — the **teaching** inventory used **on the control node**: the control node manages itself (`ansible_connection=local`) and reaches workers by **private IP**.

The teaching inventory exposes these groups (playbooks target them via `hosts:`):
`control`, `workers`, `ubuntu`, `redhat`, `amazon`, plus the implicit `all`.
`ubuntu` = control + worker-ubuntu; `workers` = the three worker nodes.

---

## Repo Structure

```
ansible-in-one-shot/
├── ansible.cfg                    # Ansible settings (inventory, SSH, output)
├── requirements.yml               # Galaxy collections (community.general)
├── .ansible-lint                  # Lint config (truthy enforced, FQCN for collections)
├── terraform/                     # EC2, security group, auto-inventory
├── playbooks/
│   └── bootstrap_control_node.yml # One-time: sets up the control node
├── inventories/dev/
│   ├── hosts.ini                  # Teaching inventory (generated; on control node)
│   ├── bootstrap.ini              # Laptop→control bootstrap inventory (generated)
│   ├── group_vars/                # all, ubuntu, redhat, amazon
│   └── host_vars/                 # control-node-ubuntu
├── roles/                         # common, docker, nginx
├── modules/                       # 6 learning modules (start here!)
├── docs/                          # Ad-hoc cheat sheet, manual setup walkthrough
└── STUDY_CHECKLIST.md             # 7-day self-paced assignment
```

---

## Common Commands

Run these **from the control node** (after Phase 2 above):

```bash
# Test connectivity
ansible all -m ping

# Target a group
ansible ubuntu -m ping
ansible workers -a "uptime"

# Run a playbook
ansible-playbook modules/01-basics/01_ping.yml

# Run with verbose output
ansible-playbook modules/01-basics/02_gather_facts.yml -v

# Run specific tags only
ansible-playbook modules/04-loops-and-conditionals/05_tags_demo.yml --tags install

# Dry run (check mode)
ansible-playbook modules/01-basics/03_install_packages.yml --check

# Lint the content
ansible-lint
```

### Validate a playbook before you run it

```bash
ansible-playbook <file>.yml --syntax-check   # YAML and structure only, no connection to hosts
ansible-playbook <file>.yml --list-hosts     # which hosts it will touch
ansible-playbook <file>.yml --list-tasks     # which tasks it will run
ansible-playbook <file>.yml --check --diff   # dry run, shows file changes
ansible-playbook <file>.yml                  # run it again afterwards: expect changed=0
```

There is no `--validate` flag. Use `--syntax-check`. See the
[cheat sheet](docs/adhoc-cheatsheet.md) for host patterns such as `'workers:&ubuntu'` and `--limit`.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Bootstrap: `Permission denied (publickey)` | Key not synced — re-run key-gen step so `terraform/terra-key-ansible.pub` matches `~/keys/terra-key-ansible` |
| Bootstrap: repo missing your latest changes | The bootstrap clones from GitHub — `git push` first |
| On control node: `No hosts matched` | The teaching `hosts.ini` wasn't copied — re-run the bootstrap playbook |
| Workers `UNREACHABLE` from control node | Key must be at `~/keys/terra-key-ansible` on the control node; workers' SG must allow port 22 (it allows intra-VPC by default) |
| `couldn't resolve module community.general.*` | Run `ansible-galaxy collection install -r requirements.yml` |
| `become: permission denied` | Default EC2 users have sudo — check your `hosts` line |
| Ad-hoc `dnf`/`apt` command fails with a permissions error | `ansible.cfg` sets `become = False`. Add `-b`, e.g. `ansible redhat -b -a "dnf update -y"` |
| `ansible servers:node-1 ...` hits more hosts than expected | `:` is a union. Use the host name alone, `'group:&host'`, or `--limit`. See the [cheat sheet](docs/adhoc-cheatsheet.md#host-patterns) |
| `object of type 'dict' has no attribute 'ansible_distribution'` | Keys inside `ansible_facts` have no `ansible_` prefix: use `ansible_facts['distribution']` |
| Docker fails to start on RHEL: `Extension addrtype ... missing kernel module` | The running kernel lacks `xt_addrtype` (a kernel update was installed but not booted into). The `docker` role installs `kernel-modules-extra` and reboots if needed. By hand: `sudo dnf install -y kernel-modules-extra && sudo reboot`, then `sudo systemctl reset-failed docker && sudo systemctl start docker` |
| `Start request repeated too quickly` for a service | systemd rate-limited the restarts. Fix the cause, then run `systemctl reset-failed <service>` before starting it again |
| `ansible-playbook: error: unrecognized arguments: --validate` | The flag is `--syntax-check` |

---

## Cleanup

```bash
cd terraform && terraform destroy -auto-approve
```

---

## Maintainer

**Shubham Londhe** — [TrainWithShubham](https://github.com/LondheShubham153)
