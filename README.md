# TAK Server Ansible Deployment

Automated deployment of TAK Server on Ubuntu/Debian using Ansible. This project converts the interactive [installTAK](https://github.com/myTeckNet/installTAK) bash installer into a fully automated, repeatable Ansible role.

## Prerequisites

- **Control node**: Ansible 2.14+ with the `community.general` collection
- **Target host**: Ubuntu 22.04 LTS or 24.04 LTS (Debian 12 also supported)
- **TAK Server .deb**: Place the installer (e.g., `takserver_5.3-RELEASE4_all.deb`) in the project root or set `tak_deb_file` to its path
- **Minimum RAM**: 8 GB on the target host
- **SSH access**: Passwordless sudo on the target

## Quick Start

1. Clone the repo and place your TAK Server `.deb` in the project root:

```bash
git clone https://github.com/cwilliams001/TAK-Ansible.git
cd TAK-Ansible
cp /path/to/takserver_5.3-RELEASE4_all.deb .
```

2. Edit `inventory.ini` with your target host:

```ini
[tak_servers]
192.168.1.100 ansible_user=ubuntu
```

3. Customize variables in `group_vars/tak_servers.yml`:

```yaml
cert_country: "US"
cert_state: "Virginia"
cert_city: "McLean"
cert_organization: "MyOrg"
enable_cert_enrollment: true
enable_federation: true
```

4. Run the playbook:

```bash
ansible-playbook -i inventory.ini site.yml
```

5. Retrieve your artifacts from `fetched_files/<hostname>/`:
   - `webadmin.p12` - Admin certificate for browser import
   - `enrollmentDP.zip` - DataPackage for ATAK/iTAK client enrollment
   - `caCert.p12` - CA truststore (if enrollment is disabled)

## Project Structure

```
TAK-Ansible/
├── site.yml                          # Main playbook
├── inventory.ini                     # Target hosts
├── group_vars/
│   └── tak_servers.yml               # User-configurable overrides
└── roles/
    └── takserver/
        ├── defaults/main.yml         # All default variable values
        ├── handlers/main.yml         # Service restart handler
        ├── tasks/
        │   ├── main.yml              # Task orchestration
        │   ├── prerequisites.yml     # System packages, PostgreSQL repo
        │   ├── install.yml           # TAK .deb installation
        │   ├── certificates.yml      # Full PKI chain generation
        │   ├── coreconfig.yml        # CoreConfig.example.xml manipulation
        │   ├── federation.yml        # Federation XML configuration
        │   ├── service.yml           # Service start and readiness wait
        │   ├── enrollment.yml        # DataPackage creation and cert fetch
        │   └── firewall.yml          # UFW firewall rules
        └── templates/
            ├── cert-metadata.sh.j2   # Certificate DN configuration
            ├── config.pref.j2        # ATAK client connection preferences
            └── MANIFEST.xml.j2       # DataPackage manifest
```

## Variables Reference

All variables are defined in `roles/takserver/defaults/main.yml` and can be overridden in `group_vars/tak_servers.yml` or on the command line with `-e`.

### TAK Server Package

| Variable | Default | Description |
|----------|---------|-------------|
| `tak_version` | `"5.3-RELEASE4"` | TAK Server version string used to locate the `.deb` file |
| `tak_deb_file` | `"takserver_{{ tak_version }}_all.deb"` | Filename of the `.deb` package on the control node |

### Certificate Distinguished Name

| Variable | Default | Description |
|----------|---------|-------------|
| `cert_country` | `"US"` | Two-letter country code for certificate DN |
| `cert_state` | `"XX"` | State or province |
| `cert_city` | `"XX"` | City or locality |
| `cert_organization` | `"TAK"` | Organization name |
| `cert_organizational_unit` | `"TAK"` | Organizational unit |

### Certificate Authority

| Variable | Default | Description |
|----------|---------|-------------|
| `ca_root_name` | `"TAK-ROOT-CA-01"` | Name for the Root CA |
| `ca_intermediate_name` | `"TAK-ID-CA-01"` | Name for the Intermediate (issuing) CA |
| `cert_password` | `"atakatak"` | Password for all certificate keystores |
| `server_cert_name` | `"{{ ansible_hostname }}"` | Server certificate CN (defaults to target hostname) |
| `admin_cert_name` | `"webadmin"` | Admin client certificate name |

### Feature Toggles

| Variable | Default | Description |
|----------|---------|-------------|
| `enable_cert_enrollment` | `false` | Enable certificate auto-enrollment (port 8446) |
| `enable_federation` | `false` | Enable TAK Server Federation v2 (port 9001) |
| `enable_ssl` | `true` | Enable TCP/SSL client connections (port 8089) |
| `enable_quic` | `false` | Enable UDP/QUIC client connections (port 8090) |
| `enable_admin_ui` | `false` | Enable admin management UI on cert_https connector |
| `enable_webtak` | `false` | Enable WebTAK on cert_https connector |
| `enable_non_admin_ui` | `false` | Enable non-admin UI on cert_https connector |
| `enable_fips` | `false` | Pass `--fips` flag to certificate generation scripts |
| `randomize_db_password` | `true` | Generate a random PostgreSQL password in CoreConfig |
| `configure_firewall` | `true` | Configure UFW firewall rules |
| `create_enrollment_datapackage` | `true` | Build enrollment DataPackage zip when enrollment is enabled |

### DataPackage / Connectivity

| Variable | Default | Description |
|----------|---------|-------------|
| `server_description` | `"TAK Server"` | Description shown to TAK clients in the connection list |
| `server_endpoint` | `""` | Server IP/FQDN for client connections (auto-detected if empty) |
| `readiness_timeout` | `600` | Seconds to wait for TAK Server to become ready |

### Paths (advanced)

| Variable | Default | Description |
|----------|---------|-------------|
| `tak_dir` | `"/opt/tak"` | TAK Server installation directory |
| `tak_certs_dir` | `"{{ tak_dir }}/certs"` | Certificate scripts directory |
| `tak_cert_files_dir` | `"{{ tak_dir }}/certs/files"` | Generated certificate files directory |
| `tak_user` | `"tak"` | System user that owns TAK Server files |
| `tak_group` | `"tak"` | System group for TAK Server |

## Tags

Run specific sections of the playbook with `--tags`:

```bash
ansible-playbook -i inventory.ini site.yml --tags certificates
ansible-playbook -i inventory.ini site.yml --tags coreconfig
ansible-playbook -i inventory.ini site.yml --tags firewall
```

Available tags: `prerequisites`, `install`, `certificates`, `coreconfig`, `federation`, `service`, `enrollment`, `firewall`

## Example Configurations

### Minimal (SSL only, no enrollment)

```yaml
# group_vars/tak_servers.yml
cert_country: "US"
cert_state: "Virginia"
cert_city: "McLean"
cert_organization: "MyOrg"
```

### Full-featured (enrollment + federation + WebTAK)

```yaml
# group_vars/tak_servers.yml
cert_country: "US"
cert_state: "Virginia"
cert_city: "McLean"
cert_organization: "MyOrg"
cert_organizational_unit: "Operations"
ca_root_name: "MYORG-ROOT-CA-01"
ca_intermediate_name: "MYORG-ID-CA-01"
cert_password: "strongpassword"
enable_cert_enrollment: true
enable_federation: true
enable_ssl: true
enable_quic: true
enable_admin_ui: true
enable_webtak: true
server_endpoint: "tak.example.com"
```

### FIPS mode

```yaml
# group_vars/tak_servers.yml
enable_fips: true
cert_password: "fips-compliant-password"
```

## Importing the Admin Certificate

After the playbook completes, the `webadmin.p12` admin certificate is fetched to `fetched_files/<hostname>/webadmin.p12` on your control node. Import it into your browser to access the TAK Server admin UI at `https://<server-ip>:8443`.

### Chrome / Edge

1. Go to **Settings > Privacy and Security > Security > Manage certificates**
2. Click **Import** under the **Your certificates** tab
3. Select the `webadmin.p12` file
4. Enter the certificate password (default: `atakatak` or your `cert_password` value)
5. Navigate to `https://<server-ip>:8443` and select the imported certificate when prompted

### Firefox

1. Go to **Settings > Privacy & Security > Certificates > View Certificates**
2. Under the **Your Certificates** tab, click **Import**
3. Select the `webadmin.p12` file and enter the certificate password
4. Navigate to `https://<server-ip>:8443` and select the certificate when prompted

### macOS (Safari / System-wide)

1. Double-click the `webadmin.p12` file to open it in Keychain Access
2. Enter the certificate password when prompted
3. The certificate will appear in your **login** keychain
4. Navigate to `https://<server-ip>:8443` in Safari

### Command line (curl)

```bash
curl -k --cert-type P12 --cert fetched_files/<hostname>/webadmin.p12:<cert_password> \
  https://<server-ip>:8443/Marti/api/version
```

## Security Notes

- The default certificate password `atakatak` is well-known. Change `cert_password` for any non-lab deployment.
- Consider using `ansible-vault` to encrypt `group_vars/tak_servers.yml` if it contains sensitive passwords:
  ```bash
  ansible-vault encrypt group_vars/tak_servers.yml
  ansible-playbook -i inventory.ini site.yml --ask-vault-pass
  ```
- Admin certificates (`webadmin.p12`) are fetched to the control node under `fetched_files/`. Protect this directory.
- The `randomize_db_password` option (enabled by default) generates a unique PostgreSQL password per deployment.

## Firewall Ports

When `configure_firewall: true`, UFW rules are applied based on enabled features:

| Port | Protocol | Feature | Condition |
|------|----------|---------|-----------|
| 22 | TCP | SSH | Always |
| 8089 | TCP | SSL client connections | `enable_ssl: true` |
| 8090 | UDP | QUIC client connections | `enable_quic: true` |
| 8443 | TCP | Web Admin API | Always |
| 8446 | TCP | Certificate enrollment | `enable_cert_enrollment: true` |
| 9001 | TCP | Federation v2 | `enable_federation: true` |

## Mapping to installTAK

This role automates every interactive dialog prompt from the installTAK bash installer:

| installTAK Dialog | Ansible Variable |
|-------------------|-----------------|
| Certificate Properties form | `cert_country`, `cert_state`, `cert_city`, `cert_organization`, `cert_organizational_unit` |
| Certificate password prompt | `cert_password` |
| Root CA name | `ca_root_name` |
| Intermediate CA name | `ca_intermediate_name` |
| Certificate auto-enrollment | `enable_cert_enrollment` |
| Federation enable | `enable_federation` |
| Connection type (SSL/QUIC) | `enable_ssl`, `enable_quic` |
| WebTAK options | `enable_admin_ui`, `enable_webtak`, `enable_non_admin_ui` |
| DataPackage server description | `server_description` |
| DataPackage server endpoint | `server_endpoint` |
