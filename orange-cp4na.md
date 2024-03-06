```
[root@bdxmet237 ~]# sudo subscription-manager register

Registering to: subscription.rhsm.redhat.com:443/subscription
Username: iicparis
Password: 
The system has been registered with ID: 7b2f6b08-e8c7-45cf-8f45-d86f980b89e5
The registered system name is: bdxmet237.iicparis.fr.ibm.com
```

```
[root@bdxmet237 ~]# sudo subscription-manager attach --auto

Installed Product Current Status:
Product Name: Red Hat Enterprise Linux for x86_64
Status:       Subscribed
```



```
[root@bdxmet237 ~]# sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y

Updating Subscription Management repositories.
Last metadata expiration check: 0:01:54 ago on Tue 21 Mar 2023 02:35:31 PM CET.
epel-release-latest-8.noarch.rpm                                55 kB/s |  24 kB     00:00 Dependencies resolved.
```

```
[root@bdxmet237 ~]# ARCH=$( /bin/arch ) && echo $ARCH

[root@bdxmet237 ~]# sudo subscription-manager repos --enable "codeready-builder-for-rhel-8-${ARCH}-rpms"

Repository 'codeready-builder-for-rhel-8-x86_64-rpms' is enabled for this system.

[root@bdxmet237 ~]# sudo yum repolist epel

Updating Subscription Management repositories.
repo id                 repo name                                                       status
epel                    Extra Packages for Enterprise Linux 8 - x86_64                  enabled
```



```
yum update -y
cat /proc/cpuinfo | egrep "vmx|svm"
lscpu | grep Virtualization
yum install @virt -y
lsmod | grep kvm
virt-host-validate
yum -y install libvirt-devel virt-top libguestfs-tools
systemctl enable --now libvirtd
systemctl status libvirtd
yum -y install virt-manager virt-install

# https://slash-root.fr/network-manager-configuration-dun-reseau-bridge-pour-qemu-kvm/
nm-connection-editor

```



```
[ ! -f ~/.ssh/id_rsa ] && yes y | ssh-keygen -b 4096 -f ~/.ssh/id_rsa -N ""

PUB_KEY=$(cat ~/.ssh/id_rsa.pub) && echo ${PUB_KEY}

cat > core-authorized_keys.ign << EOF
{
  "ignition": {
    "version": "3.1.0"
  },
  "passwd": {
    "users": [
      {
        "name": "core",
        "sshAuthorizedKeys": [
          "${PUB_KEY}"
        ]
      }
    ]
  }
}
EOF

```



