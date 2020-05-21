This repo contains demo for my talk about service meshes. It is based on [microservices-demo application](https://github.com/microservices-demo/microservices-demo) with some minor modifications to make it play nicely with istio. 

This demo is deployed and tested with `kubernetes 1.16` and `istio 1.5`

### 0. Install istio
1. Refer to [istio docs](https://istio.io/docs/setup/install/) for different methods on how install istio. Istio will be installed in a deferent namespace called `istio-system`
2. Create a namespace for our application and add a namespace label to instruct Istio to automatically inject Envoy sidecar proxies during deployment of sock-shop app. 
```
$ kubectl apply -f 1-deploy-app/manifests/sock-shop-ns.yaml 
$ kubectl label namespace sock-shop istio-injection=enabled
```

### 1. Deploy application
No changes we're made to the original k8s manifests from [microservices-demo application](https://github.com/microservices-demo/microservices-demo) except:

+ updating `Deployment` resources to use the sable api `apps/v1` required since [k8s 1.16](https://kubernetes.io/blog/2019/09/18/kubernetes-1-16-release-announcement/)
+ added `version: v1` label to all Kubernetes deployments. We need it for Istio `Destination Rules` to work properly

1. deploy the application
```
$ kubectl apply -f 1-deploy-app/manifests
```

2. Configure Istio virtual services & Distination rules
```
$ kubectl apply -f 1-deploy-app/sockshop-virtual-services.yaml
```
Along with virtual services, destination rules are a key part of Istio’s traffic routing functionality. You can think of virtual services as how you route your traffic to a given destination, and then you use destination rules to configure what happens to traffic for that destination.

3. Configure Istio ingress gateway
```
$ kubectl apply -f 1-deploy-app/sockshop-gateway.yaml
```
An ingress Gateway describes a load balancer operating at the edge of the mesh that receives incoming HTTP/TCP connections. It configures exposed ports, protocols, etc. but, unlike Kubernetes Ingress Resources, does not include any traffic routing configuration.

4. Verifying our config
```
$ istioctl proxy-status
```
Using the `istioctl proxy-status` command allows us to get an overview of our mesh. If you suspect one of your sidecars isn’t receiving configuration or is not synchronized, proxy-status will let you know. 

If everything is fine, run the below command to open sock-shop app in your browser
```
$ open "http://$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'):$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')"
```
If everything is fine you should see the app up and running along with some socks :) 
5. User accounts
|Username |	Password|
|---------|:--------|
|user	  | password|
|user1	  | password|

### 2. Traffic Management
Istio’s traffic routing rules let you easily control the flow of traffic and API calls between services. Istio simplifies configuration of service-level properties like circuit breakers, timeouts, and retries, and makes it easy to set up important tasks like A/B testing, canary rollouts, and staged rollouts with percentage-based traffic splits.

We start by rolling a new Deployment
```
$ kubectl apply -f 2-traffic-management/canary/front-end-dep-v2.yaml  
```
now we have 2 versions of the front-end app running side by side. However if you hit the browser you'll have only the v1 (blue)

#### 1. Blue/Green Deployment
Blue-green deployment is a technique that reduces downtime and risk by running two identical production environments called Blue and Green.
Now let's swicth to v1 (red) as live environment serving all production traffic. 
```
$ kubectl apply -f 2-traffic-management/blue-green/frontv2-virtual-service.yaml
```
Now if you check the bapp, you'll see only the v2 (red version) of our application. Same way, you can rollback at any moment to the old version

#### 2. Canary deployment
Istio’s routing rules provides important advantages; you can easily control fine-grained traffic percentages (e.g., route 1% of traffic without requiring 100 pods) and you can control traffic using other criteria (e.g., route traffic for specific users to the canary version). To illustrate, let’s look at deploying the `front-end` service and see how simple to achieve canary deployment using istio.

For that, we need to set a routing rule to control the traffic distribution by sending 20% of the traffic to the canary (v2). execute the following command
```
$ kubectl apply -f 2-traffic-management/canary/canary-virtual-service.yaml 
```
and refresh the page a couple of times. The majority of pages return v1 (blue), with some v2 (red) from time to time.
#### 3. Route based on some criteria
With istio, we can easily route requests when they met some desired criteria. 
For now we have v1 and v2 deployed in our clusters, we can forward all forward all users using `Firefox` to v2, and serve v1 to all other clients:
```
$ kubectl apply -f 2-traffic-management/route-headers/frontv2-virtual-service-firefox.yaml
```
#### 4. Mirroring
Traffic mirroring, also called shadowing, is a powerful concept that allows feature teams to bring changes to production with as little risk as possible. Mirroring sends a copy of live traffic to a mirrored service. You can then send the traffic to out-of-band of the critical request path for the primary service (Content inspection, Threat monitoring, Troubleshooting)

```
$ kubectl apply -f 2-traffic-management/mirorring/mirror-v2.yaml 
```
This new rule sends 100% of the traffic to v1 while mirroring the same traffic to v2. you can check the logs of v1 and v2 pods to verify that logs created in v2 are the mirrored requests that are actually going to v1.

### 3. Fault injection
Fault injection is an incredibly powerful way to test and build reliable distributed applications.
Istio allows you to configure faults for HTTP traffic, injecting arbitrary delays or returning specific response codes (e.g., 500) for some percentage of traffic.
#### 1. Delay fault
In this example. we gonna inject a five-second delay for all traffic calling the `catalogue` service. This is a great way to reliably test how our frontend app behaves on a bad network.
```
$ kubectl apply -f 3-fault-injection/delay-faults/delay-fault-injection-virtual-service.yaml
```
Open the application and you can see that it takes now longer to render catalogs
#### 2. Abort fault
Replying to clients with specific response codes, like a 429 or a 500, is also great for testing. For example, it can be challenging to programmatically test how your application behaves when a third-party service that it depends on begins to fail. Using Istio, you can write a set of reliable end-to-end tests of your application’s behavior in the presence of failures of its dependencies.

For example, we can simulate 10% of requests to `catalogue` service is failing at runtime with a 500 response code.
```
$ kubectl apply -f 3-fault-injection/delay-faults/abort-fault-injection-virtual-service.yaml 
$ open "http://$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'):$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')/catalogue"
```
Refresh the page a couple of times. You'll notice that sometimes it doesn't return the json list of catalogs

--- Ressources:
+ [Istio documentation](https://istio.io/docs/t)
+ [Microservices demo](https://microservices-demo.github.io/)
+ [istio up and running](https://www.oreilly.com/library/view/istio-up-and/9781492043775/)