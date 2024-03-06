```
OFFLINEDIR="/root/offline/wml"

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-wml-cpd-4.0.5.tgz \
  --inventory wmlOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--inputDir ${OFFLINEDIR} --recursive"

oc get catalogsource -n openshift-marketplace ibm-cpd-wml-operator-catalog -o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'

cat <<EOF |oc apply -f -
---
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
  channel: v1.1
  installPlanApproval: Automatic
  name: ibm-cpd-wml-operator
  source: ibm-cpd-wml-operator-catalog
  sourceNamespace: openshift-marketplace
EOF

oc get sub -n ibm-common-services ibm-cpd-wml-operator-subscription -o jsonpath='{.status.installedCSV} {"\n"}'

oc get csv -n ibm-common-services ibm-cpd-wml-operator.v1.1.3 -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ibm-common-services -l olm.owner="ibm-cpd-wml-operator.v1.1.3" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"

cat <<EOF |oc apply -f -
---
apiVersion: wml.cpd.ibm.com/v1beta1
kind: WmlBase
metadata:
  name: wml-cr
  namespace: cpd-instance
  labels:
    app.kubernetes.io/instance: wml-cr
    app.kubernetes.io/managed-by: ibm-cpd-wml-operator
    app.kubernetes.io/name: ibm-cpd-wml-operator
spec:
  ignoreForMaintenance: false
  scaleConfig: small
  license:
    accept: true
    license: Enterprise
  storageClass: ${STORAGE_CLASS_NAME}
EOF

oc get WmlBase wml-cr -o jsonpath='{.status.wmlStatus} {"\n"}'

oc get WmlBase wml-cr -o json | jq '.status.wmlStatus'
```

```
OFFLINEDIR="/root/offline/wsl"

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-wsl-2.0.4.tgz \
  --inventory wslSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
  --args "--inputDir ${OFFLINEDIR} --recursive"

oc get catalogsource -n openshift-marketplace ibm-cpd-ws-operator-catalog -o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'

cat <<EOF |oc apply -f -
---
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

oc get sub -n ibm-common-services ibm-cpd-ws-operator-catalog-subscription -o jsonpath='{.status.installedCSV} {"\n"}'

oc get csv -n ibm-common-services ibm-cpd-wsl.v2.0.4 -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n ibm-common-services -l olm.owner="ibm-cpd-wsl.v2.0.4" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"

cat <<EOF |oc apply -f -
---
apiVersion: ws.cpd.ibm.com/v1beta1
kind: WS
metadata:
  name: ws-cr
  namespace: cpd-instance
spec:
  license:
    accept: true
    license: Enterprise
  version: 4.0.4
  storageClass: ${STORAGE_CLASS_NAME}
EOF

oc get WS ws-cr -o jsonpath='{.status.wsStatus} {"\n"}'

oc get WS ws-cr -o json | jq '.status.wsStatus'
```


Pour accéder à la plateforme après connexion au serveur VPN,
ajouter et ou modifier les lignes suivantes au fichier hosts du client.
e.g. /etc/hosts ou C:\Windows\System32\drivers\etc\hosts
 
...
172.16.187.60 console-openshift-console.apps.ocp6.iicparis.fr.ibm.com
172.16.187.60 oauth-openshift.apps.ocp6.iicparis.fr.ibm.com
172.16.187.60 cpd-cpd-instance.apps.ocp6.iicparis.fr.ibm.com
...

Note: Tout le trafic passe par le load balancer (172.16.187.60). 
Donc si un fqdn n'est pas résolu alors ajouter une ligne dans le fichier hosts.
e.g. 172.16.187.60 myapp-mynamespace.apps.ocp6.iicparis.fr.ibm.com

Accès à la console Cloud Pak for Data en tant que user admin mot de passe admin:
https://cpd-cpd-instance.apps.ocp6.iicparis.fr.ibm.com

Accès à Data Virtualization et Data Management Console:
https://cpd-cpd-instance.apps.ocp6.iicparis.fr.ibm.com/zen/#/myInstances

Accès aux projets Watson Machine Learning et Watson Studio:
https://cpd-cpd-instance.apps.ocp6.iicparis.fr.ibm.com/zen/#/projectList
 
Accès à la console Openshift en tant que user admin mot de passe admin (via htpasswd_provider):
https://console-openshift-console.apps.ocp6.iicparis.fr.ibm.com
[](img/loginwith-4.4.jpg)
 
Accès au cluster via la commande oc:
oc login https://172.16.187.60:6443 -u admin -p admin --insecure-skip-tls-verify=true -n cpd-instance
 
Installer les commandes oc and kubectl si nécessaire:
Linux: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
Windows: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-windows.zip
MacOS: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-mac.tar.gz
