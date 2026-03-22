# FreshTomato ARM — Documentation

This documentation guides you through the full setup required to manage a FreshTomato ARM router using the Ansible `nvram` role enclosed in this repository. It covers router-side prerequisites, local workstation setup (it is ready for macOS or Linux; for Windows, the described steps require adjustments), and finally playbook execution.

## Overview

The `nvram` Ansible role allows configuration of hardware running FreshTomato firmware via `nvram set` commands executed over SSH. This implementation covers the `freshtomato-arm` architecture; currently the most commonly used configuration parameters are implemented. Configuration parts not covered by this role can be easily extended by adding templates, tasks, and parameters for the needed router services.

Running Python 3 on the router (required by Ansible) demands more memory than the router provides by default — hence the need for USB-backed SWAP and Entware before any Ansible work begins.

The setup follows these sequential steps:

1. **Configure `SWAP` partition and `/opt` mount** — Prepare a USB drive with SWAP and OPT partitions on a separate host, then enable USB storage on the router and mount both partitions. See [Enable USB Storage for the FreshTomato Router](enable_usb_storage_for_the_freshtomato_router.md).

2. **Install Entware, `openssh-sftp-server` and `python3` packages** — Bootstrap Entware into `/opt` and install the packages needed for SSH file transfer and Ansible remote execution. See [Install Entware and Packages on the FreshTomato Router](install_entware_and_packages_on_the_freshtomato_router.md).

3. **Install Ansible and dependencies on local environment** — Set up a Python virtual environment on your workstation, install Ansible and its dependencies, and configure the Ansible vault. See [Local Environment Setup](local_environment_setup.md).

4. **Configure access to target router** — Deploy your SSH public key to the router and configure the Ansible inventory so the playbook can reach the target host. See [Target FreshTomato Router Access Setup](target_freshtomato_router_access_setup.md).

5. **Configure and run the playbook** — Define per-host variables, enable feature toggles, and run the `nvram` playbook to apply the desired router configuration. See [Configure and Run Playbook](configure_and_run_playbook.md).

6. **Appendix** - Ansible `nvram` role implementation documentation. See [Ansible nvram role README.md](../roles/nvram/README.md).

## Conclusion

Once all steps are complete, the router is fully managed via Ansible. Configuration changes — network settings, Wi-Fi, firewall rules, port forwarding, VPN, and more — can be applied by editing host variables and re-running the playbook with the appropriate tags. The `reboot` tag commits `nvram` and restarts the router to activate changes.
