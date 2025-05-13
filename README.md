# Installing Cloud Pak for Data 5.0.0 on self-managed Redhat Openshift 4.14



> :information_source:[Installing IBM Cloud Pak for Data](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.0.x?topic=installing)



[TOC]



## Setting up a client workstation



### Installing tools

```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y

sudo subscription-manager repos --enable "codeready-builder-for-rhel-8-$(/bin/arch)-rpms"

sudo yum install -y httpd-tools podman ca-certificates openssl skopeo jq bind-utils git wget screen
```

> :bulb: Set **enabled=1** for repo [epel] in /etc/yum.repos.d/epel.repo if necessary



### Installing the IBM Cloud Pak for Data command-line interface

> :information_source:[Installing the IBM Cloud Pak for Data command-line interface](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=workstation-installing-cloud-pak-data-cli)


```
cd ~

URL="https://github.com/IBM/cpd-cli/releases/download/v14.0.0/cpd-cli-linux-EE-14.0.0.tgz" && echo ${URL}

FILE="$(echo ${URL} | awk -F '/' '{print $NF}')" && echo ${FILE}

wget -c ${URL}

tar xvzf ${FILE}
rm -f ${FILE}

ln -sf cpd-cli* cpd-cli

[ -d ~/cpd-cli ] && ~/cpd-cli/cpd-cli version || echo "ERROR: ~/cpd-cli/cpd-cli NOT FOUND"
```



### Installing oc

```
cd ~

wget -c https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz

tar xvzf openshift-client-linux.tar.gz -C ~/bin

oc version --client

rm -f openshift-client-linux.tar.gz

sudo yum -y install bash-completion

oc completion bash | sudo tee /etc/bash_completion.d/oc_completion

source /etc/bash_completion.d/oc_completion
```



### Installing ibmcloud command

```
cd ~

[ -f /usr/local/bin/ibmcloud ] && { sudo /usr/local/ibmcloud/uninstall; rm -rf ~/.bluemix/; } || echo "ibmcloud not installed."

curl -fsSL https://clis.cloud.ibm.com/install/linux | sh

ibmcloud plugin install vpc-infrastructure
ibmcloud plugin install container-service

USERID="sebastien.gautier@fr.ibm.com"
PASSWD=""

ibmcloud login -u $USERID -p $PASSWD --sso --no-region

APIKEY_NAME="ibmcloud-clarins-ds"
APIKEY_FILE="/home/fr054721/Documents/clarins/DataStage/${APIKEY_NAME}"
ibmcloud iam api-key-create $APIKEY_NAME -d "$APIKEY_NAME" --file $APIKEY_FILE

ibmcloud logout

ibmcloud login --apikey @$APIKEY_FILE --no-region
```



## Collecting information required to install IBM Cloud Pak for Data



### Obtaining your IBM entitlement API key

> Get [entitlement API key](https://myibm.ibm.com/products-services/containerlibrary) and paste it in **~/ibm-entitlement-key**



### Setting up installation environment variables

> :information_source:[Setting up installation environment variables](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.0.x?topic=information-setting-up-installation-environment-variables)

> :warning: Setup file below with values from you environment before sourcing.


```
tee ~/cpd-cli/cpd-vars.sh << EOF
#===============================================================================
# Cloud Pak for Data installation variables
#===============================================================================

# ------------------------------------------------------------------------------
# Client workstation 
# ------------------------------------------------------------------------------
# Set the following variables if you want to override the default behavior of the Cloud Pak for Data CLI.
#
# To export these variables, you must uncomment each command in this section.

export CPD_CLI_MANAGE_WORKSPACE="${HOME}/cpd-cli/cpd-cli-workspace"
# export OLM_UTILS_LAUNCH_ARGS=<enter launch arguments>

# ------------------------------------------------------------------------------
# Cluster
# ------------------------------------------------------------------------------

# export OCP_URL="$(oc whoami --show-server)"
export OCP_URL="https://api.6696b0858fdee8001e559692.ocp.techzone.ibm.com:6443"
export OPENSHIFT_TYPE="self-managed"
export IMAGE_ARCH="amd64"
export OCP_USERNAME="kubeadmin"
export OCP_PASSWORD="6TYtt-rTy8T-DqtQj-Y8xkV"
# export OCP_TOKEN="sha256~sy-E8jkl6eWRvKcw1O6unsABXaiYvA8rmN7JYzw0J8o"
# export SERVER_ARGUMENTS="--server=${OCP_URL}"
# export LOGIN_ARGUMENTS="--username=${OCP_USERNAME} --password=${OCP_PASSWORD}"
export LOGIN_ARGUMENTS="--token=${OCP_TOKEN}"
export CPDM_OC_LOGIN="cpd-cli manage login-to-ocp ${SERVER_ARGUMENTS} ${LOGIN_ARGUMENTS}"
export OC_LOGIN="oc login ${OCP_URL} ${LOGIN_ARGUMENTS}"

# ------------------------------------------------------------------------------
# Projects
# ------------------------------------------------------------------------------

export PROJECT_CERT_MANAGER="ibm-cert-manager"
export PROJECT_LICENSE_SERVICE="ibm-licensing"
# export PROJECT_SCHEDULING_SERVICE=<enter your scheduling service project>
# export PROJECT_IBM_EVENTS=<enter your IBM Events Operator project>
# export PROJECT_PRIVILEGED_MONITORING_SERVICE=<enter your privileged monitoring service project>
export PROJECT_CPD_INST_OPERATORS="ibm-common-services"
export PROJECT_CPD_INST_OPERANDS="ibm-cpd"
# export PROJECT_CPD_INSTANCE_TETHERED=<enter your tethered project>
# export PROJECT_CPD_INSTANCE_TETHERED_LIST=<a comma-separated list of tethered projects>

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
export IBM_REGISTRY_PASSWORD="$(cat ${HOME}/ibm-entitlement-key)"
export IBM_ENTITLEMENT_KEY="$(cat ${HOME}/ibm-entitlement-key)"

# ------------------------------------------------------------------------------
# Private container registry
# ------------------------------------------------------------------------------
# Set the following variables if you mirror images to a private container registry.
#
# To export these variables, you must uncomment each command in this section.

# export PRIVATE_REGISTRY_LOCATION=""
# export PRIVATE_REGISTRY_PUSH_USER=""
# export PRIVATE_REGISTRY_PUSH_PASSWORD=""
# export PRIVATE_REGISTRY_PULL_USER=""
# export PRIVATE_REGISTRY_PULL_PASSWORD=""

# ------------------------------------------------------------------------------
# Cloud Pak for Data version
# ------------------------------------------------------------------------------

export VERSION=5.0.0

# ------------------------------------------------------------------------------
# Components
# ------------------------------------------------------------------------------

export COMPONENTS=ibm-cert-manager,ibm-licensing,cpfs,cpd_platform
# export COMPONENTS=ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,datastage_ent
# export COMPONENTS_TO_SKIP=<component-ID-1>,<component-ID-2>
EOF
```



## Preparing your cluster for IBM Cloud Pak for Data

### Setting up Amazon Elastic File System

> :information_source: [Setting up Amazon Elastic File System](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=storage-setting-up-amazon-elastic-file-system)

> :bulb: If EFS was deployed and configured already then at [Getting the connection details for your Amazon Elastic File System](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=storage-setting-up-amazon-elastic-file-system#efs-stg__title__8) step.


```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}


export EFS_LOCATION="fs-036f5d9f92b617ce6.efs.us-east-2.amazonaws.com"
export EFS_PATH="/"
export PROJECT_NFS_PROVISIONER="nfs-provisioner"
export EFS_STORAGE_CLASS="efs-nfs-client"
export NFS_IMAGE="registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2"


~/cpd-cli/cpd-cli manage setup-nfs-provisioner \
--nfs_server=${EFS_LOCATION} \
--nfs_path=${EFS_PATH} \
--nfs_provisioner_ns=${PROJECT_NFS_PROVISIONER} \
--nfs_storageclass_name=${EFS_STORAGE_CLASS} \
--nfs_provisioner_image=${NFS_IMAGE}

```



### Changing the process IDs limit

> :information_source:[Changing the process IDs limit](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=settings-changing-process-ids-limit)


```
source ~/cpd-cli/cpd-vars.sh

oc login -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}

oc apply -f - << EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: cpd-pidslimit-kubeletconfig
spec:
  kubeletConfig:
    podPidsLimit: 16384
  machineConfigPoolSelector:
    matchExpressions:
    - key: pools.operator.machineconfiguration.openshift.io/worker
      operator: Exists
EOF
```


```
NODES=$(oc get nodes | awk 'NR>1 {print $1}') && echo $NODES

for NODE in $NODES; do oc debug node/${NODE} -- bash -lc "chroot /host sudo crio-status config | grep pids_limit"; done
```

> :bulb: Change on each node if necessary

```
LIMIT="16384"

cat << EOF | sudo tee /etc/crio/crio.conf.d/01-ctrcfg-pidsLimit
[crio]
  [crio.runtime]
    pids_limit = $LIMIT
EOF

sudo systemctl restart crio

sudo crio-status config | grep pids_limit
```



### Changing kernel parameter settings

> :information_source:[Changing kernel parameter settings](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.7.x?topic=settings-changing-kernel-parameter)

```
source ~/cpd-cli/cpd-vars.sh

NODES=$(oc get nodes | awk 'NR>1 {print $1}') && echo $NODES

oc debug node/${NODE} -- bash -lc chroot /host

cat << EOF | sudo tee -a /etc/sysctl.d/99-sysctl.conf

kernel.shmmni = 32768
kernel.sem = 250 1024000 100 32768
kernel.msgmax = 65536
kernel.msgmnb = 65536
kernel.msgmni = 32768
EOF

sudo sysctl -p 
KERNEL_PARMS="kernel.shmall|kernel.shmmax|kernel.shmmni|kernel.sem|kernel.msgmax|kernel.msgmnb|kernel.msgmni|vm.max_map_count"
sudo sysctl -a 2>/dev/null | egrep -w $KERNEL_PARMS 
```

```
NODES=$(oc get nodes | awk 'NR>1 {print $1}') && echo $NODES

oc debug node/${NODE} -- bash -lc chroot /host

sudo vi /etc/kubernetes/kubelet.conf 

allowedUnsafeSysctls:
- "kernel.msg*"
- "kernel.shm*"
- "kernel.sem"

for NODE in ${NODES}; do \
oc debug node/${NODE} -T -- chroot /host sh -c \
'tee -a /etc/kubernetes/kubelet.conf << EOF

allowedUnsafeSysctls:
- "kernel.msg*"
- "kernel.shm*"
- "kernel.sem"
EOF' \
; done

for NODE in ${NODES}; do oc debug node/${NODE} -T -- chroot /host sh -c 'tail -10 /etc/kubernetes/kubelet.conf'; done

sudo systemctl restart kubelet
sudo systemctl status kubelet | egrep -w "Active:|kubelet.service"
```



### Updating the global image pull secret for IBM Cloud Pak for Data

> :information_source:[Updating the global image pull secret for IBM Cloud Pak for Data](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.0.x?topic=cluster-updating-global-image-pull-secret)

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}
~/cpd-cli/cpd-cli manage login-to-ocp -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage add-icr-cred-to-global-pull-secret --entitled_registry_key=${IBM_ENTITLEMENT_KEY}
```

> :bulb: Monitor MCP
```
watch -n5 'oc get mcp -o wide; echo; oc get node -o "custom-columns=NAME:metadata.name,STATE:metadata.annotations.machineconfiguration\\.openshift\\.io/state,DESIRED:metadata.annotations.machineconfiguration\\.openshift\\.io/desiredConfig,CURRENT:metadata.annotations.machineconfiguration\\.openshift\\.io/currentConfig,REASON:metadata.annotations.machineconfiguration\\.openshift\\.io/reason"'

```

```
source ~/cpd-cli/cpd-vars.sh

oc login --token=${OCP_TOKEN} --server=${OCP_URL}

oc extract secret/pull-secret -n openshift-config --confirm

oc registry login --registry="${IBM_REGISTRY_LOCATION}" --auth-basic="${IBM_REGISTRY_USER}:${IBM_REGISTRY_PASSWORD}" --to=./.dockerconfigjson

podman login --authfile .dockerconfigjson $IBM_REGISTRY_LOCATION

oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=./.dockerconfigjson
```

> :warning: If you are installing Cloud Pak for Data on Red Hat OpenShift on IBM Cloud®, you must manually reload the worker nodes in your cluster for the changes to take effect.

```
APIKEY_FILE="/home/fr054721/Documents/clarins/DataStage/${APIKEY_NAME}"

ibmcloud login --apikey @$APIKEY_FILE --no-region

GROUP=$(ibmcloud resource groups --output json | jq -r .[0].name) && echo $GROUP
REGION="eu-de"
ibmcloud target -g $GROUP -r ${REGION}

CLUSTER_NAME="cluster-clarins"

CLUSTER_ID=$(ibmcloud oc clusters | awk '/'$CLUSTER_NAME'/ {print $2}') && echo $CLUSTER_ID

ibmcloud oc worker ls -c ${CLUSTER_ID}
NODES=$(ibmcloud oc worker ls -c ${CLUSTER_ID} -q | awk '{print $1}') && echo ${NODES}
for wid in $(echo ${NODES}); do ibmcloud oc worker replace -c ${CLUSTER_ID} -w $wid -f; done
```




#### Test worker node settings

```
source ~/cpd-cli/cpd-vars.sh

oc login -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}

NS="default"
IMG="cp.icr.io/cp/cpd/ds-runtime@sha256:10ca71c7dc85e066eae697877d44e40f80bf2322a08ceb2750a4a4b63986de92"
NODE="worker-3"
oc project ${NS}


IMG="icr.io/cpopen/cpd/olm-utils-v3"
NODE="pvln0118"
POD="test-pvln0118"


cat << EOF | oc apply -f -
kind: Pod
apiVersion: v1
metadata:
  name: ${POD}
  namespace: ${NS}
spec:
  nodeName: ${NODE}
  containers:
    - name: ${POD}
      image: ${IMG}
      resources:
        requests:
          cpu: 0.5
          memory: 1024M
        limits:
          cpu: 1
          memory: 2048M
EOF

```
  securityContext:
    sysctls:
    - name: kernel.msgmax
      value: "65536"          
```


oc exec -n ${NS} -it ${POD} -- cat /proc/sys/kernel/msgmax

oc delete po ${POD} -n ${NS}
```

> :bulb: oc exec command above should display **65536**



### Manually creating projects (namespaces) for the shared cluster components for IBM Cloud Pak for Data

> :information_source:[Manually creating projects (namespaces) for the shared cluster components for IBM Cloud Pak for Data](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.0.x?topic=cluster-manually-creating-projects-namespaces-shared-components)



```
source ~/cpd-cli/cpd-vars.sh
ROLE="cpd-dev"

oc login --token ${OCP_TOKEN} ${OCP_URL}

oc new-project ${PROJECT_CERT_MANAGER}

oc annotate namespace ${PROJECT_CERT_MANAGER} 'scheduler.alpha.kubernetes.io/defaultTolerations'='[{"operator": "Equal", "effect": "NoSchedule", "key": "'${ROLE}'", "value": ""}]' --overwrite

oc annotate namespace ${PROJECT_CERT_MANAGER} 'openshift.io/node-selector='${ROLE}'=' --overwrite

oc delete netpol --all -n ${PROJECT_CERT_MANAGER}

oc new-project ${PROJECT_LICENSE_SERVICE}

oc annotate namespace ${PROJECT_LICENSE_SERVICE} 'scheduler.alpha.kubernetes.io/defaultTolerations'='[{"operator": "Equal", "effect": "NoSchedule", "key": "'${ROLE}'", "value": ""}]' --overwrite

oc annotate namespace ${PROJECT_LICENSE_SERVICE} 'openshift.io/node-selector='${ROLE}'=' --overwrite

oc delete netpol --all -n ${PROJECT_LICENSE_SERVICE}
```



### Installing shared cluster components for IBM Cloud Pak for Data

> :information_source:[Installing shared cluster components for IBM Cloud Pak for Data](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=cluster-installing-shared-components)

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}
~/cpd-cli/cpd-cli manage login-to-ocp -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--cert_manager_ns=${PROJECT_CERT_MANAGER} \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
--preview=true
```



## Preparing to install an instance of IBM Cloud Pak for Data



### Manually creating projects (namespaces) for an instance of IBM Cloud Pak for Data

> :information_source:[Manually creating projects (namespaces) for an instance of IBM Cloud Pak for Data](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=data-manually-creating-projects-namespaces)

```
source ~/cpd-cli/cpd-vars.sh
ROLE="ibm.com/product=cp4d"

oc login --token ${OCP_TOKEN} ${OCP_URL}

oc new-project ${PROJECT_CPD_INST_OPERATORS}

oc annotate namespace ${PROJECT_CPD_INST_OPERATORS} 'scheduler.alpha.kubernetes.io/defaultTolerations'='[{"operator": "Equal", "effect": "NoSchedule", "key": "'${ROLE}'", "value": ""}]' --overwrite

oc annotate namespace ${PROJECT_CPD_INST_OPERATORS} 'openshift.io/node-selector='${ROLE} --overwrite

oc delete netpol --all -n ${PROJECT_CPD_INST_OPERATORS}

oc new-project ${PROJECT_CPD_INST_OPERANDS}

oc annotate namespace ${PROJECT_CPD_INST_OPERANDS} 'scheduler.alpha.kubernetes.io/defaultTolerations'='[{"operator": "Equal", "effect": "NoSchedule", "key": "'${ROLE}'", "value": ""}]' --overwrite

oc annotate namespace ${PROJECT_CPD_INST_OPERANDS} 'openshift.io/node-selector='${ROLE}'=' --overwrite

oc delete netpol --all -n ${PROJECT_CPD_INST_OPERANDS}
```



### Applying the required permissions to the projects (namespaces) for an instance of IBM Cloud Pak for Data

> :information_source:[Applying the required permissions to the projects (namespaces) for an instance of IBM Cloud Pak for Data](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=data-applying-required-permissions-projects-namespaces)

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}
~/cpd-cli/cpd-cli manage login-to-ocp -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}


~/cpd-cli/cpd-cli manage authorize-instance-topology \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```



## Installing an instance of IBM Cloud Pak for Data



### Installing the IBM Cloud Pak foundational services for Cloud Pak for Data

> :information_source:[Installing the IBM Cloud Pak foundational services for Cloud Pak for Data](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=data-installing-cloud-pak-foundational-services)

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}
~/cpd-cli/cpd-cli manage login-to-ocp -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage setup-instance-topology \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--block_storage_class=${STG_CLASS_BLOCK}
```



### Specifying the privileges that Db2U runs with

> :information_source:[Specifying the privileges that Db2U runs with](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=data-specifying-privileges-that-db2u-runs)

```
oc apply -f - << EOF
apiVersion: v1
data:
  DB2U_RUN_WITH_LIMITED_PRIVS: "false"
kind: ConfigMap
metadata:
  name: db2u-product-cm
  namespace: ${PROJECT_CPD_INST_OPERATORS}
EOF
```

> :bulb: ROSA is a managed OpenShift cluster, you must allow Db2U to run with elevated privileges. Set DB2U_RUN_WITH_LIMITED_PRIVS: **false** in the db2u-product-cm ConfigMap.



## Installing IBM Cloud Pak for Data

:information_source:[Installing IBM Cloud Pak for Data](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=data-installing-cloud-pak)



### Install the operators

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}
~/cpd-cli/cpd-cli manage login-to-ocp -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=${COMPONENTS}
```



### Install the operands

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}
~/cpd-cli/cpd-cli manage login-to-ocp -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-cr \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=${COMPONENTS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true
```



> :hourglass_flowing_sand: :smoking::coffee::smoking::coffee::smoking::coffee::smoking: :coffee: :hourglass_flowing_sand: :beer::beer::beer::pill:  :zzz::zzz: :zzz::zzz: :zzz::zzz::hourglass_flowing_sand: :smoking::coffee: :toilet: :shower: :smoking: :coffee::smoking: :coffee: :smoking: :coffee: :hourglass: 



### Confirm that the status of the operands is Completed

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp --token${OCP_TOKEN} --server=${OCP_URL}
~/cpd-cli/cpd-cli manage login-to-ocp -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```



### Get the url and the default password of the default administrator

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp --token${OCP_TOKEN} --server=${OCP_URL}
~/cpd-cli/cpd-cli manage login-to-ocp -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage get-cpd-instance-details \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--get_admin_initial_credentials=true
```



<!--

CPD Url: 

cp-console-ibm-cpd.cluster-clarins-b4a69a14292d9bd7e8dc18022d4be5d2-0000.eu-de.containers.appdomain.cloud
cpd-ibm-cpd.cluster-clarins-b4a69a14292d9bd7e8dc18022d4be5d2-0000.eu-de.containers.appdomain.cloud
cp-console-ibm-cpd.cluster-clarins-b4a69a14292d9bd7e8dc18022d4be5d2-0000.eu-de.containers.appdomain.cloud
cp-console-ibm-cpd.cluster-clarins-b4a69a14292d9bd7e8dc18022d4be5d2-0000.eu-de.containers.appdomain.cloud
cp-console-ibm-cpd.cluster-clarins-b4a69a14292d9bd7e8dc18022d4be5d2-0000.eu-de.containers.appdomain.cloud
cp-console-ibm-cpd.cluster-clarins-b4a69a14292d9bd7e8dc18022d4be5d2-0000.eu-de.containers.appdomain.cloud
cp-console-ibm-cpd.cluster-clarins-b4a69a14292d9bd7e8dc18022d4be5d2-0000.eu-de.containers.appdomain.cloud
cp-console-ibm-cpd.cluster-clarins-b4a69a14292d9bd7e8dc18022d4be5d2-0000.eu-de.containers.appdomain.cloud
cp-console-ibm-cpd.cluster-clarins-b4a69a14292d9bd7e8dc18022d4be5d2-0000.eu-de.containers.appdomain.cloud
cp-console-ibm-cpd.cluster-clarins-b4a69a14292d9bd7e8dc18022d4be5d2-0000.eu-de.containers.appdomain.cloud
CPD Username: cpadmin
CPD Password: KfAwxvS8QYiLddPCpSEoOmcnoFEnedLX

CPD Url: cp-console-ibm-cpd.apps.6601c4aa1b0194001e820efb.cloud.techzone.ibm.com
cpd-ibm-cpd.apps.6601c4aa1b0194001e820efb.cloud.techzone.ibm.com
cp-console-ibm-cpd.apps.6601c4aa1b0194001e820efb.cloud.techzone.ibm.com
cp-console-ibm-cpd.apps.6601c4aa1b0194001e820efb.cloud.techzone.ibm.com
cp-console-ibm-cpd.apps.6601c4aa1b0194001e820efb.cloud.techzone.ibm.com
cp-console-ibm-cpd.apps.6601c4aa1b0194001e820efb.cloud.techzone.ibm.com
cp-console-ibm-cpd.apps.6601c4aa1b0194001e820efb.cloud.techzone.ibm.com
cp-console-ibm-cpd.apps.6601c4aa1b0194001e820efb.cloud.techzone.ibm.com
cp-console-ibm-cpd.apps.6601c4aa1b0194001e820efb.cloud.techzone.ibm.com
cp-console-ibm-cpd.apps.6601c4aa1b0194001e820efb.cloud.techzone.ibm.com
CPD Username: cpadmin
CPD Password: 4h850JnR8PJh1dnxn0kVSTOhc7Bir4we


-->



> :checkered_flag: :checkered_flag: :checkered_flag: 



## Installing DataStage

### Install the operators

```
source ~/cpd-cli/cpd-vars.sh
COMPONENTS="datastage_ent"

~/cpd-cli/cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=${COMPONENTS}
```

### Install the operands

```
source ~/cpd-cli/cpd-vars.sh
COMPONENTS="datastage_ent"

~/cpd-cli/cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-cr \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=${COMPONENTS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true
```



## Installing IBM Watson Pipelines

### Install the operators

```
source ~/cpd-cli/cpd-vars.sh
COMPONENTS="ws_pipelines"

~/cpd-cli/cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=${COMPONENTS}
```

### Install the operands

```
source ~/cpd-cli/cpd-vars.sh
COMPONENTS="ws_pipelines"

~/cpd-cli/cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-cr \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=${COMPONENTS} \
--license_acceptance=true
```


CPD 4.8.4 installed on ROSA cluster: 
https://console-openshift-console.apps.lcl-rosa.g4tb.p1.openshiftapps.com/
CPD Url: https://cpd-ibm-cpd.apps.lcl-rosa.g4tb.p1.openshiftapps.com
CPD Username: cpadmin
CPD Password: N67RYhc6DnUN0pdKzQxTj5f0ofnR4OPD

Next steps for .ai install once we get GPUs (total ~2 hours):

Install NFD & Nvidia GPU operators:
https://pages.github.ibm.com/CESC-Infrastructure-Services/Installation-CP4D-WatsonX/#/operators_installation

Install .ai component for CPD 4.8.4

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=${COMPONENTS}
```


```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-cr \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=${COMPONENTS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true

~/cpd-cli/cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=${COMPONENTS}

~/cpd-cli/cpd-cli manage apply-entitlement \
--cpd_instance_ns=$PROJECT_CPD_INST_OPERANDS \
--entitlement=watsonx-ai --production=false --apply_branding=true
```

https://cpd-ibm-cpd.apps.lcl-rosa.g4tb.p1.openshiftapps.com/zen/#/homepage
cpadmin
N67RYhc6DnUN0pdKzQxTj5f0ofnR4OPD


Deploy LLM for inference

ibm-granite-13b-instruct-v1
google-flan-ul2
google-flan-t5-xxl
meta-llama-llama-2-70b-chat
google-flan-t5-xl

```
source ~/cpd-cli/cpd-vars.sh
oc login -u=${OCP_USERNAME} -p=${OCP_PASSWORD} --server=${OCP_URL}

oc patch watsonxaiifm watsonxaiifm-cr \
--namespace=${PROJECT_CPD_INST_OPERANDS} \
--type=merge \
--patch='{"spec":{"install_model_list": ["ibm-granite-13b-instruct-v1","google-flan-ul2","google-flan-t5-xxl","meta-llama-llama-2-70b-chat","google-flan-t5-xl"]}}'


oc patch watsonxaiifm watsonxaiifm-cr \
--namespace=${PROJECT_CPD_INST_OPERANDS} \
--type=merge \
--patch='{"spec":{"install_model_list": ["google-flan-t5-xl", "meta-llama-llama-2-13b-chat"]}}'
```

