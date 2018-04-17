## 初识Envoy概念

本文转自[初战Envoy](https://github.com/imjoey/blog/issues/26)，外加自己实践理解。

### Envoy配置

Envoy有两种启动模式，SideCar模式和类似API GateWay的部署模式。

#### front-proxy 

front-envoy.json是启动envoy的配置文件，service-cluster表示它的集群名。

```powershell
FROM envoyproxy/envoy:latest
RUN apt-get update && apt-get -q install -y curl
CMD /usr/local/bin/envoy -c /etc/front-envoy.json --service-cluster front-proxy
```

`admin`表示可以通过8001端口查看envoy本身的信息。`listeners`表示envoy的监听器对象，一个 Envoy 进程（部署实例）可包含多个监听器，目前仅支持 TCP 类型的监听器。每个 listener 都可配置若干 filter。listener 配置可从远端的 LDS 服务获取。

1. `filters`支持http_connection_manager、mongo_proxy、redis_proxy、tcp_proxy几种代理方式。
2. `codec`类型跟协议有关，`stat_prefix`跟统计项的前缀有关
3. `route_config`里面配置的是一组规则，类似ingress在HTTP层做的分发，根据path匹配请求的路由，分发到后端不同的集群。这是作为APIGrateWay时envoy的路由配置。

```json
{
  "listeners": [{
      "address": "tcp://0.0.0.0:80",
      "filters": [{
          "name": "http_connection_manager",
          "config": {
            "codec_type": "auto",
            "stat_prefix": "ingress_http",
            "route_config": {
              "virtual_hosts": [{
                  "name": "backend",
                  "domains": ["*"],
                  "routes": [{
                      "timeout_ms": 0,
                      "prefix": "/service/1",
                      "cluster": "service1"
                    }, {
                      "timeout_ms": 0,
                      "prefix": "/service/2",
                      "cluster": "service2"
                    }]
                }]
            },
            "filters": [{
                "name": "router",
                "config": {}
              }]
          }
        }]
  }],
  "admin": {
    "access_log_path": "/dev/null",
    "address": "tcp://0.0.0.0:8001"
  },
  "cluster_manager": {
    "clusters": [{
        "name": "service1",
        "connect_timeout_ms": 250,
        "type": "strict_dns",
        "lb_type": "round_robin",
        "features": "http2",
        "hosts": [{
            "url": "tcp://service1:80"
         }]
      }, {
        "name": "service2",
        "connect_timeout_ms": 250,
        "type": "strict_dns",
        "lb_type": "round_robin",
        "features": "http2",
        "hosts": [{
            "url": "tcp://service2:80"
         }]
      }]
}}
```

`cluster_manager`表示这个APIGrateWay后端的所有集群信息，就是后端所有Upstream服务信息，它可以通过CDSAPI从istio管理面获得。clusters填的是集群列表。

1. type字段表示集群内部所有Host的发现方式，支持static静态IP、strict_dns或者SDSAPI做集群发现。
2. lbtype是指集群内的负载均衡策略，支持round_robin、least_request、ringhash、random这几种。
3. hosts是访问集群的url。

> **个人理解**：cluster的含义是**a group of logically similar upstream hosts that envoy connect to**. 比如bookinfo这个示例中，reviews就是个cluster，而它包含的三个版本不同的Deployment 分别是cluster内 对外提供服务的upstream。通过集群hosts字段的url访问集群，就能按照lb策略访问到不同的Upstream。

#### SideCar





