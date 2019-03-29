
# 使用kubectl管理多集群

`kubectl`会使用`$HOME/.kube`目录下的`config`文件作为缺省的配置文件。我们可以使用`kubectl config view`查看配置信息：

```
$kubectl config view

apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.18.100.90:6443
  name: cluster-1
contexts:
- context:
    cluster: cluster-1
    user: cluster-1-admin
  name: cluster-1-admin@cluster-1
current-context: cluster-1-admin@cluster-1
kind: Config
preferences: {}
users:
- name: cluster-1-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

可以看到，配置文件主要包含了`clusters`，`users`和`contexts`三部分信息。`context`是访问一个`kubernetes`集群所需要的参数集合。每个`context`有三个参数：

* `cluster`：要访问的集群信息
* `namespace`：用户工作的`namespace`，缺省值为`default`
* `user`：连接集群的认证用户
  
缺省情况下，`kubectl`会使用`current-context`指定的`context`作为当前的工作集群环境。不难想象，切换`context`就可以切换到不同的`kubernetes`集群。

在不了解`context`的概念之前，想访问不同的集群，每次都要把集群对应的`config`文件copy到`$HOME/.kube`目录下，同时要记得使用`kubectl cluster-info`确认当前访问的集群：

```
$kubectl cluster-info

Kubernetes master is running at https://172.18.100.90:6443
KubeDNS is running at https://172.18.100.90:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

在看了[这篇文档](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)后，才知道`kubectl`可以切换`context`来管理多个集群。如果你有多个集群的`config`文件，可以在系统环境变量`KUBECONFIG`中指定每个`config`文件的路径，例如：

```
export  KUBECONFIG=/home/mazhen/kube-config/config-cluster-1:/home/mazhen/kube-config/config-cluster-1
```

再使用`kubectl config view`查看集群配置时，`kubectl`会自动合并多个`config`的信息：

```
$ kubectl config view

apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.20.51.11:6443
  name: cluster-2
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.18.100.90:6443
  name: cluster-1
contexts:
- context:
    cluster: cluster-2
    user: cluster-2-admin
  name: cluster-2-admin@cluster-2
- context:
    cluster: cluster-1
    user: cluster-1-admin
  name: cluster-1-admin@cluster-1
current-context: cluster-1-admin@cluster-1
kind: Config
preferences: {}
users:
- name: cluster-2-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: cluster-1-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

可以看到，配置中包含了两个集群，两个用户，以及两个`context`。我们可以使用`kubectl config get-contexts`查看配置中所有的`context`：

```
$ kubectl config get-contexts

CURRENT   NAME                         CLUSTER      AUTHINFO           NAMESPACE
          cluster-2-admin@cluster-2    cluster-2    cluster-2-admin
*         cluster-1-admin@cluster-1    cluster-1    cluster-1-admin
```

星号`*`标识了当前的工作集群。如果想访问另一个集群，使用`kubectl config use-context`进行切换：

```
$ kubectl config use-context cluster-2-admin@cluster-2

Switched to context "cluster-2-admin@cluster-2".
```

我们可以再次确认切换的结果：

```
$ kubectl config get-contexts

CURRENT   NAME                         CLUSTER      AUTHINFO           NAMESPACE
*         cluster-2-admin@cluster-2    cluster-2    cluster-2-admin
          cluster-1-admin@cluster-1    cluster-1    cluster-1-admin

$ kubectl cluster-info

Kubernetes master is running at https://172.20.51.11:6443
KubeDNS is running at https://172.20.51.11:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
kubernetes-dashboard is running at https://172.20.51.11:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

看吧，`kubectl`切换`context`管理多集群是多么的方便。