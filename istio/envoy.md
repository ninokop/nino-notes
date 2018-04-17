## 初识Envoy

本文转自[初战Envoy](https://github.com/imjoey/blog/issues/26)，外加自己实践理解。

https://www.envoyproxy.io/docs/envoy/v1.5.0/api-v1/listeners/listeners#config-listener-filters

### Envoy配置

Envoy有两种启动模式，SideCar模式和类似API GateWay的部署模式。

#### Front-Proxy 

front-envoy.json是启动envoy的配置文件，service-cluster表示它的集群名。

```powershell
FROM envoyproxy/envoy:latest
RUN apt-get update && apt-get -q install -y curl
CMD /usr/local/bin/envoy -c /etc/front-envoy.json --service-cluster front-proxy
```

`listeners`表示envoy的监听器对象，一个 Envoy 进程（部署实例）可包含多个监听器，目前仅支持 TCP 类型的监听器。每个 listener 都可配置若干 filter。listener 配置可从远端的 LDS 服务获取。`filters`是必填项，

`admin`表示可以通过8001端口查看envoy本身的信息。

`cluster_manager`表示这个APIGrateWay后端的所有集群信息，就是后端所有Upstream服务信息，它可以通过CDSAPI从istio管理面获得。clusters填的是集群列表。

1. type字段表示集群内部所有Host的发现方式，支持static静态IP、strict_dns、logical_dns、original_ds或者外接SDS通过API去做集群内主机发现。
2. lbtype是指集群内的负载均衡策略，支持round_robin、least_request、ringhash、random这几种。
3. hosts是集群内的主机列表。

```json
{
  "listeners": [
    {
      "address": "tcp://0.0.0.0:80",
      "filters": [
        {
          // filter名称，必选项，目前仅支持 http_connection_manager/client_ssl_auth/
          //    echo/mongo_proxy/ratelimit/redis_proxy/tcp_proxy 几种类型
          // 
          "name": "http_connection_manager",
          // filter 详细配置，必选项
          "config": {
            // http connection manager 使用的 codec类型，必选项，
            // 支持 http1/http2/auto，大多数情况下使用 auto 即可。
            "codec_type": "auto",
            // 统计使用的前缀，必选项，后面可紧跟较多的统计数据项
            "stat_prefix": "ingress_http",
            // https://www.envoyproxy.io/docs/envoy/v1.5.0/api-v1/route_config/route_config.html#config-http-conn-man-route-table
            // 路由配置
            "route_config": {
              // 路由表，必选项
              // https://www.envoyproxy.io/docs/envoy/v1.5.0/api-v1/route_config/vhost.html#config-http-conn-man-route-table-vhost
              "virtual_hosts": [
                {
                  // 节点名称，必选项
                  "name": "backend",
                  // 匹配的域名，必选项，支持正则，
                  // "*" 表示匹配所有域名，只能有一个virtual_host 的 domains 有"*"
                  // domains 中的项在多个 virtual_host 之间必须唯一，否则配置会出错
                  "domains": ["*"],
                  // 路由项，必选项，列表类型，按照顺序匹配
                  "routes": [
                    {
                      // 指定 route 的超时时间，单位是 s，不指定时默认15s
                      "timeout_ms": 0,
                      // 路由前缀，prefix/regrex/path 三者必须指定一个
                      "prefix": "/service/1",
                      // 指定 forwardto 的 upstream 集群
                      // 如果此 route 不是redirect 类型，cluster/cluster_header/weighted_clusters必须指定一个
                      // weighted_clusters 可用于 traffic splitting（分流）
                      "cluster": "service1"
                    },
                    {
                      "timeout_ms": 0,
                      "prefix": "/service/2",
                      "cluster": "service2"
                    }

                  ]
                }
              ]
            },
            // http connection manager 专用的 filter 列表
            // https://www.envoyproxy.io/docs/envoy/v1.5.0/api-v1/http_filters/router_filter.html#config-http-filters-router-v1
            // route filter 实现 HTTP forwarding，几乎所有的 HTTP Proxy 部署方式都会用到
            "filters": [
              {
                "name": "router",
                "config": {}
              }
            ]
          }
        }
      ]
    }
  ],
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

