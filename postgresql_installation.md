# PostgreSQL Database
- [PostgreSQL Database](#postgresql-database)
  - [Introduction](#introduction)
  - [Requirements](#requirements)
  - [Installation](#installation)
  - [Complex Installation Playbook](#complex-installation-playbook)
  - [Roles Description](#roles-description)
    - [`galaxyproject.postgresql`](#galaxyprojectpostgresql)
    - [`galaxyproject.postgresql_objects`](#galaxyprojectpostgresql_objects)
  - [Verify Succefull Installation and Configuration](#verify-succefull-installation-and-configuration)
  - [References](#references)
  - [Author Information](#author-information)


## Introduction

PostgreSQL is a powerful *open-source* relational database management system (RDBMS). It provides an efficient and reliable solution for storing, managing, and retrieving structured data. PostgreSQL supports ACID (Atomicity, Consistency, Isolation, Durability) properties, ensuring data integrity and reliability.  

This documentation provides guidelines for installing and managing PostgreSQL databases using Ansible roles. The two main roles are covered: the `galaxyproject.postgresql` role focuses on server installation and management, while `galaxyproject.postgresql_objects` specializes in managing PostgreSQL users, groups, databases, and privileges.

## Requirements

Ensure that you have the following:

- Ansible version 2.9 or later.
- Infrastructure: A master database VM running either Debian or RedHat.


## Installation

It is recommended to install PostgreSQL version 12 or higher as some important features may not be supported in earlier versions.

## Complex Installation Playbook 

[database.yml](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/database.yml) playbook allows for basic PostgerSQL server installation.  
Check variables that are located in:  
- `group_vars/database.yml`

| Variable                     | Description                                                   |
| ---------------------------- | ------------------------------------------------------------- |
| postgresql_version           | PostgreSQL version to be installed.                           |
| postgresql_objects_users     | Name and password of user(s) (Galaxy) in a list format.       |
| postgresql_objects_databases | Name and owner of database(s) (Galaxy) in a list format.      |
| postgresql_conf              | PostgreSQL configuration options in a list format.            |
| postgresql_pg_hba_conf       | Connection and authentication options for PostgreSQL clients. |

- `secret_group_vars/db-main.yml` encrypted file for sensitive variables.  

| Variable             | Description                                                   |
| -------------------- | ------------------------------------------------------------- |
| _galaxy_db_host      | **Sensitive value**. IP address of Master PostgreSQL sever.   |
| _galaxy_db_name      | **Sensitive value**. Name of Galaxy database.                 |
| _galaxy_db_user      | **Sensitive value**. Name of Galaxy  user.                    |
| _galaxy_db_port      | **Sensitive value**. PostgreSQL connection port.              |
| _galaxy_db_pass      | **Sensitive value**. Password for Galaxy user.                |
| galaxy_db_connection | **Sensitive value**. PostgreSQL connection string for Galaxy. |

If you don't want to set up database cluster (i.e. install only database server, without replica and backup servers), run:
```bash 
ansible-playbook --private-key <path_to_priv_key> -i hosts database.yml --extra-vars "postgres_backup=false,postgres_replication=false"
```  
Otherwise, refer to [Database Cluster](https://github.com/usegalaxy-it/documentation/blob/main/database_cluster.md) documentation.  

## Roles Description

### `galaxyproject.postgresql` 

The PostgreSQL role is designed to install and manage PostgreSQL servers on both Debian and RedHat based systems. It handles:
- PostgreSQL configurations;
- backup scripts for Continuous Archiving and Point-in-Time Recovery.

The main features of the PostgreSQL role include:
1. **Installation**: The role installs PostgreSQL using the distribution's packages (APT or YUM) or the PostgreSQL Global Development Group (PGDG) packages.

2. **Configuration**: Customize PostgreSQL version, `postgresql.conf` options (e.g. indicate `listen_addresses` option), and manage `pg_hba.conf` (define the client authentication rules for connecting to a PostgreSQL database server)

3. **Backup Scripts**: Enable Point-in-Time Recovery (PITR) backups with a specified backup directory and schedule.  
To enable PITR set the value of `postgresql_backup_dir` to the desired directory.  

**NB!** Backup script doesn't work when you indicate a remote location for backup. To handle this situation additional database dump to remote machine is done by `usegalaxy-it.postgres_backup`. More details about backup options for usegalaxy.it can be found in [Postgresql Backup](https://github.com/usegalaxy-it/documentation/blob/main/postgresql_backup.md) and [Postgresql Replication](https://github.com/usegalaxy-it/documentation/blob/main/postgresql_replication.md) documentation.

Sample playbook for PGSQL v.15 installation with local PITR daily backup:
```YAML
- name: PostgreSQL Installation
  hosts: <pgsql_host>
  become: true
  roles:
    - role: galaxyproject.postgresql
      vars:
          postgresql_version: 15
          postgresql_backup_dir: /archive
          postgresql_backup_rsync_backup_opts: '-r -p -t -g'
          postgresql_conf:
            - listen_addresses: "'<pgsql_host_ip>'" # preserve these quotes
          postgresql_pg_hba_conf:
            - host     all     all     <pgsql_host_ip>/<subnet_mask>     md5
```

### `galaxyproject.postgresql_objects`

The PostgreSQL Objects role is designed to manage PostgreSQL users, groups, databases, and privileges.

Dependencies for the PostgreSQL Objects role include the Python `python3-psycopg2` module, which can be installed via a pre-task in your playbook. Alternatively, if you use the `galaxyproject.postgresql` role, psycopg2 will be installed automatically.

The main features of the PostgreSQL Objects role include:

1. **User Management**: You can create or drop PostgreSQL users using the `postgresql_objects_users` variable.

2. **Group Management**: PostgreSQL groups can be created or dropped using the `postgresql_objects_groups` variable. Additionally, you can specify a list of users to add to the group.

3. **Database Management**: You can create or drop databases using the `postgresql_objects_databases` variable.

4. **Privilege Management**: Privileges can be granted or revoked using the `postgresql_objects_privileges` variable.   

**NB!** Variables below are sensitive, set them in a vault encrypted file!   

Sample playbook for PGSQL objects management:
```YAML
- name: PostgreSQL Management
  hosts: <pgsql_host>
  become: true
  pre_tasks:
    - name: Install acl and psycopg2 packages
      package:
        name: ["acl", "python3-psycopg2"]
      become: true
  roles:
    - role: galaxyproject.postgresql_objects
      become: true
      become_user: postgres
      vars:
        postgresql_objects_users:
          - name: <galaxy_db_user>              # sensitive value
            password: <galaxy_db_password>      # sensitive value
        postgresql_objects_databases:
          - name: <galaxy_db_name>              # sensitive value
            owner: <galaxy_db_owner>            # sensitive value
```

Remeber to configure Galaxy to connect to PostgreSQL using varibles defined above
```bash
galaxy_db_connection: "postgresql://<galaxy_db_user>:<galaxy_db_password>@<pgsql_host>/<galaxy_db_name>"
```
## Verify Succefull Installation and Configuration

On the PGSQL VM check for installed PGSQL.  
All configuration files by default are stored in `/var/lib/pgsql/<postgresql_version>/data` directory.  
You can check that the configuration files are created correctly and have all entries that have been indcated. Check for:
- `/var/lib/pgsql/<postgresql_version>/data/conf.d/25ansible_postgresql.conf`
- `/var/lib/pgsql/<postgresql_version>/data/pg_hba.conf`

Find the location of current log file by:
```bash
$ sudo su postgres
$ cd /var/lib/pgsql/<postgresql_version>/data
$ cat current_logfiles 
stderr log/postgresql-Fri.log
```

Check for existing users, their roles and databases:
```
$ sudo su postgres
$ psql

postgres=# \dg
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 galaxy    |                                                            | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}

postgres=# \l+
                                                                List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   |  Size   | Tablespace |                Description                 
-----------+----------+----------+---------+---------+-----------------------+---------+------------+--------------------------------------------
 galaxy    | galaxy   | UTF8     | C.UTF-8 | C.UTF-8 |                       | 35 MB   | pg_default | 
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |                       | 7993 kB | pg_default | default administrative connection database
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +| 7849 kB | pg_default | unmodifiable empty database
           |          |          |         |         | postgres=CTc/postgres |         |            | 
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +| 7849 kB | pg_default | default template for new databases
           |          |          |         |         | postgres=CTc/postgres |         |            | 
(4 rows)

```

## References

[PostgreSQL Documentation](https://www.postgresql.org/docs/)
[galaxyproject.postgresql_objects](https://github.com/galaxyproject/ansible-postgresql-objects) Ansible playbook
[galaxyproject.postgresql](https://github.com/galaxyproject/ansible-postgresql) Ansible playbook  
Optional features to configure:  
- [usegalaxy-it.postgres-replication](https://github.com/usegalaxy-it/usegalaxy-it.postgres-replication) Ansible playbook  
- [usegalaxy-it.postgres-backup](https://github.com/usegalaxy-it/usegalaxy-it.postgres-backup) Ansible playbook  
See also UseGalaxy.it-only features:  
- [PostgreSQL Cluster](https://github.com/usegalaxy-it/documentation/blob/main/database_cluster.md) documentation  
- [PostgreSQL Backup](https://github.com/usegalaxy-it/documentation/blob/main/postgresql_backup.md) documentation  
- [PostgreSQL Replication](https://github.com/usegalaxy-it/documentation/blob/main/postgresql_replication.md) documentation  

## Author Information

[Polina Khmelevskaia](https://github.com/po-khmel)