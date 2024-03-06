# CP4D466 in a single namespace and selected node(s)



## Installing the IBM Cloud Pak for Data command-line interface

```
cd ~

URL="https://github.com/IBM/cpd-cli/releases/download/v12.0.6/cpd-cli-linux-EE-12.0.6.tgz" && echo ${URL}

FILE="$(echo ${URL} | awk -F '/' '{print $NF}')" && echo ${FILE}

wget -c ${URL}

tar xvzf ${FILE}
rm -f ${FILE}

ln -sf cpd-cli* cpd-cli

[ -d ~/cpd-cli ] && ~/cpd-cli/cpd-cli version || echo "ERROR: ~/cpd-cli/cpd-cli NOT FOUND"
```

## Obtaining your IBM entitlement API key

> Get [entitlement API key](https://myibm.ibm.com/products-services/containerlibrary) and paste it in **${HOME}/ibm-entitlement-key**



## Install cloudctl cli

```
wget -c https://github.com/IBM/cloud-pak-cli/releases/download/v3.23.4/cloudctl-linux-amd64.tar.gz

tar -xf cloudctl-linux-amd64.tar.gz
sudo cp cloudctl-linux-amd64 /usr/local/sbin/cloudctl
cloudctl version
```

## Setting up installation environment variables

```
tee ~/cpd-cli/cpd-vars.sh << EOF
#===============================================================================
# Cloud Pak for Data installation variables
#===============================================================================

# ------------------------------------------------------------------------------
# Client workstation
# ------------------------------------------------------------------------------

export CPD_CLI_MANAGE_WORKSPACE="$HOME/cpd-cli/cpd-cli-workspace"
export CASE_REPO_PATH="https://github.com/IBM/cloud-pak/raw/master/repo/case"
export USE_SKOPEO=true
# export OLM_UTILS_LAUNCH_ARGS=<enter launch arguments>

# ------------------------------------------------------------------------------
# Cluster
# ------------------------------------------------------------------------------

export OCP_URL="https://api.ocp14.iicparis.fr.ibm.com:6443"
export OPENSHIFT_TYPE="self-managed"
export OCP_USERNAME="admin"
export OCP_PASSWORD="admin"
# export OCP_TOKEN=""

# ------------------------------------------------------------------------------
# Projects
# ------------------------------------------------------------------------------

export PROJECT_CPFS_OPS="cpd"
export PROJECT_CPD_OPS="cpd"
# export PROJECT_CPFS_OPS="ibm-common-services"
# export PROJECT_CPD_OPS="ibm-common-services"
export PROJECT_CATSRC="openshift-marketplace"
export PROJECT_CPD_INSTANCE="cpd"
# export PROJECT_TETHERED=<enter the tethered project>

# ------------------------------------------------------------------------------
# Storage
# ------------------------------------------------------------------------------

export STG_CLASS_BLOCK="ocs-storagecluster-ceph-rbd"
export STG_CLASS_FILE="ocs-storagecluster-cephfs"

# ------------------------------------------------------------------------------
# IBM Entitled Registry
# ------------------------------------------------------------------------------

export IBM_REGISTRY_LOCATION="cp.icr.io"
export IBM_REGISTRY_USER="cp"
export IBM_REGISTRY_PASSWORD="$(cat ~/ibm-entitlement-key)"
export IBM_ENTITLEMENT_KEY="$(cat ~/ibm-entitlement-key)"

# ------------------------------------------------------------------------------
# Private container registry
# ------------------------------------------------------------------------------
# Set the following variables if you mirror images to a private container registry.
#
# To export these variables, you must uncomment each command in this section.

# export PRIVATE_REGISTRY_LOCATION=""
# export PRIVATE_REGISTRY_USER=""
# export PRIVATE_REGISTRY_PASSWORD=""

# ------------------------------------------------------------------------------
# Cloud Pak for Data version
# ------------------------------------------------------------------------------

export VERSION="4.6.6"

# ------------------------------------------------------------------------------
# Components
# ------------------------------------------------------------------------------
# Set the following variable if you want to install or upgrade multiple components at the same time.
#
# To export the variable, you must uncomment the command.

export COMPONENTS="cpfs,cpd_platform"
EOF
```

## Log to cluster

```
source ~/cpd-cli/cpd-vars.sh

oc login -u ${OCP_USERNAME} -p ${OCP_PASSWORD} ${OCP_URL}
```

## Updating the global image pull secret

```
source ~/cpd-cli/cpd-vars.sh

oc extract secret/pull-secret -n openshift-config --confirm

oc registry login --registry="${IBM_REGISTRY_LOCATION}" --auth-basic="${IBM_REGISTRY_USER}:${IBM_REGISTRY_PASSWORD}" --to=./.dockerconfigjson

podman login --authfile .dockerconfigjson $IBM_REGISTRY_LOCATION

oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=./.dockerconfigjson
```

## Setting up projects with a node selector and toleration

```
source ~/cpd-cli/cpd-vars.sh
oc login -u ${OCP_USERNAME} -p ${OCP_PASSWORD} ${OCP_URL}

KEY="role" && echo ${KEY}
VALUE="${PROJECT_CPD_INSTANCE}" && echo ${VALUE}

WORKERS="w1.ocp14.iicparis.fr.ibm.com w2.ocp14.iicparis.fr.ibm.com"
oc adm taint node ${WORKERS} ${KEY}=${VALUE}:NoSchedule
oc label node ${WORKERS} ${KEY}=${VALUE}

oc get nodes -o wide --show-labels | grep ${KEY}=${VALUE} | awk '{print $1}'

oc apply -f - << EOF
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: ${PROJECT_CPD_INSTANCE} 
  annotations:
    openshift.io/node-selector: ${KEY}=${VALUE}
    scheduler.alpha.kubernetes.io/defaultTolerations: >-
      [{"operator": "Equal", "effect": "NoSchedule", "key":
      "${KEY}", "value": "${VALUE}"} 
      ]
EOF
```

## Install CP4D

### Install OLM

####  Create operator group

```
source ~/cpd-cli/cpd-vars.sh

cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  namespace: ${PROJECT_CPFS_OPS}
  name: operatorgroup
spec:
  targetNamespaces:
  - ${PROJECT_CPFS_OPS}
EOF
```

#### Downloading case package ibm-cp-common-services 1.19.3

```
source ~/cpd-cli/cpd-vars.sh

cloudctl case save --repo ${CASE_REPO_PATH} --case ibm-cp-common-services --version 1.19.3 --outputdir ${CPD_CLI_MANAGE_WORKSPACE}/work/offline/4.6.6/cpfs

```

#### Downloading case package ibm-cp-datacore 2.9.0

```
source ~/cpd-cli/cpd-vars.sh

cloudctl case save --repo ${CASE_REPO_PATH} --case ibm-cp-datacore --version 2.9.0 --outputdir ${CPD_CLI_MANAGE_WORKSPACE}/work/offline/4.6.6/cpd_platform --no-dependency

```

#### Install or update catalog source 'opencloud-operators' 

```
oc apply -f - << EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: opencloud-operators
  namespace: ${PROJECT_CATSRC}
spec:
  displayName: IBMCS Operators
  publisher: IBM
  sourceType: grpc
  image: icr.io/cpopen/ibm-common-service-catalog@sha256:f5d2719f3e558e56fbbd0286a881a5a812e413337ef129d4ddea1285d3339a76
EOF
```

####  Create or update configmap for custom namespace for cpfs

```
source ~/cpd-cli/cpd-vars.sh

cat << EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: common-service-maps
  namespace: kube-public
data: 
  common-service-maps.yaml: |
    defaultCsNs: ibm-common-services
    namespaceMapping:
    -   map-to-common-service-namespace: ${PROJECT_CPFS_OPS}
        requested-from-namespace:
        - ${PROJECT_CPD_INSTANCE}
EOF
```

#### Installing IBM Cloud Pak foundational services - Bedrock

##### Create subscription : ibm-common-service-operator with opencloud-operators

```
source ~/cpd-cli/cpd-vars.sh

cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-common-service-operator
  namespace: ${PROJECT_CPFS_OPS}
spec:
  channel: v3.23
  installPlanApproval: Automatic
  name: ibm-common-service-operator
  source: opencloud-operators
  sourceNamespace: ${PROJECT_CATSRC}
EOF
```

##### Verify installing IBM Cloud Pak foundational services - Bedrock

```
oc -n ${PROJECT_CPFS_OPS} get sub
oc -n ${PROJECT_CPFS_OPS} get csv
oc -n ${PROJECT_CPFS_OPS} get po
oc get crd | grep operandrequest
oc api-resources --api-group operator.ibm.com
```

#### Patch NamespaceScope CR common-service in the namespace cs to enable csvInjector

```
source ~/cpd-cli/cpd-vars.sh

oc patch NamespaceScope common-service -n ${PROJECT_CPFS_OPS} --type=merge --patch='{"spec": {"csvInjector": {"enable": true} } }'
```

####  Applying commonservice CR to scale its size up to small

```
source ~/cpd-cli/cpd-vars.sh

cat << EOF | oc apply -f -
apiVersion: operator.ibm.com/v3
kind: CommonService
metadata:
  name: common-service
  namespace: ${PROJECT_CPFS_OPS}
spec:
  size: small
EOF

oc -n ${PROJECT_CPFS_OPS} edit commonservice common-service
```

#### Install or update catalog source 'cpd-platform'

```
source ~/cpd-cli/cpd-vars.sh

oc apply -f - << EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: cpd-platform
  namespace: ${PROJECT_CATSRC}
spec:
  displayName: Cloud Pak for Data
  publisher: IBM
  sourceType: grpc
  image: icr.io/cpopen/ibm-cpd-platform-operator-catalog@sha256:9147e737ff029d573ec9aa018f265761caab37e135d09245f0770b3396259a04
  updateStrategy:
    registryPoll:
      interval: 45m
EOF
```

#### Validate opencloud-operators catalog source is READY

```
source ~/cpd-cli/cpd-vars.sh

oc get catalogsource -n ${PROJECT_CATSRC} opencloud-operators \
-o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'
```

#### Validate cpd-platform catalog source is READY

```
source ~/cpd-cli/cpd-vars.sh

oc get catalogsource -n ${PROJECT_CATSRC} cpd-platform \
-o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'
```

#### Create cpd-operator subscription

```
source ~/cpd-cli/cpd-vars.sh

cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cpd-operator
  namespace: ${PROJECT_CPFS_OPS}
spec:
  channel: v3.8
  installPlanApproval: Automatic
  name: cpd-platform-operator
  source: cpd-platform
  sourceNamespace: ${PROJECT_CATSRC}
EOF

CSV=$(oc get sub -n ${PROJECT_CPFS_OPS} cpd-operator -o jsonpath='{.status.installedCSV} {"\n"}') && echo $CSV

oc get csv -n ${PROJECT_CPFS_OPS} $CSV -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ${PROJECT_CPFS_OPS} -l olm.owner="$CSV" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"
```

### Install CR

####  Create OperandRequest

```
source ~/cpd-cli/cpd-vars.sh

cat << EOF | oc apply -f -
apiVersion: operator.ibm.com/v1alpha1
kind: OperandRequest
metadata:
  name: empty-request
  namespace: ${PROJECT_CPD_INSTANCE}
spec:
  requests: []
EOF
```

####  Applying CR for Cloud Pak for Data Control Plane

```
source ~/cpd-cli/cpd-vars.sh

cat << EOF | oc apply -f -
apiVersion: cpd.ibm.com/v1
kind:  Ibmcpd
metadata:
  name: ibmcpd-cr
  namespace: ${PROJECT_CPD_INSTANCE}
spec:
  license:
    accept: true
    license: Enterprise
  fileStorageClass: ${STG_CLASS_FILE}
  blockStorageClass: ${STG_CLASS_BLOCK}
  version: 4.6.6
  csNamespace: ${PROJECT_CPFS_OPS}
EOF

oc get Ibmcpd ibmcpd-cr -n ${PROJECT_CPD_INSTANCE} -o json | jq -r '.status.controlPlaneStatus'
```

#### Find out CP4D route and admin password

```
source ~/cpd-cli/cpd-vars.sh

oc get route -n ${PROJECT_CPD_INSTANCE}

oc extract secret/admin-user-details -n ${PROJECT_CPD_INSTANCE} --keys=initial_admin_password --to=-
```

:checkered_flag: :checkered_flag::checkered_flag:



## Annexes

### Check pods run on dedicated nodes

```
source ~/cpd-cli/cpd-vars.sh

for pod in $(oc get po -n ${PROJECT_CPFS_OPS} | awk 'NR>1 {print $1}' 2>/dev/null); do oc get po ${pod} -n ${PROJECT_CPFS_OPS} -o json | jq -r .spec.nodeName; done

for pod in $(oc get po -n ${PROJECT_CPD_OPS} | awk 'NR>1 {print $1}' 2>/dev/null); do oc get po ${pod} -n ${PROJECT_CPD_OPS} -o json | jq -r .spec.nodeName; done

for pod in $(oc get po -n ${PROJECT_CPD_INSTANCE} | awk 'NR>1 {print $1}' 2>/dev/null); do oc get po ${pod} -n ${PROJECT_CPD_INSTANCE} -o json | jq -r .spec.nodeName; done
```

