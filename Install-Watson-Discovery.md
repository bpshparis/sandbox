# Install Watson Discovery

## Hardware requirements

-  One computer which will be called **Installer** that runs Linux or MacOS.

## System requirements

- Have completed  [Prepare for Watson Discovery](https://github.com/bpshparis/sandbox/blob/master/Prepare-for-Watson-Discovery.md#prepare-for-watson-discovery)
- One **WEB server** where following files are available in **read mode**:
  - [watson-discovery-2.1.3-x86_64.tar](https://github.com/bpshparis/sandbox/blob/master/Prepare-for-Watson-Discovery.md#save-watson-discovery-downloads-to-web-server)

<br>
:checkered_flag::checkered_flag::checkered_flag:
<br>

## Install Watson Discovery

> :information_source: Commands below are valid for a **Linux/Centos 7**.

> :warning: Some of commands below will need to be adapted to fit Linux/Debian or MacOS .

### Log in OCP

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer 

```
LB_HOSTNAME="cli-ocp5"
NS="cpd"
```

```
oc login https://$LB_HOSTNAME:6443 -u admin -p admin --insecure-skip-tls-verify=true -n $NS
```

### Copy Watson Discovery downloads from web server

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer 

```
INST_DIR=~/cpd
ASSEMBLY="watson-discovery"
VERSION="2.1.3"
ARCH="x86_64"
TAR_FILE="$ASSEMBLY-$VERSION-$ARCH.tar"
WEB_SERVER_CP_URL="http://web/cloud-pak/assemblies"
```

```
cd ~
[ -d "$INST_DIR" ] && { rm -rf $INST_DIR; mkdir $INST_DIR; } || mkdir $INST_DIR
cd $INST_DIR

mkdir bin && cd bin
wget -c $WEB_SERVER_CP_URL/$TAR_FILE
tar xvf $TAR_FILE
rm -f $TAR_FILE
```

### Push Watson Discovery images to Openshift registry

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
ASSEMBLY="watson-discovery"
ARCH="x86_64"
VERSION=$(find $INST_DIR/bin/cpd-linux-workspace/assembly/$ASSEMBLY/$ARCH/* -type d | awk -F'/' '{print $NF}')

[ ! -z "$VERSION" ] && echo $VERSION "-> OK" || echo "ERROR: VERSION is not set."

```

```
podman login -u $(oc whoami) -p $(oc whoami -t) $(oc registry info)

$INST_DIR/bin/cpd-linux preloadImages \
--assembly $ASSEMBLY \
--version $VERSION \
--arch $ARCH \
--action push \
--transfer-image-to $(oc registry info)/$(oc project -q) \
--target-registry-password $(oc whoami -t) \
--target-registry-username $(oc whoami) \
--load-from $INST_DIR/bin/cpd-linux-workspace \
--accept-all-licenses
```


### Create Watson Discovery resources on cluster

> :information_source: Run this on Installer

```
$INST_DIR/bin/cpd-linux adm \
--namespace $(oc project -q) \
--assembly $ASSEMBLY \
--version $VERSION \
--arch $ARCH \
--load-from $INST_DIR/bin/cpd-linux-workspace \
--apply \
--accept-all-licenses
```

### Override values for Watson Discovery installation

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
SECRET=$(oc get secrets | grep default-dockercfg | awk '{print $1}') && echo $SECRET
TYPE="Development"
LANG="french"
OVERRIDE=$(find $INST_DIR/bin/cpd-linux-workspace/modules -type f -exec grep -q "portworx" '{}' \; -print)
[ ! -f "$OVERRIDE" ] && echo "ERROR: OVERRIDE is not a valid file." || echo $OVERRIDE "-> OK"
```

```
sed -i -e 's/\(^\s\{2\}deploymentType:\).*$/\1 "'$TYPE'"/' $OVERRIDE
sed -i -e 's/\(^\s\{4\}pullSecret:\).*$/\1 "'$SECRET'"/' $OVERRIDE
sed -i -e 's/\(^\s\{4\}'$LANG':\).*$/\1 true/' $OVERRIDE

```


### Add the cluster namespace label to Watson Discovery namespace

> :warning: The label is needed to permit communication between IBM Watson Assistant  namespace and the Cloud Pak for Data namespace by using a network policy.

> :information_source: Run this on Installer

```
oc label --overwrite namespace $(oc project -q) ns=$(oc project -q)
oc get namespace $(oc project -q) --show-labels 
```

### Install Watson Discovery

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
SC="portworx-assistant"
INT_REG=$(oc describe pod $(oc get pod -n openshift-image-registry | awk '$1 ~ "image-registry-" {print $1}') -n openshift-image-registry | awk '$1 ~ "REGISTRY_OPENSHIFT_SERVER_ADDR:" {print $2}') && echo $INT_REG
```

```
$INST_DIR/bin/cpd-linux \
--namespace $(oc project -q) \
--assembly $ASSEMBLY \
--version $VERSION \
--arch $ARCH \
--storageclass $SC \
--cluster-pull-prefix $INT_REG/$(oc project -q) \
--load-from $INST_DIR/bin/cpd-linux-workspace \
--override $OVERRIDE \
--accept-all-licenses

```

> :bulb: Check installation progress

```
watch -n5 "oc get pvc | grep 'watson-ass' && oc get po | grep 'watson-ass'"
```

### Check Watson Discovery status

> :information_source: Run this on Installer

```
$INST_DIR/bin/cpd-linux status \
--namespace $(oc project -q) \
--assembly $ASSEMBLY \
--arch $ARCH
```

![](img/dsc-ready.jpg)


### Access Watson Discovery web console

> :information_source: Run this on Installer

```
oc get routes | awk 'NR==2 {print "Access the web console at https://" $2}'
```

> :bulb: Login as **admin** using **password** for password 

<br>
:checkered_flag::checkered_flag::checkered_flag:
<br>

