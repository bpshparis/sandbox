# DNS

:information_source: Commands below are valid for a **bind** running on a **Centos** / **RHEL**.

## Remove existing DNS

```
sudo systemctl stop named
sudo yum remove bind bind-utils -y
```
## Set environment variables

```
DOMAIN="local" && echo $DOMAIN
CLUSTER="ocp" && echo $CLUSTER
DNS_FORWARDER="192.168.1.1" && $DNS_FORWARDER
NUM="1" && echo $NUM
DNS="192.168.1.${NUM}0" && echo $DNS
IP_HEAD="192.168.1" && echo $IP_HEAD
IP_HEAD_REV=$(echo $IP_HEAD | awk -F. '{print $3 "." $2 "." $1}') && echo $IP_HEAD_REV
HOST="lb.$CLUSTER" && echo $HOST
MZONE="/etc/named/zones/$DOMAIN.hosts" && echo $MZONE
RZONE="/etc/named/zones/$IP_HEAD.rev" && echo $RZONE

LB_IP=$IP_HEAD.${NUM}0 && echo $LB_IP
M1_IP=$IP_HEAD.${NUM}1 && echo $M1_IP
M2_IP=$IP_HEAD.${NUM}2 && echo $M2_IP
M3_IP=$IP_HEAD.${NUM}3 && echo $M3_IP
W1_IP=$IP_HEAD.${NUM}4 && echo $W1_IP
W2_IP=$IP_HEAD.${NUM}5 && echo $W2_IP
W3_IP=$IP_HEAD.${NUM}6 && echo $W3_IP
W4_IP=$IP_HEAD.${NUM}7 && echo $W4_IP
W5_IP=$IP_HEAD.${NUM}8 && echo $W5_IP
BS_IP=$IP_HEAD.${NUM}9 && echo $BS_IP
```

## Disable NetworkManager service

```
sudo systemctl stop NetworkManager &&
sudo systemctl disable NetworkManager && sudo systemctl status NetworkManager
```
## Disable security

```
sudo systemctl stop firewalld &&
sudo systemctl disable firewalld && sudo systemctl status firewalld
sudo setenforce 0
sudo sed -i -e 's/^SELINUX=\w*/SELINUX=disabled/' /etc/selinux/config
```

## Install DNS

```
sudo yum install bind bind-utils -y
```

## Configure DNS

```
sudo sed -i -E 's/127\.0\.0\.1;|localhost;|::1;/any;/g' /etc/named.conf
sudo sed -i 's/dnssec-validation.*$/dnssec-validation no;/' /etc/named.conf
sudo sed -i 's!\(dnssec-enable*\)!// \1!g' /etc/named.conf
```
## Add Forwarder DNS

```
cat > forwarders << EOF

        forwarders {
                ${DNS_FORWARDER};
        };  
EOF

sudo sed -i -e '/session-keyfile/r forwarders' /etc/named.conf

echo 'include "/etc/named/named.conf.local";' | sudo tee -a /etc/named.conf
```

## Add zones files to DNS configuration

```
cat << EOF | sudo tee /etc/named/named.conf.local
zone "$DOMAIN" {
    type master;
    file "$MZONE";
};  
zone "$IP_HEAD_REV.in-addr.arpa" {
    type master;
    file "$RZONE";
};
EOF
```

## Create zones directory

```
sudo chmod 755 /etc/named
[ ! -d /etc/named/zones ] && sudo mkdir /etc/named/zones || echo "/etc/named/zones already exists."
```

## Add Master zone file

```
cat << EOF | sudo tee $MZONE
@   IN  SOA     $HOST.$DOMAIN. root.$DOMAIN. (
                                                1001    ;Serial
                                                3H      ;Refresh
                                                15M     ;Retry
                                                1W      ;Expire
                                                1D      ;Minimum TTL
                                                )

;Name Server Information
@      IN  NS      $HOST.$DOMAIN.

;IP address of Name Server
$HOST IN  A       $DNS
EOF
```

## Add Reverse zone file

```
cat << EOF | sudo tee $RZONE
@   IN  SOA     $HOST.$DOMAIN. root.$DOMAIN. (
                                                1001    ;Serial
                                                3H      ;Refresh
                                                15M     ;Retry
                                                1W      ;Expire
                                                1D      ;Minimum TTL
                                                )

;Name Server Information
@      IN  NS      $HOST.$DOMAIN.

;Reverse lookup for Name Server
$(echo $DNS | awk -F. '{print $4}')        IN  PTR     $HOST.$DOMAIN.
EOF
```

## Update hostname

```
sudo hostnamectl set-hostname $HOST.$DOMAIN && hostnamectl
```

## Update DNS and DNS suffix

```
cat << EOF | sudo tee /etc/resolv.conf
search $DOMAIN
nameserver $DNS
EOF
```

:bulb: Just in case, check /etc/sysconfig/network-scripts/ifcfg-<IF>

## Restart DNS

```
sudo systemctl restart named &&
sudo systemctl enable named && sudo systemctl status named
```

## Check DNS is resoving himself

```
dig @$DNS +short +search $HOST.$DOMAIN
dig @$DNS +short -x $LB_IP
ping -c 2 yahoo.fr
```

## Add records to master zone

```
cat << EOF | sudo tee -a $MZONE
web.$CLUSTER  IN      A       $LB_IP
m1.$CLUSTER   IN      A       $M1_IP
m2.$CLUSTER   IN      A       $M2_IP
m3.$CLUSTER   IN      A       $M3_IP
w1.$CLUSTER   IN      A       $W1_IP
w2.$CLUSTER   IN      A       $W2_IP
w3.$CLUSTER   IN      A       $W3_IP
w4.$CLUSTER   IN      A       $W4_IP
w5.$CLUSTER   IN      A       $W5_IP
bs.$CLUSTER   IN      A       $BS_IP
api.$CLUSTER  IN      A       $LB_IP
api-int.$CLUSTER      IN      A       $LB_IP
apps.$CLUSTER IN      A       $LB_IP
*.apps.$CLUSTER       IN      CNAME   apps.$CLUSTER.$DOMAIN.
etcd-0.$CLUSTER       IN      A       $M1_IP
etcd-1.$CLUSTER       IN      A       $M2_IP
etcd-2.$CLUSTER       IN      A       $M3_IP
_etcd-server-ssl._tcp.$CLUSTER        86400   IN      SRV     0 10 2380 etcd-0.$CLUSTER
_etcd-server-ssl._tcp.$CLUSTER        86400   IN      SRV     0 10 2380 etcd-1.$CLUSTER
_etcd-server-ssl._tcp.$CLUSTER        86400   IN      SRV     0 10 2380 etcd-2.$CLUSTER
EOF
```

## Add records to reverse zone

```
cat << EOF | sudo tee -a $RZONE
$(echo $LB_IP | awk -F. '{print $4}')        IN  PTR     web.$CLUSTER.$DOMAIN.
$(echo $M1_IP | awk -F. '{print $4}')    IN      PTR     m1.$CLUSTER.$DOMAIN.
$(echo $M2_IP | awk -F. '{print $4}')    IN      PTR     m2.$CLUSTER.$DOMAIN.
$(echo $M3_IP | awk -F. '{print $4}')    IN      PTR     m3.$CLUSTER.$DOMAIN.
$(echo $W1_IP | awk -F. '{print $4}')    IN      PTR     w1.$CLUSTER.$DOMAIN.
$(echo $W2_IP | awk -F. '{print $4}')    IN      PTR     w2.$CLUSTER.$DOMAIN.
$(echo $W3_IP | awk -F. '{print $4}')    IN      PTR     w3.$CLUSTER.$DOMAIN.
$(echo $W4_IP | awk -F. '{print $4}')    IN      PTR     w4.$CLUSTER.$DOMAIN.
$(echo $W5_IP | awk -F. '{print $4}')    IN      PTR     w5.$CLUSTER.$DOMAIN.
$(echo $BS_IP | awk -F. '{print $4}')    IN      PTR     bs.$CLUSTER.$DOMAIN.
EOF
```

## Restart DNS

```
sudo systemctl restart named
```

## Test master zone

```
for host in lb web m1 m2 m3 w1 w2 w3 w4 w5 bs; do echo -n $host.$CLUSTER "-> "; dig @$DNS +short +search $host.$CLUSTER.$DOMAIN; done
```

## Test reverse zone

```
for host in web m1 m2 m3 w1 w2 w3 w4 w5 bs; do IP=$(dig @$DNS +short +search $host.$CLUSTER.$DOMAIN); echo -n $IP "-> "; dig @$DNS +short +search -x $IP; done
```

## Test alias

```
dig @$DNS +short +search *.apps.$CLUSTER.$DOMAIN
```

## Test services

```
dig @$DNS +short +search _etcd-server-ssl._tcp.$CLUSTER.$DOMAIN SRV
```

:checkered_flag::checkered_flag::checkered_flag: