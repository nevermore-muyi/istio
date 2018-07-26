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

自动注入的话，需要修改apiserver的配置参数，在admission-control配置项中添加：MutatingAdmissionWebhook、ValidatingAdmissionWebhook。
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
# istioctl replace -f route/my-virtualservice.yaml
# istioctl replace -f route/my-destinationrule.yaml
即可将服务负载到version为v1，即打印"Hello there!"的服务。
```

