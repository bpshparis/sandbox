- [CPD 4.0 Install Guide Introduction](#cpd-40-install-guide-introduction)
- [Acquire or Provision an OCP 4.6.x Cluster](#acquire-or-provision-an-ocp-46x-cluster)
- [Find out your supported storage on the OCP cluster and Choose the one to use](#find-out-your-supported-storage-on-the-ocp-cluster-and-choose-the-one-to-use)
- [Set up a private registry and access to container image repositories and mirroring](#set-up-a-private-registry-and-access-to-container-image-repositories-and-mirroring)
  - [Install Client Tools](#install-client-tools)
  - [Set up CASE download location and IBM case repository](#set-up-case-download-location-and-ibm-case-repository)
  - [Set up a private image registry on a Bastion host](#set-up-a-private-image-registry-on-a-bastion-host)
  - [Export private registry's ENV variables and verify it works](#export-private-registrys-env-variables-and-verify-it-works)
  - [Obtain your IBM Entitlement API Key and verify login to cp.icr.io](#obtain-your-ibm-entitlement-api-key-and-verify-login-to-cpicrio)
  - [Configuring the global image pull secret to your private registry](#configuring-the-global-image-pull-secret-to-your-private-registry)
  - [Download ibm-cp-datacore CASE package and Configure credentials for mirroring images](#download-ibm-cp-datacore-case-package-and-configure-credentials-for-mirroring-images)
  - [Configure an image content source policy](#configure-an-image-content-source-policy)
- [Cluster and Node and OS and OCP tuning and updates](#cluster-and-node-and-os-and-ocp-tuning-and-updates)-
  - [Apply required setting changes on worker nodes](#apply-required-setting-changes-on-worker-nodes)
    - [Configure HA Proxy Timeout Settings](#configure-HA-Proxy-Timeout-Settings)
    - [Configure Cri-o Container Settings on Worker Nodes using MCO](#configure-cri-o-container-settings-on-worker-nodes-using-mco)
    - [Configure Kernel Settings on Worker Nodes utilizing Openshift Node Tuning Operator](#configure-kernel-settings-on-worker-nodes-utilizing-openshift-node-tuning-operator)
    - [Configure kubelet to allow Db2U to make SysCtl calls](#configure-kubelet-to-allow-db2u-to-make-sysctl-calls)
  - [Creating custom security context constraints for services](#creating-custom-security-context-constraints-for-services)
- [Create two namespaces for express install](#create-two-namespaces-for-express-install)
- [Create an OperatorGroup in ibm-common-services namespace](#create-an-operatorgroup-in-ibm-common-services-namespace)
- [Downloading platform and services CASE packages and Mirroring](#downloading-platform-and-services-case-packages-and-mirroring)
- [Add Service Catalog Sources](#add-service-catalog-sources)
- [Installing IBM Cloud Pak foundational services - Bedrock](#installing-ibm-cloud-pak-foundational-services---bedrock)
- [Creating an operator subscription for the scheduling service](#creating-an-operator-subscription-for-the-scheduling-service) 
- [Installing the Scheduling Service](#installing-the-scheduling-service)
- [Creating a subscription for the IBM Cloud Pak for Data platform operator](#creating-a-subscription-for-the-ibm-cloud-pak-for-data-platform-operator)
- [Installing Cloud Pak for Data Operator-ZenService-Lite](#installing-cloud-pak-for-data-operator-zenService-lite)
- [Deploy Watson Knowledge Catalog](#deploy-watson-knowledge-catalog)
- [General Flow on Installing Individual CPD Services](#general-flow-on-installing-individual-cpd-services)
- [About the CASE Download Directory](#about-the-case-download-directory)  (__IMPORTANT NOTICE TO READ__)
  - [Deploy Watson Studio Service](#deploy-watson-studio-service)
  - [Deploy Watson Machine Learning Service](#deploy-watson-machine-learning-service)
  - [Deploy Watson OpenScale Service](#deploy-watson-openscale-service)
  - [Deploy RStudio Server with R 3.6](#deploy-rstudio-server-with-r-36)
  - [Set up WSL Runtimes](#set-up-wsl-runtimes)
  - [Deploy IBM Match 360 with Watson Service](#deploy-ibm-match-360-with-watson-service)
  - [Deploy Db2 Data Management Console](#deploy-db2-data-management-console)
  - [Deploy OpenPages](#deploy-openPages)
  - [Deploy Watson Machine Learning Accelerator](#deploy-watson-machine-learning-accelerator)
  - [Deploy Data Virtualization](#deploy-data-virtualization)
  - [Deploy Db2 OLTP](#deploy-db2-oltp)
  - [Deploy Jupyter Notebooks with R 3.6](#deploy-jupyter-notebooks-with-r-36)
  - [Deploy SPSS Modeler](#deploy-spss-modeler)
  - [Deploy Db2 Warehouse Service](#deploy-db2-warehouse-service) 
  - [Deploy Decision Optimization](#deploy-decision-optimization)
  - [Deploy Db2 Big SQL](#deploy-db2-big-sql)
  - [Jupyter Notebooks with Python 3.7 for GPU](#jupyter-notebooks-with-python-37-for-gpu)
  - [Deploy Db2 Data Gate Service](#deploy-db2-data-gate-service)
  - [Deploy Analytics Engine Powered by Apache Spark](#deploy-analytics-engine-powered-by-apache-spark)
  - [Deploy Data Stage Service](#deploy-data-stage-service)
  - [Deploy Execution Engine for Apache Hadoop](#deploy-execution-engine-for-apache-hadoop)
---
### [CPD 4.0 Install Guide Introduction](#cpd-40-install-guide-introduction)

This document describes a step-by-step procedure to install Bedrock, CPD Platform Operator, Zen Operator, Zenservice/Lite, WKC (including CCS, Data Refinery) and other services based on the official CPD 4.0 documentation at: 

   https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=installing  
   https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=integrations-services

This doc performs the CPD 4.0 installation using the __Express Installation__ mode in the air-gapped environment with a private registry on a bastion host.

---
### [Acquire or Provision an OCP 4.6.x Cluster](#acquire-or-provision-an-ocp-46x-cluster)

You can provision an OCP 4.6.x cluster in Fyre or other cloud platform you have access. Preferably upgrade to the newest minor version of OCP 4.6.x.

Preferably upgrade OCP 4.6.x to the latest release. For example, our OCP cluster versions are as below:

  ```
  Client Version: 4.6.31
  Server Version: 4.6.31
  Kubernetes Version: v1.19.0+d670f74
  ```

---
### [Find out your supported storage on the OCP cluster and Choose the one to use](#find-out-your-supported-storage-on-the-ocp-cluster-and-choose-the-one-to-use)


  Your OCP cluster can have the following storage support:

  ```
  Portworx
  OpenShift Container Storage
  NFS
  IBM Cloud File Storage
  ```

  You need to find out what are available and what you want to use. Here we'll choose the NFS, and the NFS storage class is: `managed-nfs-storage`

  ```
  [root@api.souled.cp.fyre.ibm.com cpd4sre]# oc get sc
  NAME                  PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
  managed-nfs-storage   fuseim.pri/ifs   Delete          Immediate           false                  8m30s
  ```

  Next export your storage type and storage class name you choose to use by:

  ```
  export STORAGE_TYPE=nfs                             # Specify the type of storage to use, such as ocs , nfs or portworx values nfs, ocs, portworx
  export STORAGE_CLASS_NAME=managed-nfs-storage       # Replace with the name of a RWX storage class
  ```

  Note that: __in the rest of the document, if the code snippet contains the property for storage class, please double check with the official documentation to see if any storage type/vendor needs to be specified or not, add the proper storage type value if needed. In this document, we use the NFS storage class so it usually does not need to specify the storage vendor.__

---
## [Set up a private registry and access to container image repositories and mirroring](#set-up-a-private-registry-and-access-to-container-image-repositories-and-mirroring)


---
### [Install Client Tools](#install-client-tools)

  - Install the following tools

  ```
  sudo yum install -y httpd-tools podman ca-certificates openssl skopeo jq bind-utils git
  ```

  - Install the `cloudctl` cli

  ```
  wget https://github.com/IBM/cloud-pak-cli/releases/download/v3.15.0/cloudctl-linux-amd64.tar.gz
  tar -xf cloudctl-linux-amd64.tar.gz
  sudo cp cloudctl-linux-amd64 /usr/bin/cloudctl
  cloudctl version
  ```

  - Install Python2

  ```
  yum install -y python2 python2-pip
  alternatives --set python /usr/bin/python2
  pip2 install pyyaml
  
  /*
  Collecting pyyaml
    Using cached https://files.pythonhosted.org/packages/36/2b/61d51a2c4f25ef062ae3f74576b01638bebad5e045f747ff12643df63844/PyYAML-6.0.tar.gz
      Complete output from command python setup.py egg_info:
      Traceback (most recent call last):
        File "<string>", line 1, in <module>
        File "/tmp/pip-build-CuvUqx/pyyaml/setup.py", line 67, in <module>
          import sys, os, os.path, pathlib, platform, shutil, tempfile, warnings
      ImportError: No module named pathlib
      
      ----------------------------------------
  Command "python setup.py egg_info" failed with error code 1 in /tmp/pip-build-CuvUqx/pyyaml/
  You are using pip version 8.1.2, however version 21.3.1 is available.
  You should consider upgrading via the 'pip install --upgrade pip' command.
  */
  
  python3 -m pip install pyyaml
  python3 -m pip install --upgrade pip
  
  sudo yum install yamllint -y
  sudo find /usr/bin -type f -name python*
  
  sudo update-alternatives --install /usr/bin/python python /usr/bin/python2 1
  sudo update-alternatives --install /usr/bin/python python /usr/bin/python3 3
  sudo update-alternatives --config python
  update-alternatives --list 
  python -V
  
  python -m pip list
  
  /* !!!! Toggle between python2 for yum to work and python3 CP4D install catalog !!!
  
  # Install CentOS SCLo RH repository:
  yum install centos-release-scl-rh
  # Install python27-PyYAML rpm package:
  yum install python27-PyYAML
  
  yum install libyaml-devel
  
  ```

---
### [Set up CASE download location and IBM case repository](#set-up-case-download-location-and-ibm-case-repository)

  Create a local offline directory to store downloaded CASE packages and mirrored images and export their environment variables by:

  ```
  mkdir -p $HOME/offline/cpd
  mkdir -p $HOME/offline/registry
  export OFFLINEDIR=$HOME/offline/cpd
  export OFFLINE_REGISTRYDIR=$HOME/offline/registry
  export CASE_REPO_PATH=https://github.com/IBM/cloud-pak/raw/master/repo/case
  ```

---
### [Set up a private image registry on a Bastion host](#set-up-a-private-image-registry-on-a-bastion-host)


  In the official documentation at https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=registry-mirroring-images-intermediary-container, there is a pretty simple elegant method to set up a private or portable registry. Unfortunately it contains a bug on generating the self-signed certificate, and render it unusable for our purpose.  Below is just one method we choose to set it up. There are many dozens of methods you could find in the Internet to set up a private registry.

  - create directories for the private registry under the offline directory

  ```
  mkdir -p $OFFLINE_REGISTRYDIR/{auth,certs,data}
  ```

  - configure a registry username and password that will be used to authenticate to the registry (change the username and password as you like)

  ```
  htpasswd -bBc $OFFLINE_REGISTRYDIR/auth/htpasswd dockeruser dockeruser
  ```

  - generate a self-signed TLS certificate for the private registry

  ```
  openssl req -newkey rsa:4096 -nodes -sha256 -keyout $OFFLINE_REGISTRYDIR/certs/domain.key -x509 -days 7300 -out $OFFLINE_REGISTRYDIR/certs/domain.crt -subj "/C=FR/L=Bois-Colombes/O=IIC/OU=IIC Paris/CN=$(hostname -f)/emailAddress=iic_paris@fr.ibm.com"
  
  openssl x509 -noout -text -in $OFFLINE_REGISTRYDIR/certs/domain.crt
  ```

  - make OCP trust this self-signed certificate


  ```
  sudo cp -v $OFFLINE_REGISTRYDIR/certs/domain.crt /etc/pki/ca-trust/source/anchors/
  sudo update-ca-trust
  trust list | grep -i $(hostname -f)
  
  ```

  - start your private registry by

  ```
sudo podman run --privileged --name cpd40registry \
-p 5000:5000 \
-v $OFFLINE_REGISTRYDIR/data:/var/lib/registry:z \
-v $OFFLINE_REGISTRYDIR/auth:/auth:z \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
-v $OFFLINE_REGISTRYDIR/certs:/certs:z \
-e "REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt" \
-e "REGISTRY_HTTP_TLS_KEY=/certs/domain.key" \
-d docker.io/library/registry:2

sudo podman inspect cpd40registry | jq -r '.[].State.Status'
  ```

  - add the private registry to insecure registries list – The Machine Config Operator (MCO) will push updates to all nodes in the cluster and reboot them. __Remember to replace your registry hostname in the command below.__

  ```
  oc patch image.config.openshift.io/cluster --type=merge -p '{"spec":{"registrySources":{"insecureRegistries":["'$(hostname -f)':5000"]}}}' --dry-run=client
  
  oc patch image.config.openshift.io/cluster --type=merge -p '{"spec":{"registrySources":{"insecureRegistries":["'$(hostname -f)':5000"]}}}'
  ```

  And monitor the MCO update push and reboot by

  ```
  watch -n5 'oc get mcp -o wide; echo; oc get node -o "custom-columns=NAME:metadata.name,STATE:metadata.annotations.machineconfiguration\\.openshift\\.io/state,DESIRED:metadata.annotations.machineconfiguration\\.openshift\\.io/desiredConfig,CURRENT:metadata.annotations.machineconfiguration\\.openshift\\.io/currentConfig,REASON:metadata.annotations.machineconfiguration\\.openshift\\.io/reason"'
  ```

---
### [Export private registry's ENV variables and verify it works](#export-private-registrys-env-variables-and-verify-it-works)

  - Export the the following three environment variables for the private registry that you used to create the registry. For example:
    Replace with your chosen username and password and registry URL here.

  ```
  export PRIVATE_REGISTRY_USER="dockeruser"
  export PRIVATE_REGISTRY_PASSWORD="dockeruser"
  export PRIVATE_REGISTRY="$(hostname -f):5000"
  export USE_SKOPEO="true"
  ```

  where the "USE_SKOPEO" environment variable is very important for copying images successfully during the image mirroring step.

  - Verify pushing images to the private registry works

  ```
  sudo podman pull busybox
  sudo podman tag docker.io/library/busybox $PRIVATE_REGISTRY/zen/busybox
  sudo podman images
  sudo podman login --username $PRIVATE_REGISTRY_USER --password $PRIVATE_REGISTRY_PASSWORD $PRIVATE_REGISTRY --tls-verify=false
  sudo podman push $PRIVATE_REGISTRY/zen/busybox --tls-verify=false
  ```

  - List all images in the private registry

  ```
  curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY}/v2/_catalog?n=6000 |  python -m json.tool
  
  curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY}/v2/_catalog?n=6000 | jq .
  ```

  - List all tags for an image

  ```
  IMG="zen/busybox"
  
  curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY}/v2/${IMG}/tags/list |  python -m json.tool
  
  curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY}/v2/${IMG}/tags/list |  jq .
  ```



---
### [Obtain your IBM Entitlement API Key and verify login to cp.icr.io](#obtain-your-ibm-entitlement-api-key-and-verify-login-to-cpicrio)

  - Get your IBM entitlement API key
  
    1. Log in to Container software library on My IBM (https://myibm.ibm.com/products-services/containerlibrary) with the IBM ID and password that are associated with the entitled software.
    2. On the Get entitlement key tab, select Copy key to copy the entitlement key to the clipboard.
    3. Save the API key in a text file.
    
  - Export the three environment variables

  ```
  export REGISTRY_USER="cp"
  export REGISTRY_PASSWORD="$(cat ${HOME}/apikey)" && echo $REGISTRY_PASSWORD
  export REGISTRY_SERVER="cp.icr.io"
  
  ```

  - Test login to cp.icr.io

  ```
  sudo podman login -u ${REGISTRY_USER} -p ${REGISTRY_PASSWORD} ${REGISTRY_SERVER}
  ```

---
### [Download ibm-cp-datacore CASE package and Configure credentials for mirroring images](#download-ibm-cp-datacore-case-package-and-configure-credentials-for-mirroring-images)


  - download the CPD platform CASE package (make sure that you have ENV variables defined as from the previous steps)

  ```
  CASE="ibm-cp-datacore-2.0.11.tgz"
  OFFLINEDIR="$HOME/offline/cpd"
  
  cloudctl case save \
    --case ${CASE_REPO_PATH}/${CASE} \
    --outputdir ${OFFLINEDIR} \
    --no-dependency
  ```

  - configure your cp.icr.io credential in the local bastion host

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/${CASE} \
    --inventory cpdPlatformOperator \
    --action configure-creds-airgap \
    --args "--registry ${REGISTRY_SERVER} --user ${REGISTRY_USER} --pass ${REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

:bulb: [INFO] Registry secret created in /home/outscale/.airgap/secrets/cp.icr.io.json

  - configure private container registry credentials in the local bastion host

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/${CASE} \
    --inventory cpdPlatformOperator \
    --action configure-creds-airgap \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD}"
  ```

:bulb: [INFO] Registry secret created in /home/outscale/.airgap/secrets/lb.ocp.local:5000.json

---
### [Configuring the global image pull secret to your private registry](#configuring-the-global-image-pull-secret-to-your-private-registry)


  This air-gapped CPD 4.0.x deployment is using the private registry. So the GLOBAL pull secret should be configured TO the PRIVATE_REGISTRY by following the steps:

  1. review all the entries in the current global pull-secret by
  
       ```
       oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .
       ```
       
  2. back up the existing global pull secret: `oc get secret pull-secret -n openshift-config -o yaml > <YOUR-BACKUP-FILE-NAME>`
  
  3. if the output in step 1 does not contain an entry for $PRIVATE_REGISTRY, the simplest method is to copy the following code into a script as `util-add-pull-secret.sh`, and run it:

```
#!/bin/bash

if [ "$#" -lt 3 ]; then
    echo "Usage: $0 <repo-url> <artifactory-user> <API-key>" >&2
    exit 1
fi
    
# set -x
     
REPO_URL=$1
REPO_USER=$2
REPO_API_KEY=$3
     
pull_secret=$(echo -n "$REPO_USER:$REPO_API_KEY" | base64 -w0)

oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | sed -e 's|:{|:{"'$REPO_URL'":{"auth":"'$pull_secret'","email":"not-used"\},|' > /tmp/dockerconfig.json

oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/dockerconfig.json
```
then run as

```
chmod +x util-add-pull-secret.sh
./util-add-pull-secret.sh $PRIVATE_REGISTRY $PRIVATE_REGISTRY_USER $PRIVATE_REGISTRY_PASSWORD
```

if you perform this step successfully, then you can skip the step 4 - 7, just monitor the Machine Config Operator (MCO) push updates to all nodes in the cluster and reboot them as noted after step 7.

To manually update the pull-secret, follow the steps 4 -7 below:

  4. extract the current pull secrets: `oc extract secret/pull-secret -n openshift-config --confirm` This will result in a file named as `.dockerconfigjson`
  5. create and encode your regsirty username and password to obtain your authorization token by: `echo -n "$PRIVATE_REGISTRY_USER:$PRIVATE_REGISTRY_PASSWORD" | base64 -w0; echo`
  6. edit the `.dockerconfgijson` file, and insert an entry like below 
  
      ```
      "<PRIVATE_REGISTRY_DNS_NAME>": {
        "auth": "<YOUR-AUTHORIZATION-TOKEN>",
        "email": "not-used"
      },
      ```
    
     just after the beginning string phrase `{"auths":{`
   
     If the entry for <PRIVATE_REGISTRY_DNS_NAME> already exists, you can edit it in-place.
   
  7. finally set the contents of file `.dockerconfigjson` back into the global pull-secret by:
  
     ```
     oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=.dockerconfigjson
     ```

  - To monitor MCO push the global secret updates to all nodes, run the command below:

  ```
  watch -n5 'oc get mcp -o wide; echo; oc get node -o "custom-columns=NAME:metadata.name,STATE:metadata.annotations.machineconfiguration\\.openshift\\.io/state,DESIRED:metadata.annotations.machineconfiguration\\.openshift\\.io/desiredConfig,CURRENT:metadata.annotations.machineconfiguration\\.openshift\\.io/currentConfig,REASON:metadata.annotations.machineconfiguration\\.openshift\\.io/reason"'
  ```

---
### [Configure an image content source policy](#configure-an-image-content-source-policy)


  If you mirrored images to a private container registry, you must tell your cluster where to find the software images, that is, where on the private registry to locate the CPD images. 

  - Apply the following command:

  ```
  cat <<EOF |oc apply -f -
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
  EOF
  ```

  - Check the ImageContentSourcePolicy object

  ```
  oc get imageContentSourcePolicy
  ```

  - Finally monitor the MCO to push the updates to all nodes and reboot them by

  ```
  watch -n5 'oc get mcp -o wide; echo; oc get node -o "custom-columns=NAME:metadata.name,STATE:metadata.annotations.machineconfiguration\\.openshift\\.io/state,DESIRED:metadata.annotations.machineconfiguration\\.openshift\\.io/desiredConfig,CURRENT:metadata.annotations.machineconfiguration\\.openshift\\.io/currentConfig,REASON:metadata.annotations.machineconfiguration\\.openshift\\.io/reason"'
  ```


---
## [Cluster and Node and OS and OCP tuning and updates](#cluster-and-node-and-os-and-ocp-tuning-and-updates)

---
### [Apply required setting changes on worker nodes](#apply-required-setting-changes-on-worker-nodes)


__We strongly recommend performing the tasks in section as early as possible due to updating worker node's settings using MachineConfig Operator
(MCO) requires rebooting those worker nodes one by one, and may need more than one pass.__  

For the official documentation, refer to this page https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-changing-required-node-settings 
to update various CRI-O, OS kernel, and GPU settings.

- ### [Configure HA Proxy Timeout Settings](#configure-HA-Proxy-Timeout-Settings)

  To configure HA Proxy Timeout Settings on Load Balancer Node that is usually the infra node in Fyre

  - check LB haproxy's timeouts for client and server. If they are already higher than 5m, then no need to run the commands in the next section.
  
  ```
  cat /etc/haproxy/haproxy.cfg | grep timeout
        timeout http-request    10s
        timeout queue           1m
        timeout connect         10s
        timeout client          30m
        timeout server          30m
        timeout http-keep-alive 10s
        timeout check           10s
  ```
  
  - if lower than 5m, then run the commands below:
  
  ```
  sudo sed -i -e "/timeout server/s/ [0-9].*/ 5m/" /etc/haproxy/haproxy.cfg
  sudo sed -i -e "/timeout client/s/ [0-9].*/ 5m/" /etc/haproxy/haproxy.cfg
  sudo systemctl restart haproxy && sudo systemctl status haproxy
  ```
  
- ### [Configure Cri-o Container Settings on Worker Nodes using MCO](#configure-cri-o-container-settings-on-worker-nodes-using-mco)

  To change CRI-O settings, you modify the contents of the crio.conf file and pass those updates to your
nodes as a machine config.

1. Obtain a copy of the existing crio.conf file from a node. For example, run the following command:

  ```
  OCP="ocp"
  DOMAIN="$(cat /etc/resolv.conf | awk '/^search/ {print $2}')" && echo $DOMAIN
  
  scp -o StrictHostKeyChecking=no core@w1.${OCP}.${DOMAIN}:/etc/crio/crio.conf /tmp
  ```
  where replacing $node with one of those worker nodes.

2. In the crio.conf file, make the following changes in the [crio.runtime] section (uncomment the lines if needed):

   - To set the maximum number of open files, change the default_ulimits setting to at least 66560 as below:
   ```
   ......
   [crio.runtime]
   default_ulimits = [
           "nofile=66536:66536"
   ]
   ......
   ```
   
   - To set the maximum number of processes, change the pids_limit setting to at least 12288.
   
    ```
    ......
    # Maximum number of processes allowed in a container.
    pids_limit = 12288
    ......
    ```
   
3. Create a machineconfig object yaml file and apply it.

```
cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-cp4d-crio-conf
spec:
  config:
    ignition:
      version: 3.1.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,$(cat /tmp/crio.conf | base64 -w0)
        filesystem: root
        mode: 0644
        path: /etc/crio/crio.conf
EOF
```

4. Monitor MachineConfig Operator (MCO) pushes the updates to all the nodes and reboot them successfully:

  Note that: __the above configuration changes will cause the all nodes to be updated/re-synced by MCO during which nodes will go through the `NotReady/Ready,SchedulingDisabled` state one by one (one master and one worker be in parallel). To monitor the entire process, run the command below, and wait until all nodes are back to Ready state before continuing. There are several occasions below that need the same re-sync process.__

```
watch -n5 'oc get mcp -o wide; echo; oc get node -o "custom-columns=NAME:metadata.name,STATE:metadata.annotations.machineconfiguration\\.openshift\\.io/state,DESIRED:metadata.annotations.machineconfiguration\\.openshift\\.io/desiredConfig,CURRENT:metadata.annotations.machineconfiguration\\.openshift\\.io/currentConfig,REASON:metadata.annotations.machineconfiguration\\.openshift\\.io/reason"'
```

- ### [Configure Kernel Settings on Worker Nodes utilizing Openshift Node Tuning Operator](#configure-kernel-settings-on-worker-nodes-utilizing-openshift-node-tuning-operator)

  - Db2 Settings for Worker Nodes with typical 64 GB memory
  
  ```
  kernel.shmall: 33554432
  kernel.shmmax: 68719476736
  kernel.shmmni = 32768
  kernel.sem = 250 1024000 100 32768
  kernel.msgmax = 65536
  kernel.msgmnb = 65536
  kernel.msgmni = 32768
  ```
  
  - Elastic Search Settings
  
  ```
  vm.max_map_count: 262144
  ```
  
  - Copy and run the code below to apply the kernel setting changes
  
  ```
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
        kernel.shmmni = 32768
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
  ```

  - to verify the parameters are updated on all the nodes, run:
  
  ```
  for node in $(oc get nodes | awk '{ print $1 }' | grep -Ev NAME); do   echo --- from node $node; ssh core@$node -- sudo sysctl -a | grep 'kernel.shmall\|kernel.shmmax\|kernel.shmmni\|kernel.sem\|kernel.msgmax\|kernel.msgmax\|kernel.msgmni\|vm.max_map_count'; done
  
  KERNEL_PARMS="kernel.shmall|kernel.shmmax|kernel.shmmni|kernel.sem|kernel.msgmax|kernel.msgmnb|kernel.msgmni|vm.max_map_count"
  
  for node in $(oc get nodes | awk '{ print $1 }' | grep -Ev NAME); do ssh -o StrictHostKeyChecking=no core@$node 'hostname -f; sudo sysctl -a | egrep -w "'$KERNEL_PARMS'"'; done
  
  ```
  
- ### [Configure kubelet to allow Db2U to make SysCtl calls](#configure-kubelet-to-allow-db2u-to-make-sysctl-calls)

  - Update the label on the machineconfigpool worker:
  
  ```
  oc label machineconfigpool worker db2u-kubelet=sysctl
  ```
  
  - Update all of the nodes to use a custom KubletConfig by
  
  ```
  cat << EOF | oc apply -f -
  apiVersion: machineconfiguration.openshift.io/v1
  kind: KubeletConfig
  metadata:
    name: db2u-kubelet
  spec:
    machineConfigPoolSelector:
      matchLabels:
        db2u-kubelet: sysctl
    kubeletConfig:
      allowedUnsafeSysctls:
        - "kernel.msg*"
        - "kernel.shm*"
        - "kernel.sem"
  EOF
  ```

  - Wait for the cluster nodes to be sync-ed up with the updates. Run the following command to verify that the machineconfigpool worker is updated:
  
  ```
  watch -n5 'oc get mcp -o wide; echo; oc get node -o "custom-columns=NAME:metadata.name,STATE:metadata.annotations.machineconfiguration\\.openshift\\.io/state,DESIRED:metadata.annotations.machineconfiguration\\.openshift\\.io/desiredConfig,CURRENT:metadata.annotations.machineconfiguration\\.openshift\\.io/currentConfig,REASON:metadata.annotations.machineconfiguration\\.openshift\\.io/reason"'
  ```

---
### [Creating custom security context constraints for services](#creating-custom-security-context-constraints-for-services)

For details, refer to this page https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-creating-custom-sccs
to set up any required security context constraints for the service you try to install. The summary is listed below.

- __Db2__ requires the use of custom security context constraints (SCCs), which are created automatically when you install Db2. 
- __Db2 Warehouse, Db2 Big SQL, Data Virtualization__, and __OpenPages__ use the same SCC capabilities as in Db2.
- __Watson Knowledge Catalog__ requires the use of a custom security context constraint (SCC). Follow the following procedure:

1. Define the SCC in the file wkc-iis-scc.yaml, as follows:

  ```
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities: null
apiVersion: security.openshift.io/v1
defaultAddCapabilities: null
fsGroup:
  type: RunAsAny
kind: SecurityContextConstraints
metadata:
  annotations:
    kubernetes.io/description: WKC/IIS provides all features of the restricted SCC
      but runs as user 10032.
  name: wkc-iis-scc
readOnlyRootFilesystem: false
requiredDropCapabilities:

- KILL
- MKNOD
- SETUID
- SETGID
runAsUser:
  type: MustRunAs
  uid: 10032
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
users:
- system:serviceaccount:cpd-instance:wkc-iis-sa
  ```

:warning: Update namespace name for system:serviceaccount

e.g. system:serviceaccount:**cpd-instance**:wkc-iis-sa



  2. Run oc create to create the custom SCC:

    oc create -f wkc-iis-scc.yaml

  3. To check it, run:

    oc get scc wkc-iis-scc

  4. Create the SCC cluster role for wkc-iis-scc:

```
oc create clusterrole system:openshift:scc:wkc-iis-scc --verb=use --resource=scc --resource-name=wkc-iis-scc
```

  5. Assign the wkc-iis-sa service account to the SCC cluster role:

```
oc create rolebinding wkc-iis-scc-rb --clusterrole=system:openshift:scc:wkc-iis-scc --serviceaccount=cpd-instance:wkc-iis-sa
```

  6. Confirm that the wkc-iis-sa service account can use the wkc-iis-scc SCC:

```
oc adm policy who-can use scc wkc-iis-scc | grep "wkc-iis-sa"
```




---

### Shutdown and snapshot

```
for node in $(oc get nodes -o jsonpath='{.items[*].metadata.name}'); do oc debug node/${node} -- chroot /host shutdown -h 1; done
```




---
### [Create two namespaces for express install](#create-two-namespaces-for-express-install)


First create the three OCP projects/namespaces required for the CPD 4.0 Specialized Installation Mode.

  ```
  oc new-project ibm-common-services
  oc new-project cpd-instance
  ```

---
### [Create an OperatorGroup in ibm-common-services namespace](#create-an-operatorgroup-in-ibm-common-services-namespace)


Copy the following code into a shell script and run it to accomplish the task:

```
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  name: operatorgroup
  namespace: ibm-common-services
spec:
  targetNamespaces:
  - ibm-common-services
EOF
```

---
### [Downloading platform and services CASE packages and Mirroring](#downloading-platform-and-services-case-packages-and-mirroring)


  - Download the required CASE package archive files from https://github.com/IBM/cloud-pak/raw/master/repo/case. A Container Application Software for Enterprises (CASE) package is an archive file that describes a containerized component of Cloud Pak for Data.

```
# mirroring platform operator package that has been downloaded in a previous step

CASE="ibm-cp-datacore-2.0.11.tgz"

cloudctl case save \
  --case ${CASE_REPO_PATH}/${CASE} \
  --outputdir ${OFFLINEDIR} \
  --no-dependency

cloudctl case launch \
  --case ${OFFLINEDIR}/${CASE} \
  --inventory cpdPlatformOperator \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"

# ibm common services, i.e. IBM Cloud Pak foundational services

CASE="ibm-cp-common-services-1.11.1.tgz"

cloudctl case save \
--case ${CASE_REPO_PATH}/${CASE} \
--outputdir ${OFFLINEDIR}

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-common-services-1.4.1.tgz \
  --inventory ibmCommonServiceOperatorSetup \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY}_USER --pass ${PRIVATE_REGISTRY}_PASSWORD --inputDir ${OFFLINEDIR}"

# scheduling service
cloudctl case save \
  --case ${CASE_REPO_PATH}/ibm-cpd-scheduling-1.2.1.tgz \
  --outputdir ${OFFLINEDIR}

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cpd-scheduling-1.2.1.tgz \
  --inventory schedulerSetup \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"

# DB2U

CASE="ibm-db2uoperator-4.0.8.tgz" && echo $CASE
export OFFLINEDIR=$HOME/offline/db2u && echo $OFFLINEDIR
[ ! -d "$OFFLINEDIR" ] && mkdir $OFFLINEDIR

cloudctl case save \
--case ${CASE_REPO_PATH}/${CASE} \
--outputdir ${OFFLINEDIR}

cloudctl case launch \
--case ${OFFLINEDIR}/ibm-db2uoperator-4.0.8.tgz \
--inventory db2uOperatorSetup \
--action mirror-images \
--args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY}_USER --pass ${PRIVATE_REGISTRY}_PASSWORD --inputDir ${OFFLINEDIR} --arch amd64"

# Watson Knowledge Catalog
cloudctl case save \
--case ${CASE_REPO_PATH}/ibm-wkc-4.0.0.tgz \
--outputdir ${OFFLINEDIR}

cloudctl case launch \
--case ${OFFLINEDIR}/ibm-wkc-4.0.0.tgz \
--inventory wkcOperatorSetup \
--action mirror-images \
--args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY}_USER --pass ${PRIVATE_REGISTRY}_PASSWORD --inputDir ${OFFLINEDIR}"
  
# WARNING !! ${HOME}/.airgap/auth.json does not exists merge ${HOME}/.airgap/secrets/* in it
  
tail -q -n +2 ${OFFLINEDIR}/*-images.csv | while IFS="," read registry image_name tag digest mtype os arch variant insecure digest_source; do
  if [ "$arch" != "ppc64le" ] && [ "$arch" != "s390x" ]; then
    echo "sudo skopeo copy --all --authfile ${HOME}/.airgap/auth.json --dest-tls-verify=false --src-tls-verify=false docker://${registry}/${image_name}@${digest} docker://${PRIVATE_REGISTRY}/${image_name}@${digest} ${arch}"
    sudo skopeo copy --all --authfile "${HOME}/.airgap/auth.json" --dest-tls-verify=false --src-tls-verify=false \
    docker://${registry}/${image_name}@${digest} docker://${PRIVATE_REGISTRY}/${image_name}@${digest}
  fi
done
```

  - verify the mirrored images by listing them from the private registry

  ```
  curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY}/v2/_catalog?n=6000 |  python -m json.tool
  
  curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} https://${PRIVATE_REGISTRY}/v2/_catalog?n=6000 | jq .
  ```

---
### [Add Service Catalog Sources](#add-service-catalog-sources)


  - After pushing the necessary images into the private registry via mirroring, add the service catalog sources by:

  ```
CASE="ibm-cp-datacore-2.0.11.tgz" && echo $CASE
OFFLINEDIR="$HOME/offline/cpd" && echo $OFFLINEDIR

cloudctl case launch \
  --case ${OFFLINEDIR}/${CASE} \
  --inventory cpdPlatformOperator \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  :bulb: No more necessary with ibm-cp-datacore-2.0.5.tgz 

update the catalog source image path adding /cpopen after the hostname and port

  ```
  oc edit catalogsource cpd-platform -n openshift-marketplace
  ```

  - Add more required catalog sources

  ```
CASE="ibm-cp-common-services-1.11.1.tgz"
OFFLINEDIR="$HOME/offline/cpd" && echo $OFFLINEDIR

cloudctl case launch \
  --case ${OFFLINEDIR}/${CASE} \
  --inventory ibmCommonServiceOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  
CASE="ibm-db2uoperator-4.0.8.tgz"
OFFLINEDIR="$HOME/offline/db2u" && echo $OFFLINEDIR  

cloudctl case launch \
--case ${OFFLINEDIR}/${CASE} \
--inventory db2uOperatorSetup \
--namespace openshift-marketplace \
--action install-catalog \
--args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"  
  ```

  - verify the successful service catalog creation by

```
oc get catalogsource -n openshift-marketplace cpd-platform -o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'

oc get catalogsource -n openshift-marketplace opencloud-operators -o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'

oc get catalogsource -n openshift-marketplace ibm-db2uoperator-catalog -o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'
```

where those commands should list READY for those catalog sources.

Or for general overview:

```
oc get catalogsource -n openshift-marketplace
oc get po -n openshift-marketplace
```

make sure all the pods are running correctly in the output before continuing. 

:bulb: Troubleshooting: for all pod to be up needed to pull 2 images

```
skopeo copy --all --authfile "${HOME}/.airgap/auth.json" --dest-tls-verify=false --src-tls-verify=false \
docker://cp.icr.io/cpopen/ibm-common-service-catalog:latest \
docker://cli.ocp12.iicparis.fr.ibm.com:5000/cpopen/ibm-common-service-catalog:latest

skopeo copy --all --authfile "${HOME}/.airgap/auth.json" --dest-tls-verify=false --src-tls-verify=false \     docker://cp.icr.io/cpopen/ibm-cpd-platform-operator-catalog@sha256:5550dbf568c0efa04e60efda893acf55be6ad06ebe1b128dce41f0eca5a59832 \
docker://b.ocp.local:5000/cpopen/ibm-cpd-platform-operator-catalog@sha256:5550dbf568c0efa04e60efda893acf55be6ad06ebe1b128dce41f0eca5a59832

sudo skopeo copy --all --authfile "${HOME}/.airgap/auth.json" --dest-tls-verify=false --src-tls-verify=false docker://cp.icr.io/cpopen/ibm-cpd-platform-operator-catalog@sha256:5550dbf568c0efa04e60efda893acf55be6ad06ebe1b128dce41f0eca5a59832 docker://lb.ocp.local:5000/cpopen/ibm-cpd-platform-operator-catalog@sha256:5550dbf568c0efa04e60efda893acf55be6ad06ebe1b128dce41f0eca5a59832

```






---
### [Installing IBM Cloud Pak foundational services - Bedrock](#installing-ibm-cloud-pak-foundational-services---bedrock)


  - To install IBM Cloud Pak foundational services by running the code below:


  ```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-common-service-operator
  namespace: ibm-common-services
spec:
  channel: v3
  installPlanApproval: Automatic
  name: ibm-common-service-operator
  source: opencloud-operators
  sourceNamespace: openshift-marketplace
EOF
  ```

  - You can verify the successful completion by copying and running the following script:


  ```
#!/bin/bash

set -x
oc --namespace ibm-common-services get csv
echo
oc get po -n ibm-common-services
echo
oc get crd | grep operandrequest
echo
oc api-resources --api-group operator.ibm.com
  ```

You should see something as below:

  ```
+ oc --namespace ibm-common-services get csv
NAME                                          DISPLAY                                VERSION   REPLACES                                      PHASE
ibm-common-service-operator.v3.8.1            IBM Cloud Pak foundational services    3.8.1     ibm-common-service-operator.v3.8.0            Succeeded
ibm-namespace-scope-operator.v1.2.0           IBM NamespaceScope Operator            1.2.0     ibm-namespace-scope-operator.v1.1.1           Succeeded
operand-deployment-lifecycle-manager.v1.6.0   Operand Deployment Lifecycle Manager   1.6.0     operand-deployment-lifecycle-manager.v1.5.0   Succeeded
+ echo

+ oc get po -n ibm-common-services
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
---
### [Creating a subscription for the IBM Cloud Pak for Data platform operator](#creating-a-subscription-for-the-ibm-cloud-pak-for-data-platform-operator)


  Create the CPD platform operator by running the code below:

  ```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cpd-operator
  namespace: ibm-common-services
spec:
  channel: v2.0
  installPlanApproval: Automatic
  name: cpd-platform-operator
  source: cpd-platform
  sourceNamespace: openshift-marketplace
EOF
  ```



Run the following command to confirm that the subscription was triggered:

```
oc get sub -n ibm-common-services cpd-operator -o jsonpath='{.status.installedCSV} {"\n"}'
```

:bulb: Wait around a minute and verify that the command returns `cpd-platform-operator.v2.0.6`.



Run the following command to confirm that the cluster service version (CSV) is ready:

```
oc get csv -n ibm-common-services cpd-platform-operator.v2.0.6 \
-o jsonpath='{ .status.phase } : { .status.message} {"\n"}'
```

:bulb: Verify that the command returns `Succeeded : install strategy completed with no errors`.



Run the following command to confirm that the operator is ready:

```
oc get deployments -n ibm-common-services -l olm.owner="cpd-platform-operator.v2.0.6" \
-o jsonpath="{.items[0].status.availableReplicas} {'\n'}"
```

:bulb: Verify that the command returns an integer greater than or equal to `1`. If the command returns `0`, wait for the deployment to become available.



### Enabling services to use namespace scoping with third-party operators

:bulb: More detail can be found [here](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-creating-operator-subscriptions#preinstall-operator-subscriptions__nss-csv)

This setting is required for the following services:

- Data Virtualization
- Db2 Data Management Console
- IBM Match 360 with Watson™
- Watson Assistant
- Watson Assistant for Voice Interaction
- Watson Discovery
- Watson Speech to Text
- Watson Text to Speech

Run the following command to update the `IBM NamespaceScope Operator` in the `ibm-common-services` project:

```
oc patch NamespaceScope common-service \
-n ibm-common-services \
--type=merge \
--patch='{"spec": {"csvInjector": {"enable": true} } }'
```



### [Installing Cloud Pak for Data Operator-ZenService-Lite](#installing-cloud-pak-for-data-operator-zenService-lite)


  - Create an operand request to grant permission to the IBM Cloud Pak for Data platform operator and the IBM Cloud Pak foundational services operator to manage the project where you plan to install Cloud Pak for Data

  ```
NS="cpd-instance"

cat << EOF | oc apply -f -
apiVersion: operator.ibm.com/v1alpha1
kind: OperandRequest
metadata:
  name: empty-request
  namespace: $NS
spec:
  requests: []
EOF
  ```

  - Install IBM Cloud Pak for Data, i.e. Zenservice/Lite by running the code below. It creates an **Ibmcpd Custom Resource** that will trigger the installation of CPD Lite, i.e. ZenService.

   ```
cat << EOF | oc apply -f -
apiVersion: cpd.ibm.com/v1
kind: Ibmcpd
metadata:
  name: ibmcpd-cr
  namespace: $NS
spec:
  license:
    accept: true
    license: Enterprise
  storageClass: ${STORAGE_CLASS_NAME}
EOF
   ```

  - Verify the CPD Lite/ZenService is installed successfully

```
  oc get Ibmcpd ibmcpd-cr -n $NS -o yaml
```

  At the end of the output, you should see:

  ```
  ......
  status:
  conditions:
  - ansibleResult:
      changed: 2
      completion: 2021-07-20T21:39:34.301023
      failures: 0
      ok: 22
      skipped: 5
    lastTransitionTime: "2021-07-19T19:18:46Z"
    message: Awaiting next reconciliation
    reason: Successful
    status: "True"
    type: Running
    controlPlaneStatus: Completed
    lastReconcileEnd: 2021-07-20_21:39:32
    lastReconcileStart: 2021-07-20_21:38:59
  ```

  - Test login to CPD 4.0 console

  Find out the initial CPD 4.0 admin password by


  ```
  oc extract secret/admin-user-details -n $NS --keys=initial_admin_password --to=-
  ```

  Then navigate to the CPD console, and you should be able login with username "admin" and the listed password in the previous command.


---
### [Deploy Watson Knowledge Catalog](#deploy-watson-knowledge-catalog)


  - Create an Operator Subscription for the Watson Knowledge Catalog Operator by

  ```
NS="ibm-common-services"

cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    app.kubernetes.io/instance:  ibm-cpd-wkc-operator-catalog-subscription
    app.kubernetes.io/managed-by: ibm-cpd-wkc-operator
    app.kubernetes.io/name:  ibm-cpd-wkc-operator-catalog-subscription
  name: ibm-cpd-wkc-operator-catalog-subscription
  namespace: ibm-common-services
spec:
    channel: v1.0
    installPlanApproval: Automatic
    name: ibm-cpd-wkc
    source: ibm-cpd-wkc-operator-catalog
    sourceNamespace: openshift-marketplace
EOF
  ```

  - Validate that the Operator Subscription was successfully created.

  ```
NS="ibm-common-services"

SUB=$(oc get sub -n $NS ibm-cpd-wkc-operator-catalog-subscription \
-o jsonpath='{.status.installedCSV} {"\n"}') && echo $SUB

oc get csv -n $NS $SUB \
-o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n $NS -l olm.owner="$SUB" \
-o jsonpath="{.items[0].status.availableReplicas} {'\n'}"
  ```

  - Create Custom Resource to install Watson Knowledge Catalog

  ```
NS="cpd-instance"
VER="4.0.5"

cat << EOF | oc apply -f -
apiVersion: wkc.cpd.ibm.com/v1beta1
kind: WKC
metadata:
  name: wkc-cr
  namespace: $NS
spec:
  version: $VER
  license:
    accept: true
    license: Enterprise
  storageClass: ${STORAGE_CLASS_NAME}
EOF
  ```

  This will start the WKC service installation, and automatically install CCS, Data refinery, DB2 as a service, IIS, and UG in that order.

  - Monitor progress of Watson Knowledge Catalog deployment

  ```
oc get WKC wkc-cr -n $NS -o yaml
oc get CCS ccs-cr -n $NS -o yaml
oc get DataRefinery datarefinery-sample -n $NS -o yaml
oc get Db2aaserviceService db2aaservice-cr -n $NS -o yaml
oc get IIS iis-cr -n $NS -o yaml
oc get UG ug-cr -n $NS -o yaml
  ```

  Look the Status section at the end of the output, and it will show if it's in-progress, Successful, or Failed, etc. When WKC installation is completed  successfully, all the commands below should output "Completed".
    
```
oc get WKC wkc-cr -n cpd-instance -o jsonpath='{.status.wkcStatus} {"\n"}'
oc get CCS ccs-cr -n cpd-instance -o jsonpath='{.status.ccsStatus} {"\n"}'
oc get DataRefinery datarefinery-sample -n cpd-instance -o jsonpath='{.status.datarefineryStatus} {"\n"}'
oc get Db2aaserviceService db2aaservice-cr -n cpd-instance -o jsonpath='{.status.db2aaserviceStatus} {"\n"}'
oc get IIS iis-cr -n cpd-instance -o jsonpath='{.status.iisStatus} {"\n"}'
oc get UG ug-cr -n cpd-instance -o jsonpath='{.status.ugStatus} {"\n"}'
```

---
### [General Flow on Installing Individual CPD Services](#general-flow-on-installing-individual-cpd-services)


  So far we have set up and configured all the infrastructure for a bastion host based private registry; tune and update all the node and OCP parameters and settings;
  also installed all the common components like the IBM Common Services, i.e. Bedrock, scheduling service,  CPD platform Operator, Zen Operator, Zenservice/Lite, and one
  service as WKC, the rest of installing each individual service will be straightforward and much simpler. To install an individual service, follow the general steps below:

  1. download the required CASE packages for the service: refer to the section "__6. Downloading service CASE packages__" at https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=registry-mirroring-images-bastion-node
  
  2. mirror the images to the private registry: refer to the section "__7. Mirroring the images to the private registry__" at https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=registry-mirroring-images-bastion-node
  
  3. add the catalog sources required for the service: refer to the section "__Service catalog source__" at https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-configuring-your-cluster-pull-images
  
  4. create the service operator subscription to install the operators: refer to the section "__4. Creating an operator subscription for services__" at https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-creating-operator-subscriptions
  
  5. install the service by creating the service custom resource: refer to the corresponding section on installing the particular service at https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=integrations-services. For example, for WKC, see https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=iwkc-installing-watson-knowledge-catalog; and for WSL, see https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=iws-installing-watson-studio

>NB: some of service installations might require extra steps. You should double check the corresponding sections for that service under: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=integrations-services.


---
### [About the CASE Download Directory](#about-the-case-download-directory)

  The CASE archive download directory at ${OFFLINEDIR} is the location where all CASE archive files and their .csv files will be stored. As users download and install more and more services, this directory might accumulate several dozens or even hundreds of those files. There are two complications:

  1. when users do an image mirroring for a particular service, the `cloudctl` utility will go through all those .csv files and such files in any sub-directories and try to mirror them all again though majority of those images should have already been downloaded and mirrored. This will cause the download / mirroring step to take a very long time, even hours.
  
  2. if any of services users have downloaded before contains errors causing the image mirroing to fail, it might impact subsequent services image mirroring to fail too even though they are not related at all, i.e. not dependent on each other.

  To workaround the two issues above and speed up the image mirroring for the service being installed, users can move out those files from previously downloaded services to a backup directory using the commands below or similar:

  ```
  mkdir ${OFFLINEDIR}/../offline-backup
  mv -f ${OFFLINEDIR}/*.csv ${OFFLINEDIR}/../offline-backup
  mv -f ${OFFLINEDIR}/*.tgz ${OFFLINEDIR}/../offline-backup
  ```

  Ultimately we found the safest way to avoid the above problems is to use a separate download directory for each service CASE archive download, for example, before starting each service (here we use `wsl` as an example), do the following:

  ```
  mkdir ${OFFLINEDIR}/../offline-wsl
  export OFFLINEDIR=${OFFLINEDIR}/../offline-wsl
  ```

  Note that: __If you keep getting issues or a very long and slow process in the image mirroring step or you still see a lot of unrelated .csv files are been included at the beginning of the `mirror-images` step, use this workaorund.__ 


---
### [Deploy Watson Studio Service](#deploy-watson-studio-service)


  - download WSL CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-wsl-2.0.0.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the WSl images to the private registry

```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-wsl-2.0.0.tgz \
    --inventory wslSetup \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
```

  - add the WSL catalog source

```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-wsl-2.0.0.tgz \
    --inventory wslSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
```

  - create Watson Studio Service operator subscription

  ```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  annotations:
  name: ibm-cpd-ws-operator-catalog-subscription
  namespace: ibm-common-services
spec:
  channel: v2.0
  installPlanApproval: Automatic
  name: ibm-cpd-wsl 
  source: ibm-cpd-ws-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
  ```

  - install Watson Studio Service by creating a WS CR named as `ws-cr`.

  ```
cat <<EOF |oc apply -f -
apiVersion: ws.cpd.ibm.com/v1beta1
kind: WS
metadata:
  name: ws-cr
  namespace: cpd-instance
spec:
  license:
    accept: true
    license: Enterprise
  version: 4.0.0
  storageClass: ${STORAGE_CLASS_NAME}
  docker_registry_prefix: cp.icr.io/cp/cpd
EOF
  ```

---
### [Deploy Watson Machine Learning Service](#deploy-watson-machine-learning-service)


  - download WML CASE package archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-wml-cpd-4.0.1.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the WMl images to the private registry
  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-wml-cpd-4.0.1.tgz \
    --inventory wmlOperatorSetup \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the WML catalog source

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-wml-cpd-4.0.1.tgz \
    --inventory wmlOperatorSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - Create a Subscription for the Watson Machine Learning Operator

  ```
  cat <<EOF |oc apply -f -
  apiVersion: operators.coreos.com/v1alpha1
  kind: Subscription
  metadata:
    labels:
      app.kubernetes.io/instance: ibm-cpd-wml-operator-subscription
      app.kubernetes.io/managed-by: ibm-cpd-wml-operator
      app.kubernetes.io/name: ibm-cpd-wml-operator-subscription
    name: ibm-cpd-wml-operator-subscription
    namespace: ibm-common-services
  spec:
    channel: alpha
    installPlanApproval: Automatic
    name: ibm-cpd-wml-operator
    source: ibm-cpd-wml-operator-catalog
    sourceNamespace: openshift-marketplace
EOF
  ```

  - Create a Custom Resource CR to install Watson Machine Learning

  ```
cat <<EOF |oc apply -f -
apiVersion: wml.cpd.ibm.com/v1beta1
kind: WmlBase
metadata:
  name: wml-cr
  namespace: cpd-instance
spec:
  version: 4.0.0
  license:
    accept: true
    license: Enterprise
  storageVendor: ""
  storageClass: ${STORAGE_CLASS_NAME}
  is_35_upgrade: false
  ignoreForMaintenance: false
  docker_registry_prefix: cp.icr.io/cp/cpd                  
EOF
  ```

---
### [Deploy Watson OpenScale Service](#deploy-watson-openscale-service)

  - download Watson OpenScale CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-watson-openscale-2.0.1.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the Watson OpenScale images to the private registry

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-watson-openscale-2.0.1.tgz \
    --inventory ibmWatsonOpenscaleOperatorSetup \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the Watson OpenScale catalog source

```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-watson-openscale-2.0.1.tgz \
    --inventory ibmWatsonOpenscaleOperatorSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
```

  - create a subscription for Watson OpenScale Service operator


  ```
  cat <<EOF |oc apply -f -
  apiVersion: operators.coreos.com/v1alpha1
  kind: Subscription
  metadata:
    name: ibm-watson-openscale-operator-subscription
    labels:
      app.kubernetes.io/instance: ibm-watson-openscale-operator-subscription
      app.kubernetes.io/managed-by: ibm-watson-openscale-operator
      app.kubernetes.io/name: ibm-watson-openscale-operator-subscription
    namespace: ibm-common-services
  spec:
    channel: alpha
    installPlanApproval: Automatic
    name: ibm-cpd-wos
    source: ibm-openscale-operator-catalog
    sourceNamespace: openshift-marketplace
EOF
  ```

  - install Watson OpenScale Service by creating a custom resource CR for it

  ```
cat <<EOF |oc apply -f -
apiVersion: wos.cpd.ibm.com/v1
kind: WOService
metadata:
  name: aiopenscale
  namespace: cpd-instance
spec:
  version: 4.0.0
  scaleConfig: small
  license:
    accept: true
    license: Enterprise
  type: service
  storageClass: ${STORAGE_CLASS_NAME}
EOF
  ```


---
### [Deploy RStudio Server with R 3.6](#deploy-rstudio-server-with-r-36)


  - download RStudio Server with R 3.6 CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-rstudio-1.0.0.tgz \
    --outputdir ${OFFLINEDIR}
  ```
  - mirror the RStudio Server with R 3.6 images to the private registry

```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-rstudio-1.0.0.tgz \
    --inventory rstudioSetup \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
```

  - add the RStudio Server with R 3.6 catalog source

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-rstudio-1.0.0.tgz \
    --inventory rstudioSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  There is an error in the catalog image path: need to change "latest" to "latest-amd64".

  - create a subscription for RStudio Server with R 3.6 Service operator

  ```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    app.kubernetes.io/instance: ibm-cpd-rstudio-operator-catalog-subscription
    app.kubernetes.io/managed-by: ibm-cpd-rstudio-operator
    app.kubernetes.io/name: ibm-cpd-rstudio-operator-catalog-subscription
  name: ibm-cpd-rstudio-operator-catalog-subscription
  namespace: ibm-common-services
spec:
    channel: v1.0
    installPlanApproval: Automatic
    name: ibm-cpd-rstudio
    source: ibm-cpd-rstudio-operator-catalog
    sourceNamespace: openshift-marketplace
EOF
  ```

  - install RStudio Server with R 3.6 Service by creating a custom resource CR for it

  ```
  cat <<EOF |oc apply -f -
  apiVersion: rstudio.cpd.ibm.com/v1beta1
  kind: RStudioAddon
  metadata:
    name: rstudio-cr
    namespace: cpd-instance
  spec:
    version: "4.0.0"
    size: small
    state: present
    operation: install
    license:
      license: Enterprise
      accept: true
    storageClass: ${STORAGE_CLASS_NAME}
    cloudpak_instance_id: "rstudio"
    docker_registry_prefix: "cp.icr.io/cp/cpd"
EOF
​```

---
### [Set up WSL Runtimes](#set-up-wsl-runtimes)

  __You can skip this section for now unless yuo encounter any error in the later sections that complains about `ibm-wsl-runtimes` that causes failure.__

  Note that: this step is a workaround step. Without it, several of the subsequent services will fail to be installed successfully.

  - download the WSL Runtimes CASE archive

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-wsl-runtimes-1.0.0.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the WSL Runtimes images

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-wsl-runtimes-1.0.0.tgz \
    --inventory runtimesOperatorSetup \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the WSL Runtimes catalog source
  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-wsl-runtimes-1.0.0.tgz \
    --inventory runtimesOperatorSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - update the catalog source image path adding /cpopen after the hostname and port

  ```
   oc edit catalogsource ibm-cpd-ws-runtimes-operator-catalog -n openshift-marketplace
  ```

---
### [Deploy IBM Match 360 with Watson Service](#deploy-ibm-match-360-with-watson-service)

  - download IBM Match 360 with Watson CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-mdm-1.0.1.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the IBM Match 360 with Watson images to the private registry

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-mdm-1.0.1.tgz \
    --inventory mdmOperator \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the IBM Match 360 with Watson catalog source

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-mdm-1.0.1.tgz \
    --inventory mdmOperator \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - create a subscription for IBM Match 360 with Watson Service operator

  ```
  cat <<EOF |oc apply -f -
  apiVersion: operators.coreos.com/v1alpha1
  kind: Subscription
  metadata:
    name: ibm-mdm-operator-subscription
    namespace: ibm-common-services
    labels:
      app.kubernetes.io/instance: ibm-mdm-operator-subscription
      app.kubernetes.io/managed-by: ibm-mdm-operator
      app.kubernetes.io/name: ibm-mdm-operator-subscription
  spec:
    channel: alpha
    installPlanApproval: Automatic
    name: ibm-mdm
    source: ibm-mdm-operator-catalog
    sourceNamespace: openshift-marketplace
EOF
  ```

  - install IBM Match 360 with Watson Service by creating a custom resource CR for it

  ```
cat <<EOF |oc apply -f -
apiVersion: mdm.cpd.ibm.com/v1
kind: MasterDataManagement
metadata:
  name: mdm-cr
  namespace: cpd-instance
spec:
  license:
    accept: true
    license: Enterprise
  version: 1.0.0
  persistence:
    storage_class: ${STORAGE_CLASS_NAME}
EOF
​```

  Note that: __FAILED-HERE__ the installation will fail and stuck due to some wrong and unmirrored images.

---
### [Deploy Db2 Data Management Console](#deploy-db2-data-management-console)


  - download Db2 Data Management Console CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-dmc-4.0.0.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the Db2 Data Management Console images to the private registry

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-dmc-4.0.0.tgz \
    --inventory dmcOperatorSetup \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the Db2 Data Management Console catalog source

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-dmc-4.0.0.tgz \
    --inventory dmcOperatorSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - create a subscription for Db2 Data Management Console Service operator

  ```
  cat <<EOF |oc apply -f -
  apiVersion: operators.coreos.com/v1alpha1
  kind: Subscription
  metadata:
    name: ibm-dmc-operator-subscription
    namespace: ibm-common-services
  spec:
    channel: v1.0
    installPlanApproval: Automatic
    name: ibm-dmc-operator
    source: ibm-dmc-operator-catalog
    sourceNamespace: openshift-marketplace
EOF
  ```

  - run OCP patch command below (needed?)

  ```
  oc patch namespacescope common-service --type='json' -p='[{"op":"replace", "path": "/spec/csvInjector/enable", "value":true}]' -n ibm-common-services
  ```

  - install Db2 Data Management Console Service by creating a custom resource CR for it

  ```
  cat <<EOF |oc apply -f -
  apiVersion: dmc.databases.ibm.com/v1
  kind: Dmcaddon
  metadata:
    name: dmc-addon
    namespace: cpd-instance
  spec:
    version: 4.0.0
    license:
      accept: true
      license: Enterprise
EOF
  ```

---
### [Deploy OpenPages](#deploy-openPages)


  - download OpenPages CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-openpages-2.0.0+20210615.125604.820387.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the OpenPages images to the private registry

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-openpages-2.0.0+20210615.125604.820387.tgz \
    --inventory operatorSetup \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the OpenPages catalog source

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-openpages-2.0.0+20210615.125604.820387.tgz \
    --inventory operatorSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - create a subscription for OpenPages Service operator

  ```
  cat <<EOF |oc apply -f -
  apiVersion: operators.coreos.com/v1alpha1
  kind: Subscription
  metadata:
    name: ibm-cpd-openpages-operator
    namespace: ibm-common-services
  spec:
    channel: v1.0
    installPlanApproval: Automatic
    name: ibm-cpd-openpages-operator
    source: ibm-cpd-openpages-operator-catalog
    sourceNamespace: openshift-marketplace
EOF
  ```

  - install OpenPages Service by creating a custom resource CR for it 

  ```
  cat <<EOF |oc apply -f -
  apiVersion: openpages.cpd.ibm.com/v1 
  kind: OpenPagesService 
  metadata:
    name: cp4d-openpages-service
    namespace: cpd-instance
    labels:
      app.kubernetes.io/name: openpages
      app.kubernetes.io/instance: openpages-service
      app.kubernetes.io/version: "8.203.0"
      app.kubernetes.io/managed-by: ibm-cpd-openpages-operator 
  spec:
    version: "8.203.0"
    license:
      accept: true
      license: Enterprise
EOF
  ```

---
### [Deploy Watson Machine Learning Accelerator](#deploy-watson-machine-learning-accelerator)


  - download Watson Machine Learning Accelerator CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-wml-accelerator-2.3.0.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the Watson Machine Learning Accelerator images to the private registry

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-wml-accelerator-2.3.0.tgz \
    --inventory wmla_operator_deploy \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the Watson Machine Learning Accelerator catalog source

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-wml-accelerator-2.3.0.tgz \
    --inventory wmla_operator_deploy \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - create a subscription for Watson Machine Learning Accelerator Service operator

  ```
  cat <<EOF |oc apply -f -
  apiVersion: operators.coreos.com/v1alpha1
  kind: Subscription
  metadata:
    name: ibm-cpd-wml-accelerator-operator
    namespace: ibm-common-services
  spec:
    name: ibm-cpd-wml-accelerator-operator
    channel: WML-Accelerator-2.3
    installPlanApproval: Automatic
    source: ibm-cpd-wml-accelerator-catalog
    sourceNamespace: openshift-marketplace
EOF
  ```

  - install Watson Machine Learning Accelerator Service by creating a custom resource CR for it

  ```
  cat << EOF | oc apply -f -
  apiVersion: spectrumcomputing.ibm.com/v1
  kind: Wmla-add-on
  metadata:
    name: wmla
    namespace: cpd-instance
  spec:
    version: "2.3.0"
EOF
  ```

---
### [Deploy Data Virtualization](#deploy-data-virtualization)


  - download Data Virtualization CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-dv-case-1.7.0.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the Data Virtualization images to the private registry

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-dv-case-1.7.0.tgz \
    --inventory dv \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the Data Virtualization catalog source

  Db2U

  ```
  cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-db2uoperator-4.0.0.tgz \
  --inventory db2uOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  Note that: _this step will fail, and ignore it, and continue!_

  Data Virtualization

  ```
  cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-dv-case-1.7.0.tgz \
  --inventory dv \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - WORKAROUND - edit the ibm-dv-operator-catalog to add `/cpopen` in the image path after the hostname and port

  ```
  oc edit catalogsource ibm-dv-operator-catalog -n openshift-marketplace
  ```

  - create a subscription for Db2U operator  and Data Virtualization Service operator

  ```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-db2u-operator
  namespace: ibm-common-services
spec:
  channel: v1.1
  name: db2u-operator
  installPlanApproval: Automatic
  source: ibm-db2uoperator-catalog
  sourceNamespace: openshift-marketplace
EOF

cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-dv-operator-catalog-subscription
  namespace: ibm-common-services
spec:
  channel: v1.7
  installPlanApproval: Automatic
  name: ibm-dv-operator
  source: ibm-dv-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
  ```

  - Run this command to support watching multiple namespaces using the NamespaceScope operator

  ```
  oc patch namespacescope common-service --type='json' -p='[{"op":"replace", "path": "/spec/csvInjector/enable", "value":true}]' -n ibm-common-services
  ```

  - install Data Virtualization Service by creating a custom resource CR for it

  ```
cat <<EOF |oc apply -f -
apiVersion: db2u.databases.ibm.com/v1
kind: DvService
metadata:
  name: dv-service
  namespace: cpd-instance
spec:
  license:
    accept: true
    license: Enterprise
  version: 1.7.3
  size: "small"
  storageClass: ${STORAGE_CLASS_NAME}
EOF

cat <<EOF |oc apply -f -
	apiVersion: db2u.databases.ibm.com/v1
	kind: DvService
	metadata:
	  name: dv-service
	  namespace: cpd-instance
	spec:
	  version: 1.7.0
	  license:
	    accept: true
	    license: Enterprise
	  size: "small"
	  storageClass: ${STORAGE_CLASS_NAME}
	  docker_registry_prefix: "cp.icr.io/cp/cpd"
EOF
  ```

---
### [Deploy Db2 OLTP](#deploy-db2-oltp)

  - download Db2 OLTP CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-db2oltp-4.0.0.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the Db2 OLTP images to the private registry

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-db2oltp-4.0.0.tgz \
    --inventory db2oltpOperatorSetup \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the Db2 OLTP catalog source

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-db2uoperator-4.0.0.tgz \
    --inventory db2uOperatorSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  Note that: _This one will fail, and ignore it for now, and continue!_
    
  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-db2oltp-4.0.0.tgz \
    --inventory db2oltpOperatorSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - Create an Operator Subscription for the Db2 OLTP Operator

  Db2U

  ```
	cat <<EOF |oc apply -f -
	apiVersion: operators.coreos.com/v1alpha1
	kind: Subscription
	metadata:
	  name: ibm-db2uoperator-catalog-subscription
	  namespace: ibm-common-services
	spec:
	  channel: v1.1
	  name: db2u-operator
	  installPlanApproval: Automatic
	  source: ibm-db2uoperator-catalog
	  sourceNamespace: openshift-marketplace
EOF
  ```

  Db2oltp

  ```
	cat <<EOF |oc apply -f -
	apiVersion: operators.coreos.com/v1alpha1
	kind: Subscription
	metadata:
	  name: ibm-db2oltp-cp4d-operator-catalog-subscription
	  namespace: ibm-common-services
	spec:
	  channel: v1.0
	  name: ibm-db2oltp-cp4d-operator
	  installPlanApproval: Automatic
	  source: ibm-db2oltp-cp4d-operator-catalog
	  sourceNamespace: openshift-marketplace
EOF
  ```

  - install Db2 OLTP Service by creating a custom resource CR for it


  ```
	cat <<EOF |oc apply -f -
	apiVersion: databases.cpd.ibm.com/v1
	kind: Db2oltpService
	metadata:
	  name: db2oltp-cr
	  namespace: cpd-instance
	spec:
	  version: 4.0.0
	  license:
	    accept: true
	    license: Advanced
	  storageClass: ${STORAGE_CLASS_NAME}
EOF
  ```

---
### [Deploy Jupyter Notebooks with R 3.6](#deploy-jupyter-notebooks-with-r-36)

  - download Jupyter Notebooks with R 3.6 CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-wsl-runtimes-1.0.0.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the Jupyter Notebooks with R 3.6 images to the private registry

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-wsl-runtimes-1.0.0.tgz \
    --inventory runtimesOperatorSetup \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the Jupyter Notebooks with R 3.6 catalog source

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-wsl-runtimes-1.0.0.tgz \
    --inventory runtimesOperatorSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - Create an Operator Subscription for the Jupyter Notebooks with R 3.6 Operator


  ```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    app.kubernetes.io/instance: ibm-cpd-ws-runtimes-operator-catalog-subscription
    app.kubernetes.io/managed-by: ibm-cpd-ws-runtimes-operator
    app.kubernetes.io/name: ibm-cpd-ws-runtimes-operator
  name: ibm-cpd-ws-runtimes-operator
  namespace: ibm-common-services
spec:
  channel: v1.0
  installPlanApproval: Automatic
  name: ibm-cpd-ws-runtimes
  source: ibm-cpd-ws-runtimes-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
​```

  - Create Custom Resource to install Jupyter Notebooks with R 3.6

  ```
	cat <<EOF |oc apply -f -
	apiVersion: ws.cpd.ibm.com/v1beta1
	kind: NotebookRuntime
	metadata:
	  name: ibm-cpd-ws-runtime-r36
	  namespace: cpd-instance
	spec:
	  license:
	    accept: true
	    license: Enterprise
	  version: 4.0.0
EOF
  ```

---
### [Deploy SPSS Modeler](#deploy-spss-modeler)

  - download SPSS Modeler CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-spss-1.0.0.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the SPSS Modeler images to the private registry

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-spss-1.0.0.tgz \
    --inventory spssSetup \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```
  - add the SPSS Modeler catalog source

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-spss-1.0.0.tgz \
    --inventory spssSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  Note that: _This step will produce some error messages at the end, ignore it for now, continue!_


  - Create an Operator Subscription for the SPSS Modeler Operator

  ```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    app.kubernetes.io/instance: ibm-cpd-spss-operator-catalog-subscription
    app.kubernetes.io/managed-by: ibm-cpd-spss-operator
    app.kubernetes.io/name: ibm-cpd-spss-operator-catalog-subscription
  name: ibm-cpd-spss-operator-catalog-subscription
  namespace: ibm-common-services
spec:
    channel: v1.0
    installPlanApproval: Automatic
    name: ibm-cpd-spss
    source: ibm-cpd-spss-operator-catalog
    sourceNamespace: openshift-marketplace
EOF
​```

  - Create Custom Resource to install SPSS Modeler


  ```
cat <<EOF |oc apply -f -
apiVersion: spssmodeler.cpd.ibm.com/v1
kind: Spss
metadata:
  name: spss-cr
  namespace: cpd-instance
spec:
  version: "4.0.0"
  scaleConfig: "small"     
  architecture: "amd64"   
  operation: "install"    
  license:
    accept: true
    license: "Enterprise"
  storageClass: ${STORAGE_CLASS_NAME}
  docker_registry_prefix: "cp.icr.io/cp/cpd"
EOF
​```

---
### [Deploy Db2 Warehouse Service](#deploy-db2-warehouse-service)

   - download Db2 Warehouse Service CASE archive file

  ```
  cloudctl case save \
--case ${CASE_REPO_PATH}/ibm-db2wh-4.0.0.tgz \
--outputdir ${OFFLINEDIR}
  ```

   - mirror the Db2 Warehouse Service images to the private registry

  ```
  cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-db2wh-4.0.0.tgz \
  --inventory db2whOperatorSetup \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
```

  - add the Db2 Warehouse Service catalog source

```
  cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-db2uoperator-4.0.0.tgz \
  --inventory db2uOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  Note that: this step will fail, and ignore it, and continue!

  ```
  cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-db2wh-4.0.0.tgz \
  --inventory db2whOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
    
  ```

  - create a subscription for Db2 Warehouse Service Service operator

  ```
  cat <<EOF |oc apply -f -
	apiVersion: operators.coreos.com/v1alpha1
	kind: Subscription
	metadata:
	  name: ibm-db2wh-cp4d-operator-catalog-subscription
	  namespace: ibm-common-services
	spec:
	  channel: v1.0
	  name: ibm-db2wh-cp4d-operator
	  installPlanApproval: Automatic
	  source: ibm-db2wh-cp4d-operator-catalog
	  sourceNamespace: openshift-marketplace
EOF
  ```

   - install Db2 Warehouse Service Service by creating a custom resource CR for it

  ```
  cat <<EOF |oc apply -f -
	apiVersion: databases.cpd.ibm.com/v1
	kind: Db2whService
	metadata:
	  name: db2wh-cr
	  namespace: cpd-instance
	spec:
	  license:
	    accept: true
	    license: Enterprise
EOF
  ```

---
### [Deploy Decision Optimization](#deploy-decision-optimization)


__Note that: the code for DODS below should be correct, but it does not work likely due to some packaging or staging bugs__ 

  - download Decision Optimization CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-dods-4.0.0.tgz \
    --outputdir ${OFFLINEDIR}
```

  - mirror the Decision Optimization images to the private registry

```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-dods-4.0.0.tgz \
    --inventory dodsOperatorSetup \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the Decision Optimization catalog source

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-dods-4.0.0.tgz \
    --inventory dodsOperatorSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - create a subscription for Decision Optimization Service operator

  ```
  cat <<EOF |oc apply -f -
  apiVersion: operators.coreos.com/v1alpha1
  kind: Subscription
  metadata:
    labels:
      app.kubernetes.io/instance: ibm-cpd-dods-operator-catalog-subscription
      app.kubernetes.io/managed-by: ibm-cpd-dods-operator
      app.kubernetes.io/name: ibm-cpd-dods-operator-catalog-subscription
    name: ibm-cpd-dods-operator-catalog-subscription
    namespace: ibm-common-services
  spec:
    channel: alpha
    installPlanApproval: Automatic
    name: ibm-cpd-dods
    source: ibm-cpd-dods-operator-catalog
    sourceNamespace: openshift-marketplace
EOF
  ```

  - install Decision Optimization Service by creating a custom resource CR for it

  ```
  cat <<EOF |oc apply -f -
  apiVersion: dods.cpd.ibm.com/v1beta1
  kind: DODS
  metadata:
    name: dods-cr
    namespace: cpd-instance
  spec:
    version: 4.0.0
    license:
      accept: true
      license: Enterprise
EOF
  ```

---
### [Deploy Db2 Big SQL](#deploy-db2-big-sql)

  - download Db2 Big SQL CASE archive file

  ```
cloudctl case save \
  --case ${CASE_REPO_PATH}/ibm-bigsql-case-7.2.0.tgz \
  --outputdir ${OFFLINEDIR}
  ```

  - mirror the Db2 Big SQL images to the private registry

  ```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-bigsql-case-7.2.0.tgz \
  --inventory bigsql \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the Db2 Big SQL catalog source

  ```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-db2uoperator-4.0.0.tgz \
  --inventory db2uOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"

  ......
Welcome to the CASE launcher
Attempting to retrieve and extract the CASE from the specified location
error: Error occurred extracting the specified CASE tgz: open /root/offline/ibm-db2uoperator-4.0.0.tgz: no such file or directory
  ```

  Ignore the above error, continue!

  ```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-bigsql-case-7.2.0.tgz \
  --inventory bigsql \
  --namespace openshift-marketplace \
  --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - Update: add /cpopen after hostname:port in the catalog image path

  ```
oc edit catalogsource ibm-bigsql-operator-catalog -n openshift-marketplace
​```

  - create a subscription for Db2 Big SQL Service operator

  ```
	cat <<EOF |oc apply -f -
	apiVersion: operators.coreos.com/v1alpha1
	kind: Subscription
	metadata:
	  name: ibm-bigsql-operator-catalog-subscription
	  namespace: ibm-common-services
	spec:   
	  channel: v1.0
	  installPlanApproval: Automatic
	  name: ibm-bigsql-operator
	  source: ibm-bigsql-operator-catalog
	  sourceNamespace: openshift-marketplace
EOF
  ```

  - install Db2 Big SQL Service by creating a custom resource CR for it

  ```
	cat <<EOF |oc apply -f -
	apiVersion: db2u.databases.ibm.com/v1
	kind: BigsqlService
	metadata:
	  name: bigsql-service-cr
	  namespace: cpd-instance
	spec:
	  license:
	    accept: true
	    license: Enterprise
	  version: 7.2.0
	  storageClass: ${STORAGE_CLASS_NAME}
EOF
  ```

SUCCESS - Db2 BigSQL installed!

---
### [Jupyter Notebooks with Python 3.7 for GPU](#jupyter-notebooks-with-python-37-for-gpu)

  - download Jupyter Notebooks with Python 3.7 for GPU CASE archive file
  ```
cloudctl case save \
  --case ${CASE_REPO_PATH}/ibm-wsl-runtimes-1.0.0.tgz \
  --outputdir ${OFFLINEDIR}
  ```

  - mirror the Jupyter Notebooks with Python 3.7 for GPU images to the private registry

  ```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-wsl-runtimes-1.0.0.tgz \
  --inventory runtimesOperatorSetup \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

 - add the Jupyter Notebooks with Python 3.7 for GPU catalog source

  ```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-wsl-runtimes-1.0.0.tgz \
  --inventory runtimesOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - edit the catalog source by adding `/cpopen` in the image path after the DNS hostname and port

  ```
  oc edit catalogsource ibm-cpd-ws-runtimes-operator-catalog -n openshift-marketplace
  ```

  - create a subscription for Jupyter Notebooks with Python 3.7 for GPU Service operator

  ```
	cat <<EOF |oc apply -f -
	apiVersion: operators.coreos.com/v1alpha1
	kind: Subscription
	metadata:
	  labels:
	    app.kubernetes.io/instance: ibm-cpd-ws-runtimes-operator-catalog-subscription
	    app.kubernetes.io/managed-by: ibm-cpd-ws-runtimes-operator
	    app.kubernetes.io/name: ibm-cpd-ws-runtimes-operator
	  name: ibm-cpd-ws-runtimes-operator
	  namespace: ibm-common-services
	spec:
	  channel: v1.0
	  installPlanApproval: Automatic
	  name: ibm-cpd-ws-runtimes
	  source: ibm-cpd-ws-runtimes-operator-catalog
	  sourceNamespace: openshift-marketplace
EOF
  ```

  - install Jupyter Notebooks with Python 3.7 for GPU Service by creating a custom resource CR for it

  ```
	cat <<EOF |oc apply -f -
	apiVersion: ws.cpd.ibm.com/v1beta1
	kind: NotebookRuntime
	metadata:
	  name: ibm-cpd-ws-runtime-py37gpu
	  namespace: cpd-instance
	spec:
	  license:
	    accept: true
	    license: Enterprise
	  version: 4.0.0
EOF
  ```

---
### [Deploy Db2 Data Gate Service](#deploy-db2-data-gate-service)

  - download Db2 Data Gate Service CASE archive file

  ```
cloudctl case save \
  --case ${CASE_REPO_PATH}/ibm-datagate-prod-4.0.0.tgz \
  --outputdir ${OFFLINEDIR}
  ```

  - mirror the Db2 Data Gate Service images to the private registry

  ```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-datagate-prod-4.0.0.tgz \
  --inventory datagateOperatorSetup \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the Db2 Data Gate Service catalog source

  ```
cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-datagate-prod-4.0.0.tgz \
  --inventory datagateOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

 - create a subscription for Db2 Data Gate Service Service operator

  ```
	cat <<EOF |oc apply -f -
	apiVersion: operators.coreos.com/v1alpha1
	kind: Subscription
	metadata:
	  labels:
	    app.kubernetes.io/instance: ibm-datagate-operator-subscription
	    app.kubernetes.io/managed-by: ibm-datagate-operator
	    app.kubernetes.io/name: ibm-datagate-operator-subscription
	  name: ibm-datagate-operator-subscription
	  namespace: ibm-common-services  
	spec:
	    channel: alpha
	    installPlanApproval: Automatic
	    name: ibm-datagate-operator
	    source: ibm-datagate-operator-catalog
	    sourceNamespace: openshift-marketplace
EOF
  ```

  - install Db2 Data Gate Service Service by creating a custom resource CR for it

  ```
	cat <<EOF |oc apply -f -
	apiVersion: datagate.cpd.ibm.com/v1
	kind: DatagateService
	metadata:
	  name: datagateservice-cr
	  namespace: cpd-instance
	spec:
	  license:
	    accept: true
	    license: Enterprise
	  datagate: yes
	  version: 4.0.0
	  storageClass: ${STORAGE_CLASS_NAME}
EOF
  ```

---
### [Deploy Analytics Engine Powered by Apache Spark](#deploy-analytics-engine-powered-by-apache-spark)

  - download Analytics Engine Powered by Apache Spark CASE archive file

  ```
  cloudctl case save \
   --case ${CASE_REPO_PATH}/ibm-analyticsengine-4.0.0.tgz \
   --outputdir ${OFFLINEDIR}
  ```

  - mirror the Analytics Engine Powered by Apache Spark images to the private registry

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-analyticsengine-4.0.0.tgz \
    --inventory analyticsengineOperatorSetup \
    --action mirror-images \
    --namespace not-used-arg \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the Analytics Engine Powered by Apache Spark catalog source

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-analyticsengine-4.0.0.tgz \
    --inventory analyticsengineOperatorSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - create a subscription for Analytics Engine Powered by Apache Spark Service operator


  ```
	cat <<EOF |oc apply -f -
	apiVersion: operators.coreos.com/v1alpha1
	kind: Subscription
	metadata:
	  labels:
	    app.kubernetes.io/instance: ibm-cpd-ae-operator-subscription
	    app.kubernetes.io/managed-by: ibm-cpd-ae-operator
	    app.kubernetes.io/name: ibm-cpd-ae-operator-subscription
	  name: ibm-cpd-ae-operator-subscription
	  namespace: ibm-common-services
	spec:
	  channel: stable-v1
	  installPlanApproval: Automatic
	  name: analyticsengine-operator
	  source: ibm-cpd-ae-operator-catalog
	  sourceNamespace: openshift-marketplace
EOF
  ```

  - install Analytics Engine Powered by Apache Spark Service by creating a custom resource CR for it


  ```
	cat <<EOF |oc apply -f -
	apiVersion: ae.cpd.ibm.com/v1
	kind: AnalyticsEngine
	metadata:
	  name: analyticsengine-sample
	  namespace: cpd-instance
	spec:
	  version: 4.0.0
	  license:
	    accept: true
	    license: Enterprise
	  storageClass: ${STORAGE_CLASS_NAME}
EOF
  ```


---
### [Deploy Data Stage Service](#deploy-data-stage-service)

  - download data Stage Service CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-datastage-enterprise-4.0.0.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the data Stage Service images to the private registry

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-datastage-enterprise-4.0.0.tgz \
    --inventory datastageOperatorSetup \
    --action mirror-images \
    --namespace -not-used-arg \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the data Stage Service catalog source

  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-datastage-enterprise-4.0.0.tgz \
    --inventory datastageOperatorSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - modify the catalog source image path, and change the image path after the host DNS name and port to be: ```/cpopen/ds-ent-operator-catalog:latest-amd64``` by editing the catalog source manifest:

  ```
  oc edit  catalogsource ibm-datastage-operator-catalog -n openshift-marketplace
  ```

  - Create an Operator Subscription for the Data Stage Operator

  ```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata: 
  name: ibm-datastage-operator
  namespace: ibm-common-services
spec: 
  channel: v1.0
  installPlanApproval: Automatic 
  name: ibm-datastage-operator
  source: ibm-datastage-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
​```

  - install data Stage Service Service by creating a custom resource CR for it

  ```
cat <<EOF |oc apply -f -
apiVersion: dfd.cpd.ibm.com/v1alpha1
kind: DataStageService
metadata:
  name: datastage-cr
  namespace: cpd-instance
spec:
  version: 4.0.0
  license:
    accept: true
    license: Enterprise
EOF
​```

---
### [Deploy Execution Engine for Apache Hadoop](#deploy-execution-engine-for-apache-hadoop)
***

  - download xecution Engine for Apache Hadoop CASE archive file

  ```
  cloudctl case save \
    --case ${CASE_REPO_PATH}/ibm-hadoop-4.0.0.tgz \
    --outputdir ${OFFLINEDIR}
  ```

  - mirror the xecution Engine for Apache Hadoop images to the private registry
  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-hadoop-4.0.0.tgz \
    --inventory hadoopSetup \
    --action mirror-images \
    --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"
  ```

  - add the xecution Engine for Apache Hadoop catalog source
  ```
  cloudctl case launch \
    --case ${OFFLINEDIR}/ibm-hadoop-4.0.0.tgz \
    --inventory hadoopSetup \
    --namespace openshift-marketplace \
    --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"
  ```

  - Edit the Hadoop catalog source by adding `/cpopen` in its image path after the host DNS name and port:

  ```
  oc edit  catalogsource ibm-cpd-hadoop-operator-catalog -n openshift-marketplace
  ```

  - Create an Operator Subscription for the Execution Engine for Apache Hadoop Operator


  ```
	cat <<EOF |oc apply -f -
	apiVersion: operators.coreos.com/v1alpha1
	kind: Subscription
	metadata:
	  labels:
	    app.kubernetes.io/instance: ibm-cpd-hadoop-operator-catalog-subscription
	    app.kubernetes.io/managed-by: ibm-cpd-hadoop-operator
	    app.kubernetes.io/name: ibm-cpd-hadoop-operator-catalog-subscription
	  name: ibm-cpd-hadoop-operator-catalog-subscription
	  namespace: ibm-common-services
	spec:
	  channel: v1.0
	  installPlanApproval: Automatic
	  name: ibm-cpd-hadoop
	  source: ibm-cpd-hadoop-operator-catalog
	  sourceNamespace: openshift-marketplace
EOF
  ```

  - Create Custom Resource to install Execution Engine for Apache Hadoop


  ```
	cat <<EOF |oc apply -f -
	apiVersion: hadoop.cpd.ibm.com/v1
	kind: Hadoop
	metadata:
	  name: hadoop-cr
	  namespace: cpd-instance
	spec:
	  version: 4.0.0
	  license:
	    accept: true
	    license: Enterprise
	  storageClass: "${STORAGE_CLASS_NAME}"
	  docker_registry_prefix: cp.icr.io/cp/cpd
EOF
  ```

```