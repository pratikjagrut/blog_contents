+++
author = "Suraj Deshmukh"
title = "Add new Node to k8s cluster with cert rotation"
description = "Use this technique to add node to the cluster without providing any certificates"
date = "2018-10-16T01:00:51+05:30"
categories = ["kubernetes"]
tags = ["kubernetes", "security"]
+++

The setup here is created by following [*Kubernetes the Hard Way*](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/) by [**Kelsey Hightower**](https://twitter.com/kelseyhightower). So if you are following along in this then do all the setup till the step [*Bootstrapping the Kubernetes Worker Nodes*](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/09-bootstrapping-kubernetes-workers.md). In this just don't start the `kubelet`, start other services like `containerd` and `kube-proxy`.

## master node

Following the docs of [*TLS Bootstrapping*](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/), let's first create the token authentication file. Create a file with following content:

```bash
$ cat tokenfile 
02b50b05283e98dd0fd71db496ef01e8,kubelet-bootstrap,10001,"system:bootstrappers"
```

You should create the token which is as random as possible by running following command:

```bash
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```

Now we need to tell the `kube apiserver` about this file. So add following flag to the `kube-apiserver` service file, with the path to above token file.

```bash
--token-auth-file=/home/vagrant/tokenfile
```

So my `kube-apiserver` service file looks like following:

```bash
$ cat /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=192.168.50.10 \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/audit.log \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --client-ca-file=/var/lib/kubernetes/ca.pem \
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --enable-swagger-ui=true \
  --etcd-cafile=/var/lib/kubernetes/ca.pem \
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \
  --etcd-servers=https://192.168.50.10:2379 \
  --event-ttl=1h \
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \
  --kubelet-https=true \
  --runtime-config=api/all \
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --token-auth-file=/home/vagrant/tokenfile \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Once this is done, just run following commands to restart the apiserver:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kube-apiserver
sudo systemctl status kube-apiserver
```

Now that api-server has restart we need to give permissions so that our worker node can ask for certs automatically, for that run following commands:

```bash
kubectl create clusterrolebinding kubelet-bootstrap \
    --clusterrole=system:node-bootstrapper \
    --user=kubelet-bootstrap

kubectl create clusterrolebinding node-client-auto-approve-csr \
    --clusterrole=system:certificates.k8s.io:certificatesigningrequests:nodeclient \
    --group=system:node-bootstrappers

kubectl create clusterrolebinding node-client-auto-renew-crt \
    --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeclient \
    --group=system:nodes
```

## worker node

First create a bootstrap kubeconfig file that will be used by `kubelet`. Run following commands to create it.

```
# I have used the ip address of my api-server use yours
kubectl config set-cluster kthw \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.50.10:6443 \
  --kubeconfig=bootstrap.kubeconfig

# this token is above generated
kubectl config set-credentials kubelet-bootstrap \
  --token=02b50b05283e98dd0fd71db496ef01e8 \
  --kubeconfig=bootstrap.kubeconfig

kubectl config set-context default \
  --cluster=kthw \
  --user kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig

kubectl config use-context default \
  --kubeconfig=bootstrap.kubeconfig 
```

Now that we have this file, add following flags to the `kubelet` systemd service file.

```bash
--bootstrap-kubeconfig=/home/vagrant/bootstrap.kubeconfig
--kubeconfig=/home/vagrant/kubeconfig
--rotate-certificates=true
--rotate-server-certificates=true
```

Provide path to the `bootstrap.kubeconfig` you have generated just before. And even if you don't have `kubeconfig` still provide some path where `kubelet` has permission to write. `kubelet` will create this file for you.

So my `kubelet`, systemd file looks like following:

```bash
$ cat /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --bootstrap-kubeconfig=/home/vagrant/bootstrap.kubeconfig \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --container-runtime=remote \
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
  --image-pull-progress-deadline=2m \
  --kubeconfig=/home/vagrant/kubeconfig \
  --network-plugin=cni \
  --register-node=true \
  --rotate-certificates=true \
  --rotate-server-certificates=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

And `KubeletConfiguration` looks like this:

```bash
$ cat /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "10.200.0.0/24"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
```

Once this is done, just run following commands to restart the kubelet:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo systemctl status kubelet
```

## master node

List the request that node has made:

```bash
$ kubectl get csr
NAME                                                   AGE   REQUESTOR           CONDITION
node-csr-WXnon3AEhdxgZ1FZ2reqKRQmWS-pwP3x263YbwUAH9k   11m   kubelet-bootstrap   Pending
```

Approve that request:

```bash
$ kubectl certificate approve node-csr-WXnon3AEhdxgZ1FZ2reqKRQmWS-pwP3x263YbwUAH9k
certificatesigningrequest.certificates.k8s.io/node-csr-WXnon3AEhdxgZ1FZ2reqKRQmWS-pwP3x263YbwUAH9k approved
```


Now you can see that the node has been created and you can list by running:

```bash
$ kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
node   Ready    <none>   27s   v1.12.0
```
