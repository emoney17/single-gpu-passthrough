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
![Screenshot from 2023-05-09 22-45-58](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/c90b85f5-6b03-4db8-9cdd-dbebb6faa166)

In the overview section, make sure to select the Q35 Chipset and UEFI Firmware.
![Screenshot from 2023-05-09 22-59-10](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/9f4e5101-64e6-4b55-860f-643c1c575150)

Next go to the cpu section and in the Configuration section uncheck ```host-passthrough``` and look for ```host-model``` After doing some research it seems the Ryzen 5 CPU we have requires us to do this, otherwise we get a windows blue screen every time we boot the virtual machine.
![Screenshot from 2023-05-09 23-07-26](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/74a7f941-d257-4f39-8d10-46e34f022995)

You can now check the cpu and memory sections and modify them as you see fit.

Once you get to the boot options, we want to change the boot order so that the ISO comes first
![Screenshot from 2023-05-09 22-57-43](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/c61ea83e-e6a2-4fc2-867b-5cefba8e8b73)

Then hit the ```Add Hardware``` button at the bottom left because we are going to add another rom so that we can download some drivers.

First go to https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.215-2/ 

Then donwload the ```virtio-win.iso```

Now after pressing add hardware add a new storage device and make it a cd rom
![Screenshot from 2023-05-09 23-00-24](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/0ccdae4d-459b-412b-b4e0-968157a728e2)

After that go to the now named SATA CDROM 2 and browse for the virtio-win.iso you downloaded on your computer.

**BEFORE YOU PROCEED MAKE SURE ALL SETTINGS HAVE ACTUALLY BEEN APPLIED**

You should now be able to start the vm and begin the windows installation process!
![Screenshot from 2023-05-09 23-13-28](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/93d75f8f-5511-46cc-978f-484e5a8dd4b7)
