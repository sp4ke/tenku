## gitian-host
##
## VERSION 0.1.0
##
## gitian host privileged container
## see also https://github.com/gdm85/tenku/tree/master/docker/gitian-host/README.md
##
#

## this image must be debootstrapped with build-wheezy.sh
FROM gdm85/wheezy

MAINTAINER Giuseppe Mazzotta "gdm85@users.noreply.github.com"

## some core packages
RUN apt-get update && apt-get install -y aptitude
RUN aptitude update && aptitude safe-upgrade -y
RUN aptitude install htop tmux less locales openssh-server -y

## these 2 files can be customized
COPY etc/timezone /etc/
COPY etc/locale.gen /etc/
RUN dpkg-reconfigure locales && dpkg-reconfigure tzdata

## continue installation of gitian-builder packages
RUN echo "deb http://http.debian.net/debian wheezy main contrib non-free" > /etc/apt/sources.list.d/non-free.list
RUN apt-get install -y iptables ca-certificates sudo bridge-utils git python-cheetah qemu-utils parted kpartx ruby apt-cacher-ng lxc parallel patch

## disable password logins, reverse DNS lookups & privilege separation (due to chroot issues on latest docker)
RUN sed -i -e 's/^#PasswordAuthentication yes/PasswordAuthentication no/' -e 's/^UsePrivilegeSeparation.*$/UsePrivilegeSeparation no/' /etc/ssh/sshd_config && echo 'UseDNS no' >> /etc/ssh/sshd_config

EXPOSE 22

# this version of Debian needs to run as root, so make sure
# that the build script can exectute it without providing a password
RUN echo "%sudo ALL=NOPASSWD: /usr/bin/lxc-start" > /etc/sudoers.d/gitian-lxc
RUN echo 'debian ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/debian-sudo

RUN useradd -m -s /bin/bash debian && adduser debian sudo && mkdir /home/debian/.ssh

## provide authorized keys to allow connections inside host
## NOTE: this is placed in debian user directory by setup.sh script
## reason is that there is a Docker glitch with file permissions when adding this
COPY authorized_keys /root/

WORKDIR /home/debian

## download in clear HTTP & check md5 right afterwards
RUN wget http://archive.ubuntu.com/ubuntu/pool/universe/v/vm-builder/vm-builder_0.12.4+bzr489.orig.tar.gz && echo "ec12e0070a007989561bfee5862c89a32c301992dd2771c4d5078ef1b3014f03  vm-builder_0.12.4+bzr489.orig.tar.gz" | sha256sum -c
RUN tar -zxvf vm-builder_0.12.4+bzr489.orig.tar.gz && cd vm-builder-0.12.4+bzr489 && python setup.py install

## these must always be loaded when you login as 'debian' user
COPY bash_profile /home/debian/.bash_profile

## script used to build base VMs
COPY bin/build-base-vms.sh /home/debian/

RUN git clone https://github.com/devrandom/gitian-builder.git && cd gitian-builder && git checkout  ad3f9cc4c2c8c0899961a366f5b9fbd1483b0ee3

COPY gitian.patch /home/debian/

## patch to allow paralle creation of VMs
RUN cd gitian-builder && patch -p1 < ../gitian.patch && rm ../gitian.patch
RUN chown -R debian.debian .bash_profile . && chown -R apt-cacher-ng.apt-cacher-ng /var/cache/apt-cacher-ng/

## suggested: run this image with mounted volumes for pre-downloaded inputs and apt-cacher-ng cache
## -v inputs:/home/debian/gitian-builder/inputs -v apt-cacher-ng:/var/cache/apt-cacher-ng
##NOTE: if you do use them, then not forget to set proper user rights for these mounted volumes

## all necessary setup instructions (SSH, LXC)
COPY bin/setup.sh /usr/local/bin/setup.sh

## on run failure, check logs to see if setup failed. on success connect via SSH
CMD /usr/local/bin/setup.sh && /etc/init.d/apt-cacher-ng start && /usr/sbin/sshd -D -e
