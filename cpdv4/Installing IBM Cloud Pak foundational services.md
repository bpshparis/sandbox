## Installing IBM Cloud Pak foundational services

### Installing the IBM Common Service Operator

```
source ~/cpd-cli/cpd-vars.sh

cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-operator-catalog
  namespace: ${PROJECT_CATSRC}
spec:
  displayName: ibm-operator-catalog
  publisher: IBM Content
  sourceType: grpc
  image: icr.io/cpopen/ibm-operator-catalog:v1.25-20230606.181339-D0C80EAC1
  updateStrategy:
    registryPoll:
      interval: 45m
EOF

oc -n ${PROJECT_CATSRC} get catalogsource ibm-operator-catalog -o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'
```

### Create resources

```
source ~/cpd-cli/cpd-vars.sh

cat << EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: ${PROJECT_CPFS_OPS}

---
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  name: operatorgroup
  namespace: ${PROJECT_CPFS_OPS}
spec:
  targetNamespaces:
  - ${PROJECT_CPFS_OPS}

---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-common-service-operator
  namespace: ${PROJECT_CPFS_OPS}
spec:
  channel: v3.23
  installPlanApproval: Automatic
  name: ibm-common-service-operator
  source: ibm-operator-catalog
  sourceNamespace: ${PROJECT_CATSRC}
EOF


oc -n ${PROJECT_CPFS_OPS} get csv
oc get crd | grep operandrequest
```

### Setting the hardware profile and accepting the license

```
source ~/cpd-cli/cpd-vars.sh

oc -n ${PROJECT_CPFS_OPS} edit commonservice common-service
```
> Update the spec.size parameter to set the hardware profile, and add the spec.license.accept: true parameter to accept the license.

```
spec:
  license:
    accept: true  
  size: small
```

### Installing foundational services

```
source ~/cpd-cli/cpd-vars.sh

cat << EOF | oc apply -f -
apiVersion: operator.ibm.com/v1alpha1
kind: OperandRequest
metadata:
  name: common-service
  namespace: ${PROJECT_CPFS_OPS}
spec:
  requests:
    - operands:
        - name: ibm-cert-manager-operator
        - name: ibm-iam-operator
        - name: ibm-monitoring-grafana-operator
        - name: ibm-healthcheck-operator
        - name: ibm-management-ingress-operator
        - name: ibm-licensing-operator
        - name: ibm-commonui-operator
        - name: ibm-ingress-nginx-operator
        - name: ibm-auditlogging-operator
        - name: ibm-platform-api-operator
        - name: ibm-events-operator
        - name: ibm-zen-operator
      registry: common-service
      registryNamespace: ${PROJECT_CPFS_OPS}
EOF
```