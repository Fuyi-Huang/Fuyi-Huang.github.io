---
title: 用kubeadm安装kubernetes集群过程记录
tags: 
    - linux 
article_header:
  type: cover
  image:
    src: assets/images/header_images/zelda.jpg
mathjax: true
mathjax_autoNumber: true
---

## Before you begin
1. 本次使用kubeadm搭建kubernetes集群使用两台Linux主机。如果是单台主机想要学习搭建kubernetes集群，建议使用[kind](https://fuyi-huang.github.io/2022/05/30/kind-kubernetes.html)
2. 使用kubeadm最好是有一个镜像仓库，[kind](https://kind.sigs.k8s.io/)有个好处是，可以把主机的docker镜像加载到kind k8s集群内使用
3. **注意** 本文的内容仅对v1.24.1版本有效，因为之前的版本可能不太符合，主要是这个版本之后kubernetes开始废弃了对docker的内置兼容（docker不符合CRI），所以对于v1.24版本以后的kubernetes，需要安装cri-docker

https://docs.github.com/en/actions/publishing-packages/publishing-docker-images#publishing-images-to-github-packages

## 安装docker
在每台机器上都需要安装docker
```sh
sudo apt  update
sudo apt install docker.io
```
安装docker对应的CRI，因为现在docker不遵循标准的CRI协议，需要另外一个cri-docker来让kubernetes支持docker作为容器的运行时。
参考：https://computingforgeeks.com/install-mirantis-cri-dockerd-as-docker-engine-shim-for-kubernetes/
```shell

```

## 安装flanneld
```
wget https://github.com/flannel-io/flannel/releases/download/v0.18.1/flanneld-amd64
chmod +x flanneld-amd64
sudo mkdir -p /opt/bin
sudo mv flanneld-amd64 /opt/bin/
```


## kubeadm
安装kubeadm， kubectl， kubelet
按照教程走就好：https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
```shell
## 安装之后检查版本
ubuntu@VM-0-2-ubuntu:~$ kubectl version
kubeadm version: &version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.1", GitCommit:"3ddd0f45aa91e2f30c70734b175631bec5b5825a", GitTreeState:"clean", BuildDate:"2022-05-24T12:24:38Z", GoVersion:"go1.18.2", Compiler:"gc", Platform:"linux/amd64"}
```

### kubeadm 创建k8s集群
```
Found multiple CRI endpoints on the host. Please define which one do you wish to use by setting the 'criSocket' field in the kubeadm configuration file: unix:///var/run/containerd/containerd.sock, unix:///var/run/cri-dockerd.sock
To see the stack trace of this error execute with --v=5 or higher
```
sudo kubeadm init  --cri-socket unix:///var/run/cri-dockerd.sock    --pod-network-cidr=192.168.0.0/16

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.19.0.2:6443 --token elzb6i.9pepuyiwx0qfjluh \
        --discovery-token-ca-cert-hash sha256:f09a8599a256edd75f5b8ae3913d73e691bc871ef20e3f423d83ff3aaca2fe96
```
最后一行
`kubeadm join 172.19.0.2:6443 --token kjalfo.hx7609lm2qdg1c05 \
        --discovery-token-ca-cert-hash sha256:6fe5bfd1e16e60955335915fca006b81145e8615254392303c10dd8b377bf6a4`是添加worker节点到k8s集群
的命令，在worker节点上执行即可添加该节点到创建的k8s集群。
这个token的时效是24小时，如果过了时效，需要生成新的token
```shell
kubeadm token create --print-join-command
```
执行时，和`kubeadm init`一样，需要加上cri-docker.sock。
如果提示下面的错误
```
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR IsPrivilegedUser]: user is not running as root
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```
则需要以root权限执行

`sudo kubeadm join 172.19.0.2:6443 --token elzb6i.9pepuyiwx0qfjluh  --discovery-token-ca-cert-hash sha256:f09a8599a256edd75f5b8ae3913d73e691bc871ef20e3f423d83ff3aaca2fe96   --cri-socket unix:///var/run/cri-dockerd.sock`
```
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

### Installing a Pod network add-on
Container Network Interface (CNI) 
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
修改kube-flannel.yaml的子网
kubectl apply -f kube-flannel.yml
kubectl get pods --all-namespaces  -o wide
```
NAMESPACE     NAME                                    READY   STATUS    RESTARTS        AGE     IP           NODE            NOMINATED NODE   READINESS GATES
kube-system   coredns-6d4b75cb6d-9l489                1/1     Running   2 (5d11h ago)   6d11h   172.17.0.3   vm-0-2-ubuntu   <none>           <none>
kube-system   coredns-6d4b75cb6d-rbsw8                1/1     Running   1 (5d11h ago)   6d11h   172.17.0.2   vm-0-2-ubuntu   <none>           <none>
kube-system   etcd-vm-0-2-ubuntu                      1/1     Running   1 (5d11h ago)   6d11h   172.19.0.2   vm-0-2-ubuntu   <none>           <none>
kube-system   kube-apiserver-vm-0-2-ubuntu            1/1     Running   1 (5d11h ago)   6d11h   172.19.0.2   vm-0-2-ubuntu   <none>           <none>
kube-system   kube-controller-manager-vm-0-2-ubuntu   1/1     Running   1 (5d11h ago)   6d11h   172.19.0.2   vm-0-2-ubuntu   <none>           <none>
kube-system   kube-flannel-ds-gt94g                   1/1     Running   1 (5d11h ago)   6d11h   172.19.0.2   vm-0-2-ubuntu   <none>           <none>
kube-system   kube-proxy-q8wdw                        1/1     Running   1 (5d11h ago)   6d11h   172.19.0.2   vm-0-2-ubuntu   <none>           <none>
kube-system   kube-scheduler-vm-0-2-ubuntu            1/1     Running   1 (5d11h ago)   6d11h   172.19.0.2   vm-0-2-ubuntu   <none>           <none>
```
检查IPAddress docker给容器分配的IP地址是不是来自子网192.168.0.0/16。可以看到coredns两个pod的IP不是我们分配的IP范围。这是因为cri-docker默认使用docker0网卡分配的IP，我们上面安装了flannel CNI，会新增一个网卡flannel.1，
```shell
ubuntu@VM-0-2-ubuntu:~$ ifconfig
cni0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 192.168.0.1  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::30d4:e8ff:fecd:4a5e  prefixlen 64  scopeid 0x20<link>
        ether 32:d4:e8:cd:4a:5e  txqueuelen 1000  (Ethernet)
        RX packets 1269  bytes 102330 (102.3 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1210  bytes 154727 (154.7 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:a0ff:fe62:4e0e  prefixlen 64  scopeid 0x20<link>
        ether 02:42:a0:62:4e:0e  txqueuelen 0  (Ethernet)
        RX packets 1205098  bytes 97589569 (97.5 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1204823  bytes 110779877 (110.7 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.19.0.2  netmask 255.255.240.0  broadcast 172.19.15.255
        inet6 fe80::5054:ff:fea2:2386  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:a2:23:86  txqueuelen 1000  (Ethernet)
        RX packets 3345151  bytes 889666268 (889.6 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2956412  bytes 578603922 (578.6 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 192.168.0.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::a420:65ff:fe3e:34ee  prefixlen 64  scopeid 0x20<link>
        ether a6:20:65:3e:34:ee  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 167 overruns 0  carrier 0  collisions 0
```
我们期望分配的IP是来自flannel网卡，修改方式 https://github.com/Mirantis/cri-dockerd/issues/73 
```txt
## 编辑cri-docker服务启动文件
sudo vim /etc/systemd/system/cri-docker.service
## 在文件里修改--network-plugin=cni
ExecStart=/usr/local/bin/cri-dockerd --container-runtime-endpoint fd:// --network-plugin=cni
## 重启cri-docker
sudo systemctl daemon-reload
sudo systemctl restart cri-docker.service
```
删掉重启那俩个coredns pod
```shell
ubuntu@VM-0-2-ubuntu:~$ kubectl delete pod -n kube-system   coredns-6d4b75cb6d-9l489
ubuntu@VM-0-2-ubuntu:~$ kubectl delete pod -n kube-system     coredns-6d4b75cb6d-rbsw8
ubuntu@VM-0-2-ubuntu:~$ kubectl get pods --all-namespaces  -o wide
  NAMESPACE     NAME                                    READY   STATUS    RESTARTS        AGE     IP            NODE            NOMINATED NODE   READINESS GATES
  kube-system   coredns-6d4b75cb6d-6tfm5                1/1     Running   0               4m22s   192.168.0.2   vm-0-2-ubuntu   <none>           <none>
  kube-system   coredns-6d4b75cb6d-bwjqz                1/1     Running   0               4m14s   192.168.0.3   vm-0-2-ubuntu   <none>           <none>
  kube-system   etcd-vm-0-2-ubuntu                      1/1     Running   1 (5d11h ago)   6d11h   172.19.0.2    vm-0-2-ubuntu   <none>           <none>
  kube-system   kube-apiserver-vm-0-2-ubuntu            1/1     Running   1 (5d11h ago)   6d11h   172.19.0.2    vm-0-2-ubuntu   <none>           <none>
  kube-system   kube-controller-manager-vm-0-2-ubuntu   1/1     Running   1 (5d11h ago)   6d11h   172.19.0.2    vm-0-2-ubuntu   <none>           <none>
  kube-system   kube-flannel-ds-gt94g                   1/1     Running   1 (5d11h ago)   6d11h   172.19.0.2    vm-0-2-ubuntu   <none>           <none>
  kube-system   kube-proxy-q8wdw                        1/1     Running   1 (5d11h ago)   6d11h   172.19.0.2    vm-0-2-ubuntu   <none>           <none>
  kube-system   kube-scheduler-vm-0-2-ubuntu            1/1     Running   1 (5d11h ago)   6d11h   172.19.0.2    vm-0-2-ubuntu   <none>           <none>
```
可以看到新的pod的IP是从flannel网卡分配的了。

## 创建k8s拉起镜像secret
这里镜像仓库厂商不限，可以是docker.io、腾讯云镜像仓库。但是腾讯云镜像仓库收费很贵，docker.io免费账号只有一个私有镜像名额，且每天限制拉取镜像的次数。这里我是用的是github镜像仓库ghcr.io。免费、不限制私有镜像数量，不限制拉取镜像次数。

首先创建GitHub镜像仓库token, 在 https://github.com/settings/tokens/new 进行创建，创建时需要至少勾选 `write:packages` 和 `read:packages` 。

然后登陆github镜像仓库
``` shell
# 使用创建的token替换掉echo后的XXX
echo "XXX" | docker login ghcr.io -u shaowenchen --password-stdin
```
然后`cat ~/.docker/config.json`把输出内容做base64 encode，将encode结果填在下面的`.dockerconfigjson`处。
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: fuyi-ghcr-secret
  namespace: website
data:
  .dockerconfigjson: UmVhbGx5IHJlYWxseSByZWVlZWVlZWVlZWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWxsbGxsbGxsbGxsbGxsbGxsbGxsbGxsbGxsbGxsbGx5eXl5eXl5eXl5eXl5eXl5eXl5eSBsbGxsbGxsbGxsbGxsbG9vb29vb29vb29vb29vb29vb29vb29vb29vb25ubm5ubm5ubm5ubm5ubm5ubm5ubm5ubmdnZ2dnZ2dnZ2dnZ2dnZ2dnZ2cgYXV0aCBrZXlzCg
type: kubernetes.io/dockerconfigjson
```
执行`kubectl apply -f <filename>.yaml`添加拉取镜像的密钥.

如果提示没有`website`namespace，则先创建ns
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: website
```


## 部署后台
先部署MySQL服务，通过[Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)部署。
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/mysql"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```
mysql deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:8.0.28
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: xiaoman
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```
mysql service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
```
kubectl apply 

website backend deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: website-backend
  namespace: website
spec:
  selector:
    matchLabels:
      app: website-backend
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: website-backend
    spec:
      containers:
      - image: ghcr.io/fuyi-huang/website-backend:v0.0.2
        name: website-backend
        ports:
        - containerPort: 8080
          name: backend-port
      imagePullSecrets:
      - name: fuyi-ghcr-secret
```
website backend service
```yaml
kind: Service
apiVersion: v1
metadata:
  name: website-backend-service
  namespace: website
spec:
  selector:
    app: website-backend
  ports:
  # Default port used by the image
  - port: 8080
```

部署前端：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: website-frontend
  namespace: website
spec:
  selector:
    matchLabels:
      app: website-frontend
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: website-frontend
    spec:
      containers:
      - image: ghcr.io/fuyi-huang/website-frontend:v0.0.1
        name: website-frontend
        ports:
        - containerPort: 80
          name: frontend-port
      imagePullSecrets:
      - name: fuyi-ghcr-secret
```

```yaml
kind: Service
apiVersion: v1
metadata:
  name: website-frontend-service
  namespace: website
spec:
  selector:
    app: website-frontend
  ports:
  # Default port used by the image
  - port: 80
```

部署shadowsocks：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shadowsocks
  namespace: website
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shadowsocks
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: shadowsocks
    spec:
      containers:
      - image: ghcr.io/fuyi-huang/shadowsocks-libev:latest
        name: shadowsocks
        ports:
        - containerPort: 10080
          name: ss-port
      imagePullSecrets:
      - name: fuyi-ghcr-secret
```

```yaml
kind: Service
apiVersion: v1
metadata:
  name: shadowsocks-service
  namespace: website
spec:
  selector:
    app: shadowsocks
  ports:
  # Default port used by the image
  - port: 10080
```

## 部署ingress
创建ingress的域名证书
```shell
kubectl create secret tls tls-secret --cert=path/to/tls.cert --key=path/to/tls.key  --namespace=website
```

部署ingress controller:
一般有一下集中部署方式：
  * 在云上部署，一般云厂商都有提供Load Balancer，部署的ingress controller会自动生成一个LB IP，使用该IP提供对k8s集群外服务的能力。
  * 自建的k8s集群没有自带的LB，可以通过部署[metallb](https://metallb.universe.tf/), metallb提供Load Balancer能力，达到和上一点接近的效果。
  * 通过NodePort方式部署ingress controller，来暴露k8s内部的服务给外部访问。

这里我使用NodePort的方式部署：
```shell
# 下载nginx ingress controller node port 部署文件：
wget  https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/baremetal/deploy.yaml
```

这样只有一台worker节点部署了ingress controller，虽然也可以使用，在nginx ingress controller 的启动参数里spec.template.spec.containers[*].args里加上之前建的TLS密钥`--default-ssl-certificate=website/tls-secret`

```yaml
spec:
  containers:
  - args:
    - /nginx-ingress-controller
    - --election-id=ingress-controller-leader
    - --controller-class=k8s.io/ingress-nginx
    - --ingress-class=nginx
    - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
    - --validating-webhook=:8443
    - --validating-webhook-certificate=/usr/local/certificates/cert
    - --validating-webhook-key=/usr/local/certificates/key
    - --default-ssl-certificate=website/tls-secret
```
参考：https://kubernetes.github.io/ingress-nginx/user-guide/tls/

以daemonset部署nginx-ingress-controller https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  minReadySeconds: 0
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/name: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
    spec:
      hostNetwork: true
      tolerations:
      # these tolerations are to have the daemonset runnable on control plane nodes
      # remove them if your control plane nodes should not run pods
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - args:
        - /nginx-ingress-controller
        - --election-id=ingress-controller-leader
        - --controller-class=k8s.io/ingress-nginx
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        - --default-ssl-certificate=website/tls-secret
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: LD_PRELOAD
          value: /usr/local/lib/libmimalloc.so
        image: k8s.gcr.io/ingress-nginx/controller:v1.2.0@sha256:d8196e3bc1e72547c5dec66d6556c0ff92a23f6d0919b206be170bc90d5f9185
        imagePullPolicy: IfNotPresent
        lifecycle:
          preStop:
            exec:
              command:
              - /wait-shutdown
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        name: controller
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          name: https
          protocol: TCP
        - containerPort: 8443
          name: webhook
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 100m
            memory: 90Mi
        securityContext:
          allowPrivilegeEscalation: true
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - ALL
          runAsUser: 101
        volumeMounts:
        - mountPath: /usr/local/certificates/
          name: webhook-cert
          readOnly: true
      dnsPolicy: ClusterFirst
      nodeSelector:
        kubernetes.io/os: linux
      serviceAccountName: ingress-nginx
      terminationGracePeriodSeconds: 300
      volumes:
      - name: webhook-cert
        secret:
          secretName: ingress-nginx-admission
```

部署ingress: 替换`www.example.com`为正确的域名
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: website-ingress
  namespace: website
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
      - www.example.com
    secretName: tls-secret
  rules:
  - host: www.example.com
    http:
      paths:
      - path: /go
        pathType: Prefix
        backend:
          service:
            name: website-backend-service
            port:
              number: 8080

  - host: www.example.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: website-frontend-service
            port:
              number: 80

  - host: www.example.com
    http:
      paths:
      - pathType: Prefix
        path: "/ss"
        backend:
          service:
            name: shadowsocks-service
            port:
              number: 10080
```
如果遇到下面的错误
```
Error from server (InternalError): error when creating "ingress": Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": Post https://ingress-nginx-controller-admission.ingress-nginx.svc:443/networking/v1beta1/ingresses?timeout=10s: service "ingress-nginx-controller-admission" not found
```
尝试先关掉校验再部署试试，关掉校验的方式：`kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission`


## 参考
1. https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
2. https://docs.github.com/en/actions/publishing-packages/publishing-docker-images#publishing-images-to-github-packages
3. https://docs.github.com/en/packages/managing-github-packages-using-github-actions-workflows/publishing-and-installing-a-package-with-github-actions#upgrading-a-workflow-that-accesses-ghcrio
4. https://computingforgeeks.com/install-mirantis-cri-dockerd-as-docker-engine-shim-for-kubernetes/
5. https://kubernetes.io/zh/docs/tasks/configure-pod-container/assign-pods-nodes/
6. https://kubernetes.io/zh/docs/tasks/configure-pod-container/pull-image-private-registry/
7. https://kubernetes.github.io/ingress-nginx/user-guide/tls/
8. https://kubernetes.github.io/ingress-nginx/deploy/baremetal/
9. https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
10. https://github.com/Mirantis/cri-dockerd/issues/73
11. https://www.chenshaowen.com/blog/github-container-registry.html