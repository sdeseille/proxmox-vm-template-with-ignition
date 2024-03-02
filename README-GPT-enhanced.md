# Building a Container-Optimized VM Template with Ignition on Proxmox 8.x

In my recent endeavors, I set up a lab machine powered by a second-hand Dell Optiplex 7040 Mini, featuring an i7 6700t processor. Despite its compact size and silent operation, this machine offers sufficient performance to host multiple virtual machines concurrently. I utilize this setup to experiment with various prototypes, such as running podman containers in rootless mode on an openSUSE Leap VM hosted within it.

A few months ago, I embarked on a journey to learn about building and utilizing Kubernetes infrastructure. Following the course "Architecting with Google Kubernetes Engine" on [Coursera](https://www.coursera.org/) provided invaluable insights and knowledge.

Now, it's time to put that knowledge into practice and gain hands-on experience. To kickstart my journey into Kubernetes, I need to establish a foundation—a VM template that will serve as the cornerstone of my endeavors.

## Obtaining a Container-Optimized System

To create our VM template, we first need to acquire system images from a distribution of our choice. Given my previous experience with podman on openSUSE Leap, I opted to explore the container-optimized system from openSUSE, known as MicroOS. You can find more information about it [here](https://get.opensuse.org/microos/).

### Downloading the openSUSE-MicroOS.x86_64-ContainerHost Image

After selecting our distribution, we navigate to the Proxmox Web interface. Accessing the Shell interface of our standalone Cluster node, we encounter a system prompt like the one below.

```bash
Linux midgard 6.5.11-7-pve #1 SMP PREEMPT_DYNAMIC PMX 6.5.11-7 (2023-12-05T09:44Z) x86_64
...
```

Execute the following command to download the image:

```bash
wget https://download.opensuse.org/tumbleweed/appliances/openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2
```

This command fetches the desired image, which includes support for the qemu guest agent.

## Creating and Configuring the Initial VM for MicroOS

Following the guidance provided in the Proxmox documentation, I executed the command below to create a new VM with specific configurations:

```bash
qm create 9001 --memory 2048 --net0 virtio,bridge=vmbr1,tag=10 --scsihw virtio-scsi-pci
```

## Importing the MicroOS Image to Local-LVM Storage

To make use of the downloaded openSUSE MicroOS disk image, it needs to be imported into the local-lvm storage and attached as a SCSI drive. The following command accomplishes this task:

```bash
qm set 9001 --scsi0 local-lvm:0,import-from=/root/openSUSE-MicroOS.x86_64-ContainerHost-kvm-and-xen.qcow2
```

A warning is issued regarding the potentially large size of the disk.

## Customizing the Template

As the default Proxmox setup utilizes cloud-init for template customization, adjustments were made to accommodate Ignition instead.

### Set Display device

It is possible to use default display (vga) hors to use qxl (spice). I choosed qxl.

```bash
qm set 9001 --vga qxl
```

### Activating qemu-guest-agent

To facilitate interaction between the host and guest machine, the qemu-guest-agent package needed activation within the VM template:

```bash
qm set 9001 --agent 1
```

### Adding a CD-ROM Device

To load Ignition settings from ignition.iso, a cdrom device was added to the VM template:

```bash
qm set 9001 --ide2 none,media=cdrom
```

### Setting Boot Order to scsi0

The boot order was configured to prioritize scsi0 to ensure the VM boots directly from it:

```bash
qm set 9001 --boot order=scsi0
```

### Finalizing the openSUSE MicroOS VM Template

To transform the VM into a template for future use, the VM was named and then converted using the following commands:

```bash
qm set 9001 --name template-opensuse-MicroOS-202403
qm template 9001
```

## Validating the VM Template

### First Boot Setting with Ignition

This section outlines the process of configuring the first boot settings for the VM using Ignition. Ignition is a tool used to configure the initial settings of a machine, such as users, passwords, and network configuration, in an automated and reproducible manner. Follow these steps to create and apply the Ignition configuration:

#### 1. Prepare the Ignition Configuration Template

Start by creating an Ignition configuration template file. This file will contain the initial settings you want to apply to the VM upon first boot. Here's how to create the template file:

```bash
mkdir -p iso/ignition
touch config.tpl
```

Edit the `config.tpl` file to define the desired configuration settings using JSON format. Include details such as user accounts, passwords, SSH keys, and any other system configurations required for your environment.

```json
{
  "ignition": {
    "version": "3.1.0"
  },
  "passwd": {
    "users": [
      {
        "name": "root",
        "passwordHash": "<password_hash>",
        "sshAuthorizedKeys": [
          "<ssh_public_key>"
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
      "contents": {
        "source": "data:,HOSTNAME_TO_REPLACE"
      }
    }]
  }
}
```

Replace `<password_hash>` with the hashed password for the root user and `<ssh_public_key>` with the SSH public key you want to authorize for remote access. Additionally, replace `HOSTNAME_TO_REPLACE` with the desired hostname for the VM.

#### 2. Generate the Ignition Configuration File

Once you've defined the configuration settings in the template file, use `sed` to replace placeholders in the template with actual values and generate the Ignition configuration file (`config.ign`).

```bash
sed 's/HOSTNAME_TO_REPLACE/<hostname_of_vm>/' config.tpl > iso/ignition/config.ign
```

Replace `<hostname_of_vm>` with the hostname you want to assign to the VM.

#### 3. Create the Ignition ISO

Next, create an ISO image containing the Ignition configuration file (`config.ign`). Use the `mkisofs` command to generate the ISO image.

```bash
mkisofs -o ignition.iso -V ignition iso
```

This command creates an ISO image named `ignition.iso` in the `iso` directory.

#### 4. Apply the Ignition Configuration to the VM

Now that you have the Ignition ISO image, you can apply the configuration to the VM. Follow these steps:

1. Copy the `ignition.iso` file to the directory used to store ISO images in Proxmox (`/var/lib/vz/template/iso/`).

    ```bash
    cp ignition.iso /var/lib/vz/template/iso/
    ```

2. Create a VM from the template.

    ```bash
    qm clone 9001 108 --name test-vm-auto
    ```

3. Set the CD-ROM drive of the VM to boot from the `ignition.iso` file.

    ```bash
    qm set <id_of_vm> --ide2 local:iso/ignition.iso,media=cdrom
    ```

    Replace `<id_of_vm>` with the ID of the VM.

#### 5. Start the VM

Start the VM to initiate the first boot process. Ignition will automatically apply the configuration settings defined in the `config.ign` file during the boot process.

```bash
qm start <id_of_vm>
```

Replace `<id_of_vm>` with the ID of the VM.

After completing these steps, the VM will boot with the specified configuration settings applied, allowing for automated and consistent provisioning of new VM instances.

Upon successful boot, the VM was accessible via SSH using the configured Public Key.

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

## Conclusion

Congratulations on reaching the end of this guide! By following the steps outlined here, you've successfully created your first VM template with Proxmox, paving the way for efficient VM deployment and management. Whether you're setting up a small lab environment or preparing for larger-scale deployments, understanding how to create and customize VM templates is a valuable skill.

As you've seen, Proxmox offers powerful tools and features for creating and managing VM templates, including support for Ignition configuration and automated provisioning. While this guide focused on using Ignition for initial VM configuration, Proxmox also supports other configuration methods such as Cloud-init, providing flexibility to meet different requirements.

I'd like to note that this article has been improved with the assistance of ChatGPT, an AI language model developed by OpenAI. By leveraging ChatGPT's capabilities, the content was refined to ensure clarity, accuracy, and reader engagement. This collaboration demonstrates the potential of AI-driven tools to enhance the quality and effectiveness of technical documentation.

Thank you for taking the time to explore this guide. I hope you found it informative and valuable for your virtualization projects. If you have any questions or feedback, feel free to reach out. Happy virtualizing!

All files referenced in this article are available on my [GitHub Account](https://github.com/sdeseille).
