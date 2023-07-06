---
title: Infrastructure deployment
---

## Infrastructure creation using Terraform
- [install terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli#install-terraform)
- clone the [infrastructure repository](https://github.com/pmandreoli/infrastructure)
- source the openstack v2 auth file
- create the infrastructure using `terraform init`, `terraform plan -o galaxy`, `terraform apply galaxy`
- the command `terraform output` gives in output all the ip of the newly created virtual machines, that can be used to populate the Ansible inventory file in the next configuration step 

## Configuration using Ansible

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
##### Download roles and collections needed

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

1. NFS
2. Postgresql
3. Rabbitmq
4. Galaxy
5. Nginx

##### NFS
to run proper job on the htcondor cluster the galaxy machine and condor workers has to share 2 main dir on NFS: 
- /data/share
- /opt/galaxy 

**df -h result on the galaxy VM** 
``` bash
Filesystem                 Size  Used Avail Use% Mounted on
devtmpfs                   3.9G     0  3.9G   0% /dev
tmpfs                      3.9G     0  3.9G   0% /dev/shm
tmpfs                      3.9G  656K  3.9G   1% /run
tmpfs                      3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/vda2                   20G   15G  5.8G  72% /
172.30.16.133:/data/share  200G  2.7G  198G   2% /data/share
tmpfs                      783M     0  783M   0% /run/user/1000
172.30.16.133:/opt/galaxy   20G   13G  7.4G  64% /opt/galaxy
```
after the creation of the NFS you can mount it by using the `mount.yml` playbook

the configuration of the nfs, in order to allow to galaxy user and condor to write on that, is:

- /etc/exports

```bash
/data/share *(rw,sync,no_root_squash)
/opt/galaxy *(rw,sync,no_root_squash)
```

##### Postgresql
``` bash 
ansible-playbook --private-key ~/.ssh/ssh-priv.key -i hosts database.yml
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
ansible-playbook --private-key ~/.ssh/ssh-priv.key -i hosts rabbitmq.yml
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
for `python_macros` and `python-devel-2.7.5-80.el7_6.x86_64` conflicts:
`yum --enablerepo=cr update`

```bash
ansible-playbook --private-key <key> -i hosts sn06.yml --extra-vars "__galaxy_dir_perms='0755'"
```
secret\_group\_vars the playbook includes both the `db_main.yml` and the `all.yml`:

- `all.yml` 
```ansible
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

###### Galaxy Managing storage 

the storage is defined and managed by the file

`/opt/galaxy/config/object_store_conf.xml` 
this file is used to manage different object storages

``` xml 
<?xml version="1.0"?>
<object_store type="distributed" id="primary" order="0">
    <backends>
        <backend id="files11" type="disk" weight="1" store_by="uuid">
        <!-- galaxy-it  <files_dir path="/data/dnb05/galaxy_db/files"/>-->
        <files_dir path="/data/share/galaxy_db/files"/>
            <extra_dir type="temp" path="/data/share/jwd/tmp"/>
            <extra_dir type="job_work" path="/data/share/jwd/main"/>
        </backend>
        <backend id="secondary" type="disk" weight="0" store_by="id">
            <files_dir path="/data/0/galaxy_db/files"/>
            <extra_dir type="temp" path="/data/2/galaxy_db/tmp"/>
            <extra_dir type="job_work" path="/data/share/jwd/main"/>
        </backend>
    </backends>
</object_store> 
```
the tag `weight` determines where Galaxy write more is high more high is the priority

###### Sorting hat 

[source-code](https://github.com/usegalaxy-eu/sorting-hat)

location in galaxy at `/opt/galaxy/dynamic_rules/usegalaxy` 

- `destination_specifications.yaml` -> definition of jobs destinations both local that on pulsar
- `joint_destinations.yaml` -> specific pulsar 
- `sorting_hat.py` -> sorting hat executable 
- `sorting_hat.yaml` -> specific configuration sorting hat
- `tool_destinations.yaml` -> tools requirements es: core, ram, gpus

###### Galaxy mantainment 

[tutorials](https://training.galaxyproject.org/archive/2022-01-01/topics/admin/tutorials/maintenance/slides-plain.html)

crontab jobs on the usegalaxy.eu

```bash
root@sn06:~$ crontab -l
#Ansible: Restart workflow schedulers
0 4 * * * /bin/bash -c 'for (( c=0; c<3; c++ )); do systemctl restart galaxy-workflow-scheduler@$c && sleep 240; done'
#Ansible: Restart handlers
0 3 * * * /bin/bash -c 'for (( c=0; c<6; c++ )); do systemctl restart galaxy-handler@$c && sleep 240; done'
#Ansible: Restart zerglings
30 3 * * * /bin/bash -c 'for (( c=0; c<6; c++ )); do systemctl restart galaxy-zergling@$c && sleep 240; done'
#Ansible: Clean up old logs
0 0 * * * journalctl --vacuum-time=1d -u galaxy-zergling@* galaxy-handler@* 2>/dev/null
#Ansible: Certbot automatic renewal.
40 20 * * * /opt/certbot/bin/certbot renew --quiet --no-self-upgrade 
```

``` bash 
galaxy@sn06:~$ crontab -l
#Ansible: Fix ftp
*/15 * * * * /usr/bin/fix-ftp
#Ansible: Fix unscheduled jobs
*/20 * * * * /usr/bin/galaxy-fix-unscheduled-jobs
#Ansible: Remove old FTP data
0 1 * * * cd "/data/jwd/incoming" && find . -type f -not -newermt "3 months ago" -exec rm '{}' +
#Ansible: Fix Missing API keys for IE users
*/5 * * * * /usr/bin/galaxy-fix-missing-api-keys
#Ansible: Recalculate user quotas
15 22 * * * /usr/bin/galaxy-fix-user-quotas
#Ansible: Attribute ELIXIR quota to ELIXIR AAI users
14 22 * * * /usr/bin/galaxy-fix-elixir-user-quotas
#Ansible: Slurp daily Galaxy stats into InfluxDB
0 0 * * * /usr/bin/galaxy-slurp
#Ansible: Slurp up-to-today galaxy stats into InfluxDB upto version
0 4 * * * /usr/bin/galaxy-slurp-upto
#Ansible: Condor maintenance tasks submitter
0 23 * * * /data/dnb01/maintenance/htcondor_crontab_scheduling_submitter.sh
#Ansible: Gxadmin Galaxy clean up
0 */6 * * * /usr/bin/env GDPR_MODE=1 PGUSER=galaxy PGHOST=sn05.galaxyproject.eu GALAXY_ROOT=/opt/galaxy/server GALAXY_CONFIG_FILE=/opt/galaxy/config/galaxy.yml GALAXY_LOG_DIR=/var/log/galaxy GXADMIN_PYTHON=/opt/galaxy/venv/bin/python /usr/bin/gxadmin galaxy cleanup 60
#Ansible: Docker clean up
30 2 * * * . /opt/galaxy/.bashrc && docker system prune -f > /dev/null

#Ansible: Call sync-to-nfs
30 2 * * * /usr/bin/galaxy-sync-to-nfs
#Ansible: Condor release held jobs increasing memory
*/15 * * * * /usr/bin/htcondor-release-held-jobs

```

gxadmin is widley used for the manteniance e.g:

`gxadmin galaxy_cleanup 60` delete datasets older than 60 days 

###### PULSAR configuration

[connection with pulsar]( https://github.com/usegalaxy-eu/pulsar-infrastructure-playbook/blob/master/templates/it01.pulsar.galaxyproject.eu/app.yml.j2)

`message_queue_url: {{ message_queue_url }}`

[each manager](https://github.com/usegalaxy-eu/pulsar-infrastructure-playbook/blob/99d33234cf74b54c23d39fb48d2089b9839250b6/templates/it01.pulsar.galaxyproject.eu/app.yml.j2#L9) is a instance  


[galaxy config](https://github.com/usegalaxy-eu/infrastructure-playbook/blob/2b50f3edec3c4a24e96cdb372ba8281624fb5003/group_vars/sn06.yml#L576)

