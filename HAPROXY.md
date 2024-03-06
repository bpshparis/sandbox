# HAPROXY



```
DOMAIN=$(cat /etc/resolv.conf | awk '$1 ~ "^search" {print $2}') && echo $DOMAIN
CLUSTER="ocp" && echo $CLUSTER
LB_CONF="/etc/haproxy/haproxy.cfg" && echo $LB_CONF
[ -f "$LB_CONF" ] && echo "haproxy already installed" || sudo yum install haproxy -y

sudo sed -i '/^\s\{1,\}maxconn\s\{1,\}3000$/q' $LB_CONF

cat << EOF | sudo tee -a $LB_CONF

listen stats
    bind :9000
    mode http
    stats enable
    stats uri /

frontend ingress-http
    bind *:80
    default_backend ingress-http
    mode tcp
    option tcplog

backend ingress-http
    balance source
    mode tcp
    server w1.$CLUSTER $(dig +short +search w1.$CLUSTER):80 check
    server w2.$CLUSTER $(dig +short +search w2.$CLUSTER):80 check

frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

backend ingress-https
    balance source
    mode tcp
    server w1.$CLUSTER $(dig +short +search w1.$CLUSTER):443 check
    server w2.$CLUSTER $(dig +short +search w2.$CLUSTER):443 check

frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    mode tcp
    option tcplog

backend openshift-api-server
    balance source
    mode tcp
    server m1.$CLUSTER $(dig +short +search m1.$CLUSTER):6443 check
    server m2.$CLUSTER $(dig +short +search m2.$CLUSTER):6443 check
    server m3.$CLUSTER $(dig +short +search m3.$CLUSTER):6443 check
    server bs.$CLUSTER $(dig +short +search bs.$CLUSTER):6443 check

frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    mode tcp
    option tcplog

backend machine-config-server
    balance source
    mode tcp
    server m1.$CLUSTER $(dig +short +search m1.$CLUSTER):22623 check
    server m2.$CLUSTER $(dig +short +search m2.$CLUSTER):22623 check
    server m3.$CLUSTER $(dig +short +search m3.$CLUSTER):22623 check
    server bs.$CLUSTER $(dig +short +search bs.$CLUSTER):22623 check

EOF

sudo systemctl restart haproxy &&
sudo systemctl enable haproxy && sudo systemctl status haproxy
curl -I http://lb.$CLUSTER:9000



```

