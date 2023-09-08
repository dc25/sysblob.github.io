---
title: Customizing your image with Cloud-init
image: cloud-init-logo.png
img_path: /images/
date: 2023-09-05
categories: [homelabbing]
tags: [cloud-init,image editing,virt-edit]
pin: false
comments: true
---

One of the immediate burning questions I had when I began spinning up virtual machines was how can I automate more of the manual installation of the operating system? Clicking the same mundane options for every new VM is boring. I'd heard about cloud-init and seen it working in the background on some of my AWS EC2 instances or on my Ubuntu servers but couldn't quite put all the pieces together. What is cloud-init? What are cloud-ready images? Eventually, when I got into enterprise automation, spinning up new VMs fast and efficiently became a necessity. This forced me to sit down and learn quite a bit so I could build my perfect image. Let's take a look at cloud-init.

## What is cloud-init?

![cloud-init logo](cloud-init-intro.png){: w="840" h="400" }{: .shadow }

Cloud-init is a distribution neutral way of deploying a new operating system pre-configured so it can be deployed non-interactively such as through a hypervisor template. Most distributions of linux provide "cloud ready" images which are intended to be used for this purpose. Think of cloud-init as performing the basic things you need to do before you hand off to your configuration management tool. You can do as much or as little as you need to. This can include setting up login/SSH, usernames, time zones, packages, uploading files, and much more.

I'm a big fan of Rocky linux since its goal is to remain a 1:1 clone of RedHat Enterprise Linux. This makes it a great image of choice if you're spinning up servers you're learning on for future employment. Let's take a look at editing the Rocky linux default cloud image.

## Editing an image with virt-edit

First we will download a Rocky cloud image for Rocky 8.8. 

```bash
https://download.rockylinux.org/pub/rocky/8.8/images/x86_64/Rocky-8-GenericCloud-Base.latest.x86_64.qcow2
```
Alternatively download directly to your server.

```bash
wget https://download.rockylinux.org/pub/rocky/8.8/images/x86_64/Rocky-8-GenericCloud-Base.latest.x86_64.qcow2
```
This comes in the form of a qcow2 file which is a generic virtual disk. We can edit this type of file by temporarily mounting directories inside it to a host using a program called virt-edit. We install virt-edit on a generic linux machine which downloaded the image.

```bash
sudo yum -y install libguestfs-tools # or apt
```
We set our variable for guestfs, install nano editor and assign it as default editor. Then display help.

```bash
export LIBGUESTFS_BACKEND=direct
```
```bash
sudo yum install nano -y
export EDITOR=nano
```
```bash
virt-customize --help
```
At this point you can edit files using the virt-edit commands found below. Or if you're savvy you can edit the file which controls all cloud-init which is located at `/etc/cloud/cloud.cfg`

```bash
virt-edit -a Rocky-8-GenericCloud-Base.latest.x86_64.qcow2 /etc/cloud/cloud.cfg
```
Tips for editing cloud.cfg directly:

- You can change disable_root to 0 in order to be able to login as root
- lock_passwd can be changed to false to allow password based authentication
- We can also install packages by adding:

```yaml
packages:
 - qemu-guest-agent
 - nano
 - wget
 - curl
 - net-tools
```

- Make sure to edit the /etc/ssh/sshd_config file to allow password authentication as well if desired. This can be done using the same virt-edit command above.
```bash
PermitRootLogin yes
PasswordAuthentication yes
```

## Alternate virt-customize commands

Virt-customize allows you to inject things into your image without having to open and edit the necessary files.

Setting root password
```bash
virt-customize -a Rocky-8-GenericCloud-Base.latest.x86_64.qcow2 --root-password
```

```bash
virt-customize -a Rocky-8-GenericCloud-Base.latest.x86_64.qcow2 --install [vim,bash-completion,wget,curl,telnet,unzip]
```

Upload files. (local:remote)

```bash
virt-customize -a Rocky-8-GenericCloud-Base.latest.x86_64.qcow2 --upload rhsm.conf:/etc/rhsm/rhsm.conf
```

Set timezone

```bash
virt-customize -a Rocky-8-GenericCloud-Base.latest.x86_64.qcow2 --timezone "America/New_York"
```
Add SSH key
```bash
virt-customize -a Rocky-8-GenericCloud-Base.latest.x86_64.qcow2  --ssh-inject jmutai:file:./id_rsa.pub
```

## Running on first boot only

I found that the first time my cloud-init configured instance booted I wanted it to finish configuring and then reboot itself one time to register the IP address properly in DNS. Use virt-edit and then add something at the bottom of the file like this to reboot the machine 60 seconds after cloud-init finishes.

```yaml
power_state:
    delay: 1
    mode: reboot
    message: Rebooting machine
    condition: true
```

## Creating a VM template in Proxmox

I use Proxmox as my open-source hypervisor so this guide will also cover creating a VM template from this image.

We do this by SSHing into Proxmox and executing the following commands.

Create a VM with ID 8000, 8GB memory, 4 cores, name it rocky-cloud and give it basic networking.

```bash
qm create 8000 --memory 8192 --core 4 --name rocky-cloud --net0 virtio,bridge=vmbr0
```
Import the image. vm-storage would be the name of the local storage you want to host this VM. Proxmox default storage location is local-lvm.
```bash
qm importdisk 8000 Rocky-8-GenericCloud-Base.latest.x86_64.qcow2 vm-storage
```
Create a scsi controller to handle the disks and attach the image disk.
```bash
qm set 8000 --scsihw virtio-scsi-pci --scsi0 vm-storage:vm-8000-disk-0
```
attach the CD drive as a cloud-init drive.
```bash
qm set 8000 --ide2 vm-storage:cloudinit
```
Set the image disk to boot first.
```bash
qm set 8000 --boot c --bootdisk scsi0
```
Increase storage by desired size.
```bash
qm resize 8000 scsi0 +15G
```
Convert to template
```bash
qm template 8000
```
## Last steps

Now that we have a working template `go into proxmox and find the template we created. Now you can click "cloud-init" and edit the final settings.` Here we edit the default username and password, SSH keys, and **it is extremely important to set the DHCP to dynamic otherwise the host will not grab networking**. I add my domain in the DNS domain field. I also go under options and check the box for **QEMU guest agent = enabled.**

> For Rocky Linux the VM will not work on proxmox unless you also go under hardware and select processors and under type make your type at the very bottom "host".
{: .prompt-warning }

Once all this is done you can now use this template by right clicking it and selecting clone. `Full clone option will make the clone's hard drive entirely independent and is preferred`. Once the machine is ran cloud-init will configure it and after 5 minutes or so you should be able to connect to it via SSH/login.