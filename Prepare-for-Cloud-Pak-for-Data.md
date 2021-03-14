# Prepare for Cloud Pak for Data

## Hardware requirements

-  One computer which will be called **Installer** that runs Linux or MacOS.

## System requirements

- Have completed  [Prepare Redhat Openshift for Cloud Paks](https://github.com/bpshparis/sandbox/blob/master/Prepare-Redhat-Openshift-for-Cloud-Paks.md#prepare-redhat-openshift-for-cloud-paks)
- One **WEB server** where following files are available in **read mode**:
  - [cpd-cli-linux-EE-3.5.2.tgz](https://github.com/IBM/cpd-cli/releases/download/v3.5.2/cpd-cli-linux-EE-3.5.2.tgz)
  - [IBM® Cloud Pak for Data entitlement license API key](https://myibm.ibm.com/products-services/containerlibrary) saved in apikey file.
  - [repo.yaml](scripts/repo.yaml)

<br>
:checkered_flag::checkered_flag::checkered_flag:
<br>

## Prepare for Cloud Pak for Data

> :information_source: Commands below are valid for a **Linux/Centos 7**.

> :warning: Some of commands below will need to be adapted to fit Linux/Debian or MacOS .

### Changing load balancer timeout settings

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

#### Check

```
egrep -w 'timeout client|timeout server'  /etc/haproxy/haproxy.cfg
```

#### Change if necessary

> :bulb: The recommended timeout is at least 5 minutes.

```
TIMEOUT="5m"
```

```
sed -e "/timeout client/s/ [0-9].*/ 5m/" /etc/haproxy/haproxy.cfg
sed -e "/timeout server/s/ [0-9].*/ 5m/" /etc/haproxy/haproxy.cfg
```


### Changing CRI-O container settings on worker nodes

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
PIDS_LIMIT="16384"

oc get no -l node-role.kubernetes.io/worker --no-headers -o name | xargs -I {} --  oc debug {} -- bash -c 'chroot /host sed -i "s/^\(pids_limit\)\s\{0,\}=.*/\1 = '$PIDS_LIMIT'/" /etc/crio/crio.conf'

oc get no -l node-role.kubernetes.io/worker --no-headers -o name | xargs -I {} --  oc debug {} -- bash -c 'chroot /host grep  "^pids_limit" /etc/crio/crio.conf'
```

### Enable container_manage_cgroup on worker nodes

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
oc get no -l node-role.kubernetes.io/worker --no-headers -o name | xargs -I {} --  oc debug {} -- bash -c 'chroot /host setsebool -P container_manage_cgroup true'
```

### Changing kernel parameter settings on worker nodes

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

#### Check

```
KERNEL_PARMS="kernel.shmall|kernel.shmmax|kernel.shmmni|kernel.sem|kernel.msgmax|kernel.msgmnb|kernel.msgmni|vm.max_map_count"
```

```
for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S sysctl -a | egrep -w "'$KERNEL_PARMS'"'; done
```

#### Change if necessary

> :bulb: Settings below suits for a 64GB RAM worker

```
OCP="ocp9"
WORKERS="w1-$OCP w2-$OCP w3-$OCP"
NEW_VALUES="kernel.shmmni=32768 kernel.msgmax=65536 kernel.msgmnb=65536 kernel.msgmni=32768"
ROOT_PWD="password"
```

```
for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo passwd root --stdin'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; for value in '$NEW_VALUES'; do echo '$ROOT_PWD' | sudo -S sysctl -w $value; done'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S chmod 646 /etc/sysctl.conf; ls -Alhtr /etc/sysctl.conf'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S sed -i "/^[^#]/d" /etc/sysctl.conf'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; for value in '$NEW_VALUES'; do echo '$ROOT_PWD' | sudo -S echo $value | tee -a /etc/sysctl.conf; done'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S tail /etc/sysctl.conf'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S chmod 644 /etc/sysctl.conf; ls -Alhtr /etc/sysctl.conf'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S sysctl -p'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S sysctl -a | egrep -w "'$KERNEL_PARMS'"'; done
```


### Install the cpd command

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
WEB_SERVER_CP_URL="http://web/cloud-pak"
INST_FILE="cpd-cli-linux-EE-3.5.2.tgz"
CONF_FILE="repo.yaml"
INST_DIR=~/cpd && echo $INST_DIR
```

```
[ -d "$INST_DIR" ] && { rm -rf $INST_DIR; mkdir $INST_DIR; } || { mkdir $INST_DIR; }
cd $INST_DIR

wget -c $WEB_SERVER_CP_URL/$INST_FILE
tar xvzf $INST_FILE
rm $INST_FILE -f
wget -c $WEB_SERVER_CP_URL/$CONF_FILE
```

### Set repo.yaml

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer 

```
WEB_SERVER_CP_URL="http://web/cloud-pak"
APIKEY_FILE="apikey"
```

```
wget -c $WEB_SERVER_CP_URL/$APIKEY_FILE
USERNAME="cp" && echo $USERNAME
APIKEY=$(cat $APIKEY_FILE) && echo $APIKEY

```

#### Test your entitlement key against Cloud Pak registry

> :information_source: Run this on Installer 

```
REG="cp.icr.io/cp/cpd"
```

```
[ -z $(command -v podman) ] && { yum install podman runc buildah skopeo -y; } || echo "podman already installed"

podman login -u $USERNAME -p $APIKEY $REG
```

#### Add username and apikey to repo.yaml

> :information_source: Run this on Installer

```
sed -i -e 's/\(^\s\{4\}username:\).*$/\1 '$USERNAME'/' repo.yaml

sed -i -e 's/\(^\s\{4\}apikey:\).*$/\1 '$APIKEY'/' repo.yaml
```

### Download  Cloud Pak for Data resources definitions

> :warning: You have to be on line to execute this step.

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer 

```
INST_DIR=~/cpd
ASSEMBLY="lite"
ARCH="x86_64"
```

```
$INST_DIR/cpd-cli adm --repo $INST_DIR/repo.yaml --assembly $ASSEMBLY --arch $ARCH --accept-all-licenses 
```

> : bulb:  **$INST_DIR/cpd-cli-workspace** have been created and populated with yaml files.

### Download  Cloud Pak for Data images

> :warning: You have to be on line to execute this step.

> :warning: To avoid network failure, launch installation on locale console or in a screen

> :information_source: Run this on Installer

```
[ ! -z $(command -v screen) ] && echo screen installed || yum install screen -y

pkill screen; screen -mdS ADM && screen -r ADM
```

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
INST_DIR=~/cpd
ASSEMBLY="lite"
ARCH="x86_64"
```

```
$INST_DIR/cpd-cli preload-images --action download -a $ASSEMBLY --arch $ARCH --repo $INST_DIR/repo.yaml --accept-all-licenses
```

> :bulb:  Images have been copied in **$INST_DIR/cpd-cli-workspace/images/**

### Save Cloud Pak for Data Downloads to web server

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
INST_DIR=~/cpd
ASSEMBLY="lite"
ARCH="x86_64"
CPD_BIN="cpd-cli"
CPD_WKS="cpd-cli-workspace/"
WEB_SERVER="web"
WEB_SERVER_PATH="/web/cloud-pak/assemblies"
WEB_SERVER_USER="root"
WEB_SERVER_PASS="password"
VERSION=$(find $INST_DIR/cpd-cli-workspace/assembly/$ASSEMBLY/$ARCH/* -type d | awk -F'/' '{print $NF}')

[ ! -z "$VERSION" ] && echo $VERSION "-> OK" || echo "ERROR: VERSION is not set."
TAR_FILE="$ASSEMBLY-$VERSION-$ARCH.tar" && echo $TAR_FILE
```

```
cd $INST_DIR
tar cvf $TAR_FILE $CPD_BIN $CPD_WKS

[ -z $(command -v sshpass) ] && yum install -y sshpass || echo "sshpass already installed"

[ -z $(echo $SSHPASS) ] && export SSHPASS="WEB_SERVER_PASS" || echo "SSHPASS  already set"

sshpass -e scp -o StrictHostKeyChecking=no $TAR_FILE $WEB_SERVER_USER@$WEB_SERVER:$WEB_SERVER_PATH

sshpass -e ssh -o StrictHostKeyChecking=no $WEB_SERVER_USER@$WEB_SERVER "chmod -R +r $WEB_SERVER_PATH"

```
<br>
:checkered_flag::checkered_flag::checkered_flag:
<br>

