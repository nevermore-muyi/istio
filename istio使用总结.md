# istio使用总结

#### 安装

```
istio的安装比较简单，官网有具体的安装步骤，以0.8.0为例，手动下载安装：
# curl -L https://git.io/getLatestIstio | sh -
# cd istio-0.8.0
# export PATH=$PWD/bin:$PATH
# kubectl apply -f install/kubernetes/istio-demo.yaml  （非认证）
确定安装
# kubectl get svc -n istio-system
# kubectl get pods -n istio-system

至此，安装完成。
```

#### 注入

```
官网有bookinfo的实例，非常好用，这里我自己作了一个例子，见demo目录
其中调用顺序为zb-->za2，入口时zb微服务，za2负载了za和zc为服务，za版本为v1，zc为v3。
需要注意的是，我这里使用的手动注入，所以创建deployment的时候，需要使用istioctl转化：
# kubectl create -f <(istioctl kube-inject -f deploy/za-deploy.yaml)
# kubectl create -f <(istioctl kube-inject -f deploy/zb-deploy.yaml)
# kubectl create -f <(istioctl kube-inject -f deploy/zc-deploy.yaml)
# kubectl create -f za2-svc.yaml
# kubectl create -f zb-svc.yaml

自动注入的话，需要修改apiserver的配置参数，在admission-control配置项中添加：MutatingAdmissionWebhook、ValidatingAdmissionWebhook，并且需要给namespace加上label istio-injection=enabled。
```

#### 查看

```
可以通过servicegraph或者tracing微服务查看信息，将其设置为NodePort，注意的是servicegraph访问时，需要带上后缀/dotviz。
```

#### 注意点

```
1.每个Pod只能关联一个Service；
2.Service端口要有特定的name，如http、http2、grpc、mongo、redis等；
3.label中必须带有app开头的key。
```

#### 路由配置

##### 简单路由配置

```
可以配置简单的将负载分发到某个服务上
# istioctl create -f route/my-virtualservice.yaml
# istioctl create -f route/my-destinationrule.yaml
即可将服务负载到version为v1，即打印"Hello there!"的服务。
```

##### 权重配置

```
可以按照version进行权重配置
# istioctl replace -f route/my-destinationrule-weight-10-90.yaml
# istioctl replace -f route/my-virtualservice-weight-10-90.yaml
按照9:1的权重分配，这样访问时大部分访问到的是"Hello there!"，少部分显示"i am different!"
```

##### 错误注入

```
1.Delay
# istioctl replace -f route/my-virtualservice-delay.yaml
可以看到，会有50%的概率延迟5秒之后响应

2.Abort
# istioctl replace -f route/my-virtualservice-abort.yaml
可以看到，会有50%的概率访问失败，显示"fault filter abort"
```

##### Ingress

```
1.首先获取到默认提供的gateway
# kubectl get svc istio-ingressgateway -n istio-system
2.创建规则
# istioctl create -f route/my-ingress.yaml
3.访问
浏览器通过访问ingress的pod所在ip和svc的nodeport端口访问，访问的微服务即为za2，即通过ingress的route访问到最终的service。
可以将其与Kubernetes的Ingress类比。
需要注意的是，同一个ingressgateway不要被多个gateway使用，否则会出现访问不通的情况。
```

##### Ingress-Https

```
1.生成证书：
# git clone https://github.com/nicholasjackson/mtls-go-example
# cd mtls-go-example
# ./generate.sh <url> <password>
2.创建Secret
# kubectl create -n istio-system secret tls istio-ingressgateway-certs --key 3_application/private/<..pem> --cert 3_application/certs/<..pem>
3.Gateway、VirtualService创建
生成的密钥等文件必须放在/etc/istio/ingressgateway-certs目录下，参考官网
TLS有两种形式，SIMPLE和MUTUAL，即单向和双向。
```

##### Egress

```
默认情况下，启用了Istio的服务在Pod内部curl某些网站不通，比如curl https://www.baidu.com，但是通过创建egress规则后，
# istioctl create -f route/my-egress.yaml
在Pod内部再去curl，可以返回数据。
除了创建ServiceEntry，还可以通过直接配置global.proxy.includeIPRanges参数。
```

##### Circuit Breaking

```
Circuit Breaking即断路器，可以对请求做一些处理，主要通过创建DestinationRule控制访问策略，如maxConnections、connectionPool等。
```

##### Mirror

```
可以通过mirror功能将负载到一个服务的流量同时负载另外一个服务中。
使用场景：将生产环境版本的数据同时作为测试的版本数据负载过来，这样升级新版本就可以拥有生产环境的数据的测试。
```

#### 参考资料

http://istio.doczh.cn/docs/reference/config/istio.networking.v1alpha3.html

https://istio.io/docs/concepts/traffic-management/rules-configuration/#injecting-faults-in-the-request-path