## Pilot



pilot类比一个服务中心，它可以扩展不同后端实现服务发现。比如k8s上的服务注册是通过应用自己发布Service关联自己注册的。同时还附带配置中心的功能，比如发现路由规则的变化，通过EnvoyAPI给数据面使用。总结来说pilot提供了一组发现服务DS，包括服务发现SDS、集群发现CDS、路由规则发现RDS和监听器发现LDS。

#### SDS

> SDS可以发现cluster内的所有节点，是Envoy官方推荐的服务发现方式，因为不会因为DNS长查询而导致延时很大，还能比DNS携带更多信息，用于实现智能LB和routing。

通过以下接口可以查到所有服务，比如在istio官方例子上的details和reviews服务。其中reviews服务关联了三个Deployment，每个Dp副本为1，区别是labels里的version不同。

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

查询集群里某个服务信息。

```json
  "cache_stats": {
   "/v1/clusters/details/sidecar~10.244.0.37~details-v1-748b65d849-8dvnf.nino~nino.svc.cluster.local": {
    "hit": 228114,
    "miss": 170
   },
   "/v1/clusters/istio-ingress/ingress~~istio-ingress-577d7b7fc7-sxszz.istio-system~istio-system.svc.cluster.local": {
    "hit": 230448,
    "miss": 223
   },
   "/v1/clusters/productpage/sidecar~10.244.0.40~productpage-v1-8666ffbd7c-mf5f4.nino~nino.svc.cluster.local": {
    "hit": 228206,
    "miss": 166
   },
   "/v1/clusters/ratings/sidecar~10.244.0.39~ratings-v1-6659b586fc-c76d7.nino~nino.svc.cluster.local": {
    "hit": 228126,
    "miss": 178
   },
   "/v1/clusters/reviews/sidecar~10.244.0.38~reviews-v1-5b7d69d695-vpgrv.nino~nino.svc.cluster.local": {
    "hit": 228058,
    "miss": 169
   },
   "/v1/clusters/reviews/sidecar~10.244.0.41~reviews-v3-b6d4dd775-xrjdm.nino~nino.svc.cluster.local": {
    "hit": 228099,
    "miss": 165
   },
   "/v1/clusters/reviews/sidecar~10.244.0.42~reviews-v2-c84995789-pgzwt.nino~nino.svc.cluster.local": {
    "hit": 227925,
    "miss": 165
   },

```







curl http://10.96.23.66:8080/v1/routes/8084/reviews/sidecar~10.244.0.42~reviews-v2-c84995789-pgzwt.nino~nino.svc.cluster.local

```json

 }root@blueocean:~# curl http://10.96.23.66:8080/v1/routes/9080/reviews/^C
root@blueocean:~# curl http://10.96.23.66:8080/v1/routes/9080/reviews/sidecar~10.244.0.38~reviews-v1-5b7d69d695-vpgrv.nino~nino.svc.cluster.local
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
 }r
```



### 环境信息

159.138.5.60 159.138.2.112