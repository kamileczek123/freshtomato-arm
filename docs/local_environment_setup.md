# Local Environment Setup

Guide for configuring a local workstation to run the Ansible `nvram` role against FreshTomato routers.

## Prerequisites

| Component  | Minimum Version | Notes                              |
|------------|-----------------|------------------------------------|
| Python 3   | 3.9+            | required by Ansible                |
| pip        | latest          | Python package manager             |
| Ansible    | 2.14+           | configuration management           |

## Setup Python Virtual Environment

1. Clone this `frestomato-arm` repository with release tage `2025.5.1`, adjust the tag to specific release you need

```bash
git clone --branch 2025.5.1 https://github.com/kamileczek123/freshtomato-arm.git
```

2. Create and activate a dedicated virtual environment for the project

```bash
cd freshtomato-arm

python3 -m venv .venv
source .venv/bin/activate
```

## Install Python Packages

Install the packages listed in `requirements.txt`:

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

The file pulls in:

| Package | Purpose |
| --- | --- |
| `pip_search` | search PyPI from the CLI |
| `pypisearch` | alternative PyPI search utility |
| `pexpect` | required by Ansible `expect` module |
| `passlib` | password hashing (used by `community.general` filters) |
| `ansible` | Ansible tool |

## Install Ansible Collections

Install the collections listed in `requirements.yml`:

```bash
ansible-galaxy collection install -r requirements.yml
```

This installs:

| Collection | Purpose |
| --- | --- |
| `community.general` | provides extra modules and filters used by the role |

## Ansible Configuration

The playbook expects Ansible to locate the vault password file automatically.
Create (or verify) `~/.ansible.cfg`:

```ini
[defaults]
vault_password_file = ~/.ansible.vault_freshtomato-arm
```

Then store the vault password in the referenced file:

```bash
echo 'YOUR_VAULT_PASSWORD' > ~/.ansible.vault_freshtomato-arm
chmod 600 ~/.ansible.vault_freshtomato-arm
```

> The vault file is used to decrypt `config/vaultfile.yml`, which contains sensitive variables such as Wi-Fi passwords, credentials, VPN keys, etc.
