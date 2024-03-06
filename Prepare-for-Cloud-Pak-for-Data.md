# Prepare for Cloud Pak for Data

## Hardware requirements

-  One computer which will be called **Installer** that runs Linux.

## System requirements

- Have completed  [Prepare Redhat Openshift for Cloud Paks](https://github.com/bpshparis/sandbox/blob/master/Prepare-Redhat-Openshift-for-Cloud-Paks.md#prepare-redhat-openshift-for-cloud-paks)
- Check latest [**cpd-cli**](https://github.com/IBM/cpd-cli/releases) release
- One **WEB server** where following files are available in **read mode**:
  - [Latest cpd-cli](https://github.com/IBM/cpd-cli/releases/download/v3.5.6/cpd-cli-linux-EE-3.5.6.tgz)
  - [IBM® Cloud Pak for Data entitlement license API key](https://myibm.ibm.com/products-services/containerlibrary) saved in apikey file.

<br>
:checkered_flag::checkered_flag::checkered_flag:
<br>

## Prepare for Cloud Pak for Data

> :information_source: Commands below are valid for a **Linux/Centos 7**.

> :warning: Some of commands below will need to be adapted to fit Linux/Debian or MacOS .

### Checking I/O performance

[howto](https://www.ibm.com/docs/en/cloud-paks/cp-data/3.5.0?topic=installation-checking-io-performance)

### Changing load balancer timeout settings

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Load Balancer

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
sed -i -e "/timeout client/s/ [0-9].*/ 5m/" /etc/haproxy/haproxy.cfg
sed -i -e "/timeout server/s/ [0-9].*/ 5m/" /etc/haproxy/haproxy.cfg

egrep -w 'timeout client|timeout server'  /etc/haproxy/haproxy.cfg

systemctl restart haproxy
```


### Changing CRI-O container settings on worker nodes

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
PIDS_LIMIT="16384"

oc get no -l node-role.kubernetes.io/worker --no-headers -o name | xargs -I {} --  oc debug {} -- bash -c 'chroot /host sed -i "s/^\(pids_limit\)\s\{0,\}=.*/\1 = '$PIDS_LIMIT'/" /etc/crio/crio.conf'

oc get no -l node-role.kubernetes.io/worker --no-headers -o name | xargs -I {} --  oc debug {} -- bash -c 'chroot /host grep  "^pids_limit" /etc/crio/crio.conf'
```

```
OCP="ocp11"
WORKERS="w1-$OCP w2-$OCP w3-$OCP" && echo $WORKERS
ROOT_PWD="password"
```

```
cat > ulimits << EOF
default_ulimits = [
        "nofile=66560:66560"
]
EOF
```

```
for node in $WORKERS; do scp -o StrictHostKeyChecking=no ulimits core@$node:/tmp; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo passwd root --stdin'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S sed -e "/^#default_ulimits = \[$/,/^#\]/!b" -e "/^#\]/!d; r /tmp/ulimits" -e "d" -i /etc/crio/crio.conf'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S grep -A2 "^default_ulimit" /etc/crio/crio.conf'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S systemctl restart crio'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S systemctl daemon-reload'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S systemctl status crio | egrep -w "Active:|crio.service"'; done
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
OCP="ocp9"
WORKERS="w1-$OCP w2-$OCP w3-$OCP"
KERNEL_PARMS="|kernel.shmall|kernel.shmmax|kernel.shmmni|kernel.sem|kernel.msgmax|kernel.msgmnb|kernel.msgmni|vm.max_map_count"
```

```
for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S sysctl -a | egrep -w "'$KERNEL_PARMS'"'; done
```

#### Change if necessary

> :bulb: Settings below suits for a 64GB RAM worker. Find explanations [here](https://www.ibm.com/docs/en/db2/11.5?topic=unix-kernel-parameter-requirements-linux)

```
RAM_GB=128 && echo RAM_GB=${RAM_GB}
[[ $(getconf PAGESIZE 2>/dev/null) =~ ^[0-9]+$ ]] && PAGESIZE=$(getconf PAGESIZE) || PAGESIZE=4096
echo PAGESIZE=${PAGESIZE}

((SHMMNI=256 * ${RAM_GB}))
[ ${SHMMNI} -gt $(sysctl -n kernel.shmmni) ] && SHMMNI=${SHMMNI} || SHMMNI=$(sysctl -n kernel.shmmni)
echo SHMMNI=${SHMMNI}

((SHMMAX=${RAM_GB} * 1024 * 1024 * 1024)) 
[ ${SHMMAX} -gt $(sysctl -n kernel.shmmax) ] 2>/dev/null && SHMMAX=${SHMMAX} || SHMMAX=$(sysctl -n kernel.shmmax)
echo SHMMAX=${SHMMAX}

((SHMALL=2 * (${SHMMAX} / ${PAGESIZE}))) && echo SHMALL=${SHMALL}
[ ${SHMALL} -gt $(sysctl -n kernel.shmall) ] 2>/dev/null && SHMALL=${SHMALL} || SHMALL=$(sysctl -n kernel.shmall)
echo SHMMAX=${SHMMAX}

SEMMSL=250
[ ${SEMMSL} -gt $(sysctl -n kernel.sem  | awk '{print $1}') ] 2>/dev/null && SEMMSL=${SEMMSL} || SEMMSL=$(sysctl -n kernel.sem  | awk '{print $1}')
echo SEMMSL=${SEMMSL}

SEMMNS=256000
[ ${SEMMNS} -gt $(sysctl -n kernel.sem  | awk '{print $2}') ] 2>/dev/null && SEMMNS=${SEMMNS} || SEMMNS=$(sysctl -n kernel.sem  | awk '{print $2}')
echo SEMMNS=${SEMMNS}

SEMOPM=32
[ ${SEMOPM} -gt $(sysctl -n kernel.sem  | awk '{print $3}') ] 2>/dev/null && SEMOPM=${SEMOPM} || SEMOPM=$(sysctl -n kernel.sem  | awk '{print $3}')
echo SEMOPM=${SEMOPM}

((SEMMNI=256 * ${RAM_GB}))
[ ${SEMMNI} -gt $(sysctl -n kernel.sem  | awk '{print $4}') ] 2>/dev/null && SEMMNI=${SEMMNI} || SEMMNI=$(sysctl -n kernel.sem  | awk '{print $4}')
echo SEMMNI=${SEMMNI}

echo SEM=${SEMMSL} ${SEMMNS} ${SEMOPM} ${SEMMNI}

((MSGMNI=1024 * ${RAM_GB}))
[ ${MSGMNI} -gt $(sysctl -n kernel.msgmni) ] 2>/dev/null && MSGMNI=${MSGMNI} || MSGMNI=$(sysctl -n kernel.msgmni)
echo MSGMNI=${MSGMNI}

MSGMAX=65536 && echo MSGMAX=${MSGMAX}

MSGMNB=65536 && echo MSGMNB=${MSGMNB}

cat << EOF | oc apply -f -
apiVersion: tuned.openshift.io/v1
kind: Tuned
metadata:
  name: cp4d-wkc-ipc
  namespace: openshift-cluster-node-tuning-operator
spec:
  profile:
  - name: cp4d-wkc-ipc
    data: |
      [main]
      summary=Tune IPC Kernel parameters on OpenShift Worker Nodes running WKC Pods
      [sysctl]
      kernel.shmall = 33554432
      kernel.shmmax = 68719476736
      kernel.shmmni = 16384
      kernel.sem = 250 1024000 100 32768
      kernel.msgmax = 65536
      kernel.msgmnb = 65536
      kernel.msgmni = 32768
      vm.max_map_count = 262144
  recommend:
  - match:
    - label: node-role.kubernetes.io/worker
    priority: 10
    profile: cp4d-wkc-ipc
EOF



NEW_VALUES="kernel.shmmni=32768 kernel.msgmax=65536 kernel.msgmnb=65536 kernel.msgmni=32768"
ROOT_PWD="password"
```

```
for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo passwd root --stdin'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S chmod 646 /etc/sysctl.conf; ls -Alhtr /etc/sysctl.conf'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S sed -i "/^[^#]/d" /etc/sysctl.conf'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; for value in '$NEW_VALUES'; do echo '$ROOT_PWD' | sudo -S echo $value | tee -a /etc/sysctl.conf; done'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S tail /etc/sysctl.conf'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S chmod 644 /etc/sysctl.conf; ls -Alhtr /etc/sysctl.conf'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S sysctl -p'; done

for node in $WORKERS; do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; echo '$ROOT_PWD' | sudo -S sysctl -a | egrep -w "'$KERNEL_PARMS'"'; done
```

<!--
https://docs.openshift.com/container-platform/4.6/nodes/nodes/nodes-nodes-managing.html
-->


### Install the cpd command

<!-- https://www.ibm.com/docs/en/cloud-paks/cp-data/3.5.0?topic=installing -->

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
WEB_SERVER_CP_URL="http://web/cloud-pak"
INST_FILE="cpd-cli-linux-EE-3.5.6.tgz"
INST_DIR=~/cpd && echo $INST_DIR
```

```
[ -d "$INST_DIR" ] && { rm -rf $INST_DIR; mkdir $INST_DIR; } || { mkdir $INST_DIR; }
cd $INST_DIR

wget -c $WEB_SERVER_CP_URL/$INST_FILE
tar xvzf $INST_FILE
rm $INST_FILE -f

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
INST_DIR=$(pwd) && echo $INST_DIR
ASSEMBLY="lite" && echo $ASSEMBLY
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

pkill screen; screen -mdS CPD && screen -r CPD
```

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
INST_DIR=$(pwd) && echo $INST_DIR
ASSEMBLY="lite" && echo $ASSEMBLY
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
CPD_PLUGINS="plugins/"
CPD_LICENSES="LICENSES/"
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
tar cvf $TAR_FILE $CPD_BIN $CPD_WKS $CPD_PLUGINS $CPD_LICENSES

[ -z $(command -v sshpass) ] && yum install -y sshpass || echo "sshpass already installed"

[ -z $(echo $SSHPASS) ] && export SSHPASS="WEB_SERVER_PASS" || echo "SSHPASS  already set"

sshpass -e scp -o StrictHostKeyChecking=no $TAR_FILE $WEB_SERVER_USER@$WEB_SERVER:$WEB_SERVER_PATH

sshpass -e ssh -o StrictHostKeyChecking=no $WEB_SERVER_USER@$WEB_SERVER "chmod -R +r $WEB_SERVER_PATH"

```
<br>
:checkered_flag::checkered_flag::checkered_flag:
<br>

