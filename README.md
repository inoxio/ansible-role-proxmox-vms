Ansible Role: Create virtual machines with Proxmox and installs (unattended) an ubuntu distribution on it
=========

The inoxio.proxmox_vms role creates all vms that are listed within
your role execution (see example playbook) and installs an ubuntu version via preseeding on them. 

This role was developed based on morph027's pve-infra-poc role (https://gitlab.com/morph027/pve-infra-poc).

The main.yml in the 'default' directory contains default values for the vms. 

The main.yml of the 'tasks' directory contains all tasks to create virtual machines and to install the operating system. 
At first the vms will be created. After that the tasks for preparing the os installations will start. 
The newest netboot version of the given ubuntu version will be fetched and unzipped to get the initrd and kernel.
Then the preseed file and finish-installation-script will be moved into the initrd and packed afterwards.
After that the installation begins and the installation-arguments will be deleted 
(in order to start the installed os after the next reboot).
The vms will be rebooted after finishing all installations and a success message will be displayed when reboot
was successful (by polling the IP of the vm and searching for 'OpenSSH').

Requirements
------------

Running proxmox server.  

Dependencies
------------

geerlingguy.pip (used to install proxmoxer)

Role Variables
--------------

api_user, api_password, api_host: these are needed to log into the proxmox server.

* File: defaults/main.yml  
    * `Defaults`: these variables are the standard configuration for a vm.
        * `cpu`: Specify emulated CPU type. 'host' uses the same cpu type as the host system.
            * Default: `host`
        * `net`: A hash/dictionary of network interfaces for the VM. net='{"key":"value", "key":"value"}'.  
                 Keys allowed are - net[n] where 0 ≤ n ≤ N.  
                 Virtio was chosen to be the main platform for IO virtualization in KVM.  
                 The bridge parameter can be used to automatically add the interface to a bridge device.  
                 The Proxmox VE standard bridge is called 'vmbr0'.  
                 Option rate is used to limit traffic bandwidth from and to this interface.  
                 It is specified as floating point number, unit is 'Megabytes per second'.  
            * Default: `{"net0":"virtio,bridge=vmbr0"}`
        * `cores`: Specify number of cores per socket.
            * Default: `2`
        * `memory_size`: Memory size in MB for instance.
            * Default: `2048`
        * `balloon`: Specify the amount of RAM for the VM in MB. Using zero disables the balloon driver.  
                     The virtio balloon device allows KVM guests to reduce their memory size (thus relinquishing memory to the host) and  
                     to increase it back (thus taking memory from the host).
            * Default: `1024`
        * `scsihw`: Specifies the SCSI controller model.  
                    Choices: lsi, lsi53c810, virtio-scsi-pci, virtio-scsi-single, megasas, pvscsi  
                    virtio-scsi-pci: A virtio storage interface for efficient I/O that overcomes virtio-blk limitations and  
                    supports advanced SCSI hardware.
            * Default: `virtio-scsi-pci`
        * `virtio`: A hash/dictionary of volume used as VIRTIO hard disk. virtio='{"key":"value", "key":"value"}'.  
                    Keys allowed are - virto[n] where 0 ≤ n ≤ 15.  
                    Values allowed are - "storage:size,format=value".  
                    storage is the storage identifier where to create the disk.  
                    size is the size of the disk in GB.  
                    format is the drive's backing file's data format. qcow2|raw|subvol
            * Default: `{"virtio0":"local-lvm:10,cache=writeback,discard=on"}`
        * `ostype`: Specifies guest operating system. This is used to enable special optimization/features for specific operating systems.  
                    The l26 is Linux 2.6/3.X Kernel.  
                    Choices: other, wxp, w2k, w2k3, w2k8, wvista, win7, win8, l24, l26, solaris
            * Default: `l26`
        * `onboot`: Specifies whether a VM will be started during system bootup.
            * Default: `yes`
        * `locale`: Locale is a set of parameters that defines the user's language.
                    This is not necessary for proxmox_kvm but for the preseed-file. This value is commited to the deploy-args file.
        * `deploy_args_template`: Specifies the name of the template file which contains the deploy arguments for the vm. The arguments in it are attached
                                  to the args variable of a vm (see 'create vms' in tasks/main.yml) and are used when installing an os on a vm.
                                  This file is used when the vm definition does not contain a path for a custom file.
        * `preseed_template`: Specifies the name of the template preseed file, which will be taken if the definition of the vm in the playbook
                              has no preseed path. This file is used when the vm definition does not contain a path for a custom file.
        * `ubuntu_distribution`: Specifies the ubuntu distribution which will be installed on the vm.
    
* File: your playbook (see example playbook)  
    * `proxmox`: Contains login data for the proxmox server which should be encrypted. You need a file which contains  
                 the password for the ansible vault.
                 Encrypting is done by the following command on the terminal:  
                 `ansible-vault encrypt_string --vault-id <path_to_the_password_file> '<password>' --name '<variable_name>'`  
                 Example: `ansible-vault encrypt_string --vault-id ~/.ansible.secret 'some_password' --name 'api_password'`
        * `api_user`: Specifies the user-name for the proxmox login.
        * `api_password`: Specifies the password of a user for proxmox login.
        * `api_host`: Specifies the host name or ip of the proxmox server.
    * `vms`: lists all virtual machines to be installed. When some variables are not stated, the default values will be taken 
             (see defaults in defaults/main.yml). 
        * `<vm_name>`: Specifies the name of the vm.
            * `node`: Specifies the name of the node on the proxmox server. Under the node the vm will be installed.
            * `ubuntu_distribution`: Specifies the ubuntu distribution which will be installed on the vm. See deployments in defaults/main.yml.
            * `locale`: Locale is a set of parameters that defines the user's language.
            * `root_password`: Specifies the root password of the vm.
            * `memory_size`: Specifies the size of memory in MB for the vm.
            * `virtio`: Specifies the the hard-disk and its size to be used by the vm. See default/main.yml.
            * `network`:
                * `ip`: Specifies the ip address of the vm.
                * `netmask`: Specifies the netmask to be used by the vm.
                * `gateway`: Specifies the gateway ip to be used by the vm.
                * `nameserver`: Specifies the nameserver ips to be used by the vm.
                * `domainname`: Specifies the domain name of the vm. 
            * `additional_packages`: (Optional/No Default) Contains a list of additional packages that will be installed.
            * `scripts`: (Optional/No Default) Scripts is a list of files whose content will be inserted into the finish-installation file.
                        E.g. copying ssh keys from aws.
        
Example playbook
----------------

```
- hosts: <host_name> # see hosts file
  remote_user: root

  roles:
    - role: inoxio.proxmox_vms
      proxmox:
        api_user: <encrypted user>
        api_password: <encrypted password>
        api_host: <encrypted host>
      vms:
        <vm1_name>:
          node: <node_name>
          ubuntu_distribution: <distribution_name>
          locale: en_US
          root_password:  <encrypted root password>
          memory_size: <ram_size_in_MB>
          virtio: '{"virtio0":"local-lvm:<disk_size_in_GB>,cache=writeback,discard=on"}'
          network:
            ip: <ip>
            netmask: <netmask>
            gateway: <gateway_ip>
            nameserver: <nameserver1> <nameserver2>
            domainname: <domainname>
          additional_packages:
             - curl
             - gnupg
          scripts:
            - files/scripts/my_script.sh
            
        <vm2_name>:
          node: <node_name>
          root_password: <encrypted root password>
          ubuntu_distribution: 'xenial'
          memory_size: <ram_size_in_MB>
          virtio: '{"virtio0":"local-lvm:<disk_size_in_GB>,cache=writeback,discard=on"}'
          network:
            ip: <ip>
            netmask: <netmask>
            gateway: <gateway_ip>
            nameserver: <nameserver1> <nameserver2>
            domainname: <domainname>
            
```


Add new ubuntu distribution
------------

To add a new ubuntu distribution you have to:
* You can add the deploy-args file in templates directory. E.g. deploy-args-cosmic.j2. But this is optional.
The role tries to find a deploy-args file named like the given distribution name and if it can't find it the default
preseed file will be taken.
* You can add preseed-file in files directory. E.g. ubuntu-cosmic.seed. But this is optional.
The role tries to find a preseed file named like the given distribution name and if it can't find it the default
preseed file will be taken.

Deploy Arguments
------------

The deploy-args-file delivers necessary settings for an unattended ubuntu installation.
These settings could be delivered by the preseed-file itself, but since the deploy-args file is a jinja2 file (see 
jinja2 documentation: http://jinja.pocoo.org/docs/2.10/) you can code which settings should be used for given
parameters. You can also include other files. And use ansible variables to make the settings dependending on the vm
definition. E.g. locale={{ item.value.locale }}.UTF-8. If the local is en_US, jinja2 replaces it to 
locale=en_US.UTF-8

Preseed File
------------

The preseed-file contains settings for a unattended installation of an ubuntu distribution. E.g. information how the disk
should be partitioned, which user should be created and so on. It contains a section for late-commands. These are
commands that will be executed when the installation is done and will be executed in the installed os itself. The
late-commands can be delivered in a shell-script.

Finish-installation
------------

The finish-installation-template will be executed when the os installation is finished. It can be used for example 
to install packages or transfer ssh-keys. The file is a jinja2 template file and will add additional packages, which
are stated in the vm definition, and it adds scripts, which can also be stated in the vm definition.
