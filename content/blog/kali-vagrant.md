---
title: "Disposable Kali Environments with Vagrant and Ansible"
date: 2021-06-29T19:58:24-04:00
---

## The Problem

As I've spent time working in Kali Linux I've noticed that, due to the nature of the work done, the installation seems to "rot" a bit faster than average. Let's face it, we install strange libraries and packages, do horrible things to networking, and generally just abuse the install. This is unfortunate, as getting Kali configured just the way you want it can take a little doing and spending time reinstalling everything sucks.

This is where Vagrant and Ansible can help. Leveraging these two tools in concert can give you the ability to nuke and pave your Kali instance in a matter of minutes, including getting it back to the configuration you like. 

## Install Vagrant and Ansible

Checkout [Vagrant's Docs](https://www.vagrantup.com/docs/installation) for install directions. It's worth noting that this post is written with \*nix operating systems in mind, your milage may vary when using Windows. You'll want to install Vagrant and Virtualbox before continuing.

Ansible can be installed via most Linux distributions or on MacOS via [Homebrew](https://brew.sh/)
## Download Kali's Vagrantfile

Our next step is to grab the Kali Linux Vagrantfile. A Vagrantfile contains all the configuration needed for Vagrant to spin up a Kali VM. This is as easy as running the command `vagrant init kalilinux/rolling` from the directory you want to store the configuration for this Vagrant VM. 

## Modify Vagrantfile
Now that we have the Vagrantfile, lets make a few changes that will make life much easier. I've pasted below some snippets of changes I made to the Kali Vagrantfile including some comments to explain the changes.

``` ruby
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

---- snip -----

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
   config.vm.synced_folder "~/workspace", "/root/workspace"  # Instead of the default shared folder which is the directory on your host machine mounted to /vagrant in the VM I prefer my "workspace" directory. This is simply the location of all my projects, cloned repos and the like

----- snip -----

  config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
     vb.memory = "16384"      # Uncomment these lines, and increase memory. the 1G default wont cut it. I did 16G, but my machine has 64G of memory. Allocate as much as you think you can spare
   end
  #

--- snip ----

    config.vm.provision "ansible" do |ansible| # Ansible provisioner, we will talk about this in the next step
      ansible.playbook = "provision.yaml"
      ansible.extra_vars = { ansible_python_interpreter:"/usr/bin/python3" } # This line is important as /usr/bin/python in Kali is still Python2, which wont play nice with Ansible
    end
end


```

## Ansible Provisioning Playbook

We configured the Ansible provisioner in Vagrant to run a playbook called "provision.yaml". Since we didn't specifiy a path to the playbook, Vagrant will look in the same directory the Vagrantfile is in. This playbook can do anything Ansible is capable of. I've shared a portion of mine below to get you started, but you'll likely want your own configuration.

```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: Update apt cache
      shell: apt update

    - name: Update all packages
      apt:
        upgrade: dist

    - name: Install apt packages
      apt:
        name: ['amass', 'sublist3r','gobuster', 'golang-go', 'python-pip-whl', 'python3-pip', 'crackmapexec', 'docker.io', 'virtualbox-guest-x11', 'masscan', 'seclists']
        state: latest

    - name: Clone dotfiles repo
      git:
        repo: "https://github.com/bseb/dotfiles.git"
        dest: /root/dotfiles
      notify:
        - config bootstrap

    - name: Enable Postgres for Metasploit
      service:
        name: postgresql
        state: started
        enabled: yes

  handlers:
    - name: config bootstrap
      shell: /root/dotfiles/bootstrap.sh
```


## Cattle Not Pets

After all this configuration building your Vagrant VM is as simple as running `vagrant up` Notice that this Vagrant configuration will also launch a desktop so you an access all the GUI based tools in Kali. There are two users, vagrant and root who share the password vagrant. You can also use the command `vagrant ssh` to access your machine via ssh. When you are all done working logging out of the machine and using the command `vagrant halt` will shut it down.

Now when your Kali environment starts to get stale, there is an update available, or its Monday and you just can bring yourself to do anything else you can run `vagrant destroy` followed by `vagrant up` to get a fresh new Kali install. Another useful command is `vagrant provision` if you need to re-run your Ansible playbook for any reason.

## Conclusion

I hope you have found this useful. Now a fresh Vagrant evironment is only a few minutes away and you are free to spend your energy on something more interesting than configuring a virtual machine.
