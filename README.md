# Kubernetes Cluster on-premise with kubeadm using Ubuntu 16.04 & Vagrant
---
**#Important:**
Any issues you face during the setup, please refer to the Challenges Faced section at the end of this document.

#### Update and install docker (On both master and slave Nodes)
```sh
$ sudo apt-get update 
$ sudo apt-get install -qy docker.io
```
#### Install kubernetes

- Install apt repo
```sh
$ sudo apt-get install -y apt-transport-https
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

* Update sources.list with kubenetes repo
```sh
$ echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
$ sudo apt-get update
```

* Update and Install kubelet, kubeadm and kubernetes-cni
```sh
$ sudo apt-get install -y kubelet kubeadm kubernetes-cni
```
# On the master node

#### Initialize the cluster with kubeadm

* **\-\-apiserver-advertise-address** will be the ip of your host
* **stable-<version>** depends on the version of kubelet version, this should be equal to the version on kubelet. To check type command ```kubelet --version``. e.g. stable-1.11

```sh
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.33.10 --kubernetes-version stable-<version>
```
**Note:-**
Save the join token in the slave node that you will get after running the above command

#### Next commands we will be running as regular user

* Create a new unpreviliged user-account in master node

```sh
$ sudo useradd arun -G sudo -m -s /bin/bash
$ sudo passwd arun
```
Switch into the new user account with: ```sudo su arun``` and run below commands:-

```sh
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
* Create environment variable and export changes to ```~/.bashrc```

```sh
$ export KUBECONFIG=$HOME/.kube/config
$ echo "export KUBECONFIG=$HOME/.kube/config" | tee -a ~/.bashrc
```

#### For our demo we will be using flannel for networking between pods

**Note:-**
1. There are many other third party pod network provider such as:- Calico, Canal, Kube-router, Romana, Weave Net.
2. You can install only one pod network per cluster

- Set **/proc/sys/net/bridge/bridge-nf-call-iptables to 1** by running ```sysctl net.bridge.bridge-nf-call-iptables=1``` to pass bridged IPv4 traffic to iptables’ chains. This is a requirement for some CNI plugins to work,

**Important**
- If you are using virtual machines provisioned through vagrant in that case you have to refer topic ```Default NIC When using flannel as the pod network in Vagrant```under **Known Issues** in the end.

```sh
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
```

#### Master Isolation (optional)

By default, your cluster will not schedule pods on the master for security reasons. If you want to be able to schedule pods on the master, e.g. for a single-machine Kubernetes cluster for development, run:

```sh
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

**Note:-**

If you run above command, you need to make sure this command works fine ```$ kubectl get all --namespace=kube-system```

#### Output:-

```sh
$ kubectl get all --namespace=kube-system
NAME                                               READY     STATUS    RESTARTS   AGE
pod/etcd-master-ubuntu-xenial                      1/1       Running   0          3h
pod/kube-apiserver-master-ubuntu-xenial            1/1       Running   0          3h
pod/kube-controller-manager-master-ubuntu-xenial   1/1       Running   0          3h
pod/kube-dns-86f4d74b45-72plh                      3/3       Running   0          3h
pod/kube-flannel-ds-569h6                          1/1       Running   0          37m
pod/kube-flannel-ds-bbr7j                          1/1       Running   0          31m
pod/kube-proxy-66rcq                               1/1       Running   0          3h
pod/kube-proxy-p4g2g                               1/1       Running   0          31m
pod/kube-scheduler-master-ubuntu-xenial            1/1       Running   0          3h

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   3h

NAME                             DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR                   AGE
daemonset.apps/kube-flannel-ds   2         2         2         2            2           beta.kubernetes.io/arch=amd64   37m
daemonset.apps/kube-proxy        2         2         2         2            2           <none>                          3h

NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kube-dns   1         1         1            1           3h

NAME                                  DESIRED   CURRENT   READY     AGE
replicaset.apps/kube-dns-86f4d74b45   1         1         1         3h
```

# On Slave node

The slave nodes are where your workloads (containers and pods, etc) run. To add new nodes to your cluster do the following for each machine:

- As a root user run the below command that was output by kubeadm init. For example:

```sh
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

```sh
$ kubeadm join 192.168.33.10:6443 --token 06gp1e.axvrayn76l44m3q7 --discovery-token-ca-cert-hash sha256:68c603560e11a2dac4b60172112901600729766a10ae356c2de38b8e89cff716
```

Now if you run command ```$ kubectl get nodes``` in Master node you would be able to see the slave node in the cluster along with master node.

# On both master and slave nodes

Run the bridge command to make sure the funnel network is setup properly and node and master are able to communicate using that bridge through host-only netowrk interface.

#### On Master node

```sh
$ bridge fdb show
...
ae:4f:e8:3b:5c:0d dev flannel.1 dst 192.168.33.30 self permanent
...
```

#### On Slave node

```sh
$ bridge fdb show
...
0e:6f:05:45:00:37 dev flannel.1 dst 192.168.33.20 self permanent
...
```

**Note:-** 
If there are on same network then only proceed with running a microservice

# Running a microservice (Switch to master node)

To check already running pods (containers)

```sh
$ kubectl get pods
```
Deploy a container

```sh
$ kubectl run jenkins --image=jenkins/jenkins:alpine --port=8080
deployment.apps "jenkins" created
```
Commands to check pods

```sh
$ kubectl get pods
NAME                       READY     STATUS              RESTARTS   AGE
jenkins-8588bd6d9c-8w86p   0/1       ContainerCreating   0          1m
```
Describe Pod

```sh
$ kubectl describe pod jenkins-8588bd6d9c-8w86p
Name:           jenkins-8588bd6d9c-8w86p
Namespace:      default
Node:           slave-ubuntu-xenial/10.0.2.15
Start Time:     Wed, 09 May 2018 06:57:33 +0000
Labels:         pod-template-hash=4144682857
                run=jenkinn
                ....
```

Get the IP of the pod in order the check the connectivity, make sure you get the response after the curl command

```sh
$ kubectl describe po jenkins-8588bd6d9c-8w86p | grep IP
IP:             10.244.1.50

$ curl 10.244.1.50:8080
<html><head><meta http-equiv='refresh' content='1;url=/login?from=%2F'/><script>window.location.replace('/login?from=%2F');</script></head><body style='background-color:white; color:white;'>

Authentication required
<!--
You are authenticated as: anonymous
Groups that you are in:

Permission you need to have (but didn't): hudson.model.Hudson.Administer
-->

</body></html>
```
In order to see the logs from the Pod

```sh
$ kubectl logs jenkins-8588bd6d9c-8w86p

Running from: /usr/share/jenkins/jenkins.war
webroot: EnvVars.masterEnvVars.get("JENKINS_HOME")
May 15, 2018 9:36:28 AM org.eclipse.jetty.util.log.Log initialized
INFO: Logging initialized @449ms to org.eclipse.jetty.util.log.JavaUtilLog
May 15, 2018 9:36:28 AM winstone.Logger logInternal
INFO: Beginning extraction from war file
May 15, 2018 9:36:29 AM org.eclipse.jetty.server.handler.ContextHandler setContextPath
WARNING: Empty contextPath
May 15, 2018 9:36:29 AM org.eclipse.jetty.server.Server doStart
INFO: jetty-9.4.z-SNAPSHOT, build timestamp: 2017-11-21T21:27:37Z, git hash: 82b8fb23f757335bb3329d540ce37a2a2615f0a8
May 15, 2018 9:36:30 AM org.eclipse.jetty.webapp.StandardDescriptorProcessor visitServlet
INFO: NO JSP Support for /, did not find org.eclipse.jetty.jsp.JettyJspServlet
May 15, 2018 9:36:30 AM org.eclipse.jetty.server.session.DefaultSessionIdManager doStart
INFO: DefaultSessionIdManager workerName=node0
May 15, 2018 9:36:30 AM org.eclipse.jetty.server.session.DefaultSessionIdManager doStart
INFO: No SessionScavenger set, using defaults
May 15, 2018 9:36:30 AM org.eclipse.jetty.server.session.HouseKeeper startScavenging
INFO: Scavenging every 660000ms
Jenkins home directory: /var/jenkins_home found at: EnvVars.masterEnvVars.get("JENKINS_HOME")
May 15, 2018 9:36:30 AM org.eclipse.jetty.server.handler.ContextHandler doStart
INFO: Started w.@4c36250e{/,file:///var/jenkins_home/war/,AVAILABLE}{/var/jenkins_home/war}
May 15, 2018 9:36:30 AM org.eclipse.jetty.server.AbstractConnector doStart
INFO: Started ServerConnector@5ae81e1{HTTP/1.1,[http/1.1]}{0.0.0.0:8080}
May 15, 2018 9:36:30 AM org.eclipse.jetty.server.Server doStart
INFO: Started @3003ms
May 15, 2018 9:36:30 AM winstone.Logger logInternal
INFO: Winstone Servlet Engine v4.0 running: controlPort=disabled
May 15, 2018 9:36:32 AM jenkins.InitReactorRunner$1 onAttained
INFO: Started initialization
May 15, 2018 9:36:32 AM jenkins.InitReactorRunner$1 onAttained
INFO: Listed all plugins
May 15, 2018 9:36:33 AM jenkins.InitReactorRunner$1 onAttained
INFO: Prepared all plugins
May 15, 2018 9:36:33 AM jenkins.InitReactorRunner$1 onAttained
INFO: Started all plugins
May 15, 2018 9:36:33 AM jenkins.InitReactorRunner$1 onAttained
INFO: Augmented all extensions
May 15, 2018 9:36:33 AM jenkins.InitReactorRunner$1 onAttained
INFO: Loaded all jobs
May 15, 2018 9:36:34 AM hudson.model.AsyncPeriodicWork$1 run
INFO: Started Download metadata
May 15, 2018 9:36:34 AM jenkins.util.groovy.GroovyHookScript execute
INFO: Executing /var/jenkins_home/init.groovy.d/tcp-slave-agent-port.groovy
May 15, 2018 9:36:35 AM org.springframework.context.support.AbstractApplicationContext prepareRefresh
INFO: Refreshing org.springframework.web.context.support.StaticWebApplicationContext@980149a: display name [Root WebApplicationContext]; startup date [Tue May 15 09:36:35 GMT 2018]; root of context hierarchy
May 15, 2018 9:36:35 AM org.springframework.context.support.AbstractApplicationContext obtainFreshBeanFactory
INFO: Bean factory for application context [org.springframework.web.context.support.StaticWebApplicationContext@980149a]: org.springframework.beans.factory.support.DefaultListableBeanFactory@278655c6
May 15, 2018 9:36:35 AM org.springframework.beans.factory.support.DefaultListableBeanFactory preInstantiateSingletons
INFO: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@278655c6: defining beans [authenticationManager]; root of factory hierarchy
May 15, 2018 9:36:35 AM org.springframework.context.support.AbstractApplicationContext prepareRefresh
INFO: Refreshing org.springframework.web.context.support.StaticWebApplicationContext@5588c247: display name [Root WebApplicationContext]; startup date [Tue May 15 09:36:35 GMT 2018]; root of context hierarchy
May 15, 2018 9:36:35 AM org.springframework.context.support.AbstractApplicationContext obtainFreshBeanFactory
INFO: Bean factory for application context [org.springframework.web.context.support.StaticWebApplicationContext@5588c247]: org.springframework.beans.factory.support.DefaultListableBeanFactory@267f9436
May 15, 2018 9:36:35 AM org.springframework.beans.factory.support.DefaultListableBeanFactory preInstantiateSingletons
INFO: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@267f9436: defining beans [filter,legacy]; root of factory hierarchy
May 15, 2018 9:36:35 AM jenkins.install.SetupWizard init
INFO:

*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

8d9530ac19e04408903354a5ffa8a407

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************

--> setting agent port for jnlp
--> setting agent port for jnlp... done
May 15, 2018 9:36:54 AM hudson.model.UpdateSite updateData
INFO: Obtained the latest update center data file for UpdateSource default
May 15, 2018 9:36:55 AM hudson.model.DownloadService$Downloadable load
INFO: Obtained the updated data file for hudson.tasks.Maven.MavenInstaller
May 15, 2018 9:36:55 AM hudson.model.AsyncPeriodicWork$1 run
INFO: Finished Download metadata. 21,466 ms
May 15, 2018 9:37:32 AM hudson.model.UpdateSite updateData
INFO: Obtained the latest update center data file for UpdateSource default
May 15, 2018 9:37:32 AM jenkins.InitReactorRunner$1 onAttained
INFO: Completed initialization
May 15, 2018 9:37:32 AM hudson.WebAppMain$3 run
INFO: Jenkins is fully up and running
```

Check deployed apps, by default it will look in default namespace.

```sh
$ kubectl get deployment
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
jenkins   1         1         1            0           1m
```

Exposing deployments 

**Note:-** It necessary to expose the deployments because if the container dies then it will be able to still able to serve requests through this cluster service.

```sh
$ kubectl expose deployment jenkins
service "jenkins" exposed
```

You can now access Jenkins with the cluster IP even if your contianer goes down for that to work efficiently you need to create replicas

```sh
$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
jenkins      ClusterIP   10.98.194.121   <none>        8080/TCP   2s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    5d
```

Exposing deployments to be accessed from extrernal Ip and as an load balancer

```sh
$ kubectl expose deployment jenkins --port=8081 --target-port=8080 --external-ip=192.168.33.20 --type=LoadBalancer
service "jenkins" exposed

$ kubectl get services -n default
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
jenkins      LoadBalancer   10.98.16.190    192.168.33.10   8081:31118/TCP   2m
kubernetes   ClusterIP      10.96.0.1       <none>          443/TCP          1h
```

Now if you curl the cluster IP or external IP for Jenkins service you will still get the same respose as earlier.

```sh
$ curl 10.103.74.134:8080
<html><head><meta http-equiv='refresh' content='1;url=/login?from=%2F'/><script>window.location.replace('/login?from=%2F');</script></head><body style='background-color:white; color:white;'>

Authentication required
<!--
You are authenticated as: anonymous
Groups that you are in:

Permission you need to have (but didn't): hudson.model.Hudson.Administer
-->

</body></html>
```

Creating replicas in order to scale your deployments

```sh
$ kubectl scale deployment jenkins --replicas=3
deployment.extensions "jenkins" scaled
```

Checking replicas after creation

```sh
$ kubectl get po
NAME                       READY     STATUS    RESTARTS   AGE
jenkins-7fc6766c86-5txxm    1/1       Running   0          19s
jenkins-7fc6766c86-8lwnq    1/1       Running   0          19s
jenkins-8588bd6d9c-8w86p    1/1       Running   3          4d
```

**Note:-**

Once you have the replicas created, now every time you query the cluster IP for jenkins service it will everytime serve request from a different Pod as shown above, till now you have auto scaled and applied load balancing in your infrastructure through kubernetes.

In order to debug a container you can login into the shell of the container using below command

```sh
$ kubectl exec -i -t jenkins-8588bd6d9c-8w86p sh
/ $ uname -a
Linux jenkins-8588bd6d9c-8w86p 4.4.0-122-generic #146-Ubuntu SMP Mon Apr 23 15:34:04 UTC 2018 x86_64 GNU/Linux
```

Deploying the Dashboard UI

```sh
$ kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
rolebinding.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
deployment.apps "kubernetes-dashboard" created
service "kubernetes-dashboard" created
```

Accessing the Dashboard UI

```sh
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

**Note:-**

The above command will let you access the dashboard through localhost (127.0.0.1) on port 8001. But in our case as we are using Vagrant VM we will not be able to access the dasboard through localhost. For this we have to mention the master IP address through which we can access it in our local computer.

$ kubectl proxy --address <MASTER_EXTERNAL_IP> --port=8001 --accept-hosts='^*$' 

**The regular experssion in arg --accept-hosts means that it can be accessed from any IP through which master IP is pingable, if you don't use this argument then you get unauthorized response from the kubernetes dashboard, in case of more security you can also use a particular IP address or a network through which you want to access the dashboard**

```sh
$ kubectl proxy --address 192.168.33.10 --port=8001 --accept-hosts='^*$'
Starting to serve on 192.168.33.10:8001
```
Using the above command we can access the master vm through our local machine and woulb be able to access the Dashboard in the local machine's browser.

**Dashboard will be available at**
http://<master-IP>:<port>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/.

**In our case master IP will be: 192.168.33.10 & port will be: 8001**

Login to the Dashboard UI

Before configuring or loging in to the dashboard you might want to look the Access Control page for better understanding
https://github.com/kubernetes/dashboard/wiki/Access-control

**There are two ways to login to the dashboard UI of Kubernetes**
1. KubeConfig file
2. Access Token

As of release 1.7 and above Dashboard no longer has full admin privileges granted by default. All the privileges are revoked and only minimal privileges granted, that are required to make Dashboard work. In case Dashboard is accessible only by trusted set of people, all with full admin privileges you may want to grant it admin privileges.

**Kubeconfig**
User in kubeconfig file need either username & password or token, while admin.conf only have client-certificate.

```sh
$ kubectl config set-credentials cluster-admin --token=bearer_token
```

**Getting token with kubectl**

There are many Service Accounts created in Kubernetes by default. All with different access permissions. In order to find any token, that can be used to log in we'll use kubectl:

```sh
# Check existing secrets in kube-system namespace
$ kubectl -n kube-system get secret
# All secrets with type 'kubernetes.io/service-account-token' will allow to log in.
# Note that they have different privileges.
NAME                                             TYPE                                  DATA      AGE
attachdetach-controller-token-p2xks              kubernetes.io/service-account-token   3         6d
bootstrap-signer-token-r9h9h                     kubernetes.io/service-account-token   3         6d
certificate-controller-token-khwtl               kubernetes.io/service-account-token   3         6d
clusterrole-aggregation-controller-token-rslwc   kubernetes.io/service-account-token   3         6d
cronjob-controller-token-5pmss                   kubernetes.io/service-account-token   3         6d
daemon-set-controller-token-grbmw                kubernetes.io/service-account-token   3         6d
default-token-dw8j4                              kubernetes.io/service-account-token   3         6d
deployment-controller-token-lhnzl                kubernetes.io/service-account-token   3         6d
disruption-controller-token-zrc6q                kubernetes.io/service-account-token   3         6d
endpoint-controller-token-z2zx2                  kubernetes.io/service-account-token   3         6d
flannel-token-rm2zt                              kubernetes.io/service-account-token   3         6d
generic-garbage-collector-token-hj5nt            kubernetes.io/service-account-token   3         6d
horizontal-pod-autoscaler-token-lzjm2            kubernetes.io/service-account-token   3         6d
job-controller-token-z86nd                       kubernetes.io/service-account-token   3         6d
kube-dns-token-vsrzk                             kubernetes.io/service-account-token   3         6d
kube-proxy-token-wj7pr                           kubernetes.io/service-account-token   3         6d
kubernetes-dashboard-certs                       Opaque                                0         1d
kubernetes-dashboard-key-holder                  Opaque                                2         5d
kubernetes-dashboard-token-6rrvc                 kubernetes.io/service-account-token   3         1d
namespace-controller-token-8fpxs                 kubernetes.io/service-account-token   3         6d
node-controller-token-qt2dc                      kubernetes.io/service-account-token   3         6d
persistent-volume-binder-token-vlbkd             kubernetes.io/service-account-token   3         6d
pod-garbage-collector-token-4vzh4                kubernetes.io/service-account-token   3         6d
pv-protection-controller-token-vztzj             kubernetes.io/service-account-token   3         6d
pvc-protection-controller-token-nwglm            kubernetes.io/service-account-token   3         6d
replicaset-controller-token-rrkdz                kubernetes.io/service-account-token   3         6d
replication-controller-token-q2s7m               kubernetes.io/service-account-token   3         6d
resourcequota-controller-token-75x6m             kubernetes.io/service-account-token   3         6d
service-account-controller-token-nl5rf           kubernetes.io/service-account-token   3         6d
service-controller-token-x24dx                   kubernetes.io/service-account-token   3         6d
statefulset-controller-token-kkt2c               kubernetes.io/service-account-token   3         6d
token-cleaner-token-2mhcx                        kubernetes.io/service-account-token   3         6d
ttl-controller-token-4t2p4                       kubernetes.io/service-account-token   3         6d

$ kubectl describe secret deployment-controller-token-lhnzl -n kube-system
Name:         deployment-controller-token-lhnzl
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=deployment-controller
              kubernetes.io/service-account.uid=8618cb23-5422-11e8-a038-02d4207bc2aa

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkZXBsb3ltZW50LWNvbnRyb2xsZXItdG9rZW4tbGhuemwiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVwbG95bWVudC1jb250cm9sbGVyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiODYxOGNiMjMtNTQyMi0xMWU4LWEwMzgtMDJkNDIwN2JjMmFhIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRlcGxveW1lbnQtY29udHJvbGxlciJ9.lPv4cawnlq0uZIuVGq0gOUqVRiuomTJ2uUZcejOyO2fQGm71vqRyLVALdlPkbK5xLID64QAskxZkOCi88zemnAyjHUUhZb2N_BHzh_N2Ls_x6f0C5i55yp3OdKgt1J2laxI-_mwrH-D8onrg_7_NlYAFDlTUBMDrUSuiuUtJ4PWwofriS1uSllgrxvuxHCu8yq7w1JJTyN7I_2nNgh7Rm-u8psoz15Ps5ziIL2zzgYV1o9g0QSNu7rrVOnfCMR0QUlQzFfPqalNIRYkErygpUQrCCuO6DlueUBNYKT2R5h66L8U-W1yQlolgBcj-uijqU6B4ZJx5k62h3aKpNJFn5Q
ca.crt:     1025 bytes
namespace:  11 bytes
```

**Now use the above ```token``` to login into the dashboard.**

#### If the above two doesn't work for you, then you can try to give Admin privileges

**Note:-**

Granting admin privileges to Dashboard's Service Account might be a security risk.

You can grant full admin privileges to Dashboard's Service Account by creating below ```ClusterRoleBinding```. Copy the YAML file based on chosen installation method and save as, i.e. ```dashboard-admin.yaml```. Use ```kubectl create -f dashboard-admin.yaml``` to deploy it. Afterwards you can use ```Skip``` option on login page to access Dashboard.

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
  ```

#### Congratulations! You have successfully created a Kubernetes Cluster and deployed an app on top of it.

# Known Issues
---
#### 1. Issue while accessing kubectl commands

**Error:-** 

- The connection to the server localhost:8080 was refused - did you specify the right host or port?

**Possible Senarios**

- This is the case when you are not accessing or running ```kubectl``` command by the authorized user, even if you are ```root``` user.
- Misconfiguration issue with the admin.conf file

**Solution:-** 

- Switch to the correct user and then run the command.
- Copy admin.conf file again as an authorized user ```cp -i /etc/kubernetes/admin.conf $HOME/.kube/config```.

**Note:- 
If you are using flannel as the pod network then you are likely to face the error as mentioned below.**
---
#### 2. Default NIC When using flannel as the pod network in Vagrant
The following error might indicate that something was wrong in the pod network:

```Error from server (NotFound): the server could not find the requested resource```

If you’re using flannel as the pod network inside vagrant, then you will have to specify the default interface name for flannel.

Vagrant typically assigns two interfaces to all VMs. The first, for which all hosts are assigned the IP address ```10.0.2.15```, is for external traffic that gets NATed.

This may lead to problems with flannel. By default, flannel selects the first interface on a host. This leads to all hosts thinking they have the same public IP address. To prevent this issue, pass the --iface eth1 flag to flannel so that the second interface is chosen.

**Solution:-**

Configuring flannel to use a non default interface in kubernetes

1. Download the kube-flannel.yml file
    
    ```sh
    wget https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
    ```

2. Open the file using vim or any other text editor, there you should add the required "--iface=enp0s8" argument.
    
    ```sh
    $ vim kube-flannel.yml
    containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.10.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=enp0s8
        resources:
        ...
        ...
    ```

After that run:
```sh
$ kubectl apply -f kube-flannel.yml

$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
```

**You can also set your 2nd interface gateway as a default one by running the command below**

$ ip route change default via **<gateway>** dev **<interface_name>**

```sh
$ sudo ip route change default via 192.168.33.1 dev enp0s8
$ systemctl restart networking.service
```

**Note:-**
If you change your default route to 2nd interface you will loose internet connectivity as that would be hostonly network interface. If you want to still use only one interface then you need bridged network unlike NAT network for the 1st interface.

#### 3. Flannel forwarding traffic to different network

The cause is the bridge's fdb table has lost the entries of unreachable nodes.

**Solution:-**
Use "bridge fdb show" to list all fdb entries, and check if unreachable node's entries is missed.

```sh
$ bridge fdb show
xx:xx:xx:57:27:d8 dev flannel.1 dst 192.168.110.89 self permanent
xx:xx:xx:64:26:2f dev flannel.1 dst 192.168.110.86 self permanent
xx:xx:xx:25:6e:65 dev flannel.1 dst 192.168.110.82 self permanent
....
```
Use ```bridge``` command to add the lost entry, but if you get this **error:** ```"RTNETLINK answers: File exists"```.

```sh
$ sudo bridge fdb add xx:xx:xx:cc:dd:a7 dev flannel.1 dst 192.168.110.18 self permanent
RTNETLINK answers: File exists
```

It means some existing file prevented from adding fdb entry. So, first "del" and then add it again.

```sh
$ sudo bridge fdb del xx:xx:xx:cc:dd:a7 dev flannel.1
$ sudo bridge fdb add xx:xx:xx:cc:dd:a7 dev flannel.1 dst 192.168.110.18 self permanent
```
After I make this fix on both side of unreachable nodes. They can ping each other.

---

#### 4. Error Code 404 (checked through verbose)

Everything was great, we could deploy a service like nginx and proxy to it. However, we hit a snag when we wanted to use busybox to debug something with an application container. For some reason, running kubectl exec appeared with an error message!

```sh
$ kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
head      Ready     master    2m        v1.9.1
worker1   Ready     <none>    1m        v1.9.1

$ kubectl apply -f busybox.yml
deployment "busybox" created

$ kubectl get pods
NAME                       READY     STATUS              RESTARTS   AGE
busybox-7c9687585b-297lt   0/1       ContainerCreating   0          30s

$ kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
busybox-7c9687585b-297lt   1/1       Running   0          1m

$ kubectl exec -it busybox-7c9687585b-297lt -- /bin/sh
error: unable to upgrade connection: pod does not exist

$ kubectl logs busybox-7c9687585b-297lt
Error from server (NotFound): the server could not find the requested resource ( pods/log busybox-7c9687585b-297lt)

$ kubectl -v=9 exec -it busybox-7c9687585b-297lt -- /bin/sh
...
I0117 20:39:23.041933   57121 round_trippers.go:436] POST https://192.168.205.10:6443/api/v1/namespaces/default/pods/busybox-7c9687585b-297lt/exec?command=%2Fbin%2Fsh&container=busybox&container=busybox&stdin=true&stdout=true&tty=true 404 Not Found in 14 milliseconds
I0117 20:39:23.041971   57121 round_trippers.go:442] Response Headers:
I0117 20:39:23.041975   57121 round_trippers.go:445] Date: Thu, 18 Jan 2018 01:38:58 GMT
I0117 20:39:23.041978   57121 round_trippers.go:445] Content-Length: 18
I0117 20:39:23.041980   57121 round_trippers.go:445] Content-Type: text/plain; charset=utf-8
F0117 20:39:23.042166   57121 helpers.go:119] error: unable to upgrade connection: pod does not exist

$ kubectl -v=9 logs busybox-7c9687585b-297lt
...
I0117 20:39:34.539175   57215 request.go:873] Response Body: {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"the server could not find the requested resource ( pods/log busybox-7c9687585b-297lt)","reason":"NotFound","details":{"name":"busybox-7c9687585b-297lt","kind":"pods/log"},"code":404}
I0117 20:39:34.539529   57215 helpers.go:201] server response object: [{
  "metadata": {},
  "status": "Failure",
  "message": "the server could not find the requested resource ( pods/log busybox-7c9687585b-297lt)",
  "reason": "NotFound",
  "details": {
    "name": "busybox-7c9687585b-297lt",
    "kind": "pods/log"
  },
  "code": 404
}]
F0117 20:39:34.539555   57215 helpers.go:119] Error from server (NotFound): the server could not find the requested resource ( pods/log busybox-7c9687585b-297lt)
```
Got HTTP Error Code 404.

**Solution**

It seems like the right server endpoint needs to be passed in order for the connection to happen. As explained by Kubernetes’s page on Master-Node connection, the API server communicates with the kubelet based on its HTTPS endpoint.

```sh
$ kubectl get nodes worker1 -o yaml
apiVersion: v1
kind: Node
...
status:
  addresses:
  - address: 10.0.2.15
...
```

From the above output we figured out the **IP address of the node is wrong**.
Why? because, vagrant creates two network interfaces for each machine. eth0 is NAT networked, eth1 is a private network.
The main Kubernetes interface is on eth1. In order for the worker to properly join the master, we had to explicitly state the ```--apiserver-advertise-address``` to point to eth1.

We needed to add an explicit IP address that uses the eth1 interface on the worker, enabling the master’s API server to properly access the worker’s kubelet. WE looked at the man page of the kubelet and I found the following option:
```--node-ip``` string, IP address of the node. If set, kubelet will use this IP address for the node.

In order to set this IP address we have to change kubelet Configuration. As kubeadm cannot inject anything into the kubelet’s drop-in configuration via CLI. Instead, we had to edit the kubelet’s configuration file to add the additional argument.

Add ```Environment="KUBELET_EXTRA_ARGS=--node-ip=<worker IP address>"```to 10-kubeadm.conf file.

```sh
$ less /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
Environment="KUBELET_EXTRA_ARGS=--node-ip=<worker IP address>"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CGROUP_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS
```

Reload the deamon and restart Kubelet service

```sh
$ systemctl daemon-reload
$ systemctl restart kubelet
```
---

For more troubleshooting options for ```kubeadm``` you can refer to the this link https://kubernetes.io/docs/setup/independent/troubleshooting-kubeadm/

---

**Refered from sources https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/ & https://blog.alexellis.io & https://stackoverflow.com/ & https://github.com/kubernetes/dashboard/wiki/Access-control & https://github.com/coreos/flannel/issues/654**

---
