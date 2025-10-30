# DB2 TLS

## Creating a keystore

:information: Have to be run on DB2 server as ctginst1

```
PASSWORD="abc123"
INSTANCE="ctginst1"

gsk8capicmd_64 -keydb -create -db ${INSTANCE}.p12 -pw ${PASSWORD} -stash

```

## Creating a certificate signing request (CSR)

:information: Have to be run on DB2 server as ctginst1

```
INSTANCE="ctginst1"
SERVERNAME="DB2_SERVER_FQDN"

gsk8capicmd_64 -certreq -create -db ${INSTANCE}.p12 -stashed -label ${INSTANCE} -dn "CN=${SERVERNAME}" -size 2048 -sigalg SHA256_WITH_RSA -target ${INSTANCE}.csr

gsk8capicmd_64 -certreq -list -db ${INSTANCE}.p12 -stashed
```

> Send ${INSTANCE}.csr to Certificate Authority (CA)

## CA side


> CA Will provide rootCA.cer, (optional) intermdiateCA.cr and generate server.cer 

## Adding the root, intermediate and server certificates

:information: Have to be run on DB2 server as ctginst1


```
INSTANCE="ctginst1"
CA_CORP="Corp"

gsk8capicmd_64 -cert -add -db ${INSTANCE}.p12 -stashed -file root.cer -label ${CA_CORP}$RootCA
gsk8capicmd_64 -cert -add -db ${INSTANCE}.p12 -stashed -file intermediate.cer -label ${CA_CORP}IntermediateCA
gsk8capicmd_64 -cert -receive -db ${INSTANCE}.p12 -stashed -file ${INSTANCE}.cer

gsk8capicmd_64 -cert -list -db ${INSTANCE}.p12 -stashed
```

## Configuring TLS support

> :information: Have to be run on DB2 server as ctginst1

> :bulb: If db2level >= v11.5.8 then run db2 update dbm cfg using SSL_VERSIONS TLSV12,TLSV13


```
INSTANCE="ctginst1"

db2 update dbm cfg using SSL_SVR_KEYDB ${HOME}/${INSTANCE}.p12
db2 update dbm cfg using SSL_SVR_STASH ${HOME}/${INSTANCE}.sth

db2 update dbm cfg using SSL_SVR_LABEL ${INSTANCE}
db2 update dbm cfg using SSL_SVCENAME 50001
db2 update dbm cfg using SSL_VERSIONS TLSV12

db2set -i db2inst1 DB2COMM=SSL,TCPIP

db2stop force && db2start 

ss -pantu | grep 5000
```

> :bulb: ss command should list listening on both **50000** and **500001**

## JDBC Connection url

:information: Have to be set on client side (e.g. MAS)

```
SERVERNAME="DB2_SERVER_FQDN"
DB="MXDB"
PASSWORD="abc123"
INSTANCE="ctginst1"
P12_FILE="//opt//ibm//mas/certs//${INSTANCE}.p12"

echo jdbc:db2://${SERVERNAME}:50001/${DB}:sslConnection=true;sslTrustStoreLocation=${P12_FILE};sslTrustStorePassword=${PASSWORD};verifyServerCertificate=false;useSSL=true;requireSSL=true;sslVersion=TLSv1.2;"
```
