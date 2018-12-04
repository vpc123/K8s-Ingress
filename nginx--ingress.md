### Nginx-ingress 完全部署手册并且实验

准备工作：

k8s安装部署helm
参考链接：
1  https://github.com/vpc123/K8s_Helm/blob/master/helm%E5%9B%BD%E5%86%85%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2%E5%AE%8C%E5%85%A8%E6%89%8B%E5%86%8C.md

2  https://github.com/vpc123/Docker-VPN


##  利用 Helm部署 Ingress

Ingress 只是一个统称，其由 Ingress 和 Ingress Controller 两部分组成。Ingress 用作将原来需要手动配置的规则抽象成一个 Ingress 对象，使用 YAML 格式的文件来创建和管理。Ingress Controller 用作通过与 Kubernetes API 交互，动态的去感知集群中 Ingress 规则变化。


目前可用的 Ingress Controller 类型有很多，比如：Nginx、HAProxy、Traefik 等，我们将演示如何部署一个基于 Nginx 的 Ingress Controller。Nginx Ingress使用ConfigMap来管理Nginx配置，nginx是大家熟知的代理和负载均衡软件。

查找软件仓库中是否有 Nginx Ingress 包

    [root@k8s-master stable]# helm search nginx-ingress
    NAMECHART VERSION   APP VERSION DESCRIPTION 
    stable/nginx-ingress1.0.0   0.21.0  An nginx Ingress controller that uses ConfigMap to store ...
    stable/nginx-lego   0.3.1   Chart for nginx-ingress-controller and kube-lego  

#### 使用 Helm 部署 Nginx Ingress Controller

Ingress Controller 本身对外暴露的方式有几种，比如：hostNetwork、externalIP 等。这里我们采用 externalIP 的方式进行，如果你要使用 hostNetwork 方式，可以使用 controller.hostNetwork=true 参数进行设置。



    # 启用 RBAC 支持
    $ helm install --name nginx-ingress --set "rbac.create=true,controller.service.externalIPs[0]=192.168.100.211,controller.service.externalIPs[1]=192.168.100.212,controller.service.externalIPs[2]=192.168.100.213" stable/nginx-ingress
     
    NAMESPACE: default
    STATUS: DEPLOYED
     
    RESOURCES:
    ==> v1/ConfigMap
    NAME  DATA  AGE
    nginx-ingress-controller  1 2m
     
    ==> v1beta1/ClusterRole
    NAME   AGE
    nginx-ingress  2m
     
    ==> v1beta1/ClusterRoleBinding
    NAME   AGE
    nginx-ingress  2m
     
    ==> v1beta1/RoleBinding
    NAME   AGE
    nginx-ingress  2m
     
    ==> v1/ServiceAccount
    NAME   SECRETS  AGE
    nginx-ingress  12m
     
    ==> v1beta1/Role
    NAME   AGE
    nginx-ingress  2m
     
    ==> v1/Service
    NAME   TYPE  CLUSTER-IP  EXTERNAL-IP  PORT(S)   AGE
    nginx-ingress-controller   LoadBalancer  10.254.84.72192.168.100.211,192.168.100.212,192.168.100.213  80:8410/TCP,443:8948/TCP  2m
    nginx-ingress-default-backend  ClusterIP 10.254.206.175  <none>   80/TCP2m
     
    ==> v1beta1/Deployment
    NAME   DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
    nginx-ingress-controller   111   1  2m
    nginx-ingress-default-backend  111   1  2m
     
    ==> v1beta1/PodDisruptionBudget
    NAME   MIN AVAILABLE  MAX UNAVAILABLE  ALLOWED DISRUPTIONS  AGE
    nginx-ingress-controller   1  N/A  02m
    nginx-ingress-default-backend  1  N/A  02m
     
    ==> v1/Pod(related)
    NAME   READY  STATUS   RESTARTS  AGE
    nginx-ingress-controller-665cd897fc-n5rq4  1/1Running  0 2m
    nginx-ingress-default-backend-df594cfc6-pbcxk  1/1Running  0 2m
     
    NOTES:
    The nginx-ingress controller has been installed.
    It may take a few minutes for the LoadBalancer IP to be available.
    You can watch the status by running 'kubectl --namespace default get services -o wide -w nginx-ingress-controller'
     
    An example Ingress that makes use of the controller:
     
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
    annotations:
    kubernetes.io/ingress.class: nginx
    name: example
    namespace: foo
    spec:
    rules:
    - host: www.example.com
    http:
    paths:
    - backend:
    serviceName: exampleService
    servicePort: 80
    path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
    - hosts:
    - www.example.com
    secretName: example-tls
     
    If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:
     
    apiVersion: v1
    kind: Secret
    metadata:
    name: example-tls
    namespace: foo
    data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
    type: kubernetes.io/tls



部署完成后我们可以看到 Kubernetes 服务中增加了 nginx-ingress-controller 和 nginx-ingress-default-backend 两个服务。nginx-ingress-controller 为 Ingress Controller，主要做为一个七层的负载均衡器来提供 HTTP 路由、粘性会话、SSL 终止、SSL直通、TCP 和 UDP 负载平衡等功能。nginx-ingress-default-backend 为默认的后端，当集群外部的请求通过 Ingress 进入到集群内部时，如果无法负载到相应后端的 Service 上时，这种未知的请求将会被负载到这个默认的后端上。

由于我们采用了 externalIP 方式对外暴露服务， 所以 nginx-ingress-controller 会在 192.168.100.211、192.168.100.212、192.168.100.213 三台节点宿主机上的 暴露 80/443 端口。

访问 Nginx Ingress Controller

我们可以使用以下命令来获取 Nginx 的 HTTP 和 HTTPS 地址。


    $ kubectl --namespace default get services -o wide -w nginx-ingress-controller
    NAME   TYPE   CLUSTER-IP EXTERNAL-IP   PORT(S)AGE   SELECTOR
    nginx-ingress-controller   LoadBalancer   10.254.84.72   192.168.100.211,192.168.100.212,192.168.100.213   80:8410/TCP,443:8948/TCP   4happ=nginx-ingress,component=controller,release=nginx-ingress



因为我们还没有在 Kubernetes 集群中创建 Ingress资源，所以直接对 ExternalIP 的请求被负载到了 nginx-ingress-default-backend 上。nginx-ingress-default-backend 默认提供了两个 URL 进行访问，其中的 /healthz 用作健康检查返回 200，而 / 返回 404 错误。


    # 返回 200
    $ curl -I  http://192.168.100.212/healthz/
    HTTP/1.1 200 OK
    Server: nginx/1.13.8
    Date: Tue, 24 Jul 2018 06:25:58 GMT
    Content-Type: text/html
    Content-Length: 0
    Connection: keep-alive
    Strict-Transport-Security: max-age=15724800; includeSubDomains;
     
    # 返回 404
    $ curl -I  http://192.168.100.212/
    HTTP/1.1 404 Not Found
    Server: nginx/1.13.8
    Date: Tue, 24 Jul 2018 06:26:39 GMT
    Content-Type: text/plain; charset=utf-8
    Content-Length: 21
    Connection: keep-alive
    Strict-Transport-Security: max-age=15724800; includeSubDomains;
     
    # 返回 200
    $ curl -I --insecure https://192.168.100.212/healthz/
    HTTP/2 200
    server: nginx/1.13.8
    date: Tue, 24 Jul 2018 06:27:41 GMT
    content-type: text/html
    content-length: 0
    strict-transport-security: max-age=15724800; includeSubDomains;
     
    # 返回 404
    $ curl -I --insecure https://192.168.100.212/
    HTTP/2 404
    server: nginx/1.13.8
    date: Tue, 24 Jul 2018 06:28:25 GMT
    content-type: text/plain; charset=utf-8
    content-length: 21
    strict-transport-security: max-age=15724800; includeSubDomains;



在几台节点宿主机上查看，我们可以看到 ExternalIP 的 Service 是通过 Kube-Proxy对外暴露的，这里的 192.168.100.211、192.168.100.212、192.168.100.213 是三个内网 IP。 实际生产应用中是需要通过边缘路由器或全局统一接入层的负载均衡器将到达公网 IP 的外网流量转发到这几个内网 IP 上，外部用户再通过域名访问集群中以 Ingress 暴露的所有服务。

    $ sudo netstat -tlunp|grep kube-proxy|grep -E '80|443'
    tcp0  0 192.168.100.211:80  0.0.0.0:*   LISTEN  714/kube-proxy
    tcp0  0 192.168.100.211:443 0.0.0.0:*   LISTEN  714/kube-proxy
     
    $ sudo netstat -tlunp|grep kube-proxy|grep -E '80|443'
    tcp0  0 192.168.100.212:80  0.0.0.0:*   LISTEN  690/kube-proxy
    tcp0  0 192.168.100.212:443 0.0.0.0:*   LISTEN  690/kube-proxy
     
    $ sudo netstat -tlunp|grep kube-proxy|grep -E '80|443'
    tcp0  0 192.168.100.213:80  0.0.0.0:*   LISTEN  748/kube-proxy
    tcp0  0 192.168.100.213:443 0.0.0.0:*   LISTEN  748/kube-proxy


卸载 Nginx Ingress Controller

使用 Helm 卸载 Nginx Ingress Controller 非常的简单，只需下面一条指令就搞定了。

    $ helm delete --purge nginx-ingress

使用 --purge 参数可以彻底删除 Release 不留下任何记录，否则下一次部署的时候不能使用重名的 Release。

#### 部署 Ingress

接下来，我们通过 Helm 以 Ingress 方式在 Kubernetes 集群中部署两个应用。由于测试环境没有使用 PersistentVolume（持久卷，简称 PV），下面两个例子中都暂时将其关闭。有关于 PersistentVolume 的知识点，我将在后面的文章来讲一讲，敬请期待。

##### 部署 DokuWiki

DokuWiki 是一个针对小公司文件需求而开发的 Wiki 引擎，用 PHP 语言开发。DokuWiki 基于文本存储，不需要数据库。因为 DokuWiki 非常的轻量，所以我选择它来做演示。

这里我们使用 Helm 官方仓库里的 Chart 包来进行，因为阿里云镜像仓库中的很多 Chart 都不是最新的版本并且不支持以 Ingress 方式部署。

    # 从 Helm 官方 Chart 仓库迁出所有软件包
    $ git clone https://github.com/helm/charts.git

使用 helm install 进行一键部署，并通过 ingress.enabled=true 和 ingress.hosts[0].name=wiki.hi-linux.com 参数启用 Ingress 特性和设置对应的主机名。

    $ cd /home/k8s/charts/stable
    $ helm install --name dokuwiki --set "ingress.enabled=true,ingress.hosts[0].name=wiki.hi-linux.com,persistence.enabled=false" dokuwiki
     
    NAMESPACE: default
    STATUS: DEPLOYED
     
    RESOURCES:
    ==> v1beta1/Ingress
    NAME   HOSTS  ADDRESS  PORTS  AGE
    dokuwiki-dokuwiki  wiki.hi-linux.com  80   53s
     
    ==> v1/Pod(related)
    NAMEREADY  STATUS   RESTARTS  AGE
    dokuwiki-dokuwiki-747b45cddb-qt8l2  1/1Running  0 53s
     
    ==> v1/Secret
    NAME   TYPEDATA  AGE
    dokuwiki-dokuwiki  Opaque  1 54s
     
    ==> v1/Service
    NAME   TYPE  CLUSTER-IP  EXTERNAL-IP  PORT(S)   AGE
    dokuwiki-dokuwiki  LoadBalancer  10.254.235.137  <pending>80:8430/TCP,443:8848/TCP  54s
     
    ==> v1beta1/Deployment
    NAME   DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
    dokuwiki-dokuwiki  111   1  54s
     
    NOTES:
     
    ** Please be patient while the chart is being deployed **
     
    1. Get the DokuWiki URL indicated on the Ingress Rule and associate it to your cluster external IP:
     
    export CLUSTER_IP=$(minikube ip) # On Minikube. Use: `kubectl cluster-info` on others K8s clusters
    export HOSTNAME=$(kubectl get ingress --namespace default dokuwiki-dokuwiki -o jsonpath='{.spec.rules[0].host}')
    echo "Dokuwiki URL: http://$HOSTNAME/"
    echo "$CLUSTER_IP  $HOSTNAME" | sudo tee -a /etc/hosts
     
    2. Login with the following credentials
     
    echo Username: user
    echo Password: $(kubectl get secret --namespace default dokuwiki-dokuwiki -o jsonpath="{.data.dokuwiki-password}" | base64 --decode)

部署完成后，根据提示生成相应的登陆用户名和密码。

    $ echo Username: user
    Username: user
     
    $ echo Password: $(kubectl get secret --namespace default dokuwiki-dokuwiki -o jsonpath="{.data.dokuwiki-password}" | base64 --decode)
    Password: e2GrABBkwF

测试从各节点的宿主机 IP 访问应用，这里我们直接使用 Curl 命令进行访问。

    $ curl -I  http://wiki.hi-linux.com/doku.php -x 192.168.100.211:80
    HTTP/1.1 200 OK
    Server: nginx/1.13.8
    Date: Wed, 25 Jul 2018 05:16:02 GMT
    Content-Type: text/html; charset=utf-8
    Connection: keep-alive
    Vary: Accept-Encoding
    X-Powered-By: PHP/7.0.31
    Vary: Cookie,Accept-Encoding
    Set-Cookie: DokuWiki=k2clt6f2qe472ehsq6tcmh6v20; path=/; HttpOnly
    Expires: Thu, 19 Nov 1981 08:52:00 GMT
    Cache-Control: no-store, no-cache, must-revalidate
    Pragma: no-cache
    Set-Cookie: DW68700bfd16c2027de7de74a5a8202a6f=deleted; expires=Thu, 01-Jan-1970 00:00:01 GMT; Max-Age=0; path=/; HttpOnly
    X-UA-Compatible: IE=edge,chrome=1
     
    $ curl -I  http://wiki.hi-linux.com/doku.php -x 192.168.100.212:80
    HTTP/1.1 200 OK
    Server: nginx/1.13.8
    Date: Wed, 25 Jul 2018 05:18:13 GMT
    Content-Type: text/html; charset=utf-8
    Connection: keep-alive
    Vary: Accept-Encoding
    X-Powered-By: PHP/7.0.31
    Vary: Cookie,Accept-Encoding
    Set-Cookie: DokuWiki=ork8sv8qpurteblasuq3eb3nt2; path=/; HttpOnly
    Expires: Thu, 19 Nov 1981 08:52:00 GMT
    Cache-Control: no-store, no-cache, must-revalidate
    Pragma: no-cache
    Set-Cookie: DW68700bfd16c2027de7de74a5a8202a6f=deleted; expires=Thu, 01-Jan-1970 00:00:01 GMT; Max-Age=0; path=/; HttpOnly
     
    $ curl -I  http://wiki.hi-linux.com/doku.php -x 192.168.100.213:80
    HTTP/1.1 200 OK
    Server: nginx/1.13.8
    Date: Wed, 25 Jul 2018 05:18:30 GMT
    Content-Type: text/html; charset=utf-8
    Connection: keep-alive
    Vary: Accept-Encoding
    X-Powered-By: PHP/7.0.31
    Vary: Cookie,Accept-Encoding
    Set-Cookie: DokuWiki=6ulgtsddqq3rlo0mriavj64jc4; path=/; HttpOnly
    Expires: Thu, 19 Nov 1981 08:52:00 GMT
    Cache-Control: no-store, no-cache, must-revalidate
    Pragma: no-cache
    Set-Cookie: DW68700bfd16c2027de7de74a5a8202a6f=deleted; expires=Thu, 01-Jan-1970 00:00:01 GMT; Max-Age=0; path=/; HttpOnly
    X-UA-Compatible: IE=edge,chrome=1

Curl 用法很多，你也可以使用下面方式来达到相同的效果。

    $ curl -H "Host:wiki.hi-linux.com"  "http://192.168.100.211/doku.php"

当然你也可以在本地 hosts 文件中对 IP 和域名进行绑定后，通过浏览器访问该应用。

#### 部署 Minio

Minio 是一个基于 Apache License v2.0 开源协议的对象存储服务，Minio 使用 Go 语言开发，具有良好的跨平台性，同样是一个非常轻量的服务。它兼容亚马逊 S3 云存储服务接口，非常适合于存储大容量非结构化的数据，例如：图片、视频、日志文件、备份数据和容器/虚拟机镜像等。

    # 从 Helm 官方 Chart 仓库迁出所有软件包
    $ git clone https://github.com/helm/charts.git

如果你需要修改主机名，请修改 values.yaml 文件中的 hosts 的值。我想你一定觉得很奇怪，为什么在这个例子我没用使用传递参数的方式来动态修改模板中对应的值？真相只有一个，哪就是我没有找到能成功修改模板中对应的变量，惊不惊喜，意不意外呢？哈哈哈。如果你知道可以留言告诉我哟！

    $ cd /home/k8s/charts/stable
    $ cat minio/values.yaml
    hosts:
    - minio.hi-linux.com
    #- chart-example.local

使用 helm install 进行一键部署，并通过 ingress.enabled=true 参数启用 Ingress 特性。



    $ helm install --name minio  --set "ingress.enabled=true,persistence.enabled=false" minio
     
    NAMESPACE: default
    STATUS: DEPLOYED
     
    RESOURCES:
    ==> v1beta1/Ingress
    NAME   HOSTS   ADDRESS  PORTS  AGE
    minio  minio.hi-linux.com  80   1m
     
    ==> v1/Pod(related)
    NAME   READY  STATUS   RESTARTS  AGE
    minio-7c7cf49d4-gqf8p  1/1Running  0 1m
     
    ==> v1/Secret
    NAME   TYPEDATA  AGE
    minio  Opaque  2 1m
     
    ==> v1/ConfigMap
    NAME   DATA  AGE
    minio  2 1m
     
    ==> v1/Service
    NAME   TYPE   CLUSTER-IP  EXTERNAL-IP  PORT(S)   AGE
    minio  ClusterIP  None<none>   9000/TCP  1m
     
    ==> v1beta2/Deployment
    NAME   DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
    minio  111   1  1m
     
    NOTES:
     
    Minio can be accessed via port 9000 on the following DNS name from within your cluster:
    minio-svc.default.svc.cluster.local
     
    To access Minio from localhost, run the below commands:
     
    1. export POD_NAME=$(kubectl get pods --namespace default -l "release=minio" -o jsonpath="{.items[0].metadata.name}")
     
    2. kubectl port-forward $POD_NAME 9000 --namespace default
     
    Read more about port forwarding here: http://kubernetes.io/docs/user-guide/kubectl/kubectl_port-forward/
     
    You can now access Minio server on http://localhost:9000. Follow the below steps to connect to Minio server with mc client:
     
    1. Download the Minio mc client - https://docs.minio.io/docs/minio-client-quickstart-guide
     
    2. mc config host add minio-local http://localhost:9000 AKIAIOSFODNN7EXAMPLE wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY S3v4
     
    3. mc ls minio-local
     
    Alternately, you can use your browser or the Minio SDK to access the server - https://docs.minio.io/categories/17

部署完成后，我们在本地 hosts 文件中对 IP 和域名进行绑定，并通过浏览器访问该应用。

登陆用户名和密码在部署完成后的提示信息中。

最后我们在 Kubernetes 上来查看下部署成功后的 Ingress 信息。

    $ kubectl get ingress
    NAMEHOSTSADDRESS   PORTS AGE
    dokuwiki-dokuwiki   wiki.hi-linux.com  8044m
    minio   minio.hi-linux.com 8050s
    