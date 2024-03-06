# CP4D466 in a single namespace and selected node(s)



## Installing the IBM Cloud Pak for Data command-line interface

```
cd ~

URL="https://github.com/IBM/cpd-cli/releases/download/v12.0.5/cpd-cli-linux-EE-12.0.5.tgz" && echo ${URL}

URL="https://github.com/IBM/cpd-cli/releases/download/v12.0.2/cpd-cli-linux-EE-12.0.2.tgz"

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

export PROJECT_CPFS_OPS="cpd1"
export PROJECT_CPD_OPS="cpd1"
# export PROJECT_CPFS_OPS="ibm-common-services"
# export PROJECT_CPD_OPS="ibm-common-services"
export PROJECT_CATSRC="openshift-marketplace"
export PROJECT_CPD_INSTANCE="cpd1"
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

## Updating the global image pull secret

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}

~/cpd-cli/cpd-cli manage add-icr-cred-to-global-pull-secret ${IBM_REGISTRY_PASSWORD}
```

## Log to cluster

```
source ~/cpd-cli/cpd-vars.sh

oc login -u ${OCP_USERNAME} -p ${OCP_PASSWORD} ${OCP_URL}
```

## Setting up projects with a node selector and toleration

```
source ~/cpd-cli/cpd-vars.sh

KEY="role" && echo ${KEY}
VALUE="${PROJECT_CPD_INSTANCE}" && echo ${VALUE}

WORKERS="worker1.cacib-ewra.mop.ibm worker2.cacib-ewra.mop.ibm"
WORKERS="w1.ocp14.iicparis.fr.ibm.com"
# oc adm taint node ${WORKERS} ${KEY}=${VALUE}:NoSchedule --overwrite
# oc label node ${WORKERS} ${KEY}=${VALUE} --overwrite
oc adm taint node ${WORKERS} ${KEY}=${VALUE}:NoSchedule
oc label node ${WORKERS} ${KEY}=${VALUE}
# oc adm taint node ${WORKERS} ${KEY}-
# oc label node ${WORKERS} ${KEY}-

oc get nodes -o wide --show-labels | grep ${KEY}=${VALUE} | awk '{print $1}'

oc apply -f - << EOF
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: ${NS} 
  annotations:
    openshift.io/node-selector: ${KEY}=${VALUE}
    scheduler.alpha.kubernetes.io/defaultTolerations: >-
      [{"operator": "Equal", "effect": "NoSchedule", "key":
      "${KEY}", "value": "${VALUE}"} 
      ]
EOF
```

```
      - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker,cpd-dev]}


oc adm taint nodes ld008ocp00230f.cib.group.gca reservedFor=ewra-dev:NoSchedule

oc adm taint nodes ld008ocp00229f.cib.group.gca reservedFor=ewra-dev:NoSchedule

oc annotate namespace ewra-cpd-dev 'scheduler.alpha.kubernetes.io/defaultTolerations'='[{"operator": "Equal", "effect": "NoSchedule", "key": "reservedFor", "value": "ewra-dev"}]'

oc annotate namespace ewra-cpd-dev 'openshift.io/node-selector'='dedicated=ewra-dev'
```





## Install CP4D

### Installing IBM Cloud Pak foundational services in a custom namespace

```
source ~/cpd-cli/cpd-vars.sh

cat << EOF | tee common-service-maps.yaml
namespaceMapping:
- requested-from-namespace:
  - ${PROJECT_CPD_INST_OPERANDS}
  map-to-common-service-namespace: ${PROJECT_CPD_INST_OPERATORS}       
defaultCsNs: ibm-common-services
EOF

oc create configmap common-service-maps --from-file=./common-service-maps.yaml -n kube-public
```

### Install OLM

```
pkill screen; screen -mdS CPD && screen -r CPD

source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}

echo ~/cpd-cli/cpd-cli manage apply-olm \
--release=${VERSION} \
--components=${COMPONENTS} \
--cs_ns=${PROJECT_CPFS_OPS} \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--preview
```

### Verify OLM installation

```
oc get catalogsource -n ${PROJECT_CATSRC}
oc -n ${PROJECT_CPFS_OPS} get sub
oc -n ${PROJECT_CPFS_OPS} get csv
oc -n ${PROJECT_CPFS_OPS} get po
oc get crd | grep operandrequest
oc api-resources --api-group operator.ibm.com
```

### Validate opencloud-operators catalog source is READY

```
source ~/cpd-cli/cpd-vars.sh

oc get catalogsource -n ${PROJECT_CATSRC} opencloud-operators \
-o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'
```

### Validate cpd-platform catalog source is READY

```
source ~/cpd-cli/cpd-vars.sh

oc get catalogsource -n ${PROJECT_CATSRC} cpd-platform \
-o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'
```


### Install CR

```
pkill screen; screen -mdS CPD && screen -r CPD

source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}

~/cpd-cli/cpd-cli manage setup-instance-ns \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--cs_ns=${PROJECT_CPFS_OPS}

echo ~/cpd-cli/cpd-cli manage apply-cr \
--components=${COMPONENTS} \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--cs_ns=${PROJECT_CPFS_OPS} \
--license_acceptance=true \
--preview
```

### Verify CR installation

```
source ~/cpd-cli/cpd-vars.sh

oc get Ibmcpd ibmcpd-cr -n ${PROJECT_CPD_INSTANCE} -o json | jq -r '.status.controlPlaneStatus'
```

### Find out CP4D route and admin password

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage get-cpd-instance-details \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--get_admin_initial_credentials=true
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


## Monitor MCP

```
watch -n5 'oc get mcp -o wide; echo; oc get node -o "custom-columns=NAME:metadata.name,STATE:metadata.annotations.machineconfiguration\\.openshift\\.io/state,DESIRED:metadata.annotations.machineconfiguration\\.openshift\\.io/desiredConfig,CURRENT:metadata.annotations.machineconfiguration\\.openshift\\.io/currentConfig,REASON:metadata.annotations.machineconfiguration\\.openshift\\.io/reason"'
```

## ISCP
```
PRIVATE_REGISTRY="????"

cat << EOF | tee cloud-pak-for-data-mirror.ya
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: cloud-pak-for-data-mirror
spec:
  repositoryDigestMirrors:
  - mirrors:
    - ${PRIVATE_REGISTRY}/opencloudio
    source: quay.io/opencloudio
  - mirrors:
    - ${PRIVATE_REGISTRY}/cp
    source: cp.icr.io/cp
  - mirrors:
    - ${PRIVATE_REGISTRY}
    - ${PRIVATE_REGISTRY}/cp/cpd
    source: cp.icr.io/cp/cpd
  - mirrors:
    - ${PRIVATE_REGISTRY}/cpopen
    source: icr.io/cpopen
  - mirrors:
    - ${PRIVATE_REGISTRY}/cpopen/cpfs
    source: icr.io/cpopen/cpfs    
  - mirrors:
    - ${PRIVATE_REGISTRY}/cpopen/db2u
    source: icr.io/cpopen/db2u
  - mirrors:
    - ${PRIVATE_REGISTRY}/db2u
    source: icr.io/db2u
EOF

oc get imageContentSourcePolicy cloud-pak-for-data-mirror -o yaml
```

### Test MCP

```
WORKER="w1.ocp14.iicparis.fr.ibm.com"
ROLE="cpd"

oc label node ${WORKER} node-role.kubernetes.io/${ROLE}=

oc label node ${WORKER} node-role.kubernetes.io/worker-

cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: ${ROLE}
spec:
  machineConfigSelector:
    matchExpressions:
      - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker,${ROLE}]}
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/${ROLE}: ""
EOF

cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: cpd
  name: 51-cpd
spec:
  config:
    ignition:
      version: 2.2.0
    storage:
      files:
      - contents:
          source: data:,cpd
        filesystem: root
        mode: 0644
        path: /etc/cpdtest
EOF

oc get mcp

oc get pods -n openshift-machine-config-operator -l k8s-app=machine-config-daemon --field-selector "spec.nodeName=${WORKER}"


cat <<EOF |oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: cpd
  name: db2u-kubelet
spec:
  kubeletConfig:
    allowedUnsafeSysctls:
      - "kernel.msg*"
      - "kernel.shm*"
      - "kernel.sem"
EOF

oc label machineconfigpool worker db2u-kubelet=sysctl --overwrite=true
```

## Get container CreateCommand

```
podman inspect olm-utils-play | jq -r '.[].Config.CreateCommand|@tsv'
```

:warning: --env may have to be surrounded with **"**



## TROUBLESHOOTINGS

> the secret cs-ca-certificate-secret is missing 

```

source ~/cpd-cli/cpd-vars.sh

oc login -u ${OCP_USERNAME} -p ${OCP_PASSWORD} ${OCP_URL}
oc login --token=${OCP_TOKEN} --server=${OCP_URL}

cat << EOF| tee cloudpak-cross-namespaces.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cloudpak-cross-namespaces
  namespace: ${PROJECT_CPFS_OPS} # apply this to every network-access= all namespace
spec:
  podSelector: {}
  ingress:
      - from:
        - namespaceSelector:
            matchLabels:
              network-access: all
  egress:
      - to:
        - namespaceSelector:
            matchLabels:
              network-access: all
EOF
```

> https://www.ibm.com/docs/en/cloud-paks/foundational-services/3.23?topic=tcms-problem-when-you-install-two-different-cert-managers



https://ibm-france.slack.com/archives/CM95C10RK/p1687219073426139

```
oc -n ${PROJECT_CPFS_OPS} get cert | grep cs-ca-certificate-secret
```





CM default -> certmanagercainjector memory 512 to 2048



```
apiVersion: operator.ibm.com/v3
kind: CommonService
metadata:
  creationTimestamp: '2023-07-06T11:30:22Z'
  generation: 1
  managedFields:
    - apiVersion: operator.ibm.com/v3
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:ownerReferences':
            .: {}
            'k:{"uid":"7582473c-3740-421e-b16e-eb7cb9382f56"}': {}
        'f:spec':
          .: {}
          'f:services': {}
      manager: OpenAPI-Generator
      operation: Update
      time: '2023-07-06T11:30:22Z'
    - apiVersion: operator.ibm.com/v3
      fieldsType: FieldsV1
      fieldsV1:
        'f:status':
          .: {}
          'f:phase': {}
      manager: manager
      operation: Update
      subresource: status
      time: '2023-07-06T11:31:08Z'
  name: ewra-cpd-dev-common-service
  namespace: ewra-cpd-dev
  ownerReferences:
    - apiVersion: zen.cpd.ibm.com/v1
      kind: ZenService
      name: lite-cr
      uid: 7582473c-3740-421e-b16e-eb7cb9382f56
  resourceVersion: '555637834'
  uid: 0c3167d5-3baa-433c-a6fb-c289e4b31e48
spec:
  size: medium
  services:
    - name: ibm-cert-manager-operator
      spec:
        certManager:
          certManagerCAInjector:
            resources:
              limits:
                cpu: 100m
status:
  phase: Succeeded
```



### Label nodes

```
SUFFIX="eu.airbus.corp"
WORKERS="fr0-spectrum026.${SUFFIX} fr0-spectrum027.${SUFFIX} fr0-spectrum028.${SUFFIX} fr0-spectrum033.${SUFFIX} fr0-spectrum020.${SUFFIX} fr0-spectrum021.${SUFFIX} fr0-spectrum022.${SUFFIX} fr0-spectrum023.${SUFFIX} fr0-spectrum024.${SUFFIX}"

echo ${WORKERS}

WORKERS="worker-1"
WORKERS="worker-2 worker-3"

oc label node ${WORKERS} node-role.kubernetes.io/cpd-dev=
```

### Create cpd-dev mcp

```
cat  << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: cpd-dev
  labels:
    custom-crio: high-pid-limit-cpd-dev
spec:
  machineConfigSelector:
    matchExpressions:
      - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker,cpd-dev]}
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/cpd-dev: ""
EOF
```

 

##oc label machineconfigpool cpd-dev custom-crio=high-pid-limit-cpd-dev


### Changing CRI-O container settings

```
PIDS_LIMIT=16384

cat  << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: ContainerRuntimeConfig
metadata:
  name: large-pidlimit-cpd-dev
spec:
  machineConfigPoolSelector:
    matchLabels:
      custom-crio: high-pid-limit-cpd-dev
  containerRuntimeConfig:
    pidsLimit: 16384
EOF
```


### Monitor MCP

```
watch -n5 'oc get mcp -o wide; echo; oc get node -o "custom-columns=NAME:metadata.name,STATE:metadata.annotations.machineconfiguration\\.openshift\\.io/state,DESIRED:metadata.annotations.machineconfiguration\\.openshift\\.io/desiredConfig,CURRENT:metadata.annotations.machineconfiguration\\.openshift\\.io/currentConfig,REASON:metadata.annotations.machineconfiguration\\.openshift\\.io/reason"'
```



### Changing kernel parameters

```
cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: db2u-kubelet
spec:
  machineConfigPoolSelector:
    matchLabels:
      custom-crio: high-pid-limit-cpd-dev
  kubeletConfig:
    allowedUnsafeSysctls:
      - "kernel.msg*"
      - "kernel.shm*"
      - "kernel.sem"
EOF
```



### Test node settings for kernel update

```
NS="default"
IMG="icr.io/cpopen/cpd/olm-utils:latest"
NODE="worker2.cacib-ewra.mop.ibm"
POD="test"

oc project ${NS}

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
  securityContext:
    sysctls:
    - name: kernel.msgmax
      value: "65536"
EOF

oc exec -n ${NS} -it ${POD} -- cat /proc/sys/kernel/msgmax

oc delete po ${POD} -n ${NS}
```

### 



###################### Kernel params with tuning Operator

 

node-tuning operator already installed by default

 

cat cr-tuning-node.yaml

apiVersion: tuned.openshift.io/v1

kind: Tuned

metadata:

  name: db2u-ipc-tune-cpd-dev

  namespace: openshift-cluster-node-tuning-operator

spec:

  profile:

  - name: openshift-db2u-ipc-cpd-dev

    data: |

      [main]

      summary=Tune IPC Kernel parameters on OpenShift nodes running Db2U engine PODs

      include=openshift-node

      [sysctl]

      kernel.shmmni = 32768

      kernel.shmmax = 18446744073692774399

      kernel.shmall = 18446744073692774399

      kernel.sem = 250 1024000 100 32768

      kernel.msgmni = 32768

      kernel.msgmax = 65536

      kernel.msgmnb = 65536

  recommend:

  - match:

    - label: node-role.kubernetes.io/cpd-dev

    - label: dedicated

      value: ewra-dev

    priority: 10

    profile: openshift-db2u-ipc-cpd-dev

 



for p in `oc get pods -n openshift-cluster-node-tuning-operator -l openshift-app=tuned -o=jsonpath='{range .items[*]}{.metadata.name} {end}'`; do printf "\n*** $p ***\n" ; oc logs pod/$p -n openshift-cluster-node-tuning-operator | grep applied; done

 

#################### Namespace toleration

oc adm taint nodes ld008ocp00230f.cib.group.gca reservedFor=ewra-dev:NoSchedule

oc adm taint nodes ld008ocp00229f.cib.group.gca reservedFor=ewra-dev:NoSchedule

 

oc annotate namespace ewra-cpd-dev 'scheduler.alpha.kubernetes.io/defaultTolerations'='[{"operator": "Equal", "effect": "NoSchedule", "key": "reservedFor", "value": "ewra-dev"}]'

oc annotate namespace ewra-cpd-dev 'openshift.io/node-selector'='dedicated=ewra-dev'

 

oc project ewra-cpd-dev


oc delete limitrange default-limits

oc delete resourcequota default-object-count-quota

oc delete resourcequota default-resource-quota

 



-------------> Check kernel params values

-------------> check unsafesysctl comment vérifier?


```



### Changing Kernel Settings

```



```
cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: db2u-kubelet
spec:
  machineConfigPoolSelector:
    matchLabels:
      custom-crio: high-pid-limit-cpd-dev
  kubeletConfig:
    allowedUnsafeSysctls:
      - "kernel.msg*"
      - "kernel.shm*"
      - "kernel.sem"
EOF



oc get pods -A | grep -Ev '([[:digit:]])/\1.*R|Comp'

```

```



oc get pods -A | grep -Ev '([[:digit:]])/\1.*R|Comp'