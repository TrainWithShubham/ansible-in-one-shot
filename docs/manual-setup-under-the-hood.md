# Manual setup: what the bootstrap does under the hood

The Quick Start uses Terraform and `playbooks/bootstrap_control_node.yml` to build the lab for you.
This page does the same thing by hand, so you understand what each step is for.
It is optional. It is also how you would set Ansible up on servers you already have,
without Terraform.

You need two or more Linux servers you can SSH into. In the examples below the control node is
Ubuntu, and the managed nodes are called `node-1` (Ubuntu) and `node-2` (RHEL). Replace the
placeholders (`<...>`) with your own values.

## Map: manual step to what the repo automates

| Manual step | Automated by |
|-------------|--------------|
| 1. Install Ansible on the control node | `Install Ansible` tasks in `bootstrap_control_node.yml` |
| 2. Create an SSH key | The Quick Start generates one key pair on your laptop; `bootstrap_control_node.yml` copies the private key to the control node (`~/keys/terra-key-ansible`) |
| 3. Authorize the key on every managed node | Terraform: `aws_key_pair` registers the public key and attaches it to all 4 instances |
| 4. Write an inventory | `terraform/templates/inventory.tpl` generates `inventories/dev/hosts.ini` |
| 5. Test with ping | `ansible all -m ping` |

## 1. Install Ansible on the control node (Ubuntu)

```bash
sudo apt update
sudo apt install -y software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
ansible --version
```

The PPA gives a current `ansible-core`. The version in Ubuntu's default repos is older.

## 2. Create an SSH key on the control node

```bash
ssh-keygen -t ed25519          # accept the defaults, empty passphrase for a lab
cat ~/.ssh/id_ed25519.pub      # this is the PUBLIC key; safe to copy
```

Ansible connects over plain SSH, so the control node needs a key that the managed nodes trust.

## 3. Authorize the key on each managed node

The managed nodes must have the control node's public key in `~/.ssh/authorized_keys`.
On EC2 you usually already have a key pair (`.pem`) that lets you in, so use that to install the new one:

```bash
# from your laptop, using the key pair that already works
ssh-copy-id -f -i ~/.ssh/id_ed25519.pub \
  -o "IdentityFile=/path/to/existing-key.pem" \
  <user>@<managed-node-address>
```

Or append it directly:

```bash
ssh -i /path/to/existing-key.pem <user>@<managed-node-address> \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys" \
  < ~/.ssh/id_ed25519.pub
```

Notes:
- `ssh-copy-id -i` takes the key you are installing. The key you log in with goes in `-o IdentityFile=`.
- The login user depends on the OS: `ubuntu` on Ubuntu, `ec2-user` on RHEL and Amazon Linux.
- It must be an OpenSSH public key (`ssh-ed25519 AAAA...` or `ssh-rsa AAAA...`). A PEM public key
  (`-----BEGIN PUBLIC KEY-----`) will not work. Convert it with `ssh-keygen -i -m PKCS8 -f key.pem`.
- Private keys must be `chmod 400` or `600`, or SSH refuses to use them.

Confirm each node from the control node before involving Ansible:

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@<node-1-address>
ssh -i ~/.ssh/id_ed25519 ec2-user@<node-2-address>
```

If plain `ssh` doesn't work, Ansible won't either.

## 4. Write an inventory

The default inventory is `/etc/ansible/hosts`, and the file is owned by root. You can either use
`sudo vim`, change the owner (`sudo chown $USER /etc/ansible/hosts`), or better, keep your own inventory
file next to your playbooks and pass `-i`:

```ini
[servers]
node-1 ansible_host=<node-1-address> ansible_user=ubuntu
node-2 ansible_host=<node-2-address> ansible_user=ec2-user

[all:vars]
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

Check that Ansible reads it the way you expect:

```bash
ansible-inventory -i inventory.ini --list
ansible-inventory -i inventory.ini --graph
```

This repo's `ansible.cfg` already sets `inventory = inventories/dev/hosts.ini`, so you don't need `-i`
when you run commands from the repo root.

## 5. Test

```bash
ansible -i inventory.ini all -m ping
ansible -i inventory.ini all -a "date"
```

`pong` from every node means SSH, Python and the inventory are all correct. From here, continue with
the [ad-hoc cheat sheet](adhoc-cheatsheet.md) and Module 1.

## Common problems

| Symptom | Cause |
|---------|-------|
| `Permission denied (publickey)` | The public key isn't in the node's `authorized_keys`, or you are using the wrong user |
| `UNREACHABLE` | Security group or firewall blocks port 22, or the wrong address |
| `WARNING: UNPROTECTED PRIVATE KEY FILE` | Private key permissions too open. Run `chmod 400 <key>` |
| `No hosts matched` | The inventory isn't the one you think. Check with `ansible-inventory --graph` |
| `/etc/ansible/hosts: Permission denied` when saving | The file is root-owned. Use `sudo` or your own `-i` file |
