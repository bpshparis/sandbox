# Install Cloud Pak for Data

## Hardware requirements

-  One computer which will be called **Installer** that runs Linux or MacOS.

## System requirements

- Have completed  [Prepare for Cloud Pak for Data](https://github.com/bpshparis/sandbox/blob/master/Prepare-for-Cloud-Pak-for-Data.md#prepare-for-cloud-pak-for-data)
- One **WEB server** where following files are available in **read mode**:
  - [lite-3.5.2-x86_64.tar](https://github.com/bpshparis/sandbox/blob/master/Prepare-for-Cloud-Pak-for-Data.md#save-cloud-pak-for-data-downloads-to-web-server)

<br>
:checkered_flag::checkered_flag::checkered_flag:
<br>

## Install Cloud Pak for Data

> :information_source: Commands below are valid for a **Linux/Centos 7**.

> :warning: Some of commands below will need to be adapted to fit Linux/Debian or MacOS .

### Log in OCP

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer 

```
OCP="ocp9"
LB_HOSTNAME="cli-$OCP"
PASSWD="admin"
```

```
oc login https://$LB_HOSTNAME:6443 -u admin -p $PASSWD --insecure-skip-tls-verify=true
```

### Create Cloud Pak for Data project

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
PRJ="cpd"
PRJ_ADMIN="admin"
```
```
oc new-project $PRJ

oc adm policy add-role-to-user cpd-admin-role $PRJ_ADMIN --role-namespace=$(oc project -q) -n $(oc project -q)
```

### Copy Cloud Pak for Data Downloads from web server

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer 

```
INST_DIR=~/cpd
ASSEMBLY="lite"
VERSION="3.5.2"
ARCH="x86_64"
TAR_FILE="$ASSEMBLY-$VERSION-$ARCH.tar"
WEB_SERVER_CP_URL="http://web/cloud-pak/assemblies"
```

```
cd ~
[ -d "$INST_DIR" ] && { rm -rf $INST_DIR; mkdir $INST_DIR; } || { mkdir $INST_DIR; }
cd $INST_DIR

wget -c $WEB_SERVER_CP_URL/$TAR_FILE
tar xvf $TAR_FILE
rm -f $TAR_FILE
```

### Push Cloud Pak for Data images to Openshift registry

> :warning: To avoid network failure, launch installation on locale console or in a screen

> :information_source: Run this on Installer

```
[ ! -z $(command -v screen) ] && echo screen installed || yum install screen -y

pkill screen; screen -mdS ADM && screen -r ADM
```

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
INST_DIR=$(pwd) && echo $INST_DIR
ASSEMBLY="lite"
ARCH="x86_64"
VERSION=$(find $INST_DIR/cpd-cli-workspace/assembly/$ASSEMBLY/$ARCH/* -type d | awk -F'/' '{print $NF}')

[ ! -z "$VERSION" ] && echo $VERSION "-> OK" || echo "ERROR: VERSION is not set."
```

```
podman login -u $(oc whoami) -p $(oc whoami -t) $(oc registry info)

$INST_DIR/cpd-cli preload-images \
--assembly $ASSEMBLY \
--arch $ARCH \
--action push \
--transfer-image-to $(oc registry info)/$(oc project -q) \
--target-registry-password $(oc whoami -t) \
--target-registry-username $(oc whoami) \
--load-from $INST_DIR/cpd-cli-workspace \
--parallelism 5 \
--verbose \
--max-image-retry 1 \
--accept-all-licenses
```


### Create Cloud Pak for Data resources on cluster

> :information_source: Run this on Installer

```
$INST_DIR/cpd-cli adm \
--namespace $(oc project -q) \
--assembly $ASSEMBLY \
--arch $ARCH \
--load-from $INST_DIR/cpd-cli-workspace \
--apply \
--latest-dependency \
--accept-all-licenses
```

> :bulb: Check **cpd-admin-sa, cpd-editor-sa, cpd-norbac-sa and cpd-viewer-sa** services account have been created

```
oc get sa
```

### Install Cloud Pak for Data

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
SC="portworx-shared-gp3"
OVERRIDE_CONFIG="portworx"
SC="managed-nfs-storage" && echo $SC
INT_REG=$(oc registry info --internal) && echo $INT_REG
```



```
$INST_DIR/cpd-cli install \
--namespace $(oc project -q) \
--assembly $ASSEMBLY \
--arch $ARCH \
--storageclass $SC \
--cluster-pull-prefix $INT_REG/$(oc project -q) \
--load-from $INST_DIR/cpd-cli-workspace \
--latest-dependency \
--accept-all-licenses

```

> :bulb: Check installation progress

```
watch -n5 "oc get pvc && oc get po"
```

### Check Cloud Pak for Data status

> :information_source: Run this on Installer

```
$INST_DIR/cpd-cli status \
--namespace $(oc project -q) \
--assembly $ASSEMBLY \
--arch $ARCH
```

![](img/lite-ready.jpg)


### Access Cloud Pak for Data web console

> :information_source: Run this on Installer

```
oc get routes | awk 'NR==2 {print "Access the web console at https://" $2}'
```

> :bulb: Login as **admin** using **password** for password 

<br>
:checkered_flag::checkered_flag::checkered_flag:
<br>

