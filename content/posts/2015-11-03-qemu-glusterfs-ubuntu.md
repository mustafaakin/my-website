---
title: Compiling QEMU with GlusterFS Support on Ubuntu 14.04
date: 2015-11-03
---

On Ubuntu 14.04 (and maybe later) Qemu does not come with GlusterFS support. To compile it, you have to fetch the sources and enable flags. You also need to get the build dependencies, and some debian package tools.

You can install the dependencies as follows:

```bash
$ sudo apt-get install build-essential glusterfs-common devscripts dpkg-dev
$ sudo apt-get build-dep qemu
```

You also need to execute the following command, but it might be better to create a script file and execute it:

```bash
PACKAGE="qemu"  
PACKAGEDIR="/home/mustafa/gfs/build/${PACKAGE}/"  
DEBFULLNAME="Mustafa Akin"  
DEBEMAIL="mustafa91@gmail.com"  
PACKAGE_IDENTIFIER="identifier"  
DEBCOMMENT="with glusterfs support"

export DEBFULLNAME=${DEBFULLNAME}  
export DEBEMAIL=${DEBEMAIL}  
test -d ${PACKAGEDIR} && rm -r ${PACKAGEDIR}  
mkdir -p ${PACKAGEDIR}  
cd ${PACKAGEDIR}  
apt-get source ${PACKAGE}  
cd $(find ${PACKAGEDIR} -maxdepth 1 -mindepth 1 -type d -name "*${PACKAGE}*")/debian  
debchange -l ${PACKAGE_IDENTIFIER} ${DEBCOMMENT} -D trusty  
cp control control.org  
sed 's#\#\#--enable-glusterfs todo#\# --enable-glusterfs\n glusterfs-common,#g' < control.org > control  
rm control.org  
debuild -us -uc -i -I
```

After the execution, you will have the required *deb files and you can install them with `sudo dpkg -i *.deb`


