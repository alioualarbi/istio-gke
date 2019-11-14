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
###### Creating the Gateway object
[Documentation](https://istio.io/docs/reference/config/networking/v1alpha3/gateway/)

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

###### Creating the VirtualService object
[Documentation](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/)

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
