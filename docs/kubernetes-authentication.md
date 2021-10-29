# Kubernetes集群的身份验证

用户在访问`Kubernetes`集群的`API server`时，访问请求需要经过身份验证、授权和准入控制这三个阶段的检查，才能真正到达`API`服务，如下图所示：

![access-control-overview](./media/k8s-auth/access-control-overview.svg)

`Kubernetes`中的用户有两种类型：`service accounts` 和 `normal users`。`service accounts` 由 `Kubernetes`管理，它是`Pod`中的进程用于访问`API`服务的`account`，为`Pod`中的进程提供了一种身份标识。`normal users`是由外部系统管理，在`Kubernetes`中并没有对应的 `user` 对象，它为人类用户使用`kubectl`之类的工具访问`API`服务时提供身份标识。所有用户，不管是使用 `kubectl`、客户端lib、还是直接发起`REST`请求访问`API server`，都需要经过上述三个步骤的检查。

本文将介绍`Kubernetes`集群的身份验证，即`Kubernetes`如何确认来访者的身份。

`Kubernetes`支持[多种方式的身份验证](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)：客户端证书，Password， Plain Tokens，JWT(JSON Web Token)，HTTP basic auth等。你可以同时启用多种认证，一般建议至少使用两种：

* 为验证`normal users`身份的客户端证书方式
* 为验证`Service accounts`身份的 `JWT Tokens`方式

## 使用客户端证书进行身份验证

### 理解数字证书

非对称加密算法是证书的基础。数字签名、数字证书等一系列概念有点绕，但只要记住：**公钥用来加密**，**私钥用来签名** 就可以了。

怎么理解呢？公钥可以随意分发，谁都可以持有，如果你用私钥加密，任何持有对应公钥的人都可以解密，这样做和没加密一样，没什么意义。因此，我们需要用公钥加密，只有持有私钥的那个人才能解密。私钥之所以称为私钥，一定会私密保存，不会向其他人泄漏。同时，用私钥加密虽然没有意义，但如果别人用公钥解开了私钥加密的信息，就能够证明信息是由私钥持有者发出的，验证了信息发送者的身份，这就是数字签名。

每个人制作好自己的公钥和私钥，然后把公钥发布出去。两个人如果都有对方的公钥，就可以用对方的公钥给对方发送加密信息，同时附上用私钥加密的信息摘要作为数字签名，证明消息发送者的身份。

通过加密防止了窃听风险，通过数字签名防止了冒充风险，数字签名内的消息摘要防止了篡改风险，一起看似很完美。

等等，这里有个很重要的问题被忽略了：如何安全的将公钥发布出去？如果双方希望安全通信，最好当面交换公钥，以免被别人冒充，并且要保护好自己的电脑，避免公钥被别有用心的人替换。现实中不可能这样分发公钥，效率太低，几乎无法大规模实行。于是出现了`CA`(certificate authority)，为公钥做认证。`CA`用自己的私钥，对申请用户的公钥和一些身份信息加密，生成"数字证书"（Digital Certificate）。你现在可以用任何方式将内含公钥的数字证书发布出去，例如有客户发起请求，希望以`HTTPS`的方式访问你的WEB服务，你可以在第一次回复客户的响应中带上数字证书。客户拿到你的数字证书，用`CA`的公钥解开数字证书，安全的获得你的公钥。有了`CA`为你的数字证书背书，客户可以确定你的身份，不是有人在冒充你。

那么`CA`的公钥如何安全的分发呢？首先，证书的签发是“链”式结构，给你签发证书的`CA`，它的证书可能还是由上一级`CA`机构签发的，这样一直往上追溯，最终会到某个“根证书”。如果“根证书”是被我们信任的，那么整条“链”上的证书都可信。

![certificates in a chain](./media/k8s-auth/ca-chain.png)

其次，操作系统都内置了“受信任的根证书”。我们拿到某个证书，如果它的根证书在系统的“受信任的根证书”列表中，那么这个证书就是可信的。例如知乎的证书：

![zhihu crt](./media/k8s-auth/zhihu-crt.png)

可以看到，它的根证书是[DigiCert Global Root CA](https://www.digicert.com/digicert-root-certificates.htm)，在操作系统的“受信任的根证书”列表中能找到它：

![root ca](./media/k8s-auth/root-ca.png)

根证书是通过预装的方式完成的分发，因此安装来源不明的操作系统有风险，可能潜伏了非法的根证书。一旦被植入了非法的根证书，一整套的安全体系瞬间土崩瓦解。同时，不能随意向系统中添加可信任的根证书，你很难验证根证书的真伪，它已经是root，没人能为它做背书了。12306网站早期的根证书就不在操作系统的“受信任根证书”列表中，需要用户手工安装，在网上[引起轩然大波](https://www.williamlong.info/archives/3461.html)。最终12306在17年底的时候换成了Digicert的证书。

简单总结一下基于非对称加密算法的公钥/私钥体系，**公钥用来加密**，**私钥用来签名**，**引入`CA`保证公钥的安全分发**。你可以找`CA`签发数字证书，那么你的客户就可以根据本地“受信任的根证书”验证你的数字证书，从而确认你的身份，然后用证书内包含的公钥给你发加密的信息。同样，你也可以要求对方的数字证书，以便确认对方的身份，并给他回加密的信息。

理解了数字证书的基本原理，我们再看看`Kubernetes`中如何使用客户端证书进行身份验证。

### 数字证书在Kubernetes中的应用

`Kubernetes`各组件之间的通信都是基于`TLS`，实现服务的加密访问，同时支持基于证书的双向认证。

我们在搭建私有`Kubernetes`集群时，一般是自建`root CA`，因为参与认证的所有集群节点，包括远程访问集群的客户端桌面都完全由自己控制，我们可以安全的将根证书分发到所有节点。有了`CA`，我们再用`CA`的私钥/公钥为各个组件签发所需的证书。

`CA`的创建，以及一系列客户端、服务端证书的签发，实际上是建立了`Kubernetes`集群的[PKI(Public key infrastructure)](https://en.wikipedia.org/wiki/Public_key_infrastructure)。

`Kubernetes`中的组件比较多，所以需要的证书会非常多，[这篇文档](https://kubernetes.io/docs/setup/certificates/)做了介绍。我按证书的用途归类总结一下：

* `CA`证书
  * `Kubernetes` 一般用途
  * `etcd` 集群根证书
  * [aggregation 相关功能](https://kubernetes.io/docs/tasks/access-kubernetes-api/configure-aggregation-layer/)
* 服务端证书
  * `API server`
  * `etcd`
  * `kubelet`
* 访问`API server`时进行身份验证的客户端证书
  * `kubelet`　　　　　　-> `API server`
  * `controller-manager` -> `API server`
  * `kube-scheduler` 　　-> `API server`
  * `admin`用户　　　　　-> `API server`
* `API server` 访问其他组件时进行身份验证的客户端证书
  * `API server` -> `etcd`
  * `API server` -> `kubelet`
  * `API server` -> `aggregated API server`
* `etcd` 相关功能
  * `etcd` 集群中节点互相通信使用的客户端证书
  * 如果`etcd`是以Pod方式运行，针对`etcd`的 Liveness 需要的客户端证书
* `Service accounts` 私钥/公钥对，用于生成`Service accounts`身份验证的 `JWT Tokens`

最后一个不是证书，不过也在`Kubernetes PKI`的管理范围。关于`Service accounts` 私钥/公钥对的作用，后面会讲到。

理论上`CA`根证书可以只使用一个，不过为了安全和方便管理，官方强调在不同的上下文最好使用不同的`CA`：

> **Warning**: Do not reuse a CA that is used in a different context unless you understand the risks and the mechanisms to protect the CA’s usage.

可以看出，`API server`是核心组件，其他组件、包括admin用户对它的访问都需要`TLS`双向认证，所以会有`API server`的服务端证书和各个组件的客户端证书。`API server`作为客户端需要访问`etcd`、`kubelet`和`aggregated API server`，所以也会有相应的服务端、客户端证书。

当我们使用`kubeadm`安装`Kubernetes`时，`kubeadm`会为我们生成上述的一系列私钥和证书，放在`/etc/kubernetes/`目录下：

```
# tree --dirsfirst /etc/kubernetes/
/etc/kubernetes/
├── manifests                         组件的配置文件，以Pod方式运行在集群中
│   ├── etcd.yaml
│   ├── kube-apiserver.yaml
│   ├── kube-controller-manager.yaml  
│   └── kube-scheduler.yaml           
├── pki                               
│   ├── etcd                          
│   │   ├── ca.crt                    etcd 集群CA证书
│   │   ├── ca.key                    etcd 集群CA私钥
│   │   ├── healthcheck-client.crt    Liveness 健康检查使用的客户端证书
│   │   ├── healthcheck-client.key
│   │   ├── peer.crt                  etcd节点间通信使用的客户端证书
│   │   ├── peer.key                  
│   │   ├── server.crt                etcd服务端证书
│   │   └── server.key
│   ├── apiserver.crt                 API Server 服务端证书
│   ├── apiserver.key
│   ├── apiserver-etcd-client.crt     API server -> etcd
│   ├── apiserver-etcd-client.key
│   ├── apiserver-kubelet-client.crt  API server -> kubelet
│   ├── apiserver-kubelet-client.key
│   ├── ca.crt                        CA证书
│   ├── ca.key                        CA私钥
│   ├── front-proxy-ca.crt            aggregation 相关功能CA证书
│   ├── front-proxy-ca.key            aggregation 相关功能CA私钥
│   ├── front-proxy-client.crt        API server -> aggregated API server
│   ├── front-proxy-client.key
│   ├── sa.key                        Service accounts 私钥
│   └── sa.pub                        Service accounts 公钥
├── admin.conf                        admin              -> API server 
├── controller-manager.conf           controller-manager -> API server 
├── kubelet.conf                      kubelet　　　　　　  -> API server 
└── scheduler.conf                    kube-scheduler     -> API server 
```

注意，最后四个`*.conf`是[kubeconfig file](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)，内容包含了集群、用户、namespace等信息，还有用来认证的CA证书、客户端证书和私钥。例如`admin.conf`就是`kubectl`访问集群用到的`kubeconfig file`，缺省情况下`kubectl`会使用`$HOME/.kube/config`，你也可以通过`KUBECONFIG`环境变量，或`kubectl` 的 `--kubeconfig` 参数进行设置。

`kubelet`运行在每个工作节点，无法提前预知 `node` 的 `IP` 信息，所以 `kubelet` 一般不会明确指定服务端证书, 而是只指定 `CA` 根证书, 让 `kubelet` 根据本机信息自动生成服务端证书，保存到配置参数指定的[--cert-dir](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)目录中。`cert-dir`的缺省值是`/var/lib/kubelet/pki`。

```
# tree /var/lib/kubelet/pki

/var/lib/kubelet/pki
├── kubelet-client-2019-04-28-10-48-13.pem
├── kubelet-client-current.pem -> /var/lib/kubelet/pki/kubelet-client-2019-04-28-10-48-13.pem
├── kubelet.crt   kubelet服务端证书
└── kubelet.key   kubelet服务端私钥
```

另外，`API server`有很多认证相关的[启动参数](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/#options)，参数名称让人容易混淆，有人还专门提了[issue](https://github.com/kubernetes/kubernetes/issues/54665)，[这个回答](https://github.com/kubernetes/kubernetes/issues/54665#issuecomment-340960398)根据用途对这些参数进行了分类，说明的非常清晰。

### API server 如何用客户端证书进行身份验证

前面提到，当用户使用`kubectl`访问`API server`时，需要以某种方式进行身份验证，最常用的方式就是使用客户端证书。`Kubernetes`是没有 `user` 这种 API 对象，`kubectl`的用户身份信息就包含在客户端证书中。`API server`验证了客户端证书，也就可以从证书中获得用户名和所属的`group`。

我们以`/etc/kubernetes/admin.conf` 为例，看看客户端证书中提供了那些信息。 

先查看`admin.conf`文件的内容：

```
$ kubectl --kubeconfig /etc/kubernetes/admin.conf  config view

apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.18.100.90:6443
  name: test
contexts:
- context:
    cluster: test
    user: kubernetes-admin
  name: kubernetes-admin@test
current-context: kubernetes-admin@test
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

这个文件提供了`API server`的地址，以及身份验证用到的证书：

* `certificate-authority-data` ：CA证书
* `client-certificate-data` ：客户端证书
* `client-key-data`： 客户端私钥

客户端证书是以`base64`编码的方式保存在`client-certificate-data`字段中，我们将证书提取出来：

```
# cat /etc/kubernetes/admin.conf | grep client-certificate-data | cut -d " " -f 6 | base64 -d > admin.crt
# cat admin.crt

-----BEGIN CERTIFICATE-----
MIIC8jCCAdqgAwIBAgIIaBuxevPYGaswDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
AxMKa3ViZXJuZXRlczAeFw0xOTA0MjgwMjQwMDBaFw0yMDA0MjcwMjQwMDNaMDQx
FzAVBgNVBAoTDnN5c3RlbTptYXN0ZXJzMRkwFwYDVQQDExBrdWJlcm5ldGVzLWFk
bWluMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvRTIQ4YEMh0mUWKP
17pLdUocgZMLVCK6tYmj0DJIihRk+wKvNzYSStfxsug9nnEqVVzmbW5/UxER776H
844y/1NGk/8LsDIkFGspf3cEmQ8OE8TlLNW7h9gWIGymLQ/K1qhYfNOPDYoJXPix
eUWTJgn0+neJNbJ3JoJk2WRlDFwbE0uXgYYczuDcJablSdbb8Oc+E4qJ1U7u9YMN
Bo/JBY68wYtdjXHl6Mg28aCioVZrs5eZWkNzNpXMVjQwFdZAdWbnS3OJGN1b6IrV
gWk9PMoCE2TtFv5NdlHSYFtEAaBEwfl3/D3rGHKb4ZH/fgKWsepy8ffxxibM6pND
pLnmAwIDAQABoycwJTAOBgNVHQ8BAf8EBAMCBaAwEwYDVR0lBAwwCgYIKwYBBQUH
AwIwDQYJKoZIhvcNAQELBQADggEBALPYorv2mlXyu6jpX/6gE1kTvpPGK4vylfSY
9jl4PtQZgRaXvmVsUKpyIdtVhdMZp9EFaYNC4AYkqaVEOoAbU96SYhdO/6h7Rn8T
0Ae+f1Vwt+8GxErEN3xp4noHfXM0eSEuFLPXt43BBJInYRyx1J0urAjYtNCvc9wX
uQFVmNKsqgmjvHQsRkvKcb8HEzcaD1TqqnTpq3usGjNggVZFTChB58R909yGPEXL
n7VsilmN86gom3fgqwCn2C00iKcuzCOwYN2T+Mi8KI2DraDDoVeRMSaYQNUfKNIX
Ngeod/C4piq+OAdyrPPFEINdLi404EYHyod0CgiD6uhoX5W06O4=
-----END CERTIFICATE-----
```

使用`openssl`查看证书内容：

```
# openssl x509 -in ./admin.crt -text

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 7501784745950845355 (0x681bb17af3d819ab)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Apr 28 02:40:00 2019 GMT
            Not After : Apr 27 02:40:03 2020 GMT
        Subject: O=system:masters, CN=kubernetes-admin
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
......
```

注意这一行：

```
Subject: O=system:masters, CN=kubernetes-admin
```

`API server`会将证书中`CN(Common Name)`作为用户名，`O(Organization)`作为用户所属的`group`。`API server`从这个证书得到的信息是：admin用户所属的group是`system:masters`。至此，身份验证阶段完成。

下一步是授权检查，也就是检查用户有没有权限执行这个操作。这是另外一个[话题](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)，本文不做详细讨论，只是简单介绍一下。根据[官方文档](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)，`Kubernetes`提供了缺省的 `ClusterRole - group` 绑定关系，已经将`system:masters` group 和 角色 `cluster-admin`绑定到了一起：

```
# kubectl get clusterrolebindings  cluster-admin -o yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2019-04-28T02:40:17Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
  resourceVersion: "98"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/cluster-admin
  uid: f58a87a7-695e-11e9-91ca-005056ac1c1c
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:masters
```

这个绑定关系的意思是，属于`system:masters`group的用户，都拥有`cluster-admin`角色包含的权限。我们再看看角色`cluster-admin`的具体权限信息：

```
# kubectl get clusterrole cluster-admin -o yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2019-04-28T02:40:17Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
  resourceVersion: "44"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/cluster-admin
  uid: f5523c21-695e-11e9-91ca-005056ac1c1c
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
```

从`rules`列表可以看出，`cluster-admin`这个角色对所有资源都有无限制的操作权限。因此，使用了这个`kubeconfig file`的`kubectl`的请求就有了操控和管理整个集群的权限。

## 使用`JWT Tokens`进行身份验证

运行在`Pod`中的进程需要访问`API server`时，同样需要进行身份验证和授权检查。如何让`Pod`具有用户身份呢？通过给`Pod`指定`service account`来实现。`service account`是`Kubernetes`内置的一种 “服务账户”，它为`Pod`中的进程提供了一种身份标识。如果`Pod`没有显式的指定`service account`，系统会自动为其关联`default service account`。

我们自己创建`service account`对象非常简单：

```
//serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-example

# kubectl apply -f serviceaccount.yaml
serviceaccount/nginx-example created
```

查看刚刚创建的`service account`：

```
# kubectl describe serviceaccounts nginx-example

Name:                nginx-example
Namespace:           default
Labels:              <none>
Annotations:         kubectl.kubernetes.io/last-applied-configuration:
                       {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"name":"nginx-example","namespace":"default"}}
Image pull secrets:  <none>
Mountable secrets:   nginx-example-token-r2cv6
Tokens:              nginx-example-token-r2cv6
Events:              <none>
```

当`service account`对象创建成功，`controller-manager`会发现这个新对象，然后为它生成`token`。`token`实际上是`secret`对象，内部包含了用来身份验证的`token`。`service account`对象的`Tokens`列引用的就是`controller-manager`为它创建的`token`。

我们来看看`token`的内容：

```
# kubectl get secrets nginx-example-token-r2cv6 -o yaml

apiVersion: v1
data:
  ca.crt: (APISERVER'S CA BASE64 ENCODED)
  namespace: ZGVmYXVsdA==
  token: (BEARER TOKEN BASE64 ENCODED)
kind: Secret
metadata:
  ......
type: kubernetes.io/service-account-token
```

可以看到，这个`secret`对象的`type`是`service-account-token`，包含了三部分数据：

* `ca.crt`： `API Server`的`CA`证书，用于`Pod`中的进程访问`API Server`时对服务端证书进行校验
* `namespace`： `secret`所在`namespace`，使用了`base64`编码
* `token`：`JWT Tokens`
 
`JWT Tokens` 是 `controller-manager` 用 `service account`私钥`sa.key`签发的，其中包含了用户的身份信息，`API Server`可以用`sa.pub`验证`token`，拿到用户身份信息，从而完成身份验证。

如果是使用`kubeadm`安装的`Kubernetes`，我们可以在`/etc/kubernetes/manifests/`目录中的配置文件确认`sa.key`和`sa.pub`的作用：

```
# cat /etc/kubernetes/manifests/kube-controller-manager.yaml
...
spec:
  containers:
  - command:
    - kube-controller-manager
    ...
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
...

# cat /etc/kubernetes/manifests/kube-apiserver.yaml
...
spec:
  containers:
  - command:
    - kube-apiserver
    ...
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
...
```

`controller-manager`用私钥`sa.key`签名，`API Server`用公钥`sa.pub`验签。

运行在`Pod`中的进程在向`API server`发起`HTTP`请求时，`HTTP header`中会携带`token`，`API server`从`header`中拿到`token`，进行身份验证：

```
Authorization: Bearer [token]
```

`JWT Tokens`的是由用`.`分割的三部分组成：

* Header
* Payload
* Signature

因此，一个`JWT Tokens`看起来是这样的：

```
xxxxxxx.yyyyyyyy.zzzzzzz
```

`Header`和`Payload`都是`base64`编码的`JSON`，以上面`nginx-example`关联的`token`为例，看看`Header`和`Payload`的内容：

```
$ kubectl describe secrets nginx-example-token-r2cv6 | grep token: | cut -d " " -f 7
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6Im5naW54LWV4YW1wbGUtdG9rZW4tcjJjdjYiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoibmdpbngtZXhhbXBsZSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjBmZWRlOWM1LTc2YjUtMTFlOS05MWNhLTAwNTA1NmFjMWMxYyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0Om5naW54LWV4YW1wbGUifQ.VOjVQBr5PKg1WyZYtIW0Fos5fxFN4cYE3Mz9p1eWbQP6rQRQGDEiGX-LBuM6ECI9cpSL-F4nYQAL9vmIlA4vbAgS4OFgC4nwu8SzLu2FVeE7RDpguvsdAsj-4T_LxEGX1RPljGTpvlt8HRjTnp9K8W4dy7PyJQEB5XvCf-IVNAs3zESgmuJ7wJwO7mXQe5WdeqhI5vXjcZiXP97oH0VRYT1vTKVP-GooC5YfaNhU7rHoJ0gmR10xNqZjwKGsHKkq5maC5BOrXFLlHRqVRwm9-hRn-ZLgAoCwujCIpLvPaFUR8HaatzX4GQ_HWev2soJnk1qcav0smxfjC-fu540vZA

$ kubectl describe secrets nginx-example-token-r2cv6 | grep token: | cut -d " " -f 7 | cut -d "." -f 1 | base64 -d | python -mjson.tool
{
    "alg": "RS256",
    "kid": ""
}

$ kubectl describe secrets nginx-example-token-r2cv6 | grep token: | cut -d " " -f 7 | cut -d "." -f 2 | base64 -d | python -mjson.tool
{
    "iss": "kubernetes/serviceaccount",
    "kubernetes.io/serviceaccount/namespace": "default",
    "kubernetes.io/serviceaccount/secret.name": "nginx-example-token-r2cv6",
    "kubernetes.io/serviceaccount/service-account.name": "nginx-example",
    "kubernetes.io/serviceaccount/service-account.uid": "0fede9c5-76b5-11e9-91ca-005056ac1c1c",
    "sub": "system:serviceaccount:default:nginx-example"
}
```

`Header`中的`alg`指明了签名用到的加密算法，`Payload` 中包含了用户的身份信息，可以知道这个`service account`属于的`namespace`为`default`，名称为`nginx-example`。

第三部分`Signature`的构造方式如下，如果加密算法选择了PKCS SHA：

```
PKCSSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

`controller-manager`用`sa.key`签名，`API Server`用公钥`sa.pub`验签，进行身份验证。

如果先深入了解`JWT(JSON Web Token)`，建议阅读这篇文档[<JWT: The Complete Guide to JSON Web Tokens>](https://blog.angular-university.io/angular-jwt/)

`Pod`中的进程如何获得这个`token`呢？`Kubernetes`在创建`Pod`时，会将`service account token`映射到`Pod`的`/var/run/secrets/kubernetes.io/serviceaccount` 目录下。我们通过一个例子演示一下。

创建`Pod`：

```
// simple.yaml
apiVersion: v1
kind: Pod
metadata:
    name: firstpod
spec:
    serviceAccountName: nginx-example
    containers:
    - image: nginx
      name: stan

$ kubectl apply -f simple.yaml
pod/firstpod created
```
查看`Pod`内`/var/run/secrets/kubernetes.io/serviceaccount`目录的内容：

```
$ kubectl exec firstpod -- ls  /var/run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token

```

查看`Pod`内文件`/var/run/secrets/kubernetes.io/serviceaccount/token`的内容：

```
$ kubectl exec firstpod -- cat  /var/run/secrets/kubernetes.io/serviceaccount/token | cut -d "." -f 2 | base64 -d | python -mjson.tool
{
    "iss": "kubernetes/serviceaccount",
    "kubernetes.io/serviceaccount/namespace": "default",
    "kubernetes.io/serviceaccount/secret.name": "nginx-example-token-r2cv6",
    "kubernetes.io/serviceaccount/service-account.name": "nginx-example",
    "kubernetes.io/serviceaccount/service-account.uid": "0fede9c5-76b5-11e9-91ca-005056ac1c1c",
    "sub": "system:serviceaccount:default:nginx-example"
}
```

可以看到，映射进`Pod`中的`token`，正是我们在配置中通过`serviceAccountName`指定的`nginx-example`。`Pod`中的进程可以通过访问文件`/var/run/secrets/kubernetes.io/serviceaccount/token`拿到`token`。

如何为`service account`授权？通过定义`service account`和`role`的绑定完成。本文简单演示一下，详细的说明参加[官方文档](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)。

创建`role`：

```
// example-role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

$ kubectl apply -f example-role.yaml
role.rbac.authorization.k8s.io/example-role created
```

`role`可以理解为一组权限的集合，例如上面创建的`example-role`对`default` Namesapce内的`Pods`有`get`、`watch`和`list`操作权限。

下一步就是将`service account`和`role`进行绑定：

```
//example-rolebinding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount
  name: nginx-example
  namespace: default
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io

$ kubectl apply -f example-rolebinding.yaml
rolebinding.rbac.authorization.k8s.io/example-rolebinding created
```

通过绑定的创建，`service account`就拥有了`role`定义的权限。

## 总结

用户对`API server`的访问需要通过身份验证、授权和准入控制这三个阶段的检查。

一般集群外部用户访问`API Server`使用客户端证书进行身份验证。`Kubernetes`各组件之间的通信都使用了`TLS`加密传输，同时支持基于证书的双向认证。因此`Kubernetes`的安装过程涉及很多证书的创建，本文分类介绍了这些证书的作用。

集群内`Pod`中的进程访问`API server`时，使用`service account`关联的`token`进行身份验证。

每个`Pod`都会关联一个`service account`，没有明确指定时使用`default`。当我们创建`service account`对象，`controller-manager`会为这个`service account`生成`Secret`，内部包含了用来身份验证的`JWT Tokens`。`Kubernetes`会将`token`文件`mount`到`Pod`的`/var/run/secrets/kubernetes.io/serviceaccount/token`，`Pod`内的进程在向`API server`发起的`HTTP`时，就可以在请求头中携带这个`token`。