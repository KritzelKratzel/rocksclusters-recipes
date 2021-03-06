# Rocksclusters 7.0 Kernel Update Recipe
This recipe describes how to update the kernel shipped with Rockclusters 7.0. The final goal is to have a cluster installation with a state-of-the-art longterm kernel of release series 5.4.x on frontend node and on all compute nodes. However - typically you don't really have to do this if your Rocks 7.0 installation is upgraded to CentOS 7.7 (or [above](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/7.8_release_notes/index)) including most recent updates.

All the procedures outlined in this document have been extensively tested on a virtualized cluster using VMWare Fusion 11.5.3 running on a 2017 MacBook Pro. Use it on your own risk.

------

## Step 1a - Preparation for CentOS 7.4

**Notice:** This section is only necessary if you run the update procedure from a CentOS 7.4 based system. In case you have updated your system to CentOS 7.7 (like suggested [here](https://github.com/KritzelKratzel/rocksclusters-recipes/blob/master/install/README.md#step-2---upgrade--update-towards-centos-77)) go ahead with the next section. 

CentOS 7.4 ships `gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-16)` which unfortunately does not have the [retpoline patches](https://support.google.com/faqs/answer/7625886), required to counter the [Spectre security vulnerability](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)). As a result the kernel build procedure would fail with an error message of type "your compiler is too old". You can either try to rebuild a newer Red Hat GCC or simply use a vanilla GCC release series 7.x at lest. At the time of writing this guide [GCC 7.5.0](https://gcc.gnu.org/gcc-7/) was the most recent version from this release series. Before GCC build can start some development tools have to be installed.

```bash
yum install gmp-devel mpfr-devel libmpc-devel
```

Download, unpack, configure, make and install GCC 7.5.0 like given in the following commands. The compiler will be installed into a customized location under `/usr/local/gcc-7.5.0` because we need the new compiler just for the kernel build but definitely not for the following generation of the kernel-roll. This also leaves the frontend-node installation somewhat untouched.

```bash
cd /root
wget http://mirror.koddos.net/gcc/releases/gcc-7.5.0/gcc-7.5.0.tar.xz
tar xf gcc-7.5.0.tar.xz && cd gcc-7.5.0
./configure --prefix=/usr/local/gcc-7.5.0 --disable-multilib --enable-languages=c,c++
make
make install
cd ..
rm -rf gcc-7.5.0
```

Create a small shell script in order to be able to activate the new GCC 7.5.0 environment when needed.

```
vim /usr/local/gcc-7.5.0/env.sh
```

Add the following content:

```bash
export GCC7DIR=/usr/local/gcc-7.5.0
export PATH=$GCC7DIR/bin:$PATH
export MANPATH=$GCC7DIR/share/man:$MANPATH
export LD_LIBRARY_PATH=$GCC7DIR/lib64:$LD_LIBRARY_PATH
```

Now you have the choice to activate the new GCC just on request by applying the next command in a terminal window:

```bash
source /usr/local/gcc-7.5.0/env.sh
```

Test the availability of gcc:

```bash
[root@host ~]# which gcc
/usr/local/gcc-7.5.0/bin/gcc
[root@host ~]# gcc --version
gcc (GCC) 7.5.0
Copyright (C) 2017 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
[root@host ~]# 
```

At this point we are ready to create a new set of kernel RPM-files for the present Rocksclusters installation.

## Step 1b - Another Option

**Notice:** This section is only necessary if you run the update procedure from a CentOS 7.4 based system. In case you have updated your system to CentOS 7.7 (like suggested [here](https://github.com/KritzelKratzel/rocksclusters-recipes/blob/master/install/README.md#step-2---upgrade--update-towards-centos-77)) go ahead with the next section. 

Update compiler release 4.8.5-16 to 4.8.5-39 (includes [retpoline patches](https://support.google.com/faqs/answer/7625886))

```bash
wget http://vault.centos.org/7.7.1908/os/Source/SPackages/gcc-4.8.5-39.el7.src.rpm

yum groupinstall "Development Tools"
yum install dejagnu texinfo texinfo-tex gcc-gnat libgnat graphviz dblatex docbook5-style-xsl gmp-devel mpfr-devel libmpc-devel glibc-2.17-196.el7.i686 glibc-devel-2.17-196.el7.i686

rpmbuild --rebuild gcc-4.8.5-39.el7.src.rpm
```

This will take several hours and finally produces the following RPM-packages:

```bash
...
Wrote: /root/rpmbuild/RPMS/x86_64/gcc-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libgcc-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/gcc-c++-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libstdc++-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libstdc++-devel-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libstdc++-static-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libstdc++-docs-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/gcc-objc-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/gcc-objc++-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libobjc-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/gcc-gfortran-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libgfortran-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libgfortran-static-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libgomp-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libmudflap-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libmudflap-devel-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libmudflap-static-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libquadmath-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libquadmath-devel-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libquadmath-static-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libitm-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libitm-devel-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libitm-static-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libatomic-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libatomic-static-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libasan-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libasan-static-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libtsan-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libtsan-static-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/cpp-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/gcc-gnat-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libgnat-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libgnat-devel-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libgnat-static-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/gcc-go-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libgo-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libgo-devel-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/libgo-static-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/gcc-plugin-devel-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/gcc-debuginfo-4.8.5-39.el7.centos.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/gcc-base-debuginfo-4.8.5-39.el7.centos.x86_64.rpm
...
```

Copy those files into `/export/rocks/install/contrib/7.0/x86_64/RPMS/` , rebuild Rocks distribution and install:

```bash
cp -v rpmbuild/RPMS/x86_64/* /export/rocks/install/contrib/7.0/x86_64/RPMS/
cd /export/rocks/install/
rocks create distro
yum clean all
yum update
```

Now your compiler is at CentOS 7.7 level.

## Step 2 - Build kernel-5.4.28 (LTS) RPMs

The key is to use and to modify one of those SPEC files which the friends at [ElRepo](https://elrepo.org/linux/kernel/el7/SRPMS/) are using for their own kernel packaging. Before we can start some additional packages required for kernel compilation need to be installed:

```bash
yum install flex asciidoc xmlto
```

Now go on with preparing the kernel sources and kernel configuration. Basically we extract some stuff from the `kernel-ml-5.x` ElRepo source RPM-file and discard the unnecessary files. Then we modify the SPEC file to our requirements. At the time of writing the current kernel version for 5.6 mainline kernel was 5.6.2. Check the currently available source RPM at https://elrepo.org/linux/kernel/el7/SRPMS and modify following commands for newer kernel versions accordingly.

```bash
cd /root
wget https://elrepo.org/linux/kernel/el7/SRPMS/kernel-ml-5.6.2-1.el7.elrepo.nosrc.rpm
rpm2cpio kernel-ml-5.6.2-1.el7.elrepo.nosrc.rpm | cpio -dium
```

This will extract the following files:

```bash
-rw-rw-r-- 1 root root 210883  2. Apr 16:53 config-5.6.2-x86_64
-rw-rw-r-- 1 root root    150  2. Apr 16:53 cpupower.config
-rw-rw-r-- 1 root root    294  2. Apr 16:53 cpupower.service
-rw-r--r-- 1 root root 155170  2. Apr 17:16 kernel-ml-5.6.2-1.el7.elrepo.nosrc.rpm
-rw-rw-r-- 1 root root 113253  2. Apr 16:53 kernel-ml-5.6.spec

```

Now remove the unnecessary files. We just need the `spec`-file and both `cpupower`-files.

```bash
rm -f kernel-ml-5.6.2-1.el7.elrepo.nosrc.rpm config-5.6.2-x86_64
```

------

**Only for CentOS 7.4 with self-compiled GCC 7.5.0:** At this point be sure that your are using the GCC 7.5.0 and not the default compiler of CentOS 7.4 located in `/usr/bin`. If your are opening a new terminal window then apply first:

```bash
source /usr/local/gcc-7.5.0/env.sh
```

These settings are gone if you close the terminal. 

------

Now select a kernel of your choice, download its sources and prepare its configuration. Here I use the current longterm kernel `linux-5.4.30`, in case of other kernel versions change commands where necessary. 

```bash
cd /root
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.4.30.tar.xz
mkdir -p rpmbuild/SOURCES
mv cpupower.* linux-5.4.30.tar.xz /root/rpmbuild/SOURCES
cd /root/rpmbuild/SOURCES/
tar xf linux-5.4.30.tar.xz
cd linux-5.4.30
cp /boot/config-3.10.0-693.5.2.el7.x86_64 .config
make olddefconfig
make menuconfig
```

Make your adjustments to kernel configuration. For me personally I would like to get rid of the `Nouveau` kernel module because  the original closed-source driver made by NVIDIA shall be used laster in my cluster installation. In addition I want to have support for ExFAT drives as well. Therefore I chose: `Device Drivers -> Graphics support -> Nouveau (NVIDIA) cards [ ]` and `Device Drivers -> Staging Drivers -> exFAT fs support <M>`. After finishing kernel configuration save the resulting `.config` file to the right place with the right name and remove the unpacked kernel source directory, but not the tarball.

```bash
cp .config ../config-5.4.30-x86_64
cd ..
rm -rf linux-5.4.30
```

Now the contents of your `rpmbuild/SOURCE` directory should look like this:

```bash
[root@host ~]# ls -l /root/rpmbuild/SOURCES
total 107124
-rw-r--r-- 1 root root    188257 Mar 30 19:44 config-5.4.30-x86_64
-rw-rw-r-- 1 root root       150 Mar 25 17:16 cpupower.config
-rw-rw-r-- 1 root root       294 Mar 25 17:16 cpupower.service
-rw-r--r-- 1 root root 109498288 Mar 25 08:34 linux-5.4.30.tar.xz
[root@host ~]#
```

Proceed with modifying the SPEC file still located in /root. Notice the new naming scheme.

```bash
cd /root
cp kernel-ml-5.6.spec kernel-5.4.30-1.spec
vim kernel-5.4.30-1.spec
```

Basically we need to take care of the right package naming. Therefore apply changes to the following lines: 

- Line 4: `%define LKAver 5.4.30`
- Line 7: `%define buildid .el7`
- Line 51: `%define with_doc 1`
- Line 63: `%define pkg_release 1%{?buildid}`
- Line 93: `Name: kernel`


The diff between both SPEC files should look like this, with 5.5.13 being the kernel on which ElRepos's original SPEC file is based:

```diff
[root@host ~]# diff kernel-ml-5.6.spec kernel-5.4.30-1.spec 
4c4
< %define LKAver 5.6.2
---
> %define LKAver 5.4.30
7c7
< #define buildid .
---
> %define buildid .el7
51c51
< %define with_doc 0
---
> %define with_doc 1
63c63
< %define pkg_release 1%{?buildid}%{?dist}
---
> %define pkg_release 1%{?buildid}
93c93
< Name: kernel-ml
---
> Name: kernel
[root@host ~]# 
```

Now kick off kernel compilation and kernel rpm creation. This takes a while and does not produce much terminal output, because make is launched internally in silent mode `make -s`.

```bash
rpmbuild -ba kernel-5.4.30-1.spec --without perf
```

This command should create the following RPM-files:

```bash
...
Wrote: /root/rpmbuild/SRPMS/kernel-5.4.30-1.el7.nosrc.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/kernel-5.4.30-1.el7.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/kernel-devel-5.4.30-1.el7.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/kernel-doc-5.4.30-1.el7.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/kernel-headers-5.4.30-1.el7.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/kernel-tools-5.4.30-1.el7.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/kernel-tools-libs-5.4.30-1.el7.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/kernel-tools-libs-devel-5.4.30-1.el7.x86_64.rpm
...
```

The kernel source RPM file `/root/rpmbuild/SRPMS/kernel-5.4.30-1.el7.nosrc.rpm` is not important for the moment. More importantly, copy the seven binary RPM-files in `/root/rpmbuild/RPMS/x86_64` to `/export/rocks/install/contrib/7.0/x86_64/RPMS/` ...

```bash
cp -v /root/rpmbuild/RPMS/x86_64/kernel-* /export/rocks/install/contrib/7.0/x86_64/RPMS/
```

... and rebuild rocks distribution:

```bash
cd /export/rocks/install
rocks create distro
```

Finally check whether all new kernel-RPM files have been added successfully to rocks distribution:

```bash
[root@host install]# find rocks-dist -name "kernel*5.4.30*"
rocks-dist/x86_64/RedHat/RPMS/kernel-5.4.30-1.el7.x86_64.rpm
rocks-dist/x86_64/RedHat/RPMS/kernel-tools-5.4.30-1.el7.x86_64.rpm
rocks-dist/x86_64/RedHat/RPMS/kernel-doc-5.4.30-1.el7.x86_64.rpm
rocks-dist/x86_64/RedHat/RPMS/kernel-tools-libs-5.4.30-1.el7.x86_64.rpm
rocks-dist/x86_64/RedHat/RPMS/kernel-devel-5.4.30-1.el7.x86_64.rpm
rocks-dist/x86_64/RedHat/RPMS/kernel-headers-5.4.30-1.el7.x86_64.rpm
rocks-dist/x86_64/RedHat/RPMS/kernel-tools-libs-devel-5.4.30-1.el7.x86_64.rpm
[root@host install]#
```

**Only for CentOS 7.4 with self-compiled GCC 7.5.0:** From this step on we don't need the GCC 7.5.0 compiler anymore. Close the current terminal window and open another one to be sure to return to the original CentOS 7.4 compiler.

## Step 3 - Install new Kernel on Frontend Node

Now that we have freshly built kernel RPM-files we want to install them on the frontend node.

```bash
yum clean all
yum update
```

Reboot the frontend-node with the new installed kernel and check all kernel-related installed packages again:

```bash
[root@host install]# rpm -qa |grep "^kernel*"
kernel-devel-3.10.0-693.5.2.el7.x86_64
kernel-tools-libs-5.4.30-1.el7.x86_64
kernel-devel-3.10.0-1062.18.1.el7.x86_64
kernel-tools-5.4.30-1.el7.x86_64
kernel-devel-5.4.30-1.el7.x86_64
kernel-headers-5.4.30-1.el7.x86_64
kernel-5.4.30-1.el7.x86_64
kernel-3.10.0-1062.18.1.el7.x86_64
kernel-doc-5.4.30-1.el7.x86_64
kernel-3.10.0-693.5.2.el7.x86_64
kernel-devel-3.10.0-1062.el7.x86_64
kernel-3.10.0-1062.el7.x86_64
[root@host install]#
```

Remove the unnecessary kernel packages of 3.x release series:

```bash
yum remove kernel-devel-3.10.0-693.5.2.el7.x86_64 kernel-devel-3.10.0-1062.18.1.el7.x86_64 kernel-3.10.0-1062.18.1.el7.x86_64 kernel-3.10.0-693.5.2.el7.x86_64 kernel-devel-3.10.0-1062.el7.x86_64 kernel-3.10.0-1062.el7.x86_64
```

Now the list of installed kernel RPM-files on the frontend node should look like this:

```bash
[root@host ~]# rpm -qa | grep "^kernel"
kernel-devel-5.4.30-1.el7.x86_64
kernel-tools-5.4.30-1.el7.x86_64
kernel-5.4.30-1.el7.x86_64
kernel-tools-libs-5.4.30-1.el7.x86_64
kernel-doc-5.4.30-1.el7.x86_64
kernel-headers-5.4.30-1.el7.x86_64
[root@host ~]# 
```

Now list the version of the running kernel with `uname -srvo`.  The result should be something like this:

```
Linux 5.4.30-1.el7.x86_64 #1 SMP Fri Apr 3 16:16:43 CEST 2020 GNU/Linux
```

This step finalizes the kernel update on frontend node.

## Step 4 - Build a new Kernel Roll

Now we need to create a new kernel roll for updating the compute nodes. Clone the [kernel repository](https://github.com/rocksclusters/kernel) form Github and do the `bootstrap` and `make roll` steps. Internet connection is required, as a number of files will be downloaded and installed during bootstrap. 

**Only for CentOS 7.4 with self-compiled GCC 7.5.0:** Be sure of not using the previously compiled and installed GCC 7.5.0 - the default compiler set coming along with CentOS 7.4 must be used here.

```bash
cd /root
git clone https://github.com/rocksclusters/kernel.git
```

Change into the `kernel` directory and modify the `version.mk` file.

```bash
cd kernel/
vim version.mk
```

Increase the release counter like this:

```
RELEASE         = 3
```

This is a good procedure in order to make a difference to the unofficially released Rocks 7.0 kernel-7.0-**2** roll available at http://central-7-0-x86-64.rocksclusters.org/isos/testing/kernel-7.0-2.x86_64.disk1.iso. Then do the `bootstrap` and `make roll` steps.

------

**Notice**: On CentOS 7.7 and above the `make roll` command will presumably fail as the lorax root filesystem size of 2 GB has become too small. Fortunately on CentOS 7.7 lorax is new enough to counter this problem:

```bash
rpm -q --changelog lorax
...
* Tue Jun 04 2019 Brian C. Lane <bcl@redhat.com> 19.7.25-1
- lorax: Add --rootfs-size (bcl)
  Resolves: rhbz#1715116
...
```

Edit ``src/rocks-boot-7/Makefile``  and change the lorax call in line 84 from `lorax $(ISFINAL) ...`  to `lorax --rootfs-size 3 $(ISFINAL) ...` which extends the root filesystem size to 3 GB.

------

Now build the kernel roll:

```bash
./bootstrap.sh 2>&1 | tee /tmp/bootstrap-kernel-roll.out
make roll 2>&1 | tee /tmp/make-kernel-roll.out
```

This takes a while. When done, two ISO-image files will be created. 

```
[root@host kernel]# ls -l /root/kernel/*.iso
-rw-r--r-- 1 root root 1255145472 Mar 31 12:31 /root/kernel/kernel-7.0-3.x86_64.disk1.iso
-rw-r--r-- 1 root root  591396864 Mar 31 12:25 /root/kernel/rocks-installer-7-0.iso
[root@host kernel]#
```

Check whether the new kernel was actually included into the kernel roll by inspecting the log file:

```
[root@host kernel]# grep 5.4.30 /tmp/make-kernel-roll.out | grep installing
(569/732) [100%] installing kernel-5.4.30-1.el7.x86_64
[root@host kernel]#
```

Now add the new kernel roll while deactivating the old roll and create the rocks distribution:

```
rocks add roll kernel-7.0-3.x86_64.disk1.iso clean=y
cd /export/rocks/install
rocks create distro
yum clean all
```

The next `yum update` command would fail, because there is typically not enough space in `/boot` partition. Therefore remove by hand what would be updated anyway and run the update:

```bash
rm -rf /boot/kickstart/default/
yum update
```

This final command will update packages `rocks-boot` and `rocks-boot-pxe` on the frontend node.

The bootstrap step is somewhat invasive on the frontend node and alters it with installing some packages that might not be needed any further. This can be examined in the yum history.

```
[root@host install]# yum history
Loaded plugins: fastestmirror, langpacks
ID     | Login user               | Date and time    | Action(s)      | Altered
-------------------------------------------------------------------------------
     7 | root <root>              | 2020-03-31 15:01 | Update         |    2   
     6 | root <root>              | 2020-03-31 14:20 | Install        |    1   
     5 | root <root>              | 2020-03-31 14:12 | Install        |    2   
     4 | root <root>              | 2020-03-31 14:11 | Install        |    1   
     3 | root <root>              | 2020-03-31 14:11 | Install        |   19  <
     2 | root <root>              | 2020-03-31 14:01 | I, U           |    6 > 
     1 | System <unset>           | 2020-03-14 16:13 | Install        | 1834   
history list
[root@host install]# 
```

Here in this example ID 1 is the original system install, followed by ID2 which was the kernel update, whereas ID7 here is the last update of `rocks-boot` and `rocks-boot-pxe` on the frontend node. That means, steps 6 to 3 can be made undone.

```
yum history undo 6
yum history undo 5
yum history undo 4
yum history undo 3
```

You have finally made it! 💪🏻 Now you are ready to reinstall your compute-nodes with a state-of-the-art kernel and all previous problems with unsupported software or missing drivers are gone. If still not, you have at least all tools at hand to add or update other kernel modules to your kernel when needed.

# References

1. http://disbauxes.upc.es/code/analysis/send-a-newer-pxe-kernel-version-to-a-new-computer-node-on-rocks-cluster-6-2-without-altering-the-frontend/
2. https://lists.sdsc.edu/pipermail/npaci-rocks-discussion/2018-January/071263.html
3. https://elrepo.org/linux/kernel/el7/x86_64/RPMS/
4. http://central-7-0-x86-64.rocksclusters.org/roll-documentation/base/7.0/install-frontend-7.html
