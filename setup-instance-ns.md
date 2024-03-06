#  create OperandRequest in the cpd namespace
cat <<EOF |oc apply -f -
apiVersion: operator.ibm.com/v1alpha1
kind: OperandRequest
metadata:
  name: empty-request
  namespace: "cpd"
spec:
  requests: []

EOF

