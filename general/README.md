# Tipps and Tricks

Loose collection of tips, tricks and related stuff. My general advice - practice, practice, practice. Get Rocks working as a virtual machine and train your administration skills before tweaking a real cluster installation.

This is work in progress.

Last edited: 2023-12-16

------

## NVIDIA Kernel Module via DKMS

Another way of getting a NVIDIA kernel module on a Rocksclusters compute-node without `cuda-roll` or lousy self-extracting `*.run` files. Just right for applications like [CST Studio Suite](https://www.3ds.com/de/produkte-und-services/simulia/produkte/cst-studio-suite/) on Linux compute clusters. Here in this case we use current driver version `535.54.03` for compute-nodes running CST Studio Suite Version 2024 and equipped with Tesla V100 or V100S GPU cards. The following procedure is useful in particular for corporate compute clusters with limited access to open-source software repositories as a result of tight firewall and proxy-server configurations.

See also: https://updates.cst.com/downloads/GPU_Computing_Guide_2024.pdf

------

Change into `/export/rocks/install/contrib/<version>/x86_64/RPMS`  (replace `<version>` with your actual Rocks version) and `wget` the following RPM files:

```bash
# From CentOS EPEL repository
wget http://ftp-stud.hs-esslingen.de/pub/epel/7/x86_64/Packages/d/dkms-3.0.12-1.el7.noarch.rpm
wget http://ftp-stud.hs-esslingen.de/pub/epel/7/x86_64/Packages/o/ocl-icd-2.2.12-1.el7.x86_64.rpm
wget http://ftp-stud.hs-esslingen.de/pub/epel/7/x86_64/Packages/o/opencl-filesystem-1.0-5.el7.noarch.rpm

# From NVIDIA developer repository
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/kmod-nvidia-latest-dkms-535.54.03-1.el7.x86_64.rpm
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/nvidia-driver-latest-535.54.03-1.el7.x86_64.rpm
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/nvidia-driver-latest-cuda-535.54.03-1.el7.x86_64.rpm
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/nvidia-driver-latest-cuda-libs-535.54.03-1.el7.x86_64.rpm
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/nvidia-driver-latest-devel-535.54.03-1.el7.x86_64.rpm
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/nvidia-driver-latest-libs-535.54.03-1.el7.x86_64.rpm
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/nvidia-driver-latest-NvFBCOpenGL-535.54.03-1.el7.x86_64.rpm
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/nvidia-driver-latest-NVML-535.54.03-1.el7.x86_64.rpm
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/nvidia-modprobe-latest-535.54.03-1.el7.x86_64.rpm
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/nvidia-persistenced-latest-535.54.03-1.el7.x86_64.rpm
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/nvidia-xconfig-latest-535.54.03-1.el7.x86_64.rpm
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/yum-plugin-nvidia-0.5-1.el7.noarch.rpm
```

Edit file `/export/rocks/install/site-profiles/<version>/nodes/extend-compute.xml`. If not yet existent, create it from `skeleton.xml`:

```bash
cd cd /export/rocks/install/site-profiles/<version>/nodes/
cp skeleton.xml extend-compute.xml
```

Add the following packages between `pre`  and `post` section:

```xml
...
</pre>

<!-- contrib packages -->
<package> dkms </package>
<package> kmod-nvidia-latest-dkms </package>
<package> nvidia-driver-latest </package>
<package> nvidia-driver-latest-cuda </package>
<package> nvidia-driver-latest-cuda-libs </package>
<package> nvidia-driver-latest-devel </package>
<package> nvidia-driver-latest-libs </package>
<package> nvidia-driver-latest-NvFBCOpenGL </package>
<package> nvidia-driver-latest-NVML </package>
<package> nvidia-modprobe-latest </package>
<package> nvidia-persistenced-latest </package>
<package> nvidia-xconfig-latest </package>
<package> ocl-icd </package>
<package> opencl-filesystem </package>
<package> yum-plugin-nvidia </package>

<!-- os roll packages determined by manual package dependency analysis -->
<package> elfutils-libelf-devel </package>
<package> libXdmcp </package>
<package> libXfont2 </package>
<package> libglvnd-gles </package>
<package> libglvnd-opengl </package>
<package> libvdpau </package>
<package> libxkbfile </package>
<package> vulkan-filesystem </package>
<package> xorg-x11-server-Xorg </package>
<package> xorg-x11-server-common </package>
<package> xorg-x11-xkb-utils </package>
<package> zlib-devel </package>

<post>
  /usr/bin/systemctl enable dkms
</post>
...
```

Rebuild rocks distribution and reinstall compute nodes:

```bash
cd /export/rocks/install
rocks create distro
rocks set host boot compute-X-Y action=install
rocks run host compute-X-Y reboot
```

Test kernel module after compute-node reinstall:

```
[root@compute-X-Y ~]# dkms status -m nvidia -v 535.54.03
nvidia/535.54.03, 3.10.0-1160.105.1.el7.x86_64, x86_64: installed
[root@compute-X-Y ~]# 
```

The `nouveau` kernel module is automatically blacklisted, see line 6 in `/etc/default/grub`. All 
required device nodes are created automatically, too.

```bash
[root@compute-X-Y ~]# nvidia-smi
<date and time>
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.54.03    Driver Version: 535.54.03    CUDA Version:          |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla V100-PCIE...  Off  | 00000000:3B:00.0 Off |                  Off |
| N/A   35C    P0    27W / 250W |      4MiB / 32510MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   1  Tesla V100S-PCI...  Off  | 00000000:5E:00.0 Off |                  Off |
| N/A   33C    P0    25W / 250W |      4MiB / 32510MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   2  Tesla V100S-PCI...  Off  | 00000000:AF:00.0 Off |                  Off |
| N/A   32C    P0    24W / 250W |      4MiB / 32510MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   3  Tesla V100-PCIE...  Off  | 00000000:D8:00.0 Off |                  Off |
| N/A   33C    P0    24W / 250W |      4MiB / 32510MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
[root@compute-X-Y ~]#
[root@compute-X-Y ~]# ll /dev/nvidia*
crw-rw-rw- 1 root root 195,   0 Jan 18 15:36 /dev/nvidia0
crw-rw-rw- 1 root root 195,   1 Jan 18 15:36 /dev/nvidia1
crw-rw-rw- 1 root root 195,   2 Jan 18 15:36 /dev/nvidia2
crw-rw-rw- 1 root root 195,   3 Jan 18 15:36 /dev/nvidia3
crw-rw-rw- 1 root root 195, 255 Jan 18 15:36 /dev/nvidiactl
crw-rw-rw- 1 root root 195, 254 Jan 18 15:36 /dev/nvidia-modeset
crw-rw-rw- 1 root root 235,   0 Jan 18 15:36 /dev/nvidia-uvm
crw-rw-rw- 1 root root 235,   1 Jan 18 15:36 /dev/nvidia-uvm-tools

/dev/nvidia-caps:
total 0
cr-------- 1 root root 238, 1 Jan 18 15:42 nvidia-cap1
cr--r--r-- 1 root root 238, 2 Jan 18 15:42 nvidia-cap2
[root@compute-X-Y ~]#
```
**Hint:**
If you get error messages in `/var/log/messages` like this ...

```bash
Apr 17 10:23:33 compute-X-Y kernel: NVRM: GPU 0000:d8:00.0: RmInitAdapter failed! (0x26:0xffff:1227)
Apr 17 10:23:33 compute-X-Y kernel: NVRM: GPU 0000:d8:00.0: rm_init_adapter failed, device minor number 3
```
... indicating that not all GPU cards on your compute-node have come up correctly, then power cycle
the compute-node by unplugging all power cords. Use `nvidia-smi -L` to check if all GPUs are present.

**Further reading:**

- https://github.com/shawfdong/hyades/wiki/DKMS-on-CentOS-7
- https://github.com/dell/dkms

## Missing cluster-kickstart-pxe

Apparently with Rocks 7.0 `/boot/kickstart/cluster-kickstart-pxe`  no longer exists. Therefore something like...

```bash
tentakel -g compute_linux /boot/kickstart/cluster-kickstart-pxe
```

... in order to kickstart all compute-nodes in one step is no longer possible. Use the following commands instead:

```bash
rocks set host boot compute action=install
rocks run host compute reboot
```

**Notice:** This procedure only works if boot sequence on compute-node shows `PXE` at first place.

Source: https://lists.sdsc.edu/pipermail/npaci-rocks-discussion/2018-September/072183.html

## SGE qmon Font Problem

Trying to run the graphical frontend `qmon` for SGE, but getting font errors like this?

```bash
[root@frontend-0-0 ~]# qmon 
Warning: Cannot convert string "-adobe-helvetica-medium-r-*--14-*-*-*-p-*-*-*" to type FontStruct
...
Warning: Cannot convert string "-adobe-helvetica-medium-r-*--10-*-*-*-p-*-*-*" to type FontStruct
X Error of failed request:  BadName (named color or font does not exist)
  Major opcode of failed request:  45 (X_OpenFont)
  Serial number of failed request:  1088
  Current serial number in output stream:  1099
[root@frontend-0-0 ~]# 
```

Simply install:

```bash
yum install xorg-x11-fonts-ISO8859-1-75dpi
```

Source: https://lists.sdsc.edu/pipermail/npaci-rocks-discussion/2017-December/071151.html

## Kernel-4.18.0 on Centos 7.7 ?

Surprisingly there is a kernel-series 4.18.0 source RPM-package available in the CentOS 7.7.1908 directory tree. This procedure aims towards building this kernel. Download the kernel source, run `rpmbuild --rebuild kernel-4.18.0-147.0.3.el7.src.rpm`  and it will tell you what is missing.

- Info 1: https://www.softwarecollections.org/en/scls/rhscl/llvm-toolset-7.0/
- Info 2: https://cbs.centos.org/koji/buildinfo?buildID=26092

Shell snippets:

```bash
wget http://vault.centos.org/7.7.1908/updates/Source/SPackages/kernel-4.18.0-147.0.3.el7.src.rpm

yum install hmaccalc python3-devel devtoolset-8-build devtoolset-8-binutils devtoolset-8-gcc devtoolset-8-make python3-rpm-macros flex "perl(ExtUtils::Embed)" java-devel numactl-devel python-docutils libcap-devel libcap-ng-devel llvm-toolset-7.0 pesign xmlto asciidoc

yum --enablerepo=extras install centos-release-scl
yum install llvm-toolset-7.0
yum install devtoolset-8-build devtoolset-8-make

wget https://cbs.centos.org/kojifiles/packages/rpm/4.13.0.2/1.el7.c8/src/rpm-4.13.0.2-1.el7.c8.src.rpm
yum install readline-devel nss-devel nss-softokn-freebl-devel file-devel lua-devel libacl-devel
rpmbuild --rebuild rpm-4.13.0.2-1.el7.c8.src.rpm
cp -v /root/rpmbuild/RPMS/x86_64/* /export/rocks/install/contrib/7.0/x86_64/RPMS/
cd /export/rocks/install/
rocks create distro
yum clean all
yum update

cd /root
rpmbuild --rebuild kernel-4.18.0-147.0.3.el7.src.rpm
```

The build apparently work, but I haven't had the time to let the build come to an end. In addition, a new kernel-roll would have to be set up and built, which is a completely different topic.

Roll Development Guide: https://www.rocksclusters.org/assets/usersguides/roll-documentation/developers-guide/6.0/

## Building kernel-roll on CentOS 7.7 needs LORAX-tweaking

Building an new kernel-roll for having latest drivers in `initramfs` at boot time is rather [straightforward](https://github.com/KritzelKratzel/rocksclusters-recipes/blob/master/kernel/README.md#step-4---build-a-new-kernel-roll):

```bash
cd /root
git clone https://github.com/rocksclusters/kernel.git
cd kernel/
vim version.mk
./bootstrap.sh 2>&1 | tee /tmp/bootstrap-kernel-roll.out
make roll 2>&1 | tee /tmp/make-kernel-roll.out
```

Surprisingly on CentOS 7.7 upgraded rocks installations the `make roll`  step fails nearly at the end with `No space left on device` complaints:

```bash
...
Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done   

cp: error writing '/var/tmp/lorax/lorax.imgutils.Pf2qIt/./usr/share/locale/zh_HK/LC_MESSAGES/iso_15924.mo': No space left on device
cp: failed to extend '/var/tmp/lorax/lorax.imgutils.Pf2qIt/./usr/share/locale/zh_HK/LC_MESSAGES/iso_15924.mo': No space left on device
...
```

It turns out that the default lorax root filesystem size of 2 GB is not enough anymore. See details of similar error messages [here](https://github.com/weldr/lorax/issues/384#issuecomment-400101118). However newer versions of `lorax`  support a new command line option `-rootfs-size`. Fortunately on CentOS 7.7 the installed version of `lorax` is new enough:

```bash
rpm -q --changelog lorax
...
* Tue Jun 04 2019 Brian C. Lane <bcl@redhat.com> 19.7.25-1
- lorax: Add --rootfs-size (bcl)
  Resolves: rhbz#1715116
...
```

Edit ``src/rocks-boot-7/Makefile``  and change the lorax call in line 84 from `lorax $(ISFINAL) ...`  to `lorax --rootfs-size 3 $(ISFINAL) ...` which extends the root filesystem size to 3 GB. Then run the `make roll`  step as outlined above.

An updated kernel roll based on kernel `3.10.0-1062.18.1` released on 2020-03-24 is available [here](https://drive.google.com/drive/folders/15YaHVG9lzqc7sb2XkF48p_7-3cJnj5tZ).

