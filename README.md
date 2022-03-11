# xait_database_pgbackrest

Installs and configures pgbackrest, PostgreSQL is assumed to already be installed.

## Usage

[Offical Ansible documentation](https://docs.ansible.com/ansible/latest/galaxy/user_guide.html#installing-multiple-roles-from-a-file).

### TL;DR

Install manually
```sh
# If roles_path is set in ansible.cfg
ansible-galaxy install git+ssh://gitlab.xait.no/ansible-roles/xait_database_pgbackrest.git
# If not, assuming you want to install to ./roles
ansible-galaxy install -p roles git+ssh://gitlab.xait.no/ansible-roles/xait_database_pgbackrest.git
```

With requirements.yml
```yml
- src: git@gitlab.xait.no:ansible-roles/xait_database_pgbackrest.git
  scm: git
```
```sh
# Same caveat about roles_path applies
ansible-galaxy install -r requirements.yml
```

## Requirements

`systemd` is required.

PostgreSQL must already be installed and running. You must also add the following to `postgresql.conf`

```ini
archive_command = 'pgbackrest --stanza=<name> archive-push %p'
archive_mode = on
max_wal_senders = 3
wal_level = replica
```

On the standby server(s), you might want to include `hot_standby = on` as well.

If using [Xait's postgres role](https://gitlab.xait.no/collab/xait_software_postgres), you can add the above config in the `postgresql_extra_conf` variable.

## Role Variables

- `pgbackrest_restore_standby` set to true to restore from a standby from the stanza with `recovery-option` set.
- `pgbackrest_timer_state` set backup timer(s) state, defaults to `started`. Set this to `stopped` for standbys.
- `pgbackrest_timer_enabled` set to `false` if running on standby.
- `pgbackrest_conf_extra` raw text that is added at the end of the config file.
- `pgbackrest_global_config` mapping of options that will be added to the `[global]` section, defaults to

```yml
pgbackrest_global_config:
  log-level-file: "debug"
  log-level-console: "info"
  start-fast: "y"
  delta: "y"
```

- `pgbackrest_stanzas` defines stanzas and (optional) scheduled backups, example below

```yml
pgbackrest_stanzas:
  - name: backup
    pg_config:
      # Local pg server
      - path: "/var/lib/pgsql/{{ postgresql_version }}/data"
      # Remote (if using backup-standby=y)
      - path: "/var/lib/pgsql/{{ postgresql_version }}/data"
        host: "10.0.50.51"
        host-user: "postgres"
      - path: "/var/lib/pgsql/{{ postgresql_version }}/data"
        host: "10.0.50.52"
        host-user: "postgres"
    schedules:
      - backup_type: full
        oncalendar: 'Weekly'
    extra:
      backup-standby: "y"
    # On standby
  - name: repl
    pg_config:
      - path: "/var/lib/pgsql/{{ postgresql_version }}/data"
    extra:
      recovery-option: "primary_conninfo=host=10.0.50.11 user=replic_user port=5432"
```

- `pgbackrest_repos` list of repositories to configure (in global section)

Example with repo1 being Azure blob storage and repo2 replication from primary server:

```yml
pgbackrest_repos:
  # Backup to Azure Blob
  - path: "/atl-main"
    retention-full: "7"
    retention-diff: "3"
    type: azure
    azure-account: atlpgstorage
    azure-container: pgbackrest
    azure-key: somelongkeyhere
  - path: "/example"
    user: "postgres"
  # replication
  - host: "10.0.50.11"
    host-user: "postgres"
```

The above examples and default config result in `/etc/pgbackrest.conf` as follows:

```ini
[global]
repo1-path=/atl-main
repo1-retention-full=7
repo1-retention-diff=3
repo1-type=azure
repo1-azure-account=atlpgstorage
repo1-azure-container=pgbackrest
repo1-azure-key=somelongkeyhere
repo2-path=/example
repo2-user=postgres
repo3-host=10.0.50.11
repo3-host-user=postgres
log-level-file=debug
log-level-console=info
start-fast=y
delta=y

[backup]
pg1-path=/var/lib/pgsql/13/data
pg2-path=/var/lib/pgsql/13/data
pg2-host=10.0.50.51
pg2-host-user=postgres
pg3-path=/var/lib/pgsql/13/data
pg3-host=10.0.50.52
pg3-host-user=postgres
backup-standby=y

[repl]
pg1-path=/var/lib/pgsql/13/data
recovery-option=primary_conninfo=host=10.0.50.11 user=replic_user port=5432
```

## Example Playbook

This example is for a single server backed up to Azure blobs

```yml
- hosts: servers
  vars:
    postgresql_version: 13
  roles:
    - role: xait_software_postgres
      vars:
        postgresql_extra_conf: |
          archive_command = 'pgbackrest --stanza=backup archive-push %p'
          archive_mode = on
          max_wal_senders = 3
          wal_level = replica
    - role: xait_database_pgbackrest
      vars:
        pgbackrest_repos:
          - path: "/atl-main"
            retention-full: "7"
            retention-diff: "3"
            type: azure
            azure-account: atlpgstorage
            azure-container: pgbackrest
            azure-key: somelongkeyhere
        pgbackrest_stanzas:
          - name: backup
            pg_config:
              - path: "/var/lib/pgsql/{{ postgresql_version }}/data"
            schedules:
              - backup_type: full
                oncalendar: 'Weekly'
```

## Author Information

Loosely based on [dudefellah's role](https://github.com/dudefellah/ansible-role-pgbackrest).

Andrei Costescu
