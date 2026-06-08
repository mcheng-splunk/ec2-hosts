# OpenTelemetry Ansible Setup for EC2 Hosts

This project deploys OpenTelemetry Collector (otelcol-contrib) across EC2 infrastructure using Ansible. It supports multiple host tiers (webservers, dbservers) across different environments (dev, test, prod).

## Architecture Overview

### High-Level Structure

```
ec2-hosts/
├── site.yml                  # Main entry point (imports all playbooks)
├── webservers.yml           # Playbook for webserver tier
├── db_machines.yml          # Playbook for database tier
├── teardown.yml             # Playbook to remove OpenTelemetry Collector
├── test-otel-trace.yml      # Test playbook with OTel tracing enabled
├── ansible.cfg              # Ansible configuration (includes OTel callback plugin)
├── callback_plugins/        # Custom Ansible callback plugins
│   └── opentelemetry_tracer.py
├── .github/
│   └── workflows/           # CI/CD GitHub Actions workflows
│       └── ansible-gitops.yml
├── inventories/             # Environment-specific configurations
│   ├── dev/                 # Development environment
│   │   ├── hosts.ini        # Host inventory with connection details
│   │   └── group_vars/      # Variables per host group
│   │       ├── all.yaml     # Environment-wide variables
│   │       ├── webservers/
│   │       │   └── vars.yaml
│   │       └── db_machines/
│   │           ├── vars.yaml
│   │           └── vault.yaml   # Encrypted secrets (Vault)
│   └── test/                # Test environment (same structure)
└── roles/
    └── otel_collector/      # Main role for OpenTelemetry deployment
        ├── tasks/main.yml       # Deployment tasks
        ├── handlers/main.yml    # Service handlers
        ├── vars/main.yml        # Default variables
        └── templates/
            └── config.yaml.j2   # OTel config template
```

### OpenTelemetry Instrumentation

The project includes an Ansible callback plugin (`callback_plugins/opentelemetry_tracer.py`) that automatically captures OpenTelemetry traces for all playbook executions:

**What gets traced:**
- Playbook start/end (root span)
- Each play start/end
- Each task execution per host (ok, failed, skipped, unreachable)
- Playbook statistics (success/failure counts per host)

**To enable tracing:**
1. Install dependencies: `pip install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp-proto-grpc`
2. Set `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable (e.g., `http://localhost:4317`)
3. Run any playbook normally - the callback plugin is enabled in `ansible.cfg`

See [oneuptime.com](https://oneuptime.com/blog/post/2026-02-06-instrument-ansible-playbook-opentelemetry/) for more details on the callback plugin implementation.

### Testing with OpenTelemetry Tracing

The `test-otel-localhost.yml` playbook tests the OTel callback plugin by running tasks on localhost:

```bash
# Run with debug mode (spans printed to console)
unset OTEL_EXPORTER_OTLP_ENDPOINT
ansible-playbook -i inventories/dev/ test-otel-localhost.yml

# Run with OTel endpoint (spans exported to collector)
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317"
ansible-playbook -i inventories/dev/ test-otel-localhost.yml

# Target specific hosts using --limit
ansible-playbook -i inventories/dev/ test-otel-localhost.yml --limit db_machines
ansible-playbook -i inventories/dev/ test-otel-localhost.yml --limit "18.143.144.122"
```

To test against real EC2 hosts (requires vault):
```bash
ansible-playbook -i inventories/dev/ test-otel-localhost.yml --limit db_machines --ask-vault-pass
```

### Host Tiers

| Tier | Description | Example Hosts |
|------|-------------|---------------|
| `webservers` | Web application servers | dev-web-01, dev-web-02, test-web-01, etc. |
| `db_machines` | Database servers | dev-db-01, test-db-01, test-db-02, etc. |

### Supported Databases

The OpenTelemetry configuration is dynamically generated based on `db_type`:

| Database Type | Port | Receiver | Config |
|--------------|------|----------|--------|
| PostgreSQL | 5432 | `postgresql` | TLS insecure mode enabled |
| MySQL | 3306 | `mysql` | Standard connection |
| MSSQL | 1433 | `sqlserver` | SQL Server connection string |
| Oracle | 1521 | `oracledb` | XEPDB1 container |

### OpenTelemetry Pipeline

Each host runs an OpenTelemetry Collector with:

**Receivers:**
- `otlp` (gRPC/HTTP) - For incoming OTLP traces/metrics
- `hostmetrics` - System metrics (CPU, memory, load)
- Database-specific receiver (based on `db_type`)

**Processors:**
- `batch` - Batches samples for efficient export
- `memory_limiter` - Prevents memory exhaustion (512 MiB limit)

**Exporters:**
- `debug` - Currently outputs to console (detailed verbosity)

## Folder Structure Details

### Inventories (`inventories/`)

Each environment (dev, test, prod) has its own inventory directory with:

**hosts.ini** - Host definitions with connection parameters:
```ini
[webservers]
webserver-name ansible_host=IP ansible_user=user ansible_ssh_private_key_file=key-path

[db_machines]
dbserver-name ansible_host=IP ansible_user=user db_type=postgres otel_metrics_port=port ansible_ssh_private_key_file=key-path
```

**group_vars/** - Variables organized by host group:
- `all.yaml` - Environment-wide variables (e.g., `env_stage: dev`)
- `webservers/vars.yaml` - Web tier overrides (package type, monitoring target)
- `db_machines/vars.yaml` - DB tier overrides
- `db_machines/vault.yaml` - **Encrypted** secrets (database credentials)

### Roles (`roles/`)

The `otel_collector` role contains:

**tasks/main.yml** - Deployment steps:
1. Install Python prerequisites (python3-apt/dnf)
2. Create configuration directory
3. Download OpenTelemetry package
4. Install package (apt/dnf based on OS)
5. Deploy config template
6. Start/enable otelcol-contrib service

**templates/config.yaml.j2** - Jinja2 template generating environment-specific OTel config with conditional database receivers

## Deployment Scenarios

### Prerequisites

1. **SSH Access:**
   - Private key file path configured in `hosts.ini`
   - SSH key has access to all target hosts
   - User (typically `ec2-user` or `ubuntu`) has sudo privileges

2. **Vault Setup (for database secrets):**
   - Create `.vault_pass.txt` with the Vault decryption password
   - Ensure `vault_db_password` is set in `group_vars/db_machines/vault.yaml`
   - **Note:** When running from CLI, use `--ask-vault-pass` to enter the password interactively. When running from CI/CD (GitHub Actions), use `--vault-password-file` with the secret stored in repository secrets.

3. **OpenTelemetry Package:**
   - Supported: RPM (RHEL/Amazon Linux) or DEB (Debian/Ubuntu)
   - Set `otel_pkg_type` in `group_vars/*/vars.yaml`

### Deploy to All Hosts in Dev Environment

```bash
# Using site.yml (deploys both tiers)
# If running from CLI: use --ask-vault-pass
# If running from CI/CD: use --vault-password-file with the secret stored there
ansible-playbook -i inventories/dev/ site.yml --ask-vault-pass

# Or run individually
ansible-playbook -i inventories/dev/ webservers.yml --ask-vault-pass
ansible-playbook -i inventories/dev/ db_machines.yml --ask-vault-pass
```

### Deploy Only to Webservers

```bash
# For dev environment
ansible-playbook -i inventories/dev/ webservers.yml --ask-vault-pass

# For test environment
ansible-playbook -i inventories/test/ webservers.yml --ask-vault-pass
```

### Deploy Only to DB Servers

```bash
# Deploy to all dbservers in dev (all DB types supported)
ansible-playbook -i inventories/dev/ db_machines.yml --ask-vault-pass

# Deploy to specific DB server (using --limit)
ansible-playbook -i inventories/test/ db_machines.yml --limit test-db-01 --ask-vault-pass
```

### Deploy to Different Environments

```bash
# Test environment
ansible-playbook -i inventories/test/ site.yml --ask-vault-pass

# Production environment (requires prod/ inventory directory)
ansible-playbook -i inventories/prod/ site.yml --ask-vault-pass
```

### Dry Run (Check Mode)

```bash
# Preview changes without applying
ansible-playbook -i inventories/dev/ site.yml --check --ask-vault-pass
```

### Destroy/Teardown OpenTelemetry Collector

```bash
# Remove OTel collector from all hosts in dev
ansible-playbook -i inventories/dev/ teardown.yml --ask-vault-pass

# Remove from specific tier
ansible-playbook -i inventories/dev/ teardown.yml --limit webservers --ask-vault-pass
ansible-playbook -i inventories/dev/ teardown.yml --limit db_machines --ask-vault-pass

# Remove from specific host (by name)
ansible-playbook -i inventories/dev/ teardown.yml --limit test-db-01 --ask-vault-pass

# Remove from specific host (by IP)
ansible-playbook -i inventories/dev/ teardown.yml --limit "18.143.144.122" --ask-vault-pass

# Check mode - see what would be removed
ansible-playbook -i inventories/dev/ teardown.yml --check --ask-vault-pass
```

The teardown playbook will:
1. Stop and disable the `otelcol-contrib` service
2. Remove the OpenTelemetry package (apt/dnf)
3. Delete the configuration directory (`/etc/otelcol-contrib`)
4. Clean up temporary download files in `/tmp`

### Deploy with Verbose Output

```bash
# Verbose logging
ansible-playbook -i inventories/dev/ site.yml -v --ask-vault-pass

# Very verbose (debug)
ansible-playbook -i inventories/dev/ site.yml -vvvv --ask-vault-pass
```

## Variable Reference

### Default Variables (role/vars/main.yml)

| Variable | Description | Default |
|----------|-------------|---------|
| `otel_version` | OpenTelemetry Collector version | `0.152.0` |
| `otel_arch` | Architecture | `amd64` |
| `otel_config_dir` | Configuration directory | `/etc/otelcol-contrib` |
| `otel_config_file` | Config filename | `config.yaml` |
| `otel_download_url` | Download URL (auto-generated) | GitHub Releases |

### Environment Variables (group_vars/all.yaml)

| Variable | Description | Example |
|----------|-------------|---------|
| `env_stage` | Environment name | `dev`, `test`, `prod` |

### Host Variables (group_vars/*/vars.yaml)

| Variable | Description | Example |
|----------|-------------|---------|
| `otel_pkg_type` | Package type (rpm/deb) | `rpm` for Amazon Linux |
| `otel_monitoring_target` | Application type identifier | `web-app` |
| `otel_log_path` | Log file path to monitor | `/var/log/nginx/access.log` |
| `otel_metrics_port` | Application metrics port | `8080` |

### Host Inventory Variables (hosts.ini)

| Variable | Description | Required |
|----------|-------------|----------|
| `ansible_host` | IP address or hostname | Yes |
| `ansible_user` | SSH username | Yes |
| `ansible_ssh_private_key_file` | Path to private key | Yes |
| `db_type` | Database type | Yes (for db_machines) |
| `otel_metrics_port` | Database metrics port | Yes (for db_machines) |

### Vault Variables (group_vars/db_machines/vault.yaml)

| Variable | Description | Encrypted |
|----------|-------------|-----------|
| `vault_db_password` | Database monitor password | Yes |

## CI/CD Deployment (GitHub Actions)

The `.github/workflows/ansible-gitops.yml` workflow enables:

1. **Automatic deployment on git push** to main branch when files change
2. **Manual deployment** via workflow_dispatch with environment/tier selection

### Triggering Manual Deployment

1. Go to GitHub > Actions > "Ansible GitOps"
2. Click "Run workflow" dropdown
3. Select:
   - **Target Environment**: dev, test, or prod
   - **Target Tier**: db_machines, webservers, or ALL

### Required Secrets

Add these to your GitHub repository settings:

| Secret | Description |
|--------|-------------|
| `SSH_PRIVATE_KEY` | Private SSH key for host access |
| `ANSIBLE_VAULT_PASSWORD` | Password for decrypting vault.yaml |

## Adding a New Environment

1. Create inventory directory: `mkdir -p inventories/prod/group_vars/{webservers,db_machines}`
2. Copy and modify templates:
   - `inventories/dev/hosts.ini` → `inventories/prod/hosts.ini` (update IPs)
   - `inventories/dev/group_vars/all.yaml` → `inventories/prod/group_vars/all.yaml` (change `env_stage`)
3. Create vault file with secrets: `ansible-vault create inventories/prod/group_vars/db_machines/vault.yaml`
4. Test deployment with `--check` mode first

## Adding a New Database Type

1. Update `roles/otel_collector/templates/config.yaml.j2`:
   - Add new receiver block in the `receivers:` section
   - Add receiver name to the pipeline `receivers:` list
2. Update host inventory with `db_type=newdbtype`
3. Add database credentials to vault file

## Troubleshooting

### Common Issues

1. **SSH Connection Failed:**
   ```bash
   # Verify SSH key and connectivity
   ssh -i ~/.ssh/aws-id-rsa.pem ec2-user@HOST_IP
   ```

2. **Vault Decryption Error:**
   ```bash
   # Verify vault password file exists and is readable
   cat .vault_pass.txt
   # Verify vault file is actually encrypted
   head inventories/dev/group_vars/db_machines/vault.yaml
   ```

3. **Package Installation Failed:**
   - Verify `otel_pkg_type` matches host OS (rpm vs deb)
   - Check network connectivity to GitHub Releases

4. **Collector Not Starting:**
   ```bash
   # SSH to host and check logs
   ssh -i ~/.ssh/aws-id-rsa.pem ec2-user@HOST_IP
   sudo journalctl -u otelcol-contrib -f
   ```

### Verifying Deployment

```bash
# Check if OTel collector is running
ansible -i inventories/dev/ db_machines -m service -a "name=otelcol-contrib state=started" --ask-vault-pass

# View logs from all hosts
ansible -i inventories/dev/ all -m shell -a "sudo journalctl -u otelcol-contrib -n 20" --ask-vault-pass
```

### Verifying Teardown

```bash
# Check if OTel collector is stopped
ansible -i inventories/dev/ db_machines -m service -a "name=otelcol-contrib state=stopped" --ask-vault-pass

# Verify package is removed
ansible -i inventories/dev/ all -m shell -a "dpkg -l | grep otelcol-contrib || rpm -qa | grep otelcol-contrib" --ask-vault-pass
```
