# Cluster Install from Scratch

A little log of installing a brand new cluster from scratch with all updates and tweaks...

## Step 1 - Plain Vanilla Installation of Rocks 7.0

1. Download original [kernel roll](http://central-7-0-x86-64.rocksclusters.org/isos/kernel-7.0-0.x86_64.disk1.iso) and boot frontend with it.
2. Configure cluster to your needs. Basically follow the instructions in http://central-7-0-x86-64.rocksclusters.org/roll-documentation/base/7.0/install-frontend-7.html.
3. Select rolls for usage. In my case I typically choose `base`, `core`, `kernel`, `CentOS-7.4.1708`, `Updates-CentOS-7.4.1708`, `ganglia`, `sge`, `yumfix`. 
4. Kick-off frontend-node installation and lean back. Downloading all the rolls takes some time, as long as you haven't created a mirror [roll-server](https://github.com/rocksclusters/roll-server) nearby.

It is really a pity that only network installation is possible.

## Step 2 - Upgrade / Update towards CentOS 7.7

So now you have a working frontend node with a CentOS 7.4 software installation baselined in [September 2017](https://lists.centos.org/pipermail/centos-announce/2017-September/022532.html). As long as there is no new Rocks X.Y based on CentOS 8 or whatever (and probably there will be [no major update](https://marc.info/?l=npaci-rocks-discussion&m=158481906006702&w=2) in the foreseeable future) the least you can do is to upgrade everything up to CentOS 7.7 status. This is how it goes...

Create a new CentOS roll. Procedure based on:

- http://central-7-0-x86-64.rocksclusters.org/roll-documentation/base/7.0/update.html
- https://marc.info/?l=npaci-rocks-discussion&m=157077535409824&w=2

Choose a mirror next to you.

```bash
cd /root
mkdir centos-7-7 && cd centos-7-7
mirror=http://ftp.hosteurope.de/mirror/centos.org
osversion=7.7.1908
rocks create mirror ${mirror}/${osversion}/os/x86_64/Packages/ rollname=CentOS-${osversion}
```

This creates a new roll called...

```bash
-rw-r--r-- 1 root root 10416445440  1. Apr 21:58 CentOS-7.7.1908-7.0-0.x86_64.disk1.iso
```

Create a new CentOS-Updates roll:

```bash
cd /root
mkdir centos-7-7-update && cd centos-7-7-update
mirror=http://ftp.hosteurope.de/mirror/centos.org
osversion=7.7.1908
version=`date +%F`
rocks create mirror ${mirror}/${osversion}/updates/x86_64/Packages/ rollname=Updates-CentOS-${osversion} version=${version}
```

This creates an update roll with all updates as of your current date.

```
Updates-CentOS-7.7.1908-2020-04-01-0.x86_64.disk1.iso
```

