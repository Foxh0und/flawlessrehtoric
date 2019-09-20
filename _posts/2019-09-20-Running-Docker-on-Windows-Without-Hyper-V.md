---
layout: post
title: "Running Docker on Windows Without Hyper-V"
description: "VirtualBox and other Virtualisers can live in harmony with Docker on Windows"
tag: Computers
---

I'm a big advocate for using Docker, however, it can be a large pain on Windows. Although I am primarily a linux user, I do a lot of my development on a Windows machine for work, and run several VM's for various purposes, predominatly using Virtual Box. Docker Community Edition's requirment for Hyper-V limtis it's availability, with many neeeding to forgo it for their other needs. 

<br>
This tutorial will show you how to setup Docker using Windows Subsystem for Linux (WSL), hiding the daemon away in a VM so you can still run all your commands and access your containers from a VM. It assumes that you WSL installed. If not, follow [these instructions](https://docs.microsoft.com/en-us/windows/wsl/install-win10).

<br><br>
## Setup your VM and install the Docker Daemon
<br>
1. Install Virtual Box from [here](https://www.virtualbox.org/wiki/Downloads), and ensure that you install the Host Only Network Adapter.

<br>
2. Once setup, download the [CentOS Minimal ISO](https://www.centos.org/download/), and build a VM, installing CentOS. I gave my machine 2048MB of RAM. It's best too also give your machine a user, and avoid using the root account. **USE THE SAME USERNAME AS YOUR WSL USERNAME**. This isn't completley essentials, but is helpful for avoiding potential errors later on. After setup has concluded, and you can ensure the machine can get out to the internet, power it down.

<br>
3. Now, go to the settings of your newly created VM, in my case named container. Under Networking, enable Adapter 2, and attach it to your Host Only Network Adapter. Power on the machine.
<br>
![HostOnlyNetwork](https://i.imgur.com/hFRvdPw.png)

<br>
4. Back inside the machine, power it on, we need to discover our new adapters IP, which we'll use for later. To do so, we'll need the ifconfig command found from net-tools.
Run the following commands:

    sudo yum -y install net-tools
    ifconfig -a

The second command will result in something similar to the following.

![IP](https://i.imgur.com/EUywvxS.png)

Make sure you write the IP down somewhere. 

<br>
5. Time to install Docker. I've pulled the commands from [Docker's Docs](https://docs.docker.com/install/linux/docker-ce/centos/), which I recommend reading for your own knowledge.

        sudo yum install -y yum-utils \
            device-mapper-persistent-data \
            lvm2
    
        sudo yum-config-manager \
            --add-repo \
            https://download.docker.com/linux/centos/docker-ce.repo

        sudo yum install docker-ce docker-ce-cli containerd.io

Docker is now installed, time to enable it.

    sudo systemctl enable docker

And now, make sure it's running with

    sudo systemctl status docker

Finally, add our user too the docker group.
Note, if you're using a different username on WSL, add it instead of $USER.
    
    sudo usermod -a -G docker $USER

<br>
## Setting up Docker CLI on WSL to control Docker from Windows.
<br>

1. Now, exit your VM, and load up WSL. Install Docker's dependencies, add the pgp key, and add the stable repository for Ubuntu.
    
        sudo apt-get install -y \
            apt-transport-https \
            ca-certificates \
            curl \
            software-properties-common

        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

        sudo add-apt-repository \
            "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
            $(lsb_release -cs) \
            stable"

<br>
2. Now, update your system, and install the latest
    
        sudo apt-get update -y
        sudo apt-get install -y docker-ce

<br>
3. Install Docker Compose using PIP (with commands should you not already have it)

    sudo apt-get install -y python python-pip
    pip install --user docker-compose

<br>
4. Ensure Volume Mounts Work. 
<br>
I'd like to thank [Nick Jatekins and his blog post](https://nickjanetakis.com/blog/setting-up-docker-for-windows-and-wsl-to-work-flawlessly#ensure-volume-mounts-work) for pointing this solution out to me. 
<br>
For this, we need to edit our volume mounts to make sure things are working. This requires you to have the Windows 10 1803 or later.
<br>
Add the following too 3 lines to /etc/wsl.conf with a text editor of your choice. I use VIM.
    
    [automount]
    root = /
    options = "metadata"

<br>
You'll need to restart to make these changes take effect.

<br>
5. We need to point WSL to use our VM's Docker Host. Remember the IP for the Host Only Network Adapter before? Here is where we will use it, substituting it(VMIP) and the VM's username below (VMUN).

        echo "export DOCKER_HOST=ssh://<VMUN>@<VMIP>:2375" >> ~/.bashrc && source ~/.bashrc

<br>
6. Time to get going, run docker info, and we should be good to go.
    
    docker info

<br>
We'll be prompted for the VM's password, and then the result will be returned.

<br>
## Setting up SSH Keys
<br>
It obviously will get very tedious setting up SSH Keys every time we run a command, so we can setup SSH Keys to avoid this.

<br>
1. Generate an RSA SSH Key in WSL

        ssh-keygen -t rsa

<br>
2. Create the .ssh folder on your VM
    
        ssh <VMUN>@<VMIP> mkdir -p .ssh

<br>
3. Add your SSH Key to the Authorised Keys on the VM

         cat .ssh/id_rsa.pub | ssh <VMUN>@<VMIP> 'cat >> .ssh/authorized_keys'

<br>
4. Set he permissions on the VM's .ssh directory
        
        ssh <VMUN>@<VMIP> "chmod 700 .ssh; chmod 640 .ssh/authorized_keys"