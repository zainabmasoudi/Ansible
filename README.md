# Ansible Infrastructure Automation
## Description
This project demmonstrates the automation of administrative tasks using Ansible within a heterogeneous environment. We will configure three virtual machines (VMS) with different operating systems:
1. Ansible Manager (Ubuntu 20.04)
2. AnsibleServer1 (Ubuntu Server)
3. AnsibleServer2 (Debian)

In this setup, the Ansible Manager will automate various tasks such as:
- Enabling passwordless SSH access to the managed nodes
- Installing Apache and MariaDB on the remote servers
- Updating and upgrading the remote systems
- Monitoring the system states from the control node
- The project also introduces the concept of YAML configuration files, which are used to define Ansible playbooks.

## Objective
The primary goal of this document is to automate administrative tasks using Ansible. Managing the remote servers and automating the process of updating, upgrading, and installation of services like (Apache and MariaDB). Moreover, we will learn the basic concept of the YMAL configuration language and its role in defining playbooks.

### Configuration

#### Ansible Manager Configuration
1. Set Static IP and Hostname: Edit the ``/etc/netplan/50-cloud-init.yaml`` to set a static IP.
```
network:
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.239.110/24
      routes:
        - to: default
          via: 192.168.239.2
      nameservers:
        addresses:
          - 8.8.8.8
          - 9.9.9.9
  version: 2
  ```
  Apply the changes:
   `` netplan apply``

  2. Update the HostName:
  Edit ``/etc/hostname`` and change the system name to __AnsibleManager__. Then, either reboot or restart the system:
``systemctl restart system-logind``
  3. Edit /etc/hosts:
```
127.0.1.1          AnsibleManager

192.168.239.110    AnsibleManager
192.168.239.111    AnsibleServer1
192.168.239.112    AnsibleServer2

```
#### AnsibleServer1 Configuration
For both servers, follow similar steps:
1. Set Static IP and Hostname:
For AnsibleServer1, edit ``/etc/netplan/50-cloud-init.yaml`` to set the static IP and apply with netplan apply.
2. Update ``/etc/hosts``:
For both servers, ensure that ``/etc/hosts`` includes the IP addresses and names of all servers in the network.

```
network:
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.239.111/24
      routes:
        - to: default
          via: 192.168.239.2
      nameservers:
        addresses:
          - 8.8.8.8
          - 9.9.9.9
  version: 2
  ```
  Once we add the IP address, we will run the ``netpaln apply`` command to apply the changes.

3. Nano ```/etc/hosts``` file and add the servers with names and IPs
```
127.0.1.1          AnsibleServer1

192.168.239.110    AnsibleManager
192.168.239.111    AnsibleServer1
192.168.239.112    AnsibleServer2

```

### AnsibleServer2 Configuration
For AnsibleServer2, 
1. configure ``/etc/network/interfaces.d/ens33.conf`` and use the static IP configuration.

```
iface ens33 inet static
      address 192.168.239.112
      netmask 255.255.255.0
      gateway 192.168.239.2
dns-nameservers 8.8.8.8 9.9.9.9

```
Nano the ``/etc/network/interface`` file and add `` allow-hotplug ens33 `` to allow your interface.
Apply the changes:
   ``systemctl restart NetworkManager``
     
2. Edit /etc/hosts file 
```
127.0.1.1          AnsibleServer2

192.168.239.110    AnsibleManager
192.168.239.111    AnsibleServer1
192.168.239.112    AnsibleServer2

```
### SSH Key Setup
To enable passwordless SSH between Ansible Manager and the remote servers, follow these steps:
1. Generate SSH Key:
On the Ansible Manager, run the following command to generate an SSH key:
```
ssh-keygen -t rsa -b 4096 -f ~/.ssh/local.rsa -C user@gamil.com
```
2. Copy the SSH Key to Remote Servers:
Use ssh-copy-id to copy the SSH key to each of the managed servers:
```
ssh-copy-id -i ~/.ssh/local.rsa.pub user@ansibleserver1
ssh-copy-id -i ~/.ssh/local.rsa.pub user@ansibleserver1
```

### Installing Ansible
To install Ansible on the Ansible Manager node:
```
sudo apt-add-repository ppa:ansible/ansible
sudo apt update
sudo apt install ansible -y
```
Update the Ansible inventory ``/etc/ansible/hosts`` with the following content:
```
[ubuntu24_04]
AnsibleServer1
AnsibleServer2

[webservers]
AnsibleServer1

[mariadbservers]
AnsibleServer2
```
Verify the connection with:

```
ansible all -m ping -u user

ansible all -a "df -h" -u user
```
### Ansible Playbooks
Update and Upgrade Playbook ``update.yml``
Create a playbook to update and upgrade remote systems:
```
# Task name
- name: Updates
# Target is both AnsibleServer1 and AnsibleServer2
  hosts: all
# 
  gather_facts: yes
# do with sudo privilaged 
  become: yes

  tasks:
# Define task's name
    - name: Perform a dist-upgrade.
      ansible.builtin.apt:
# dist option tells apt to upgrade the remote system
        upgrade: dist
# update the remote server's cache
        update_cache: yes
# Check the condition if the remote systems require the reboot or not after updating & Upgrading
    - name: Check if a reboot is required.
      ansible.builtin.stat:
        path: /var/run/reboot-required
# The result of this task will be saved in register file
      register: reboot_required_file
# Checks the if statement and if the file reboot is exist it will reboot the server
    - name: Reboot the server (if required).
      ansible.builtin.reboot:
      when: reboot_required_file.stat.exists == true

    - name: Remove dependencies that are no longer required.
# apt module is used to manage the packages
      ansible.builtin.apt:
# Remove dependencies
        autoremove: yes
```
Run the playbook:
```ansible-playbook /etc/ansible/update.yml --ask-become-pass```
### Web Server Installation Playbook ``Install Apache.yml``
To install Apache:

```
- name: Install Apache
  hosts: webservers
  become: true
  
  tasks:
  - name: Install Apache
    apt:
      name: apache2
# Ensures that Apache is installed
      state: present
  
  - name: Start Apache
    service:
      name: apache2
# Ensures that Apache is running if it's notو ansible will restart it 
      state: started
# Ensures that Apache starts automatically at system boot time
      enabled: yes

```
Run the playbook:
```ansible-playbook /etc/ansible/InstallApache.yml --ask-become-pass```

### MariaDB Installation Playbook (Install MariaDB.yml)
To install MariaDB:
```
- name: Install MariaDB
  hosts: mariadbservers
# Run this program with sudo privilege
  become: true
  
  tasks:
  - name: Install MariaDB
    apt:
      name: mariadb-server
# check if the app is installed or not if it is not it will install it.
      state: present
  
  - name: Start MariaDB
    service:
      name: mariadb
# Ensure that the mariadb is running
      state: started
      enabled: yes
```
Run the playbook:
```ansible-playbook /etc/ansible/InstallMariaDB.yml --ask-become-pass```
### Conclusion
By completing this project, we learned how to use Ansible to manage remote systems and automate administrative tasks such as updating, upgrading, and installing services like Apache and MariaDB. Additionally, we gained experience working with YAML configurations and Ansible playbooks, which are key components of Infrastructure as Code (IaC).

