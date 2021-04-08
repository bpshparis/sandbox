# Access your private Openshift api server from internet

## Get Cluster CA from first master

> :bulb: The only way to connect to cluster nodes is from the computer which installed cluster in **ssh** using **core** user.

> :information_source: From cluster installer
```
OCP="ocp19"
scp core@m1-$OCP:/etc/kubernetes/ca.crt ~/ca-$OCP.crt
```

## Install oc command

Follow these [instructions](https://docs.openshift.com/container-platform/4.3/installing/installing_bare_metal/installing-bare-metal.html#cli-installing-cli_installing-bare-metal) to install  **oc** the Openshift CLI.

## Install ngrok

Follow these [instructions](https://dashboard.ngrok.com/get-started) to install and setup **ngrok** a command for an instant, secure URL to your localhost server through any NAT or firewall.

## Connect to cluster api server via public address

> :warning: You should have both **oc** and **ngrok** copied in your $PATH and the **crt** file copied somewhere on your file system.

> :information_source: From your computer
```
OCP="ocp19"
NGROK_AUTH_TOKEN="1ZDLKSEI6cJnjTN18p..."
CA_CERT="ca-$OCP.crt"

ngrok authtoken $NGROK_AUTH_TOKEN
ngrok http https://lb-$OCP.iicparis.fr.ibm.com:6443
```

> Output from ngrok ready for connection:
![](ngrok0.png)

> :bulb:  Leave terminal above opened and keep an eye on it.

> :information_source: From your computer
```
oc login -u admin -p admin https://4f76e0c6.ngrok.io --certificate-authority=$CA_CERT --insecure-skip-tls-verify=true
```

> Output from ngrok handling HTTP Requests:
![](ngrok1.png)


:checkered_flag::checkered_flag::checkered_flag:

You should now be able to read **Login successful.** in your last terminal session.
