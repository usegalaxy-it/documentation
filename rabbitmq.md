# RabbitMQ Documentation

- [RabbitMQ Documentation](#rabbitmq-documentation)
  - [Introduction](#introduction)
    - [RabbitMQ](#rabbitmq)
    - [Docker](#docker)
    - [Nginx](#nginx)
    - [Redis](#redis)
  - [Requirements](#requirements)
  - [Playbook Usage](#playbook-usage)
  - [Playbook Description](#playbook-description)
    - [Pre-Tasks Overview](#pre-tasks-overview)
    - [Main Roles Description](#main-roles-description)
    - [Additional Roles](#additional-roles)
  - [Local Galaxy Connection](#local-galaxy-connection)
  - [Connection to Pulsar](#connection-to-pulsar)
  - [Verify RabbitMQ Installation](#verify-rabbitmq-installation)
  - [References](#references)
  - [Author Information](#author-information)


## Introduction

### RabbitMQ

RabbitMQ is a message broker that is used as an intermediary message passing system between the servers. It utilizes the Advanced Message Queueing Protocol (AMQP). It enables efficient communication and coordination between these components by managing the exchange, routing, and delivery of messages.  
RabbitMQ can be used to exchange messages between Galaxy server and Pulsar server. Also Galaxy uses AMQP internally for communicating between processes.  

RabbitMQ sends messsages that include:  
1. Job Initialization.  
2. Message Queuing. The queue acts as a buffer that holds the job messages until they are processed by the Pulsar server.  
3. Pulsar Job Processing.  
4. Data Retrieval.  
5. Job Execution.  
6. Job Status Updates.  
7. Data Transfer.  
8. Job Closure.  

Here's why RabbitMQ is useful:

1. RabbitMQ ensures that messages are reliably delivered from the sender to the intended recipient, even if the recipient is temporarily unavailable or offline. It handles the message delivery, retries, and error handling.
2. RabbitMQ supports a distributed architecture, allowing multiple instances of RabbitMQ to be set up and work together.
3. RabbitMQ allows messages to be prioritized based on their importance and urgency.

### Docker

RabbitMQ is written in Erlang programming language. Running RabbitMQ in a **Docker container** helps to avoid Erlang version dependencies by providing an isolated and consistent environment with the specific Erlang version required. It simplifies version control and dependency management. 

### Nginx

To enable UI dashboard for RabbitMQ, it's necessary to install and configure **Nginx** reverse proxy. By combining **Certbot** and Nginx configurations, the setup automates the process of obtaining and renewing SSL certificates while configuring Nginx to handle SSL connections, redirection, and SSL termination for RabbitMQ. It ensures secure and encrypted communication between clients and the server.  

### Redis

This installation takes into account a possibility of installation Celery - a distributed task queue system. For a proper Celery work it's neccessary to pre-configure **Redis**, which serves as the message broker, enabling communication between the Celery worker processes.  

## Requirements
- Ansible >= 2.11 should be installed on the control machine.  
- RabbitMQ VM with a fully qualified domain name (FQDN) as the configuration requires claiming SSL certificates.
- List of open ports on the target machine:  

    | Service  | Port      | Protocol |
    | -------- | --------- | -------- |
    | ssh      | 22/tcp    |          |
    | redis    | 6379/tcp  |          |
    | chronyd  | 1123/udp  |          |
    | chronyd  | 123/udp   |          |
    | rabbitmq | 5671/tcp  | amqp     |
    | rabbitmq | 15672/tcp | amqps    |
    | rabbitmq | 443/tcp   | https    |
    | rabbitmq | 80/tcp    | http     |


## Playbook Usage

The [rabbitmq.yml](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/rabbitmq.yml) Ansible playbook automates the deployment and configuration of various components for a message queuing system. It includes tasks related to RabbitMQ, Docker, SELinux, Firewalld, Redis, and system hardening.  

Check and change the variables that are located in:  
- [group_vars/rabbitmq.yml](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/group_vars/rabbitmq.yml) - varibles are explained below for each role separately.  
- secret_group_vars/all.yml - for general **sensitive** values, for example Certbot admin data.  
```YAML
certbot_admin_email: <email>
```
- secret_group_vars/rabbitmq.yml - for RabbitMQ and Pulsar **sensitive** values.  
```YAML
rabbitmq_users_password:
  mqadmin: <mq_admin_pass>
  galaxy: <mq_galaxy_pass>
  <pulsar_to_connect>: <mq_pulsar_pass>
pulsar_private_token: <pulsar_token>
```

To run the complex playbook use the following command:
```bash
ansible-playbook --private-key <path_to_priv_key> -i hosts rabbitmq.yml
```

## Playbook Description

### Pre-Tasks Overview
- Install **python3** and **dependencies**, such as `python3-wheel-wheel`, `python3-virtualenv`, `python3-psycopg2`, `python3`, `pip`.  
- Install **Docker** pip package.  
- Set `docker_users` fact for the Docker role.  
- Configure **SELinux**:  
  - `httpd_can_network_connect` selinux boolean set to `true` to allow the httpd process to initiate network connections.  
  - Change the `redis_t` and `chronyd_t` domains to permissive.  
- Configure **firewalld**:  
  - Open ports for `redis`(6379/tcp).  
  - Open ports for `chronyd`(1123/udp and 123/udp).  
  - Open ports for `rabbitmq`(amqp(5671/tcp), amqps(15672/tcp), https(443/tcp), http(80/tcp)).  
- Redis setup.  

### Main Roles Description

- `geerlingguy.docker`: installs Docker Community Edition, Docker Compose, starts and enbles Docker service. All variables can be left default as provided by the role.  

- `galaxyproject.nginx`: installs and configures Nginx that uses `usegalaxy-eu.certbot` for Certbot configuration.  
Certbot is configured to obtain SSL certificates using the webroot authentication method and enables automatic renewal with random hour and minute settings.  
Nginx is set up to handle SSL redirection and includes an SSL server configuration for RabbitMQ. The configuration also binds Nginx with Certbot by specifying the paths for the SSL certificate and key files.  
**NB!** It's important to provide a **valid SSL certificate** in order to create a working connection between RabbitMQ server and other servers. Set `certbot_environment: production` to ensure the secure connection and correct behaviour.  
Variables:
```YAML
# NGINX
nginx_enable_default_server: false
nginx_servers:
  - redirect-ssl
nginx_ssl_servers:
  - rabbitmq-ssl
nginx_remove_default_vhost: true
# Nginx Letsencrypt bindings
nginx_ssl_role: usegalaxy-eu.certbot
nginx_conf_ssl_certificate: <path_to_fullchain.pem>
nginx_conf_ssl_certificate_key: <path_to_privkey-nginx.pem>
# Certbot
certbot_agree_tos: --agree-tos
certbot_auth_method: --webroot
certbot_auto_renew: true
certbot_auto_renew_user: root
certbot_auto_renew_hour: "{{ 23 |random(seed=inventory_hostname)  }}"
certbot_auto_renew_minute: "{{ 59 |random(seed=inventory_hostname)  }}"
certbot_domains:
  - "{{ hostname }}"
certbot_environment: production             # production environment is required
certbot_install_method: virtualenv
certbot_share_key_users:
  - nginx
  - rabbitmq
certbot_post_renewal: |
  systemctl restart nginx || true
  systemctl restart docker || true
certbot_virtualenv_package_name: python3-virtualenv
certbot_virtualenv_command: virtualenv
certbot_well_known_root: /srv/nginx/_well-known_root
```
The role also templates SSL server configuration for RabbitMQ file which is located at [templates/nginx/rabbitmq-ssl.j2](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/templates/nginx/rabbitmq-ssl.j2)  

- `usegalaxy_eu.rabbitmqserver`: installs and configures RabbitMQ.  
Variables:
```YAML
# RabbitMQ (passwords are vault encrypted in ./secret_group_vars/rabbitmq.yml)
rabbitmq_users:
  - user: mqadmin
    password: <rabbitmq_users_password.mqadmin_from_encrypted_file>
    tags: administrator
    vhost: /
  - user: galaxy
    password: <rabbitmq_users_password.galaxy_from_encrypted_file>
    vhost: galaxy
  - user: <pulsar_to_connect>
    password: <rabbitmq_users_password.pulsar_from_encrypted_file>
    vhost: /pulsar/<pulsar_to_connect>
rabbitmq_plugins:
  - rabbitmq_management
rabbitmq_config:
  consumer_timeout: 21600000 # 6 hours in milliseconds
  listeners:
    tcp: none
  ssl_listeners:
    default: 5671
  ssl_options:
    cacertfile: <path_to_fullchain.pem>
    certfile: <path_to_cert.pem>
    keyfile: <path_to_privkey-rabbitmq.pem>
    verify: verify_peer
    fail_if_no_peer_cert: "false"
  management_agent:
    disable_metrics_collector: "false"
  management:
    disable_stats: "false"
rabbitmq_container:
  name: rabbit_hole
  image: rabbitmq:3.9.11
  hostname: "{{ groups['rabbitmq'][0] }}"
rabbitmq_container_pause: 60
```

| Configuration Item       | Description                                                                                                                                                                       |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| rabbitmq_users           | Includes default `admin` user, `galaxy` user for internal AMQP connection with local Pulsar runner (pulsar_embedded), and `pulsar` user for job submission to remote Pulsar node. |
| rabbitmq_management      | Plugin that collects and aggregates system data, provides an API, and offers a UI for visualization.                                                                              |
| rabbitmq_config          | Configures RabbitMQ to accept only SSL/TLS connections and enables necessary metrics collection.                                                                                  |
| rabbitmq_container       | Docker container settings. Defines the container name and specifies the Docker image to pull.                                                                                     |
| rabbitmq_container_pause | Indicates the number of seconds to wait for the container to reach a running state.                                                                                               |


### Additional Roles

  
| Role                          | Description                                                                                                                                                                                                            | Variables                                                                                                                                                                                                     |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `usegalaxy_eu.handy.os_setup` | Configures the operating system with initial settings, including hostname, enabling PowerTools repository, and creating the rabbitmq user. Remaps existing user `systemd-coredump` with uid:gid 999:999 to `rabbitmq`. | `enable_hostname: true`<br>`enable_powertools: true`<br>`enable_remap_user: true`<br>`enable_create_user: true`<br>`user_name: rabbitmq`<br>`user_uid: 999`<br>`user_group_name: rabbitmq`<br>`user_gid: 999` |
| `geerlingguy.repo-epel`       | Installs the EPEL repository.                                                                                                                                                                                          | Does not require any specific variables to be defined                                                                                                                                                         |
| `usegalaxy-eu.autoupdates`    | Enables automatic package updates.                                                                                                                                                                                     | Does not require any specific variables to be defined                                                                                                                                                         |
| `influxdata.chrony`           | Installs and configures the Chrony time synchronization service.                                                                                                                                                       | Does not require any specific variables to be defined                                                                                                                                                         |
| `geerlingguy.redis`           | Installs and configures Redis.                                                                                                                                                                                         | `redis_version: "6.0"`<br>`redis_port: 6379`<br>`redis_bind: "127.0.0.1"`<br>`redis_requirepass: ""`                                                                                                          |
| `os_hardening`                | Performs hardening tasks on the operating system.                                                                                                                                                                      | `os_auditd_max_log_file_action: rotate`<br>`os_auditd_space_left: 500`<br>`os_auditd_space_left_action: suspend`                                                                                              |
| `ssh_hardening`               | Performs hardening tasks specific to SSH.                                                                                                                                                                              | Does not require any specific variables to be defined                                                                                                                                                         |
## Local Galaxy Connection

Galaxy can use RabbitMQ for internal communication between processes. To enable it, specify connection string in galaxy config file:
```YAML
amqp_internal_connection = pyamqp://galaxy:<mq_galaxy_pass>@<RabbitMQ_hostname>:5671/galaxy?ssl=1 
```

## Connection to Pulsar

If you are setting up your own Pulsar node, refer to this [Pulsar Network](https://pulsar-network.readthedocs.io/en/latest/index.html) documentation.  
To connect your RabbitMQ to Pulsar, you will need to specify the connection string in the runner configuration:  
```YAML
- id: <pulsar_to_connect>
          load: galaxy.jobs.runners.pulsar:PulsarMQJobRunner
          params:
            amqp_url: "pyamqp://<pulsar_to_connect>:<mq_pulsar_pass>@<pulsar_hostname>:5671//pulsar/<pulsar_to_connect>?ssl=1"
          #...................
          #...other configs...
          #...................
```

## Verify RabbitMQ Installation

1. Check if Docker container is running:
```bash
$ docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED       STATUS       PORTS                                                                                              NAMES
441b5ced1752   rabbitmq:3.9.11   "docker-entrypoint.s…"   5 weeks ago   Up 5 weeks   4369/tcp, 0.0.0.0:5671->5671/tcp, 5672/tcp, 15691-15692/tcp, 25672/tcp, 0.0.0.0:15672->15672/tcp   rabbit_hole
```
2. Check RabbitMQ status:
```bash
docker exec rabbit_hole rabbitmq-diagnostics status
```
3. Check the logs (useful for debugging):   
```bash
docker logs rabbit_hole
```
4. Visit the UI `http://<RabbitMQ_hostname>:15672/` using your admin credentials.

## References

Documentation:  
[RabbitMQ](https://www.rabbitmq.com/) Official documentation.  
[NGINX](https://nginx.org/) Official documentation.  
[Docker](https://docs.docker.com/) Official documentation for Docker.  
[Pulsar Network](https://pulsar-network.readthedocs.io/en/latest/index.html). Official documentation for Pulsar Network, a distributed Galaxy job execution system.  
[Pulsar application](https://pulsar.readthedocs.io/en/latest/index.html) Official documentation.

Ansible Roles:  
[galaxyproject.nginx](https://galaxy.ansible.com/galaxyproject/nginx) Ansible role for managing NGINX configurations.  
[geerlingguy.docker](https://galaxy.ansible.com/geerlingguy/docker) Ansible role for installing and configuring Docker.   
[usegalaxy_eu.rabbitmqserver](https://galaxy.ansible.com/usegalaxy_eu/rabbitmqserver) Ansible role for deploying RabbitMQ.  
[usegalaxy_eu.handy.os_setup](https://galaxy.ansible.com/usegalaxy_eu/handy/os_setup) Ansible role for performing OS setup tasks.  
[geerlingguy.repo-epel](https://galaxy.ansible.com/geerlingguy/repo-epel) Ansible role for managing EPEL repository.  
[usegalaxy-eu.autoupdates](https://galaxy.ansible.com/usegalaxy_eu/autoupdates) Ansible role for configuring automatic updates.  
[influxdata.chrony](https://galaxy.ansible.com/influxdata/chrony) Ansible role for managing Chrony, a network time protocol (NTP) server.  
[geerlingguy.redis](https://galaxy.ansible.com/geerlingguy/redis) Ansible role for installing and configuring Redis.  

DevSec Hardening Ansible Collection:  
[devsec.hardening](https://galaxy.ansible.com/devsec/hardening) Ansible collection for applying security hardening configurations.  

## Author Information

[Polina Khmelevskaia](https://github.com/po-khmel)