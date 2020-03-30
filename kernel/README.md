# Kernel Update Recipe
This recipe describes how to update the kernel shipped with CentOS 7.4 on which Rockclusters 7.0 is based.

## Step 1 - Get a slightly newer GNU C Compiler

CentOS 7.4 ships `gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-16)` which is pretty old and does not have the [retpoline patches](https://support.google.com/faqs/answer/7625886), required to counter the [Spectre security vulnerability](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)). As we are not on a cutting edge system it is sufficient to update GCC to release 7. At the time of writing this guide GCC 7.5.0 was the most recent version from this release series. Before compilation can start a small number of tools has to be installed.

```bash
yum install gmp-devel mpfr-devel libmpc-devel
```

Download, unpack, configure, make and install GCC 7.5.0 like given in the following commands. The compiler will be installed into default location at `/usr/local`.

```bash
cd /root
wget http://mirror.koddos.net/gcc/releases/gcc-7.5.0/gcc-7.5.0.tar.xz
tar xf gcc-7.5.0.tar.xz && cd gcc-7.5.0
./configure --disable-multilib
make -j4
make install
cd ..
rm -rf gcc-7.5.0
```

Open a new terminal and test availability of gcc:

```bash
[root@host ~]# which gcc
/usr/local/bin/gcc
[root@host ~]# gcc --version
gcc (GCC) 7.5.0
Copyright (C) 2017 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
[root@host ~]# 
```

Now we are ready to create a new set of kernel RPM-files for our Rocksclusters installation.

## Step 2 - Prepare SPEC file for kernel 5.4.28 (LTS)

The key is to use and to modify the SPEC file which the friends at ElRepo are using.  Before we can start some additional packages essentiell for kernel compilation need to be installed:

```bash
yum install flex asciidoc xmlto
```

Now go on with preparing the kernel sources and kernel configuration. Basically we extract some stuff form the ElRepo source RPM-file and discard the unnecessary files. Then we modify the SPEC file to our requirements. At the time of writing the current kernel version for 5.5 mainline kernel was 5.5.13. Modify following commands for newer kernel versions accordingly.

```bash
cd /root
wget https://elrepo.org/linux/kernel/el7/SRPMS/kernel-ml-5.5.13-1.el7.elrepo.nosrc.rpm
rpm2cpio kernel-ml-5.5.13-1.el7.elrepo.nosrc.rpm | cpio -dium
rm -f kernel-ml-5.5.13-1.el7.elrepo.nosrc.rpm config-5.5.13-x86_64
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.4.28.tar.xz
mkdir -p rpmbuild/SOURCES
mv cpupower.* linux-5.4.28.tar.xz /root/rpmbuild/SOURCES
cd /root/rpmbuild/SOURCES/
tar xf linux-5.4.28.tar.xz
cd linux-5.4.28
cp /boot/config-3.10.0-693.5.2.el7.x86_64 .config
make olddefconfig
make menuconfig
```

Make your adjustments to kernel configuration. For me personally I would like to get rid of the `Nouveau` kernel module because  the original closed-source driver made by NVIDIA shall be used. Therefore I chose: `Device Drivers -> Graphics support -> Nouveau (NVIDIA) cards [ ]`.

After finishing kernel configuration save the resulting `.config` file to the right place with the right name:

```bash
cp .config ../config-5.4.28-x86_64
cd ..
rm -rf linux-5.4.28
```

The contents of your SOURCE directory should now look like this:

```bash
[root@host ~]# ls -l /root/rpmbuild/SOURCES
total 107124
-rw-r--r-- 1 root root    188257 Mar 30 19:44 config-5.4.28-x86_64
-rw-rw-r-- 1 root root       150 Mar 25 17:16 cpupower.config
-rw-rw-r-- 1 root root       294 Mar 25 17:16 cpupower.service
-rw-r--r-- 1 root root 109498288 Mar 25 08:34 linux-5.4.28.tar.xz
[root@host ~]#
```

Now go on with modifying the SPEC file still located in /root:

```bash
cd /root
cp kernel-ml-5.5.spec kernel-5.4.28-1.spec
vim kernel-5.4.28-1.spec
```

Basically we need to take care of the right package naming. Therefore apply changes to the following lines: 

- Line 4: `%define LKAver 5.4.28`
- Line 7: `%define buildid .el7`
- Line 51: `%define with_doc 1`
- Line 63: `%define pkg_release 1%{?buildid}`
- Line 93: `Name: kernel`


The diff between both SPEC files should look like this, with 5.5.13 being the kernel currently used in ElRepos's original SPEC file:

```bash
[root@host ~]# diff kernel-ml-5.5.spec kernel-5.4.28-1.spec 
4c4
< %define LKAver 5.5.13
---
> %define LKAver 5.4.28
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

Now kick off kernel compilation and kernel rpm creation. This takes a while and does not produce a lot of terminal output because make is launched internally in silent mode `make -s`.

```bash
rpmbuild -bb kernel-5.4.28-1.spec --without perf
```

