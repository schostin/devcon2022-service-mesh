# Installation and Code for istio

This page tries to cover the installation options I used when installing Istio for the demo

## Creation of a cluster

I simply and qucikly created a GKE cluster via clickops. kubectl is configured against this cluster.

## Installation of Istio

I don't want to install with istioctl since updates, etc. are pretty hard with that variant. Therefore
I used the [Helm installation](https://istio.io/latest/docs/setup/install/helm/) of istio.

### Pre-requisites

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
kubectl create namespace istio-system
```

### Base

```bash
helm install istio-base istio/base --values istio-base.yaml --version 1.15.2 -n istio-system
```

### Istiod (Control Plane)

Here it already got tricky. Try to gather all the [values](./istiod.yaml) I added as default config yourself... Finding
this in the documentation is nearly impossible... [https://istio.io/latest/docs/setup/additional-setup/customize-installation/](https://istio.io/latest/docs/setup/additional-setup/customize-installation/)

```bash
helm install istiod istio/istiod --values istiod.yaml -n istio-system --version 1.15.2 --wait
```

### Gateway

```bash
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
helm install istio-ingress istio/gateway --version 1.15.2 --values gateway.yaml -n istio-ingress --wait
```

```bash
> helm install istio-ingress istio/gateway --version 1.15.2 --values gateway.yaml -n istio-ingress --wait

Error: INSTALLATION FAILED: timed out waiting for the condition
```

So now what? No errors. Just a timeout... Debugging starts already. After a long debugging...

```bash
kc describe rs main-ingress-7c4447fd8c
  Warning  FailedCreate  10s (x19 over 22m)  replicaset-controller  Error creating: Internal error occurred: failed calling webhook "namespace.sidecar-injector.istio.io": failed to call webhook: Post "https://istiod.istio-system.svc:443/inject?timeout=10s": service "istiod" not found
```

Seems like the webhooks are not working as expected... The service istiod does not exist since we have set a revision. This is because my initial
install of istiod had no revision label set. Solution: I wipe everything again and all mutating and validation webhooks and I am not installing
a revisioned install for now... (Canary updates with that not possible later).

After that the installation worked, I am not using a specific revision label now.

### Custom Config

After this installation we receive a pretty blank istio installation with some defaults not configured yet (e.g. mTLS everywhere).
For this I created a quick helm chart to deploy the stuff I need. See [Custom Config](./custom-config).

I did not want to "waste" any time with setting up external facing certificates for the Gateway, therefore the Gateway is reachable via
http... Configuring the gateway to use HTTPs is nevertheless pretty simple with e.g. externalDNS / cert-manager or handing in the
certificate on our own.

I therefore copied the LoadBalancer IP of the service (with `kc get svc`) and manually put it in my DNS zone. For me I added the record
`istio-demo.sebastianneb.de`.

```bash
helm install istio-custom-config ./custom-config -n istio-system
```

With this I actually have a running gateway with istio base installed.

```bash
> curl -v istio-demo.sebastianneb.de
*   Trying 34.159.147.75:80...
* Connected to istio-demo.sebastianneb.de (34.159.147.75) port 80 (#0)
> GET / HTTP/1.1
> Host: istio-demo.sebastianneb.de
> User-Agent: curl/7.79.1
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 404 Not Found
< date: Thu, 13 Oct 2022 16:17:47 GMT
< server: istio-envoy
< content-length: 0
< 
* Connection #0 to host istio-demo.sebastianneb.de left intact
```

## Demo setup

### Bookinfo

Next up I wanted to actually deploy a demo service to show the capabilities of Istio.
I followed the instructions [here](https://istio.io/latest/docs/examples/bookinfo/)

```bash
kubectl label namespace default istio-injection=enabled
kubectl apply -f bookinfo/bookinfo.yaml
kubectl apply -f bookinfo/virtualservice-productpage.yaml
```

Bookinfo available at <http://istio-demo-bookinfo.sebastianneb.de/productpage>

### Echo server

Create a sample Echo server deployment to test e.g. fault injection

```bash
kubectl create namespace echoserver
kubectl label namespace echoserver istio-injection=enabled
kubectl apply -f echoserver/echoserver.yaml
kc apply -f echoserver/virtualservice-echoserver.yaml
```

### Debug Pod

Create a debug pod to test outgoing connectivity to external services

```bash
kubectl create namespace outgoing-traffic
kubectl label namespace outgoing-traffic istio-injection=enabled
kubectl run -i --tty --rm debug --image=curlimages/curl --restart=Never -- sh
```
