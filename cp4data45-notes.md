https://www.ibm.com/docs/en/cloud-paks/cp-data/4.5.x?topic=workstation-installing-cloud-pak-data-cli



### Installing cpd-cli

```
cd ~

URL="https://github.com/IBM/cpd-cli/releases/download/v12.0.6/cpd-cli-linux-EE-12.0.6.tgz" && echo ${URL}
URL="https://github.com/IBM/cpd-cli/releases/download/v12.0.1/cpd-cli-linux-EE-12.0.1.tgz" && echo ${URL}

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

APIKEY_NAME="ibmcloud-key"
ibmcloud iam api-key-create $APIKEY_NAME -d "$APIKEY_NAME" --file ~/$APIKEY_NAME

ibmcloud logout

ibmcloud login --apikey @~/ibmcloud-key --no-region
```





Get [apikey](https://myibm.ibm.com/products-services/containerlibrary)



### Setting up installation environment variables

```
tee ~/cpd-cli/cpd-vars.sh << EOF
#===============================================================================
# Cloud Pak for Data installation variables
#===============================================================================

# ------------------------------------------------------------------------------
# Client workstation
# ------------------------------------------------------------------------------

export CPD_CLI_MANAGE_WORKSPACE=/root/cpd-cli/cpd-cli-workspace
# export OLM_UTILS_LAUNCH_ARGS=<enter launch arguments>

# ------------------------------------------------------------------------------
# Cluster
# ------------------------------------------------------------------------------

export OCP_URL="https://api.ocp14.iicparis.fr.ibm.com:6443"
export OPENSHIFT_TYPE="self-managed"
export OCP_USERNAME="admin"
export OCP_PASSWORD="admin"
# export OCP_TOKEN="sha256~MNOf6Q5SDnG85k5BpsfE3dEHCrd0vI7J0EJsfy9bbvI"

# ------------------------------------------------------------------------------
# Projects
# ------------------------------------------------------------------------------

export PROJECT_CPFS_OPS="cpd"
export PROJECT_CPD_OPS="cpd"
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



### Creating the custom security context constraint for Watson Knowledge Catalog

```
~/cpd-cli/cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-scc \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--components=wkc

oc adm policy who-can use scc wkc-iis-scc \
--namespace ${PROJECT_CPD_INSTANCE} | grep "wkc-iis-sa"
```



### Changing load balancer timeout settings

```
sudo sed -i -e "/timeout client/s/ [0-9].*/ 5m/" /etc/haproxy/haproxy.cfg

sudo sed -i -e "/timeout server/s/ [0-9].*/ 5m/" /etc/haproxy/haproxy.cfg

sudo systemctl restart haproxy
```

### Login to cluster

```

source ~/cpd-cli/cpd-vars.sh
oc login -u ${OCP_USERNAME} -p ${OCP_PASSWORD} ${OCP_URL} --insecure-skip-tls-verify=true
```





### Changing CRI-O container settings

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-crio --openshift-type=${OPENSHIFT_TYPE}




cat << EOF | sudo tee /etc/crio/crio.conf.d/01-ctrcfg-pidsLimit
[crio]
  [crio.runtime]
    pids_limit = 16384
EOF

sudo systemctl restart crio

NODE="worker1.cacib-ewra.mop.ibm"

oc debug node/${NODE} -- bash -lc "chroot /host sudo crio-status config | grep pids_limit"
```



### Changing Kernel Settings

```
cat << EOF | sudo tee -a /etc/sysctl.d/99-sysctl.conf

kernel.shmmni = 32768
kernel.sem = 250 1024000 100 32768
kernel.msgmax = 65536
kernel.msgmnb = 65536
kernel.msgmni = 32768
EOF

sudo sysctl -p 
KERNEL_PARMS="kernel.shmall|kernel.shmmax|kernel.shmmni|kernel.sem|kernel.msgmax|kernel.msgmnb|kernel.msgmni|vm.max_map_count"
sysctl -a 2>/dev/null | egrep -w $KERNEL_PARMS 
```


### Changing node settings by running the cpd-cli manage apply-db2-kubelet command

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-db2-kubelet

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

for WORKER in $WORKERS; do ssh -o StrictHostKeyChecking=no ${USER}@${WORKER} 'hostname -f; sudo cat /etc/kubernetes/kubelet.conf | grep allowedUnsafeSysctls -A4'; done
```



### Updating the global image pull secret

```
source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}

~/cpd-cli/cpd-cli manage add-icr-cred-to-global-pull-secret ${IBM_REGISTRY_PASSWORD}
```





### Update the cluster pull secret

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

podman login -u $IBM_REGISTRY_USER -p $IBM_REGISTRY_PASSWORD $IBM_REGISTRY_LOCATION


oc login --token=${OCP_TOKEN} --server=${OCP_URL}
oc login -u ${OCP_USERNAME} -p ${OCP_PASSWORD} ${OCP_URL}

oc extract secret/pull-secret -n openshift-config --confirm

oc registry login --registry="docker.io" --auth-basic="baudelaine:" --to=./.dockerconfigjson

oc registry login --registry="${IBM_REGISTRY_LOCATION}" --auth-basic="${IBM_REGISTRY_USER}:${IBM_REGISTRY_PASSWORD}" --to=./.dockerconfigjson

podman login --authfile .dockerconfigjson $IBM_REGISTRY_LOCATION

IMG="icr.io/cpopen/cpfs/zen-metastoredb@sha256:d6f4031c215b0364b4cdeb90e845c1ca1722a3387f18c0b28ad58225b840c620"

IMG="icr.io/cpopen/cpfs/icp4data-nginx-repo@sha256:88a817144ba25663160b80739e3f91d0c807de1702614a85e76fed931fd7134a"

IMG="icr.io/cpopen/cpd/olm-utils:latest"

podman pull --authfile .dockerconfigjson ${IMG}
podman rmi ${IMG}

oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=./.dockerconfigjson

```

### Test node settings

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

### Installing IBM Cloud Pak foundational services in a custom namespace

```
source ~/cpd-cli/cpd-vars.sh

cat << EOF | tee common-service-maps.yaml
namespaceMapping:
- requested-from-namespace:
  - ${PROJECT_CPD_INSTANCE}
  map-to-common-service-namespace: ${PROJECT_CPFS_OPS}       
defaultCsNs: ibm-common-services
EOF

oc create configmap common-service-maps --from-file=./common-service-maps.yaml -n kube-public

```

### Setting up projects

```
source ~/cpd-cli/cpd-vars.sh
oc login -u ${OCP_USERNAME} -p ${OCP_PASSWORD} ${OCP_URL}
oc new-project ${PROJECT_CPFS_OPS}
oc new-project ${PROJECT_CPD_INSTANCE}
```



### Setting up projects with a node selector and toleration

```
source ~/cpd-cli/cpd-vars.sh
oc login -u ${OCP_USERNAME} -p ${OCP_PASSWORD} ${OCP_URL}

WORKERS=$(oc get node -o wide | awk '$3=="worker" && NR>1 && ORS=" " {print $1}') && echo ${WORKERS}

KEY="role"
VALUE="cpd0"

WORKERS="worker1.cacib-ewra.mop.ibm worker2.cacib-ewra.mop.ibm"
oc adm taint node ${WORKERS} ${KEY}=${VALUE}:NoSchedule
oc label node ${WORKERS} ${KEY}=${VALUE}
oc adm taint node ${WORKERS} ${KEY}-
oc label node ${WORKERS} ${KEY}-

oc get nodes -o wide --show-labels | grep ${KEY}=${VALUE} | awk '{print $1}'

PROJECT="test"

oc apply -f - << EOF
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: ${PROJECT} 
  annotations:
    openshift.io/node-selector: ${KEY}=${VALUE}
    scheduler.alpha.kubernetes.io/defaultTolerations: >-
      [{"operator": "Equal", "effect": "NoSchedule", "key":
      "${KEY}", "value": "${VALUE}"} 
      ]
EOF

oc apply -f test-project.yaml

oc run -n test nginx --image=nginx --port=80

oc get po nginx -n test -o json | jq -r .spec.nodeName

oc delete project ${PROJECT}

oc apply -f - << EOF
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: ${PROJECT_CPFS_OPS} 
  annotations:
    openshift.io/node-selector: ${KEY}=${VALUE}
    scheduler.alpha.kubernetes.io/defaultTolerations: >-
      [{"operator": "Equal", "effect": "NoSchedule", "key":
      "${KEY}", "value": "${VALUE}"} 
      ]
EOF

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

for pod in $(oc get po -n ${PROJECT_CPFS_OPS} | awk 'NR>1 {print $1}' 2>/dev/null); do oc get po ${pod} -n ${PROJECT_CPFS_OPS} -o json | jq -r .spec.nodeName; done

for pod in $(oc get po -n ${PROJECT_CPD_INSTANCE} | awk 'NR>1 {print $1}' 2>/dev/null); do oc get po ${pod} -n ${PROJECT_CPD_INSTANCE} -o json | jq -r .spec.nodeName; done
```

### Specialized installations

#### Creating OLM objects for a specialized installation

```
pkill screen; screen -mdS CPD && screen -r CPD

source ~/cpd-cli/cpd-vars.sh

~/cpd-cli/cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}

~/cpd-cli/cpd-cli manage apply-olm \
--release=${VERSION} \
--components=${COMPONENTS} \
--cs_ns=${PROJECT_CPFS_OPS} \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--preview=true

~/cpd-cli/cpd-cli manage get-olm-artifacts \
--subscription_ns=${PROJECT_CPFS_OPS}
```

#### Installing components in a specialized installation

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
--cs_ns=${PROJECT_CPFS_OPS} \
--preview=true

~/cpd-cli/cpd-cli manage apply-cr \
--components=${COMPONENTS} \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--cs_ns=${PROJECT_CPFS_OPS} \
--license_acceptance=true \
--preview=true


oc get Ibmcpd ibmcpd-cr -n ${PROJECT_CPD_INSTANCE} -o json | jq -r '.status.controlPlaneStatus'

~/cpd-cli/cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INSTANCE}

for po in $(oc get po -n ${PROJECT_CPD_INSTANCE} | awk 'NR>1 {print $1}'); do oc get po $po -n ${PROJECT_CPD_INSTANCE} -o json | jq .spec.nodeName; done

for po in $(oc get po -n ${PROJECT_CPFS_OPS} | awk 'NR>1 {print $1}'); do oc get po $po -n ${PROJECT_CPFS_OPS} -o json | jq .spec.nodeName; done
```





### Express installations

```source ~/cpd_vars.sh && cd ~/cpd-cli
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}

./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage apply-olm --release=${VERSION} --components=${COMPONENTS} --preview=true

./cpd-cli manage apply-olm --release=${VERSION} --components=${COMPONENTS}
```



```
[root@lb cpd-cli]# oc get catalogsource -n ${PROJECT_CATSRC}
NAME                  DISPLAY               TYPE   PUBLISHER   AGE
certified-operators   Certified Operators   grpc   Red Hat     50d
community-operators   Community Operators   grpc   Red Hat     50d
cpd-platform          Cloud Pak for Data    grpc   IBM         4m21s
opencloud-operators   IBMCS Operators       grpc   IBM         8m54s
redhat-marketplace    Red Hat Marketplace   grpc   Red Hat     50d
redhat-operators      Red Hat Operators     grpc   Red Hat     50d

[root@lb cpd-cli]# oc get po -n ${PROJECT_CATSRC}
40f356b96a22e8e7b1ea3f387dd06a2c605f4acc49d3c5c4aeeae7d83fsrhzm   0/1     Completed   0          7m42s
50acc115a0b9699b5530278355d84eaa547ca8aae3c2a34cac1138ef8dlwkgq   0/1     Completed   0          6m44s
5b2332a4774a1c7b2691726c8243351f418b49d267535afb592d24cdbckpl2j   0/1     Completed   0          8m51s
bc83e3371268f56c2ec37d0d86a3cadd4f98a525d52eb2c6bf97cc597d7hbwm   0/1     Completed   0          4m22s
certified-operators-nzdqw                                         1/1     Running     0          61m
community-operators-bk2zj                                         1/1     Running     0          25h
cpd-platform-txqwv                                                1/1     Running     0          5m31s
marketplace-operator-5c54c8cbd9-x7hv7                             1/1     Running     0          3d23h
opencloud-operators-nhrks                                         1/1     Running     0          10m
redhat-marketplace-crh9g                                          1/1     Running     0          3d13h
redhat-operators-7r5pf                                            1/1     Running     0          3d18h

```





```
PLAY RECAP ********************************************************************************************************************************************
localhost                  : ok=152  changed=22   unreachable=0    failed=0    skipped=125  rescued=0    ignored=0   

Tuesday 19 July 2022  15:15:45 +0000 (0:00:00.028)       0:11:45.838 ********** 
=============================================================================== 
utils : waiting for ODLM and Namespace Scope operators to come online ------------------------------------------------------------------------ 211.47s
utils : check if installedCSV: ibm-cpd-scheduling-operator.v1.3.6 'Succeeded' for Subscription: ibm-cpd-scheduling-catalog-subscription ------ 131.61s
utils : Confirm existence of Catalog Source object "opencloud-operators" ---------------------------------------------------------------------- 72.02s
utils : Confirm existence of Catalog Source object "cpd-platform" ----------------------------------------------------------------------------- 71.62s
utils : Confirm existence of Catalog Source object "ibm-cpd-scheduling-catalog" --------------------------------------------------------------- 71.57s
utils : downloading case package ibm-cp-common-services 1.15.0  ------------------------------------------------------------------------------- 11.85s
utils : downloading case package ibm-cp-datacore 2.1.0  ---------------------------------------------------------------------------------------- 8.81s
utils : Create CPFS namespace if not present ibm-common-services ------------------------------------------------------------------------------- 6.52s
utils : check if namespacescope is present in: ibm-common-services ----------------------------------------------------------------------------- 6.39s
utils : Create cpd operator namespace if not present ibm-common-services ----------------------------------------------------------------------- 6.36s
utils : confirm the Operator Deployment is ready for csv ibm-cpd-scheduling-operator.v1.3.6 ---------------------------------------------------- 6.26s
utils : confirm the Operator Deployment is ready for csv cpd-platform-operator.v3.0.0 ---------------------------------------------------------- 6.23s
utils : create operator group for CPFS namespace if not present ibm-common-services ------------------------------------------------------------ 6.04s
utils : applying operator subscription ibm-cpd-scheduling-catalog-subscription with ibm-cpd-scheduling-catalog --------------------------------- 6.03s
utils : applying operator subscription ibm-common-service-operator with opencloud-operators ---------------------------------------------------- 5.96s
utils : create operator group for cpd operators namespace if not present ibm-common-services --------------------------------------------------- 5.91s
utils : applying operator subscription cpd-operator with cpd-platform -------------------------------------------------------------------------- 5.91s
utils : check if installedCSV: cpd-platform-operator.v3.0.0 'Succeeded' for Subscription: cpd-operator ----------------------------------------- 5.85s
utils : Check Readiness of Catalog Source "opencloud-operators" before creating the subscription ----------------------------------------------- 5.83s
utils : get installedCSV for Subscription: ibm-cpd-scheduling-catalog-subscription .v1.3.6 ----------------------------------------------------- 5.82s
Error: read unixpacket @->/var/run/libpod/socket/4d671315999d4eea47b30564c077fe1f096f1f199be5c50bcd7a14e3e862de55/attach: read: connection reset by peer
[ERROR] 2022-07-19T15:15:45.910871Z cmd.Run() error: Error: read unixpacket @->/var/run/libpod/socket/4d671315999d4eea47b30564c077fe1f096f1f199be5c50bcd7a14e3e862de55/attach: read: connection reset by peer

[SUCCESS] 2022-07-19T15:15:45.910909Z The apply-olm command ran successfully. Output and logs are in the /root/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work directory.

```



```
oc get sub -n ${PROJECT_CPFS_OPS}
NAME                                       PACKAGE                        SOURCE                       CHANNEL
cpd-operator                               cpd-platform-operator          cpd-platform          v3.1
ibm-common-service-operator                ibm-common-service-operator    opencloud-operators   v3
ibm-namespace-scope-operator               ibm-namespace-scope-operator   opencloud-operators   v3
operand-deployment-lifecycle-manager-app   ibm-odlm                       opencloud-operators   v3

```



  - You can verify the successful completion by copying and running the following script:


  ```
#!/bin/bash

set -x
oc --namespace ${PROJECT_CPFS_OPS} get csv
echo
oc get po -n ${PROJECT_CPFS_OPS}
echo
oc get crd | grep operandrequest
echo
oc api-resources --api-group operator.ibm.com
  ```

You should see something as below:

  ```
+ oc --namespace ${PROJECT_CPFS_OPS} get csv
NAME                                          DISPLAY                                VERSION   REPLACES                                      PHASE
ibm-common-service-operator.v3.8.1            IBM Cloud Pak foundational services    3.8.1     ibm-common-service-operator.v3.8.0            Succeeded
ibm-namespace-scope-operator.v1.2.0           IBM NamespaceScope Operator            1.2.0     ibm-namespace-scope-operator.v1.1.1           Succeeded
operand-deployment-lifecycle-manager.v1.6.0   Operand Deployment Lifecycle Manager   1.6.0     operand-deployment-lifecycle-manager.v1.5.0   Succeeded
+ echo

+ oc get po -n ${PROJECT_CPFS_OPS}
NAME                                                    READY   STATUS    RESTARTS   AGE
ibm-common-service-operator-6987c8cff5-l27q2            1/1     Running   0          5m24s
ibm-common-service-webhook-75f48bf49b-m5h6k             1/1     Running   0          4m57s
ibm-namespace-scope-operator-67c5cc6b87-j8qbj           1/1     Running   0          5m6s
operand-deployment-lifecycle-manager-6b4b46d57b-njtxj   1/1     Running   0          4m36s
secretshare-56fbbb8df4-57r6q                            1/1     Running   0          4m51s
+ echo


+ oc get crd | grep operandrequest
operandrequests.operator.ibm.com                            2021-07-19T18:25:13Z
+ echo

+ oc api-resources --api-group operator.ibm.com
NAME                SHORTNAMES   APIGROUP           NAMESPACED   KIND
commonservices                   operator.ibm.com   true         CommonService
namespacescopes     nss          operator.ibm.com   true         NamespaceScope
operandbindinfos    opbi         operator.ibm.com   true         OperandBindInfo
operandconfigs      opcon        operator.ibm.com   true         OperandConfig
operandregistries   opreg        operator.ibm.com   true         OperandRegistry
operandrequests     opreq        operator.ibm.com   true         OperandRequest
podpresets                       operator.ibm.com   true         PodPreset
  ```



### Installing components in an express installation

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}

./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage apply-cr \ --components=${COMPONENTS} \ --release=${VERSION} \ --cpd_instance_ns=${PROJECT_CPD_INSTANCE} \ --block_storage_class=${STG_CLASS_BLOCK} \ --file_storage_class=${STG_CLASS_FILE} \ --license_acceptance=true --preview=true

./cpd-cli manage apply-cr \ --components=${COMPONENTS} \ --release=${VERSION} \ --cpd_instance_ns=${PROJECT_CPD_INSTANCE} \ --block_storage_class=${STG_CLASS_BLOCK} \ --file_storage_class=${STG_CLASS_FILE} \ --license_acceptance=true

oc get Ibmcpd ibmcpd-cr -n ${PROJECT_CPD_INSTANCE} -o yaml

oc get Ibmcpd ibmcpd-cr -n ${PROJECT_CPD_INSTANCE} -o json | jq -r '.status.controlPlaneStatus'

./cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INSTANCE}

for po in $(oc get po -n ${PROJECT_CPD_INSTANCE} | awk 'NR>1 {print $1}'); do oc get po $po -n ${PROJECT_CPD_INSTANCE} -o json | jq .spec.nodeName; done

for po in $(oc get po -n ${PROJECT_CPFS_OPS} | awk 'NR>1 {print $1}'); do oc get po $po -n ${PROJECT_CPFS_OPS} -o json | jq .spec.nodeName; done

```

```
PLAY RECAP *********************************************************************************************************************************
localhost                  : ok=55   changed=8    unreachable=0    failed=0    skipped=89   rescued=0    ignored=0   

Tuesday 30 August 2022  18:00:09 +0000 (0:00:00.067)       1:20:57.563 ******** 
=============================================================================== 
utils : check if CR status indicates completion for ibmcpd-cr in cpd-instance, max retry 25 times 300s delay --------------------- 4220.55s
utils : check if CR status indicates completion for ibm-cpd-scheduler in ibm-common-services, max retry 15 times 300s delay ------- 603.82s
utils : waiting for ODLM and Namespace Scope operators to come online --------------------------------------------------------------- 3.09s
utils : Create cpd instance namespace if not present cpd-instance ------------------------------------------------------------------- 2.07s
utils : fetch the Scheduling CR if it exists ---------------------------------------------------------------------------------------- 1.55s
utils : create a configmap olm-utils-cm to save components versions ----------------------------------------------------------------- 1.52s
utils : applying CR ibmcpd-cr for Cloud Pak for Data Control Plane ------------------------------------------------------------------ 1.51s
utils : applying CR ibm-cpd-scheduler for Scheduling Service ------------------------------------------------------------------------ 1.38s
utils : create operand request for instance namespace if not present ---------------------------------------------------------------- 1.37s
utils : verify if the package cpd-platform-operator  is available for use in the cpd-instance namespace ----------------------------- 1.35s
utils : verify if the CRD is present scheduler.spectrumcomputing.ibm.com v1 Scheduling ---------------------------------------------- 1.29s
utils : verify if the CRD is present cpd.ibm.com v1 Ibmcpd -------------------------------------------------------------------------- 1.28s
utils : verify if the package ibm-cpd-scheduling-operator  is available for use in the ibm-common-services namespace ---------------- 1.18s
utils : Print trace information ----------------------------------------------------------------------------------------------------- 1.18s
utils : fetch the Ibmcpd CR if it exists -------------------------------------------------------------------------------------------- 1.18s
utils : Pause for "1" seconds to let OLM trigger changes (to avoid getting confused by existing state) ------------------------------ 1.10s
utils : Pause for "1" seconds to let OLM trigger changes (to avoid getting confused by existing state) ------------------------------ 1.10s
utils : pre- apply-cr release patching (if any) for scheduler ----------------------------------------------------------------------- 0.71s
utils : post- apply-cr release patching (if any) for scheduler ---------------------------------------------------------------------- 0.46s
utils : pre- apply-cr release patching (if any) for cpd_platform -------------------------------------------------------------------- 0.46s
[SUCCESS] 2022-08-30T20:00:09.827883Z The apply-cr command ran successfully. Output and logs are in the /root/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work directory.

```







  - Test login to CPD console

  Find out the initial CPD admin password by


  ```
oc extract secret/admin-user-details -n ${PROJECT_CPD_INSTANCE} --keys=initial_admin_password --to=-
  ```

  Then navigate to the CPD console, and you should be able login with username "admin" and the listed password in the previous command.



### Uninstalling the components

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --username=${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL}

./cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INSTANCE}

export COMPONENTS="zen,cpd_platform"

./cpd-cli manage delete-cr --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --components=${COMPONENTS}

oc login -u ${OCP_USERNAME} -p ${OCP_PASSWORD} ${OCP_URL}

export RESOURCE_LIST="configmaps,persistentvolumeclaims,pods,secret,serviceaccounts,Service,StatefulSets,deployment,job,cronjob,ReplicaSet,Route,RoleBinding,Role,PodDisruptionBudget,OperandRequest"

oc get ${RESOURCE_LIST} -n ${PROJECT_CPD_INSTANCE} --ignore-not-found 2>/dev/null  | grep -v '^NAME' | grep -v '^$' | awk '{print "oc delete " $1 " -n ${PROJECT_CPD_INSTANCE} --force --grace-period=0 2>/dev/null"}' | sh

oc delete project ${PROJECT_CPD_INSTANCE}

```

### Delete namespace in terminating state

https://access.redhat.com/solutions/4165791



```
# Step 1: Dump the contents of the namespace in a temporary file called tmp.json:

source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

oc delete project ${PROJECT_CPD_INSTANCE}

oc get namespace ${PROJECT_CPD_INSTANCE} -o json > ${PROJECT_CPD_INSTANCE}.json

# Step 2: Edit the temporary file:

vi ${PROJECT_CPD_INSTANCE}.json

# Step 3: Remove kubernetes from the finalizer array, and save the file

screen -mdS PROXY
screen -r PROXY
oc proxy

# Back to first window (Ctrl + a + d), run this command:

curl -k -H "Content-Type: application/json" -X PUT --data-binary @${PROJECT_CPD_INSTANCE}.json http://127.0.0.1:8001/api/v1/namespaces/${PROJECT_CPD_INSTANCE}/finalize

oc get namespace ${PROJECT_CPD_INSTANCE}
```

### Uninstalling the OLM objects

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --username=${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL}

./cpd-cli manage delete-olm-artifacts

PROJECT_CPFS_OPS="ibm-common-services"

oc get catalogsource -n ${PROJECT_CATSRC}

oc get sub -n ${PROJECT_CPFS_OPS}
oc get sub -n ${PROJECT_CPFS_OPS} 2>/dev/null | awk 'NR>1 {print "oc delete sub " $1 " -n ${PROJECT_CPFS_OPS} 2>/dev/null"}' | sh

oc get csv -n ${PROJECT_CPFS_OPS} 2>/dev/null | awk 'NR>1 {print "oc delete csv " $1 " -n ${PROJECT_CPFS_OPS} 2>/dev/null"}' | sh

oc get crd | grep operandrequest
oc delete crd operandrequests.operator.ibm.com

oc api-resources --api-group operator.ibm.com

export RESOURCE_LIST="configmaps,persistentvolumeclaims,pods,secret,serviceaccounts,Service,StatefulSets,deployment,job,cronjob,ReplicaSet,Route,RoleBinding,Role,PodDisruptionBudget,OperandRequest"

oc login -u ${OCP_USERNAME} -p ${OCP_PASSWORD} ${OCP_URL}

oc get ${RESOURCE_LIST} -n ${PROJECT_CPFS_OPS} --ignore-not-found 2>/dev/null  | grep -v '^NAME' | grep -v '^$' | awk '{print "oc delete " $1 " -n ${PROJECT_CPFS_OPS} 2>/dev/null"}' | sh


# Step 1: Dump the contents of the namespace in a temporary file called tmp.json:

source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

oc get namespace ${PROJECT_CPFS_OPS} -o json > ${PROJECT_CPFS_OPS}.json

# Step 2: Edit the temporary file:

vi ${PROJECT_CPFS_OPS}.json

# Step 3: Remove kubernetes from the finalizer array, and save the file

screen -mdS PROXY
screen -r PROXY
oc proxy

# Back to first window (Ctrl + a + d), run this command:

curl -k -H "Content-Type: application/json" -X PUT --data-binary @${PROJECT_CPFS_OPS}.json http://127.0.0.1:8001/api/v1/namespaces/${PROJECT_CPFS_OPS}/finalize

oc get namespace ${PROJECT_CPFS_OPS}

```







### WKC

:bulb: OLM stands for Operator Lifecycle Manager



```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage apply-olm --release=${VERSION} --cpd_operator_ns=${PROJECT_CPD_OPS} --components=wkc --preview=true

./cpd-cli manage apply-olm --release=${VERSION} --cpd_operator_ns=${PROJECT_CPD_OPS} --components=wkc

oc get catalogsource -n ${PROJECT_CATSRC}

SUB=$(oc get sub -n ${PROJECT_CPFS_OPS} ibm-cpd-wkc-operator-catalog-subscription -o jsonpath='{.status.installedCSV} {"\n"}') && echo $SUB

oc get csv -n ${PROJECT_CPFS_OPS} $SUB -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ${PROJECT_CPFS_OPS} -l olm.owner="$SUB" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"
```



```
PLAY RECAP *********************************************************************************************************************************
localhost                  : ok=359  changed=41   unreachable=0    failed=0    skipped=286  rescued=0    ignored=0

Tuesday 30 August 2022  18:20:49 +0000 (0:00:00.063)       0:13:39.697 ********
===============================================================================
utils : get installedCSV for Subscription: ibm-cpd-wkc-operator-catalog-subscription .v1.5.1 -------------------------------------- 122.61s
utils : Confirm existence of Catalog Source object "ibm-cpd-datarefinery-operator-catalog" ----------------------------------------- 62.73s
utils : Confirm existence of Catalog Source object "ibm-cpd-wkc-operator-catalog" -------------------------------------------------- 62.66s
utils : Confirm existence of Catalog Source object "ibm-cpd-datastage-operator-catalog" -------------------------------------------- 62.59s
utils : Confirm existence of Catalog Source object "manta-adl-operator-catalog" ---------------------------------------------------- 62.56s
utils : Confirm existence of Catalog Source object "ibm-cpd-ccs-operator-catalog" -------------------------------------------------- 62.55s
utils : Confirm existence of Catalog Source object "ibm-fdb-operator-catalog" ------------------------------------------------------ 62.42s
utils : install catalog source 'ibm-cpd-wkc-operator-catalog' using /tmp/work/offline/4.5.1/wkc/ibm-wkc-4.5.1.tgz  ----------------- 53.66s
utils : install catalog source 'ibm-db2uoperator-catalog' using /tmp/work/offline/4.5.1/wkc/ibm-db2uoperator-4.5.1.tgz  ------------ 33.40s
utils : install catalog source 'ibm-db2aaservice-cp4d-operator-catalog' using /tmp/work/offline/4.5.1/wkc/ibm-db2aaservice-4.5.1.tgz  -- 33.01s
utils : install catalog source 'ibm-cpd-ae-operator-catalog' using /tmp/work/offline/4.5.1/wkc/ibm-analyticsengine-5.1.0.tgz  ------ 31.75s
utils : install catalog source 'ibm-cpd-datastage-operator-catalog' using /tmp/work/offline/4.5.1/wkc/ibm-datastage-enterprise-4.6.0.tgz  --- 3.70s
utils : install catalog source 'ibm-fdb-operator-catalog' using /tmp/work/offline/4.5.1/wkc/ibm-fdb-1.1.13.tgz  --------------------- 2.79s
utils : install catalog source 'ibm-cpd-ccs-operator-catalog' using /tmp/work/offline/4.5.1/wkc/ibm-ccs-2.1.0.tgz  ------------------ 2.61s
utils : install catalog source 'manta-adl-operator-catalog' using /tmp/work/offline/4.5.1/wkc/mantaflow-1.3.8.tgz  ------------------ 2.14s
utils : install catalog source 'ibm-cpd-datarefinery-operator-catalog' using /tmp/work/offline/4.5.1/wkc/ibm-datarefinery-2.1.0.tgz  --- 2.06s
utils : Create CPFS namespace if not present ibm-common-services -------------------------------------------------------------------- 1.82s
utils : check existing operator group in ibm-common-services namespace -------------------------------------------------------------- 1.75s
utils : apply OperandRegistry for iis ----------------------------------------------------------------------------------------------- 1.59s
utils : apply OperandConfig for dependency ------------------------------------------------------------------------------------------ 1.58s
[SUCCESS] 2022-08-30T20:20:49.220049Z The apply-olm command ran successfully. Output and logs are in the /root/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work directory.

```





```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true

tee ~/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work/install-options.yml << EOF
custom_spec:
  wkc:
    wkc_db2u_set_kernel_params: True
    iis_db2u_set_kernel_params: True
    install_wkc_core_only: False
    enableKnowledgeGraph: False
    enableDataQuality: True       
    enableMANTA: False    
EOF

chmod 777 ~/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work/install-options.yml

./cpd-cli manage apply-cr \
--components=wkc \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--param-file=/tmp/work/install-options.yml \
--license_acceptance=true

oc get WKC wkc-cr -n ${PROJECT_CPD_INSTANCE} -o jsonpath='{.status.wkcStatus} {"\n"}'

oc get WKC wkc-cr -n ${PROJECT_CPD_INSTANCE} -o yaml
oc get CCS ccs-cr -n ${PROJECT_CPD_INSTANCE} -o yaml
oc get DataRefinery datarefinery-sample -n ${PROJECT_CPD_INSTANCE} -o yaml
oc get Db2aaserviceService db2aaservice-cr -n ${PROJECT_CPD_INSTANCE} -o yaml
oc get IIS iis-cr -n ${PROJECT_CPD_INSTANCE} -o yaml
oc get UG ug-cr -n ${PROJECT_CPD_INSTANCE} -o yaml

oc get WKC wkc-cr -n ${PROJECT_CPD_INSTANCE} -o jsonpath='{.status.wkcStatus} {"\n"}'
oc get CCS ccs-cr -n ${PROJECT_CPD_INSTANCE} -o jsonpath='{.status.ccsStatus} {"\n"}'
oc get DataRefinery datarefinery-sample -n ${PROJECT_CPD_INSTANCE} -o jsonpath='{.status.datarefineryStatus} {"\n"}'
oc get Db2aaserviceService db2aaservice-cr -n ${PROJECT_CPD_INSTANCE} -o jsonpath='{.status.db2aaserviceStatus} {"\n"}'
oc get IIS iis-cr -n ${PROJECT_CPD_INSTANCE} -o jsonpath='{.status.iisStatus} {"\n"}'
oc get UG ug-cr -n ${PROJECT_CPD_INSTANCE} -o jsonpath='{.status.ugStatus} {"\n"}'
```

### Install **Data quality** 

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage update-cr --component=wkc --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --patch='{"enableDataQuality":True,"install_wkc_core_only":false}'


cpd-cli manage update-cr \
--component=wkc \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch="{\"install_wkc_core_only\":false,\"enableDataQuality\":True}" 
```



### Install DP

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--components=dp


echo ./cpd-cli manage apply-cr \
--components=dp \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true


#askanythingwkc
```



### Installing Watson Query

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--components=dv

oc get catalogsource -n ${PROJECT_CATSRC}

SUB=$(oc get sub -n ${PROJECT_CPFS_OPS} ibm-dv-operator-catalog-subscription \
-o jsonpath='{.status.installedCSV} {"\n"}') && echo $SUB

oc get csv -n ${PROJECT_CPFS_OPS} $SUB \
-o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ${PROJECT_CPFS_OPS} -l olm.owner="$SUB" \
-o jsonpath="{.items[0].status.availableReplicas} {'\n'}"


./cpd-cli manage apply-cr \
--components=dv \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--license_acceptance=true

oc get DvService dv-service -n ${PROJECT_CPD_INSTANCE} -o jsonpath='{.status.dvStatus} {"\n"}'

```



### Installing CDE

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--components=cde

#ibm-cde-operator-catalog
#ibm-cde-operator-subscription

oc get catalogsource -n ${PROJECT_CATSRC} | grep -i 'cde'
oc get sub -n ${PROJECT_CPFS_OPS} | grep -i 'cde' 

SUB=$(oc get sub -n ${PROJECT_CPFS_OPS} ibm-cde-operator-subscription \
-o jsonpath='{.status.installedCSV} {"\n"}') && echo $SUB

oc get csv -n ${PROJECT_CPFS_OPS} $SUB \
-o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ${PROJECT_CPFS_OPS} -l olm.owner="$SUB" \
-o jsonpath="{.items[0].status.availableReplicas} {'\n'}"


./cpd-cli manage apply-cr \
--components=cde \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true

oc get CdeProxyService cdeproxyservice-cr -n ${PROJECT_CPD_INSTANCE}  -o jsonpath='{.status.cdeStatus} {"\n"}'
```





PRIVATE_REGISTRY_LOCATION="10.40.10.22:5000"
PRIVATE_REGISTRY_PULL_PASSWORD="dockeruser"
PRIVATE_REGISTRY_PULL_USER="dockeruser"
PRIVATE_REGISTRY_PUSH_PASSWORD="dockeruser"
PRIVATE_REGISTRY_PUSH_USER="dockeruser"

SOURCE_IMG="docker.io/library/hello-world"

TARGET_REG="10.40.10.22:5000"

TARGET_NS="cpd-instance"

IMG="hello-world"

podman login -u baudelaine -p "\$2005ebA" docker.io

podman pull ${SOURCE_IMG}

podman tag ${SOURCE_IMG} ${TARGET_REG}/${IMG}

podman login --username ${PRIVATE_REGISTRY_PUSH_USER} --password ${PRIVATE_REGISTRY_PUSH_PASSWORD} ${PRIVATE_REGISTRY_LOCATION} --tls-verify=false

podman push ${TARGET_REG}/${IMG} --tls-verify=false

curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY_LOCATION}/v2/_catalog?n=6000 | jq .

curl -k -u ${PRIVATE_REGISTRY_PUSH_USER}:${PRIVATE_REGISTRY_PUSH_PASSWORD} https://${PRIVATE_REGISTRY_LOCATION}/v2/${IMG}/tags/list |  jq .



./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage add-cred-to-global-pull-secret \ ${PRIVATE_REGISTRY_LOCATION} \ ${PRIVATE_REGISTRY_PULL_USER} \ ${PRIVATE_REGISTRY_PULL_PASSWORD}

oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .

oc get secret/pull-secret -n openshift-config -o json | jq -r '.data.".dockerconfigjson"' | base64 -d



source ~/cpd_vars.sh

SOURCE_IMG="icr.io/cpopen/ibm-cpd-scheduling-operator:1.3.6-20220606.001060.efdb112-amd64"

TARGET_IMG="${PRIVATE_REGISTRY_LOCATION}/cpopen/ibm-cpd-scheduling-operator:1.3.6-20220606.001060.efdb112-amd64"

AUTH="/root/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work/.airgap/auth.json"

skopeo copy --all --authfile ${AUTH} --dest-tls-verify=false --src-tls-verify=false docker://${SOURCE_IMG} docker://${TARGET_IMG}

curl -k -u ${PRIVATE_REGISTRY_PUSH_USER}:${PRIVATE_REGISTRY_PUSH_PASSWORD} https://${PRIVATE_REGISTRY_LOCATION}/v2/_catalog?n=6000 |  jq .

```
cat << EOF | oc apply -f -
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: test-mirror
spec:
  repositoryDigestMirrors:
  - mirrors:
    - ${PRIVATE_REGISTRY_LOCATION}/sig-storage
    source: k8s.gcr.io/sig-storage
EOF


oc get imageContentSourcePolicy


kubectl --namespace nfs-storage create secret docker-registry priv-reg  --docker-server=10.40.10.22:5000 --docker-username=dockeruser --docker-password=dockeruser --docker-email="a@b.c"

cat << EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: mypod-container
      image: 10.40.10.22:5000/sig-storage/nfs-subdir-external-provisioner:v4.0.2
  imagePullSecrets:
    - name: priv-reg      
EOF
```







AUTH="/root/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work/.airgap/auth.json"

source ~/cpd_vars.sh && cd ~/cpd-cli

COMPONENTS="openscale"

OFFLINEDIR="/root/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work/offline/4.5.1"

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage list-images --components=${COMPONENTS} --release=${VERSION} --inspect_source_registry=true

tail -q -n +2 ${OFFLINEDIR}/*-images.csv | while IFS="," read registry image_name tag digest mtype os arch variant insecure digest_source; do
  if [ "$arch" != "ppc64le" ] && [ "$arch" != "s390x" ]; then
    echo "sudo skopeo copy --all --authfile $AUTH --dest-tls-verify=false --src-tls-verify=false docker://${registry}/${image_name}@${digest} docker://${PRIVATE_REGISTRY_LOCATION}/${image_name}@${digest} ${arch}"
    sudo skopeo copy --all --authfile $AUTH --dest-tls-verify=false --src-tls-verify=false \
    docker://${registry}/${image_name}@${digest} docker://${PRIVATE_REGISTRY_LOCATION}/${image_name}@${digest}
  fi
done

### Mirror images



```
source ~/cpd_vars.sh && cd ~/cpd-cli
OFFLINEDIR="/root/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work/offline/4.5.1"
AUTH="/root/auth.json"

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true

COMPONENTS="ws"
./cpd-cli manage list-images --components=${COMPONENTS} --release=${VERSION} --inspect_source_registry=true

OFFLINEDIR=${OFFLINEDIR}/${COMPONENTS} && echo ${OFFLINEDIR}

tail -q -n +2 $(find $OFFLINEDIR -type f -name "*-images.csv" ) | while IFS="," read registry image_name tag digest mtype os arch variant insecure digest_source; do
  if [ "$arch" != "ppc64le" ] && [ "$arch" != "s390x" ]; then
    echo "Processing ${registry}/${image_name}:${tag}  ${arch} ..."
    echo "sudo skopeo copy --all --authfile $AUTH --dest-tls-verify=false --src-tls-verify=false docker://${registry}/${image_name}:${tag} docker://${PRIVATE_REGISTRY_LOCATION}/${image_name}:${tag}"
sudo skopeo copy --all --authfile $AUTH --dest-tls-verify=false --src-tls-verify=false \
docker://${registry}/${image_name}:${tag} docker://${PRIVATE_REGISTRY_LOCATION}/${image_name}:${tag}
  fi
done

# Count images in private registry
curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY_LOCATION}/v2/_catalog?n=6000 | jq '.repositories[]' | wc -l

IMG_PATTERN="nginx"

find ${OFFLINEDIR} -type f -exec grep -q ${IMG_PATTERN} '{}' \; -print

IMG="icr.io/cpopen/cpfs/icp4data-nginx-repo:4.5.0-236-amd64"

skopeo inspect --authfile /root/auth.json docker://cp.icr.io/cp/cpd/wkc-factsheets-ui@sha256:d8d0efdfaf98d6c85771e5352640f2942797ecf1258dc34e861218f195e588ab | jq -r '[.Architecture,  .Name,  .Digest] | join(";")'

skopeo inspect --authfile /root/auth.json docker://cp.icr.io/cp/cpd/wkc-factsheets-ui@sha256:d8d0efdfaf98d6c85771e5352640f2942797ecf1258dc34e861218f195e588ab | jq -r '[.Architecture,  .Name,  .Digest] | join(";")'

cat pull-wkc.sh | awk '{ print "skopeo inspect --authfile /root/auth.json docker://" $5 }' | sh | jq -r '[.Architecture,  .Name,  .Digest] | join(";")' | tee -a wkc-allArch.csv

sed '/^s390x\|^ppc64le/d' wkc-allArch.csv | tee wkc-amd64.csv

for img in $(cat wkc-amd64.csv); do echo $img | awk -F";" '{print "podman pull --authfile /root/auth.json " $2 "@" $3 }' | sh; done

```



### Work on images

```
cat cpd.csv | grep '^+' | awk '{print "skopeo inspect --authfile /root/auth.json " $9}' | sh | jq -r '[.Architecture,  .Name,  .Digest] | join(";")' | tee -a cpd-byArch.csv




```





### Check images are mirrored correctly

 ```
 source ~/cpd_vars.sh
 AUTH="/root/auth.json"
 
 IMG=$(curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY_LOCATION}/v2/_catalog?n=6000 | jq -r '.repositories[0]')
 
 echo ${IMG}
 
 FIRST_TAG=$(curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD}  https://${PRIVATE_REGISTRY_LOCATION}/v2/${IMG}/tags/list |  jq -r '.tags[0]')
 
 echo ${FIRST_TAG}
 
 podman pull --authfile $AUTH ${PRIVATE_REGISTRY_LOCATION}/${IMG}:${FIRST_TAG}
 
 DIGEST=$(podman inspect ${PRIVATE_REGISTRY_LOCATION}/${IMG}:${FIRST_TAG} | jq -r '.[]["Digest"]')
 
 echo ${DIGEST}
 
 curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} -X DELETE https://${PRIVATE_REGISTRY_LOCATION}/v2/${IMG}/manifests/${DIGEST}
 
 ```



```
source ~/cpd_vars.sh
AUTH="/root/auth.json"
SRC="docker.io/library/nginx:latest"
IMG="default/nginx"
TAG="latest"

skopeo copy --all --authfile $AUTH --dest-tls-verify=false --src-tls-verify=false \
docker://${SRC} docker://${PRIVATE_REGISTRY_LOCATION}/${IMG}:${TAG}

curl -k -u ${PRIVATE_REGISTRY_PULL_USER}:${PRIVATE_REGISTRY_PULL_PASSWORD} https://${PRIVATE_REGISTRY_LOCATION}/v2/_catalog?n=6000 | grep ${IMG} | jq . 

curl -k -u ${PRIVATE_REGISTRY_PULL_USER}:${PRIVATE_REGISTRY_PULL_PASSWORD}  https://${PRIVATE_REGISTRY_LOCATION}/v2/${IMG}/tags/list |  jq -r '.tags[]'

oc run nginx --image=${PRIVATE_REGISTRY_LOCATION}/${IMG}:${TAG} --restart=Never

oc exec -it nginx -- bash

scp -o StrictHostKeyChecking=no $REGISTRY_DIR/certs/domain.crt root@10.40.10.31:/tmp

ssh -o StrictHostKeyChecking=no -l root 10.40.10.31 "sudo cp -v /tmp/domain.crt /etc/pki/ca-trust/source/anchors/; sudo update-ca-trust"

ssh -o StrictHostKeyChecking=no -l root 10.40.10.31 "sudo trust list | grep -i acpr-tmp-cp4d-install"

ssh -o StrictHostKeyChecking=no -l root 10.40.10.31 "sudo mkdir /etc/containers/certs.d/10.40.10.22:5000; sudo cp -v /tmp/domain.crt /etc/containers/certs.d/10.40.10.22:5000"


NS="nfs-storage"
SECRET="private-reg"

oc --namespace ${NS} create secret docker-registry ${SECRET} --docker-server=${PRIVATE_REGISTRY_LOCATION} --docker-username=${PRIVATE_REGISTRY_PULL_USER} --docker-password=${PRIVATE_REGISTRY_PULL_PASSWORD} --docker-email="a@b.c"

oc patch -n ${NS} serviceaccount/default -p '{"imagePullSecrets":[{"name": "all-icr-io"}, {"name": "private-reg"}]}'


```





    sudo skopeo copy --all --authfile $AUTH --dest-tls-verify=false --src-tls-verify=false \
    docker://${registry}/${image_name}@${digest} docker://${PRIVATE_REGISTRY_LOCATION}/${image_name}@${digest}  fi

tail -q -n +2 ${OFFLINEDIR}/*-images.csv | while IFS="," read registry image_name tag digest mtype os arch variant insecure digest_source; do
  if [ "$arch" != "ppc64le" ] && [ "$arch" != "s390x" ]; then
    echo "${registry}/${image_name}:${tag} "
  fi
done 





tail -q -n +2 ${OFFLINEDIR}/*-images.csv | while IFS="," read registry image_name tag digest mtype os arch variant insecure digest_source; do
  if [ "$arch" != "ppc64le" ] && [ "$arch" != "s390x" ]; then
    echo "${registry}/${image_name}@${digest} "
  fi
done 



curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY_LOCATION}/v2/_catalog?n=6000 | jq .

curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY_LOCATION}/v2/_catalog?n=6000 | jq '.repositories[]' | wc -l



tail -q -n +2 ${OFFLINEDIR}/*-images.csv | while IFS="," read registry image_name tag digest mtype os arch variant insecure digest_source; do
  if [ "$arch" != "ppc64le" ] && [ "$arch" != "s390x" ]; then
    echo "${registry}/${image_name}@${digest} "
  fi
done | wc -l





podman pull --authfile $AUTH ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs/fluentd:v1.14.5-bedrock-3

podman inspect 10.40.10.22:5000/cpopen/cpfs/fluentd:v1.14.5-bedrock-3 | jq -r '.[]["Digest"]'





source ~/cpd_vars.sh

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage apply-icsp ${PRIVATE_REGISTRY_LOCATION}

oc get imageContentSourcePolicy -o json | jq



### Delete WKC

```
oc project default
oc get wkc -n cpd-instance
oc delete wkc wkc-cr -n cpd-instance
sleep 10
oc patch wkc wkc-cr  -n cpd-instance -p '{"metadata":{"finalizers":[]}}' --type=merge
oc delete wkc wkc-cr -n cpd-instance
oc get iis -n  cpd-instance
oc delete pvc volumes-profstgintrnl-pvc

oc delete operandrequests wkc-requests-analyticsengine wkc-requests-ccs wkc-requests-datarefinery wkc-requests-db2uaas wkc-requests-ds wkc-requests-iis iis-requests-db2uaas wkc-cert-mgr-dep -n cpd-instance

oc delete operandrequests wkc-requests-analyticsengine wkc-requests-ccs wkc-requests-datarefinery wkc-requests-db2uaas wkc-requests-ds wkc-requests-iis iis-requests-db2uaas wkc-cert-mgr-dep -n cpd-instance

oc delete operandrequests wkc-requests-mantaflow -n cpd-instance
oc get csv -n ibm-common-services | grep wkc
oc delete csv  ibm-cpd-wkc.v1.6.5 -n ibm-common-services
oc get subscription ibm-cpd-wkc-operator-catalog-subscription -n  ibm-common-services
oc delete subscription ibm-cpd-wkc-operator-catalog-subscription -n ibm-common-services
oc delete catalogsource -n openshift-marketplace ibm-cpd-wkc-operator-catalog
oc delete catalogsource -n openshift-marketplace ibm-cpd-iis-operator-catalog
oc delete catalogsource -n openshift-marketplace manta-adl-operator-catalog
```



### Delete CPD

```
source ~/cpd_vars.sh
./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true
./cpd-cli manage delete-cr --cpd_instance_ns=${PROJECT_CPD_INSTANCE}
./cpd-cli manage delete-olm-artifacts
```



### Update crio

```
NODES=$(oc get node -o wide | awk 'NR>1 && ORS=" " {print $1}') && echo ${NODES}
WORKERS=$(oc get node -o wide | awk '$3=="worker" && NR>1 && ORS=" " {print $1}') && echo ${WORKERS}
MASTERS=$(oc get node -o wide | awk '$3=="master" && NR>1 && ORS=" " {print $6}')
WORKERS="10.40.10.16 10.40.10.17 10.40.10.18 10.40.10.19"
USER="core"

for NODE in ${NODES}; do oc debug node/${NODE} -T -- chroot /host sh -c 'cat /etc/crio/crio.conf | grep "^pids_limit"'; done

for NODE in ${NODES}; do oc debug node/${NODE} -T -- chroot /host sh -c 'cat /etc/crio/crio.conf | grep -A2 "^default_ulimits"'; done

for NODE in ${NODES}; do oc debug node/${NODE} -T -- chroot /host sh -c 'sed -i "/^default_ulimit/a \\\t\"nofile=66560:66560\"" /etc/crio/crio.conf'; done

for NODE in ${NODES}; do oc debug node/${NODE} -T -- chroot /host sh -c 'systemctl restart crio'; done

for NODE in ${NODES}; do oc debug node/${NODE} -T -- chroot /host sh -c 'systemctl status crio | egrep -w "Active:|crio.service"'; done

for WORKER in ${WORKERS}; do ssh ${USER}@${WORKER} "hostname -f; cat /etc/crio/crio.conf | grep pids_limit"; done

sudo tee /etc/crio/crio.conf.d/01-ctrcfg-pidsLimit << EOF
[crio]
  [crio.runtime]
    pids_limit = 16384
EOF

oc debug node/w1.ocp14.iicparis.fr.ibm.com -- bash -lc "chroot /host sudo crio-status config | grep pids_limit"

vi /etc/crio/crio.conf
......
# Maximum number of processes allowed in a container.
pids_limit = 12288
pids_limit = 16384
......

for WORKER in ${WORKERS}; do ssh ${USER}@${WORKER} "hostname -f; cat /etc/crio/crio.conf | grep default_ulimits -A2"; done

vi /etc/crio/crio.conf
......
[crio.runtime]
default_ulimits = [
        "nofile=66536:66536"
]
......

sudo systemctl restart crio
sudo systemctl daemon-reload
systemctl status crio | egrep -w "Active:|crio.service"
```



### Update Kernel

:bulb: [Explainations](https://www.ibm.com/docs/en/db2/11.1?topic=unix-modifying-kernel-parameters-linux)



```
KERNEL_PARMS="kernel.shmall|kernel.shmmax|kernel.shmmni|kernel.sem|kernel.msgmax|kernel.msgmnb|kernel.msgmni|vm.max_map_count"
NODES=$(oc get node -o wide | awk 'NR>1 && ORS=" " {print $1}') && echo ${NODES}
WORKERS=$(oc get node -o wide | awk '$3=="worker" && NR>1 && ORS=" " {print $6}') && echo ${WORKERS}
MASTERS=$(oc get node -o wide | awk '$3=="master" && NR>1 && ORS=" " {print $6}')
WORKERS="10.40.10.16 10.40.10.17 10.40.10.18 10.40.10.19"
USER="root"

for NODE in ${NODES}; do oc debug node/${NODE} -T -- chroot /host sh -c 'sysctl -a 2>/dev/null | egrep -w "'$KERNEL_PARMS'"'; done

for NODE in ${NODES}; do \
oc debug node/${NODE} -T -- chroot /host sh -c \
'tee -a /etc/sysctl.conf << EOF 

kernel.shmmni = 32768
kernel.sem = 250 1024000 100 32768
kernel.msgmax = 65536
kernel.msgmnb = 65536
kernel.msgmni = 32768
EOF' \
; done

for NODE in ${NODES}; do oc debug node/${NODE} -T -- chroot /host sh -c 'tail -10 /etc/sysctl.conf'; done

for NODE in ${NODES}; do oc debug node/${NODE} -T -- chroot /host sh -c 'sysctl -p'; done

for NODE in ${NODES}; do oc debug node/${NODE} -T -- chroot /host sh -c 'KERNEL_PARMS="kernel.shmall|kernel.shmmax|kernel.shmmni|kernel.sem|kernel.msgmax|kernel.msgmnb|kernel.msgmni|vm.max_map_count"
sysctl -a 2>/dev/null | egrep -w $KERNEL_PARMS'; done

for WORKER in $WORKERS; do ssh -o StrictHostKeyChecking=no ${USER}@${WORKER} 'hostname -f; sysctl -a 2>/dev/null | egrep -w "'$KERNEL_PARMS'"' ; done

vi /etc/sysctl.conf

...
kernel.shmmni = 32768
kernel.sem = 250 1024000 100 32768
kernel.msgmax = 65536
kernel.msgmnb = 65536
kernel.msgmni = 32768
...

sysctl -p 
KERNEL_PARMS="kernel.shmall|kernel.shmmax|kernel.shmmni|kernel.sem|kernel.msgmax|kernel.msgmnb|kernel.msgmni|vm.max_map_count"
sysctl -a 2>/dev/null | egrep -w $KERNEL_PARMS 

```



### Configure `kubelet` to allow Db2U to make unsafe sysctl

:bulb: Not available on **roks** because no machineConfigPool exists

```
oc get mcp # No resources found
```



:bulb: Check on each worker with:

```
NODES=$(oc get node -o wide | awk 'NR>1 && ORS=" " {print $6}') && echo ${NODES}
WORKERS=$(oc get node -o wide | awk '$3=="worker" && NR>1 && ORS=" " {print $1}') && echo ${WORKERS}
MASTERS=$(oc get node -o wide | awk '$3=="master" && NR>1 && ORS=" " {print $6}')
WORKERS="10.40.10.15 10.40.10.29 10.40.10.30 10.40.10.31"
USER="core"

for WORKER in $WORKERS; do ssh -o StrictHostKeyChecking=no ${USER}@${WORKER} 'hostname -f; sudo cat /etc/kubernetes/kubelet.conf | jq ".allowedUnsafeSysctls"' ; done


cat /etc/kubernetes/kubelet.conf | jq '.allowedUnsafeSysctls' 
```

> ```
> "allowedUnsafeSysctls" : ["kernel.msg*","kernel.shm*","kernel.sem"]
> ```
>
> or for ROKS append this to end of /etc/kubernetes/kubelet.conf for all workers
>
>  

```
vi /etc/kubernetes/kubelet.conf 

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

systemctl restart kubelet
systemctl status kubelet | egrep -w "Active:|kubelet.service"

for WORKER in $WORKERS; do ssh -o StrictHostKeyChecking=no ${USER}@${WORKER} 'hostname -f; sudo cat /etc/kubernetes/kubelet.conf | grep allowedUnsafeSysctls -A4'; done


```



### Setting up projects (Express)

```
source ~/cpd_vars.sh
oc new-project ${PROJECT_CPFS_OPS}
oc new-project ${PROJECT_CPD_INSTANCE}

NS="${PROJECT_CATSRC}" && echo ${NS}
NS="${PROJECT_CPD_INSTANCE}" && echo ${NS}
NS="${PROJECT_CPD_OPS}" && echo ${NS}
NS="${PROJECT_CPFS_OPS}" && echo ${NS}

# exec for all NS above

SECRET="private-registry"
IBM_SECRET="ibm-registry"

oc project ${NS}

oc --namespace ${NS} create secret docker-registry ${SECRET} --docker-server=${PRIVATE_REGISTRY_LOCATION} --docker-username=${PRIVATE_REGISTRY_USER} --docker-password=${PRIVATE_REGISTRY_PASSWORD} --docker-email="a@b.c"

oc --namespace ${NS} create secret docker-registry ${IBM_SECRET} --docker-server=${IBM_REGISTRY_LOCATION} --docker-username=${IBM_REGISTRY_USER} --docker-password=${IBM_REGISTRY_PASSWORD} --docker-email="a@b.c"



oc patch -n ${NS} serviceaccount/default -p '{"imagePullSecrets":[{"name": "'${SECRET}'"}, {"name": "'${IBM_SECRET}'"}]}'

# oc patch -n ${NS} serviceaccount/default -p '{"imagePullSecrets":[]}'

oc get serviceaccount/default -o json

```



### Set workers to trust private registry

```
source ~/cpd_vars.sh

WORKERS="10.40.10.16 10.40.10.17 10.40.10.18 10.40.10.19"
WORKERS=$(oc get node -o wide | awk '$3=="worker" && NR>1 && ORS=" " {print $1}') && echo ${WORKERS}
USER="core"

PRIV_REGISTRY_HOST="acpr-tmp-cp4d-install"
PRIV_REGISTRY_HOST="lb.ocp9"
NODE="w5.ocp9.iicparis.fr.ibm.com" && echo ${NODE}

scp -o StrictHostKeyChecking=no $PRIVATE_REGISTRY_DIR/certs/domain.crt ${USER}@${NODE}:/tmp

ssh -o StrictHostKeyChecking=no -l ${USER} ${NODE} "sudo cp -v /tmp/domain.crt /etc/pki/ca-trust/source/anchors/; sudo update-ca-trust"

ssh -o StrictHostKeyChecking=no -l ${USER} ${NODE} "sudo trust list | grep -i "${PRIV_REGISTRY_HOST}

ssh -o StrictHostKeyChecking=no -l ${USER} ${NODE} "sudo mkdir /etc/containers/certs.d/"${PRIVATE_REGISTRY_LOCATION}"; sudo cp -v /tmp/domain.crt /etc/containers/certs.d/"${PRIVATE_REGISTRY_LOCATION}
```



### Test if worker are able to create pod using private registry in each CP4DATA NS

```
source ~/cpd-cli/cpd-vars.sh
AUTH="/root/auth.json"
IMG="docker.io/library/nginx:latest"
IMG="icr.io/cpopen/cpfs/icp4data-nginx-repo@sha256:88a817144ba25663160b80739e3f91d0c807de1702614a85e76fed931fd7134a"

IMG_PATTERN="default/nginx"
TAG="latest"
IMG="${PRIVATE_REGISTRY_LOCATION}/${IMG_PATTERN}:${TAG}"

skopeo copy --all --authfile $AUTH --dest-tls-verify=false --src-tls-verify=false \
docker://${SRC} docker://${PRIVATE_REGISTRY_LOCATION}/${IMG_PATTERN}:${TAG}

skopeo inspect --authfile ${AUTH} docker://${PRIVATE_REGISTRY_LOCATION}/${IMG_PATTERN}:${TAG}

# oc run nginx --image=nginx --restart=Never

WORKERS=$(oc get node -o wide | awk '$3=="worker" && NR>1 && ORS=" " {print $1}') && echo ${WORKERS}

WORKERS="10.40.10.16 10.40.10.17 10.40.10.18 10.40.10.19"
NODE="w5.ocp9.iicparis.fr.ibm.com" && echo ${NODE}
POD="nginx" && echo ${POD}
NS="${PROJECT_CATSRC}" && echo ${NS}
NS="${PROJECT_CPD_INSTANCE}" && echo ${NS}
NS="${PROJECT_CPD_OPS}" && echo ${NS}
NS="${PROJECT_CPFS_OPS}" && echo ${NS}

# exec for each NS on each NODE above

for WORKER in $WORKERS; do scp -o StrictHostKeyChecking=no ${AUTH} ${USER}@${WORKER}:/home/${USER}/auth.json ; done


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



### Creating the custom security context constraint for Watson Knowledge Catalog

```
source ~/cpd_vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}
#./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true


./cpd-cli manage apply-scc --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --components=wkc

oc get scc wkc-iis-scc

oc adm policy who-can use scc wkc-iis-scc --namespace ${PROJECT_CPD_INSTANCE} | grep "wkc-iis-sa"
```



### Updating the global image pull secret

```
source ~/cpd_vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true
#./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true


./cpd-cli manage add-cred-to-global-pull-secret ${PRIVATE_REGISTRY_LOCATION} ${PRIVATE_REGISTRY_USER}  ${PRIVATE_REGISTRY_PASSWORD}

oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .


oc get secret all-icr-io -n default -o json | jq .

```



### Configure an image content source policy

```
source ~/cpd_vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true
#./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true


./cpd-cli manage apply-icsp ${PRIVATE_REGISTRY_LOCATION}

oc get imageContentSourcePolicy cloud-pak-for-data-mirror -o json | jq -r '.spec.repositoryDigestMirrors[]'

{
  "mirrors": [
    "10.40.10.22:5000/opencloudio"
  ],
  "source": "quay.io/opencloudio"
}
{
  "mirrors": [
    "10.40.10.22:5000/cp"
  ],
  "source": "cp.icr.io/cp"
}
{
  "mirrors": [
    "10.40.10.22:5000/cp/cpd",
    "10.40.10.22:5000",
  ],
  "source": "cp.icr.io/cp/cpd"
}
{
  "mirrors": [
    "10.40.10.22:5000/cpopen"
  ],
  "source": "icr.io/cpopen"
}
{
  "mirrors": [
    "10.40.10.22:5000/cpopen/cpfs"
  ],
  "source": "icr.io/cpopen/cpfs"
}
{
  "mirrors": [
    "10.40.10.22:5000/db2u"
  ],
  "source": "icr.io/db2u"
}


```



### Configure an image content source policy for ROKS

> Append to /etc/containers/registries.conf worker nodes

```
PRIVATE_REGISTRY="????"

cat << EOF | sudo tee -a

[[registry]]
  location = "quay.io/opencloudio"
  insecure = false
  blocked = false
  mirror-by-digest-only = false
  prefix = ""

  [[registry.mirror]]
  location = "${PRIVATE_REGISTRY}/opencloudio"
  insecure = false

[[registry]]
  location = "cp.icr.io/cp"
  insecure = false
  blocked = false
  mirror-by-digest-only = false
  prefix = ""

  [[registry.mirror]]
  location = "${PRIVATE_REGISTRY}/cp"
  insecure = false

[[registry]]
  location = "cp.icr.io/cp/cpd"
  insecure = false
  blocked = false
  mirror-by-digest-only = false
  prefix = ""

  [[registry.mirror]]
  location = "${PRIVATE_REGISTRY}/cp/cpd"
  insecure = false

[[registry]]
  location = "icr.io/cpopen"
  insecure = false
  blocked = false
  mirror-by-digest-only = false
  prefix = ""

  [[registry.mirror]]
  location = "${PRIVATE_REGISTRY}/cpopen"
  insecure = false

[[registry]]
  location = "icr.io/cpopen/cpf"
  insecure = false
  blocked = false
  mirror-by-digest-only = false
  prefix = ""

  [[registry.mirror]]
  location = "${PRIVATE_REGISTRY}/cpopen/cpf"
  insecure = false

[[registry]]
  location = "icr.io/db2u"
  insecure = false
  blocked = false
  mirror-by-digest-only = false
  prefix = ""

  [[registry.mirror]]
  location = "${PRIVATE_REGISTRY}/db2u"
  insecure = false
EOF
```



### Get a image digest to test mirroring from nodes

```
source ~/cpd_vars.sh
OFFLINEDIR="/root/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work/offline/4.5.1"
AUTH="/root/auth.json"
IMG_PATTERN="cp/cpd/ibm-cpd-scheduler-webhook"
IMG_PATTERN="db2u/db2u"
IMG_PATTERN="cpopen/cpfs/privatecloud-usermgmt"
ARCH="amd64"

FILE=$(find ${OFFLINEDIR} -type f -name "*-images.csv" -exec grep -q ${IMG_PATTERN} '{}' \; -print | awk 'NR==1 {print $0}') && echo ${FILE}

SRC="$(grep "${IMG_PATTERN}" $FILE | grep ${ARCH} | awk -F ',' 'NR==1 {print $1 "/" $2 ":" $3}')" && echo ${SRC}

FIRST_TAG=$(curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD}  https://${PRIVATE_REGISTRY_LOCATION}/v2/${IMG_PATTERN}/tags/list |  jq -r '.tags[0]') && echo ${FIRST_TAG}

IMG="${IMG_PATTERN}:${FIRST_TAG}" && echo ${IMG}

skopeo copy --all --authfile $AUTH --dest-tls-verify=false --src-tls-verify=false docker://${SRC} docker://${PRIVATE_REGISTRY_LOCATION}/${IMG}

skopeo inspect --authfile ${AUTH} docker://${PRIVATE_REGISTRY_LOCATION}/${IMG} | jq -r '.Name'

podman pull --authfile $AUTH ${PRIVATE_REGISTRY_LOCATION}/${IMG}

DIGEST=$(podman inspect ${PRIVATE_REGISTRY_LOCATION}/${IMG} | jq -r '.[]["Digest"]') && echo ${DIGEST}

echo ${SRC}
echo ${IMG}
echo ${IMG_PATTERN}@${DIGEST}
```





### Check mirror work correctly

To be done based on what's explained [here](https://docs.openshift.com/container-platform/4.10/openshift_images/image-configuration.html#images-configuration-registry-mirror_image-configuration)

```
source ~/cpd_vars.sh
AUTH="/root/auth.json"

NODES=$(oc get node -o wide | awk 'NR>1 && ORS=" " {print $6}') && echo ${NODES}
WORKERS=$(oc get node -o wide | awk '$3=="worker" && NR>1 && ORS=" " {print $1}')
MASTERS=$(oc get node -o wide | awk '$3=="master" && NR>1 && ORS=" " {print $6}')
WORKERS="10.40.10.16 10.40.10.17 10.40.10.18 10.40.10.19"
USER="core"

for WORKER in $WORKERS; do scp -o StrictHostKeyChecking=no ${AUTH} ${USER}@${WORKER}:/home/${USER}/auth.json ; done


# On nodes


sudo subscription-manager repos --enable=rhel-7-server-extras-rpms
sudo yum -y install podman

sudo trust list | grep -i acpr-tmp-cp4d-install

# Get var below from Get a image digest to test mirroring from nodes

REG="icr.io"
IMG_TAG="${REG}/db2u/db2u:11.5.7.0-cn5-6346-amd64"
IMG_DIGEST="${REG}/db2u/db2u@sha256:3aad18d74db3c6df40589d2c9cca46f76350248adfa3ac9121d7c5a6d798efbe"

skopeo inspect --authfile ~/auth.json docker://${IMG_TAG} | jq -r '.Name'
skopeo inspect --authfile ~/auth.json docker://${IMG_DIGEST} | jq -r '.Name'

REG="cp.icr.io"
IMG_TAG="${REG}/cp/cpd/ibm-cpd-scheduler-webhook:1.4.0-20220706.000059.549bdea"
IMG_DIGEST="${REG}/cp/cpd/ibm-cpd-scheduler-webhook@sha256:70c71162c2fad458e994dfae101d9e1c2bcb8560982659b536af8dcb97872bfc"

skopeo inspect --authfile ~/auth.json docker://${IMG_TAG} | jq -r '.Name'
skopeo inspect --authfile ~/auth.json docker://${IMG_DIGEST} | jq -r '.Name'

podman pull --authfile ~/auth.json --log-level=debug ${IMG_TAG}


WORKERS="10.40.10.16 10.40.10.17 10.40.10.18 10.40.10.19"
USER="root"

for WORKER in $WORKERS; do scp -o StrictHostKeyChecking=no cpd-amd64.csv ${USER}@${WORKER}:/root ; done

for img in $(cat cpd-amd64.csv); do echo $img | awk -F";" '{print "skopeo inspect --authfile /root/auth.json docker://"$2 "@" $3 " | jq -r .Name"}' | sh; done

cat pull-cpd.sh | awk '{print "skopeo inspect --authfile /root/auth.json docker://"$5}' | sh | jq .Name

for img in $(cat cpd-amd64.csv); do echo $img | awk -F";" '{print "podman pull --authfile /root/auth.json " $2 "@" $3 }' | sh; done

df -h /
df -h /var/data/criorootstorage

```



### Create AUTH

```
source ~/cpd_vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true
#./cpd-cli manage login-to-ocp --username ${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL} --insecure-skip-tls-verify=true

./cpd-cli manage add-cred-to-global-pull-secret ${PRIVATE_REGISTRY_LOCATION} ${PRIVATE_REGISTRY_USER} ${PRIVATE_REGISTRY_PASSWORD}

./cpd-cli manage add-cred-to-global-pull-secret ${IBM_REGISTRY_LOCATION} ${IBM_REGISTRY_USER} ${IBM_REGISTRY_PASSWORD}

AUTH="/root/auth.json"
oc get secret/pull-secret -n openshift-config -o json | jq -r '.data.".dockerconfigjson"' | base64 -d | tee $AUTH
```



### Increase node boot partition to 250GB

```

# START To do one time only
curl -fsSL https://clis.cloud.ibm.com/install/linux | sh

USERID="sebastien.gautier@fr.ibm.com"
PASSWD=""

ibmcloud login -u $USERID -p $PASSWD --sso --no-region

APIKEY_NAME="apikey0"
ibmcloud iam api-key-create $APIKEY_NAME -d "$APIKEY_NAME" --file ~/$APIKEY_NAME

ibmcloud logout

# END To do one time only

APIKEY_NAME="apikey0"

ibmcloud login --apikey @~/$APIKEY_NAME --no-region

ibmcloud plugin repo-plugins -r 'IBM Cloud'

ibmcloud plugin install vpc-infrastructure
ibmcloud plugin install container-service

ibmcloud resource groups --output json | jq -r '.[].name'
ibmcloud regions --output JSON | jq -r .[].Name

REGION="eu-de"
GROUP="acpr-workload"

ibmcloud target -r ${REGION} -g ${GROUP}

alias ic="/usr/local/bin/ibmcloud"

ic oc clusters

ic oc worker ls --cluster acpr-workload-ocp

ibmcloud iam oauth-tokens

ibmcloud is target --gen 2

ic is vols
ZONE="eu-de-1"
VOLUME="data19"

ibmcloud is volume-create ${VOLUME} general-purpose ${ZONE} --capacity 300
ic oc clusters
ic oc worker ls --cluster acpr-workload-ocp

ibmcloud oc storage attachment ls --cluster acpr-workload-ocp --worker kube-cbdb97uf07qqj6kv8oeg-acprworkloa-acprwor-00000611

[root@acpr-tmp-cp4d-install ~]# ibmcloud oc storage attachment ls --cluster acpr-workload-ocp --worker kube-cbdb97uf07qqj6kv8oeg-acprworkloa-acprwor-00000611
Listing volume attachments...
OK
ID                                          Name                                     Status     Type   Volume ID                                   Volume Name                                                   Worker ID   
02b7-8b163252-3301-4989-a127-f9cbc6cf226b   exceeding-overtake-appealing-gigahertz   attached   boot   r010-096d8b70-bbe6-440f-8ff9-c71ac9818de4   kube-cbdb97uf07qqj6kv8oeg-acprworkloa-acprwor-00000611-boot   kube-cbdb97uf07qqj6kv8oeg-acprworkloa-acprwor-00000611   

ibmcloud oc clusters
CLUSTER=""
ibmcloud oc worker ls --cluster ${CLUSTER}
WORKER_ID=""
VOLUME=""
ibmcloud is vol ${VOLUME}
VOLUME_ID=""

ibmcloud oc storage attachment create --cluster ${CLUSTER} --volume ${VOLUME_ID} --worker ${WORKER_ID}


DISK=/dev/vdd
PART=/dev/vdd1
VG=data
LV=data
MOUNT_POINT="/var/data/criorootstorage"

(
echo n
echo p
echo 1
echo
echo
echo w
) | fdisk $DISK

# sudo subscription-manager repos --enable=rhel-7-server-extras-rpms

[ -z "$(command -v pvcreate)" ] && yum install -y lvm2 || echo "pvcreate already installed"
pvcreate ${PART}

vgcreate ${VG} ${PART}

EXT=$(pvdisplay $PART | awk '/Free PE/ {print $3}')
lvcreate -l ${EXT} -n ${LV} ${VG}
lvs

mkfs.ext4 /dev/${VG}/${LV}

echo "/dev/mapper/${VG}-${LV}   ${MOUNT_POINT}   ext4    defaults 0 0" | tee -a /etc/fstab


```





### Delete pod in terminating state

```
POD="ibm-vpc-block-csi-node-zdngv"
NS="kube-system"

oc delete pods ${POD} --grace-period=0 --force

oc get po -n ${NS} | awk 'NR>1 {print "oc delete pods " $1 " --grace-period=0 --force"}' | sh
```





### Remove images not used for containers

```
podman rmi $(podman images -q)
```



### Images on ROKS at Node Startup

```
REPOSITORY                                         TAG                                        IMAGE ID       CREATED         SIZE
icr.io/ext/logdna-agent                            stable                                     8b64c1e378ab   2 days ago      337 MB
de.icr.io/armada-master/ibm-vpc-block-csi-driver   v4.3.5                                     3edebf109290   8 days ago      596 MB
icr.io/obs/armada-storage-secret                   v1.1.13                                    184ea6c28ae5   8 days ago      140 MB
de.icr.io/armada-master/haproxy                    6514a23e8f51a2b11713623f3fa94a0212bf25c9   5ad3b0f56e03   12 days ago     295 MB
icr.io/ext/sysdig/agent                            latest                                     04f8dc9256a2   3 weeks ago     1.9 GB
icr.io/cpopen/ibm-cpd-platform-operator-catalog    <none>                                     7cf01c68a4f5   3 weeks ago     204 MB
de.icr.io/armada-master/vpn-client                 2.5.6-r1-IKS-629                           8aa50c764771   4 weeks ago     10.7 MB
de.icr.io/armada-master/keepalived-watcher         2058                                       4b27690ca53e   5 weeks ago     46.2 MB
de.icr.io/armada-master/armada-calico-extension    997                                        88eb9c599892   5 weeks ago     185 MB
icr.io/cpopen/ibm-cpd-scheduler-operator-catalog   <none>                                     97049b28386d   6 weeks ago     155 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     06be469c8cd0   7 weeks ago     498 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     11274ee0147d   8 weeks ago     377 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     8ca597d537ba   8 weeks ago     413 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     dd2f16dd98d7   8 weeks ago     748 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     39c4bfbeb3b9   8 weeks ago     402 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     82a4a182c2bc   8 weeks ago     458 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     e5e1fc4b14d9   8 weeks ago     474 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     78923a35ee84   8 weeks ago     426 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     30f18e2744ac   8 weeks ago     464 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     52ad3b73e8bb   8 weeks ago     402 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     f375222835d3   8 weeks ago     409 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     a29a04bc7309   8 weeks ago     338 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     6c85fcda8744   8 weeks ago     551 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     7a1026e8c6a9   8 weeks ago     370 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     2196775698ec   8 weeks ago     370 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     e9ece310cab6   8 weeks ago     471 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     74695c709d29   8 weeks ago     319 MB
quay.io/openshift-release-dev/ocp-v4.0-art-dev     <none>                                     8b1e83f9d529   8 weeks ago     306 MB
de.icr.io/armada-master/pause                      3.7                                        692cfaed9d08   4 months ago    718 kB
de.icr.io/armada-master/calico/node                v3.22.2                                    ef5e13a25c96   4 months ago    200 MB
de.icr.io/armada-master/calico/cni                 v3.22.2                                    4544db6553c5   4 months ago    237 MB
icr.io/ext/sig-storage/livenessprobe               v2.3.0                                     2da31ea0f7e6   12 months ago   18.4 MB
icr.io/ext/sig-storage/csi-node-driver-registrar   v2.2.0                                     10c83cb7b1e0   12 months ago   19.9 MB

```



### Troubeshooting

```
If
Error: could not get runtime: lock "/data/containers/overlay-layers/layers.lock" is not a read-only lock
then
chmod 644 /data/containers/overlay-layers/layers.lock
```



### Install WML

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli
./cpd-cli manage login-to-ocp --username=${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL}

./cpd-cli manage apply-olm --release=${VERSION} --cpd_operator_ns=${PROJECT_CPD_OPS} --components=wml

oc get catalogsource -n ${PROJECT_CATSRC}
oc get po -n ${PROJECT_CATSRC}
oc get sub -n ${PROJECT_CPFS_OPS}

SUB="ibm-cpd-wml-operator"

SUB=$(oc get sub -n ${PROJECT_CPFS_OPS} ${SUB} -o jsonpath='{.status.installedCSV} {"\n"}') && echo $SUB

oc get csv -n ${PROJECT_CPFS_OPS} $SUB -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ${PROJECT_CPFS_OPS} -l olm.owner="$SUB" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"
```



```
./cpd-cli manage apply-cr \
--components=wml \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true

oc get crd | grep -i wml
> wmlbases.wml.cpd.ibm.com                                          2022-08-31T12:17:12Z
oc get crd wmlbases.wml.cpd.ibm.com -o json | jq -r '.spec.names.kind'
> WmlBase
oc get WmlBase
> NAME     VERSION   BUILD        STATUS       RECONCILED   AGE
> wml-cr   4.5.1     4.5.1-3500   InProgress                39m
oc get WmlBase wml-cr -n ${PROJECT_CPD_INSTANCE} -o json | jq -r .status.wmlStatus
> InProgress


```



### Install DP

```
source ~/cpd_vars.sh && cd ~/cpd-cli
./cpd-cli manage login-to-ocp --username=${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL}

./cpd-cli manage apply-olm --release=${VERSION} --cpd_operator_ns=${PROJECT_CPD_OPS} --components=dp

oc get catalogsource -n ${PROJECT_CATSRC}
oc get po -n ${PROJECT_CATSRC}
oc get sub -n ${PROJECT_CPFS_OPS}

SUB="ibm-cpd-dp-operator-catalog-subscription"

SUB=$(oc get sub -n ${PROJECT_CPFS_OPS} ${SUB} -o jsonpath='{.status.installedCSV} {"\n"}') && echo $SUB

oc get csv -n ${PROJECT_CPFS_OPS} $SUB -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ${PROJECT_CPFS_OPS} -l olm.owner="$SUB" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"

```



```
./cpd-cli manage apply-cr \
--components=dp \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true

oc get crd | grep -i dp
> dp.dp.cpd.ibm.com                                                 2022-08-31T13:40:02Z
oc get crd dp.dp.cpd.ibm.com -o json | jq -r '.spec.names.kind'
> DP
oc get DP
> NAME    AGE
> dp-cr   109s
oc get DP dp-cr -n ${PROJECT_CPD_INSTANCE} -o json | jq -r .status.dpStatus
> InProgress
```



### Install DMC

```
source ~/cpd_vars.sh && cd ~/cpd-cli
./cpd-cli manage login-to-ocp --username=${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL}

COMPONENT="dmc"

./cpd-cli manage apply-olm --release=${VERSION} --cpd_operator_ns=${PROJECT_CPD_OPS} --components=${COMPONENT}

oc get catalogsource -n ${PROJECT_CATSRC}
oc get po -n ${PROJECT_CATSRC}
oc get sub -n ${PROJECT_CPFS_OPS}

SUB="ibm-dmc-operator-subscription"

SUB=$(oc get sub -n ${PROJECT_CPFS_OPS} ${SUB} -o jsonpath='{.status.installedCSV} {"\n"}') && echo $SUB

oc get csv -n ${PROJECT_CPFS_OPS} $SUB -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ${PROJECT_CPFS_OPS} -l olm.owner="$SUB" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"

```



```
./cpd-cli manage apply-cr \
--components=${COMPONENT} \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--license_acceptance=true

oc get crd | grep -i ${COMPONENT}
> dmc4ocps.dmc.databases.ibm.com                                    2022-08-31T14:30:36Z
> dmcaddons.dmc.databases.ibm.com                                   2022-08-31T14:30:37Z
> dmcs.dmc.databases.ibm.com                                        2022-08-31T14:30:37Z


oc get crd dmc4ocps.dmc.databases.ibm.com -o json | jq -r '.spec.names.kind'
> Dmc4ocp
oc get crd dmcaddons.dmc.databases.ibm.com -o json | jq -r '.spec.names.kind'
> Dmcaddon
oc get crd dmcs.dmc.databases.ibm.com -o json | jq -r '.spec.names.kind'
> Dmc

oc get Dmcaddon
> NAME        VERSION   STATUS      AGE
> dmc-addon   4.5.1     Completed   4m6s

oc get Dmcaddon dmc-addon -n ${PROJECT_CPD_INSTANCE} -o json | jq -r .status.dmcAddonStatus
```



### Installing DB2OLTP

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli
./cpd-cli manage login-to-ocp --username=${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL}

COMPONENT="db2oltp"

./cpd-cli manage apply-olm --release=${VERSION} --cpd_operator_ns=${PROJECT_CPD_OPS} --components=${COMPONENT}

oc get catalogsource -n ${PROJECT_CATSRC}
oc get po -n ${PROJECT_CATSRC}
oc get sub -n ${PROJECT_CPFS_OPS}

SUB="ibm-db2oltp-cp4d-operator-catalog-subscription"

SUB=$(oc get sub -n ${PROJECT_CPFS_OPS} ${SUB} -o jsonpath='{.status.installedCSV} {"\n"}') && echo $SUB

oc get csv -n ${PROJECT_CPFS_OPS} $SUB -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ${PROJECT_CPFS_OPS} -l olm.owner="$SUB" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"

```



```
echo ./cpd-cli manage apply-cr \
--components=${COMPONENT} \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--license_acceptance=true

COMPONENT="db2oltp"
oc get crd | grep -i ${COMPONENT}
> db2oltpservices.databases.cpd.ibm.com                             2022-08-31T14:53:39Z
oc get crd db2oltpservices.databases.cpd.ibm.com -o json | jq -r '.spec.names.kind'
> Db2oltpService
oc get Db2oltpService
> NAME         AGE
> db2oltp-cr   2m41s
oc get Db2oltpService db2oltp-cr -n ${PROJECT_CPD_INSTANCE} -o json | jq -r .status.db2oltpStatus
> InProgress
```



### Installing WS

```
source ~/cpd_vars.sh && cd ~/cpd-cli
./cpd-cli manage login-to-ocp --username=${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL}

COMPONENT="ws"

./cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--components=${COMPONENT}

oc get catalogsource -n ${PROJECT_CATSRC}
oc get po -n ${PROJECT_CATSRC}
oc get sub -n ${PROJECT_CPFS_OPS}

SUB="ibm-cpd-ws-operator"

SUB=$(oc get sub -n ${PROJECT_CPFS_OPS} ${SUB} -o jsonpath='{.status.installedCSV} {"\n"}') && echo $SUB

oc get csv -n ${PROJECT_CPFS_OPS} $SUB -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ${PROJECT_CPFS_OPS} -l olm.owner="$SUB" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"
```



```
./cpd-cli manage apply-cr \
--components=${COMPONENT} \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true

oc get crd | grep -i ${COMPONENT}
> ws.ws.cpd.ibm.com                                                 2022-08-31T15:22:07Z
oc get crd ws.ws.cpd.ibm.com -o json | jq -r '.spec.names.kind'
> WS
oc get WS
> NAME    VERSION   RECONCILED   STATUS       AGE
> ws-cr   4.5.1                  InProgress   101s
oc get WS ws-cr -n ${PROJECT_CPD_INSTANCE} -o json | jq -r .status.wsStatus
> InProgress
```



### Installing OPENSCALE

```
source ~/cpd_vars.sh && cd ~/cpd-cli
./cpd-cli manage login-to-ocp --username=${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL}

COMPONENT="openscale"

./cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--components=${COMPONENT}

oc get catalogsource -n ${PROJECT_CATSRC} | grep -i ${COMPONENT}
oc get po -n ${PROJECT_CATSRC} | grep -i ${COMPONENT}
oc get sub -n ${PROJECT_CPFS_OPS} | grep -i ${COMPONENT}

SUB="ibm-watson-openscale-operator-subscription"

SUB=$(oc get sub -n ${PROJECT_CPFS_OPS} ${SUB} -o jsonpath='{.status.installedCSV} {"\n"}') && echo $SUB

oc get csv -n ${PROJECT_CPFS_OPS} $SUB -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ${PROJECT_CPFS_OPS} -l olm.owner="$SUB" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"
```



```
./cpd-cli manage apply-cr \
--components=${COMPONENT} \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true

oc get crd | grep wos
> woservices.wos.cpd.ibm.com                                        2022-08-31T15:46:02Z
oc get woservices.wos.cpd.ibm.com -o json | jq -r '.items[].kind'
> WOService
oc get WOService
> NAME          TYPE      STORAGE   SCALECONFIG   PHASE        RECONCILED   STATUS
> aiopenscale   service             small         Installing                Installing
oc get WOService aiopenscale -o json | jq -r '.status.wosStatus'
> Installing
```



### Installing Watson Discovery OLM

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli
./cpd-cli manage login-to-ocp --username=${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL}

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true

COMPONENTS="watson_discovery"

./cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--components=${COMPONENTS}

oc get catalogsource -n ${PROJECT_CATSRC} 
oc get po -n ${PROJECT_CATSRC} 
oc get sub -n ${PROJECT_CPFS_OPS} 

SUB="ibm-watson-discovery-operator-subscription"

SUB=$(oc get sub -n ${PROJECT_CPFS_OPS} ${SUB} -o jsonpath='{.status.installedCSV} {"\n"}') && echo $SUB

oc get csv -n ${PROJECT_CPFS_OPS} $SUB -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ${PROJECT_CPFS_OPS} -l olm.owner="$SUB" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"


```

```
PLAY RECAP *******************************************************************************************************************************
localhost                  : ok=312  changed=40   unreachable=0    failed=0    skipped=295  rescued=0    ignored=0   

Wednesday 14 September 2022  09:57:57 +0000 (0:00:00.031)       0:14:48.274 *** 
=============================================================================== 
utils : confirm operand request cloud-native-postgresql is Running -------------------------------------------------------------- 123.35s
utils : check if installedCSV: ibm-watson-discovery-operator.v4.6.0 'Succeeded' for Subscription: ibm-watson-discovery-operator-subscription - 122.41s
utils : Confirm existence of Catalog Source object "ibm-elasticsearch-catalog" --------------------------------------------------- 67.09s
utils : Confirm existence of Catalog Source object "ibm-watson-discovery-operator-catalog" --------------------------------------- 62.37s
utils : Confirm existence of Catalog Source object "cloud-native-postgresql-catalog" --------------------------------------------- 62.19s
utils : Confirm existence of Catalog Source object "ibm-watson-gateway-operator-catalog" ----------------------------------------- 62.05s
utils : Confirm existence of Catalog Source object "ibm-rabbitmq-operator-catalog" ----------------------------------------------- 61.95s
utils : Confirm existence of Catalog Source object "ibm-etcd-operator-catalog" --------------------------------------------------- 61.95s
utils : Confirm existence of Catalog Source object "ibm-model-train-operator-catalog" -------------------------------------------- 61.91s
utils : Confirm existence of Catalog Source object "ibm-minio-operator-catalog" -------------------------------------------------- 61.86s
utils : install catalog source 'ibm-watson-discovery-operator-catalog' using /tmp/work/offline/4.5.2/watson_discovery/ibm-watson-discovery-4.6.0.tgz  -- 22.15s
utils : install catalog source 'ibm-model-train-operator-catalog' using /tmp/work/offline/4.5.2/watson_discovery/ibm-model-train-operator-1.2.1.tgz  --- 3.48s
utils : install catalog source 'cloud-native-postgresql-catalog' using /tmp/work/offline/4.5.2/watson_discovery/ibm-cloud-native-postgresql-4.6.0.tgz  --- 3.32s
utils : downloading case package ibm-model-train-operator 1.2.1  ------------------------------------------------------------------ 3.27s
utils : install catalog source 'ibm-watson-gateway-operator-catalog' using /tmp/work/offline/4.5.2/watson_discovery/ibm-watson-gateway-operator-2.0.18.tgz  --- 3.27s
utils : Create CPFS namespace if not present ibm-common-services ------------------------------------------------------------------ 3.27s
utils : install catalog source 'ibm-etcd-operator-catalog' using /tmp/work/offline/4.5.2/watson_discovery/ibm-etcd-operator-2.0.17.tgz  --- 3.19s
utils : install catalog source 'ibm-rabbitmq-operator-catalog' using /tmp/work/offline/4.5.2/watson_discovery/ibm-rabbitmq-operator-1.0.19.tgz  --- 3.16s
utils : install catalog source 'ibm-minio-operator-catalog' using /tmp/work/offline/4.5.2/watson_discovery/ibm-minio-operator-1.0.16.tgz  --- 3.11s
utils : downloading case package ibm-watson-discovery 4.6.0  ---------------------------------------------------------------------- 3.09s
[SUCCESS] 2022-09-14T11:57:57.961647Z You may find output and logs in the "/home/fr054721/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work" directory.
[SUCCESS] 2022-09-14T11:57:57.961683Z The apply-olm command ran successfully.

```

```
oc get sub -n ${PROJECT_CPFS_OPS} 
NAME                                                                          PACKAGE                         SOURCE                                  CHANNEL
cloud-native-postgresql-catalog-subscription                                  cloud-native-postgresql         cloud-native-postgresql-catalog         stable
ibm-cert-manager-operator                                                     ibm-cert-manager-operator       ibm-operator-catalog                    v3.20
ibm-common-service-operator                                                   ibm-common-service-operator     ibm-operator-catalog                    v3.20
ibm-elasticsearch-operator-v1.1-ibm-operator-catalog-openshift-marketplace    ibm-elasticsearch-operator      ibm-operator-catalog                    v1.1
ibm-etcd-operator-v1.0-ibm-operator-catalog-openshift-marketplace             ibm-etcd-operator               ibm-operator-catalog                    v1.0
ibm-minio-operator-v1.0-ibm-operator-catalog-openshift-marketplace            ibm-minio-operator              ibm-operator-catalog                    v1.0
ibm-model-train-operator-v1.1-ibm-operator-catalog-openshift-marketplace      ibm-model-train-operator        ibm-operator-catalog                    v1.1
ibm-namespace-scope-operator                                                  ibm-namespace-scope-operator    ibm-operator-catalog                    v3.20
ibm-watson-discovery-operator-subscription                                    ibm-watson-discovery-operator   ibm-watson-discovery-operator-catalog   v4.6
ibm-watson-gateway-operator-v1.0-ibm-operator-catalog-openshift-marketplace   ibm-watson-gateway-operator     ibm-operator-catalog                    v1.0
ibm-zen-operator                                                              ibm-zen-operator                ibm-operator-catalog                    v3.20
operand-deployment-lifecycle-manager-app                                      ibm-odlm                        ibm-operator-catalog                    v3.20
rabbitmq-operator                                                             ibm-rabbitmq-operator           ibm-rabbitmq-operator-catalog           v1.0

```

```
oc get catalogsource -n ${PROJECT_CATSRC} 
NAME                                    DISPLAY                                 TYPE   PUBLISHER      AGE
certified-operators                     Certified Operators                     grpc   Red Hat        39h
cloud-native-postgresql-catalog         Cloud Native Postgresql Catalog         grpc   IBM            9m49s
community-operators                     Community Operators                     grpc   Red Hat        39h
ibm-elasticsearch-catalog               IBM Opencontent Elasticsearch Catalog   grpc   CloudpakOpen   14m
ibm-etcd-operator-catalog               IBM etcd operator Catalog               grpc   IBM            13m
ibm-minio-operator-catalog              IBM Minio Operator Catalog              grpc   IBM            12m
ibm-model-train-operator-catalog        ibm-model-train-operator-catalog        grpc   IBM            15m
ibm-operator-catalog                    ibm-operator-catalog                    grpc   IBM Content    21h
ibm-rabbitmq-operator-catalog           IBM RabbitMQ operator Catalog           grpc   IBM            10m
ibm-watson-discovery-operator-catalog   Watson Discovery                        grpc   IBM            7m15s
ibm-watson-gateway-operator-catalog     IBM Watson Gateway Operator Catalog     grpc   IBM            8m41s
opencloud-operators                     IBMCS Operators                         grpc   IBM            21h
redhat-marketplace                      Red Hat Marketplace                     grpc   Red Hat        39h
redhat-operators                        Red Hat Operators                       grpc   Red Hat        39h

```

### Installing Watson Discovery Service

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true

COMPONENTS="watson_discovery"

tee ~/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work/install-options.yml << EOF
discovery_deployment_type: Starter
EOF

./cpd-cli manage apply-cr \
--components=${COMPONENTS} \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--param-file=/tmp/work/install-options.yml \
--license_acceptance=true


oc get crd | grep watson
> watsondiscoveries.discovery.watson.ibm.com

oc get watsondiscoveries.discovery.watson.ibm.com -o json | jq -r '.items[].kind'
> WatsonDiscovery

oc get WatsonDiscovery
NAME   VERSION   READY   READYREASON    UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE   DATASTOREQUIESCE   AGE
wd     4.5.1     False   Initializing   True       VerifyWait       9/23       0/23                                    88m

oc get WatsonDiscovery wd -o json | jq -r '.status.watsonDiscoveryStatus'
> InProgress


oc get crd | grep watson | awk '{ print "oc get " $1 " -o json | jq -r .items[].kind"}' | sh | tee watsonCrdKind
for KIND in $(cat watsonCrdKind); do oc get ${KIND}; done

> NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           23/23      23/23      NOT_QUIESCED   NOT_QUIESCED       39m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           1/1        1/1        NOT_QUIESCED                      12m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           4/4        4/4        NOT_QUIESCED                      30m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           4/4        4/4        NOT_QUIESCED                      30m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           3/3        3/3        NOT_QUIESCED                      33m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           3/3        3/3        NOT_QUIESCED                      30m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           4/4        4/4        NOT_QUIESCED                      30m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           1/1        1/1        NOT_QUIESCED                      33m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           1/1        1/1        NOT_QUIESCED                      32m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           1/1        1/1        NOT_QUIESCED                      17m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           2/2        2/2        NOT_QUIESCED                      30m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           3/3        3/3        NOT_QUIESCED                      15m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           12/12      12/12      NOT_QUIESCED                      30m
NAME   VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   DEPLOYED   VERIFIED   QUIESCE        DATASTOREQUIESCE   AGE
wd     4.5.1     True    Stable        False      Stable           1/1        1/1        NOT_QUIESCED                      30m
NAME                          VERSION   READY   READYREASON   UPDATING   UPDATINGREASON   AGE
wd-discovery-watson-gateway   main      True    Stable        False      Stable           9m41s


```



```
Monday 19 September 2022  11:04:04 +0000 (0:00:00.032)       0:35:38.187 ******
skipping: [localhost]

PLAY RECAP *****************************************************************************************************************************************
localhost                  : ok=39   changed=5    unreachable=0    failed=0    skipped=59   rescued=0    ignored=0

Monday 19 September 2022  11:04:04 +0000 (0:00:00.024)       0:35:38.212 ******
===============================================================================
utils : check if CR status indicates completion for wd in cpd-instance, max retry 40 times 300s delay ------------------------------------ 2112.32s
utils : waiting for ODLM and Namespace Scope operators to come online ----------------------------------------------------------------------- 6.34s
utils : create a configmap olm-utils-cm to save components versions ------------------------------------------------------------------------- 3.37s
utils : Create cpd instance namespace if not present cpd-instance --------------------------------------------------------------------------- 3.37s
utils : verify if the CRD is present discovery.watson.ibm.com v1 WatsonDiscovery ------------------------------------------------------------ 1.28s
utils : applying CR wd for Watson Discovery ------------------------------------------------------------------------------------------------- 1.16s
utils : fetch the WatsonDiscovery CR if it exists ------------------------------------------------------------------------------------------- 1.11s
utils : create operand request for instance namespace if not present ------------------------------------------------------------------------ 1.10s
utils : Pause for "1" seconds to let OLM trigger changes (to avoid getting confused by existing state) -------------------------------------- 1.06s
utils : verify if the package ibm-watson-discovery-operator  is available for use in the cpd-instance namespace ----------------------------- 1.05s
utils : Print trace information ------------------------------------------------------------------------------------------------------------- 0.74s
utils : checking ocp cluster connection status ---------------------------------------------------------------------------------------------- 0.62s
utils : get cluster arch -------------------------------------------------------------------------------------------------------------------- 0.48s
utils : pre- apply-cr release patching (if any) for watson_discovery ------------------------------------------------------------------------ 0.22s
utils : post- apply-cr release patching (if any) for watson_discovery ----------------------------------------------------------------------- 0.21s
utils : include_vars ------------------------------------------------------------------------------------------------------------------------ 0.13s
utils : remove any existing generated preview yaml file ------------------------------------------------------------------------------------- 0.11s
utils : merging release_components_meta and global_components_meta -------------------------------------------------------------------------- 0.08s
utils : merge with override_components_meta ------------------------------------------------------------------------------------------------- 0.08s
utils : fail -------------------------------------------------------------------------------------------------------------------------------- 0.07s
[SUCCESS] 2022-09-19T13:04:04.677287Z You may find output and logs in the "/home/fr054721/cpd-cli/cpd-cli-workspace/olm-utils-workspace/work" directory.
[SUCCESS] 2022-09-19T13:04:04.679107Z The apply-cr command ran successfully.

```



### Installing Watson Knowledge Studio OLM

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli
./cpd-cli manage login-to-ocp --username=${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL}

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true

COMPONENTS="watson_ks"

./cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--components=${COMPONENTS}

oc get catalogsource -n ${PROJECT_CATSRC} 
oc get po -n ${PROJECT_CATSRC} 
oc get sub -n ${PROJECT_CPFS_OPS} 

SUB="ibm-watson-ks-operator-subscription"

SUB=$(oc get sub -n ${PROJECT_CPFS_OPS} ${SUB} -o jsonpath='{.status.installedCSV} {"\n"}') && echo $SUB

oc get csv -n ${PROJECT_CPFS_OPS} $SUB -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'
> Succeeded : install strategy completed with no errors

oc get deployments -n ${PROJECT_CPFS_OPS} -l olm.owner="$SUB" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"
> 1
```



### Installing Watson Knowledge Studio Service

```
source ~/cpd-cli/cpd-vars.sh && cd ~/cpd-cli

./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL} --insecure-skip-tls-verify=true

tee << EOF | oc apply -f -
apiVersion: knowledgestudio.watson.ibm.com/v1
kind: KnowledgeStudio
metadata:
  name: wks
  namespace: ${PROJECT_CPD_INSTANCE}
spec:
  license:
    accept: true
  global:
    storageClassName: ${STG_CLASS_BLOCK}
    size: small
EOF

oc get crd | grep knowledgestudios
> knowledgestudios.knowledgestudio.watson.ibm.com

oc get knowledgestudios.knowledgestudio.watson.ibm.com -o json | jq -r '.items[].kind'
> KnowledgeStudio

oc get KnowledgeStudio
> NAME   DEPLOYED
wks    

oc get KnowledgeStudio wks -o json | jq -r '.status.conditions[]'
> {
  "lastTransitionTime": "2022-09-19T11:27:13Z",
  "status": "True",
  "type": "Initialized"
}
{
  "lastTransitionTime": "2022-09-19T11:33:04Z",
  "message": "1. You can run deployment verification test with following command.\n\n  helm test wks --cleanup --timeout 600 --tls\n\nAfter the sucessful installation, a WKS add-on tile with the release name is shown up on your CP4D console. You can provision a WKS instance and launch your WKS tooling application there.\n\n2. Open your web browser and login to CP4D console.\n\n3. Move to Add-on catalog. You can find the add-on of IBM Watson Knowledge Studio with the release name.\n\n4. Click the Watson Knowledge Studio add-on tile and provision an instance.\n\n5. Open the created instance and click \"Launch Tool\" button.\n\n6. You can start using IBM Watson Knowledge Studio.\n",
  "reason": "UpgradeSuccessful",
  "status": "True",
  "type": "Deployed"
}
  
oc get pods -l 'release in (wks,wks-minio,wks-ibm-watson-ks)'



```







```
APIKEY_NAME="ibmcloud-key"

ibmcloud login --apikey @~/$APIKEY_NAME --no-region

CLUSTER_ID=$(ibmcloud oc clusters -output json | jq -r .[].id) && echo ${CLUSTER_ID}

ibmcloud oc worker ls -c ${CLUSTER_ID}
NODES=$(ibmcloud oc worker ls -c ${CLUSTER_ID} -q | awk '{print $1}') && echo ${NODES}
for wid in $(echo ${NODES}); do ibmcloud oc worker replace -c ${CLUSTER_ID} -w $wid -f; done
```



```
apiVersion: v1
kind: Pod
metadata:
  name: "MYAPP"
  namespace: default
  labels:
    app: "MYAPP"
spec:
  containers:
  - name: MYAPP
    image: "debian-slim:latest"
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: MYAPP
          key: DB_HOST
    ports:
    - containerPort:  80
      name:  http
    volumeMounts:
    - name: localtime
      mountPath: /etc/localtime
  volumes:
    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
  restartPolicy: Always
```

```
NS="s5-ppd-dlg-aoacc-common"
IMG="icr.io/cpopen/cpfs/iam-policy-administration:3.11.0"
POD="test"

oc project ${NS}

cat << EOF | oc apply -f -
kind: Pod
apiVersion: v1
metadata:
  name: ${POD}
  namespace: ${NS}
spec:
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



```
docker create --name olm-utils-play \
--env CMD_PREFIX="cpd-cli manage" \
-v ${HOME}/cpd-cli/cpd-cli-workspace/work:/tmp/work \
localhost/olm-utils:latest

```

