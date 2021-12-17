# xait_software_pgbackrest

Installs and configures pgbackrest, PostgreSQL is assumed to already be installed.

## Usage

[Offical Ansible documentation](https://docs.ansible.com/ansible/latest/galaxy/user_guide.html#installing-multiple-roles-from-a-file).

### TL;DR

Install manually
```sh
# If roles_path is set in ansible.cfg
ansible-galaxy install git+ssh://gitlab.xait.no/ansible-roles/xait_software_pgbackrest.git
# If not, assuming you want to install to ./roles
ansible-galaxy install -p roles git+ssh://gitlab.xait.no/ansible-roles/xait_software_pgbackrest.git
```

With requirements.yml
```yml
- src: git@gitlab.xait.no:ansible-roles/xait_software_pgbackrest.git
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
archive_command = 'pgbackrest --stanza=main archive-push %p'
archive_mode = on
max_wal_senders = 3
wal_level = replica
```

If using [Xait's postgres role](https://gitlab.xait.no/collab/xait_software_postgres), you can add the above config in the `postgresql_extra_conf` variable.

## Role Variables

- `pgbackrest_stanzas` defines stanzas and (optional) scheduled backups

```yml
pgbackrest_stanzas:
  - stanza: main
    pg_path: "/var/lib/pgsql/{{ postgresql_version }}/data"
    # schedules is optional
    schedules:
      - backup_type: full
        oncalendar: 'Weekly'
      - backup_type: diff
        oncalendar: 'Daily'
```

## Example Playbook

```yml
- hosts: servers
  vars:
    postgresql_version: 13
  roles:
    - role: xait_software_postgres
      vars:
        postgresql_extra_conf: |
          archive_command = 'pgbackrest --stanza=main archive-push %p'
          archive_mode = on
          max_wal_senders = 3
          wal_level = replica
    - role: xait_software_pgbackrest
      vars:
        pgbackrest_stanzas:
          - stanza: main
            pg_path: "/var/lib/pgsql/{{ postgresql_version }}/data"
```

## Author Information

Loosely based on [dudefellah's role](https://github.com/dudefellah/ansible-role-pgbackrest).

Andrei Costescu
