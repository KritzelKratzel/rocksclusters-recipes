# Tipps and Tricks

Loose collection of tips, tricks and related stuff. My general advice - practice, practice, practice. Get Rocks working as a virtual machine and train your administration skills before tweaking a real cluster installation.

This is work in progress.

Last edited: 2020-04-10

------

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

