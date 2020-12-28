# 步骤4 - Kubernetes Service 与 Ingress



Kubernetes 的 Service 和 Ingress 可以 AWS ELB 集成。其中，

- 对于 Kubernetes Service，可以使用 AWS 网络负载均衡器 (NLB，instance 或 IP 模式) 或 传统负载均衡器 (CLB，仅 instance 模式) 对跨 Pod 的 4 层网络流量进行负载均衡。
- 对于 Kubernetes Ingress，可以使用 AWS 应用负载均衡器 (ALB，instance 或 IP 模式) 对跨 Pod 的 7 层网络流量进行负载均衡。

对于 Kubernetes Service 的 4 层网络流量，当使用 instance 模式将流量负载均衡到目标时，Kubernetes in-tree 的负载均衡控制器会负责创建 NLB (instance 模式) 或 CLB，不需要额外部署任何控制器，只需要将 Service Type 设置为 LoadBalancer 即可。

如果 Service 要使用 NLB - IP 模式 或者 使用 ALB 实现 7 层网络流量的转发，则需要部署 AWS Load Balancer Controller，由这个控制器来实现负载均衡器的创建和配置。

[AWS Load Balancer Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) 是一个控制器，可帮助管理 Kubernetes 集群的 Elastic Load Balancer。

- 它通过部署和配置 Application Load Balancers - ALB 来提供 Kubernetes Ingress 资源。
- 它通过部署和配置 Network Load Balancers - NLB 来提供 Kubernetes Service 资源。



在本实验中，我们会部署 AWS Load Balancer Controller，并实现 Kubernetes Service 和 Ingress 资源与 AWS NLB/ALB 的集成。

## Kubernetes in-tree Load Balancer Controller

在部署 AWS Load Balancer Controller 之前，我们可以查看在 步骤 2 创建 EKS 集群 时 创建的 nginx service。该 Kubnernetes Service 使用的是 Kubernetes in-tree 的负载均衡控制器，Service Type 为 LoadBalancer，并通过 annotation 指定负载均衡器类型为 NLB。

```bash
[ec2-user@ip-172-31-19-174 workspace]$ kubectl describe svc service-nginx                                                                                                                                    
Name:                     service-nginx
Namespace:                default
Labels:                   <none>
Annotations:              service.beta.kubernetes.io/aws-load-balancer-type: nlb
Selector:                 app=nginx
Type:                     LoadBalancer
IP:                       10.100.186.111
LoadBalancer Ingress:     a40ddad4121f74da6bee4dacbf056e87-0dcb806a4efcaaf2.elb.cn-north-1.amazonaws.com.cn
Port:                     <unset>  28080/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31324/TCP
Endpoints:                192.168.68.252:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

创建 Service 的 yaml 文件如下：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: "service-nginx"
  annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 28080
    targetPort: 80
```

通过 AWS Console 可查看该 NLB 的目标组类型为 instance。

![image-lb-controller-nlb-instance](images/image-lb-controller-nlb-instance.jpg)



接下来我们部署  AWS Load Balancer Controller 并实现  NLB - IP 模式 以及 与 ALB 的集成。



## 部署 AWS Load Balancer Controller

### 创建 IAM 策略

下载中国区 IAM 策略文件

```bash
curl -o lb_controller_iam_policy_cn.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/install/iam_policy_cn.json
```

如果无法连接到 raw.githubusercontent.com 下载，可使用 resources/aws-loadbalancer-controller 下面的策略文件 lb_controller_iam_policy_cn.json

进入文件所在目录，使用策略文件创建 IAM 策略

```bash
aws iam create-policy \
 --policy-name AWSLoadBalancerControllerIAMPolicy \
 --policy-document file://lb_controller_iam_policy_cn.json
```

记录返回的 策略 ARN，示例如下

<img src="images/image-lb-controller-irsa-1.jpg" alt="image-lb-controller-irsa-1" style="zoom:50%;" />



### 创建 IAM role 和 service account

创建 IAM role 和 Kubernetes service account，以及相应的 cluster role 和 cluster role binding 供 AWS Load Balancer Controller 后续使用。执行如下命令，注意将 cluster 名称和 policy-arn 替换成你自己的值。

```bash
eksctl create iamserviceaccount \
  --cluster=democluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws-cn:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

创建完成后，可以开始部署 AWS Load Balancer Controller。以下是 eksctl 创建 service account 的输出信息。

```bash
[ℹ]  eksctl version 0.34.0
[ℹ]  using region cn-north-1
[ℹ]  1 existing iamserviceaccount(s) (kube-system/aws-node) will be excluded
[ℹ]  1 iamserviceaccount (kube-system/aws-load-balancer-controller) was included (based on the include/exclude rules)
[ℹ]  1 iamserviceaccount (kube-system/aws-node) was excluded (based on the include/exclude rules)
[!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
[ℹ]  1 task: { 2 sequential sub-tasks: { create IAM role for serviceaccount "kube-system/aws-load-balancer-controller", create serviceaccount "kube-system/aws-load-balancer-controller" } }
[ℹ]  building iamserviceaccount stack "eksctl-democluster-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
[ℹ]  deploying stack "eksctl-democluster-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
[ℹ]  created serviceaccount "kube-system/aws-load-balancer-controller"
```





### 部署 AWS Load Balancer Controller

可以通过 Helm 或者手工的方式部署，在本次实验中我们使用手工的方式。关于 Helm 部署或其他详细信息可参考 https://docs.aws.amazon.com/eks/latest/userguide/load-balancing.html#load-balancer-ip

**安装 cert-manager**

```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.2/cert-manager.yaml
```

**安装 Controller**

下载 Controller 的 manifest 文件

```bash
curl -o lb_controller_v2_1_0_full.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/install/v2_1_0_full.yaml
```

如果无法连接到 raw.githubusercontent.com 下载，可使用 resources/aws-loadbalancer-controller 下面的 yaml 文件 lb_controller_v2_1_0_full.yaml

进入 yaml 文件所在目录，编辑 lb_controller_v2_1_0_full.yaml 文件，搜索 --cluster-name 并将其值改为自己的 EKS 集群名称。另外，由于中国区没有 WAF 和 Shield，因此需要将相关参数都设置为 false，如下所示。

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: aws-load-balancer-controller
  name: aws-load-balancer-controller
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/name: aws-load-balancer-controller
  template:
    metadata:
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/name: aws-load-balancer-controller
    spec:
      containers:
        - args:
            - --cluster-name=democluster
            - --ingress-class=alb
            - --enable-shield=false
            - --enable-waf=false
            - --enable-wafv2=false
...
```



保存修改并部署 Controller

```bash
kubectl apply -f lb_controller_v2_1_0_full.yaml 
```

查看部署状态和日志

```bash
[ec2-user@ip-172-31-19-174 workspace]$ kubectl get deployment -n kube-system aws-load-balancer-controller
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   1/1     1            1           56s
[ec2-user@ip-172-31-19-174 workspace]$ kubectl logs aws-load-balancer-controller-d44584646-xrk8g -n kube-system
{"level":"info","ts":1608617322.5291483,"msg":"version","GitVersion":"v2.1.0","GitCommit":"f95827224a66cae20a1af999d8ef1d46ec462547","BuildDate":"2020-12-01T21:02:38+0000"}
{"level":"info","ts":1608617322.6316931,"logger":"controller-runtime.metrics","msg":"metrics server is starting to listen","addr":":8080"}
{"level":"info","ts":1608617322.6331828,"logger":"setup","msg":"adding health check for controller"}
{"level":"info","ts":1608617322.6332483,"logger":"controller-runtime.webhook","msg":"registering webhook","path":"/mutate-v1-pod"}
{"level":"info","ts":1608617322.6332812,"logger":"controller-runtime.webhook","msg":"registering webhook","path":"/mutate-elbv2-k8s-aws-v1beta1-targetgroupbinding"}
{"level":"info","ts":1608617322.633311,"logger":"controller-runtime.webhook","msg":"registering webhook","path":"/validate-elbv2-k8s-aws-v1beta1-targetgroupbinding"}
{"level":"info","ts":1608617322.6333928,"logger":"setup","msg":"starting podInfo repo"}
{"level":"info","ts":1608617324.6335676,"logger":"controller-runtime.manager","msg":"starting metrics server","path":"/metrics"}
I1222 06:08:44.634429       1 leaderelection.go:242] attempting to acquire leader lease  kube-system/aws-load-balancer-controller-leader...
I1222 06:08:44.649001       1 leaderelection.go:252] successfully acquired lease kube-system/aws-load-balancer-controller-leader
```



## Kubernetes Service 与 NLB IP 模式

创建一个新的 Service，指定使用 nlb-ip 模式。首先创建 yaml 文件如下

```bash
cat << EOF > service-nlb-ip.yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: nginx-service-nlb-ip
>   annotations:
>     service.beta.kubernetes.io/aws-load-balancer-type: nlb-ip
> spec:
>   ports:
>     - port: 28080
>       targetPort: 80
>       protocol: TCP
>   type: LoadBalancer
>   selector:
>     app: nginx
> EOF
```

部署 Service

```bash
kubectl apply -f service-nlb-ip.yaml
```

查看 Service 和 NLB 的创建情况，以及 Pod 是否以 IP 形式挂载

```bash
[ec2-user@ip-172-31-19-174 workspace]$ kubectl describe service nginx-service-nlb-ip
Name:                     nginx-service-nlb-ip
Namespace:                default
Labels:                   <none>
Annotations:              service.beta.kubernetes.io/aws-load-balancer-type: nlb-ip
Selector:                 app=nginx
Type:                     LoadBalancer
IP:                       10.100.62.155
LoadBalancer Ingress:     k8s-default-nginxser-eee9c60531-d8a49c4ed3e5bf23.elb.cn-north-1.amazonaws.com.cn
Port:                     <unset>  28080/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  32669/TCP
Endpoints:                192.168.68.252:80
Session Affinity:         None
External Traffic Policy:  Cluster
[ec2-user@ip-172-31-19-174 workspace]$ 
[ec2-user@ip-172-31-19-174 workspace]$ curl -I k8s-default-nginxser-eee9c60531-d8a49c4ed3e5bf23.elb.cn-north-1.amazonaws.com.cn:28080
HTTP/1.1 200 OK
Server: nginx/1.19.6
Date: Tue, 22 Dec 2020 10:09:45 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 15 Dec 2020 13:59:38 GMT
Connection: keep-alive
ETag: "5fd8c14a-264"
Accept-Ranges: bytes
```

在 AWS Console 上，查看 NLB 后面挂载的目标组，为 IP 模式；目标 IP 为 192.168.68.252，即上面 kubectl describe service 得到的 Endpoints 地址

<img src="images/image-lb-controller-nlb-ip.jpg" alt="image-lb-controller-nlb-ip-2" style="zoom:50%;" />



## Kubernetes Ingress 与 ALB 集成

接下来我们创建一个 Kubernetes Ingress 资源

通过以下命令下载 2048 游戏 yaml 文件

```bash
curl -o 2048_full.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/examples/2048/2048_full.yaml
```

如果无法连接到 raw.githubusercontent.com 下载，可使用 resources/aws-loadbalancer-controller 下面的 yaml 文件 2048_full.yaml

进入文件所在目录，部署 2048 游戏。也可以打开 yaml 文件查看，该文件里创建了一个 game-2048 namespace，并在这个 namespace 里创建了 deployment-2048，service-2048 和 ingress-2048。 

其中 ingress 资源的配置如下，ingress.class 为 alb，将创建一个 internet-facing 的 IP 模式的 ALB。通过 annotation 将 ALB 的侦听端口改为 28080。

有关更多的 基于 ALB 的 Ingress 的使用和配置，可参考 https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/

```yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 28080}]'
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: service-2048
              servicePort: 80
```

进入文件所在目录，部署 yaml 文件

```bash
kubectl apply -f 2048_full.yaml 
```

查看 Ingress 资源状态

```bash
[ec2-user@ip-172-31-19-174 workspace]$ kubectl describe ing -n game-2048
Name:             ingress-2048
Namespace:        game-2048
Address:          k8s-game2048-ingress2-399cd4b400-1861204396.cn-north-1.elb.amazonaws.com.cn
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /*   service-2048:80 (192.168.73.39:80,192.168.75.61:80,192.168.82.18:80 + 2 more...)
Annotations:  alb.ingress.kubernetes.io/listen-ports: [{"HTTP": 28080}]
              alb.ingress.kubernetes.io/scheme: internet-facing
              alb.ingress.kubernetes.io/target-type: ip
              kubernetes.io/ingress.class: alb
Events:
  Type    Reason                  Age   From     Message
  ----    ------                  ----  ----     -------
  Normal  SuccessfullyReconciled  33s   ingress  Successfully reconciled
```



在 AWS Console 上也可以查看 AWS Load Balancer Controller 创建的 ALB 和 目标组 信息，可以看到目标 Pod 以 IP 模式挂载到 ALB 后端，目标 IP 地址即是 service-2048 的 Backends Endpoint IP。

<img src="images/image-lb-controller-alb.jpg" alt="image-lb-controller-alb" style="zoom:50%;" />



通过 Ingress 对应的 ALB 地址访问 service-2048，在浏览器打开 ALB URL：

<img src="images/image-lb-controller-alb-2.jpg" alt="image-lb-controller-alb-2" style="zoom:50%;" />

