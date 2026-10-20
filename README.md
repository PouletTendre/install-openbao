# Ansible Role: `install-openbao`

This Ansible role installs and configures **OpenBao** on Debian/Ubuntu systems using the official DEB822 repository, with:
- Integrated **Raft** storage backend
- **TLS** encryption (deploys the server fullchain certificate, private key, and adds the internal Root CA to the OS trust store)
- Local file audit logging (`audit "file"`)

---

## ⚠️ Security Warning (TLS Private Key)

> **Important**: The [`files/openbao.key`](files/openbao.key) file included in this repository is a **dummy test private key** for lab/testing purposes only.
>
> Before deploying to a real or production environment:
> 1. Replace [`files/openbao.key`](files/openbao.key), [`files/openbao-fullchain.crt`](files/openbao-fullchain.crt), and [`files/root-ca.crt`](files/root-ca.crt) with your actual certificates and private key.
> 2. **Encrypt the private key using Ansible Vault** before committing it to version control:
>    ```bash
>    ansible-vault encrypt files/openbao.key
>    ```
> The task copying the private key sets `no_log: true` to prevent sensitive key material from leaking into Ansible output or CI/CD logs.

---

## Requirements

- **Target OS**: Debian (12+) or Ubuntu (22.04+)
- **Ansible**: `ansible-core >= 2.15` (required for the `ansible.builtin.deb822_repository` module)
- The following files must be present in the role's `files/` directory:
  - `openbao.asc`: Official OpenBao repository GPG public key
  - `openbao-fullchain.crt`: Full TLS certificate chain for OpenBao (Server + Intermediate CA)
  - `openbao.key`: OpenBao TLS private key (encrypted with `ansible-vault` in production)
  - `root-ca.crt`: Internal Root CA certificate to install into the OS trust store

---

## Role Variables

Default variables are defined in [`defaults/main.yml`](defaults/main.yml) and can be overridden in your inventory (`host_vars` / `group_vars`) or playbook:

| Variable | Default Value | Description |
| :--- | :--- | :--- |
| `openbao_listen_address` | `"0.0.0.0:8200"` | TCP address and port OpenBao listens on |
| `openbao_api_addr` | `"https://{{ ansible_default_ipv4.address \| default('127.0.0.1') }}:8200"` | API address advertised to clients and other cluster nodes |
| `openbao_cluster_addr` | `"https://{{ ansible_default_ipv4.address \| default('127.0.0.1') }}:8201"` | Cluster address used for Raft node-to-node communication |
| `openbao_ui_enabled` | `true` | Enable or disable the OpenBao Web UI |
| `openbao_node_id` | `"{{ inventory_hostname }}"` | Unique node identifier within the Raft cluster |
| `openbao_data_dir` | `"/var/lib/openbao/data"` | Directory where Raft storage data is persisted |
| `openbao_tls_dir` | `"/etc/openbao/tls"` | Directory where TLS certificate and private key are stored |
| `openbao_log_dir` | `"/var/log/openbao"` | Directory where OpenBao audit logs are stored |
| `openbao_config_path` | `"/etc/openbao/openbao.hcl"` | Path to the main OpenBao HCL configuration file |
| `openbao_tls_cert_src` | `"openbao-fullchain.crt"` | Source filename for the public certificate in `files/` |
| `openbao_tls_key_src` | `"openbao.key"` | Source filename for the private key in `files/` |
| `openbao_ca_cert_src` | `"root-ca.crt"` | Source filename for the Root CA certificate in `files/` |
| `openbao_audit_enabled` | `true` | Enable or disable declarative local file audit logging |
| `openbao_audit_file_path` | `"{{ openbao_log_dir }}/audit.log"` | Path to the local audit log file |

---

## Role Structure

```text
.
├── defaults/main.yml        # Default role variables
├── files/
│   ├── openbao.asc          # Official OpenBao GPG public key
│   ├── openbao-fullchain.crt# TLS server certificate (fullchain)
│   ├── openbao.key          # TLS private key (dummy key, encrypt in prod)
│   └── root-ca.crt          # Internal Root CA certificate
├── handlers/main.yml        # OS CA store update and OpenBao service restart
├── meta/main.yml            # Ansible Galaxy metadata
├── tasks/
│   ├── main.yml             # Main tasks entrypoint
│   ├── 00-install.yml       # Prerequisites, DEB822 repository, and package install
│   ├── 01-prepare.yml       # Directory creation and TLS/CA certificate deployment
│   └── 02-deploy.yml        # HCL template deployment and systemd service management
└── templates/
    └── config.hcl.j2        # OpenBao HCL configuration template
```

---

## Example Playbook

```yaml
---
- name: Deploy OpenBao
  hosts: openbao_servers
  become: true
  vars:
    openbao_api_addr: "https://openbao.internal.lan:8200"
    openbao_cluster_addr: "https://openbao.internal.lan:8201"
  roles:
    - role: install-openbao
```

If the private key is encrypted with Ansible Vault:
```bash
ansible-playbook -i inventory playbook.yml --ask-vault-pass
```

---

## Post-Deployment Steps

This role installs and starts the `openbao` systemd service, but **initialization and unsealing must be performed manually** upon first boot:

1. **Check the service status**:
   ```bash
   export BAO_ADDR="https://127.0.0.1:8200"
   bao status
   ```
2. **Initialize OpenBao**:
   ```bash
   bao operator init
   ```
   *(Store the generated unseal keys and initial root token in a secure location).*
3. **Unseal OpenBao**:
   ```bash
   bao operator unseal
   ```
4. **Next Steps (PKI Provisioning)**:
   Once OpenBao is initialized and unsealed, provision the internal PKI and secrets engines using **Terraform**.

---

## License

MIT-0

