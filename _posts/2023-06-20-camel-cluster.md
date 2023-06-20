---
title: "Apache Camel K8s Cluster 구성"
date: 2023-06-20 22:37:00 +0900
categories: [Integration, Camel]
tags: [camel,cluster,kubernetes]
author: psanghun
---

> Apache Camel을 이용해 Integration Product를 개발 중 Kubernetes 환경에서<br>
> Cron을 사용했을 때 여러 인스턴스 중에 단 한곳에서만 Batch Job 이 구동되어야만 했다.<br>
> 기존에는 서버를 지정해서 하나에서만 Route를 등록했으나, k8s 환경에서는 동적으로 pod의 개수가<br>
> 달라지고 서버가 바뀔 수 있기 때문에 적절한 해법이 아니었다.<br><br>
> Camel 이 제공하는 cluster 에는 File 및 zookeeper 등 몇가지 옵션이 있는데, 우리는 어차피 k8s 로 <br>
> 구성할 예정이기에 k8s cluster 로 간다~!

# Apache Camel K8s Cluster 구성

## 개발 환경
>
> Framework : SpringBoot 2.5 <br>
> Language : Kotlin 1.7 ( Java 11 )<br>
> Camel : Apache Camel 3.13

## 1. Spring 환경 설정
#### build.gradle.kt 에 kubernetes dependency 추가
```groovy
...
extra["camelVersion"] = "3.13.0"
dependencies {
  ...
  // Kubernetes
  implementation("org.apache.camel.springboot:camel-kubernetes-starter:${property("camelVersion")}")
  ...
}
```

## 2. Application 소스 수정
#### Cron Router 를 생성하는 코드에 Cluster 서비스 추가 및 Policy 설정 추가
```kotlin
/**
 * 배치용 Cron Listener Builder
 */
class CronListenerBuilder : BaseListenerBuilder() {

    /**
     * Cron Listener 구성
     */
    override fun buildListener(
        flow: FlowDTO,
        routesDefinition: RoutesDefinition,
        restsDefinition: RestsDefinition,
        camelContext: CamelContext,
        endpointContext: EndpointContext,
        organizationProjectContext: OrganizationProjectContext
    ): RouteDefinition? {

        // K8s Cluster Service 등록
        val clusterService = KubernetesClusterService()
        if (!camelContext.hasService(clusterService))
            camelContext.addService(clusterService)

        val listenerUri = ComponentUtil.getOptionValue(flow.listener!!, "uri")!!

        val uri = "cron:" + listenerUri + "?schedule=" + ComponentUtil.getOptionValue(flow.listener!!, "schedule")!!
        val routeDefinition = routesDefinition.from(uri)
        routeDefinition.setBody().simple(listenerUri)
        // Cron Listener 의 경우 Custer 구성을 통해 1개만 실행되도록 보장
        routeDefinition.routePolicy(ClusteredRoutePolicy.forNamespace(Constants.Kubernetes.NAMESPACE))

        // 추가 속성 설정
        setRouteProperties(routeDefinition, flow, organizationProjectContext)

        return routeDefinition
    }

}
```
## 3. Kubernetes 설정
#### Cluster Role 생성
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: lease-access
rules:
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - get
  - create
  - update
  - list
- apiGroups:
    - ""
  resources:
    - pods
  verbs:
    - get
    - list
```
#### Service Account에 Cluster Role Binding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: lease-access
roleRef:
  kind: ClusterRole
  name: lease-access
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: {{ .Values.serviceAccount.name }}
  namespace: {{ .Values.serviceAccount.namespace }}
```
## 4. 실행 결과
#### pod 가 1개인 경우
Cluster 내부의 pod 가 1개 이므로 해당 pod 가 Master 역할 수행 : 새로운 Leader가 선출 되었음을 로그에서 확인 할 수 있다.
<br>
<div style="background-color: #0b0b0b; padding:1rem;">
<span style="color: #0d97ff"> [ 서비스 Log 중  ]</span><br><br>
<pre style="color: #9a9a9a; background-color: #0b0b0b">
2023-06-20T14:09:59.796Z  INFO 8 --- [rshipController] c.c.k.c.l.KubernetesLeadershipController : Pod[camel-test-6f55dd9cd7-5ld8p] Leadership has been lost by old owner. Trying to acquire the leadership...
2023-06-20T14:09:59.872Z  INFO 8 --- [rshipController] c.c.k.c.l.KubernetesLeadershipController : Pod[camel-test-6f55dd9cd7-5ld8p] Leadership acquired by current pod
2023-06-20T14:09:59.880Z  INFO 8 --- [rshipController] c.c.k.c.l.KubernetesLeadershipController : Pod[camel-test-6f55dd9cd7-5ld8p] Current pod owns the leadership, but it will be effective in 15.00 seconds...
2023-06-20T14:10:14.883Z  INFO 8 --- [rshipController] c.c.k.c.l.KubernetesLeadershipController : Pod[camel-test-6f55dd9cd7-5ld8p] Current pod is becoming the new leader now...
2023-06-20T14:10:14.923Z  INFO 8 --- [sLeaderNotifier] o.a.c.c.k.c.lock.TimedLeaderNotifier     : The cluster has a new leader: Optional[camel-test-6f55dd9cd7-5ld8p]
2023-06-20T14:10:14.935Z  INFO 8 --- [sLeaderNotifier] o.a.c.c.k.c.lock.TimedLeaderNotifier     : The list of cluster members has changed: [camel-test-6f55dd9cd7-5ld8p]
</pre>
</div>
<br>

#### pod를 2개 더 추가한 경우
Cluster 내부의 pod 가 3개가 되었고, 이를 감지하여 로그에 출력됨, 여전히 Master는 1번 pod
<br>
<div style="background-color: #0b0b0b; padding:1rem;">
<span style="color: #0d97ff"> [ 서비스 Log 중  ]</span><br><br>
<pre style="color: #9a9a9a; background-color: #0b0b0b">
2023-06-20T14:14:10.003Z  INFO 8 --- [ron:master-cron] route2                                   : Master Cron Event
2023-06-20T14:14:15.811Z  INFO 8 --- [sLeaderNotifier] o.a.c.c.k.c.lock.TimedLeaderNotifier     : The list of cluster members has changed: [camel-test-6f55dd9cd7-vcpsn, camel-test-6f55dd9cd7-jjbkc, camel-test-6f55dd9cd7-5ld8p]
</pre>
</div>
<br>

#### 첫번째 Master 였던 pod를 delete 한 경우
master pod 가 삭제되어 member가 변동됨을 감지하고, 다른 node 중 하나가 master로 승격된다
<br>
<div style="background-color: #0b0b0b; padding:1rem;">
<span style="color: #0d97ff"> [ 서비스 Log 중  ]</span><br><br>
<pre style="color: #9a9a9a; background-color: #0b0b0b">
2023-06-20T14:16:56.712Z  INFO 7 --- [sLeaderNotifier] o.a.c.c.k.c.lock.TimedLeaderNotifier     : The list of cluster members has changed: [camel-test-6f55dd9cd7-vcpsn, camel-test-6f55dd9cd7-jjbkc, camel-test-6f55dd9cd7-5ld8p, camel-test-6f55dd9cd7-7gm7g]
2023-06-20T14:17:26.600Z  INFO 7 --- [rshipController] c.c.k.c.l.KubernetesLeadershipController : Pod[camel-test-6f55dd9cd7-vcpsn] Leadership has been lost by old owner. Trying to acquire the leadership...
2023-06-20T14:17:27.095Z  INFO 7 --- [rshipController] c.c.k.c.l.KubernetesLeadershipController : Pod[camel-test-6f55dd9cd7-vcpsn] Leadership acquired by current pod
2023-06-20T14:17:27.096Z  INFO 7 --- [rshipController] c.c.k.c.l.KubernetesLeadershipController : Pod[camel-test-6f55dd9cd7-vcpsn] Current pod owns the leadership, but it will be effective in 15.00 seconds...
2023-06-20T14:17:39.124Z  INFO 7 --- [sLeaderNotifier] o.a.c.c.k.c.lock.TimedLeaderNotifier     : The cluster has a new leader: Optional.empty
2023-06-20T14:17:42.098Z  INFO 7 --- [rshipController] c.c.k.c.l.KubernetesLeadershipController : Pod[camel-test-6f55dd9cd7-vcpsn] Current pod is becoming the new leader now...
2023-06-20T14:17:42.146Z  INFO 7 --- [sLeaderNotifier] o.a.c.c.k.c.lock.TimedLeaderNotifier     : The cluster has a new leader: Optional[camel-test-6f55dd9cd7-vcpsn]
2023-06-20T14:17:42.203Z  INFO 7 --- [sLeaderNotifier] o.a.c.c.k.c.lock.TimedLeaderNotifier     : The list of cluster members has changed: [camel-test-6f55dd9cd7-vcpsn, camel-test-6f55dd9cd7-jjbkc, camel-test-6f55dd9cd7-7gm7g]
</pre>
</div>

## 5. 결론
#### 이로써 K8s 환경에서 cron router 는 단 하나의 router 만 구동되도록 보장되고 master 에 문제가 발생시 자동으로 leader가 선출되어 서비스가 지속 될 수 있음을 확인했다.
