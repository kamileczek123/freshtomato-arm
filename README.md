# freshtomato-arm

Ansible playbook for managing and configuring routers running [FreshTomato](https://github.com/FreshTomato-Project/freshtomato-arm) firmware on ARM hardware.

This repository implements the `nvram` role, which configures FreshTomato firmware via `nvram set` commands executed over SSH. Currently the most commonly used configuration parameters are covered. The role can be easily extended by adding templates, tasks, and parameters for any router service not yet implemented.

References:

- [FreshTomato ARM source code repository](https://github.com/FreshTomato-Project/freshtomato-arm)

Version compatibility, see appropriate releases:

- `freshtomato-arm 2025.5`
- `freshtomato-arm 2026.1`

## Documentation

Full setup and usage documentation is available in [docs/](docs/index.md):

1. [Enable USB Storage for the FreshTomato Router](docs/enable_usb_storage_for_the_freshtomato_router.md)
2. [Install Entware and Packages on the FreshTomato Router](docs/install_entware_and_packages_on_the_freshtomato_router.md)
3. [Local Environment Setup](docs/local_environment_setup.md)
4. [Target FreshTomato Router Access Setup](docs/target_freshtomato_router_access_setup.md)
5. [Configure and Run Playbook](docs/configure_and_run_playbook.md)
6. [Appendix - Ansible nvram role README.md](roles/nvram/README.md).

## Prerequisites

The following must be completed before running the playbook:

1. `SWAP` partition and `/opt` mount configured on the router's USB storage
2. Entware, `openssh-sftp-server` and `python3` installed on the router
3. Ansible and dependencies installed on the local workstation
4. SSH key-based access to the target router configured
5. Ansible inventory and host variables configured

## Role Structure

```text
roles/nvram/
├── README.md             # role documentation
├── defaults/
│   └── main.yml          # feature toggles (all false by default) and nvram default values
├── vars/
│   └── main.yml          # template path mappings and internal role variables
├── tasks/
│   ├── main.yml          # task dispatcher — imports task files based on feature toggles
│   ├── reboot.yml        # commits nvram changes and reboots router with wait_for_connection
│   └── *.yml             # individual task files per FreshTomato configuration scope
└── templates/
    ├── administration/   # admin_access, logging, scheduler, scripts, tomatoanon, snmp, rstats, cstats
    ├── advanced/         # dhcp_dns, firewall, vlan, virtual_wireless, wireless
    ├── basic/            # network, time, identification
    ├── port_forwarding/  # basic (port forwards), upnp_nat_pmp
    ├── usb_nas/          # usb_support, scripts/
    └── vpn/              # openvpn_server_1, openvpn_server_2, wireguard_wg0, wireguard_wg1
```

## Available Tags

### Tags requiring reboot

Always combine these with the `reboot` tag:

| Tag | FreshTomato scope |
| --- | --- |
| `network` | Basic > Network |
| `identification` | Basic > Identification |
| `virtual_wireless` | Advanced > Virtual Wireless |
| `wireless` | Advanced > Wireless |
| `vlan` | Advanced > VLAN |
| `dhcp_dns` | Advanced > DHCP/DNS |
| `scripts` | Administration > Scripts |

### Tags with service restart

| Tag | FreshTomato scope | Service(s) restarted |
| --- | --- | --- |
| `usb_support` | USB and NAS > USB Support | `usb` |
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

### Commit only

| Tag | FreshTomato scope | Notes |
| --- | --- | --- |
| `admin_access` | Administration > Admin Access | Takes effect on next web/SSH session |

### Utility

| Tag | Description |
| --- | --- |
| `reboot` | Commits nvram and reboots the router (~5 min); always run last |

## Usage

Configure a single service:

```bash
ansible-playbook ./main.yml -i ./router1.lo.ini --tags "firewall"
```

Configure network settings (requires reboot):

```bash
ansible-playbook ./main.yml -i ./router1.lo.ini --tags "network,reboot"
```

Dry run (check mode):

```bash
ansible-playbook ./main.yml -i ./router1.lo.ini --tags "usb_support" --check --diff
```

Full deployment of all enabled services:

```bash
ansible-playbook ./main.yml -i ./router1.lo.ini \
    --tags "usb_support,admin_access,network,identification,time,dnsmasq,dhcp_dns,virtual_wireless,wireless,firewall,vlan,pf_basic,pf_upnp_nat_pmp,scripts,logging,scheduler,rstats,cstats,tomatoanon,snmp,openvpn_server_1,openvpn_server_2,wireguard_wg0,wireguard_wg1"
```

## Ansible Vault

Sensitive variables (Wi-Fi passwords, WireGuard keys, SSH authorized keys, OpenVPN certificates) are stored encrypted in `config/vaultfile.yml`. See [Local Environment Setup](docs/local_environment_setup.md) for vault configuration.

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

## Information about code versioning

The versioning will follow the FreshTomato Firmware versioning, i.e. for firmware release `2025.5` the `nvram` role release will be `2025.5.X`  where `X` is the number of each following release of the `nvram` role (improved or extended) compatible to firmware release.

## Tested Routers

The playbook so far was tested with following router hardware:

|  Brand  |   Model   |
|---------|-----------|
| Netgear | R7000     |
| Netgear | R8000     |
| Asus    | RT-AC68U  |
| Asus    | RT-AC3200 |
