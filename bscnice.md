it 's a one time password to download both VPN client (1) and VPN configuration file (2):

 

![](img/bscnice-vpn.png)

 

 

@all

 

Once connexion is established:

 

Add following lines to your hosts file (e.g. /etc/hosts or C:\Windows\System32\Drivers\etc\hosts):
 ...
 172.16.187.110 console-openshift-console.apps.ocp11.iicparis.fr.ibm.com
 172.16.187.110 oauth-openshift.apps.ocp11.iicparis.fr.ibm.com
 172.16.187.110 cpe-cp4a.apps.ocp11.iicparis.fr.ibm.com
 172.16.187.110 graphql-cp4a.apps.ocp11.iicparis.fr.ibm.com
 172.16.187.110 navigator-cp4a.apps.ocp11.iicparis.fr.ibm.com
 172.16.187.110 ums.cp4a.apps.ocp11.iicparis.fr.ibm.com
 172.16.187.110 ums-scim.ums.cp4a.apps.ocp11.iicparis.fr.ibm.com
 172.16.187.110 ums-sso.ums.cp4a.apps.ocp11.iicparis.fr.ibm.com
 172.16.187.110 ums-teams.ums.cp4a.apps.ocp11.iicparis.fr.ibm.com
 172.16.187.110 backend.cp4a.apps.ocp11.iicparis.fr.ibm.com
 172.16.187.110 bas.cp4a.apps.ocp11.iicparis.fr.ibm.com
 172.16.187.110 navigator-cp4a.apps.ocp11.iicparis.fr.ibm.com
 172.16.187.110 frontend.cp4a.apps.ocp11.iicparis.fr.ibm.com

 

You can access ACCE and Navigator via the following URLs:
 https://cpe-cp4a.apps.ocp11.iicparis.fr.ibm.com/acce
 https://navigator-cp4a.apps.ocp11.iicparis.fr.ibm.com/navigator

User credentials:
 ================

ACCE usename: cp4admin
 ACCE user password: 3sFq6CCWmeqdNTIv9eam

Navigator usename: cp4admin
 Navigator user password: 3sFq6CCWmeqdNTIv9eam

 

You can access Automation Content Analyzer via the following URLs:
 https://backend.cp4a.apps.ocp11.iicparis.fr.ibm.com

https://frontend.cp4a.apps.ocp11.iicparis.fr.ibm.com/?tid=ont1&ont=ONT1
 User credentials:
 ================

Default administrator username: cp4admin
 Default administrator password: 3sFq6CCWmeqdNTIv9eam
 You can access Business Automation Studio via the following URLs:
 https://bas.cp4a.apps.ocp11.iicparis.fr.ibm.com/BAStudio

 

 

To start working with CP4A capabilities:

IBM Business Automation Content Analyzer documentation
 https://www.ibm.com/support/knowledgecenter/SSUM7G/com.ibm.bacanalyzertoc.doc/bacanalyzer_1.0.html

IBM Business Automation Navigator
 https://www.ibm.com/support/knowledgecenter/en/SSYHZ8_20.0.x/com.ibm.dba.offerings/topics/con_ban.html

IBM FileNet Content Manager
 https://www.ibm.com/support/knowledgecenter/en/SSYHZ8_20.0.x/com.ibm.dba.offerings/topics/con_cm.html

 

Access to Openshift web console:
 https://console-openshift-console.apps.ocp11.iicparis.fr.ibm.com/
 Login with htpasswd_provider as **admin** using **admin** as password

 Openshift command line access:

oc login https://172.16.187.110:6443 -u **admin** -p **admin** --insecure-skip-tls-verify=true

Install oc and kubectl if needed:
 Linux:
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
 Windows:
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-windows.zip
 MacOS:
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-mac.tar.gz