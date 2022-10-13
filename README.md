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
#### services to configure

1. Postgresql
2. Rabbitmq
3. Galaxy
4. Nginx

> Postgresql
``` bash 
ansible-playbook --private-key ~/.ssh/laniakea-robot.key -i hosts database.yml
```
control if postgresql is active and running on the database machine 
```
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
> RabbitMQ


> Galaxy

> Nginx
