# NGINX

> :information_source: Commands below are valid for a **nginx** running on a **Centos 7**.

```
sudo yum install wget -y
sudo wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo rpm -Uvh epel-release-latest-7.noarch.rpm
sudo yum repolist

sudo yum install -y nginx

sudo vi /etc/nginx/nginx.conf

...
        listen       81;
        # listen       [::]:81;
        server_name  _;
        root         /web;

        location / {
                autoindex on;
                autoindex_exact_size off;
                autoindex_localtime on;
        }
...


[ ! -d /web ] && { sudo mkdir /web; sudo chmod 777 /web; } || echo "/web already exists."

touch /web/a

sudo systemctl start nginx &&
sudo systemctl enable nginx && systemctl status nginx

wget -c http://web.ocp:81/a

```



<!--

yes y | cp -rf -v ~/.ssh/* install-config.yaml /web

INSTALLER_FILE="openshift-install-linux.tar.gz"
CLIENT_FILE="openshift-client-linux.tar.gz"

INSTALLER_FILE="openshift-install-linux.tar.gz"
wget -c https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.8/openshift-install-linux.tar.gz
tar xvzf $INSTALLER_FILE
./openshift-install version

CLIENT_FILE="openshift-client-linux.tar.gz"
[ ! -z $(command -v oc) ] && { rm -f $(command -v oc); rm -f $(command -v kubectl); }
wget -c https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz
tar -xvzf $CLIENT_FILE -C $(echo $PATH | awk -F":" 'NR==1 {print $1}')
[ ! -z $(command -v oc) ] && { echo "oc installed successfully"; oc version --client; } || echo "ERROR: oc not found in PATH"

yes y | cp -v *.ign /web

-->
