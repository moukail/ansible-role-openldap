# Ansible Role: `openldap`

Installs, configures, and initializes an OpenLDAP (`slapd`) directory server. The role checks existing system and custom-prefix installations first, attempts installation via the OS package manager (`dnf` on RedHat family, `apt` on Debian family), falls back to compiling OpenLDAP from checksum-verified upstream source if needed, bootstraps a `cn=config` + `mdb` database, and initializes the base Directory Information Tree (DIT).

## Requirements

- **Ansible:** `ansible-core >= 2.16`
- **Supported OS Families:**
  - RedHat (e.g., AlmaLinux 9, Rocky Linux 9, RHEL 9)
  - Debian (e.g., Debian 12, Ubuntu 22.04/24.04)
- **Init System:** `systemd` (for `slapd.service`)
- **Privileges:** Privilege escalation (`become: true`) is required for package installation, `make install`, user/directory creation, and systemd service management.
- **Network Access:** Required to reach OS repositories and `openldap_download_url` (when compiling from source).

## Role Variables

Available variables are listed below along with their defaults (see [`defaults/main.yaml`](defaults/main.yaml) and [`meta/argument_specs.yml`](meta/argument_specs.yml)):

### Installation Variables

| Variable | Type | Required | Default | Description |
| :--- | :--- | :---: | :--- | :--- |
| `openldap_version` | `str` | No | `"2.6.16"` | Minimum OpenLDAP version required on the target host. |
| `openldap_install_prefix` | `path` | No | `/usr/local` | Prefix used when compiling OpenLDAP from source and locating custom binaries (`<prefix>/libexec/slapd`, `sbin/`, `bin/`). |
| `openldap_download_url` | `str` | No | `https://www.openldap.org/software/download/OpenLDAP/openldap-release/openldap-{{ openldap_version }}.tgz` | URL of the OpenLDAP source archive (`.tgz`) used for source compilation. |
| `openldap_source_checksums` | `dict` | No | *(map for `2.6.13`, `2.6.16`, `2.7.2`)* | Version-to-SHA-256 map used to resolve `openldap_source_checksum` automatically for known releases. |
| `openldap_source_checksum` | `str` | No | `{{ openldap_source_checksums[openldap_version] \| default('') }}` | Checksum specification passed to `ansible.builtin.get_url` (`<algorithm>:<digest>` or `<algorithm>:<url>`). Automatically resolved from `openldap_source_checksums`, or skipped when empty. |

### Configuration & Initialization Variables

| Variable | Type | Required | Default | Description |
| :--- | :--- | :---: | :--- | :--- |
| `openldap_root_password` | `str` | **Yes** | *(none)* | Administrator password for `openldap_root_dn` (hashed with `slappasswd -h '{SSHA}'`; marked `no_log`). Pass via Ansible Vault or play/inventory vars. |
| `openldap_base_dir` | `path` | No | `/opt/openldap` | Base directory containing `data/` (`0700`), `config/` (`0750`), `certs/` (`0750`), and bootstrap LDIF files. |
| `openldap_base_dn` | `str` | No | `dc=moukail,dc=nl` | Base Distinguished Name (MDB `olcSuffix`) of the directory. |
| `openldap_organization` | `str` | No | `Moukail` | Organization (`o` attribute) on the base DN entry. |
| `openldap_user` | `str` | No | `ldap` | System user owning OpenLDAP directories and running `slapd`. |
| `openldap_group` | `str` | No | `ldap` | System group owning OpenLDAP directories and running `slapd`. |
| `openldap_root_dn` | `str` | No | `cn=admin,{{ openldap_base_dn }}` | Root administrator DN (`olcRootDN`) for the MDB database. |
| `openldap_schema_dir` | `path` | No | `{{ openldap_install_prefix }}/etc/openldap/schema` | Path to OpenLDAP LDIF schema files (`core`, `cosine`, `nis`, `inetorgperson`). |

## How It Works

1. **Version Check (`tasks/check.yaml`):** Probes `/usr/sbin/slapd -VV` and `{{ openldap_install_prefix }}/libexec/slapd -VV` to determine whether an installed version already satisfies `>= openldap_version`.
2. **Package Installation (`tasks/install_package.yaml`):** If installation is required, attempts to install:
   - `epel-release` and `openldap-servers >= {{ openldap_version }}` via `dnf` on RedHat systems.
   - `slapd>={{ openldap_version }}` via `apt` on Debian systems.
3. **Source Compilation Fallback (`tasks/compile.yaml`):** If package installation fails or cannot meet the requested minimum version, installs build dependencies (`gcc`, `make`, `tar`, `gzip`, `groff-base`, `openssl-devel`/`libssl-dev`), downloads and SHA-256-verifies the source archive into a temporary directory, compiles and installs into `openldap_install_prefix`, and removes the temporary build directory in an `always:` block.
4. **Configuration (`tasks/configure.yaml`):** Creates the `openldap_user`/`openldap_group` system account and `openldap_base_dir` subdirectories (`data`, `config`, `certs`), installs `slapd.service`, hashes `openldap_root_password`, renders `slapd.ldif`, and bootstraps the `cn=config` database with `slapadd -n 0`.
5. **Initialization (`tasks/initialize.yaml`):** Renders `base.ldif` (`openldap_base_dn`, `ou=people`, `ou=groups`), checks whether `openldap_base_dn` already exists via `ldapsearch`, and loads `base.ldif` via `ldapadd` when absent.

## Dependencies

None.

## Example Playbook

```yaml
---
- name: Provision OpenLDAP
  hosts: openldap
  become: true

  tasks:
    - name: Install, configure, and initialize OpenLDAP
      ansible.builtin.include_role:
        name: openldap
      vars:
        openldap_version: "2.6.16"
        openldap_base_dn: dc=moukail,dc=nl
        openldap_organization: Moukail
        openldap_root_password: "{{ vault_openldap_root_password }}"
```

## Testing

This role includes a [Molecule](molecule/default/molecule.yml) scenario using **Podman** with systemd-enabled **AlmaLinux 9** (`almalinux/9-init`) and **Debian 12** (`geerlingguy/docker-debian12-ansible`) containers. It verifies convergence, idempotence, `slapd -VV` version, directory permissions, `cn=config` bootstrap, `slapd.service` status, port `389`, and `ldapsearch` for the base DN and OUs.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install molecule "molecule-plugins[podman]"
ansible-galaxy collection install containers.podman ansible.posix

cd roles/openldap
molecule test
```

Inspect documented role arguments via `ansible-doc`:

```bash
ANSIBLE_ROLES_PATH=./roles ansible-doc -t role openldap
```
