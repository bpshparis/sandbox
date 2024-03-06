Hi all

your sandbox with followings assemblies:

\- Watson Discovery
\- Watson Studio

\- Watson Knowledge Studio

is now available.

Please find below a one time user/password for each of you 

User:password
sindelicato:E/9XO2l0
rbarbero:wWfUqUn4
lfalco:OJXjGSqZ
fmalosio:S+pahdHM



to download from here:

https://bscvpn.nca.ihost.com

both VPN client (1) and VPN configuration file (2)

![](img/bscnice-vpn.png)



Download and connect with OpenVPN client

OpenVPN Connect for Windows
https://openvpn.net/downloads/openvpn-connect-v3-windows.msi

OpenVPN Connect for macOS
https://openvpn.net/downloads/openvpn-connect-v3-macos.dmg

Once connexion is established:

Add following lines to your hosts file (e.g. **/etc/hosts** or **C:\Windows\System32\Drivers\etc\hosts**):
```
172.16.187.140 console-openshift-console.apps.ocp14.iicparis.fr.ibm.com  
172.16.187.140 oauth-openshift.apps.ocp14.iicparis.fr.ibm.com
172.16.187.140 cpd-cpd-instance.apps.ocp14.iicparis.fr.ibm.com
172.16.187.140 cli.ocp14.iicparis.fr.ibm.com
```
Access Cloud Pak for Data console as user **admin** using **admin** as password: 
https://cpd-cpd-instance.apps.ocp14.iicparis.fr.ibm.com

Access Openshift console  logging with **httppasswd_provider** (not kebube:admin) as user **admin** using **admin** as password: 
https://console-openshift-console.apps.ocp14.iicparis.fr.ibm.com

Access Openshift cluster via command line:
```
oc login https://cli.ocp14:6443 -u admin -p admin --insecure-skip-tls-verify=true
```

Install oc and kubectl command if necessary: 

Linux: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz 
Windows: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-windows.zip 
MacOS: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-mac.tar.gz

**TROUBLESHOOTING**

if hosts are not resolved feel free to add line like this:

${SERVICE_NAME}-${PROJECT}.apps.ocp14.iicparis.fr.ibm.com 

e.g. :

```
172.16.187.140 myservice-myproject.apps.ocp14.iicparis.fr.ibm.com
```

to your hosts file (e.g. /etc/hosts or C:\Windows\System32\Drivers\etc\hosts)