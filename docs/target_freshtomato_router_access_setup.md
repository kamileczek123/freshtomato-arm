# Target FreshTomato Router Access Setup

Guide for configuring access to target for Ansible playbook.

## Prerequisites

| Component   | Minimum Version | Notes                                                |
|-------------|-----------------|------------------------------------------------------|
| SSH client  | any             | OpenSSH bundled with macOS / Linux, PuTTY on Windows |
| SSH keypair | rsa             | keypair generated in ssh-keygen (macOS / Linux) or   |
|             |                 | PuTTYgen on Windows

## SSH Access

The playbook connects as `root` directly (see `main.yml`):

```yaml
remote_user: root
become: true
become_user: root
```

Make sure your public SSH key, from the machine from which you will be calling this Ansible playbook, is deployed on the target router and that you can
connect without a password prompt:

```bash
ssh root@<router_ip> -p <port>
```

### Configure via Router Web UI

FreshTomato uses Dropbear SSH; upload your public key through the
*Administration → Admin Access → SSH Daemon → Authorized Keys* page.
Paste your public ssh key there and click save button.

### Configure via Terminal

1. Login into router via ssh
2. Configure SSH public keys via `nvram` command, add multiple keys separately one per new line, be careful, this command overwrites any keys already configured in the router 

```bash
nvram set sshd_authkeys='
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIExampleKey1 user1@laptop
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQExampleKey2 user2@desktop
'
nvram commit
```

## Ansible Inventory

Two inventory example files are provided:

| File | Scope | Hosts |
| --- | --- | --- |
| `hosts.ini` | collection of multiple hosts processed in a batch, example file | multiple routers |
| `router1.lo.ini` | single host (`192.168.1.1:22`), example file | router1.lo |

Edit network connecton parameters to router in one or both inventory files so ansible could reach target router. The following parameters can be used:

- `ansible_host` - router hostname or ip address, FreshTomato default is `192.168.1.1`, if your network has DNS server running and e.g. `router1.lo` is a FQDN of the router, example host inventory configuration used in this code follows the Ansible practice, see inventory Ansible documentation for more details
- `ansible_port` - router ssh port, default `22`
- `ansible_ssh_private_key_file` - path to your private SSH key, default is `~/.ssh/id_rsa`, pubic part of this key should be already installed in the router, this parameter is optional then custom path to the key file is required
- `ansible_ssh_common_args` - additional optional SSH connection options, e.g.  `'-o StrictHostKeyChecking=no'`

## Test Ansible connetion to router

Assuming that you are in python3 vritual environment (`.venv`), and `router1.lo.ini` inventory is used call the following command:

```bash
ansible all -e "ansible_python_interpreter=/opt/bin/python3" -m ansible.builtin.ping -i router1.lo.ini

# output
router1.lo | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```
