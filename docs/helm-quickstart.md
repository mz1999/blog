
# Helm的安装和使用

## 安装helm客户端

在`macOS`上安装很简单：

```
brew install kubernetes-helm
```

其他平台请参考[Installing Helm](https://helm.sh/docs/using_helm/#installing-helm)

## 配置RBAC

定义`rbac-config.yaml`文件，创建`tiller`账号，并和`cluster-admin`绑定：

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

执行命令：

```
$ kubectl create -f rbac-config.yaml
serviceaccount "tiller" created
clusterrolebinding "tiller" created
```

## 安装Tiller镜像

在强国环境内，需要参考[kubernetes-for-china](https://github.com/maguowei/kubernetes-for-china)，将`helm`服务端部分`Tiller`的镜像下载到集群节点上。


## 初始化helm

执行初始化命令，注意指定上一步创建的`ServiceAccount`：

```
helm init --service-account tiller --history-max 200
```

命令执行成功，会在集群中安装`helm`的服务端部分`Tiller`。可以使用`kubectl get pods -n kube-system`命令查看：

```
$kubectl get pods -n kube-system

NAME                                   READY   STATUS    RESTARTS   AGE
...
tiller-deploy-7fbf5fc745-lxzxl         1/1     Running   0          179m
```

## Quickstart

* 所有可用`chart`列表：

```
helm repo update
helm search
```

* 搜索`tomcat chart`：

```
helm search tomcat
```

* 查看`stable/tomcat`的详细信息
```
helm inspect stable/tomcat
```

`stable/tomcat`使用 `sidecar` 方式部署web应用，通过参数`image.webarchive.repository`指定`war`的镜像，不指定会部署缺省的`sample`应用。

* 安装`tomcat`：

如果是在私有化集群部署，设置`service.type`为`NodePort`：

```
helm install --name my-web  --set service.type=NodePort  stable/tomcat
```

* 测试安装效果

```
export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services my-web-tomcat)
export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT

# 访问sample应用
curl http://$NODE_IP:$NODE_PORT/sample/
```

* 列表和删除

```
helm list
helm del --purge my-web
```