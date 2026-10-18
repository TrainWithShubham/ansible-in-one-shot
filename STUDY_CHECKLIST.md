# Study checklist: Ansible In One Shot

A 7-day self-paced assignment for working through this repo alongside the course video.
Plan for roughly 1.5 to 2 hours a day (12 to 14 hours total): 3 to 4 hours of video plus hands-on practice.

> For every module: watch it, type the examples yourself, break something on purpose,
> then redo the task from memory without looking.

Everything below runs from the control node (see [Quick Start](README.md#quick-start)),
using the groups `control`, `workers`, `ubuntu`, `redhat`, `amazon`.

## How to use this file

- Copy it, or tick the boxes in your own fork (`- [ ]` becomes `- [x]`).
- Each day ends with a "You're done when" line. Don't move on until every line is true.
- The self-check questions have no answer key. If you can't answer one, re-read that module's README.
- Keep a short notes file: 3 bullets per day on what you learned and what surprised you.

## Time at a glance

| Day | Topic | Video | Total |
|-----|-------|-------|-------|
| 1 | Lab setup | 20m | 1.5 to 2 h |
| 2 | [Module 1: Basics](modules/01-basics/) | 45m | 2 h |
| 3 | [Module 2: Variables and Facts](modules/02-variables-and-facts/) | 40m | 1.5 to 2 h |
| 4 | [Module 3: Templates and Handlers](modules/03-templates-and-handlers/) | 30m | 1.5 h |
| 5 | [Module 4: Loops and Conditionals](modules/04-loops-and-conditionals/) | 30m | 1.5 h |
| 6 | [Module 5: Roles](modules/05-roles/) | 50m | 2 to 2.5 h |
| 7 | [Module 6: Vault](modules/06-vault/) + capstone | 25m | 2 to 2.5 h |

Short on time? Never skip Module 5 (Roles) or Module 6 (Vault). Cut extra practice on Modules 3 and 4 first.

---

## Day 1: Set up the lab

- [ ] Install the prerequisites (Terraform, Ansible, AWS CLI) and generate your SSH key
- [ ] `terraform apply` creates the 4 instances
- [ ] Run `playbooks/bootstrap_control_node.yml`
- [ ] SSH into the control node
- [ ] `ansible all -m ping` returns `pong` from all 4 nodes
- [ ] Run `ansible-inventory --graph` and identify each group
- [ ] Run `terraform destroy`, then rebuild once to prove you can

Optional: read [docs/manual-setup-under-the-hood.md](docs/manual-setup-under-the-hood.md) to see what the bootstrap automates.

Self-check:
- Why does the control node use `ansible_connection=local`?
- Which hosts are in `workers`, and which in `ubuntu`?

You're done when: `ansible all -m ping` is green and you can rebuild the lab from scratch.

---

## Day 2: Module 1, Basics

- [ ] Run the ad-hoc commands from the [Module 1 README](modules/01-basics/README.md)
- [ ] Run all 5 playbooks, `01_ping.yml` through `05_services.yml`
- [ ] Run `03_install_packages.yml` twice and compare the `changed` count
- [ ] Practice host patterns (see the [ad-hoc cheat sheet](docs/adhoc-cheatsheet.md)):
  - [ ] `ansible workers -m ping`
  - [ ] `ansible 'workers:&ubuntu' -m ping` (intersection)
  - [ ] `ansible workers -m ping --limit worker-redhat`
- [ ] Run one ad-hoc command that needs root, with and without `-b`
- [ ] Validate before running: `--syntax-check`, then `--check --diff`

Break it on purpose:
- [ ] Run `ansible ubuntu:redhat -m ping` (union), then `ansible 'ubuntu:&redhat' -m ping` (intersection), and explain why the second matches no hosts
- [ ] Misspell a package name in a copy of `03_install_packages.yml` and read the error

Self-check:
- What is the difference between `command` and `shell`?
- Why does the second run of a playbook report `changed=0`?
- When would you use `package` and when `dnf` or `apt`?

You're done when: you can explain `changed=0` and write an ad-hoc command that installs a package on one OS group only.

---

## Day 3: Module 2, Variables and Facts

- [ ] Run all 4 playbooks in [Module 2](modules/02-variables-and-facts/)
- [ ] Run `ansible workers -m setup` and read the facts for each OS
- [ ] Filter facts: `ansible workers -m setup -a "filter=ansible_distribution*"`
- [ ] Look at `inventories/dev/group_vars/` and find which file applies to the `redhat` group
- [ ] Write a small playbook that prints `os_family` and the total memory for every host

Break it on purpose:
- [ ] In `ansible_facts`, keys have no `ansible_` prefix. Write `when: ansible_facts["ansible_distribution"] == "Ubuntu"` and read the error, then fix it to `ansible_facts["distribution"]`
- [ ] Reference an undefined variable and see what Ansible says

Self-check:
- Why is `ansible_facts['os_family']` preferred over `ansible_os_family`?
- What is the variable precedence between `group_vars`, `host_vars` and `-e`?
- Why do you quote `"{{ my_var }}"` at the start of a value?

You're done when: you can write a fact-based `when` condition without looking, and you know where facts live.

---

## Day 4: Module 3, Templates and Handlers

- [ ] Run all 3 playbooks in [Module 3](modules/03-templates-and-handlers/)
- [ ] Run `03_handlers_demo.yml` twice and note when the handler fires
- [ ] Edit `templates/index.html.j2`, re-run, and watch the handler fire only when the file changed
- [ ] Open the rendered file on a worker and compare it with the template

Break it on purpose:
- [ ] Add a Jinja2 syntax error to a template and read the error
- [ ] Notify the same handler from two tasks and confirm it runs once

Self-check:
- What is the difference between `copy` and `template`?
- Why do handlers run at the end of the play?
- What has to be true for a handler to run?

You're done when: you can add a templated config file with a restart handler to a new service.

---

## Day 5: Module 4, Loops and Conditionals

- [ ] Run the playbooks in [Module 4](modules/04-loops-and-conditionals/), `01` through `06`
- [ ] Run `05_tags_demo.yml` three ways: `--tags install`, `--skip-tags verify`, `--tags "install,verify"`
- [ ] Use `--list-tasks` and `--list-hosts` to preview a playbook without running it
- [ ] Loop over a list of packages with `loop`
- [ ] Wrap a task in `block` / `rescue` / `always`

Break it on purpose:
- [ ] Make a task inside `04_block_rescue.yml` fail and trace what `rescue` does
- [ ] Use a `when` condition that is true on only one OS and confirm the others skip

Self-check:
- How does `block` / `rescue` / `always` map to try / catch / finally?
- Why is one `when` on a `block` better than repeating it on every task?

You're done when: you can write one playbook that installs OS-specific packages with `block` + `when`, and you can run only part of it with tags.

---

## Day 6: Module 5, Roles

- [ ] Read the structure of `roles/common`, `roles/docker` and `roles/nginx`
- [ ] Run `modules/05-roles/site.yml`
- [ ] Run it again with `--limit ubuntu`, then with `--check --diff`
- [ ] Override a role default from the command line with `-e nginx_listen_port=8080`
- [ ] Read `roles/docker/tasks/install_redhat.yml` and explain each step, including why it checks for the `xt_addrtype` kernel module and when it reboots
- [ ] Create a new role with `ansible-galaxy init roles/my_role`
- [ ] Assignment: turn one of your own playbooks into a role with defaults, tasks and a handler

Break it on purpose:
- [ ] Apply the `docker` role to a RHEL worker after a plain `dnf update` and watch the kernel check (it reboots only if needed)
- [ ] Add a second handler with the same name in two roles and see what happens

Self-check:
- Where do role defaults rank in variable precedence?
- Why does `docker` use `include_tasks` with an OS-specific file?
- When is a reboot needed after updating packages?

You're done when: your own role passes `ansible-lint` and a second run reports `changed=0`.

---

## Day 7: Module 6, Vault, and the capstone

- [ ] Do the one-time vault setup in the [Module 6 README](modules/06-vault/README.md)
- [ ] `ansible-vault view`, then `edit`, the encrypted file
- [ ] Run all 3 playbooks with `--vault-password-file`, then once with `--ask-vault-pass`
- [ ] Confirm `no_log: true` hides secrets in the output
- [ ] Confirm `git status` shows no plaintext secrets

### Capstone (no peeking at the repo)

Write one playbook or role that:

- [ ] Targets all three OS types using `ansible_facts['os_family']`
- [ ] Installs Docker and nginx
- [ ] Renders a template and restarts through a handler
- [ ] Reads a vaulted secret and uses `no_log`
- [ ] Uses a `loop` for packages and a tag for each stage
- [ ] Passes `--syntax-check` and `ansible-lint`
- [ ] Reports `changed=0` on the second run
- [ ] Is checked first with `--check --diff`

Self-check:
- Which of your tasks would break if the VM were rebooted? Is Docker enabled at boot?
- What would you change before running this on production?

You're done when: the capstone runs clean on all three OS types and is idempotent.

---

## Validate your work (every day)

```bash
ansible-playbook <file>.yml --syntax-check      # YAML and structure only, no connection
ansible-lint <file>.yml                         # style and best practices
ansible-playbook <file>.yml --list-hosts        # which hosts it will touch
ansible-playbook <file>.yml --list-tasks        # which tasks it will run
ansible-playbook <file>.yml --check --diff      # dry run, shows file changes
ansible-playbook <file>.yml                     # the real run
ansible-playbook <file>.yml                     # run it again: expect changed=0
```

There is no `--validate` flag. Use `--syntax-check`.

## Housekeeping

- [ ] Run `terraform destroy` at the end of each day to avoid EC2 charges
- [ ] Update your notes

## Stretch goals

- [ ] `community.docker` modules instead of `command: docker ...`
- [ ] `ansible-lint` in CI
- [ ] Dynamic AWS inventory
- [ ] `serial` for rolling updates
- [ ] Molecule to test a role

## Progress log

| Day | Date | Done | Time spent | Blockers |
|-----|------|------|------------|----------|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |
| 4 | | | | |
| 5 | | | | |
| 6 | | | | |
| 7 | | | | |
