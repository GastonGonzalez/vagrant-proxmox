# Vagrant Proxmox Provider

This is a [Vagrant](http://www.vagrantup.com) plugin that adds a [Proxmox](http://proxmox.com/) provider to Vagrant, 
allowing Vagrant to manage and provision Proxmox virtual machines.


## About this Fork

The primary motivation for this fork was to support the author's goals for managing OpenVZ containers hosted in
Proxmox. As such, this README and patches in this fork are tailored for this purpose. If you are interested in the 
original project, refer to the upstream [project](https://github.com/telcat/vagrant-proxmox). With this said, this fork
has the following goals:

* Provide instructions for building an installing the plugin locally. Unlike the original, this plugin must be built and
  installed locally. It is not available for installation via RubyGems.
* Clarify that this project only works with Proxmox 3.x. In the author's experience, this plugin is not compatible with
  Proxmox 4.x APIs.
* Fix a bug in the OpenVZ implementation to allow Vagrant to SSH directly into the OpenVZ containers.
* Provide detailed instructions on configuring Proxmox to use a custom Vagrant-compatible CentOS 7 OpenVZ container.

## Requirements

* Vagrant 1.8+
* Ruby 2.3+
* Promox 3.x (This plugin was only tested with 3.4)

## Building and Installing the Plugin (OS X)

The author's local development environment is an OS X workstation. If you are on OS X, ensure that you have 
[Homebrew](http://brew.sh/) installed.

1. Upgrade Ruby to the latest 2.x version.

        $ brew install ruby

2. Open a new terminal and ensure that Ruby has been updated to the latest version.

        $ ruby --version
        ruby 2.3.0p0 (2015-12-25 revision 53290) [x86_64-darwin15]

3. Clone this project and build the plugin.

        $ git clone https://github.com/GastonGonzalez/vagrant-proxmox.git
        $ cd vagrant-proxmox
        $ gem install bundler
        $ bundle install
        $ rake build

4. Uninstall the `vagrant-proxmox` plugin if exists and install the plugin locally.

        $ vagrant plugin uninstall vagrant-proxmox
        $ vagrant plugin install pkg/vagrant-proxmox-0.0.10.gem
        $ vagrant plugin list

5. Add the dummy Vagrant box for this provider.
   
        $ vagrant box add dummy dummy_box/dummy.box

> Note: For ongoing development, run the following to rebuild and reinstall the plugin.
  `$ rake build && vagrant plugin uninstall vagrant-proxmox && vagrant plugin install pkg/vagrant-proxmox-0.0.10.gem`
        

## Upload a Base OpenVZ Template

Next, we need to create an OpenVZ template that has minimal support for Vagrant. As an example, a CentOS 7 template
will be created. However, the instructions can be easily modified to work with different OpenVZ templates.

1. Download a precreated CentOS 7 template from [https://openvz.org/Download/template/precreated](https://openvz.org/Download/template/precreated).
   It should be noted that Proxmox has a well-defined [naming convention](https://pve.proxmox.com/wiki/Template_naming_convention) for OpenVZ templates,
   as such we save the OpenVZ template using this convention (i.e. `centos-7-minimal_7.0_amd64.tar.gz`).

        $ curl https://download.openvz.org/template/precreated/centos-7-x86_64-minimal.tar.gz -o centos-7-minimal_7.0_amd64.tar.gz
       
2. Log into the Proxmox web GUI and upload the template.


## Add a Promox User (Optional)

Optionally, create a Vagrant user that the plugin can use to manage VMs. Using root is not recommended.

Promox supports multiple [authentication realms](https://pve.proxmox.com/wiki/User_Management#pveum_authentication_realms)—namely, 
PAM and Proxmox VE Authentication Server. We will use the latter.

1. Log into the Proxmox server as root via SSH or the console.

2. Create a vagrant group.
 
        # pveum groupadd vagrant  -comment "Vagrant group"
        
3. Allow the vagrant group to manage the root as a Proxmox administrator.

        # pveum aclmod / -group vagrant -role PVEAdmin
        
4. Create a vagrant user and assign it to the vagrant group.

        # pveum useradd vagrant@pve -group vagrant -password
        
        
## Build a Vagrant-compatible OpenVZ Template

1. Log into the Proxmox web GUI and create a new OpenVZ container. Select `centos-7-minimal_7.0_amd64.tar.gz` as the template.

2. Start the VM and log in.

3. Update CentOS 7 to ensure we have the latest patches for the OS and binaries.

        # yum update -y
        
4. Install basic networking utilities, rsync and sudo. You can add other packages based on your base CentOS 7 needs.

        # yum install -y net-tools rsync sudo
        
5. Create a vagrant user.

        # useradd vagrant
        
6. Allow the vagrant user to become root without a password.

        # visudo
        Defaults:vagrant !requiretty
        vagrant ALL=(ALL) NOPASSWD:ALL  

7. Create an SSH key pair on your local workstation. Most Vagrant boxes use the SSH private/public key pair from 
   https://github.com/mitchellh/vagrant/tree/master/keys. This key pair is insecure as everyone in the world has 
   access to the private key. As such, we will generate a new SSH key pair on our local workstation.
   
        $ mkdir ~/vagrant-keys
        $ chmod 700 ~/vagrant-keys
        $ ssh-keygen -t rsa -C "Vagrant SSH key pair" -f ~/vagrant-keys/vagrant -N ""
   
8. From your local workstation run the following to set up the SSH public key on the VM.

         $ ssh root@YourVmIpAdddress 'mkdir /home/vagrant/.ssh'
         $ scp ~/vagrant-keys/vagrant.pub root@YourVmIpAdddress:/home/vagrant/.ssh/authorized_keys
         $ ssh root@YourVmIpAdddress 'chmod 700 /home/vagrant/.ssh'
         $ ssh root@YourVmIpAdddress 'chmod 600 /home/vagrant/.ssh/authorized_keys'
         $ ssh root@YourVmIpAdddress 'chown -R vagrant:vagrant /home/vagrant/.ssh'
         

9. From your workstation, test your key and ensure that you can log in without a password. Then, test switching to root without a password.

         $ ssh -i ~/vagrant-keys/vagrant vagrant@YourVmIpAdddress
         [vagrant@centos7-base ~]$ sudo su
         [root@centos7-base vagrant]#
         
10. Log into the VM and clean up the disk space and the Bash history.

         [root@centos7-base vagrant]# yum clean all
         [root@centos7-base vagrant]# cat /dev/null > ~/.bash_history && history -c && exit
         [vagrant@centos7-base ~]$ cat /dev/null > ~/.bash_history && history -c && exit
         
11. Log into the Proxmox web GUI and shutdown the VM and then remove the network interface from the _Network_ tab.

12. Log into the Proxmox server via SSH and navigate to the OpenVZ node location (`/var/lib/vz/private/<node ID>`) for 
    the newly created CentOS 7 container. In my case the node ID was _100_.
    
         # cd /var/lib/vz/private/100
         # tar -czpf /var/lib/vz/template/cache/centos-7-vagrantminimal_7.0_amd64.tar.gz .

14. Confirm that you can see your new template under the _Storage View_ > _node_ > local  > _Content_ in the 
    Proxmox web GUI. It should be called `centos-7-vagrantminimal_7.0_amd64.tar.gz`. This will be used within our 
    `Vagrantfile` to provision all CentOS 7 containers moving forward.


## Create a Vagrantfile and Provision

1. On your local workstation, create a `Vagrantfile`. Simply change:

   * `config.ssh.private_key_path` - This should point to the private key on your local workstation that was created earlier. If 
      you are on Windows adjust the path (i.e., C:\Users\gaston\vagrant-keys\vagrant).
   * `proxmox.endpoint` - This should be updated with the IP or hostname of your Proxmox server.
   * `box.vm.network :public_network, ip:` This should be set to the IP address that you wish to assign to your new VM.

```
Vagrant.configure('2') do |config|

    config.ssh.private_key_path = '/Users/gaston/vagrant-keys/vagrant'
    config.ssh.port = 22
    config.vm.hostname = 'centos7'

    config.vm.provider :proxmox do |proxmox|
        proxmox.endpoint = 'https://192.168.5.110:8006/api2/json'
        proxmox.user_name = 'vagrant@pve'
        proxmox.password = 'vagrant'
        proxmox.vm_id_range = 901..910
        proxmox.vm_name_prefix = 'vagrant_'
        proxmox.openvz_os_template = 'local:vztmpl/centos-7-vagrantminimal_7.0_amd64.tar.gz'
        proxmox.vm_type = :openvz
        proxmox.vm_memory = 2048
    end

    config.vm.define :box, primary: true do |box|
        box.vm.box = 'dummy'
        box.vm.network :public_network, ip: '192.168.5.111'
    end

end
```

2. Provision a VM.

        $ vagrant up --provider=proxmox 

If you run into issues, add the `--debug` flag.


## Options

* `endpoint` URL of the JSON API endpoint of your Proxmox installation
* `user_name` The name of the Proxmox user that Vagrant should use
* `password` The password of the above user
* `vm_id_range` The possible range of machine ids. The smallest free one is chosen for a new machine
* `vm_name_prefix` An optional string that is prepended before the vm name
* `vm_type` The virtual machine type, e.g. :openvz or :qemu
* `openvz_os_template` The name of the template from which the OpenVZ container should be created
* `openvz_template_file` The openvz os template file to upload and use for the virtual machine (can be specified instead of `openvz_os_template`)
* `replace_openvz_template_file` Set to true if the openvz os template file should be replaced on the server (default: false)
* `vm_memory` The container's main memory size
* `task_timeout` How long to wait for completion of a Proxmox API command (in seconds)
* `task_status_check_interval` Interval in seconds between checking for completion of a Proxmox API command
* `ssh_timeout` The maximum timeout for a ssh connection to a virtual machine (in seconds)
* `ssh_status_check_interval` The interval between two ssh reachability status retrievals (in seconds)
* `imgcopy_timeout` The maximum timeout for a proxmox server task in case it's an upload (in seconds)
* `selected_node` If specified, only this specific node is used to create machines 
