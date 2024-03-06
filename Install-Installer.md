# Install Installer

## Hardware requirements

One Lenovo **X3550M5** or similar to host **1** virtual machine:

| name                         | role                      | vcpus | ram (GB) | storage (GB) | ethernet (10GB) |
| ---------------------------- | ------------------------- | ----- | -------- | ------------ | --------------- |
| cli-ocp5.iicparis.fr.ibm.com | load balancer + installer | 4     | 16       | 500          | 1               |
| **TOTAL**                    |                           | **4** | **16**   | **500**      | **1**           |


## System requirements

- One **VMware vSphere Hypervisor** [5.5](https://my.vmware.com/en/web/vmware/evalcenter?p=free-esxi5), [6.7](https://my.vmware.com/en/web/vmware/evalcenter?p=free-esxi6) or [7.0](https://my.vmware.com/en/web/vmware/evalcenter?p=free-esxi7) with **ESXi Shell access enabled**. VCenter is NOT required.

- Two **vmdk (centos.vmdk and centos-flat.vmdk)** file who host  a [minimal Centos 7](https://docs.centos.org/en-US/centos/install-guide/Simple_Installation/) **booting in dhcp**, **running VMware Tools** with **localhost.localdomain** as hostname. 

- One **DNS server**.
- One **DHCP server**.
- One **WEB server** where following files are available in **read mode**:
  - centos.vmdk
  - centos-flat.vmdk
  - [centos.vmx](scripts/centos.vmx)
  - [createCli.sh](scripts/createCli.sh)
  - [setHostAndIP.sh](scripts/setHostAndIP.sh)
  - [extendRootLV.sh](scripts/extendRootLV.sh)
  - [getVMAddress.sh](scripts/getVMAddress.sh)

:checkered_flag::checkered_flag::checkered_flag:

## Add DNS records

> :information_source: Commands below are valid for a **bind9** running on a **Ubuntu 16**

### Set environment

> :warning: Adapt settings to fit to your environment

> :information_source: Run this on DNS

```
DOMAIN=$(cat /etc/resolv.conf | awk '$1 ~ "search" {print $2}') && echo $DOMAIN
IP_HEAD="172.16"
OCP="ocp14"
CLI_IP=$IP_HEAD.187.140
MZONE=/var/lib/bind/$DOMAIN.hosts
RZONE=/var/lib/bind/$IP_HEAD.rev
```
### Add records to master zone

> :information_source: Run this on DNS

```
cat >> $MZONE << EOF
cli.$OCP.$DOMAIN.   IN      A       $CLI_IP
EOF
```

### Add records to reverse zone

> :information_source: Run this on DNS

```
cat >> $RZONE << EOF
$(echo $CLI_IP | awk -F. '{print $4 "." $3 "." $2 "." $1}').in-addr.arpa.    IN      PTR     cli.$OCP.$DOMAIN.
EOF
```

### Restart DNS service

> :information_source: Run this on DNS

```
service bind9 restart 
```

### Test master zone

> :information_source: Run this on DNS

```
for host in cli; do echo -n $host.$OCP "-> "; dig @localhost +short $host.$OCP.$DOMAIN; done
```

### Test reverse zone

> :information_source: Run this on DNS

```
for host in cli; do IP=$(dig @localhost +short $host.$OCP.$DOMAIN); echo -n $IP "-> "; dig @localhost +short -x $IP; done
```

:checkered_flag::checkered_flag::checkered_flag:


## Create Installer

### Download material

> :information_source: Run this on ESX

```
WEB_SERVER_VMDK_URL="http://web/vmdk"
WEB_SERVER_SOFT_URL="http://web/softs"
VMDK_PATH="/vmfs/volumes/datastore1/vmdk/"
```



```
[ ! -d "$VMDK_PATH" ] && mkdir "$VMDK_PATH" || echo "$VMDK_PATH" already exists
wget -c $WEB_SERVER_VMDK_URL/centos-gui-flat.vmdk -P $VMDK_PATH
wget -c $WEB_SERVER_VMDK_URL/centos-gui.vmdk -P $VMDK_PATH
wget -c $WEB_SERVER_VMDK_URL/rhel.vmx -P $VMDK_PATH
wget -c $WEB_SERVER_SOFT_URL/createCli.sh
```

### Create Installer

>:warning: Set **OCP**, **DATASTORE**, **VMS_PATH**, **CENTOS_VMDK** and **VMX** variables accordingly in **createCli.sh** before proceeding.

> :information_source: Run this on ESX

```
chmod +x ./createCli.sh
./createCli.sh
```

### Start Installer

> :information_source: Run this on ESX

```
PATTERN="cli"
vim-cmd vmsvc/getallvms | awk '$2 ~ "'$PATTERN'" && $1 !~ "Vmid" {print "vim-cmd vmsvc/power.on " $1}' | sh
vim-cmd vmsvc/getallvms | awk '$2 ~ "'$PATTERN'" && $1 !~ "Vmid" {print "vim-cmd vmsvc/power.getstate " $1}' | sh
```

### Get Installer dhcp address

> :information_source: Run this on ESX

```
CLI_DYN_ADDR="cli-addresse"
```

> :warning: Set **IP_HEAD** variables accordingly in **getVMAddress.sh** before proceeding.

> :information_source: Run this on ESX

```
wget -c $WEB_SERVER_SOFT_URL/getVMAddress.sh
chmod +x ./getVMAddress.sh
watch -n 5 "./getVMAddress.sh | tee $CLI_DYN_ADDR"
```

> :bulb: Wait for Cli to be up and display its dhcp address in the **3rd column**

> :bulb: Leave watch with **Ctrl + c**

### Configure Installer 

#### Download necessary stuff

> :information_source: Run this on ESX

```
WEB_SERVER_SOFT_URL="http://web/softs"

wget -c $WEB_SERVER_SOFT_URL/setHostAndIP.sh 
chmod +x setHostAndIP.sh
wget -c $WEB_SERVER_SOFT_URL/extendRootLV.sh
chmod +x extendRootLV.sh
```

#### Create and copy ESX public key to Installer

> :warning: To be able to ssh from ESX you need to enable sshClient rule outgoing port

> :information_source: Run this on ESX

```
esxcli network firewall ruleset set -e true -r sshClient
```

> :information_source: Run this on ESX

```
[ ! -d "/.ssh" ] && mkdir /.ssh || echo /.ssh already exists

/usr/lib/vmware/openssh/bin/ssh-keygen -t rsa -b 4096 -N "" -f /.ssh/id_rsa

for ip in $(awk -F ";" '{print $3}' $CLI_DYN_ADDR); do cat /.ssh/id_rsa.pub | ssh -o StrictHostKeyChecking=no root@$ip '[ ! -d "/root/.ssh" ] && mkdir /root/.ssh && cat >> /root/.ssh/authorized_keys'; done
```

#### Extend Installer Root logical volume

>:warning: Set **DISK**, **PART**, **VG** and **LV** variables accordingly in **extendRootLV.sh** before proceeding.

> :information_source: Run this on ESX

```
for ip in $(awk -F ";" '{print $3}' $CLI_DYN_ADDR); do echo "copying extendRootLV.sh to" $ip "..."; scp -o StrictHostKeyChecking=no extendRootLV.sh root@$ip:/root; done

for ip in $(awk -F ";" '{print $3}' $CLI_DYN_ADDR); do ssh -o StrictHostKeyChecking=no root@$ip 'hostname -f; /root/extendRootLV.sh'; done
```

#### Set Installer static ip address and reboot Installer

> :information_source: Run this on ESX

```
for ip in $(awk -F ";" '{print $3}' $CLI_DYN_ADDR); do echo "copy to" $ip; scp -o StrictHostKeyChecking=no setHostAndIP.sh root@$ip:/root; done

for LINE in $(awk -F ";" '{print $0}' $CLI_DYN_ADDR); do  HOSTNAME=$(echo $LINE | cut -d ";" -f2); IPADDR=$(echo $LINE | cut -d ";" -f3); echo $HOSTNAME; echo $IPADDR; ssh -o StrictHostKeyChecking=no root@$IPADDR '/root/setHostAndIP.sh '$HOSTNAME; done

for ip in $(awk -F ";" '{print $3}' $CLI_DYN_ADDR); do ssh -o StrictHostKeyChecking=no root@$ip 'reboot'; done
```

#### Check Installer static ip address

> :warning: Wait for cluster nodes to be up and display it static address in the **3rd column**

> :information_source: Run this on ESX

```
watch -n 5 "./getVMAddress.sh"
```

> :bulb: Leave watch with **Ctrl + c** 

### Set Installer environment

> :information_source: Run this on Installer

```
cat >> ~/.bashrc << EOF

export OCP=ocp14
export SSHPASS=spcspc
alias l='ls -Alhtr'

EOF

source ~/.bashrc
```

### :bulb: Optional: Disable security

> :information_source: Run this on Installer

```
systemctl stop firewalld &&
systemctl disable firewalld &&
setenforce 0 &&
sed -i -e 's/^SELINUX=\w*/SELINUX=disabled/' /etc/selinux/config
```

### Install vncviewer

> :information_source: Run this on Installer

```
VNC_PWD="spcspc"
```

```
[ -z $(command -v vncviewer) ] && sudo yum install -y tigervnc || echo "vncviewer already installed"
[ -z $(command -v vncpasswd) ] && sudo yum install -y tigervnc-server-minimal || echo "vncpasswd already installed"
[ -f ~/.vnc ] || mkdir ~/.vnc
echo $VNC_PWD | vncpasswd -f > ~/.vnc/passwd
```

### Install oc and kubectl and podman and sshpass

> :information_source: Run this on Installer

```
OC_URL="https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz"

[ -z $(command -v oc) ] && { wget -c $OC_URL; tar -xvzf $(echo $OC_URL | awk -F'/' '{print $NF}') -C $(echo $PATH | awk -F":" 'NR==1 {print $1}'); oc version --client; } || { echo "oc and kubectl already installed"; }

[ -z $(command -v podman) ] && { yum install podman runc buildah skopeo -y; } || echo "podman already installed"

sudo yum -y install sshpass screen

sudo yum -y install bash-completion
oc completion bash | sudo tee /etc/bash_completion.d/oc_completion
```

> :bulb: Logout and login bash for change to take effect.


<br>

:checkered_flag::checkered_flag::checkered_flag:

<br>

