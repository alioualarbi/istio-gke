#                  Create cluster with the Istio GKE addon
#### 1. Creating a cluster: 
```
$ export GCP_ZONE=[YOUR-ZONE]
$ gcloud beta container clusters create "istio-cluster" --zone $GCP_ZONE --no-enable-basic-auth --cluster-version "1.13.7-gke.24" --machine-type "n1-standard-2" --num-nodes "3" --enable-stackdriver-kubernetes --enable-ip-alias --addons HorizontalPodAutoscaling,HttpLoadBalancing,Istio --istio-config auth=MTLS_PERMISSIVE --enable-autoupgrade --enable-autorepair
```
#### 2. Getting credentials

```
$ gcloud container clusters get-credentials istio-cluster --zone $GCP_ZONE 
```
#### 3. verify istio services and pods
```
$ kubectl get svc -n istio-system
$ kubectl get pods -n istio-system
```
##### Example services output
```
$ kubectl get svc -n istio-system
NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                                                                                                                      AGE
istio-citadel            ClusterIP      10.xxx.5.41     <none>          8060/TCP,15014/TCP                                                                                                                           9d
istio-galley             ClusterIP      10.xxx.4.124    <none>          443/TCP,15014/TCP,9901/TCP                                                                                                                   9d
istio-ingressgateway     LoadBalancer   10.xxx.0.90     xx.xxx.xxx.xxx  15020:32425/TCP,80:30981/TCP,443:30428/TCP,31400:30104/TCP,15029:31617/TCP,15030:32527/TCP,15031:30954/TCP,15032:30462/TCP,15443:30912/TCP   9d
istio-pilot              ClusterIP      10.xxx.12.78    <none>          15010/TCP,15011/TCP,8080/TCP,15014/TCP                                                                                                       9d
istio-policy             ClusterIP      10.xxx.3.177    <none>          9091/TCP,15004/TCP,15014/TCP                                                                                                                 9d
istio-sidecar-injector   ClusterIP      10.xxx.10.218   <none>          443/TCP                                                                                                                                      9d
istio-telemetry          ClusterIP      10.xxx.13.111   <none>          9091/TCP,15004/TCP,15014/TCP,42422/TCP                                                                                                       9d
promsd                   ClusterIP      10.xxx.3.19     <none>          9090/TCP                                      
```
##### Example pods output
```
$ kubectl get pods -n istio-system
NAME                                             READY   STATUS      RESTARTS   AGE
istio-citadel-554b499885-d6rxp                   1/1     Running     0          9d
istio-cleanup-secrets-1.1.16-gke.0-gk5tk         0/1     Running     0          9d
istio-galley-7954555f7b-lt4pm                    1/1     Running     0          9d
istio-ingressgateway-6d8f9d87f8-fjsbd            1/1     Running     0          9d
istio-pilot-78d6847769-blrw8                     2/2     Running     0          9d
istio-policy-6b799c557-tbgqz                     2/2     Running     2          9d
istio-security-post-install-1.1.16-gke.0-hllfk   0/1     Running     0          9d
istio-sidecar-injector-8cd757776-nrwhd           1/1     Running     0          9d
istio-telemetry-799668466f-6dw5n                 2/2     Running     2          9d
promsd-76f8d4cff8-zjv59                          2/2     Running     1          9d
```

#### 4. install istio tool

We should map our GKE version to Istio GKE Addon version mapping in our case ==> 1.13.7-gke.24 -> Istio 1.1.13-gke.0
```
$ wget https://github.com/istio/istio/releases/download/1.1.13/istio-1.1.13-linux.tar.gz
$ tar -xvf istio-1.1.13-linux.tar.gz
$ cd istio-1.1.13
$ export PATH=$PWD/bin:$PATH
```
##### Verification to Get an overview of your mesh

```
$ istioctl proxy-status  ==< (istioctl ps)
NAME                                                   CDS        LDS        EDS               RDS          PILOT                            VERSION
istio-ingressgateway-6d8f9d87f8-fjsbd.istio-system     SYNCED     SYNCED     SYNCED (100%)     NOT SENT     istio-pilot-78d6847769-blrw8     1.1.3
```
#### 5. Getting details about envoy proxies programming using istioctl proxy-config (istioctl pc)

###### CDS “cluster discovery service”
```
$ istioctl pc clusters $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system
```
###### LDS “listener discovery service”
```
$ istioctl pc listeners $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system
```
###### EDS “endpoints discovery service”
```
$ istioctl pc endpoints $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system
```
###### RDS “router discovery service”
```
$ istioctl pc routes $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system
```
###### Looking at one config in more details
```
$ istioctl pc routes $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system -o json
```

#### 5. Let's deploy a sample app ([Bookinfo Application](https://istio.io/docs/examples/bookinfo/#deploying-the-application))

![alt text](https://istio.io/docs/examples/bookinfo/withistio.svg)

##### 5.1 Enabling automatic sidecar injection on the default namespace
with the GKE addon, the WebHook needed for sidecar injection is pre-configured and enabled by default. However, the default namespace does not have automatic sidecar injection enabled by default. In order to start using Istio with automatic sidecar injection, one needs to enable injection on the desired namespace. Here we will enable it on the default namespace:
```
$ kubectl label namespace default istio-injection=enabled
```
Checking sidecar injection on all namespaces
```
kubectl get namespace -L istio-injection
```
##### 5.2 Manual sidecar injection
Another feature with the istio  tool is to facilitates manual sidecar injection

```
$ istioctl kube-inject -f deployment.yaml -o deployment-injected.yaml
```

A popular usage of this command, is the “on-the-fly” update of a resource as follows:
```
$ kubectl apply -f <(istioctl kube-inject -f <resource.yaml>)
```

##### 5.3 Deploying the sample app
```
$ wget -q -O - https://raw.githubusercontent.com/istio/istio/release-1.3/samples/bookinfo/platform/kube/bookinfo.yaml | kubectl apply -f -
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created

```
##### 5.3 Verifying the deployment
```
$ kubectl get svc
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.120.9.24    <none>        9080/TCP   57s
kubernetes    ClusterIP   10.120.0.1     <none>        443/TCP    9d
productpage   ClusterIP   10.120.5.134   <none>        9080/TCP   55s
ratings       ClusterIP   10.120.3.255   <none>        9080/TCP   56s
reviews       ClusterIP   10.120.2.99    <none>        9080/TCP   56s
```
```
$ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
details-v1-68fbb76fc-m69lz            1/1     Running   0          110s
multi-tool-kubectl-78c7757894-x9wzh   1/1     Running   0          8d
productpage-v1-6c6c87ffff-jp577       1/1     Running   0          109s
ratings-v1-7bdfd65ccc-c6v6x           1/1     Running   0          110s
reviews-v1-5c5b7b9f8d-8b4jv           1/1     Running   0          110s
reviews-v2-569796655b-vt79g           1/1     Running   0          109s
reviews-v3-844bc59d88-vjhvv           1/1     Running   0          109s
```
##### 5.4 Checking internal communication between pods
 Example from the ratings pod, reach productpage
 
 ```
 $ kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
 ```
 
##### Output
 ```
<title>Simple Bookstore App</title>
 ```
 
##### 5.5 Verifying the Mesh with the app deployed
```
$ istioctl ps
```
##### 5.6 Looking at the ingress gateway

Notice that clusters now show the application services we deployed:

``` 
$ istioctl pc clusters $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system 
```

Listeners are still empty:

``` 
$ istioctl pc listeners $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system 
```

Endpoints now show the application services:

``` 
$ istioctl pc endpoints $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system  | grep default
```

Routes did not change:

``` 
$ istioctl pc routes $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system 
```


All application pods were injected with an envoy sidecar and some configuration was automatically loaded to those proxies based on the mesh default configuration and the kubernetes services that were defined.

Comparing the xDS config for the application envoy with the ingress gateway

Clusters are similar to the ingress gateway:

```
$ istioctl pc clusters $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')
```
The app has listeners for its own service, envoy and a number of other system services:

```
$ istioctl pc listeners $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')
```
Endpoints are similar as the ingress gateway:

```
$ istioctl pc endpoints $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')
```
The app has inbound routes for its own service and a number of other routes for services in the mesh:

```
$ istioctl pc routes $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')
```
We can expand a route config to see how it is configure with the following command:
  
  ```
$ istioctl pc routes $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}') --name 9080 -o json
```

#### 6. Define ingress rules to enable external access to the app

[Documentation](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/)

Let's configure the access to the sample application from outside of the cluster.

With the Istio GKE addon, the ingress gateway is exposed via Network LoadBalancer by default. If we look at the istio-ingressgateway service, we will see it has an external IP.

```
$ kubectl get svc -n istio-system -l istio=ingressgateway
```
##### Output
```
NAME                   TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                                                                                                                                      AGE
istio-ingressgateway   LoadBalancer   10.XXX.0.90   xxx.xxx.xxx.xxx   15020:32425/TCP,80:30981/TCP,443:30428/TCP,31400:30104/TCP,15029:31617/TCP,15030:32527/TCP,15031:30954/TCP,15032:30462/TCP,15443
```
Reaching the istio-ingressgateway external IP:
```
$ curl $(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```
##### Output
```
curl: (7) Failed to connect to xxx.xxxx.xxxx.xxxx port 80: Connection refused
```
The command returns ‘connection refused’ since there is no Istio programming for the ingress gateway to route requests to the sample app deployed. We will create rules for that in the next steps.


##### 6.1 Creating a Gateway and VirtualService
###### 6.1.1 Creating the Gateway object [Documentation](https://istio.io/docs/reference/config/networking/v1alpha3/gateway/)

The following gateway object configures the ingressgateway deployment to accept incoming connections to port 80 on any IP (hosts: *).
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
EOF
```

Now, The ingress gateway now has a listener for port 80
```
$ istioctl pc listeners $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system
```
And a route for http 80 was also added
```
$ istioctl pc routes $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system
```
The route defines a default 404 response
```
$ istioctl pc routes $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system --name http.80 -o json
```

At this point, we will get 404 when reaching the ingress gateway instead of ‘connection refused’:
```
$ curl -I $(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

So basically we just programmed the envoy from the istio-ingressgateway to listen on port 80 and to return 404 by default.

###### 6.1.2 Creating the VirtualService object [Documentation](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/)

What allows us to program the url mappings on the ingress gateway with our application backends, is the VirtualService object. The following VirtualService programs a set of uris which maps to the productpage application backend.
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
EOF
```
Looking at the http.80 route now, we see it is linked to the productpage backend:
```
$ istioctl pc routes $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system --name http.80 -o json
```
Checking the objects created
```
$ kubectl get gateway
$ kubectl get gateway bookinfo-gateway -o yaml
$ kubectl get virtualservice 
$ kubectl get virtualservice bookinfo -o yaml
```
Reaching the app from outside the cluster
Forming the app URL with the gateway endpoint
```
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')

export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT/productpage
```
Reaching the app from outside the cluster
```
$ curl -s http://${GATEWAY_URL} | grep -o "<title>.*</title>"
```
NOTE: the response header shows the request is served by envoy
```
$ curl -I http://${GATEWAY_URL}
```
#### 7. Accessing an external service from inside the Mesh [Documentation](https://istio.io/docs/tasks/traffic-management/egress/egress-control/)

In this version of Istio, envoy is configured by default as a passthrough for requests to external services.

To confirm that, let’s access an external http service from a pod:
```
$ kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl -I httpbin.org
```
Or using my [http incoming request application](https://github.com/alioualarbi/Http-webhook) as http incoming tester.

```
$ kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl -d '{"app":"rating", "need":"ack"}' -X POST http://34.98.83.12/eeda9dc1-8ebd-4f6e-b7ce-9f349b4735cb
```

![alt text](https://github.com/alioualarbi/istio-gke/blob/master/webhook.png)

NOTE: A ServiceEntry would be required if the global.outboundTrafficPolicy.mode was set to {REGISTRY_ONLY} instead of {ALLOW_ANY}. On Istio 1.0.x {REGISTRY_ONLY} was default but since 1.1.x it is now {ALLOW_ANY}. When using Isito OSS, users can overwrite the release default and configure the global policy they prefer.

Based on this global policy, all sidecar proxies are configured with ‘PassthroughCluster’ which forwards the request to its ‘ORIGINAL_DST’. For an example, see:

```
$ istioctl pc clusters $(kubectl get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system | grep PassthroughCluster
```
This means that the PassthroughCluster does not show on the output of ‘istioctl pc clusters’, likely {REGISTRY_ONLY} is configured for global.outboundTrafficPolicy.mode.

It is possible to verify that on a cluster, by checking the istio configmap on istio-system namespace:
```
$ kubectl -n istio-system get configmap istio -o=jsonpath='{.data.mesh}'
```

###### Output
```
# Set the default behavior of the sidecar for handling outbound traffic from the application:
# ALLOW_ANY - outbound traffic to unknown destinations will be allowed, in case there are no
#   services or ServiceEntries for the destination port
# REGISTRY_ONLY - restrict outbound traffic to services defined in the service registry as well
#   as those defined through ServiceEntries
outboundTrafficPolicy:
  mode: ALLOW_ANY

localityLbSetting:
  {}
 ```
##### 7.1 Forcing traffic through the Egress Gateway [Documentation](https://istio.io/docs/tasks/traffic-management/egress/egress-gateway/)

###### 7.1.1 install [Helm](https://helm.sh/)

```
$ wget https://get.helm.sh/helm-v2.14.3-linux-amd64.tar.gz
$ tar -xvf helm-v2.14.3-linux-amd64.tar.gz
$ sudo mv linux-amd64/helm /usr/local/bin/helm
```

###### 7.1.1 installthe egress gateway helm template

Be sure to be on the istio release directory installed above:
```
$ pwd
/home/istio-1.1.13
```
Use helm to deploy the ingress gateway based on its release template:
```
$ helm template install/kubernetes/helm/istio --name istio-egressgateway --namespace istio-system \
    -x charts/gateways/templates/deployment.yaml -x charts/gateways/templates/service.yaml \
    -x charts/gateways/templates/serviceaccount.yaml -x charts/gateways/templates/autoscale.yaml \
    -x charts/gateways/templates/role.yaml -x charts/gateways/templates/rolebindings.yaml \
    --set global.istioNamespace=istio-system --set gateways.istio-ingressgateway.enabled=false \
    --set gateways.istio-egressgateway.enabled=true | kubectl apply -f -
```


###### Output
```
$ helm template install/kubernetes/helm/istio --name istio-egressgateway --namespace istio-system \
>     -x charts/gateways/templates/deployment.yaml -x charts/gateways/templates/service.yaml \
>     -x charts/gateways/templates/serviceaccount.yaml -x charts/gateways/templates/autoscale.yaml \
>     -x charts/gateways/templates/role.yaml -x charts/gateways/templates/rolebindings.yaml \
>     --set global.istioNamespace=istio-system --set gateways.istio-ingressgateway.enabled=false \
>     --set gateways.istio-egressgateway.enabled=true | kubectl apply -f -
serviceaccount/istio-egressgateway-service-account created
service/istio-egressgateway created
deployment.extensions/istio-egressgateway created
horizontalpodautoscaler.autoscaling/istio-egressgateway created
```

###### 7.1.2 Checking the egress gateway
```
$ kubectl get pods -n istio-system
$ kubectl describe pod -l app=istio-egressgateway -n istio-system
```
We can check it as well on the mesh overview.

```
$ istioctl ps
NAME                                                   CDS        LDS        EDS               RDS          PILOT                            VERSION
istio-egressgateway-7f76d9794-4gf4b.istio-system       SYNCED     SYNCED     SYNCED (100%)     NOT SENT     istio-pilot-78d6847769-blrw8     1.1.3
istio-ingressgateway-6d8f9d87f8-fjsbd.istio-system     SYNCED     SYNCED     SYNCED (100%)     NOT SENT     istio-pilot-78d6847769-blrw8     1.1.3
```
###### 7.1.2 Making a external request directly to httpbin.org
```
$ kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl -I http://httpbin.org

HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Content-Length: 9593
Content-Type: text/html; charset=utf-8
Date: Thu, 14 Nov 2019 21:32:49 GMT
Referrer-Policy: no-referrer-when-downgrade
Server: nginx
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Connection: keep-alive
```
###### 7.1.3 Stats of the egress gateway [Documentation](https://www.envoyproxy.io/docs/envoy/v1.5.0/configuration/cluster_manager/cluster_stats)
```
$ kubectl exec -it $(kubectl get pod -l app=istio-egressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system) -n istio-system -c istio-proxy -- curl -s localhost:15000/clusters | grep httpbin | grep total
```
Note: this should be empty

###### 7.1.4 Create a ServiceEntry for httpbin.org
```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
EOF

```
Checking the stats again, we see outbound to httpbin.org has been programmed but the stats are still zero connections:
```
$ kubectl exec -it $(kubectl get pod -l app=istio-egressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system) -n istio-system -c istio-proxy -- curl -s localhost:15000/clusters | grep httpbin | grep total
```

Checking the ingegress gateway programming:

```
$ istioctl pc clusters $(kubectl get pods -l istio=egressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system | grep httpbin
outbound|80||httpbin.org::52.200.159.44:80::cx_total::0
outbound|80||httpbin.org::52.200.159.44:80::rq_total::0
outbound|80||httpbin.org::3.225.168.125:80::cx_total::0
outbound|80||httpbin.org::3.225.168.125:80::rq_total::0
outbound_.80_._.httpbin.org::52.200.159.44:80::cx_total::0
outbound_.80_._.httpbin.org::52.200.159.44:80::rq_total::0
outbound_.80_._.httpbin.org::3.225.168.125:80::cx_total::0
outbound_.80_._.httpbin.org::3.225.168.125:80::rq_total::0
```

```
$ istioctl pc endpoints $(kubectl get pods -l istio=egressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system | grep httpbin
3.225.168.125:80       HEALTHY     outbound_.80_._.httpbin.org
3.225.168.125:80       HEALTHY     outbound|80||httpbin.org
52.200.159.44:80       HEALTHY     outbound_.80_._.httpbin.org
52.200.159.44:80       HEALTHY     outbound|80||httpbin.org
```
###### 7.1.5 Create Gateway and DestinationRule

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - httpbin.org
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egressgateway-for-httpbin
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local
  subsets:
  - name: httpbin
EOF
```
This creates a listener and route for port 80 on the egress gateway proxy

```
$ istioctl pc listeners $(kubectl get pods -l istio=egressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system
ADDRESS     PORT      TYPE
0.0.0.0     80        HTTP
0.0.0.0     15090     HTTP
```

```
$ istioctl pc routes $(kubectl get pods -l istio=egressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system
NAME        VIRTUAL HOSTS
http.80     1
            1
```
However the default route created for the egress gateway is just the default 404 for port 8
```
$ istioctl pc routes $(kubectl get pods -l istio=egressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system --name http.80 -o json
[
    {
        "name": "http.80",
        "virtualHosts": [
            {
                "name": "blackhole:80",
                "domains": [
                    "*"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "directResponse": {
                            "status": 404
                        },
                        "perFilterConfig": {
                            "mixer": {}
                        }
                    }
                ]
            }
        ],
        "validateClusters": false
    }
]
```
Reaching again the endpoint and checking the stats on the egress gateway will still show 0 connections:
```
$ kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl -I http://httpbin.org
```
```
$ kubectl exec -it $(kubectl get pod -l app=istio-egressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system) -n istio-system -c istio-proxy -- curl -s localhost:15000/clusters | grep httpbin | grep total
```
###### 7.1.6 Create a VirtualService
This configures the mesh to send traffic to the egress gateway when the destination is httpbin.org and the egress gateway to send it to httpbin.org.

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-http-through-egress-gateway
spec:
  hosts:
  - httpbin.org
  gateways:
  - istio-egressgateway
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        subset: httpbin
        port:
          number: 80
      weight: 100
  - match:
    - gateways:
      - istio-egressgateway
      port: 80
    route:
    - destination:
        host: httpbin.org
        port:
          number: 80
      weight: 100
EOF
```
Both the app and the egress gateway now see a route that forwards requests to httpbin.org via the istio-egressgateway
```
$ istioctl pc routes $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') --name 80 -o json | grep httpbin
```
```
$istioctl pc routes $(kubectl get pods -l istio=egressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system).istio-system --name http.80 -o json
```
Making requests to httpbin.org through the egress gateway
```
$ kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl -I http://httpbin.org
```
Checking connections are going through the egress gateway
```
$ kubectl exec -it $(kubectl get pod -l app=istio-egressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system) -n istio-system -c istio-proxy -- curl -s localhost:15000/clusters | grep httpbin | grep total
```
##### output:
```
$ kubectl exec -it $(kubectl get pod -l app=istio-egressgateway -o jsonpath='{.items[0].metadata.name}' -n istio-system) -n istio-system -c istio-proxy -- curl -s localhost:15000/clusters | grep httpbin | grep total
outbound|80||httpbin.org::52.200.159.44:80::cx_total::0
outbound|80||httpbin.org::52.200.159.44:80::rq_total::0
outbound|80||httpbin.org::3.225.168.125:80::cx_total::1
outbound|80||httpbin.org::3.225.168.125:80::rq_total::1
outbound_.80_._.httpbin.org::3.225.168.125:80::cx_total::0
outbound_.80_._.httpbin.org::3.225.168.125:80::rq_total::0
outbound_.80_._.httpbin.org::52.200.159.44:80::cx_total::0
outbound_.80_._.httpbin.org::52.200.159.44:80::rq_total::0

```
