# sol1-kea

Ansible role to deploy and configure ISC Kea DHCP4 server from Cloudsmith's repository. This role handles installation, configuration, and management of Kea with support for multiple database backends (PostgreSQL, MySQL, or in-memory file storage).

## Features

- Automated installation of ISC Kea 3.0 from Cloudsmith repository
- Support for multiple lease and hosts database backends
- Database-agnostic configuration (PostgreSQL, MySQL, or memfile/JSON)
- HA (High Availability) mode support with hot-standby
- AppArmor profile support for Debian/Ubuntu
- Control Agent configuration for remote management

## Requirements

- Ansible 2.9+
- Supported OS: Ubuntu 24.04, Debian 13
- Python 3.6+ on target systems

## Role Variables

### Core Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `kea_version` | `'3.0'` | ISC Kea version to deploy |
| `kea_user` | `kea` | System user for Kea service |
| `kea_group` | `kea` | System group for Kea service |

### Directory Paths

| Variable | Default | Description |
|----------|---------|-------------|
| `kea_dir_config` | `/etc/kea` | Kea configuration directory |
| `kea_dir_leases` | `/etc/kea/leases` | Kea leases directory |
| `kea_dir_log` | `/var/log/kea` | Kea logs directory |

### Lease Database Backend

| Variable | Default | Description |
|----------|---------|-------------|
| `kea_dhcp4_lease_database_type` | `memfile` | Backend type: `postgresql`, `mysql`, or `memfile` |
| `kea_dhcp4_lease_file` | `/etc/kea/leases/dhcp4.leases` | Path to memfile lease database |
| `kea_dhcp4_lease_flush_interval` | `30` | Interval in seconds for flushing memfile leases to disk |

**Note:** When selecting `postgresql` or `mysql` default config is built. You can override the default config by setting `kea_dhcp4_lease_database`

### Hosts (Reservations) Database Backend

| Variable | Default | Description |
|----------|---------|-------------|
| `kea_dhcp4_hosts_database_type` | `json` | Backend type: `postgresql`, `mysql`, or `json` |
| `kea_dhcp4_hosts_json_file` | `/etc/kea/hosts.json` | Path to JSON hosts database (used when type is `json`) |

**Note:** When selecting `postgresql` or `mysql` default config is built. You can override the default config by setting `kea_dhcp4_host_database`

**Note:** Hosts database stores DHCP reservations, allowing updates without restarting Kea when using file/database backends.

### Database Credentials

Used when `kea_dhcp4_lease_database_type` or `kea_dhcp4_hosts_database_type` is not `memfile` or `json`:

| Variable | Default | Description |
|----------|---------|-------------|
| `kea_db_name` | `""` | Database name (auto-set to `kea_db` if empty) |
| `kea_db_host` | `""` | Database host (auto-set to `127.0.0.1` if empty) |
| `kea_db_username` | `""` | Database username (auto-set to `kea` if empty) |
| `kea_db_password` | `""` | Database password (auto-set to `AStrongPassword` if empty) |

### HA Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `kea_dhcp4_ha_enabled` | `false` | Enable High Availability mode |
| `kea_dhcp4_ha_mode` | `hot-standby` | HA mode type |
| `kea_dhcp4_ha_primary` | `""` | Primary HA node name |
| `kea_dhcp4_ha_primary_ip` | `""` | Primary HA node IP address |
| `kea_dhcp4_ha_standby` | `""` | Standby HA node name |
| `kea_dhcp4_ha_standby_ip` | `""` | Standby HA node IP address |

### Network Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `kea_dhcp4_interfaces` | `['eth0']` | List of interfaces for DHCP4 listener |
| `kea_dhcp4_subnet` | See below | DHCP4 subnet declarations with pools and options |
| `kea_ctrl_agent_listen_port` | `8000` | Control Agent HTTP port |
| `kea_ctrl_agent_listen_host` | Dynamic IP or `0.0.0.0` | Control Agent bind address |

Default subnet example:
```yaml
kea_dhcp4_subnet:
  - subnet: "192.168.2.0/24"
    id: 1
    pools:
      - pool: "192.168.2.100 - 192.168.2.200"
    option-data:
      - name: "routers"
        data: "192.168.2.1"
      - name: "domain-name-servers"
        data: "192.168.2.1"
```

### Hook Libraries & Logging

| Variable | Default | Description |
|----------|---------|-------------|
| `kea_dhcp4_hooks_libraries` | See below | Hook libraries to load (auto-set based on DB type) |
| `kea_dhcp4_loggers` | DHCP4 & leases logs | Logging configuration |
| `kea_packages` | `[]` | Additional Kea packages to install |
| `prereq_packages` | `[]` | Additional prerequisite packages |

Default hooks:
- `libdhcp_host_cmds.so` - Host commands hook
- `libdhcp_lease_cmds.so` - Lease commands hook
- `libdhcp_mysql.so` or `libdhcp_pgsql.so` - Database-specific hook (when using SQL backend)

## Load Variables Task

The `load_variables` task performs the following during playbook execution:

1. **OS-specific variable loading** - Loads variables based on detected OS distribution
2. **Database backend validation** - Ensures valid database types are selected:
   - Lease database must be: `postgresql`, `mysql`, or `memfile`
   - Hosts database must be: `postgresql`, `mysql`, or `json`
   - **Mixing constraint:** Cannot use `postgresql` for one and `mysql` for the other
3. **Database driver selection** - Loads appropriate database variables
   - If lease type is not `memfile`, loads `mysql.yml` or `postgresql.yml`
   - If lease type is `memfile` and hosts type is not `json`, loads hosts database vars
4. **Database configuration** - Sets up database connection details with defaults

## Database Backend Variables

When using PostgreSQL (`postgresql.yml`) or MySQL (`mysql.yml`), the following variables are auto-set:

- `_database_packages` - Database client/server packages
- `_kea_dhcp4_lease_database` - Lease database connection config
- `_kea_dhcp4_hosts_database` - Hosts database connection config
- `kea_dhcp4_hooks_libraries` - Updated to include database driver hook

### PostgreSQL Variables (postgresql.yml)
```yaml
_database_packages:
  - isc-kea-pgsql
  - postgresql
  - python3-psycopg2

_kea_dhcp4_lease_database:
  type: "postgresql"
  name: "{{ _kea_db_name }}"
  host: "{{ _kea_db_host }}"
  port: 5432
  # ... additional fields
```

### MySQL Variables (mysql.yml)
```yaml
_database_packages:
  - isc-kea-mysql
  - mariadb-client
  - mariadb-server
  - python3-pymysql

_kea_dhcp4_lease_database:
  type: "mysql"
  name: "{{ _kea_db_name }}"
  host: "{{ _kea_db_host }}"
  port: 3306
  # ... additional fields
```

## Usage Examples

### Default (In-Memory File Storage)

```yaml
- hosts: kea_servers
  roles:
    - sol1-kea
```

### PostgreSQL Backend

```yaml
- hosts: kea_servers
  vars:
    kea_dhcp4_lease_database_type: postgresql
    kea_dhcp4_hosts_database_type: postgresql
    kea_db_name: kea_production
    kea_db_host: db.example.com
    kea_db_username: kea_app
    kea_db_password: SecurePassword123
  roles:
    - sol1-kea
```

### MySQL Backend

```yaml
- hosts: kea_servers
  vars:
    kea_dhcp4_lease_database_type: mysql
    kea_dhcp4_hosts_database_type: mysql
    kea_db_name: kea_production
    kea_db_host: db.example.com
    kea_db_username: kea_app
    kea_db_password: SecurePassword123
  roles:
    - sol1-kea
```

### Memfile Leases with PostgreSQL Hosts

```yaml
- hosts: kea_servers
  vars:
    kea_dhcp4_lease_database_type: memfile
    kea_dhcp4_hosts_database_type: postgresql
    kea_db_name: kea_reservations
    kea_db_host: db.example.com
  roles:
    - sol1-kea
```

## Validation

The role performs the following validations during the `load_variables` task:

- **Database type validation:** Ensures only valid backend types are specified
- **Mixed database constraint:** Prevents using PostgreSQL and MySQL together for lease/hosts databases
- This prevents configuration issues that could cause Kea to fail startup

## Files Modified by Role

- Configuration: `/etc/kea/kea-dhcp4.conf`
- Control Agent: `/etc/kea/kea-ctrl-agent.conf`
- Leases (memfile): `/etc/kea/leases/dhcp4.leases`
- Hosts (JSON): `/etc/kea/hosts.json`
- AppArmor profiles (if applicable): `/etc/apparmor.d/usr.sbin.kea-*`

## License

See LICENSE file


