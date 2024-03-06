#  create operator group in namespace ibm-common-services
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  namespace: ibm-common-services
  name: operatorgroup
spec:
  targetNamespaces:
  - ibm-common-services



EOF

# downloading case package ibm-cp-common-services 1.19.3 
cloudctl case save --repo https://github.com/IBM/cloud-pak/raw/master/repo/case --case ibm-cp-common-services --version 1.19.3 --outputdir /tmp/work/offline/4.6.6/cpfs

# downloading case package ibm-cp-datacore 2.9.0 
cloudctl case save --repo https://github.com/IBM/cloud-pak/raw/master/repo/case --case ibm-cp-datacore --version 2.9.0 --outputdir /tmp/work/offline/4.6.6/cpd_platform --no-dependency

# install or update catalog source 'opencloud-operators' 
oc apply -f << EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: opencloud-operators
  namespace: openshift-marketplace
spec:
  displayName: IBMCS Operators
  publisher: IBM
  sourceType: grpc
  image: icr.io/cpopen/ibm-common-service-catalog@sha256:f5d2719f3e558e56fbbd0286a881a5a812e413337ef129d4ddea1285d3339a76

EOF

# create subscription : ibm-common-service-operator with opencloud-operators. Wait for subscription to complete successfully and the Operator pod to be ready.
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-common-service-operator
  namespace: ibm-common-services
spec:
  channel: v3.23
  installPlanApproval: Automatic
  name: ibm-common-service-operator
  source: opencloud-operators
  sourceNamespace: openshift-marketplace

EOF

# Patch NamespaceScope CR common-service in the namespace ibm-common-services to enable csvInjector
oc patch NamespaceScope common-service -n ibm-common-services --type=merge --patch='{"spec": {"csvInjector": {"enable": true} } }'

#  applying commonservice CR to scale its size up to small
cat <<EOF |oc apply -f -
apiVersion: operator.ibm.com/v3
kind: CommonService
metadata:
  name: common-service
  namespace: ibm-common-services
spec:
  size: small
EOF

# install or update catalog source 'cpd-platform' 
oc apply -f << EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: cpd-platform
  namespace: openshift-marketplace
spec:
  displayName: Cloud Pak for Data
  publisher: IBM
  sourceType: grpc
  image: icr.io/cpopen/ibm-cpd-platform-operator-catalog@sha256:9147e737ff029d573ec9aa018f265761caab37e135d09245f0770b3396259a04
  updateStrategy:
    registryPoll:
      interval: 45m

EOF

# validate the catalog source is READY before continuing. 
oc get catalogsource -n openshift-marketplace opencloud-operators \
-o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'

# validate the catalog source is READY before continuing. 
oc get catalogsource -n openshift-marketplace cpd-platform \
-o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'

# pre-subscription release patching (if any)
release-patches.sh pre_sub

# create subscription : cpd-operator with cpd-platform. Wait for subscription to complete successfully and the Operator pod to be ready.
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cpd-operator
  namespace: ibm-common-services
spec:
  channel: v3.8
  installPlanApproval: Automatic
  name: cpd-platform-operator
  source: cpd-platform
  sourceNamespace: openshift-marketplace

EOF

