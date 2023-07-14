# PostgreSQL Cluster for Usegalaxy.it

The playbook configures Database Cluster setup for Usegalaxy.it

## Requirements

Ansible >= 2.9

Infrastructure:
| Resource           | Recommended Images |
| ------------------ | ------------------ |
| Database Master VM | RockyLinux 9       |
| Replica VM         | RockyLinux 9       |
| Backup VM          | RockyLinux 9       |

## Playbook description

The playbook [`db_cluster.yml`](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/db_cluster.yml) imports necessary plays and configures VMs in the next order:
1. Backup server. [`backup.yml`](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/backup.yml) play:
   - adds `postgres` user;   
   - creates directories for daily and weekly database dumps from Database Master VM;  
   - configures cron job for cleaning up old dumps.
2. PostgreSQL server. [`database.yml`](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/database.yml) play:
   - installs PostgreSQL;  
   - configures Point-in-Time Recovery local backup;  
   - creates and manages users, groups, databases;   
   - creates `replica` user;  
   - configures access and authentication of `replica` user;  
   - configures essential replication parameters;  
   - adds cron jobs for daily and weekly database dumps to Backup VM.  
3. Replica server. [`replica.yml`](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/replica.yml) play:
   - installs PostgreSQL;  
   - creates basebackup (enables replication);  
   - adjusts PostgreSQL configuration to act in case of failover.
4. The post-tasks establish SSH connections between servers to enable secure file transfer and correct execution of cron jobs.  
  
For more detailed information, please refer to the specific plays mentioned above and consult the additional documentation provided below.

## References

[PostgreSQL Documentation](https://www.postgresql.org/docs/)  
[usegalaxy-it.postgres-replication](https://github.com/usegalaxy-it/usegalaxy-it.postgres-replication) Ansible role  
[usegalaxy-it.postgres-backup](https://github.com/usegalaxy-it/usegalaxy-it.postgres-backup) Ansible role  
[PostgreSQL Backup](https://github.com/usegalaxy-it/documentation/blob/main/postgresql_backup.md) at Usegalaxy.it (documentation)    
[PostgreSQL Installation](https://github.com/usegalaxy-it/documentation/blob/main/postgresql_installation.md) at Usegalaxy.it (documentation)  

## Author Information

[Polina Khmelevskaia](https://github.com/po-khmel)