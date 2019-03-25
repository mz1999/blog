# 在Ubuntu上安装Minikube

为了方便开发者体验`Kubernetes`，社区提供了可以在本地部署的[Minikube](https://github.com/kubernetes/minikube)。由于在强国网络环境内，无法顺利的安装使用`Minikube`，我们可以从阿里云的镜像地址来获取所需Docker镜像和配置。

* **安装VirtualBox**

`sudo apt-get install virtualbox`

* **安装 Minikube**

```
curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v0.35.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

* **启动Minikube**

```
$ minikube start --registry-mirror=https://registry.docker-cn.com

😄  minikube v0.35.0 on linux (amd64)
🔥  Creating virtualbox VM (CPUs=2, Memory=2048MB, Disk=20000MB) ...
📶  "minikube" IP address is 192.168.99.100
🐳  Configuring Docker as the container runtime ...
✨  Preparing Kubernetes environment ...
🚜  Pulling images required by Kubernetes v1.13.4 ...
🚀  Launching Kubernetes v1.13.4 using kubeadm ...
⌛  Waiting for pods: apiserver proxy etcd scheduler controller addon-manager dns
🔑  Configuring cluster permissions ...
🤔  Verifying component health .....
💗  kubectl is now configured to use "minikube"
🏄  Done! Thank you for using minikube!
```

* 检查状态

```
$ minikube status

host: Running
kubelet: Running
apiserver: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.100
```

`kubernetes`已经成功运行，可以使用`kubectl`访问集群：

```
$ kubectl get pods -n kube-system

NAME                                    READY   STATUS    RESTARTS   AGE
coredns-89cc84847-2k67h                 1/1     Running   0          18m
coredns-89cc84847-95zsj                 1/1     Running   0          18m
etcd-minikube                           1/1     Running   0          18m
kube-addon-manager-minikube             1/1     Running   0          19m
kube-apiserver-minikube                 1/1     Running   0          18m
kube-controller-manager-minikube        1/1     Running   0          18m
kube-proxy-f66hz                        1/1     Running   0          18m
kube-scheduler-minikube                 1/1     Running   0          18m
kubernetes-dashboard-7d8d567b4d-h82vx   1/1     Running   0          18m
storage-provisioner                     1/1     Running   0          18m
```

* **停止Minikube**

```
$ minikube stop

✋  Stopping "minikube" in virtualbox ...
🛑  "minikube" stopped.
```

* **删除本地集群**

```
$ minikube delete

🔥  Deleting "minikube" from virtualbox ...
💔  The "minikube" cluster has been deleted.
```