# single GPU passthrough guide AMD
### Info
- OS Ubuntu
- AMD Radeon RX 580
- AMD Ryzen 5 1600 (12)
- 16GB RAM

## Part 1 BIOS Config
**Before starting, make sure your BIOS is up to date.**

I ran into a lot of problems setting this up and most of them were fixed by updating...

Then make sure virtualization is enabled in your BIOS.
- IOMMU Enabled
- SVM Enabled

Also make sure to update your system.

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

(Use windows 10 Pro for less hassle)

![Screenshot from 2023-05-09 23-13-28](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/93d75f8f-5511-46cc-978f-484e5a8dd4b7)

Once the install is finished, make sure to remove the cd rom from the boot order in the settings since we wont need it anymore.

![Screenshot from 2023-05-09 23-20-46](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/521fc541-6cf0-4e99-9d5d-f69cb819eb0b)

After removing it start up the vm again and you should load into windows 10 startup.

Once you finish the first time start up (its so slow) go into your documents and look for the cd rom we loaded. It should be called virtio-win.

![Screenshot from 2023-05-09 23-51-49](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/75b08cb8-d9f3-4efa-84db-4ba3c72cab6f)

Click into it and run the ```virtio-win-guest-tools``` installer and let it run.

![Screenshot from 2023-05-09 23-53-34](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/04dbe847-7090-46c5-b936-ab9b7bb5deda)

## Part 8 Add GPU and USB hardware
At this point everything should have went smoothly. This is where it all goes down.

First we are going to add the GPU to the VM. Go to the Add Hardware button and look for PCI Host Device. Remember the two addresses we got when searching for our GPU? (**08:00.0** and **08:00.1**) We are going to look for these nubers in the list and they should be the GPU and Audio addresses. Add both of them to the VM.

Next we are going to add hardware again but this time USB Host Devices. We are going to add two, one for our mouse and one for the keyboard. You should be able to find them when selecting a USB device from the list.

![Screenshot from 2023-05-10 00-01-50](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/8cb0f2bd-e8b6-4fdd-b776-0f364f40b92c)

After adding everything we can delete all display items from the list such as the ```Display Spice``` and the ```Video QXL``` since we will be using gpu passthrough there is no need for these display options.

![Screenshot from 2023-05-10 00-03-49](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/24181b78-26c1-419b-8394-dc2e65421710)

## Part 9 Hooks
Now we need to set up hooks to disable our linux display manager when entering the vm and re-enable when exiting the vm. We are going to use [Risingprism's hook scripts](https://gitlab.com/risingprismtv/single-gpu-passthrough) to do this. To clone the  repository run:

```
git clone https://gitlab.com/risingprismtv/single-gpu-passthrough
```

```cd``` into the repo and run ```sudo ./install_hooks.sh``` to install the scripts in the proper locations.

Keep in mind, these scripts are assuming that your vm is named **win10**. If it isnt, either rename your vm or change the name in ```single-gpu-passthrough/hooks/qemu``` from win10 to whatever you named your vm.

## Part 10 GPU ROM File
The final step. This is where we find out if anything works.

We are gonna search for our GPU model at https://www.techpowerup.com/vgabios/ and get our rom file. Since I have done this before I have the correct rom file in the repo.

Download the rom and run these commands to get permissions and move it to the right location.
```
sudo chmod -R 775 Sapphire.RX580.8192.180719.rom
sudo chown $(whoami):$(whoami) Sapphire.RX580.8192.180719.rom
sudo mkdir /usr/share/vgabios
sudo mv Sapphire.RX580.8192.180719.rom /usr/share/vgabios/
```

Now that the rom is in place. We are going to edit the xml in the vm for the gpu.

Click on the PCI port for the gpu (not the audio part) and move to the xml section.

You will have to enable editing xml in the preferences of virtmanager.

In between the closing source tag and the address we are going to insert the info on our rom file
```
<rom file="/usr/share/vgabios/Sapphire.RX580.8192.180719.rom"/>
```
![Screenshot from 2023-05-10 00-17-43](https://github.com/emoney17/single-gpu-passthrough/assets/122418017/8f034270-5d85-412e-a88a-936267dc99cf)


And thats it...

After saving all the changes if you hit run on the vm it should black out and disable your display for a little bit and then start booting into windows.
After getting into windows I advise you to install the gpu drivers from your manufacturer's website so that is the real final step.
When shutting down the vm, after a bit the vm should eventually send you back to your linux display managers login screen.

## Troubleshooting
### Windows blue screen crashing on vm boot
If you are using a AMD Ryzen 5 1600 like me, you need to go into the vm's cpu settings and uncheck 

```copy host CPU configuration (host-passthrough)``` and select ```host-model```

Not sure if it is a compatabiltiy issue but it instantly let me boot into the windows installer after changing this.
### Black screen on starting vm
I was getting this error constantly the first time I added the gpu to my vm. 

After a lot of forum searching I decided to update my BIOS from my motherboard manufacturer's site and after trying it actually boot into windows first try, so try that.
