# Single Gpu Passthrough guide AMD
### Info
- OS Ubuntu
- AMD Radeon RX 580
- AMD Ryzen 5 1600 (12)
- 16GB RAM

## Part 1 BIOS Config
Make sure virtualization is enabled in your BIOS.
- IOMMU Enabled
- SVM Enabled

Also make sure to update your system before starting anything.

## Part 2 GRUB Loader
Edit the grub config file with ```sudo vim /etc/default/grub```

Find the line:

```GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"```

And change it to:

```GRUB_CMDLINE_LINUX_DEFAULT="amd_iommu=on iommu=pt iommu=1 video=efifb:off quiet splash"```

Then update the changes with ```sudo update-grub```

Reboot the PC and check that the grub loader is working properly with ```sudo cat /proc/cmdline```

The output should look like:

```BOOT_IMAGE=/boot/vmlinuz-5.4.0-60-generic root=UUID=0587b30a-06cf-4df2-82fe-fb8db547e1c5 ro amd_iommu=on iommu=pt iommu=1 video=efifb:off quiet splash vt.handoff=1```

## Part 3 GPU Address
Now we need to find the pci address of the gpu and its audio component with ```lspci -nnk```

Since we are using Radeon we can grep it like this ```lspci -nnk | grep 'Radeon'```
We are looking for the output that looks like:

```
08:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon RX 470/480/570/570X/580/580X/590] [1002:67df] (rev e7)
	Subsystem: Sapphire Technology Limited Radeon RX 570 Pulse 4GB [1da2:e353]
08:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere HDMI Audio [Radeon RX 470/480 / 570/580/590] [1002:aaf0]
	Subsystem: Sapphire Technology Limited Ellesmere HDMI Audio [Radeon RX 470/480 / 570/580/590] [1da2:aaf0]

```

Remember the address we got **08:00.0** and **08:00.1**

## Part 4 Virtualization Software
Now we install all the software needed for virtualization. Since we are on Ubuntu use the following command.

```sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils virt-manager ovmf```

## Part 5 Configure Libvirt
After that we have to update the libvirt config with ```sudo vim /etc/libvirt/libvirtd.conf```

Look for the lines:

```
#unix_sock_group = "libvirt"
#unix_sock_ro_perms = "0777"
#unix_sock_rw_perms = "0770"

#log_filters="1:qemu"
#log_outputs="1:file:/var/log/libvirt/libvirtd.log"
```

And uncomment them. If they do not exist then simply add them to the file.

Now we need to add the user to the group and enable libvirtd with systemd.
```
sudo usermod -a -G libvirt $(whoami)
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
```

## Part 6 Configure QEMU
Now we edit qemu config file with ```sudo vim /etc/libvirt/qemu.conf``` and look for the lines:
```
#user = "root"
#group = "root"
```
And replace **root** with your username. Then we restart libvirtd and add the user to more groups.
```
sudo systemctl restart libvirtd

sudo usermod -a -G kvm $(whoami) 
sudo usermod -a -G libvirt $(whoami)
```

## Part 7 Create the VM
Now we are going to create our VM. Open virtual machine manager and create a new virtual machine.

Go through the setup by selecting a local install media and going forward. Then browse for the Windows 10 ISO file in your computer.

Continue through the setup until the last page where it asks for the name. Make sure the name is win10 and select the box that says ```Customize configuation before install```
![Screenshot from 2023-05-09 22-45-58](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/bbcdc47b-a896-4f34-8f0c-2c183af9e449)
