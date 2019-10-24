# 配置istio ingress gateway作为外部服务代理

[原文链接](https://istio.io/blog/2019/proxy/) ｜原作者：VADIM EISENBERG (IBM)｜发表于：2019-10-15｜阅读时间：约5分钟



[如何控制入口流量](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/)和[如何配置不终止TLS的入口网关](https://istio.io/docs/tasks/traffic-management/ingress/ingress-sni-passthrough/)这两篇`istio`入门教学任务描述了如何在网格内配置一个入口网关，并且暴露服务给外部流量；这个服务可以是基于`HTTP`或者是`HTTPS`的；在`HTTPS`的场景下，网关直接透传流量，而不会终止`TLS`；

这篇博客文章描述了如何使用跟`istio`入口网关一样的机制来启用对外部服务的访问，而不是对网格内部应用程序的访问。 这样，整个`istio`可以充当代理服务器，并具有可观察性，流量管理和策略执行等功能价值。

这篇博客文章展示了如何配置对一个`HTTP`服务和`HTTPS`外部服务（即`httpbin.org`和`edition.cnn.com`）的访问。



## 配置入口网关

1. 首先定义一个带有服务端的入口网关，并且配置`80`和`443`端口。 确保模式被设置为`PASSTHROUGH`，在`443`端口的配置里，指示网关按原样通过入口流量，并且不终止`TLS`。

   ```bash
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: proxy
   spec:
     selector:
       istio: ingressgateway # use istio default ingress gateway
     servers:
     - port:
         number: 80
         name: http
         protocol: HTTP
       hosts:
       - httpbin.org
     - port:
         number: 443
         name: tls
         protocol: TLS
       tls:
         mode: PASSTHROUGH
       hosts:
       - edition.cnn.com
   EOF
   ```

2. 为`httpbin.org`和`edition.cnn.com`服务创建服务条目，以便其可以从入口网关访问：

   ```bash
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: httpbin-ext
   spec:
     hosts:
     - httpbin.org
     ports:
     - number: 80
       name: http
       protocol: HTTP
     resolution: DNS
     location: MESH_EXTERNAL
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: cnn
   spec:
     hosts:
     - edition.cnn.com
     ports:
     - number: 443
       name: tls
       protocol: TLS
     resolution: DNS
     location: MESH_EXTERNAL
   EOF
   ```

3. 创建服务条目并为`localhost`服务配置目标规则。 在下一步中，您需要此服务条目作为从网格内部的应用程序到外部服务的流量的目的地，以阻止来自网格内部的流量。 在此示例中，您将`istio`用作外部应用程序和外部服务之间的代理。

   ```bash
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: localhost
   spec:
     hosts:
     - localhost.local
     location: MESH_EXTERNAL
     ports:
     - number: 80
       name: http
       protocol: HTTP
     - number: 443
       name: tls
       protocol: TLS
     resolution: STATIC
     endpoints:
     - address: 127.0.0.1
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: localhost
   spec:
     host: localhost.local
     trafficPolicy:
       tls:
         mode: DISABLE
         sni: localhost.local
   EOF
   ```

4. 为每个外部服务创建一个`virtual service`以便可以路由到它， 两个`virtual service`都需要包含`proxy`配置在`gateways`节点；

   请注意`mesh`网关的`route:`部分，网关代表网格内部的应用程序。`route:`部分显示流量是如何重定向到`localhost.local`服务的，从而有效阻止了流量。

   ```bash
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin
   spec:
     hosts:
     - httpbin.org
     gateways:
     - proxy
     - mesh
     http:
     - match:
       - gateways:
         - proxy
         port: 80
         uri:
           prefix: /status
       route:
       - destination:
           host: httpbin.org
           port:
             number: 80
     - match:
       - gateways:
         - mesh
         port: 80
       route:
       - destination:
           host: localhost.local
           port:
             number: 80
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: cnn
   spec:
     hosts:
     - edition.cnn.com
     gateways:
     - proxy
     - mesh
     tls:
     - match:
       - gateways:
         - proxy
         port: 443
         sni_hosts:
         - edition.cnn.com
       route:
       - destination:
           host: edition.cnn.com
           port:
             number: 443
     - match:
       - gateways:
         - mesh
         port: 443
         sni_hosts:
         - edition.cnn.com
       route:
       - destination:
           host: localhost.local
           port:
             number: 443
   EOF
   ```

5. [开启Envoy的访问日志](https://istio.io/docs/tasks/telemetry/logs/access-log/#enable-envoy-s-access-logging)

6. 根据[定义ingress入口网关的IP和端口](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)的说明来定义`SECURE_INGRESS_PORT`和`INGRESS_HOST`环境变量

7. 根据上一步操作，通过保存在环境变量中的预先定义的`ingress IP`和端口来访问`httpbin.org`服务，通过访问`http bin.org`服务的`/status/418`路径，正常应该返回`HTTP status 418`，并且包含图形[[418 I’m a teapot](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/418)]

   ```bash
   $ curl $INGRESS_HOST:$INGRESS_PORT/status/418 -Hhost:httpbin.org
   -=[ teapot ]=-
   
      _...._
    .'  _ _ `.
   | ."` ^ `". _,
   \_;`"---"`|//
     |       ;/
     \_     _/
       `"""`
   ```

8. 如果`istio ingress gateway`部署在`istio-system` `namespace`里，通过下面的命令可以输出网关的日志

   ```bash
   $ kubectl logs -l istio=ingressgateway -c istio-proxy -n istio-system | grep 'httpbin.org'
   ```

9. 查找类似如下的日志：

   ```bash
   [2019-01-31T14:40:18.645Z] "GET /status/418 HTTP/1.1" 418 - 0 135 187 186 "10.127.220.75" "curl/7.54.0" "28255618-6ca5-9d91-9634-c562694a3625" "httpbin.org" "34.232.181.106:80" outbound|80||httpbin.org - 172.30.230.33:80 10.127.220.75:52077 -
   ```

10. 通过`ingress gateway`访问`edition.cnn.com`服务

    ```bash
    $ curl -s --resolve edition.cnn.com:$SECURE_INGRESS_PORT:$INGRESS_HOST https://edition.cnn.com:$SECURE_INGRESS_PORT | grep -o "<title>.*</title>"
    <title>CNN International - Breaking News, US News, World News and Video</title>
    ```

11. 如果`istio ingress `网关部署在`istio-system` `namespace`里，通过下面的命令可以输出网关的日志

    ```bash
    $ kubectl logs -l istio=ingressgateway -c istio-proxy -n istio-system | grep 'edition.cnn.com'
    ```

12. 查找类似如下的日志：

    ```bash
    [2019-01-31T13:40:11.076Z] "- - -" 0 - 589 17798 1644 - "-" "-" "-" "-" "172.217.31.132:443" outbound|443||edition.cnn.com 172.30.230.33:54508 172.30.230.33:443 10.127.220.75:49467 edition.cnn.com
    ```

## 清理

删除所有的`gateway`，`virtual service`以及`service entries`资源：

```bash
$ kubectl delete gateway proxy
$ kubectl delete virtualservice cnn httpbin
$ kubectl delete serviceentry cnn httpbin-ext localhost
$ kubectl delete destinationrule localhost
```

