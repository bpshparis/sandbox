# Install an SFTP server on OpenShift

## Set env

```
NODE="sno.ocp.local"
STORAGE_CLASS="nfs-csi"
LABEL="feat=sftp"
PROJECT="sftp"
GROUPID="100"
USERID="1001"
USERNAME="user"
PASSWORD="pwd"
```

## Label node

```
oc label node ${NODE} ${LABEL}
```

## Create and annotate project

```
oc new-project ${PROJECT}

oc annotate namespace ${PROJECT} 'openshift.io/node-selector='${LABEL} --overwrite
```

## Setup privileged Security Context and Service account

> :bulb: This is configured via a ServiceAccount, Role and RoleBinding

```
oc create serviceaccount sftp-serviceaccount

oc create role privileged-scc --verb=use --resource-name=privileged --resource=securitycontextconstraints

cat << EOF | oc apply -f -
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: use-privileged-scc
subjects:
 - kind: ServiceAccount
   name: sftp-serviceaccount
roleRef:
 kind: Role
 name: privileged-scc
 apiGroup: rbac.authorization.k8s.io
EOF
```

## Install sftp 

```
oc new-app atmoz/sftp

oc status
```

> :bulb: Don't pay attention to **pod will crash-looping**

## Add Security Context and to deployment

```
PATCH0='{"spec":{"template":{"spec":{"securityContext":{"fsGroup": '${GROUPID}'}}}}}'

oc patch deployment sftp -p "$PATCH0"

PATCH1='{"spec":{"template":{"spec":{"containers":[{"name":"sftp","securityContext": {"capabilities":{"add":["SYS_CHROOT"]},"runAsGroup":0,"runAsUser":0}}]}}}}'

oc patch deployment sftp -p "$PATCH1"

PATCH2='{"spec":{"template":{"spec":{"serviceAccountName":"sftp-serviceaccount"}}}}'

oc patch deployment sftp -p "$PATCH2"
```

> :bulb: Check Security Context

```
oc get deployment sftp -o json | jq -c -r '.spec.template.spec | .securityContext , .containers[].securityContext, .serviceAccountName'
```

## Setup SFTP Users

```
echo "${USERNAME}:${PASSWORD}:${USERID}:${GROUPID}:upload" > users.conf

oc create cm sftp-etc-sftp --from-file=users.conf

oc set volume deployment.apps/sftp --add --name=vol-sftp-etc --mount-path=/etc/sftp -t configmap --configmap-name=sftp-etc-sftp
```

## Create Persistent Volume Claim (PVC) to permanently store files uploaded by the users

```
oc set volume deployment.apps/sftp --add --name=vol-wynford -t pvc --claim-name=pvc-wynford --claim-size=1G -m /home/wynford/upload  --claim-class=${STORAGE_CLASS} --overwrite
```

## Set SFTP access from the internet

```
PATCH3='{"spec":{"type":"NodePort"}}'

oc patch svc sftp -p "$PATCH3"
```

## Restart deployment for change to take effect

```
oc scale deploy sftp --replicas=0 && sleep 2 && oc scale deploy sftp --replicas=1
```

## Display sftp endpoint

```
EXTERNAL_PORT=$(oc get svc -o json | jq '.items[].spec.ports[].nodePort')

oc get nodes -l ${LABEL} -o wide | awk 'NR>1  {print "SFTP server available at " $6 ":"'${EXTERNAL_PORT}'}'
```

> :checkered_flag: :checkered_flag: :checkered_flag: 
