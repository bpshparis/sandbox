# Prepare for DB2 Advanced Edition

## Hardware requirements

-  One computer which will be called **Installer** that runs Linux or MacOS.

## System requirements

- Have completed  [Install Cloud Pak for Data](https://github.com/bpshparis/sandbox/blob/master/Install-Cloud-Pak-for-Data.md#install-cloud-pak-for-data)
- Check latest [**cpd-cli**](https://github.com/IBM/cpd-cli/releases) release
- One **WEB server** where following files are available in **read mode**:
  - [Latest cpd-cli](https://github.com/IBM/cpd-cli/releases/download/v3.5.3/cpd-cli-linux-EE-3.5.3.tgz)
  - [IBM® Cloud Pak for Data entitlement license API key](https://myibm.ibm.com/products-services/containerlibrary) saved in apikey file.

<br>
:checkered_flag::checkered_flag::checkered_flag:
<br>

## Prepare for DB2 Advanced Edition

> :information_source: Commands below are valid for a **Linux/Centos 7**.

> :warning: Some of commands below will need to be adapted to fit Linux/Debian or MacOS .

### Install the cpd command

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
WEB_SERVER_CP_URL="http://web/cloud-pak"
INST_FILE="cpd-cli-linux-EE-3.5.3.tgz"
INST_DIR=~/cpd && echo $INST_DIR
```

```
[ -d "$INST_DIR" ] && { rm -rf $INST_DIR; mkdir $INST_DIR; } || mkdir $INST_DIR
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

### Download  DB2 Advanced Edition resources definitions

> :warning: You have to be on line to execute this step.

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer 

```
INST_DIR=~/cpd
ASSEMBLY="db2oltp"
ARCH="x86_64"
```

```
$INST_DIR/cpd-cli adm --repo $INST_DIR/repo.yaml --assembly $ASSEMBLY --arch $ARCH --accept-all-licenses 
```

> : bulb:  **$INST_DIR/cpd-cli-workspace** have been created and populated with yaml files.

### Download  DB2 Advanced Edition images

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
ASSEMBLY="db2oltp"
ARCH="x86_64"
```

```
$INST_DIR/cpd-cli preload-images --action download -a $ASSEMBLY --arch $ARCH --repo $INST_DIR/repo.yaml --accept-all-licenses
```

> :bulb:  Images have been copied in **$INST_DIR/bin/cpd-linux-workspace/images/**

### Save DB2 Advanced Edition Downloads to web server

> :warning: Adapt settings to fit to your environment.

> :information_source: Run this on Installer

```
INST_DIR=~/cpd
ASSEMBLY="db2oltp"
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

