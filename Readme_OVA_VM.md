### Deploy a CentOS VM from the OVA template in vCenter: 

> The OVA template that we provided will have all the required and necessary components in place. Please feel free to install 
any package that is not available on the template. We created a user called `ansible` and placed the required directory structure 
that is required for ansible program under user `ansible` home directory `/home/ansible/`. 

#### Setup the VM in vCenter using OVF Template:
**If you are using stand-alone vSphere Client application, we can deploy VM as follows:**
* Open vCenter Clieent Application, Enter the credntials and into it.
* Click on `File -> Deploy OVF Template`. In the Deploy OVF Template dilaog, choose the appropriate `Source file, Name of the VM and 
Host/Cluster/Resource pool` on which the VM should be deployed.
* Once the template is deployed, `Go to VMs and Templates -> to the appropriate location of the VM 
(selected in the above step)`, `Right click on the VM -> Edit Settings`. This is to attach the correct NIC interface 
so that we can assign an IP address to access it. 
* Click on the `Network Adapter1 -> On the Right side panel`; Under the `Network Connection`, Please select 
an appropriate host network that exists in your network.
* Once you assign the correct Network Adapter, `Right click on the VM -> Power -> Power On`.
* Once the VM is up and Running, Open Console (`Right click on VM -> Open Console`) and log in using the credentials. 
`user: root  pass: ChangeMe1!`
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
* If hostname needs to be changed, change it using : `hostnamectl set-hostname “USE YOUR HOSTNAME PATTERN”`. Please edit the hosts file using `vi /etc/hosts` and enter the details:
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
<IP ADDRESS> <HOSTNAME.DOMAIN> <HOSTNAME>
``` 
* On the Terminal, type: `systemctl restart network` to restart the network.
* `Logout` and again log into the system.

* Check whether, `hostname` got changed to the name that you set.
* Check whether, VM can access internet, `ping google.com`
* Check whether, `VM can be accessed from your local machine, ping it and check whether you can access IP/Hostname`. If you can’t check the DNS or troubleshoot for any further issues.
* **Change user : `ansible` password using `passwd ansible`.** Current password for user `ansible` is `ChangeMe1!`.

### Assuming that everything went smooth and we can access the VM from outside the vCenter using Putty connection. Please do the following for the Host-Prep using ansible and racadm.
