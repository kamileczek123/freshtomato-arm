# Configure and Run Playbook

Finally, this guide covers an usage of the `nvram` Ansible role via enclosed playbook to configure FreshTomato Router parameters.

## Host Variables

Per-host variable overrides live under `group_vars/<host>/`:

```text
group_vars/
└── router1_lo/
    ├── main.yml
    └── nvram.yml      <- nvram role default variables override
```

`router1_lo/nvram.yml` is where you:

1. enable feature toggles and override the defaults defined in `roles/nvram/defaults/main.yml`, like

```bash
usb_support: true
network: true
identification: true
...
```

2. declare particular configuration parameters via available variables commited to router via CLI, e.g. LAN and WiFi networks configuration:

```bash
lan_ifname: 'br0'
lan_ipaddr: '192.168.1.1'
lan_netmask: '255.255.255.0'
lan_proto: 'static'
wl0_radio: '1'
wl0_mode: 'ap'
wl0_net_mode: 'mixed'
wl0_ssid: 'FreshTomato24'
wl0_closed: '0'
wl0_channel: '1'
wl0_nbw_cap: '0'
wl0_nctrlsb: 'upper'
wl0_security_mode: 'wpa2_personal'
wl0_crypto: 'aes'
wl0_wpa_psk: '{{ wl0_wifi_pass_FreshTomato24 }}'
wl1_radio: '1'
wl1_mode: 'ap'
wl1_net_mode: 'mixed'
wl1_ssid: 'FreshTomato51'
wl1_closed: '0'
wl1_channel: '112'
wl1_nbw_cap: '0'
wl1_nctrlsb: 'lower'
wl1_security_mode: 'wpa2_personal'
wl1_crypto: 'aes'
wl1_wpa_psk: '{{ wl1_wifi_pass_FreshTomato51 }}'
wl2_radio: '1'
wl2_mode: 'ap'
wl2_net_mode: 'mixed'
wl2_ssid: 'FreshTomato52'
wl2_closed: '1'
wl2_channel: '36'
wl2_nbw_cap: '1'
wl2_nctrlsb: 'lower'
wl2_security_mode: 'wpa2_personal'
wl2_crypto: 'aes'
wl2_wpa_psk: '{{ wl2_wifi_pass_FreshTomato52 }}'
```

where wifi passwords are stored in `config/vaultfile.yml` file in variables:

- `wl0_wifi_pass_FreshTomato24`
- `wl1_wifi_pass_FreshTomato51`
- `wl2_wifi_pass_FreshTomato52`

or multiline parametres, via single variable, e.g.:

```bash
script_init: |
  /usr/bin/logger -s "INIT: set PATH"
  export PATH=/opt/bin:/opt/sbin:/opt/usr/bin:/opt/usr/sbin:/bin:/sbin:/usr/bin:/usr/sbin
```

or port forwarding configuration in firmware specific multiple fields syntax via single variable, e.g.:

```bash
portforward: '1<1<<80<80<192.168.1.2<HTTP>1<1<<443<443<192.168.1.2<HTTPS>1<1<<10001:10010<<192.168.1.2<SFTP>'
```

## Running the Playbook

### Dry-run (check mode)

```bash
ansible-playbook ./main.yml -i ./router1.lo.ini --tags "usb_support" --check --diff
```

### Single feature (e.g. USB Support)

```bash
ansible-playbook ./main.yml -i ./router1.lo.ini --tags "usb_support"
```

Runing playbook for USB Support will enable complete and optimized USB Support configuration - high USB performance and pendrive liveness. The default role configuration for USB Support will install following scripts:

- [`S00netwait`](https://gitea.kuliberda.pl/kamil/machines/src/branch/master/freshtomato/files/_shared/nvram/usb_nas/scripts/S00netwait) script to synchronize run Entware daemons after network is configured,
- [`S01tuning`](https://gitea.kuliberda.pl/kamil/machines/src/branch/master/freshtomato/files/_shared/nvram/usb_nas/scripts/S01tuning) script which adjusts proc ipv4/ipv6 related,
- [`mount.autorun`](https://gitea.kuliberda.pl/kamil/machines/src/branch/master/freshtomato/files/_shared/nvram/usb_nas/scripts/mount.autorun) script during each boot installs `/etc/fstab` entries and triggers daemon scripts located in `/opt/etc/init.d` folder,
- [`mount.autostop`](https://gitea.kuliberda.pl/kamil/machines/src/branch/master/freshtomato/files/_shared/nvram/usb_nas/scripts/mount.autorun) script stops daemon scripts located in `/opt/etc/init.d` folder during each shutdown.

Run playbook with `reboot` to achive hardened USB Support configuration:

```bash
ansible-playbook ./main.yml -i ./router1.lo.ini --tags "usb_support, reboot"
```

### Full nvram deploy (all tags)

```bash
ansible-playbook ./main.yml -i ./router1.lo.ini \
    --tags "usb_support,admin_access,network,identification,time,dnsmasq,dhcp_dns,virtual_wireless,wireless,firewall,vlan,pf_basic,pf_upnp_nat_pmp,scripts,logging,scheduler,rstats,cstats,tomatoanon,snmp,openvpn_server_1,openvpn_server_2,wireguard_wg0,wireguard_wg1"
```

### Reboot after changes

Append the `reboot` tag to commit nvram and restart the router (≈ 5 min):

```bash
ansible-playbook ./main.yml -i ./router1.lo.ini --tags "network,reboot"
```
