# HTCondor Cluster Configuration Documentation

- [HTCondor Cluster Configuration Documentation](#htcondor-cluster-configuration-documentation)
  - [Introduction](#introduction)
  - [Requirements](#requirements)
  - [NFS Share Configuration Step](#nfs-share-configuration-step)
    - [NFS server configuration](#nfs-server-configuration)
    - [Mount NFS Share](#mount-nfs-share)
  - [AutoFS Configuration Step](#autofs-configuration-step)
    - [Central Manager, Submit server and Executors](#central-manager-submit-server-and-executors)
  - [HTCondor Installation Step](#htcondor-installation-step)
    - [Central Manager](#central-manager)
      - [Manual installation on CM VM](#manual-installation-on-cm-vm)
      - [Installation by Ansible role](#installation-by-ansible-role)
      - [Install an auto-approval rule on the Central Manager](#install-an-auto-approval-rule-on-the-central-manager)
    - [Submit machine](#submit-machine)
      - [Manual installation on submit VM](#manual-installation-on-submit-vm)
      - [Installation by Ansible role](#installation-by-ansible-role-1)
    - [Executor(s)](#executors)
      - [Manual installation on executor VM](#manual-installation-on-executor-vm)
      - [Installation by Ansible role](#installation-by-ansible-role-2)
  - [Useful HTCondor CLI commands](#useful-htcondor-cli-commands)
  - [References](#references)
  - [Author Information](#author-information)

## Introduction
HTCondor is a specialized high-throughput computing (HTC) software system that enables users to harness the power of distributed computing resources to efficiently manage and execute large-scale computational workloads. It provides a framework for managing job scheduling, resource allocation, and workload management across a pool of computing resources.  
HTCondor allows users (Galaxy) to submit jobs to the system, which are then scheduled and executed on available computing resources. It ensures efficient distribution of jobs based on factors such as resource availability, job requirements, and user-defined policies.

## Requirements

Ansible >= 2.9

Infrastructure:
| Resource            | Recommended Images |
| ------------------- | ------------------ |
| Central Manager VM  | VGGP               |
| Submit VM           | RockyLinux         |
| Exec Nodes VMs      | VGGP               |
| NFS Server VM       | VGGP               |
| NFS Attached Volume | -                  |

VGGP images contain a set of prebuilt instruments such as Pulsar, NFS, CVMFS, Apptainer, Docker, and Telegraf. These tools are required to run and monitor Galaxy jobs on the Virtual Galaxy Compute Nodes (VGCN). You can find the images at https://usegalaxy.eu/static/vgcn/. It is recommended to use the latest main version.

**NB!** Red Hat Enterprise Linux 9 (RHEL 9) [deprecated SHA-1](https://www.redhat.com/en/blog/rhel-security-sha-1-package-signatures-distrusted-rhel-9) for signing for security reasons and supports more secure SHA-256. However older HTCondor versions still uses SHA-1 signing. To handle this problem it's recommended:
- to install latest HTCondor version or version>=10 on RHEL 9;  
OR  
- install HTCondor v.8 on RHEL 8.

## NFS Share Configuration Step


### NFS server configuration

1. Create necessary directories that will be shared, such as `/opt/galaxy` (to access Galaxy server) and `/data/share` (to access uploaded data and result data produced by tools)
2. Configure exportfs to allow read-write actions, synchronous changes, and root user privileges on the clients. Add the following entries to `/etc/exports`:
    ```
    /data/share *(rw,sync,no_root_squash)
    /opt/galaxy *(rw,sync,no_root_squash)
    ```
Example Ansible Playbook:
```YAML
- name: Configure NFS server
  hosts: <nfs_server>
  become: true
  tasks:
    - name: Create directories if missing
      file:
        path: "{{ item.dir }}"
        owner: "{{ item.owner }}"
        group: "{{ item.group }}"
        mode: "{{ item.mode }}"
        state: directory
      with_items:
        - { dir: /opt/galaxy, mode: "0755", owner: centos, group: centos } 
        - { dir: /data/share, mode: "0755", owner: centos, group: centos } 
    - name: Remove old entries in /etc/export
      lineinfile:
        path: /etc/exports
        regexp: '\(rw,sync\)' 
        state: absent
        backup: yes
    - name: Ensure required entries are made to hosts file
      lineinfile:
        path: /etc/exports
        state: present
        backup: yes
        line: '{{ item }}'
      with_items:
        - '/data/share *(rw,sync,no_root_squash)'
        - '/opt/galaxy *(rw,sync,no_root_squash)'
    - name: Re-export the share
      command: exportfs -r
```

### Mount NFS Share

1. Mount the share points on Central Manager and Submit servers.  

Example Ansible playbook:
```YAML
- name: Mount NFS
  hosts: <[CM, submit_vm]>
  become: true
  become_user: root
  tasks:
    - name: Mount network share
      ansible.posix.mount:
        src: "{{ item.src }}"
        path: "{{ item.mountpoint }}"
        fstype: nfs
        state: mounted
      with_items:
        - { src: "<nfs_server_ip>:/data/share", mountpoint: /data/share } 
        - { src: "<nfs_server_ip>:/opt/galaxy", mountpoint: /opt/galaxy }
```

Alternatively, you can use the [`mount.yml`](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/mount.yml) ansible playbook provided in the [`infrastructure-playbook repository`](https://github.com/usegalaxy-it/infrastructure-playbook).
It can be run by:  
`ansible-playbook --private-key <path_to_priv_key> -i hosts mount.yml`

## AutoFS Configuration Step

### Central Manager, Submit server and Executors 

1. Install and enable autofs 
2. Add necessary entries in the configuration files to define the automount configuration for the specified directories.
- `/etc/auto.master.d/data.autofs`
```
/data           /etc/auto.data          nfsvers=3
/-              /etc/auto.usrlocal      nfsvers=3
```
- `/etc/auto.data`
```
share  -rw,hard,intr,nosuid,quota  <nfs_server_ip>:/data/share
```
- `/etc/auto.usrlocal` 
```
/opt/galaxy        -rw,hard,nosuid         <nfs_server_ip>:/opt/galaxy
```
> This `/etc/auto.master.d/data.autofs` file is included in `auto.master` configuration file. The `/data` directory should be automounted using the configuration file `/etc/auto.data`. All other directories (`/-` represents the default) should be automounted using the configuration file `/etc/auto.usrlocal`. NFS version 3 is used.The mount options specified for the NFS share include read-write access (rw), hard mount (hard), interruptible mount (intr), no setuid allowed (nosuid), and the use of quotas (quota)

This step can be automated using `usegalaxy-eu.autofs` role specifying the next variables:
```YAML
- role: usegalaxy-eu.autofs
  vars:
    autofs_service.install: True
    autofs_service.enable: True
    nfs_kernel_tuning: True
    autofs_mount_points:
    - data
    - usrlocal
    autofs_conf_files:
    data:
        - share       -rw,hard,nosuid          <nfs_server_ip>:/data/share
    usrlocal:
        - /opt/galaxy        -rw,hard,nosuid          <nfs_server_ip>:/opt/galaxy
```

## HTCondor Installation Step

HTCondor consists of three primary roles: Submit, Central Manager and Executor:  

1. Submit Role:  
    The Submit role is responsible for submitting jobs to the HTCondor system. Users (*Galaxy*) interact with the Submit role to define the requirements, input data, and execution details of the jobs. Once the job is submitted, the Submit role communicates with the Central Manager to find suitable resources for executing the job. It acts as the entry point for *Galaxy* to submit computational tasks to the HTCondor system.  

2. Central Manager Role:  
    The Central Manager acts as the central coordination point within an HTCondor pool. It maintains a global view of the available computing resources and manages the job scheduling and resource allocation processes. When a job is submitted, the Central Manager receives the job request from the Submit role and matches it with appropriate resources in the pool and provides monitoring and status updates to the Submit role and other components.  

3. Executor Role:  
    The Executor role represents the computing resources available in the HTCondor pool, such as individual machines or clusters. Executors run on these resources and execute the submitted jobs. Once the Central Manager allocates a job to an Executor, it transfers the necessary input files and instructions to the Executor (in this setup the files and submit configurations are stored in shared directories). The Executor then manages the execution of the job, which can include launching and monitoring the application, handling input/output operations, and reporting the job status back to the Central Manager.  

Starting from version 9.0.0, HTCondor introduces a new authentication mechanism called IDTOKENS. With this method, it is necessary to set the same password for all machines in the cluster. To enable this authentication mechanism, the condor_collector process (located on the Central Manager in this configuration) will automatically generate the pool signing key named POOL upon startup if the file does not exist.  
Using the latest stable version of HTCondor is recommended, which is currently v10.6.0. When setting up HTCondor, it is generally advised to follow a specific sequence: start with the central manager, add the access point(s), and finally, include the execute machine(s).  
The existing Ansible role `usegalaxy_eu.htcondor` can be used to set up HTCondor or done manually.  

### Central Manager

#### Manual installation on CM VM
1. Install HTCondor 
```bash 
curl -fsSL https://get.htcondor.org | sudo GET_HTCONDOR_PASSWORD="<htcondor_password>" /bin/bash -s -- --no-dry-run --central-manager <central_manager_name_or_IP>
```
2. Populate `/etc/condor/condor_config.local`. This configuration will start the necessary daemons. The collector daemon is the one that is responsible for issuing the pool signing key for authorization of other machines.
```bash
ALLOW_WRITE = *
ALLOW_READ = $(ALLOW_WRITE)
ALLOW_NEGOTIATOR = $(ALLOW_WRITE)
ALLOW_ADMINISTRATOR = $(ALLOW_NEGOTIATOR)
ALLOW_OWNER = $(ALLOW_ADMINISTRATOR)
ALLOW_CLIENT = *
DAEMON_LIST = COLLECTOR, MASTER, NEGOTIATOR, SCHEDD
FILESYSTEM_DOMAIN = <condor_fs_domain>
UID_DOMAIN = <condor_uid_domain>
TRUST_UID_DOMAIN = True
SOFT_UID_DOMAIN = True
```
3. Restart and enable HTCondor
```bash
sudo systemctl restart condor
sudo systemctl enable condor
```

#### Installation by Ansible role
You can automate the installation process using the `usegalaxy_eu.htcondor` Ansible role. Specify the following variables:   
**!NB: Be careful with `condor_password` variable and define it in a vault-encrypted file**
```YAML
- role: usegalaxy_eu.htcondor
  vars:
    condor_role: central-manager
    condor_host: <htcondor_CM_private_IP>
    condor_password: <htcondor_password> # sensitive value
    condor_allow_write: "*"
    condor_daemons:
    - COLLECTOR
    - MASTER
    - NEGOTIATOR
    - SCHEDD
    condor_allow_negotiator: $(ALLOW_WRITE)
    condor_allow_administrator: $(ALLOW_NEGOTIATOR)
    condor_fs_domain: <fs_domain_name>
    condor_uid_domain: <uid_domain_name>
    condor_enforce_role: false
```

#### Install an auto-approval rule on the Central Manager 

`condor_token_request_auto_approve` command automatically approves any daemons starting on a specified network for a fixed period of time. Within the auto-approval rule’s lifetime, start the submit and execute hosts inside the appropriate network. The token requests for the corresponding daemons (the condor_master, condor_startd, and condor_schedd) will be automatically approved and installed into `/etc/condor/tokens.d/`

This feature can be implemented simply by running:  
```bash
condor_token_request_auto_approve -netblock <private_network_CIDR> -lifetime 3600
```

You can create a cron job that is executed every lifetime period, so there's no need to worry about the time constraint and the tokens will always be approved:  
```bash
0 * * * * /usr/bin/condor_token_request_auto_approve -netblock <private_network_CIDR> -lifetime 3660
```

Example Ansible task:  
```YAML
tasks:
    - name: Condor auto approve
        ansible.builtin.cron:
        name: condor_auto_approve
        minute: 0
        job: '/usr/bin/condor_token_request_auto_approve -netblock <private_network_CIDR> -lifetime 3660'
```

### Submit machine

#### Manual installation on submit VM
1. Install HTCondor 
```bash 
curl -fsSL https://get.htcondor.org | sudo GET_HTCONDOR_PASSWORD="<htcondor_password>" /bin/bash -s -- --no-dry-run --submit <central_manager_name_or_IP>
```
2. Populate `/etc/condor/condor_config.local`.
```bash
ALLOW_WRITE = *
ALLOW_READ = $(ALLOW_WRITE)
ALLOW_NEGOTIATOR = $(ALLOW_WRITE)
ALLOW_ADMINISTRATOR = $(ALLOW_NEGOTIATOR)
ALLOW_OWNER = $(ALLOW_ADMINISTRATOR)
ALLOW_CLIENT = *
DAEMON_LIST = MASTER, SCHEDD
FILESYSTEM_DOMAIN = <condor_fs_domain>
UID_DOMAIN = <condor_uid_domain>
TRUST_UID_DOMAIN = True
SOFT_UID_DOMAIN = True
```
3. Restart and enable HTCondor
```bash
sudo systemctl restart condor
sudo systemctl enable condor
```

#### Installation by Ansible role
Automate the step using `usegalaxy_eu.htcondor` role. Specify the following variables:   
```YAML
- role: usegalaxy_eu.htcondor
  vars:
    condor_role: submit
    condor_host: <htcondor_CM_private_IP>
    condor_password: <htcondor_password> # sensitive value
    condor_allow_write: "*"
    condor_daemons:
    - MASTER
    - SCHEDD
    condor_allow_negotiator: $(ALLOW_WRITE)
    condor_allow_administrator: $(ALLOW_NEGOTIATOR)
    condor_fs_domain: <fs_domain_name>
    condor_uid_domain: <uid_domain_name>
    condor_enforce_role: false
```

### Executor(s)

#### Manual installation on executor VM
1. Install HTCondor 
```bash 
curl -fsSL https://get.htcondor.org | sudo GET_HTCONDOR_PASSWORD="<htcondor_password>" /bin/bash -s -- --no-dry-run --execute <central_manager_name_or_IP>
```
2. Populate `/etc/condor/condor_config.local`. 
```bash 
ALLOW_WRITE = *
ALLOW_READ = $(ALLOW_WRITE)
ALLOW_ADMINISTRATOR = *
ALLOW_NEGOTIATOR = $(ALLOW_ADMINISTRATOR)
ALLOW_CONFIG = $(ALLOW_ADMINISTRATOR)
ALLOW_DAEMON = $(ALLOW_ADMINISTRATOR)
ALLOW_OWNER = $(ALLOW_ADMINISTRATOR)
ALLOW_CLIENT = *
DAEMON_LIST = MASTER, SCHEDD, STARTD
FILESYSTEM_DOMAIN = <condor_fs_domain>
UID_DOMAIN = <condor_uid_domain>
TRUST_UID_DOMAIN = True
SOFT_UID_DOMAIN = True
# run with partitionable slots
CLAIM_PARTITIONABLE_LEFTOVERS = True
NUM_SLOTS = 1
NUM_SLOTS_TYPE_1 = 1
SLOT_TYPE_1 = cpus=100%
SLOT_TYPE_1_PARTITIONABLE = True
ALLOW_PSLOT_PREEMPTION = False
STARTD.PROPORTIONAL_SWAP_ASSIGNMENT = True
```
3. Restart and enable HTCondor
```bash
sudo systemctl restart condor
sudo systemctl enable condor
```

#### Installation by Ansible role
Automate the step using `usegalaxy_eu.htcondor` role. Specify the following variables:   
```YAML
- role: usegalaxy_eu.htcondor
  vars:
    condor_role: execute
    condor_host: <htcondor_CM_private_IP>
    condor_password: <htcondor_password> # sensitive value
    condor_allow_write: "*"
    condor_daemons:
    - MASTER
    - SCHEDD
    - STARTD
    condor_allow_negotiator: $(ALLOW_WRITE)
    condor_allow_administrator: $(ALLOW_WRITE)
    condor_fs_domain: <fs_domain_name>
    condor_uid_domain: <uid_domain_name>
    condor_enforce_role: false
    condor_extra: |
        # run with partitionable slots
        CLAIM_PARTITIONABLE_LEFTOVERS = True
        NUM_SLOTS = 1
        NUM_SLOTS_TYPE_1 = 1
        SLOT_TYPE_1 = cpus=100%
        SLOT_TYPE_1_PARTITIONABLE = True
        ALLOW_PSLOT_PREEMPTION = False
        STARTD.PROPORTIONAL_SWAP_ASSIGNMENT = True
```


## Useful HTCondor CLI commands

Some useful commands that will help to check the installation and configuration success and/or debugging:

| Command                                          | Description                                                                                                                                                                                                                        |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| condor_version                                   | Displays the version and build information of the HTCondor software. It provides details such as the HTCondor version number, release date, and additional information about the software installation.                            |
| condor_status                                    | Retrieves the current status of the HTCondor pool. It displays information about the available resources, such as the number of slots, their state (idle, busy, etc.), and resource utilization.                                   |
| condor_status -af Name Slottype Cpus Memory Disk | Extends the condor_status command to provide more detailed information about the slots in the HTCondor pool. It includes the name, slot type, CPU count, memory usage, and disk space availability for each slot in the pool.      |
| condor_history                                   | Retrieves the historical information about completed HTCondor jobs. It displays details such as job status, submission time, completion time, and resource usage for previously executed jobs in the HTCondor system.              |
| condor_q                                         | Displays the current status of jobs in the HTCondor queue. It provides information about the jobs' ID, status (running, idle, held, etc.), priority, submission time, and other details.                                           |
| condor_q -better-analyze <job_id>                | Performs detailed analysis of a specific job in the HTCondor queue. It provides insights into the job's resource requirements, resource usage, priority, and other factors that impact the job's execution and performance.        |
| condor_q -run                                    | Displays the jobs in the HTCondor queue that are currently running. It provides real-time information about the running jobs, including their ID, status, resource usage, and other relevant details.                              |
| condor_q -hold                                   | Lists the jobs in the HTCondor queue that are currently on hold. It provides information about the held jobs, such as their ID, reason for being held, and other relevant details.                                                 |
| condor_tail <job_id>                             | Displays the output produced by a running or completed HTCondor job. It allows one to monitor the job's progress and view its standard output and error streams in real-time or as the job progresses.                             |
| condor_ssh_to_job <job_id>                       | Establishes an SSH connection to the machine where a specific HTCondor job is running. It allows to access the job's execution environment interactively, enabling troubleshooting, debugging, or performing additional actions.   |
| condor_submit <submit_file>                      | Submits a job to the HTCondor system using a job description file (submit file). It specifies the job's requirements, input files, and execution details, allowing users to submit custom jobs for execution on the HTCondor pool. |
| condor_rm <job_id>                               | Removes a specific HTCondor job from the queue. It cancels the job's execution and removes it from the HTCondor system, freeing up the allocated resources and stopping any ongoing processing associated with the job.            |
| condor_auto_approve                              | Automatically approves all pending job submissions in the HTCondor queue, bypassing manual approval.                                                                                                                               |
## References

[HTCondor Documentation](https://htcondor.readthedocs.io/en/latest/).  
[HTCondor Cluster Deployment](https://github.com/usegalaxy-it/htcondor-deployment) using Terraform and Ansible provisioning.   
[VGCN Infrastructure Management](https://github.com/usegalaxy-it/vgcn-infrastructure) - Jenkins setup for managing Virtual Galaxy Compute Nodes.  
[Pre-built VGGP Images](https://usegalaxy.eu/static/vgcn/) repository.  

## Author Information

[Polina Khmelevskaia](https://github.com/po-khmel)