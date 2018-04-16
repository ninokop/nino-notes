## Pilot

159.138.5.60

159.138.2.112

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

随便找了个reviews应用的proxy看看envoy的配置。`/etc/istio/proxy`中可以看到

curl http://10.96.23.66:8080/v1/routes/8084/reviews/sidecar~10.244.0.42~reviews-v2-c84995789-pgzwt.nino~nino.svc.cluster.local