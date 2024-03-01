# How to create VM Template with Proxmox 8.x

Lastly, I setup a lab machine where I installed Proxmox. It's based on a second hand Dell Optiplex 7040 Mini with an i7 6700t processor.
The case is really small and silent but have enough performance to host several virtual machines in parallele.
I use it to experiment some prototypes I need to build. Per example, I played with podman containers in rootless mode on openSUSE Leap VM hosted on it.

A few months ago, I decided to learn how to build and use Kubernetes Infrastructure. I followed the course **Architecting with Google Kubernetes Engine** on <https://www.coursera.org/>. It was very interesting and I learnt a lot.

But now, I must practice and get my hands dirty to gain some experiences with that knowledges. That the reason why I decided to create a little Kubernetes cluster on my environment. And the first step is to set a VM Template that will be the starting point of my journey.

## Get container optimized system

In order to create our VM template, we'll need to download system images from a distribution of our choice. As I used podman on openSUSE Leap, I decided to test the container optimized system from openSUSE and named MicroOS. You'll find some information about it [Here](https://get.opensuse.org/microos/).

### Get image for openSUSE-MicroOS.x86_64-ContainerHost

As we selected our distribs, go to Proxmox Web interface. I opened up my standalone Cluster node and switch to the Shell interface. You should see a system prompt like the one below.

```bash
Linux midgard 6.5.11-7-pve #1 SMP PREEMPT_DYNAMIC PMX 6.5.11-7 (2023-12-05T09:44Z) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Jan 19 14:27:13 CET 2024 on pts/0
root@midgard:~#
```

You can get the image with following command.

>wget https://download.opensuse.org/tumbleweed/appliances/openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2

I choosed that image because I needed qemu guest agent support.

```bash
root@midgard:~# wget https://download.opensuse.org/tumbleweed/appliances/openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2
--2024-01-19 14:28:17--  https://download.opensuse.org/tumbleweed/appliances/openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2
Resolving download.opensuse.org (download.opensuse.org)... 195.135.223.226, 2a07:de40:b250:131:10:151:131:30
Connecting to download.opensuse.org (download.opensuse.org)|195.135.223.226|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://downloadcontent.opensuse.org/tumbleweed/appliances/openSUSE-MicroOS.x86_64-16.0.0-ContainerHost-kvm-and-xen-Snapshot20240118.qcow2 [following]
--2024-01-19 14:28:18--  https://downloadcontent.opensuse.org/tumbleweed/appliances/openSUSE-MicroOS.x86_64-16.0.0-ContainerHost-kvm-and-xen-Snapshot20240118.qcow2
Resolving downloadcontent.opensuse.org (downloadcontent.opensuse.org)... 195.135.223.227, 2a07:de40:b250:131:10:151:131:31
Connecting to downloadcontent.opensuse.org (downloadcontent.opensuse.org)|195.135.223.227|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 486539264 (464M) [application/octet-stream]
Saving to: ‘openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2’

openSUSE-MicroOS.x86_64-ContainerHost-kvm 100%[==================================================================================>] 464.00M   290KB/s    in 10m 55s 

2024-01-19 14:39:12 (726 KB/s) - ‘openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2’ saved [486539264/486539264]

root@midgard:~#
```

You now have your image available to make a VM Template from it.

```bash
root@midgard:~# ls -lh openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2
-rw-r--r-- 1 root root 464M Jan 19 03:17 openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2
root@midgard:~# 
```

## Create and set initial VM for MicroOS

I followed Proxmox documentation available at this [place](https://pve.proxmox.com/wiki/Cloud-Init_Support). As I set a VLAN network on my proxmox environment, I have to specify a tag numbered with the vlan number. I already created a previous template, so my template ID will be 9001.

That command create a new VM with VirtIO SCSI controller, specify vlan network and set memory to 2048Mo.

>qm create 9001 --memory 2048 --net0 virtio,bridge=vmbr1,tag=10 --scsihw virtio-scsi-pci

```bash
root@midgard:~# qm create 9001 --memory 2048 --net0 virtio,bridge=vmbr1,tag=10 --scsihw virtio-scsi-pci
root@midgard:~#
```

## Import MicroOS image on local-lvm storage

Before using it, you must import the openSUSE MicroOS downloaded disk to the local-lvm storage, attaching it as a SCSI drive. Use the following command.

>qm set 9001 --scsi0 local-lvm:0,import-from=/root/openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2

```bash
root@midgard:~# qm set 9001 --scsi0 local-lvm:0,import-from=/root/openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2
update VM 9001: -scsi0 local-lvm:0,import-from=/root/openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2
  Logical volume "vm-9001-disk-0" created.
transferred 0.0 B of 20.0 GiB (0.00%)
transferred 206.8 MiB of 20.0 GiB (1.01%)
transferred 411.6 MiB of 20.0 GiB (2.01%)
...
transferred 19.8 GiB of 20.0 GiB (99.02%)
transferred 20.0 GiB of 20.0 GiB (100.00%)
transferred 20.0 GiB of 20.0 GiB (100.00%)
scsi0: successfully created disk 'local-lvm:vm-9001-disk-0,size=20G'
root@midgard:~#
```

>**WARNING**: As you noticed, the size of the disk can be voluminous. The previous template, I made with openSUSE Leap image was only 10Go.

## Prepare settings to load Ignition parameters

Initialy, I tried to use cloudinit presets of the virtual machine. But the Micro-OS image I choose in order to get qemu guest agent doesn't support it. So I switched to Ignition that is natively supported.

Now you can set display device. It is possible to use default display (vga) hors to use qxl (spice). I choosed qxl.

>qm set 9001 --vga qxl

```bash
root@midgard:~# qm set 9001 --vga qxl
update VM 9001: -vga qxl
root@midgard:~#
```

## Customize the template

By default Proxmox use cloud-init to customize VM template. But in our case we use a system that only support "**Ignition**".

### Activating qemu-guest-agent

To manage interaction between host and guest machine we need that "**qemu-guest-agent**" was installed in our VM. That is a reason that make me switch from [**openSUSE-MicroOS.x86_64-ContainerHost-OpenStack-Cloud.qcow2**] image to [**openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2**]. Indeed, I identified that qemu-guest-agent package are not installed in the first one. As I don't want to drift from initial image settings but need that extension, I choose the second image.

The command to active qemu-guest-agent on the VM template is:

>qm set 9001 --agent 1

```bash
root@midgard:~# qm set 9001 --agent 1
update VM 9001: -agent 1
root@midgard:~#
```

### Add a cdrom device

To load ignition settings from ignition.iso add a cdrom device to the vm template with the following command:

>qm set 9001 --ide2 none,media=cdrom

```bash
root@midgard:~# qm set 9001 --ide2 none,media=cdrom
update VM 9001: -ide2 none,media=cdrom
root@midgard:~#
```

### Set Boot Order to scsi0

Since we've introduced a CD-ROM drive to our template, we can ensure the VM boots directly from the scsi0. Restrict the BIOS boot order with the following command:

>qm set 9001 --boot order=scsi0

```bash
root@midgard:~# qm set 9001 --boot order=scsi0
update VM 9001: -boot order=scsi0
```

### Finalize openSUSE MicroOS VM Template

All is ready to transform our VM to a template from which we'll be able to clone VM. To retrieve easily the operating system associated to the template, I name the VM before creating the template. I also decided to add a suffix with the creation date of the template.

>qm set 9001 --name template-opensuse-MicroOS-202403

```bash
root@midgard:~# qm set 9001 --name template-opensuse-MicroOS-202403
update VM 9001: -name template-opensuse-MicroOS-202403
root@midgard:~#
```

You build the template with following command:

>qm template 9001

```bash
root@midgard:~# qm template 9001
  Renamed "vm-9001-disk-0" to "base-9001-disk-0" in volume group "pve"
  Logical volume pve/base-9001-disk-0 changed.
  WARNING: Combining activation change with other commands is not advised.
root@midgard:~#
```

All is done. Your template is available.

## How to validate our VM template

### First boot setting with Ignition

This chapter is inspired by that [article](https://www.phillipsj.net/posts/opensuse-microos-ignition-and-proxmox/) from [Jamie Phillips](https://github.com/phillipsj). I also read carefuly the documentation available:

- <https://en.opensuse.org/Portal:MicroOS/Ignition#First_Boot_Configuration>
- <https://en.opensuse.org/Portal:MicroOS/Ignition#JSON_Examples>
- <https://github.com/coreos/ignition>

When you start you VM for the first time, Ignition is called by the machine in order to configure itself. It relies on a file named [**config.ign**] where instructions are represented in JSON format.
The file is located in a directory name [**ignition**] at the root of the device used. In our case, we'll use ISO-Image to store it.

```text
<root directory>
└── ignition
    └── config.ign
```

Let's create all we need. I decided to create a file named config.tpl in the current directory. I'll use it to generate the config.ign uniq to each virtual machine built from template. Indeed, We have to update hostname for each virtual machine built from the template. As Ignition configuration is populated only once when the vm start, we must update its content before using it and not include it in the template.

```bash
root@midgard:~# mkdir -p iso/ignition
root@midgard:~# touch config.tpl
```

We'll start by setting user, password and ssh_key by editing the content of [**config.tpl**].

```bash
root@midgard:~# vi config.tpl
```

Copy the sample below in it and replace all parts needed.

```json
{
  "ignition": { "version": "3.1.0" },
  "passwd": {
    "users": [
      {
        "name": "root",
        "passwordHash": "<paste your passwordhash>",
        "sshAuthorizedKeys": 
          [
            "<paste your public key here>"
          ]
      }
    ]
  },
  "storage": {
    "files": [{
      "filesystem": "root",
      "path": "/etc/hostname",
      "mode": 420,
      "overwrite": true,
      "contents": { "source": "data:,HOSTNAME2REPLACE" }
    }]
  }
}
```

The passwordHash value can be created with the following command in interactive mode:

>openssl passwd -5

Per example if you set your password to "S3cR3tP4ssW0rd"

```bash
root@midgard:~# openssl passwd -5
Password: 
Verifying - Password: 
$5$3TqWkZxvgvLixYoC$V1gqna7EfO6Qjsa5SmgSia6ZM2DiwEvxP3xBkXJqUa0
root@midgard:~#
```

Documentation about openssl passwd is available at this [place](https://www.openssl.org/docs/man3.0/man1/openssl-passwd.html).

Under the bloc storage we have a string "**HOSTNAME2REPLACE**" that we'll replace via a command sed in order to generate our final ignition file (**iso/ignition/config.ign**).

>sed s/HOSTNAME2REPLACE/test-vm-auto/ config.tpl >iso/ignition/config.ign

When file is updated, it is time to generate your ISO. Use the following command:

>mkisofs -o ignition.iso -V ignition iso

```bash
root@midgard:~# mkisofs -o ignition.iso -V ignition iso
I: -input-charset not specified, using utf-8 (detected in locale settings)
Total translation table size: 0
Total rockridge attributes bytes: 0
Total directory bytes: 2048
Path table size(bytes): 26
Max brk space used 0
176 extents written (0 MB)
root@midgard:~# ls -lh ignition.iso
-rw-r--r-- 1 root root 352K Jan 19 17:30 ignition.iso
root@midgard:~#
```

In order to use it in your VM, copy ignition.iso in the directory used to store ISO in Proxmox. It is [**/var/lib/vz/template/iso/**].

```bash
root@midgard:~# cp ignition.iso /var/lib/vz/template/iso/
root@midgard:~#

root@midgard:~# ls -lh /var/lib/vz/template/iso/ignition.iso 
-rw-r--r-- 1 root root 352K Jan 19 17:39 /var/lib/vz/template/iso/ignition.iso
root@midgard:~#
```

### Create the vm from template

We have a VM template. We can verify that our work have been well done by using it to set a new VM from it.

Lets do that with following command:

>qm clone 9001 <id_of_the_vm> --name <hostname_of_the_vm>

```bash
root@midgard:~# qm clone 9001 108 --name test-vm-auto
create full clone of drive ide2 (local-lvm:vm-9001-cloudinit)
  Logical volume "vm-108-cloudinit" created.
create linked clone of drive scsi0 (local-lvm:base-9001-disk-0)
  Logical volume "vm-108-disk-0" created.
root@midgard:~#
```

After creating the vm from template, you have to load ignition settings via ignition.iso file built previously.

>qm set <id_of_the_vm> --ide2 local:iso/ignition.iso,media=cdrom

```bash
root@midgard:~# qm set 108 --ide2 local:iso/ignition.iso,media=cdrom
update VM 108: -ide2 local:iso/ignition.iso,media=cdrom
root@midgard:~#
```

Start your new VM and then logon to it with ssh.

>qm start <id_of_the_vm>

```bash
root@midgard:~# qm start 108
```

As you set an ssh pubkey in your Ignition config file, you connect to the vm with you ssh key.

```bash
Using username "root".
Authenticating with public key "imported-openssh-key"
Last login: Mon Feb  5 14:45:43 UTC 2024 from 192.168.3.10 on ssh
test-vm-auto:~ # cat /etc/os-release
NAME="openSUSE MicroOS"
# VERSION="20240228"
ID="opensuse-microos"
ID_LIKE="suse opensuse opensuse-tumbleweed"
VERSION_ID="20240228"
PRETTY_NAME="openSUSE MicroOS"
ANSI_COLOR="0;32"
CPE_NAME="cpe:/o:opensuse:microos:20240228"
BUG_REPORT_URL="https://bugzilla.opensuse.org"
SUPPORT_URL="https://bugs.opensuse.org"
HOME_URL="https://www.opensuse.org/"
DOCUMENTATION_URL="https://en.opensuse.org/Portal:MicroOS"
LOGO="distributor-logo-MicroOS"
test-vm-auto:~ #
```

You can also get information about network interfaces from your proxmox Shell with the following command.

>qm guest cmd <id_of_the_vm> network-get-interfaces

```bash
root@midgard:~# qm guest cmd 108 network-get-interfaces
```

Result in JSON format.

```json
[
   {
      "hardware-address" : "00:00:00:00:00:00",
      "ip-addresses" : [
         {
            "ip-address" : "127.0.0.1",
            "ip-address-type" : "ipv4",
            "prefix" : 8
         },
         {
            "ip-address" : "::1",
            "ip-address-type" : "ipv6",
            "prefix" : 128
         }
      ],
      "name" : "lo",
      "statistics" : {
         "rx-bytes" : 3724,
         "rx-dropped" : 0,
         "rx-errs" : 0,
         "rx-packets" : 44,
         "tx-bytes" : 3724,
         "tx-dropped" : 0,
         "tx-errs" : 0,
         "tx-packets" : 44
      }
   },
   {
      "hardware-address" : "b6:ad:86:35:12:92",
      "ip-addresses" : [
         {
            "ip-address" : "192.168.3.102",
            "ip-address-type" : "ipv4",
            "prefix" : 24
         },
         {
            "ip-address" : "fe80::e16e:6393:36f7:42ef",
            "ip-address-type" : "ipv6",
            "prefix" : 64
         }
      ],
      "name" : "ens18",
      "statistics" : {
         "rx-bytes" : 36919,
         "rx-dropped" : 0,
         "rx-errs" : 0,
         "rx-packets" : 338,
         "tx-bytes" : 170190,
         "tx-dropped" : 0,
         "tx-errs" : 0,
         "tx-packets" : 453
      }
   }
]
```

## Conclusion

If you reach this point, I supposed that you succed to built your first vm from a Proxmox vm template. Congratulations and thank you for the time you spent in reading my article. When I wrote it, I tried several path. One of them was to use Cloud-init settings that is natively supported in Proxmox. But, I understood that was not possible to use it because of the version of the OS image I would used. I plan to write another article where I'll describe steps to build a VM Template with Cloud-init.

Now that we have our container optimized system, we'll going further to build a minimal kubernetes cluster. That will be our next step. 

All files are available from my GitHub Account [Here]().
