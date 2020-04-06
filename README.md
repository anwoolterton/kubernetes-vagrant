# vagrant-kubernetes
Run provisioned Kubernetes cluster within Vagrant.

## Install Brew
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

## Virutal Box and Vagrant 
```
brew cask install vagrant
brew cask install virtualbox
```

## Vagrant Plugins
```
vagrant plugin install vagrant-ssh
vagrant plugin installvagrant-hostmanager
vagrant plugin install vagrant-ssh
```

## Vagrant Start
```
vagrant up
```

## Vagrant Down
```
vagrant destroy
```


# Access The cluster

## Login
```
vagrant ssh k8s-master

vagrant@k8s-master:~$ kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   6m39s   v1.18.0
k8s-node-1   Ready    <none>   3m45s   v1.18.0
k8s-node-2   Ready    <none>   69s     v1.18.0

```
## Create Accounts and role bindings
```
kubectl create serviceaccount k8sadmin -n kube-system
kubectl create clusterrolebinding k8sadmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8sadmin
```

## Get Authority CA

```
kubectl config view --flatten --minify
```

## View Bearer token

```
kubectl -n kube-system describe secret $( kubectl -n kube-system get secret | (grep k8sadmin || echo "$_") | awk '{print $1}') | grep token: | awk '{print $2}'
```

## Create Kubectl

```
apiVersion: v1
kind: Config
users:
- name: k8sadmin
  user:
    token: REMOVE - see `View Bearer token`
clusters:
- cluster:
    certificate-authority-data: REMOVED - see `Get Authority CA`
    server: https://k8s-master:6443
  name: k8s-master
contexts:
- context:
    cluster: k8s-master
    user: k8sadmin
  name: k8s-master
current-context: k8s-master%
```

## Kubectl example

```
➜  vagrant-kubernetes git:(master) ✗ kubectl get nodes --insecure-skip-tls-verify --kubeconfig kube
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   51m   v1.18.0
k8s-node-1   Ready    <none>   49m   v1.18.0
k8s-node-2   Ready    <none>   46m   v1.18.0
```