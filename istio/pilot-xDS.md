## Pilot-xDS

pilot类比一个服务中心，它可以扩展不同后端实现服务发现。比如k8s上的服务注册是通过应用自己发布Service关联自己注册的。同时还附带配置中心的功能，比如发现路由规则的变化，通过EnvoyAPI给数据面使用。总结来说pilot提供了一组发现服务DS，包括服务发现SDS、集群发现CDS、路由规则发现RDS和监听器发现LDS。

#### SDS

> SDS可以发现cluster内的所有节点，是Envoy官方推荐的服务发现方式，因为不会因为DNS长查询而导致延时很大，还能比DNS携带更多信息，用于实现智能LB和routing。

envoy的配置文件里，通常配置了集群信息。比如下面是一个flask-app-envoy-service的集群，集群内部的hosts通过roundrobin的方式做LB，配置了集群内服务发现的方式为SDS，因此envoy通过service_name调用SDSAPI去pilot查询服务hosts。

cluster里的sds字段是全局的sds配置，通过dns找到提供SDS服务的upstream。

```json
{
  "cluster_manager": {
    "clusters": [
      {
        "name": "flask-app-envoy-service.default.svc.cluster.local",
        "connect_timeout_ms": 250,
        "type": "sds",
        "lb_type": "round_robin",
        // servce_name 就是微服务的名称，通过此名称去 sds 获取访问方式
        "service_name": "flask-app-envoy-service.default.svc.cluster.local"
      }
    ],
    "sds": {
      "cluster": {
        "name": "kubernetes-envoy-sds.kube-system",
        "type": "logical_dns",
        "connect_timeout_ms": 250,
        "lb_type": "round_robin",
        "hosts": [
          {"url": "tcp://kubernetes-envoy-sds.kube-system:80"}
        ]
      },
      "refresh_delay_ms": 1000
    }
  }
}
```

SDS服务要提供一下接口来查询特定cluster的所有服务地址，比如在istio官方例子上的details和reviews服务。其中reviews服务关联了三个Deployment，每个Dp副本为1，区别是labels里的version不同，所以可以在reviews下发现三个hosts。

```powershell
curl http://10.96.23.66:8080/v1/registration
curl http://10.96.23.66:8080/v1/registration/reviews.nino.svc.cluster.local\|http
```

```json
[{
   "service-key": "details.nino.svc.cluster.local|http",
   "hosts": [{
     "ip_address": "10.244.0.37",
     "port": 9080
   }]
  }, {
   "service-key": "reviews.nino.svc.cluster.local|http",
   "hosts": [{
     "ip_address": "10.244.0.38",
     "port": 9080
    }, {
     "ip_address": "10.244.0.41",
     "port": 9080
    }, {
     "ip_address": "10.244.0.42",
     "port": 9080
    }]
}]
```

#### CDS

一个服务队要访问的后端集群有哪些，当然可以通过配置文件去静态配置。也可以通过CDS去管理。比如bookinfo里的productpage它可以通过CDS API去获取自身的clusters信息。这个cluster_name是在envoy启动时通过`--serviceCluster`配置的，service_node也可以通过启动参数配置，不过可能是通过envoyType、PodIP、PodName和PodNamespace合成的，这些都通过ENV的方式配置在了envoy的启动环境中。

> /v1/`{:cluster_name}`/`{:service_node}`

```json
{
    "args": [ "proxy", "sidecar",
        "--configPath", "/etc/istio/proxy",
        "--binaryPath", "/usr/local/bin/envoy",
        "--serviceCluster", "productpage",
        "--drainDuration", "45s",
        "--parentShutdownDuration", "1m0s",
        "--discoveryAddress", "istio-pilot.istio-system:8080",
        "--discoveryRefreshDelay", "1s",
        "--zipkinAddress", "zipkin.istio-system:9411",
        "--connectTimeout", "10s",
        "--statsdUdpAddress", "istio-mixer.istio-system:9125",
        "--proxyAdminPort", "15000",
        "--controlPlaneAuthPolicy", "NONE"
    ],
    "env": [{
            "name": "POD_NAME",
            "valueFrom": { "fieldRef": {
                    "apiVersion": "v1",
                    "fieldPath": "metadata.name"
         }}},{
            "name": "POD_NAMESPACE",
            "valueFrom": { "fieldRef": {
                    "apiVersion": "v1",
                    "fieldPath": "metadata.namespace"
         }}},{
            "name": "INSTANCE_IP",
            "valueFrom": { "fieldRef": {
                    "apiVersion": "v1",
                    "fieldPath": "status.podIP"
         }}}
    ],
    "image": "docker.io/istio/proxy_debug:0.7.1",
    "imagePullPolicy": "IfNotPresent",
    "name": "istio-proxy",
    "resources": {},
    "securityContext": {
        "privileged": true,
        "readOnlyRootFilesystem": false,
        "runAsUser": 1337
    },
    "terminationMessagePath": "/dev/termination-log",
    "terminationMessagePolicy": "File",
    "volumeMounts": [{
            "mountPath": "/etc/istio/proxy",
            "name": "istio-envoy"
        },{
            "mountPath": "/etc/certs/",
            "name": "istio-certs",
            "readOnly": true
       }]
}
```

下面是查到的一组productpage通过CDS获取的所有cluster，发现除了它真正要访问的reviews、details、ratings还包含跟我没关系的其他namespace下的consumer和provider服务。in.9080则代表自己作为服务提供方的真实地址。

> 如果服务cluster内部是分版本的，那相当于集群还包含了另外几组集群。比如reviews就包含了reviews|version=v1和reviews|version=v2等三个cluster。最后RDS配置的路由规则就是这种内涵cluster的分发。

```json
root@blueocean:~# curl http://10.96.23.66:8080/v1/clusters/productpage/sidecar~10.244.0.40~productpage-v1-8666ffbd7c-mf5f4.nino~nino.svc.cluster.local
{
  "clusters": [{
    "name": "in.9080",
    "connect_timeout_ms": 1000,
    "type": "static",
    "lb_type": "round_robin",
    "hosts": [{
      "url": "tcp://127.0.0.1:9080"
     }]
   },{
    "name": "out.consumer.default.svc.cluster.local|http",
    "service_name": "consumer.default.svc.cluster.local|http",
    "connect_timeout_ms": 1000,
    "type": "sds",
    "lb_type": "round_robin"
   },{
    "name": "out.details.nino.svc.cluster.local|http|version=v1",
    "service_name": "details.nino.svc.cluster.local|http|version=v1",
    "connect_timeout_ms": 1000,
    "type": "sds",
    "lb_type": "round_robin"
   }, {
    "name": "out.productpage.nino.svc.cluster.local|http|version=v1",
    "service_name": "productpage.nino.svc.cluster.local|http|version=v1",
    "connect_timeout_ms": 1000,
    "type": "sds",
    "lb_type": "round_robin"
   },{
    "name": "out.provider.default.svc.cluster.local|http",
    "service_name": "provider.default.svc.cluster.local|http",
    "connect_timeout_ms": 1000,
    "type": "sds",
    "lb_type": "round_robin"
   },{
    "name": "out.ratings.nino.svc.cluster.local|http|version=v1",
    "service_name": "ratings.nino.svc.cluster.local|http|version=v1",
    "connect_timeout_ms": 1000,
    "type": "sds",
    "lb_type": "round_robin"
   },{
    "name": "out.reviews.nino.svc.cluster.local|http|version=v1",
    "service_name": "reviews.nino.svc.cluster.local|http|version=v1",
    "connect_timeout_ms": 1000,
    "type": "sds",
    "lb_type": "round_robin"
   },{
    "name": "out.reviews.nino.svc.cluster.local|http|version=v2",
    "service_name": "reviews.nino.svc.cluster.local|http|version=v2",
    "connect_timeout_ms": 1000,
    "type": "sds",
    "lb_type": "round_robin"
   }]
 }
```

还发现cluster是可以细分version的。

```json
curl http://10.96.23.66:8080/v1/registration/reviews.nino.svc.cluster.local\|http\|version=v1
{
  "hosts": [
   {
    "ip_address": "10.244.0.38",
    "port": 9080
   }
  ]
 }
```

#### RDS

以bookinfo这个例子来说，productpage要访问不同的reviews服务。并且发布了reviews的路由规则，通过RDS接口可以查到对应服务的权重规则，和权重对应的cluster。

```json

root@blueocean:~# curl http://10.96.23.66:8080/v1/routes/9080/productpage/sidecar~10.244.0.40~productpage-v1-8666ffbd7c-mf5f4.nino~nino.svc.cluster.local
{
  "validate_clusters": true,
  "virtual_hosts": [
   {
    "name": "details.nino.svc.cluster.local|http",
    "domains": [
     "details:9080",
     "details",
     "details.nino:9080",
     "details.nino",
     "details.nino.svc:9080",
     "details.nino.svc",
     "details.nino.svc.cluster:9080",
     "details.nino.svc.cluster",
     "details.nino.svc.cluster.local:9080",
     "details.nino.svc.cluster.local",
     "10.108.87.201:9080",
     "10.108.87.201"
    ],
    "routes": [
     {
      "prefix": "/",
      "cluster": "out.details.nino.svc.cluster.local|http|version=v1",
      "timeout_ms": 0,
      "decorator": {
       "operation": "details-default"
      }
     }
    ]
   },
   {
    "name": "productpage.nino.svc.cluster.local|http",
    "domains": [
     "productpage:9080",
     "productpage",
     "productpage.nino:9080",
     "productpage.nino",
     "productpage.nino.svc:9080",
     "productpage.nino.svc",
     "productpage.nino.svc.cluster:9080",
     "productpage.nino.svc.cluster",
     "productpage.nino.svc.cluster.local:9080",
     "productpage.nino.svc.cluster.local",
     "10.107.237.55:9080",
     "10.107.237.55"
    ],
    "routes": [
     {
      "prefix": "/",
      "cluster": "out.productpage.nino.svc.cluster.local|http|version=v1",
      "timeout_ms": 0,
      "decorator": {
       "operation": "productpage-default"
      }
     }
    ]
   },
   {
    "name": "ratings.nino.svc.cluster.local|http",
    "domains": [
     "ratings:9080",
     "ratings",
     "ratings.nino:9080",
     "ratings.nino",
     "ratings.nino.svc:9080",
     "ratings.nino.svc",
     "ratings.nino.svc.cluster:9080",
     "ratings.nino.svc.cluster",
     "ratings.nino.svc.cluster.local:9080",
     "ratings.nino.svc.cluster.local",
     "10.101.178.48:9080",
     "10.101.178.48"
    ],
    "routes": [
     {
      "prefix": "/",
      "cluster": "out.ratings.nino.svc.cluster.local|http|version=v1",
      "timeout_ms": 0,
      "decorator": {
       "operation": "ratings-default"
      }
     }
    ]
   },
   {
    "name": "reviews.nino.svc.cluster.local|http",
    "domains": [
     "reviews:9080",
     "reviews",
     "reviews.nino:9080",
     "reviews.nino",
     "reviews.nino.svc:9080",
     "reviews.nino.svc",
     "reviews.nino.svc.cluster:9080",
     "reviews.nino.svc.cluster",
     "reviews.nino.svc.cluster.local:9080",
     "reviews.nino.svc.cluster.local",
     "10.108.121.171:9080",
     "10.108.121.171"
    ],
    "routes": [
     {
      "prefix": "/",
      "weighted_clusters": {
       "clusters": [
        {
         "name": "out.reviews.nino.svc.cluster.local|http|version=v1",
         "weight": 50
        },
        {
         "name": "out.reviews.nino.svc.cluster.local|http|version=v2",
         "weight": 50
        }
       ]
      },
      "timeout_ms": 0,
      "decorator": {
       "operation": "reviews-default"
      }
     }
    ]
   }
  ]
 }
```

### 环境信息

159.138.5.60 159.138.2.112