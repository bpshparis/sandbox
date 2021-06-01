

Mise à disposition de la sandbox IBM Cloud Pak for Data

Bonjour,

voici les identifiants:

User				Paswword



utiles une seule fois pour récupérer utiles une seule fois pour récupérer (voir pièce jointe pour info détaillée) le client VPN et le fichier de configuration ici:

https://bscvpn.nca.ihost.com



![](img/bscnice-vpn.png)

Une fois la connexion établie, ajouter les lignes suivantes à votre fichier hosts

e.g. /etc/hosts ou C:\Windows\ System32\drivers\etc\hosts

172.16.187.50 console-openshift-console.apps.ocp5.iicparis.fr.ibm.com

172.16.187.50 oauth-openshift.apps.ocp5.iicparis.fr.ibm.com

172.16.187.50 cpd-cpd-cpd.apps.ocp5.iicparis.fr.ibm.com



Accès à la Console Cloud Pak for Data en tant que user **admin** mot de passe **password**:
https://cpd-cpd-cpd.apps.ocp3.iicparis.fr.ibm.com



Accès à la Console Openshift en tant que user **admin** mot de passe **admin**:
https://console-openshift-console.apps.ocp3.iicparis.fr.ibm.com



Accès au cluster via la commande oc:
oc login https://172.16.187.50:6443 -u admin -p admin --insecure-skip-tls-verify=true -n cpd

Installer les commandes oc and kubectl si necessaire:
Linux:
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
Windows:
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-windows.zip
MacOS:
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-mac.tar.gz



Enjoy !