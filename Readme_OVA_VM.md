## Deploy a CentOS VM from the OVA template in vCenter: 

> The OVA template that we provided will have all the required and necessary packages/components in place. Please feel free to install 
any package that is not available on the template. We created a user called `ansible` and placed the required directory structure 
that is required for ansible program under user `ansible` home directory `/home/ansible/`. 

### Setup the VM in vCenter using OVF Template:
**If you are using stand-alone vSphere Client application, we can deploy VM as follows:**
* Open vCenter Clieent Application, Enter the credntials and into it.
* Click on `File -> Deploy OVF Template`. In the Deploy OVF Template dilaog, choose the appropriate `Source file, Name of the VM and 
Host/Cluster/Resource pool` on which the VM should be deployed.
* Once the template is deployed, 
  * `Go to VMs and Templates -> to the appropriate location of the VM (selected in the above step)`, 
  * `Right click on the VM -> Edit Settings`. This is to attach the correct NIC interface 
so that we can assign an IP address to access it. 
  * Click on the `Network Adapter1 -> On the Right side panel`; Under the `Network Connection`, Please select 
an appropriate host network that exists in your network.
* Once you assign the correct Network Adapter, `Right click on the VM -> Power -> Power On`.
* Once the VM is up and Running: 
  * Open Console (`Right click on VM -> Open Console`) and 
  * Log in using the credentials. `user: root  pass: ChangeMe1!`
**`[[ Please change the password immediately. ]]`**
* On terminal, Enter `ip addr` and note down the Ethernet device. Should see something like this:
```
[root@vcnms-lab-linux ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eno16780032: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 00:50:56:9d:3f:4f brd ff:ff:ff:ff:ff:ff
    inet 10.7.20.144/24 brd 10.7.20.255 scope global eno16780032
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe9d:3f4f/64 scope link
       valid_lft forever preferred_lft forever
```
* Use: `vi /etc/sysconfig/network-scripts/ifcfg-eno16780032` and change the values accordingly and then save the file `:wq!`.
```
TYPE=Ethernet
BOOTPROTO=none
IPADDR=<IP ADDRESS THAT IP AVAILABLE>
NETMASK=<NETMASK FOR THE ABOVE IP>
GATEWAY=<GAEWAY FOR THE ABOVE IP/NETMASK>
NAME=eno16780032
DEVICE=eno16780032
ONBOOT=yes
```
```
If the VM needs any additional network adapters, then 
please do add them and then change the values appropriately at the OS level. 
```

* On the Terminal, type: `systemctl restart network`. After a brief moment, the network should be up and running. If you see any error messages, please troubleshoot accordingly. If you want, use `service network restart` (doesn't matter).
* Enter the DNS server details using: `vi /etc/resolv.conf` . save and exit `:wq!`.
* If hostname needs to be changed, 
  * Change it using : `hostnamectl set-hostname “USE YOUR HOSTNAME PATTERN”`. 
  * For FQDN: Please edit the hosts file using `vi /etc/hosts` and enter the details:
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
<IP ADDRESS> <HOSTNAME.DOMAIN> <HOSTNAME>
``` 
* On the Terminal, type: `systemctl restart network` to restart the network.
* `Logout` and again log into the system.

### Please Check the Following items:

* Check whether, `hostname` got changed to the name that you set. Hostname FQDN using `hostname -f`.
* Check whether, VM can access internet, `ping google.com`
* Check whether, `VM can be accessed from your local machine, ping it and check whether you can access IP/Hostname`. If you can’t check the DNS or troubleshoot for any further issues.
* `HTTP` service should be up and running:
   * Check the status using: `systemctl status httpd`, **if it's not running**, then 
   * using: `systemctl start httpd` to start the service.
   * Access http://<>IPADDRESS of the VM> on any web browser from your local machine. You should see ATT Labs welcome page :relaxed:.
* `NFS server` service should be up and running:
   * Check the status using: `systemctl status nfs-server`, **if it's not running**, then 
   * using: `systemctl start nfs-server` to start the service.   
* **Change user : `ansible` password  by typing `passwd ansible`.** Current password for user `ansible` is `ChangeMe1!`.**

### Assuming that everything went smooth and we can access the VM from outside the vCenter using Putty connection. Please do the following for the Host-Prep using ansible and racadm.

**Please Download all these files and places them under the `/home/ansible` directory with the same exact directory structure.**
* Once you Logged into the user as `root`, check the `/etc/sudoers` file. If anything needs to be changed, please do so.
* Switch user to ansible using `su - ansible`. Currently user `ansible` is set as `ansible ALL=(ALL) NOPASSWD: ALL`. Change it, if you want to be prompted for each time when `sudo` is used.
* Make sure we are in the user `ansible` home directory. `/home/ansible`
* The first file we are going to edit is `inventory/hosts_prod`:
  * `vi /home/ansible/inventory/hosts_prod` : This is the file which holds all the inventory information of the servers in all the racks. It is divided into 620, 730 server groups and further divided into sub groups based on the racks.
  * Please enter the information in the following order : `<DNS NAME FOR IDRAC> ansible_host=<IP ADDRESS of idrachost1> idrac_racname=<idrachost1 name from DNS> model=<idrachost1 SERVER MODEL>`.                          
* Replace all the lines in the `/home/ansible/inventory/hosts_prod` which contains ` catalog_http_share=<IP ADDRESS of THE VM>/Dell_Repo/R620_R730_072017 `
  * Replace `<IP ADDRESS of THE VM>` with IP ADDRESS assigned with the VM (HTTP server should be up and running on the same VM).
* Please also the value of the `ansible_ssh_password=<custom password>` with the iDrac password . `default password is : calvin`. 

**Once we are done with updating the `../inventory/hosts_prod/`, we are ready to execute the playbooks on the hosts. [Make sure you can ping them from the deployed VM ].**

## How to Run the Playbooks

* **To Run on RACK3 R620 servers ::** 

```console
[Assuming that you are in the user ansible home directory]

[ansible@vcnms-lab-linux ~]$ ansible-playbook -i inventory/hosts_prod playbooks/r620_deploy_server_role.yml -l rack3_r620
```

```console
-i  -> for the hosts inventory information file.
-l -> to limit the playbook to run only on the mentioned host group.

After you run the playbook on hostgroup “rack3_r620”, you will see the Play being 
run by ansible along with the tasks and once is finished it will list the play recap 
status about the hosts. (ok means success, failed means issue in running a task on that 
hosts and unreachable means ansible unable to SSH into that host.) 
[Please see below examples for more info.]
```
```yaml
# [To RUN on RACK4 R620 servers]
ansible-playbook -i inventory/hosts_prod playbooks/r620_deploy_server_role.yml -l rack4_r620

# [To RUN on RACK5 R620 servers]
ansible-playbook -i inventory/hosts_prod playbooks/r620_deploy_server_role.yml -l rack5_r620

# [To RUN on RACK6 R620 servers]
ansible-playbook -i inventory/hosts_prod playbooks/r620_deploy_server_role.yml -l rack6_r620

# [To RUN on RACK7 R620 servers]
ansible-playbook -i inventory/hosts_prod playbooks/r620_deploy_server_role.yml -l rack7_r620

```

* **To Run on RACK4 R730 servers ::**

```console
[Assuming that you are in the user ansible home directory]

[ansible@vcnms-lab-linux ~]$ ansible-playbook -i inventory/hosts_prod playbooks/r730_deploy_server_role.yml -l rack4_r730
```
```yaml
# [To RUN on RACK5 R730 servers]
ansible-playbook -i inventory/hosts_prod playbooks/r730_deploy_server_role.yml -l rack5_r730

# [To RUN on RACK6 R730 servers]
ansible-playbook -i inventory/hosts_prod playbooks/r730_deploy_server_role.yml -l rack6_r730

```
* **To Run on RACK1 & RACK2 Management servers ::** [We don't have to limit to any group here because, we defined rack1 and rack2 as a `mgmt_servers` group which directly holds all the required host information. no sub-groups defined in this case]

```console
[Assuming that you are in the user ansible home directory]

[ansible@vcnms-lab-linux ~]$ ansible-playbook -i inventory/hosts_prod playbooks/mgmt_r620_deploy_server_role.yml 
``` 

### Example Runs of the Playbooks

**Below is a sample run of a playbook (which gets the IP address assigned to iDrac using RACADM) on the host group ` rack12_r620`**

```yaml
[ansible@vcnms-lab-linux ~]$ ansible-playbook -i inventory/hosts1 playbooks/racadm_getsysinfo.yml -l rack12_r620

PLAY [get sys info.] ****************************************************************************************************************************************************************************************************************

TASK [get sys info] *****************************************************************************************************************************************************************************************************************
changed: [r60212c5-bmc]
changed: [r60212c4-bmc]
changed: [r60212c1-bmc]
changed: [r60212c2-bmc]
changed: [r60212c3-bmc]

TASK [debug] ************************************************************************************************************************************************************************************************************************
ok: [r60212c2-bmc] => {
    "sysinfo.stdout_lines": [
        "[Key=iDRAC.Embedded.1#IPv4.1]",
        "Address=10.231.9.30"
    ]
}
ok: [r60212c1-bmc] => {
    "sysinfo.stdout_lines": [
        "[Key=iDRAC.Embedded.1#IPv4.1]",
        "Address=10.231.9.29"
    ]
}
ok: [r60212c3-bmc] => {
    "sysinfo.stdout_lines": [
        "[Key=iDRAC.Embedded.1#IPv4.1]",
        "Address=10.231.9.31"
    ]
}
ok: [r60212c4-bmc] => {
    "sysinfo.stdout_lines": [
        "[Key=iDRAC.Embedded.1#IPv4.1]",
        "Address=10.231.9.32"
    ]
}
ok: [r60212c5-bmc] => {
    "sysinfo.stdout_lines": [
        "[Key=iDRAC.Embedded.1#IPv4.1]",
        "Address=10.231.9.33"
    ]
}

PLAY RECAP **************************************************************************************************************************************************************************************************************************
r60212c1-bmc               : ok=2    changed=1    unreachable=0    failed=0
r60212c2-bmc               : ok=2    changed=1    unreachable=0    failed=0
r60212c3-bmc               : ok=2    changed=1    unreachable=0    failed=0
r60212c4-bmc               : ok=2    changed=1    unreachable=0    failed=0
r60212c5-bmc               : ok=2    changed=1    unreachable=0    failed=0
```

**Below is a sample run of a playbook (which gets the IP address assigned to iDrac using RACADM) on the host group ` rack10_r730`**
* The Password is wrong here (intentionally made playbook to fail).

```yaml
[ansible@vcnms-lab-linux ~]$ ansible-playbook -i inventory/hosts1 playbooks/racadm_getsysinfo.yml -l rack10_r730

PLAY [get sys info.] ****************************************************************************************************************************************************************************************************************

TASK [get sys info] *****************************************************************************************************************************************************************************************************************
fatal: [r60210c12-bmc]: FAILED! => {"changed": true, "failed": true, "rc": 5, "stderr": "Permission denied, please try again.\r\n", "stdout": "", "stdout_lines": []}
fatal: [r60210c13-bmc]: FAILED! => {"changed": true, "failed": true, "rc": 5, "stderr": "Permission denied, please try again.\r\n", "stdout": "", "stdout_lines": []}
fatal: [r60210c11-bmc]: FAILED! => {"changed": true, "failed": true, "rc": 5, "stderr": "Permission denied, please try again.\r\n", "stdout": "", "stdout_lines": []}
        to retry, use: --limit @/home/ansible/playbooks/racadm_getsysinfo.retry

PLAY RECAP **************************************************************************************************************************************************************************************************************************
r60210c11-bmc              : ok=0    changed=0    unreachable=0    failed=1
r60210c12-bmc              : ok=0    changed=0    unreachable=0    failed=1
r60210c13-bmc              : ok=0    changed=0    unreachable=0    failed=1

```
**Below is a sample run of a playbook (which gets the IP address assigned to iDrac using RACADM) on the host group ` mgmt_servers`**
* Some of them are `unreachable` and others were successfull in executing the task.
```yaml
[ansible@vcnms-lab-linux ~]$ ansible-playbook -i inventory/hosts1 playbooks/racadm_getsysinfo.yml

PLAY [get sys info.] ****************************************************************************************************************************************************************************************************************

TASK [get sys info] *****************************************************************************************************************************************************************************************************************
changed: [r02c01-bmc]
changed: [r01c03-bmc]
fatal: [r01c02-bmc]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: connect to host 10.231.9.150 port 22: Connection timed out\r\n", "unreachable": true}
fatal: [r01c01-bmc]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: connect to host 10.231.9.140 port 22: Connection timed out\r\n", "unreachable": true}
fatal: [r02c02-bmc]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: connect to host 10.231.9.180 port 22: Connection timed out\r\n", "unreachable": true}
fatal: [r02c03-bmc]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: connect to host 10.231.9.190 port 22: Connection timed out\r\n", "unreachable": true}

TASK [debug] ************************************************************************************************************************************************************************************************************************
ok: [r01c03-bmc] => {
    "sysinfo.stdout_lines": [
        "[Key=iDRAC.Embedded.1#IPv4.1]",
        "Address=10.231.9.16"
    ]
}
ok: [r02c01-bmc] => {
    "sysinfo.stdout_lines": [
        "[Key=iDRAC.Embedded.1#IPv4.1]",
        "Address=10.231.9.17"
    ]
}
        to retry, use: --limit @/home/ansible/playbooks/racadm_getsysinfo.retry

PLAY RECAP **************************************************************************************************************************************************************************************************************************
r01c01-bmc                : ok=0    changed=0    unreachable=1    failed=0
r01c02-bmc                : ok=0    changed=0    unreachable=1    failed=0
r01c03-bmc                : ok=2    changed=1    unreachable=0    failed=0
r02c01-bmc                : ok=2    changed=1    unreachable=0    failed=0
r02c02-bmc                : ok=0    changed=0    unreachable=1    failed=0
r02c03-bmc                : ok=0    changed=0    unreachable=1    failed=0

```

