# Running Concourse CI in kubernetes

Concourse CI is one of those little gems that can be really useful. It is a CI/CD tool that enables you to run just about any sort of delivery pipeline for your projects. It is small, selfcontained, and can talk to just about anything using third party resource types. In addition, it stores nothing locally, and forces you to store artifacts elsewhere. 

In short, the perfect little cloud job runner.

It is easy to get up and running in kubernetes, and for this demo, we will show how to set it up with microk8s, a tiny kubernetes distribution from Ubuntu that is easy to install. Concourse CI runs on any kubernetes of course, if you have greater needs. 


# Installing Concourse CI

Concourse CI comes with a nice helm chart that we will use. It enables a lot of customization, but for this demo we will use a simple setup. 

We want to install for the following:

* URL https://concourse.microk8s.local
* Username concourse
* Password concourse

## Set up helm repo

First, we need to install the helm chart repo that holds the concourse chart:

```console
helm repo add concourse https://concourse-charts.storage.googleapis.com
helm repo update
```

## Create a values file for helm install

Create this file:

```console
---
# Local user and password stored in k8s secret
secrets:
  create: true
  localUsers: "concourse:concourse"
concourse:
  # Set up two workers to run pipelines
  worker:
    enabled: true
    replicas: 2
    # This overrides the default to something we know works in k8s. 
    baggageclaim:
      driver: overlay
  # Set up an ingress too!
  web:
    ingress:
      enabled: true
      hosts:
        - concourse.microk8s.local
    # Don't create team namespaces. Enable if you like that idea ;)
    kubernetes:
      createTeamNamespaces: false
    # Name for the cluster
    clusterName: redpill
    externalUrl: https://concourse.microk8s.local
    # Needed to set up auth according to secrets above
    auth:
      mainTeam:
        localUser: "concourse"
```

## Install the helm chart

This is straightforward using helm. Make sure you're in a good namespace.

```console
helm install concourse concourse/concourse --values concourse.yaml
```

This will return a short message. With our ingress, we should be able to visit concourse at https://concourse.microk8s.local after a little while.

You can see if all pods are up:

```console
(⎈ |microk8s:concourse) ~/code/concourse-blog
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
concourse-postgresql-0           1/1     Running   0          27m
concourse-web-5d674dc8d9-xwqb8   1/1     Running   0          20m
concourse-worker-0               1/1     Running   0          27m
concourse-worker-1               1/1     Running   0          27m
```

Note that concourse uses a postgresql database for persistent storage. 

# Extra material: Set up microk8s in Ubuntu

Microk8s comes as a snap in ubuntu, and can be installed like this:

```console
$ sudo snap install microk8s --classic --channel=1.18/stable
microk8s (1.18/stable) v1.18.8 from Canonical✓ installed
(⎈ |microk8s:concourse) ~
$ sudo microk8s start
Started.
Enabling pod scheduling
node/tiny already uncordoned
(⎈ |microk8s:concourse) ~
$ sudo microk8s enable dns storage ingress
Enabling DNS
Applying manifest
serviceaccount/coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
clusterrole.rbac.authorization.k8s.io/coredns created
clusterrolebinding.rbac.authorization.k8s.io/coredns created
Restarting kubelet
DNS is enabled
Enabling default storage class
deployment.apps/hostpath-provisioner created
storageclass.storage.k8s.io/microk8s-hostpath created
serviceaccount/microk8s-hostpath created
clusterrole.rbac.authorization.k8s.io/microk8s-hostpath created
clusterrolebinding.rbac.authorization.k8s.io/microk8s-hostpath created
Storage will be available soon
Enabling Ingress
namespace/ingress created
serviceaccount/nginx-ingress-microk8s-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-microk8s-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-microk8s-role created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
configmap/nginx-load-balancer-microk8s-conf created
daemonset.apps/nginx-ingress-microk8s-controller created
Ingress is enabled
```

If you have your own way to add a kube config, you should do so. If you don't have one, do this:

```console
mkdir $HOME/.kube
sudo microk8s config > $HOME/.kube/config
```

You should then be able to see your node:

```console
(⎈ |microk8s:default) ~/code/concourse-blog
$ kubectl get nodes
NAME   STATUS   ROLES    AGE     VERSION
tiny   Ready    <none>   3m44s   v1.18.8
```

You should also see quite a few things running:

```console
$ kubectl get all --all-namespaces
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
ingress       pod/nginx-ingress-microk8s-controller-jb245   1/1     Running   0          2s
kube-system   pod/coredns-588fd544bf-hdr5z                  1/1     Running   0          5m24s
kube-system   pod/hostpath-provisioner-75fdc8fccd-tlcvj     1/1     Running   0          5m15s

NAMESPACE     NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.152.183.1    <none>        443/TCP                  5m50s
kube-system   service/kube-dns     ClusterIP   10.152.183.10   <none>        53/UDP,53/TCP,9153/TCP   5m24s

NAMESPACE   NAME                                               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ingress     daemonset.apps/nginx-ingress-microk8s-controller   1         1         1       1            1           <none>          5m14s

NAMESPACE     NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns                1/1     1            1           5m24s
kube-system   deployment.apps/hostpath-provisioner   1/1     1            1           5m16s

NAMESPACE     NAME                                              DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-588fd544bf                1         1         1       5m24s
kube-system   replicaset.apps/hostpath-provisioner-75fdc8fccd   1         1         1       5m16s
```

OK, all good!
