# GPU Passthrough from Arch Linux

##### Issues

If you have any issues with these steps please hit me up and I will try to fix them!


##### Combines these sources:

1. https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF

2. https://passthroughpo.st/quick-dirty-arch-passthrough-guide/

3. https://medium.com/@dubistkomisch/gaming-on-arch-linux-and-windows-10-with-vfio-iommu-gpu-passthrough-7c395dde5c2

4. https://pastebin.com/wetAhhVX

##### When to do this:

When you want to play windows 10 video games from your arch box because:
1. wangblows and you have access to a windows 10 iso 

2. you don't want to read mountains of text because you just want to play gaems. 

##### Required downloads:

1. a Windows 10 installation iso

**Link**: [here](https://www.microsoft.com/en-us/software-download/windows10ISO)

**Direct Download**:  [here](https://software-download.microsoft.com/pr/Win10_1809Oct_English_x64.iso?t=673fe9a0-8692-49ba-b0e0-e8ca7d314fdc&e=1544486586&h=9bb1b05b0fe6d83b41a5e8780a406244)

2. virtio* drivers for windows10 

**Link**: [here](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.160-1/)

**Direct Download**: [here](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.160-1/virtio-win-0.1.160.iso) 


##### Disclamer:

Most of this stuff is in the archlinux guide at the top, read more of that if any of this is confusing or something terribly goes wrong.

---


## PCI passthrough via OVMF (GPU)

### Initialization

1. Make sure that you have already enabled IOMMU via AMD-Vi or Intel Vt-d in your motherboard's BIOS 
HIT F10 or del or whatever the key is for your motherboard during bios initialization at beginning of startup, enable either VT-d if you have an Intel CPU or AMD-vi if you have an AMD CPU

2. edit `/etc/default/grub` and add intel_iommu=on to GRUB_CMDLINE_LINUX_DEFAULT

`$ sudo vim /etc/default/grub`

For AMD:

```
GRUB_CMDLINE_LINUX_DEFAULT="rd.driver.pre=vfio-pci loglevel=3 quiet"
GRUB_CMDLINE_LINUX="cryptdevice=UUID=0991889f-2a4b-4dfe-a076-719552f87ce3:cryptlvm rootfstype=ext4 amd_iommu=on"

```

3. re-configure your grub:

`$ sudo grub-mkconfig -o /boot/grub/grub.cfg`


4. reboot


`$ sudo reboot now`


### Isolating the GPU1:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204 [GeForce GTX 980] [10de:13c0] (rev a1)
	Subsystem: Micro-Star International Co., Ltd. [MSI] GM204 [GeForce GTX 980] [1462:3177]
	Kernel driver in use: vfio-pci
	Kernel modules: nouveau
01:00.1 Audio device [0403]: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)
	Subsystem: Micro-Star International Co., Ltd. [MSI] GM204 High Definition Audio Controller [1462:3177]
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel

and look through the given output until you find your desired GPU, they're **bold** in this case:

>0a:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104 [GeForce GTX 1080] [10de:1b80] (rev a1)
>0a:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)



### Configuring vfio-pci and Regenerating your Initramfs

Next, we need to instruct vfio-pci to target the device in question through the ID numbers gathered above.

1. edit `/etc/modprobe.d/vfio.conf` file and adding the following line with **your ids from the last step above**:

```
options vfio-pci ids=10de:13c0,10de:0fbb
```

Next, we will need to ensure that vfio-pci is loaded before other graphics drivers. 

2. edit `/etc/mkinitcpio.conf`. At the very top of your file you should see a section titled MODULES. Towards the bottom of this section you should see the uncommented line: MODULES= . Add the in the following order before any other drivers (nouveau, radeon, nvidia, etc) which may be listed: vfio vfio_iommu_type1 vfio_pci vfio_virqfd. The line should look like the following:

```
MODULES=(vfio_pci vfio vfio_iommu_type1)
```
Note As of linux 6.2 you dont need the vfio_virqfd


In the same file, also add modconf to the HOOKS line:
```
HOOKS=(base udev autodetect keyboard keymap modconf block encrypt filesystems fsck)
```

Warning!:

If you have encrpytion on your drive and you are on linux 6.0.6 you might have a frozen screen after your reboot (it might freezes when its asking about the passphrase)! To see if you have encrpyiton you can look at the linux check the grub file

Goto  `/etc/default/grub`
```
GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=/dev/nvme0n1p3:volgroup0:allow-discards loglevel=3 quiet amd_iommu=on iommu=pt"
```
3. rebuild initramfs.

`mkinitcpio -p linux`

or if you are running lts
`mkinitcpio -p linux-lts`



NOTE: If you need to have a different config for linux and linux-lts mkinitconfig file you can edit it here at this path: 

> sudo vim /etc/mkinitcpio.d/linux-lts.preset
ALL_config="/etc/mkinitcpio-lts.conf"


if you want to apply to all of your presets

`mkinitcpio -P`

4. reboot
`$ sudo reboot now`

### Checking whether it worked

1. check pci devices:

`$ lspci -nnk`

Find your GPU and ensure that under “Kernel driver in use:” vfio-pci is displayed:


```
0a:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104 [GeForce GTX 1080] [10de:1b80] (rev a1)
	Subsystem: Gigabyte Technology Co., Ltd Device [1458:3702]
	Kernel driver in use: vfio-pci
	Kernel modules: nouveau, nvidia_drm, nvidia
0a:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)
	Subsystem: Gigabyte Technology Co., Ltd Device [1458:3702]
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel

```

2. ???
3. profit

---
 
### Configuring OVMF and Running libvirt

1. download libvirt, virt-manager, ovmf, and qemu (these are all available in the AUR). OVMF is an open-source UEFI firmware designed for KVM and QEMU virtual machines. ovmf may be omitted if your hardware does not support it, or if you would prefer to use SeaBIOS. However, configuring it is very simple and typically worth the effort.

`sudo pacman -S libvirt virt-manager ovmf qemu`

2. edit `/etc/libvirt/qemu.conf` and add the path to your OVMF firmware image:

```
nvram = ["/usr/share/ovmf/ovmf_code_x64.bin:/usr/share/ovmf/ovmf_vars_x64.bin"]
```

3. start and enable both libvirtd and its logger, virtlogd.socket in systemd if you use a different init system, substitute it's commands in for systmectl start

```
$ sudo systemctl start libvirtd.service 
$ sudo systemctl start virtlogd.socket
$ sudo systemctl enable libvirtd.service
$ sudo systemctl enable virtlogd.socket
```
With libvirt running, and your GPU bound, you are now prepared to open up virt-manager and begin configuring your virtual machine. 
---

### virt-manager, a GUI for managing virtual machines

#### setting up virt-manager

**virt-manager** has a fairly comprehensive and intuitive GUI, so you should have little trouble getting your Windows guest up and running. 

1. download virt-manager

`$ sudo pacman -S virt-manager`

2. add yourself to the libvirt group (replace williampring with your username)

`$ sudo usermod -a -G libvirt williampring`

3. launch virt-manager

`$ virt-manager &`

4. when the VM creation wizard asks you to name your VM (final step before clicking "Finish"), check the "Customize before install" checkbox.

5. in the "Overview" section, set your firmware to "UEFI". If the option is grayed out, make sure that you have correctly specified the location of your firmware in /etc/libvirt/qemu.conf and restart libvirtd.service by running  `sudo systemctl restart libvirtd`

![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/uefi.png)

6. in the "CPUs" section, change your CPU model to "**host-passthrough**". If it is not in the list, you will have to type it by hand. This will ensure that your CPU is detected properly, since it causes libvirt to expose your CPU capabilities exactly as they are instead of only those it recognizes (which is the preferred default behavior to make CPU behavior easier to reproduce). Without it, some applications may complain about your CPU being of an unknown model.
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/cpu.png)


7. go into "Add Hardware" and add a Controller for **SCSI** drives of the "VirtIO SCSI" model.
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/virtioscsi.png)


8. then change the default IDE disk for a **SCSI** disk, which will bind to said controller.
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/scsi.png)


a. windows VMs will not recognize those drives by default, so you need to download the ISO containing the drivers from [here](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.160-1/virtio-win-0.1.160.iso) and add an **SATA** CD-ROM storage device linking to said ISO, otherwise you will not be able to get Windows to recognize it during the installation process.

9. make sure there is another **SATA** CD-ROM device that is handling your windows10 iso from the top links.
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/satavirtio.png)

10. setup your GPU, navigate to the “Add Hardware” section and select both the GPU and its sound device that was isolated previously in the **PCI** tab
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/gpu.png)
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/gpu-audio.png)

11. lastly, attach your usb keyboard
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/keyboard.png)
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/mouse.png)

12. don't forget to pass some good RAM as well
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/ram.png)

#### installing windows

1. test to see if it works by pressing the **play** button after configuring your VM and install windows

You may see this screen, just type `exit` and bo to the BIOs screen.
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/exit.jpg)

From the BIOs screen, select and `enter` the **Boot Manager**
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/select_boot.jpg)

Lastly, pick one of the DVD-ROM ones from these menus
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/select_dvd.jpg)

2. from here, you should be able to see windows 10 booting up, we need to load the **virtio-scsi** drivers

When you get to **Windows Setup** click `Custom: Install windows only (advanced)`
![alt text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/advanced_windows.JPG)

You should notice that our SCSI hard drive hasn't been detected yet, click `Load driver`
![alt_text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/load_drivers.jpg)

Select the correct CD-ROM labled `virto-win-XXXXX**`
![alt_text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/select_iso.jpg)

Finally, select the `amd64` architecture
![alt_text](https://github.com/williampring/GPU-zz

Check out my [virth xml file](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/virsh-win10.xml)

### CPU pinnging

#### CPU topology

1. check your cpu topology by running

`lscpu -e`

![alt_text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/lscpu.png)


### editing virsh

edit by running something similar with your desired editor and VM name:

`sudo EDITOR=vim virsh edit win10`

if this doesn't work, check your VM name:

`sudo virsh list`

![alt_text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/cpupinning.png)

your virsh config file should look something like this if your cpu is like mine, otherwise revert to the arch guide:
[cpu-pinning guide](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#CPU_pinning)

### enabling hugepages
1. edit `/etc/default/grub`

`$ sudo vim /etc/default/grub`

2. add `hugepages=2048` to **GRUB_COMMAND_LINE_DEFAULT**

your final grub should look like this:

![alt_text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/grub.png)

3. re-configure your grub:

`$ sudo grub-mkconfig -o /boot/grub/grub.cfg`

4. reboot and test it out


--- 

## Screen Frozen

### Soultion 1
1. Type your Passphraase like normal and Click enter
2. From here two things can happen either its still frozen or you will get the normal login screen asking for username and password
3. If its still frozen type your username click enter and type in your pass after and click enter 

Reference:

1. https://bbs.archlinux.org/viewtopic.php?id=280512

---
## Audio Working with Pipewire + Jack
1. pw-jack but this is not possible with libvert so download this

```
sudo pacman -S qemu-audio-jack
```

2. Edit file `/etc/libvirt/qemu.conf` 
```
user = "will"
```
3. In virt-manager and add Sound ich9
```
<sound model="ich9">
  <codec type="micro"/>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x1b" function="0x0"/>
</sound>
```

4. Add the follow block to device section change the connectPorts to your device: 
```
  <audio id="1" type="jack">
      <input clientName="win10" connectPorts="Yeti Stereo Microphone Analog Stereo:capture_F[LR]"/>
      <output clientName="win10" connectPorts="Schiit.*playback_F[LR]"/>
    </audio>
```
5. Download `qpwgraph` or `carla` to use the patch bay to see if the connection will be correct
```
sudo pacman -S qpwgraph carla
```
6. Set the directory and latency and put this block after the device and before domain
```
  <qemu:commandline>
    <qemu:env name="PIPEWIRE_RUNTIME_DIR" value="/run/user/1000"/>
    <qemu:env name="PIPEWIRE_LATENCY" value="512/48000"/>
  </qemu:commandline>
```
7. Boot the vm up and check the patch bay in qpwgraph or carla to see if the device appear and is connected correctly

![alt_text](https://github.com/williampring/GPU-Passthrough-Arch-Linux-to-Windows10/blob/master/pics/patchbay.png)

## Audio Working with Scream
1. Refer the arch linux: https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Passing_audio_from_virtual_machine_to_host_via_Scream

---
## Windows 11
1. ### You need enable secure boot for windows 11 to work
```
  <os>
    <type arch="x86_64" machine="pc-q35-7.1">hvm</type>
    <loader readonly="yes" secure="yes" type="pflash">/usr/share/edk2-ovmf/x64/OVMF_CODE.secboot.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/win11_VARS.fd</nvram>
    <boot dev="hd"/>
    <bootmenu enable="yes"/>
  </os>
```
2. ### You need enable TPM
`$ sudo pacman -Syu swtpm`
```
<tpm model="tpm-crb">
  <backend type="emulator" version="2.0"/>
</tpm>
```
---





