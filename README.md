# Private Documentation Usegalaxy Deployment
## Infrastructure creation

## Ansible
### Ansible installation
Ansible by installing [ansible-core](https://docs.ansible.com/core.html) + needed [collections](https://docs.ansible.com/ansible/latest/user_guide/collections_using.html) in order to have a more stable system
#### Install ansible core on ubuntu 20.04 

``` bash
sudo apt update
sudo apt install python3-venv
python3 -m venv master_env/
. master_env/bin/activate
pip install wheel ansible-core==2.11.3
```
##### download roles and collections needed

``` bash
git clone https://github.com/pmandreoli/infrastructure-playbook.git
git checkout dev_it
ansible-galaxy install -r requirements.yaml # install required collections and roles
```
### Infrastructure-playbook
#### configuration
- `hosts` inventory file containing each infrastructure element created using Terraform 
- `yml files` playbook to configure the services
- `group_vars` directory containing for each service a .yml file with variables 
- `secret_group_vars` containing for the services that need secrets, the secret itself are encrypted using Ansible vault  
- `templates` containing the configuration files to be created by each playbook/role
- `roles` Ansible role dir
- `collection` Ansible galaxy connection needed

#### services to configure

1. Postgresql
2. Rabbitmq
3. Galaxy
4. Nginx

##### Postgresql
``` bash 
ansible-playbook --private-key ~/.ssh/laniakea-robot.key -i hosts database.yml
```
control if postgresql is active and running on the database machine 
``` bash
# nmap -v -p 5432 127.0.0.1

Starting Nmap 7.60 ( https://nmap.org ) at 2022-10-12 14:06 UTC
Initiating SYN Stealth Scan at 14:06
Scanning localhost (127.0.0.1) [1 port]
Discovered open port 5432/tcp on 127.0.0.1
Completed SYN Stealth Scan at 14:06, 0.23s elapsed (1 total ports)
Nmap scan report for localhost (127.0.0.1)
Host is up (0.00024s latency).

PORT     STATE SERVICE
5432/tcp open  postgresql

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.33 seconds
           Raw packets sent: 1 (44B) | Rcvd: 2 (88B)
```

control postgres configuration files `/etc/postgresql/12/main/pg_hba.conf` 

``` bash 
## This file is maintained by Ansible - CHANGES WILL BE OVERWRITTEN
##

# DO NOT DISABLE!
# If you change this first entry you will need to make sure that the
# database superuser can access the database using some other method.
# Noninteractive access to all databases is required during automatic
# maintenance (custom daily cronjobs, replication, and similar tasks).
#
# Database administrative login by Unix domain socket
local   all             postgres                                peer

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5

# Entries configured in postgresql_pg_hba_conf follow

host    all             all        172.30.16.130/24              md5
```
- 172.30.16.130/24  is the galaxy private network
/etc/postgresql/12/main/conf.d/25ansible_postgresql.conf
contains:
``` bash 
listen_addresses = '<postgresql ip>'
```
secret\_group\_vars needed:
``` yml
_galaxy_db_pass: "<passwd>"
_galaxy_db_host: "<database-ip>"
_galaxy_db_name: <db_name>
_galaxy_db_user: <db_user>
_galaxy_db_port: <db_port>


galaxy_db_connection: "postgresql://{{ _galaxy_db_user }}:{{ _galaxy_db_pass }}@{{ _galaxy_db_host }}/{{ _galaxy_db_name }}"
postgres_pass: "{{ _galaxy_db_pass }}"
```
##### RabbitMQ
```bash
ansible-playbook --private-key ~/.ssh/laniakea-robot.key -i hosts rabbitmq.yml
```
check if created certificate are visible and mounted on the rabbitmq container to debug ssl connection check the [throubleshooting doc ](https://www.rabbitmq.com/troubleshooting-ssl.html)
rabbit will be installed and run on a the official docker container on the rabbitmq host
check rabbit log:
``` bash 
docker logs rabbit_hole
```
check if the users were correctly created, in the docker container:

``` bash
sudo rabbitmqctl authenticate_user <username> <password>
```

##### Galaxy
```bash
ansible-playbook --private-key <key> -i hosts sn06.yml --extra-vars "__galaxy_dir_perms='0755'"
```
secret_group_vars the playbook includes both the db_main.yml and the all.yml:
`all.yml` 
``` ansible
---
galaxysession_cookie: test1
vault_sign_pass_secret: <random>

# rabbitmq
rabbitmq_password_galaxy: <rnadom>

# galaxy
galaxy_db_connection_ro: "<random>"

# proftpd
proftpd__galaxy_db_connection: "test6"

smtp_username: "<smtp server username>"
smtp_password: "<random>"


admin_users:
 - "<email>"

id_secret: "<random>"
certbot_admin_email: "<email>"
gapars_nginx_config: ""
```
> :warning: **The installation do not set the nginx upload module, the nginx configuration has been modified**

--- 
**GALAXY troubleshooting**
using `journalctl`
check the galaxy unit 
``` bash 
journalctl -fu galaxy-zergling*
```
---
