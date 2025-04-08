# DNS

## Remove existing DNS

```
sudo systemctl stop named
sudo yum remove bind bind-utils -y
```


## Set environment variables

```
DOMAIN="baudelaine.com" && echo $DOMAIN
CLUSTER="ocp" && echo $CLUSTER
DNS_FORWARDER="192.168.1.254" && echo $DNS_FORWARDER
IP_HEAD="192.168.1" && echo $IP_HEAD
IP_HEAD_REV=$(echo $IP_HEAD | awk -F. '{print $3 "." $2 "." $1}') && echo $IP_HEAD_REV
NUM="7" && echo $NUM
MZONE="/etc/named/zones/$DOMAIN.hosts" && echo $MZONE
RZONE="/etc/named/zones/$IP_HEAD.rev" && echo $RZONE
DNS_HOST="bastion" && echo $DNS_HOST
DNS_IP="$IP_HEAD.${NUM}1" && echo $DNS_IP
SNO_NAME="sno" && echo $SNO_NAME
SNO_IP="$IP_HEAD.${NUM}2" && echo $SNO_IP


tee ${HOME}/dns-vars.sh << EOF
DOMAIN="${DOMAIN}"
CLUSTER="${CLUSTER}"
DNS_FORWARDER="${DNS_FORWARDER}"
IP_HEAD="${IP_HEAD}"
IP_HEAD_REV="${IP_HEAD_REV}"
NUM="${NUM}"
MZONE="${MZONE}"
RZONE="${RZONE}"
DNS_HOST="${DNS_HOST}"
DNS_IP="${DNS_IP}"
SNO_NAME="${SNO_NAME}"
SNO_IP="${SNO_IP}"
EOF

source ${HOME}/dns-vars.sh
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

echo 'include "/etc/named/named.conf.'${DOMAIN}'";' | sudo tee -a /etc/named.conf
```

## Add zones files to DNS configuration

```
cat << EOF | sudo tee /etc/named/named.conf.${DOMAIN}
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
[ ! -d /etc/named/zones ] && sudo mkdir /etc/named/zones || echo "/etc/named/zones already exists."
sudo chmod -R 755 /etc/named
```

## Add Master zone file

```
cat << EOF | sudo tee $MZONE
@   IN  SOA     $DNS_HOST.$DOMAIN. root.$DOMAIN. (
                                                1001    ;Serial
                                                3H      ;Refresh
                                                15M     ;Retry
                                                1W      ;Expire
                                                1D      ;Minimum TTL
                                                )

;Name Server Information
@      IN  NS      $DNS_HOST.$DOMAIN.

;IP address of Name Server
$DNS_HOST IN  A       $DNS_IP
EOF
```

## Add Reverse zone file

```
cat << EOF | sudo tee $RZONE
@   IN  SOA     $DNS_HOST.$DOMAIN. root.$DOMAIN. (
                                                1001    ;Serial
                                                3H      ;Refresh
                                                15M     ;Retry
                                                1W      ;Expire
                                                1D      ;Minimum TTL
                                                )

;Name Server Information
@      IN  NS      $DNS_HOST.$DOMAIN.

;Reverse lookup for Name Server
$(echo $DNS_IP | awk -F. '{print $4}')        IN  PTR     $DNS_HOST.$DOMAIN.
EOF
```


## Update hostname

```
sudo hostnamectl set-hostname $DNS_HOST.$DOMAIN && hostnamectl
```

## Update DNS and DNS suffix
```
IF=$(sudo nmcli -g name,type connection  show  --active | awk -F":" '{print $1}') && echo $IF

IP="${DNS_IP}"
MASK="24"
GW="192.168.1.254"

sudo nmcli con mod ${IF} ipv4.addresse ${IP}/${MASK}
sudo nmcli con mod ${IF} ipv4.gateway ${GW}
sudo nmcli con mod ${IF} ipv4.method manual

sudo nmcli con mod ${IF} ipv6.ignore-auto-dns yes
sudo nmcli con mod ${IF} ipv4.ignore-auto-dns yes
sudo nmcli con mod ${IF} ipv4.dns "${DNS_IP}"
sudo nmcli con mod ${IF} ipv4.dns-search "${DOMAIN}"

sudo systemctl restart NetworkManager
```

:bulb: remove old ip with **ip addr delete ${IP}/${MASK} dev ${IF}** if necessary

## Restart DNS

```
sudo systemctl restart named &&
sudo systemctl enable named && sudo systemctl status named
```

## Check DNS is resolving himself

```
dig @$DNS_IP +short +search $DNS_HOST.$DOMAIN
dig @$DNS_IP +short -x $DNS_IP
ping -c 2 yahoo.fr
```

## Add records to master zone

```
cat << EOF | sudo tee -a $MZONE
${SNO_NAME}.$CLUSTER   IN      A       $SNO_IP
api.$CLUSTER  IN      A       $SNO_IP
api-int.$CLUSTER      IN      A       $SNO_IP
apps.$CLUSTER IN      A       $SNO_IP
*.apps.$CLUSTER       IN      CNAME   apps.$CLUSTER.$DOMAIN.
EOF
```

## Add records to reverse zone

```
cat << EOF | sudo tee -a $RZONE
$(echo $SNO_IP | awk -F. '{print $4}')        IN  PTR     ${SNO_NAME}.$CLUSTER.$DOMAIN.
EOF
```

## Restart DNS

```
sudo systemctl restart named
```

## Test new entries in DNS

```
HOSTS="${SNO_NAME}"

for host in ${HOSTS}; do echo -n $host.$CLUSTER "-> "; dig @$DNS_IP +short +search $host.$CLUSTER.$DOMAIN; done

for host in ${HOSTS}; do IP=$(dig @$DNS_IP +short +search $host.$CLUSTER.$DOMAIN); echo -n $IP "-> "; dig @$DNS_IP +short +search -x $IP; done

dig @$DNS_IP +short +search api-int.$CLUSTER.$DOMAIN
dig @$DNS_IP +short +search api.$CLUSTER.$DOMAIN
dig @$DNS_IP +short +search *.apps.$CLUSTER.$DOMAIN
```

🏁🏁🏁
