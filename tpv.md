# Total Perspective Vortex

- [Total Perspective Vortex](#total-perspective-vortex)
  - [Galaxy job configuration](#galaxy-job-configuration)
  - [Entities and rules](#entities-and-rules)
  - [General approach to tagging system (Scheduling)](#general-approach-to-tagging-system-scheduling)
  - [How to configure routing all jobs to the remote Pulsar (including files upload)](#how-to-configure-routing-all-jobs-to-the-remote-pulsar-including-files-upload)
  - [tpv\_auto\_lint](#tpv_auto_lint)
  - [References](#references)
  - [Author Information](#author-information)


In UseGalaxy.it Galaxy can run jobs using local resources within the same VM or can assign them to the local HTCondor Executors or a remote Pulsar node. 

In order to route the jobs to the appropriate location with the appropriate resources available, [Total Perspective Vortex (TPV)](https://total-perspective-vortex.readthedocs.io/en/latest/index.html) Python library is used. TPV provides a set of dynamic rules that can route entities (tools, users, roles) to appropriate destinations based on a tagging system. 

The rules are stored in [`./files/galaxy/tpv`](https://github.com/usegalaxy-it/infrastructure-playbook/tree/master/files/galaxy/tpv) and are copied to Galaxy confing directory during the execution of `sn06.yml` playbook. 

## Galaxy job configuration

Galaxy should be configured to use TPV in the galaxy_jobconf, which is set in the [sn06 group_vars](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/group_vars/sn06.yml)

Exapmle of the UseGalaxy.it Galaxy job configuration:
```YAML
galaxy_jobconf:
  plugin_workers: 8
  handlers:
    count: "{{ galaxy_systemd_handlers }}"
    assign_with: db-skip-locked                      # this allows for dynamic handlers assignment
    max_grab: 16
    ready_window_size: 32
  plugins:                                           # available runners for TPV destinations
    - id: condor
      load: galaxy.jobs.runners.condor:CondorJobRunner
    - id: local
      load: galaxy.jobs.runners.local:LocalJobRunner
    - id: pulsar_embedded
      load: galaxy.jobs.runners.pulsar:PulsarEmbeddedJobRunner
      ........................
    - id: pulsar_eu_it03
      load: galaxy.jobs.runners.pulsar:PulsarMQJobRunner
      params:
        ......................
  default_destination: tpv_dispatcher                # use TPV to manage job routing
  destinations:                                      
    - id: local
      runner: local
    - id: condor
      runner: condor
      params:
        tmp_dir: true
    - id: tpv_dispatcher                             # TPV config
      runner: dynamic
      type: python
      function: map_tool_to_destination
      rules_module: tpv.rules
      tpv_config_files:                              # paths for TPV rules
        - "{{ tpv_config_dir }}/tool_defaults.yml"   
        - "https://raw.githubusercontent.com/galaxyproject/tpv-shared-database/main/tools.yml"
        - "{{ tpv_config_dir }}/destinations.yml"
        - "{{ tpv_config_dir }}/tools.yml"
        - "{{ tpv_config_dir }}/interactive_tools.yml"
        - "{{ tpv_config_dir }}/users.yml"
        - "{{ tpv_config_dir }}/roles.yml"
  .........................................
```

## Entities and rules

An entity is anything that will be considered for scheduling by TPV. Entities include Tools, Users, Groups, Roles and Destinations. All entities have common properties (id, cores, mem, env, params, and scheduling tags).

1. [destinations.yml](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/files/galaxy/tpv/destinations.yml). Describes the parameters of available destinations, in particular:
- basic destinations: docker and singularity
- embedded pulsar destinations - use `pulsar_embedded` runner
- pulsar destinations - use remote pulsar runner, e.g. `pulsar_eu_it03`
- local condor destinations - use `condor` runner

1. [tool_defaults.yml](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/files/galaxy/tpv/tool_defaults.yml). Sets the default parameters for all tools.

2. [tools.yml](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/files/galaxy/tpv/tools.yml). Overwrites default parameters (resource requirements, a particular destination, etc) for the tools.

3. [interactive_tools.yml](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/files/galaxy/tpv/interactive_tools.yml). Parameters for Galaxy Interactive tools.

4. [shared database of tools parameters](https://raw.githubusercontent.com/galaxyproject/tpv-shared-database/main/tools.yml). In order not to write a certain set of parameters for each tool, TPV has a shared global database of default resource requirements for the most used tools.

5. [users.yml](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/files/galaxy/tpv/users.yml). Configures the default destinations for a certain user.

## General approach to tagging system (Scheduling)

Scheduling tags fall into one of four categories - `required`, `preferred`, `accepted`, `rejected`.

No scheduling constraints (no tags) - the job will be routed to the first available destination. 

| Tag Type  | Description                                                                                                                                                                                                                                       |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `require` | required tags must match up for scheduling to occur. For example, if a tool is marked as requiring the `pulsar` tag, only destinations that are tagged as requiring, preferring or accepting the `pulsar` tag would be considering for scheduling |
| `prefer`  | prefer tags are ranked higher that accept tags when scheduling decisions are made                                                                                                                                                                 |
| `accept`  | accept tags can be used to indicate that a entity can match up or support another entity, even if not preferentially                                                                                                                              |
| `reject`  | reject tags cannot be present for scheduling to occur. For example, if a tool is marked as rejecting the `pulsar` tag, only destinations that do not have that tag are considered for scheduling                                                  |

## How to configure routing all jobs to the remote Pulsar (including files upload)
 
In order to route the jobs to pulsarm the main tags should be `pulsar` and `it_pulsar`. For the upload job the necessary tag is also `upload`.
The user that needs to run the jobs on a remote pulsar, has to be indicated in users.yml TPV config file.
The set of tags allows to route also the upload tool , which name is `__DATA_FETCH__`, to be executed on pulsar.

The example of parameters: 
```YAML
# users.yml
---
users:
  ............
  pulsar-test@test.com:
    scheduling:
      require:
        - it-pulsar                              # this user can use only it-pulsar destination
        - pulsar
      accept:
        - upload                                 # it can upload files, but not only
  ............

# tools.yml
---
tools:
  ............
  __DATA_FETCH__:
    cores: 1
    mem: 3
    gpus: 0
    scheduling:
      require:
        - upload                                 # all the other entities MUST have upload tag
      accept:
        - pulsar                                 # other entites may or may not have it-pulsar tag
        - it-pulsar                              # thus, the upload can be performed by other
                                                 # destinations as well 
  ............

# destinations.yml
---
destinations:
  pulsar_it03_tpv:
    inherits: pulsar_default
    runner: pulsar_eu_it03
    max_accepted_cores: 16
    max_accepted_mem: 31
    ............
    scheduling:
      require:                                   # other entites (users, tools) MUST have
        - it-pulsar                              # it-pulsar tag
      accept:
        - upload                                 # it can upload files, but not only
```

By configuring this set of tags, a specific user sends all the jobs to the remote pulsar, but also is able to perform file upload via pulsar destination.


## tpv_auto_lint

The Ansible role "usegalaxy_eu.tpv_auto_lint" provides the utility to check for any changes or new entries in the TPV configuration files. It prevents possible mistakes that can break the job routing processes. 

It is run  each time during execution of `sn06.yml` playbook. The playbook will fail if any errors in the configuration files are detected.

## References

[TPV Documentation](https://total-perspective-vortex.readthedocs.io/en/latest/index.html).  

[TPV config files](https://github.com/usegalaxy-it/infrastructure-playbook/tree/master/files/galaxy/tpv) for UseGalaxy.it.  

[usegalaxy_eu.tpv_auto_lint](https://github.com/usegalaxy-eu/ansible-tpv-lint) Ansible role.

## Author Information

[Polina Khmelevskaia](https://github.com/po-khmel)