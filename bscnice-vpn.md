

Mise à disposition de votre sandbox IBM Cloud Pak for Data

Bonjour,

voici vos identifiants:

User				Paswword

sfaye 		hRdvf85K
nmchela 		CFY5JhZe
skajie 		uXVfJ7h+
ehaback 		38T5u4Et

utiles une seule fois pour récupérer le client VPN et le fichier de configuration ici:

https://bscvpn.nca.ihost.com



![](img/bscnice-vpn.png)

Une fois la connexion établie, ajouter les lignes suivantes à votre fichier hosts

e.g. /etc/hosts ou C:\Windows\ System32\drivers\etc\hosts

172.16.187.90 console-openshift-console.apps.ocp9.iicparis.fr.ibm.com

172.16.187.90 oauth-openshift.apps.ocp9.iicparis.fr.ibm.com

172.16.187.90 cpd-cpd-cpd.apps.ocp9.iicparis.fr.ibm.com



Accès à la Console Cloud Pak for Data en tant que user **admin** mot de passe **password**:
https://cpd-cpd-cpd.apps.ocp9.iicparis.fr.ibm.com



Accès à DataStage:

https://cpd-cpd-cpd.apps.ocp9.iicparis.fr.ibm.com/zen/#/projectList



Accès à Db2 si besoin:

https://cpd-cpd-cpd.apps.ocp9.iicparis.fr.ibm.com/zen-databases/#/details/Db2-1/db2oltp-1620200333138185

jdbc:db2://db2inst1:y9D@5vZJ5#%_i5Ja@w1-ocp9.iicparis.fr.ibm.com:30041/BLUDB



Accès à la Console Openshift en tant que user **admin** mot de passe **admin**:
https://console-openshift-console.apps.ocp9.iicparis.fr.ibm.com

Accès au cluster via la commande oc:
oc login https://172.16.187.90:6443 -u admin -p admin --insecure-skip-tls-verify=true
Install oc and kubectl if needed:
Linux:
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
Windows:
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-windows.zip
MacOS:
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-mac.tar.gz

Enjoy !