# This role configures and manages a router running FreshTomato firmware based on freshtomato-arm

## Introduction

This role allows configuration of hardware running FreshTomato firmware via `nvram set` commands executed over SSH.
This implementation covers `freshtomato-arm` architecture, and currently only commonly/mostly used configuration parameters are implemented. Configuration parts not covered in this role can be easily extended by adding templates, tasks and parameters covering needed router services.

References:
- [Router services available via CLI](https://www.linksysinfo.org/index.php?threads/starting-stopping-and-restarting-services-from-cli-command-line-via-ssh-or-telnet.26491/)
- [FreshTomato ARM source code repository](https://github.com/FreshTomato-Project/freshtomato-arm)

Version compatibility:

- `freshtomato-arm 2025.5`

## Prerequisites

Full setup and usage documentation is available in [docs/](docs/index.md):

1. Configured `SWAP` partition and `/opt` mount -> see [Enable USB Storage for the FreshTomato Router](docs/enable_usb_storage_for_the_freshtomato_router.md) documentation
2. Installed Entware, `openssh-sftp-server` and `python3` packages -> see [Enable USB Storage for the FreshTomato Router](docs/install_entware_and_packages_on_the_freshtomato_router.md)
3. Installed Ansible and dependiencies on local environment -> see [Local Environment Setup](docs/local_environment_setup.md))
4. Configured accees to target router -> see [Target FreshTomato Router Access Setup](docs/target_freshtomato_router_access_setup.md)
5. Playbook configured -> see [Configure and Run Playbook](docs/configure_and_run_playbook.md)

## Role structure

```text
roles/nvram/
├── README.md             # this file
├── defaults/
│   └── main.yml          # feature toggles (all false by default) and nvram default values
├── vars/
│   └── main.yml          # template path mappings and internal role variables
├── tasks/
│   ├── main.yml          # task dispatcher, imports task files based on feature toggles
│   ├── reboot.yml        # commits nvram changes and reboots router with wait_for_connection
│   └── *.yml             # individual task files for each FreshTomato configuration scope
└── templates/
    ├── administration/   # admin_access, logging, scheduler, scripts, tomatoanon, snmp, rstats, cstats
    ├── advanced/         # dhcp_dns, firewall, vlan, virtual_wireless, wireless
    ├── basic/            # network, time, identification
    ├── port_forwarding/  # basic (port forwards), upnp_nat_pmp
    ├── usb_nas/          # usb_support, scripts/
    └── vpn/              # openvpn_server_1, openvpn_server_2, wireguard_wg0, wireguard_wg1
```

## Variable architecture

This role uses a three-layer variable system:

1. **`roles/nvram/defaults/main.yml`** (lowest precedence) — contains:
   - Feature toggles (`network: false`, `vlan: false`, etc.) controlling which tasks run
   - Default values for every nvram variable used in templates (safe factory-like defaults)

2. **`roles/nvram/vars/main.yml`** (role-level) — contains:
   - Template path mappings (`admin_access_nvram: "administration/admin_access.nvram.j2"`)
   - Internal role variables (`opt_dir`, `mnt_backup_dir`, `host_nvram_dir`)
   - USB boot script definitions (`usb_boot_default_scripts`)

3. **`group_vars/<router_fqdn>/nvram.yml`** (host-level, overrides defaults) — contains:
   - Feature toggles set to `true` for services configured on that router
   - Host-specific values that differ from defaults (IPs, WiFi SSIDs, passwords, WireGuard keys, port forwards, etc.)

All `.nvram.j2` templates use Jinja2 variables (`{{ variable_name }}`). If a host doesn't override a variable in `group_vars`, the default from `defaults/main.yml` is used.

### Encrypted variables

Sensitive values are stored as Ansible Vault encrypted strings in `group_vars` or `config/vaultfile.yml`:

- WiFi passwords (e.g. `{{ wl0_wpa_psk }}`)
- WireGuard private/preshared keys (e.g. `{{ wg0_priv_key }}`)
- SSH authorized keys (`{{ sshd_authkeys }}`)
- OpenVPN certificate files (vault-encrypted files in `files/`)

### Variable naming for special characters

Some nvram keys contain dots (e.g. `wl0.1_ifname`). In templates the nvram key stays as-is, but the Ansible variable uses an underscore instead of the dot:

- nvram key: `wl0.1_ifname` → Ansible variable: `wl0_1_ifname`
- Template: `nvram set wl0.1_ifname='{{ wl0_1_ifname }}'`

Hardware-specific power calibration keys (e.g. `1:maxp2ga0`, `pci/1/1/maxp2ga0`) use descriptive variable names:

- `1:maxp2ga0` → `r8000_1_maxp2ga0`
- `pci/1/1/maxp2ga0` → `r7000_pci_1_1_maxp2ga0`

## Deploy workflow

1. Tasks set nvram values by running template-rendered shell commands on the router
2. Each task commits changes with `nvram commit`
3. Depending on the scope, changes are applied by either:
   - **Service restart** — takes effect immediately (e.g. `service firewall restart`)
   - **Router reboot** — required for low-level changes (network interfaces, VLAN, wireless, init scripts)

## Reboot behaviour

### Tasks requiring reboot

The following tasks only commit changes to nvram but **require a router reboot** for them to take effect:

| Tag | FreshTomato scope | Why reboot is needed |
| --- | --- | --- |
| `network` | Basic > Network | WAN/LAN interfaces, IP addresses, WiFi radios |
| `identification` | Basic > Identification | Router hostname, domain |
| `virtual_wireless` | Advanced > Virtual Wireless | Virtual WiFi interfaces (wl0.1, wl1.1, wl2.1) |
| `wireless` | Advanced > Wireless | Radio power, country code, hardware calibration |
| `vlan` | Advanced > VLAN | VLAN port assignments, interface bindings |
| `dhcp_dns` | Advanced > DHCP/DNS | DHCP server, DNS forwarder configuration |
| `scripts` | Administration > Scripts | Init, firewall, WAN-up scripts |

**Always include the `reboot` tag** when using any of the above tags.

### Reboot task implementation

The `reboot` tag (`tasks/reboot.yml`) performs:

1. `nvram commit` — final commit before reboot
2. `sleep 2 && reboot` — runs asynchronously (`async: 1, poll: 0`) so Ansible doesn't hang on the dying SSH connection
3. `wait_for_connection` — waits ~2.5 minutes (`delay: 150`) for the router to finish rebooting, then polls for SSH availability (total `timeout: 300` seconds)

> **Note:** `kill -HUP 1` was tested as a lighter alternative to `reboot` but causes the router to hang when USB/NAS is configured. Full `reboot` is used instead.

### Tasks with service restart (no reboot needed)

These tasks commit changes and restart their respective service(s) immediately:

| Tag | FreshTomato scope | Service(s) restarted |
| --- | --- | --- |
| `usb_support` | USB and NAS > USB Support | `usb` (with Entware stop/mount cycle) |
| `time` | Basic > Time | `ntpd`, `dnsmasq`, `firewall` |
| `pf_basic` | Port Forwarding > Basic | `firewall` |
| `pf_upnp_nat_pmp` | Port Forwarding > UPnP/NAT-PMP | `upnp` |
| `firewall` | Advanced > Firewall | `firewall` |
| `tomatoanon` | Administration > TomatoAnon | `tomatoanon` |
| `dnsmasq` | Advanced > DHCP/DNS (config file) | `dnsmasq` |
| `logging` | Administration > Logging | `logging` |
| `rstats` | Administration > Bandwidth Monitoring | `rstats` |
| `cstats` | Administration > IP Traffic Monitoring | `rstats` |
| `scheduler` | Administration > Scheduler | `sched` |
| `snmp` | Administration > SNMP | `snmp`, `firewall` |
| `openvpn_server_1` | VPN > OpenVPN Server 1 | `vpnserver1` |
| `openvpn_server_2` | VPN > OpenVPN Server 2 | `vpnserver2` |
| `wireguard_wg0` | VPN > Wireguard wg0 | `wireguard0` |
| `wireguard_wg1` | VPN > Wireguard wg1 | `wireguard1` |

### Tasks with commit only (no restart, no reboot)

| Tag            | FreshTomato scope             | Notes                                        |
| ---            | ---                           | ---                                          |
| `admin_access` | Administration > Admin Access | Settings take effect on next web/SSH session |

### Task ordering and dependencies

Tasks run in a fixed order defined in `tasks/main.yml`. Some tasks have dependencies:

- file processing tasks require `usb_support`, `admin_access`, and `scheduler` to run first
- `reboot` always runs last

## Role usage

### Command examples

Configure a single service:

```bash
ansible-playbook ./main.yml -i ./routers.ini --tags "firewall"
```

Configure network settings (requires reboot):

```bash
ansible-playbook ./main.yml -i ./routers.ini --tags "network,reboot"
```

Configure multiple scopes requiring reboot (single reboot at end):

```bash
ansible-playbook ./main.yml -i ./routers.ini --tags "network,vlan,dhcp_dns,reboot"
```

Full deployment of all enabled services:

```bash
ansible-playbook ./main.yml -i ./routers.ini
```

### Role setup in main playbook

```yaml
    - name: Configure Freshtomato 'nvram'
      include_role:
        name: nvram
      tags:
        - usb_support
        - network
        - identification
        - time
        - virtual_wireless
        - wireless
        - pf_basic
        - pf_upnp_nat_pmp
        - firewall
        - vlan
        - admin_access
        - tomatoanon
        - dnsmasq
        - dhcp_dns
        - scripts
        - rstats
        - cstats
        - logging
        - scheduler
        - openvpn_server_1
        - openvpn_server_2
        - wireguard_wg0
        - wireguard_wg1
        - snmp
        - reboot
```

### Host configuration

Each host's `group_vars/<router_fqdn>/nvram.yml` must declare which services are active by setting the corresponding feature toggle to `true`. Only services set to `true` will be deployed when the matching tag is used.

## Ansible Vault usage

This role uses `./config/vaultfile.yml` in the main playbook to store sensitive variables. Verify `~/.ansible.cfg` and `~/.ansible.vault_freshtomato-arm` for Vault configuration.

Edit vault file:

```bash
ansible-vault edit ./config/vaultfile.yml
```

Encrypt a single variable:

```bash
ansible-vault encrypt_string 'secret_string' --name 'variable_name'
```

Encrypt a file (e.g. OpenVPN certificates):

```bash
ansible-vault encrypt './files/<router_fqdn>/nvram/openvpn_server_1/ca.crt'
```

File decryption is done during `copy` tasks using `decrypt: yes` parameter.
