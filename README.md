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

* Vagrant 1.9.4+
* Ruby 2.2.3
* Promox 4.4 

## Building and Installing the Plugin (OS X)

The author's local development environment is an OS X workstation. If you are on OS X, ensure that you have 
[Homebrew](http://brew.sh/) installed.

1. Install `rbenv` so that you can manage multiple versions of Ruby.

        $ brew update
        $ brew install rbenv

2. Add the rbenv shim to your PATH. It’s recommended that you add it to your Bash profile (~/.bash_profile).

        PATH=$PATH:~/.rbenv/shims
        export PATH

3. Source ~/.bash_profile to pick up your updated PATH.

        $ . ~/.bash_profile


4. Install Ruby 2.2.3 and add all the shims.

        $ rbenv install 2.2.3
        $ rbenv rehash

5. Ensure that the current active version is set to 2.2.3.

        $ rbenv versions
        * system (set by /Users/gaston/.rbenv/version)
        2.2.3

6. Verify that rbenv is actually using Ruby to 2.2.3.

        $ ruby -v

7. Clone this project and build the plugin.

        $ git clone https://github.com/GastonGonzalez/vagrant-proxmox.git
        $ cd vagrant-proxmox
        $ rbenv local 2.2.3
        $ gem install bundler
        $ bundle install
        $ rake build

8. Uninstall the `vagrant-proxmox` plugin if exists and install the plugin locally.

        $ vagrant plugin uninstall vagrant-proxmox
        $ vagrant plugin install pkg/vagrant-proxmox-0.0.10.gem
        $ vagrant plugin list

9. Add the dummy Vagrant box for this provider.
   
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

On your local workstation, create a `Vagrantfile`. Simply change:

   * `config.ssh.private_key_path` - This should point to the private key on your local workstation that was created earlier.  
   * `proxmox.endpoint` - This should be updated with the IP or hostname of your Proxmox server.
   * `box.vm.network` - The `cidr_block` is required. Most users will likely use '/24' if they are using 255.255.255.0 netmask.

```
Vagrant.configure('2') do |config|

    config.ssh.private_key_path = '/Users/gaston/vagrant-keys/vagrant'
    config.vm.hostname = 'centos7-proxmox'

    config.vm.provider :proxmox do |proxmox|
        # Useful for debugging unencrypted request/response with stunnel+Wireshark
        #proxmox.endpoint = 'http://localhost:8006/api2/json'
        proxmox.endpoint = 'https://192.168.5.110:8006/api2/json'
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
        box.vm.network :public_network, ip: '192.168.5.112', cidr_block: '/24', interface: 'eth0', bridge: 'vmbr0', gw: '192.168.5.1'
    end

end

```

Provision a VM.

        $ vagrant up --provider=proxmox 

If you run into issues, add the `--debug` flag.
