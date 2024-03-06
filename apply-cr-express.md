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

#  applying ConfigMap olm-utils-cm
cat <<EOF |oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: olm-utils-cm
  labels:
    app.kubernetes.io/name: olm-utils
data:
  release_version: 4.6.6
  release_components_meta: |-
    analyticsengine:
        case_version: 6.5.0
        cr_version: 4.6.5
        csv_version: 3.5.0
        sub_channel: v3.5
        supported_arch: amd64
    bigsql:
        case_version: 8.4.0
        cr_version: 7.4.4
        csv_version: 8.4.0
        status_field: bigsqlStatus
        status_shutdown_failed: shutdownError
        status_shutdown_success: shutdown
        sub_channel: v8.4
        support_shutdown: true
        supported_arch: amd64
    ccs:
        case_version: 6.5.0
        cr_version: 6.5.0
        csv_version: 6.5.0
        sub_channel: v6.5
        supported_arch: amd64,s390x
    cde:
        case_version: 3.4.0
        cr_version: 4.6.5
        csv_version: 3.4.0
        sub_channel: v3.4
        supported_arch: amd64,ppc64le
    cognos_analytics:
        case_version: 23.5.0
        cr_template: cognos-analytics-cr-nd-bkup.yml.j2
        cr_version: 23.5.0
        csv_version: 23.5.0
        status_field: caStatus
        sub_channel: v23.5
    cpd_platform:
        case_version: 2.9.0
        cr_version: 4.6.6
        csv_version: 3.8.0
        status_operator_info_field: controlPlaneOperatorVersion
        sub_channel: v3.8
    cpfs:
        case_version: 1.19.3
        csv_version: 3.23.3
        sub_channel: v3.23
    dashboard:
        case_version: 3.0.0
        cr_version: 4.6.0
        csv_version: 3.0.0-20230103.143250-4e698b772
        sub_channel: candidate-v3.0
    data_governor:
        case_version: 2.3.3
        csv_version: 2.3.3
        sub_channel: v2.3
    datagate:
        case_version: 5.2.0
        cr_version: 3.2.0
        csv_version: 3.2.0
        skip_image_prune: false
        status_shutdown_failed: shutdownError
        status_shutdown_success: shutdown
        sub_channel: v3.2
        support_shutdown: true
        supported_arch: amd64,s390x
    datarefinery:
        case_version: 6.5.0
        cr_version: 6.5.0
        csv_version: 6.5.0
        sub_channel: v6.5
        supported_arch: amd64,s390x
    datastage_ent:
        case_version: 5.6.0
        component_dependencies:
        - ccs
        cr_api_version: v1
        cr_version: 4.6.6
        csv_version: 3.6.0
        olm_auto_update: true
        status_shutdown_failed: shutdownError
        status_shutdown_success: shutdown
        sub_channel: v3.6
        support_shutdown: true
        supported_arch: amd64
    datastage_ent_plus:
        case_version: 5.6.0
        catsrc_name: ibm-cpd-datastage-ent-plus-operator-catalog
        component_dependencies:
        - ccs
        cr_api_version: v1
        cr_version: 4.6.6
        csv_version: 3.6.0
        olm_auto_update: true
        status_shutdown_failed: shutdownError
        status_shutdown_success: shutdown
        sub_channel: v3.6
        support_shutdown: true
        supported_arch: amd64
    db2aaservice:
        case_version: 4.6.4
        cr_version: 4.6.4
        csv_version: 3.2.0
        status_shutdown_failed: shutdownError
        status_shutdown_success: shutdown
        sub_channel: v3.2
        support_shutdown: true
    db2oltp:
        case_version: 4.6.4
        cr_version: 4.6.4
        csv_version: 3.2.0
        status_shutdown_failed: shutdownError
        status_shutdown_success: shutdown
        sub_channel: v3.2
        support_shutdown: true
    db2u:
        case_version: 5.1.3
        csv_version: 3.2.0
        sub_channel: v3.2
    db2wh:
        case_version: 4.6.4
        cr_version: 4.6.4
        csv_version: 3.2.0
        status_shutdown_failed: shutdownError
        status_shutdown_success: shutdown
        sub_channel: v3.2
        support_shutdown: true
    dmc:
        case_version: 5.2.0
        cr_version: 4.6.4
        csv_version: 2.2.0
        status_shutdown_failed: shutdownError
        status_shutdown_success: shutdown
        sub_channel: v2.2
        support_shutdown: true
    dods:
        case_version: 6.5.0
        cr_version: 6.5.0
        csv_version: 6.5.0
        status_shutdown_failed: shutdownError
        status_shutdown_success: shutdown
        sub_channel: v6.5
        support_shutdown: true
        supported_arch: amd64
    dp:
        case_version: 6.5.0
        cr_version: 4.6.5
        csv_version: 6.5.0
        sub_channel: v6.5
        supported_arch: amd64
    dpra:
        case_version: 1.7.0
        cr_version: 1.7.0
        csv_version: 1.7.0
        sub_channel: v1.7
    dv:
        case_version: 2.4.0
        cr_version: 2.0.4
        csv_version: 2.4.0
        status_field: dvStatus
        status_shutdown_failed: shutdownError
        status_shutdown_success: shutdown
        sub_channel: v2.4
        support_shutdown: true
        supported_arch: amd64
    edb_cp4d:
        case_version: 4.13.0
        cr_version: 4.13.0
        csv_version: 4.13.0
        sub_channel: v4.13
    estap:
        case_version: 1.1.0
        cr_version: 1.1.0
        csv_version: 1.1.0
        sub_channel: v1.1
    factsheet:
        case_version: 1.5.0
        cr_version: 4.6.5
        csv_version: 1.5.0
        sub_channel: v1.5
    fdb_k8s:
        csv_version: 2.4.6
        sub_channel: v2.4
    hee:
        case_version: 4.6.5
        cr_version: 4.6.5
        csv_version: 3.5.0
        sub_channel: v3.5
        supported_arch: amd64,s390x
    iis:
        case_version: 4.6.5
        cr_version: 4.6.5
        csv_version: 1.6.5
        sub_channel: v3.5
        supported_arch: amd64
    informix:
        case_version: 5.3.0
        csv_version: 5.3.0
        sub_channel: v5.3
    informix_cp4d:
        case_version: 5.3.0
        cr_version: 5.3.0
        csv_version: 5.3.0
        sub_channel: v5.3
    mantaflow:
        case_version: 1.10.0
        cr_version: 39.1.12
        csv_version: 1.10.0
        sub_channel: v1.10
        supported_arch: amd64
    match360:
        case_version: 2.3.11
        component_dependencies:
        - ccs
        - opencontent_elasticsearch
        - opencontent_redis
        - opencontent_rabbitmq
        - opencontent_fdb
        - fdb_k8s
        cr_version: 2.3.29
        csv_version: 2.3.29
        sub_channel: v2.3
        supported_arch: amd64
    model_train:
        case_version: 1.2.6
        csv_version: 1.1.8
        sub_channel: v1.1
    mongodb:
        case_version: 4.12.0
        csv_version: 1.18.0
    mongodb_cp4d:
        case_version: 4.12.0
        cr_version: 4.12.0
        csv_version: 4.12.0
        sub_channel: v4.12
    opencontent_auditwebhook:
        case_version: 1.0.24
        csv_version: 0.3.1
    opencontent_elasticsearch:
        case_version: 1.1.1336
        cr_version: 1.1.1336
        csv_version: 1.1.1336
    opencontent_etcd:
        case_version: 2.0.24
        csv_version: 1.0.16
    opencontent_fdb:
        case_version: 2.4.5
        cr_version: 2.4.6
        csv_version: 2.4.6
        sub_channel: v2.4
    opencontent_minio:
        case_version: 1.0.22
        csv_version: 1.0.17
    opencontent_rabbitmq:
        case_version: 1.0.26
        csv_version: 1.0.18
    opencontent_redis:
        case_version: 1.6.6
        csv_version: 1.6.6
        multi_arch_images: cpopen/ibm-cloud-databases-redis-catalog,cpopen/ibm-cloud-databases-redis-operator
        sub_channel: v1.6
    openpages:
        case_save_args: --no-dependency
        case_version: 4.2.0
        component_dependencies:
        - db2u
        - db2aaservice
        - opencontent_rabbitmq
        cr_version: 8.302.1
        csv_version: 4.2.0
        sub_channel: v4.2
    openpages_instance:
        cr_version: 8.302.1
    openscale:
        case_version: 4.5.0
        cr_version: 4.6.5
        csv_version: 3.5.0
        sub_channel: v3.5
        supported_arch: amd64
    planning_analytics:
        case_version: 4.6.5
        cr_version: 4.6.5
        csv_version: 4.6.5
        sub_channel: v5.5
    postgresql:
        case_version: 4.13.0
        csv_version: 1.18.3
    productmaster:
        case_version: 3.3.0
        cr_version: 3.3.0
        csv_version: 3.3.0
        sub_channel: v3.3
    productmaster_instance:
        cr_version: 3.3.0
    replication:
        case_version: 4.6.6
        cr_version: 4.6.6
        csv_version: 1.6.0
        sub_channel: v1.6
        supported_arch: amd64
    rstudio:
        case_version: 6.5.0
        cr_version: 6.5.0
        csv_version: 6.5.0
        sub_channel: v6.5
        supported_arch: amd64
    scheduler:
        case_version: 1.12.0
        cr_version: 1.12.0
        csv_version: 1.12.0
        sub_channel: v1.12
        supported_arch: amd64,ppc64le
    spss:
        case_version: 6.5.0
        cr_version: 6.5.0
        csv_version: 6.5.0
        sub_channel: v6.5
        supported_arch: amd64
    voice_gateway:
        case_version: 1.3.1
        csv_version: 1.3.1
        sub_channel: v1.3
    watson_assistant:
        case_version: 4.15.0
        component_dependencies:
        - postgresql
        - opencontent_minio
        - opencontent_etcd
        - watson_gateway
        - data_governor
        - model_train
        - opencontent_elasticsearch
        - opencontent_redis
        - opencontent_rabbitmq
        cr_version: 4.6.5
        csv_version: 4.15.0
        sub_channel: v4.15
    watson_discovery:
        case_version: 5.5.0
        cr_version: 4.6.5
        csv_version: 5.5.0
        sub_channel: v5.5
        supported_arch: amd64
    watson_gateway:
        case_version: 2.0.23
        csv_version: 1.0.17
        support_shutdown: true
    watson_ks:
        case_version: 4.9.0
        cr_version: 4.9.0
        csv_version: 4.9.0
        sub_channel: v4.9
        supported_arch: amd64
    watson_speech:
        case_version: 5.5.0
        component_dependencies:
        - postgresql
        - opencontent_minio
        - opencontent_rabbitmq
        - watson_gateway
        cr_version: 4.6.5
        csv_version: 5.5.0
        sub_channel: v5.5
    wkc:
        case_version: 4.6.5
        component_dependencies:
        - ccs
        - analyticsengine
        - db2u
        - db2aaservice
        - opencontent_fdb
        - fdb_k8s
        - datastage_ent
        - datarefinery
        - iis
        - mantaflow
        cr_version: 4.6.5
        csv_version: 1.6.5
        sub_channel: v3.5
        supported_arch: amd64
    wml:
        case_version: 6.5.0
        cr_version: 4.6.5
        csv_version: 3.5.0
        sub_channel: v3.5
        supported_arch: amd64
    wml_accelerator:
        case_version: 3.5.0
        cr_version: 3.5.0
        csv_version: 3.5.0
        sub_channel: v3.5
        supported_arch: amd64,ppc64le
    wml_accelerator_instance:
        cr_version: 3.5.0
        supported_arch: amd64,ppc64le
    ws:
        case_version: 6.5.0
        cr_version: 6.5.0
        csv_version: 6.5.0
        sub_channel: v6.5
        supported_arch: amd64
    ws_pipelines:
        case_version: 6.4.0
        cr_version: 4.6.4
        csv_version: 6.4.0
        sub_channel: v6.4
        supported_arch: amd64
    ws_runtimes:
        case_version: 6.5.0
        cr_version: 6.5.0
        csv_version: 6.5.0
        sub_channel: v6.5
        supported_arch: amd64
    zen:
        cr_version: 4.8.4
        csv_version: 1.8.4

EOF

#  applying CR for Cloud Pak for Data Control Plane
cat <<EOF |oc apply -f -
apiVersion: cpd.ibm.com/v1
kind:  Ibmcpd
metadata:
  name: ibmcpd-cr
  namespace: cpd
spec:
  license:
    accept: true
    license: Enterprise
  fileStorageClass: ocs-storagecluster-cephfs
  blockStorageClass: ocs-storagecluster-ceph-rbd
  version: 4.6.6

EOF

# post- apply-cr release patching (if any) for cpd_platform
release-patches.sh post_apply_cr

