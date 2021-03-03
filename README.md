# VirtFS 9P filesystem passthrough in CentOS 7

This document details the steps for setting up VirtFS 9P "virtio-9pfs" (Plan 9 folder sharing over Virtio - I/O virtualization framework) filesystem passthrough between CentOS 7 guest and CentOS 7 host operating systems.

## Rebuild QEMU packages with VirtFS support

```
git clone https://github.com/AlekseyChudov/qemu-kvm-virtfs.git

cd qemu-kvm-virtfs

docker run -it --privileged -v "$PWD/rpmbuild:/root/rpmbuild" \
    docker.io/centos:7 /root/rpmbuild/build
```

Upon successful completion you will have the following packages
```
rpmbuild/RPMS/x86_64/qemu-img-virtfs-<version>.x86_64.rpm
rpmbuild/RPMS/x86_64/qemu-kvm-common-virtfs-<version>.x86_64.rpm
rpmbuild/RPMS/x86_64/qemu-kvm-tools-virtfs-<version>.x86_64.rpm
rpmbuild/RPMS/x86_64/qemu-kvm-virtfs-<version>.x86_64.rpm
rpmbuild/RPMS/x86_64/qemu-kvm-virtfs-debuginfo-<version>.x86_64.rpm
rpmbuild/SRPMS/qemu-kvm-virtfs-<version>.src.rpm
```

## Install QEMU packages on the hypervisor host

Install the previously created QEMU packages on the hypervisor host as usual, using the files or local Yum repository. During installation, new packages remove previously installed packages, if any, and correctly replace them due to RPM dependencies.

```
yum -y install centos-release-qemu-ev

yum -y install \
    cloud-utils \
    libvirt-client \
    libvirt-daemon-config-network \
    libvirt-daemon-config-nwfilter \
    libvirt-daemon-driver-interface \
    libvirt-daemon-driver-qemu \
    virt-install \
    rpmbuild/RPMS/x86_64/qemu-img-virtfs-<version>.x86_64.rpm \
    rpmbuild/RPMS/x86_64/qemu-kvm-common-virtfs-<version>.x86_64.rpm \
    rpmbuild/RPMS/x86_64/qemu-kvm-virtfs-<version>.x86_64.rpm

systemctl restart libvirtd
```

## Download the latest CentOS 7 cloud image and create a guest disk image

```
curl https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-2009.qcow2 \
    -o /var/lib/libvirt/images/CentOS-7-x86_64-GenericCloud-2009.qcow2

qemu-img resize /var/lib/libvirt/images/CentOS-7-x86_64-GenericCloud-2009.qcow2 10G

qemu-img convert -O qcow2 -o preallocation=falloc \
    /var/lib/libvirt/images/CentOS-7-x86_64-GenericCloud-2009.qcow2 \
    /var/lib/libvirt/images/centos7.localhost.qcow2
```

## Create a cloud-init ISO image to automate guest configuration

```
mkdir -p cloud-init

cat << EOF > cloud-init/meta-data
instance-id: centos7.localhost
local-hostname: centos7.localhost
EOF

cat << EOF > cloud-init/user-data
#cloud-config
chpasswd:
  expire: false
password: your-ssh-password-is-here
ssh_pwauth: true
runcmd:
  - yum -y update
  - yum -y --enablerepo centosplus install kernel-plus
  - grub2-set-default 0
  - echo "virtfs /mnt/virtfs 9p trans=virtio,version=9p2000.L,nofail,_netdev,x-mount.mkdir 0 0" >> /etc/fstab
  - reboot
EOF

genisoimage -input-charset utf-8 -joliet -rock -volid cidata \
    -output /var/lib/libvirt/images/centos7.localhost.iso cloud-init
```

## Create a folder to share with the guest

Note the DAC (user, group, access) and MAC (SELinux) permissions on the shared folder.

```
mkdir -p /var/lib/libvirt/qemu/virtfs/centos7.localhost

chown -R qemu:qemu /var/lib/libvirt/qemu/virtfs

chmod -R 0770 /var/lib/libvirt/qemu/virtfs
```

## Provision a new virtual machine

```
virt-install \
    --name centos7.localhost \
    --memory 1024 \
    --vcpus 1 \
    --cpu host \
    --import \
    --disk /var/lib/libvirt/images/centos7.localhost.qcow2,device=disk,bus=virtio \
    --disk /var/lib/libvirt/images/centos7.localhost.iso,device=cdrom \
    --filesystem /var/lib/libvirt/qemu/virtfs/centos7.localhost,virtfs \
    --network default \
    --graphics none \
    --virt-type kvm \
    --os-variant centos7.0
```

Default username is "centos" and default password is "your-ssh-password-is-here". Press "Ctrl + ]" to exit the guest console.

### Resources

[Filesystem passthrough "virtio-9pfs" support in KVM in RHEL7](https://access.redhat.com/discussions/1119043)

[Virtualization Deployment and Administration Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/index)

[Setting up cloud-init](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/installation_and_configuration_guide/setting_up_cloud_init)

[VirtFS](https://www.linux-kvm.org/page/VirtFS)

[v9fs: Plan 9 Resource Sharing for Linux](https://www.kernel.org/doc/Documentation/filesystems/9p.txt)
