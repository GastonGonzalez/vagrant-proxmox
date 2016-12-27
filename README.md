# Vagrant Proxmox Provider

This is a [Vagrant](http://www.vagrantup.com) plugin that adds a [Proxmox](http://proxmox.com/) provider to Vagrant, 
allowing Vagrant to manage and provision Proxmox virtual machines.


## About this Fork

The primary motivation for this fork was to support the author's goals for managing Linux Containers (LXC)  hosted in
Proxmox. As such, this README and patches in this fork are tailored for this purpose. If you are interested in the 
original project, refer to the upstream [project](https://github.com/telcat/vagrant-proxmox). With this said, this fork
has the following goals:

* Provide instructions for building an installing the plugin locally. Unlike the original, this plugin must be built and
  installed locally. It is not available for installation via RubyGems.
  Proxmox 4.x APIs.
* Merge pull request [#26](https://github.com/telcat/vagrant-proxmox/pull/26) and [#32](https://github.com/telcat/vagrant-proxmox/pull/32)]
* Provide detailed instructions on configuring Proxmox to use a custom Vagrant-compatible CentOS 7 container.

## Requirements

* Vagrant 1.8+
* Ruby 2.3+
* Promox 4.4 

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
        

## Create Vagrant-compatible Container

Refer to the [Add a CentOS 7 Container Image](https://www.gastongonzalez.com/tech-blog/2016/12/24/building-a-developer-virtualization-lab-part-2) section on my blog
post for instructions.


## Add a Promox User (Optional)

Refer to the [Add a Promox User](https://www.gastongonzalez.com/tech-blog/2016/12/24/building-a-developer-virtualization-lab-part-2) sectopm on my blog
post for instructions.

 
## Create a Vagrantfile and Provision

1. On your local workstation, create a `Vagrantfile`. Simply change:

   * `config.ssh.private_key_path` - This should point to the private key on your local workstation that was created earlier.  
   * `proxmox.endpoint` - This should be updated with the IP or hostname of your Proxmox server.
   * `box.vm.network`

```
Vagrant.configure('2') do |config|

    config.ssh.private_key_path = '/Users/gaston/vagrant-keys/vagrant'
    config.vm.hostname = 'centos7-proxmox'

    config.vm.provider :proxmox do |proxmox|
        # Useful for debugging unencrypted request/response with stunnel+Wireshark
        proxmox.endpoint = 'http://localhost:8006/api2/json'
        #proxmox.endpoint = 'https://192.168.5.110:8006/api2/json'
        proxmox.user_name = 'vagrant@pve'
        proxmox.password = 'vagrant'
        proxmox.vm_id_range = 900..910
        proxmox.vm_name_prefix = 'vagrant_'
        proxmox.openvz_os_template = 'local:vztmpl/centos-7-base_20161225_amd64.tar.xz'
        proxmox.vm_type = :lxc
        proxmox.vm_memory = 1024
        proxmox.vm_storage = 'local-lvm'
        proxmox.vm_disk_size = '50G'
    end

    config.vm.define :box, primary: true do |box|
        box.vm.box = 'dummy'
        box.vm.network :public_network, ip: '192.168.5.112', interface: 'eth0', bridge: 'vmbr0', gw: '192.168.5.1'
    end

end

```

2. Provision a VM.

        $ vagrant up --provider=proxmox 

If you run into issues, add the `--debug` flag.
