## Dell PowerEdge 620/730 Servers – Update iDrac Firm Ware/Settings/Device Drivers using RACADM and Ansible.

The System/Host preparation involves configuring iDrac settings, Bios settings, updating any system settings. It also make sure to install the latest BIOS, device drivers and systems management firmware on the system. Finally, we would be able to boot the system to PXE boot to install the required Operating system.

We automated this process using `Ansible RAW` module and the `Local iDrac RACADM` commndline tool. we also use HTTP repository share from which we will update the recommended updates and these updates were created in the form a catalog using Dell Repository Manager.
This will update Firmware/Device Drivers, change/modify iDrac/Bios Settings using Racadm command line tool on Dell PowerEdge 620 and 730xd models. We will connect to the local racadm which resides in iDrac and execute the required commands using the Ansible RAW module.    
### Prerequisites
1. An Existing DNS server should have updated with DNS records for all the iDrac hosts. 
2. Need a system/VM running with linux OS.
 ``` 
 CentOS Linux release 7.0.1406 (Core) or above available.
 ```
3. Python should be installed.
```
[ansible@test-vm ~]$ python --version
Python 2.7.5
```
4. [Ansible](http://docs.ansible.com/intro_installation.html) should be installed.
```
[ansible@test-vm ~]$ ansible --version
ansible 2.3.1.0
  config file = /home/ansible/ansible.cfg
  configured module search path = Default w/o overrides
  python version = 2.7.5 (default, Jun 17 2014, 18:11:42) [GCC 4.8.2 20140120 (Red Hat 4.8.2-16)]
```

5. Pip (Python package Index) should installed using `epel-release`. 
```
[ansible@test-vm ~]$ pip --version
pip 8.1.2 from /usr/lib/python2.7/site-packages (python 2.7)
```
6. Jinja2 should be installed using `pip install jija2`. 
```
[ansible@test-vm ~]$pip list
..
Jinja2 (2.7.2)
..
..
```
7. A `HTTP server` should be setup, running and should be accessible to iDrac hosts. This is the location where we host Dell FW updates as HTTP share from which iDrac hosts would be able to download the required Firmware/device drivers.
 
### Installing
Download/Clone the Repository and place the directory structure as shown above. We are following the directory structure that is mentioned in anisble best practices. Please follow the below links for more understanding on how ansible directory structure works:
1. http://docs.ansible.com/ansible/latest/playbooks_best_practices.html#directory-layout
2. https://leucos.github.io/ansible-files-layout


### Inventory Setup

`inventory/hosts` : We will configure the entire stack by listing our hosts in the 'hosts' inventory file, usually grouped by their purpose:
* This file holds all the variables for individual hosts, for host-groups. **Make sure you change the values of the variables accordingly in this file**
```yaml
[all:children]
idrac_hosts

#[r620_servers]
#10.231.9.46 idrac_racname=r6c03-bmc model=620

#[r730_servers]
#10.231.9.40 idrac_racname=r6c12-bmc model=730

[idrac_hosts]
#idrachost1 ansible_host=<IP ADDRESS of idrachost1> idrac_racname=<idrachost1 name from DNS> model=<idrachost1 SERVER MODEL>
#idrachost2 ansible_host=<IP ADDRESS of idrachost2> idrac_racname=<idrachost2 name from DNS> model=<idrachost2 SERVER MODEL>
r6c03-bmc ansible_host=10.231.9.46 idrac_racname=r6c03-bmc model=620
r6c11-bmc ansible_host=10.231.9.39 idrac_racname=r6c11-bmc model=730

#[r620_servers:vars]
#[r730_servers:vars]
[idrac_hosts:vars]
ansible_ssh_pass=<PASSWORD to ACCESS iDrac>
ansible_ssh_user=root # will always be root for iDrac
idrac_dns1=<PRIMARY DNS IP ADDRESS>
idrac_dns2=<SECONDARY DNS IP ADDRESS>
idrac_gateway=<GATEWAY of The iDRAC SUBNET>
idrac_netmask=<NETMASK of The iDRAC SUBNET>
idrac_domainname=<DOMAIN NAME>
raid_force=<true/false> #true will enforce the raid reset
catalog_http_share=<PATH of HTTP SERVER SHARE for DELL CATALOG> # 10.231.7.155/DellRepo/072017
mgmt_vlanid=104
mgmt_gateway=10.7.20.1
mgmt_netmask=255.255.255.0


```

```yaml
[all:children] => main parent group
idrac_hosts     --- 
#r620_servers      | --> host groups
#r730_servers   ---

#[r620_servers]                                               
#10.231.9.46 idrac_racname=r6c03-bmc model=620  | --> host in-line variables that can be used in playbooks (applicable to only this host).

[idrac_hosts]
r6c11-bmc ansible_host=10.231.9.39 idrac_racname=r6c11-bmc model=730 | --> host in-line variables that can be used in playbooks (applicable to only this host).

#[r620_servers:vars]     | --> variables that are defined for the host-group (applicable to all the hosts in this group). 
[idrac_hosts:vars]                   
ansible_ssh_pass=******
ansible_ssh_user=root
idrac_dns1=10.231.0.101
...
...
...
```
### Run the Playbook
*Once we are done with defining all the required variables we are now ready to execute our playbook on the hosts by running the following command from the terminal*

```
ansible-playbook -i inventory/hosts playbooks/deploy_server_roles.yml
```
* **deploy_server_roles.yml** `=>` which holds the following five roles that gets executed accordingly. If you don't want a role to be executed then simply comment out that line in the YAML file.

```yaml
  roles:
#    - role: ../roles/Firmware_Updates
    - role: ../roles/Raid_R620
      when: '(raid_force | bool) and (model is defined and model == 620)'
#    - role: ../roles/Raid_R730
#      when: '(raid_force | bool) and (model is defined and model == 730)'
    - role: ../roles/iDrac_Settings
    - role: ../roles/iDrac_BIOS_Settings

```
* The roles are :

Role                            | Description                                                                              
--------------------------------|----------------------------------------------------------
1 ../roles/Firmware_Updates     | Compare against the HTTP catalog and run any updates if available.
2 ../roles/Raid_R620            | Reset the RAID forcefully and re-create RAID-1 across first two disks. Runs only when `raid_force` is set to `true` in `inventory/hosts`. Uses conditional : ` when: '(raid_force l bool) and (model is defined and model == 620)' ` .
3 ../roles/Raid_R730   | Reset the RAID forcefully and re-create RAID-1 across first two disks. Convert any SSDs into Non-RAID. Runs only when `raid_force` is set to `true` in `inventory/hosts`. Uses conditional : ` when: '(raid_force 1 bool) and (model is defined and model == 730)'] `.
4 ../roles/iDrac_Settings       | Modify/Update the iDrac settings using the variable values mentioned in `invenroty/hosts`.
5 ../roles/iDrac_BIOS_Settings  | Modify/Update the BIOS settings using the variable values mentioned in `invenroty/hosts`.

 ```yaml
+---------------------------------+--------------------------------------------------------------------------------------------+
| Role                            | Description                                                                                |
+=================================+============================================================================================+
| 1. ../roles/Firmware_Updates    | Compare against the HTTP catalog and run any updates if available.                         |
+---------------------------------+--------------------------------------------------------------------------------------------+
| 2. ../roles/Raid_R620           | Reset the RAID forcefully and re-create RAID-1 across first two disks.                     | 
|                                 | Runs only when `raid_force` is set to true in `inventory/hosts`                            |
|                                 | [when: '(raid_force | bool) and (model is defined and model == 620)'].                     |          
+---------------------------------+--------------------------------------------------------------------------------------------+
| 3. ../roles/Raid_R730           | Reset the RAID forcefully and re-create RAID-1 across first two disks.                     |
|                                 | convert any SSDs into Non-RAID. Runs only when `raid_force` is set to true in              |
|                                 | `inventory/hosts`. [when: '(raid_force 1 bool) and (model is defined and model == 730)'].  | 
+---------------------------------+--------------------------------------------------------------------------------------------+
| 4. ../roles/iDrac_Settings      | Modify/Update the iDrac settings using the variable values mentioned in invenroty/hosts.   |
+---------------------------------+--------------------------------------------------------------------------------------------+
| 5. ../roles/iDrac_BIOS_Settings | Modify/Update the BIOS settings using the variable values mentioned in invenroty/hosts.    |
+---------------------------------+--------------------------------------------------------------------------------------------+
```
## Example Scenarios:

#### Per Rack basis - Rack having 620s and 730s:

Let's say we have a rack containing both R620 and R730 models and we would like to deploy them in one go then we can do as:

* Modify the `inventory/hosts` file that includes the R620 and R730 Hosts.
```yaml
[all:children]
idrac_hosts

[idrac_hosts]
r6c01-bmc 10.231.9.21 idrac_racname=r6c01-bmc model=620 # model parameter is necessary, If it is blank/not equals to 620 then RAID role won't be running on that host (coupled with raid_force variable host group vars below).
r6c02-bmc ansible_host=10.231.9.22 idrac_racname=r6c02-bmc model=620
r6c03-bmc ansible_host=10.231.9.23 idrac_racname=r6c03-bmc model=730 # model parameter is necessary, If it is blank/not equals to 730 then RAID role won't be running on that host (couple with raid_force variable in host group vars below).
r6c04-bmc 10.231.9.23 idrac_racname=r6c03-bmc model=730

[idrac_hosts:vars]
ansible_ssh_pass=******
ansible_ssh_user=root
idrac_dns1=10.231.0.101
idrac_dns2=10.231.0.103
idrac_gateway=10.231.9.1
idrac_netmask=255.255.255.0
idrac_domainname=encore-oam.com
raid_force=false # set to TRUE if you want to run RAID role on the host (coupled with model variable above).
catalog_http_share=10.231.7.155/DellRepo/072017 # HTTP server where we host the Dell Firmware catalog Repository.
mgmt_vlanid=104
mgmt_gateway=10.7.20.1
mgmt_netmask=255.255.255.0
```
* Modify the `playbooks/deploy_server_roles.yml` file so that it runs [asynchronously](http://docs.ansible.com/ansible/latest/playbooks_strategies.html), this way the we can eliminate host dependency on each other. 
```yaml
- name: Deploy Server Full Automation
  hosts: idrac_hosts  # should match the host group that we set in the inventory/hosts files
  strategy: free      # runs in asynchronous fashion
  user: root
  become: yes
  gather_facts: false
  vars:
    target_array:
      - { target: 'BIOS.SysProfileSettings.SysProfile', job_target: 'Bios.Setup.1-1', target_set: 'SysProfile', value: 'PerfOptimized' }
      - { target: 'bios.biosbootsettings.BootMode', job_target: 'Bios.Setup.1-1', target_set: 'BootMode', value: 'Bios' }
      - { target: 'nic.nicconfig.1.LegacyBootProto', job_target: 'NIC.Integrated.1-1-1', target_set: 'LegacyBootProto', value: 'NONE' }
      - { target: 'nic.nicconfig.3.LegacyBootProto', job_target: 'NIC.Integrated.1-3-1', target_set: 'LegacyBootProto', value: 'PXE' }
  roles:
    - role: ../roles/Firmware_Updates
    - role: ../roles/Raid_R620
      when: '(raid_force | bool) and (model is defined and model == 620)'
    - role: ../roles/Raid_R730
      when: '(raid_force | bool) and (model is defined and model == 730)'
    - role: ../roles/iDrac_Settings
    - role: ../roles/iDrac_BIOS_Settings

```

#### Per Rack basis - either R620s or R730s.

If we want to run only on the R620 (or) only on R730 models, then we can do the following: 

* Modify the `inventory/hosts` file that includes the R620 / R730 Hosts.
```yaml
[all:children]
R620_hosts

[R620_hosts]
r6c01-bmc ansible_host=10.231.9.21 idrac_racname=r6c01-bmc model=620 # model parameter is necessary, If it is blank/not equals to 620 then RAID role won't be running on that host (coupled with raid_force variable host group vars below).
r6c02-bmc ansible_host=10.231.9.22 idrac_racname=r6c02-bmc model=620

[R620_hosts:vars]
ansible_ssh_pass=******
ansible_ssh_user=root
idrac_dns1=10.231.0.101
idrac_dns2=10.231.0.103
idrac_gateway=10.231.9.1
idrac_netmask=255.255.255.0
idrac_domainname=encore-oam.com
raid_force=false # set to TRUE if you want to run RAID role on the host (coupled with model variable above).
catalog_http_share=10.231.7.155/DellRepo/072017 # HTTP server where we host the Dell Firmware catalog Repository.
mgmt_vlanid=104
mgmt_gateway=10.7.20.1
mgmt_netmask=255.255.255.0
```
* Modify the `playbooks/deploy_server_roles.yml` file so that it runs in a linear and synchronous way. During this playbook LINEAR run, ansible waits for each task to be completed on all the hosts before it move on to execute the next task.

```yaml
- name: Deploy Server Full Automation
  hosts: R620_hosts  # should match the host group that we set in the inventory/hosts files
  #strategy: free      # so that all tasks gets executed in synchronous across all the 620s.
  user: root
  become: yes
  gather_facts: false
  vars:
    target_array:
      - { target: 'BIOS.SysProfileSettings.SysProfile', job_target: 'Bios.Setup.1-1', target_set: 'SysProfile', value: 'PerfOptimized' }
      - { target: 'bios.biosbootsettings.BootMode', job_target: 'Bios.Setup.1-1', target_set: 'BootMode', value: 'Bios' }
      - { target: 'nic.nicconfig.1.LegacyBootProto', job_target: 'NIC.Integrated.1-1-1', target_set: 'LegacyBootProto', value: 'NONE' }
      - { target: 'nic.nicconfig.3.LegacyBootProto', job_target: 'NIC.Integrated.1-3-1', target_set: 'LegacyBootProto', value: 'PXE' }
  roles:
    - role: ../roles/Firmware_Updates
    - role: ../roles/Raid_R620
      when: '(raid_force | bool) and (model is defined and model == 620)'
      
# The below role will skip anyway as we don't have a model 730 defined in out inventory/hosts file. However, if you uncomment it then this will simply not flood out terminal with the skipped output.

#    - role: ../roles/Raid_R730
#     when: '(raid_force | bool) and (model is defined and model == 730)'
    - role: ../roles/iDrac_Settings
    - role: ../roles/iDrac_BIOS_Settings

```

#### Per Role Basis - Run only specific roles

Sometimes, it would be useful to run only FW updates / or  only modify/update iDrac settings. We can achieve that as following: 

* Modify the `playbooks/deploy_server_roles.yml` file, and comment out the roles which we don't want to be executed. The following playbook only runs with `roles/Firmware_Updates` and `roles/iDrac_Settings` on the hosts included in the `inventory/hosts`.

```yaml
- name: Deploy Server Full Automation
  hosts: R620_hosts   # should match the host group that we set in the inventory/hosts files
  strategy: free      #  runs in asynchronous fashion
  user: root
  become: yes
  gather_facts: false
  vars:
    target_array:
      - { target: 'BIOS.SysProfileSettings.SysProfile', job_target: 'Bios.Setup.1-1', target_set: 'SysProfile', value: 'PerfOptimized' }
      - { target: 'bios.biosbootsettings.BootMode', job_target: 'Bios.Setup.1-1', target_set: 'BootMode', value: 'Bios' }
      - { target: 'nic.nicconfig.1.LegacyBootProto', job_target: 'NIC.Integrated.1-1-1', target_set: 'LegacyBootProto', value: 'NONE' }
      - { target: 'nic.nicconfig.3.LegacyBootProto', job_target: 'NIC.Integrated.1-3-1', target_set: 'LegacyBootProto', value: 'PXE' }
  roles:
    - role: ../roles/Firmware_Updates
#    - role: ../roles/Raid_R620
#      when: '(raid_force | bool) and (model is defined and model == 620)'
#    - role: ../roles/Raid_R730
#     when: '(raid_force | bool) and (model is defined and model == 730)'
    - role: ../roles/iDrac_Settings
#    - role: ../roles/iDrac_BIOS_Settings

```
