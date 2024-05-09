---
title: "Zero Downtime Istio Upgrade"
last_modified_at: 2025-05-01T16:20:02-05:00
classes: wide
categories:
  - Blog
tags:
  - istio
  - networking
  - day-2-operations
  - security
  - traffic
---

# Zero downtime Istio Upgrade

As Istio evolves and matures there is the need to also evolve the mechanichs on how its maintained, upgraded and installed. For this reason nowdays the recommended way to install Istio is using Helm due to the easiness to manage revisions and do canary tests. There are two main failure modes for Istio that would affect your workloads, one the loss of configuration propagation for the sidecars and the second and more critical is the loss of traffic flow through the ingress gateway.

If your istio-agent sidecars lose the ability to communicate with Istiod or are incompatible with the configuration being sent, your workloads will not be able to join or communicate with the mesh. This can even affect existing workloads as endpoint discovery will not be up to date, and you may try to reach workloads that no longer exist. New workloads, however, will not be able to join and will remain down until the issue is resolved. Due to this type of outage, it is recommended for istio-agents to match and retain the same version as the control plane (Istiod). It also makes sense that during an upgrade, the existing control plane deployment remain in place rather than upgrading it directly. It is desirable to do a blue/green deployment as a step toward upgrading Istio without downtime.

Unlike the loss of control plane, an outage in the ingress gateway will have an immediate impact on your end users. Since this is a critical path for the flow of traffic, extra care should be taken for upgrading Istio without downtime. That includes being able to fall back to the existing gateway if the upgrade fails. This is why it’s also recommended to do blue/green ingress gateway deployments.

Before we get into the specifics of how to perform an upgrade, it is important to distinguish between an `in-place` upgrade and a `canary` upgrade. In an `in-place` upgrade you are effectively replacing Istio components with new versions of them and traffic disruption may occur during the upgrade process. There is also the risk of running into incompatible configuration and lacking the ability to rollback to the previous state deems this type of upgrade as not recommended and the one with most risk. A `canary` upgrade can be done by first running a canary deployment of the new control plane, allowing you to monitor the effect of the upgrade with a small percentage of the workloads before migrating all of the traffic to the new version. When installing Istio, the revision installation setting can be used to deploy multiple independent control planes at the same time. A `canary` version of an upgrade can be started by installing the new Istio version’s control plane next to the old one, using a different revision setting. Each revision is a full Istio control plane implementation with its own Deployment, Service, etc, being the reason why a `canary` upgrade is safer than doing an in-place upgrade and is the recommended upgrade method.

Shown below is an example upgrade using Helm with Zero downtime.

## Download and install

1.  Install [kind](https://kind.sigs.k8s.io/)

1.  Download the [latest version of Istio](/docs/setup/getting-started/#download) (v1.21.0 or later) with Alpha support for ambient mode.

1.  Deploy a new local `kind` cluster:

    ```bash
    kind create cluster --config=- <<EOF
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    name: zeroDownTime
    nodes:
    - role: control-plane
    - role: worker
    - role: worker
    EOF
    ```

> **_Tip:_**
To get IP address assignment for `Loadbalancer` service types in `kind`, you may need to install a tool like [MetalLB](https://metallb.universe.tf/). Please consult [this guide](https://kind.sigs.k8s.io/docs/user/loadbalancer/) for more information.

To install MetalLB apply:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```

After getting the MetalLB CRDs in the cluster we need to find the CIDR Docker is using to define the MetalLB CRDs `IPAddressPool` and `L2Advertisement`:

```bash
docker network inspect -v -f '{{.IPAM.Config}}' kind
```

The output should be something like:

```shell
[{172.18.0.0/16  172.18.0.1 map[]} {fc00:f853:ccd:e793::/64  fc00:f853:ccd:e793::1 map[]}]
```

Now create the `IPAddressPool` and `L2Advertisement` needed:

```bash
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: kind
  namespace: metallb-system
spec:
  addresses:
  - 172.18.101.1-172.18.101.254
EOF
```

```bash
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: kind
  namespace: metallb-system
spec:
  ipAddressPools:
  - kind
EOF
```

## Add Helm repos

```
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

- Before upgrading Istio, it is recommended to run the istioctl x precheck command to make sure the upgrade is compatible with your environment:

```
istioctl x precheck
```

> **_Disclaimer:_** Helm does not upgrade or delete CRDs when performing an upgrade. Because of this restriction, an additional step is required when upgrading Istio with Helm.

## Install default Istio revision

```
kubectl create namespace istio-system
```

- Install the Istio base chart version 1.20.4 which contains cluster-wide resources used by the Istio control plane:

```
helm install istio-base istio/base -n istio-system --set defaultRevision=default --version=1.20.4
```

- Validate installation:

```
helm ls -n istio-system
```

Expect something along the lines:

```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
istio-base      istio-system    1               2024-04-30 15:18:39.107697 -0500 -05    deployed        base-1.20.4     1.20.4
```

- Install the Istio discovery chart which deploys the istiod service:

```
helm install istiod istio/istiod -n istio-system --wait --version=1.20.4
```

- Validate again:

```
helm ls -n istio-system
```

Expect something along the lines:

```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
istio-base      istio-system    1               2024-04-30 15:18:39.107697 -0500 -05    deployed        base-1.20.4     1.20.4
istiod          istio-system    1               2024-04-30 15:21:12.440376 -0500 -05    deployed        istiod-1.20.4   1.20.4
```

- Check istiod is running:

```
k get pods -n istio-system
```

Expect:

```
NAME                      READY   STATUS    RESTARTS   AGE
istiod-6fc46dcbd4-wb4mt   1/1     Running   0          76s
```

- Install an ingress gateway:

```
kubectl create namespace istio-ingress
```

```
helm install istio-ingress istio/gateway -n istio-ingress --set revision=default --wait --version=1.20.4
```

Expect:

```
NAME: istio-ingress
LAST DEPLOYED: Tue Apr 30 15:25:33 2024
NAMESPACE: istio-ingress
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
"istio-ingress" successfully installed!

To learn more about the release, try:
  $ helm status istio-ingress
  $ helm get all istio-ingress

Next steps:
  * Deploy an HTTP Gateway: https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/
  * Deploy an HTTPS Gateway: https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/
```

Validate:

```
helm ls -A
```

Expect:

```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
istio-base      istio-system    1               2024-04-30 15:18:39.107697 -0500 -05    deployed        base-1.20.4     1.20.4
istio-ingress   istio-ingress   1               2024-04-30 15:25:33.941145 -0500 -05    deployed        gateway-1.20.4  1.20.4
istiod          istio-system    1               2024-04-30 15:21:12.440376 -0500 -05    deployed        istiod-1.20.4   1.20.4
```

- Validate version:

```
istioctl version
```

Expect:

```
client version: 1.20.4
control plane version: 1.20.4
data plane version: 1.20.4 (1 proxies)
```

- Deploy bookinfo:

```
kubectl label namespace default istio-injection=enabled
```

```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/platform/kube/bookinfo.yaml
```

```
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

- The application will start. As each pod becomes ready, the Istio sidecar will be deployed along with it.

```
kubectl get pods
```

```bash
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-558b8b4b76-2llld       2/2     Running   0          2m41s
productpage-v1-6987489c74-lpkgl   2/2     Running   0          2m40s
ratings-v1-7dc98c7588-vzftc       2/2     Running   0          2m41s
reviews-v1-7f99cc4496-gdxfn       2/2     Running   0          2m41s
reviews-v2-7d79d5bd5d-8zzqd       2/2     Running   0          2m41s
reviews-v3-7dbcdcbc56-m8dph       2/2     Running   0          2m41s
```

- Verify everything is working correctly up to this point. Run this command to see if the app is running inside the cluster and serving HTML pages by checking for the page title in the response:

```bash
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

```xml
<title>Simple Bookstore App</title>
```

- Create a Gateway and VirtualService configurations:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: default
spec:
  selector:
    istio: ingress
  servers:
  - hosts:
    - bookinfo.example.com
    port:
      name: http
      number: 80
      protocol: HTTP
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo
  namespace: default
spec:
  gateways:
  - bookinfo-gateway
  hosts:
  - bookinfo.example.com
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

- Get the ingress gateway external IP:

```
kubectl get svc -n istio-ingress
```

Expect:

```
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                      AGE
istio-ingress   LoadBalancer   10.96.146.247   172.18.101.1   15021:30181/TCP,80:30884/TCP,443:31993/TCP   37
```

Export the IP:

```
export GATEWAY_IP=172.18.101.1
```

- Test accessing bookinfo:

```
curl -v -H Host:bookinfo.example.com "http://$GATEWAY_IP/productpage"
```

Expect a response along the lines:

```
*   Trying [::1]:8181...
* Connected to localhost (::1) port 8181
> GET /productpage HTTP/1.1
> Host:bookinfo.example.com
> User-Agent: curl/8.4.0
> Accept: */*
>
< HTTP/1.1 200 OK
< server: istio-envoy
< date: Tue, 30 Apr 2024 21:02:01 GMT
< content-type: text/html; charset=utf-8
< content-length: 4294
< x-envoy-upstream-service-time: 5033
<
<!DOCTYPE html>
<html>
  <head>
    <title>Simple Bookstore App</title>
    ...
```

- Create a loop to constantly hit the ingress gateway and generate traffic on a separate shell:

```
while true; do curl -I -H Host:bookinfo.example.com "http://$GATEWAY_IP/productpage"; sleep 1; done;
```

## Upgrade to version 1.21.2 with Zero Downtime

While traffic hits the ingress gateway, install the Istio's istiod service version 1.21.2:

```
helm install istiod-canary istio/istiod -n istio-system --set revision=canary --wait --version=1.21.2
```

- Validate again:

```
helm ls -n istio-system
```

Expect something along the lines:

```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
istio-base      istio-system    1               2024-04-30 15:18:39.107697 -0500 -05    deployed        base-1.20.4     1.20.4
istiod          istio-system    1               2024-04-30 15:21:12.440376 -0500 -05    deployed        istiod-1.20.4   1.20.4
istiod-canary   istio-system    1               2024-04-30 16:10:21.099893 -0500 -05    deployed        istiod-1.21.2   1.21.2
```

- Verify that you have two versions of istiod installed in your cluster:

```
kubectl get pods -l app=istiod -L istio.io/rev -n istio-system
```

Expect:

```
NAME                             READY   STATUS    RESTARTS   AGE    REV
istiod-6fc46dcbd4-wb4mt          1/1     Running   0          51m    default
istiod-canary-7655d86d7c-k6zmb   1/1     Running   0          113s   canary
```

- Install a canary revision of the Gateway chart by setting the revision value:

```
helm install istio-ingress-canary istio/gateway -n istio-ingress --set revision=canary --wait --version=1.21.2 --set service.type=None --set labels.istio=ingress,labels.app=istio-ingress,labels.canary=true
```

Pay attention to the flags passed to the gateway Helm chart where we are not activating the service, we are matching the labels used to select the pods and we are adding the label `canary=true`.

Verify Helm:

```
helm ls -n istio-ingress
```

Expect:

```
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
istio-ingress           istio-ingress   1               2024-04-30 16:17:47.511392 -0500 -05    deployed        gateway-1.20.4  1.20.4
istio-ingress-canary    istio-ingress   1               2024-04-30 16:21:41.767588 -0500 -05    deployed        gateway-1.21.2  1.21.2
```

Verify pods:

```
kubectl get pods -L istio.io/rev -n istio-ingress
```

Expect:

```
NAME                                   READY   STATUS    RESTARTS   AGE    REV
istio-ingress-7987dd958c-nqdtb         1/1     Running   0          4m2s   default
istio-ingress-canary-bfbfbd4cd-569wb   1/1     Running   0          8s     canary
```

- Verify the default gateway revision:

```
istioctl proxy-status | grep "$(kubectl -n istio-ingress get pod -l istio.io/rev=default -o jsonpath='{.items..metadata.name}')" | awk '{print $10}'
```

Expect:

```
1.20.4
```

- Check the canary ingress gateway revision:

```
istioctl proxy-status | grep "$(kubectl -n istio-ingress get pod -l canary=true -o jsonpath='{.items..metadata.name}')" | awk '{print $10}'
```

Expect:

```
1.21.2
```

- In order to `canary` test the new gateway instance we are going to edit the Istio Gateway configuration to match the new instance, before that, lets check the k8s endpoints to make sure the service is selecting both gateway instances:

```
kubectl get endpoints -n istio-ingress -o yaml
```

Expect something along the lines:

```
...
apiVersion: v1
    kind: Endpoints
    metadata:
      annotations:
        endpoints.kubernetes.io/last-change-trigger-time: "2024-05-01T02:38:31Z"
      creationTimestamp: "2024-05-01T02:35:19Z"
      labels:
        app: istio-ingress
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: istio-ingress
        app.kubernetes.io/version: 1.20.4
        helm.sh/chart: gateway-1.20.4
        istio: ingress
      name: istio-ingress
      namespace: istio-ingress
      resourceVersion: "85128"
      uid: d82fad8b-3e8a-4cf0-8f88-2507587b89b7
    subsets:
      - addresses:
          - ip: 10.244.1.15
            nodeName: zeroDownTime-worker
            targetRef:
              kind: Pod
              name: istio-ingress-7987dd958c-p7hds
              namespace: istio-ingress
              uid: a84391eb-44b0-4894-95c0-e2be2c457e29
          - ip: 10.244.1.16
            nodeName: zeroDownTime-worker
            targetRef:
              kind: Pod
              name: istio-ingress-canary-6877b466fb-nj7k8
              namespace: istio-ingress
              uid: 6287abf0-37ff-4d62-b805-540edc335621
...
```

Note how the endpoint contains the two ingress gateway pods, one the default revision and one the canary revision.

- Edit the Istio Gateway configuration to add the selector:

```
kubectl edit gateways.networking.istio.io bookinfo-gateway
```

and add the `canary=true` label for selecting, like this:

```
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: default
spec:
  selector:
    istio: ingress
    canary: "true"
  servers:
  - hosts:
    - bookinfo.example.com
    port:
      name: http
      number: 80
      protocol: HTTP
```

- Add the telemetry API to check logs on the ingress gateways:

```
kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  # no selector specified, applies to all workloads
  accessLogging:
  - providers:
    - name: envoy
EOF
```

- Tail the `canary` ingress gateway logs to see the requests now going to the new gateway:

```
kubectl logs -n istio-ingress istio-ingress-canary-6877b466fb-nj7k8
```

Expect:

```
[2024-05-01T03:22:12.491Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 5038 5037 "10.244.1.1" "curl/8.4.0" "07a690a7-dc72-4986-9692-efdf04a410f9" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:49366 10.244.1.16:80 10.244.1.1:8652 - -
[2024-05-01T03:22:19.606Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 39 39 "10.244.1.1" "curl/8.4.0" "d1b24181-15cb-429a-859d-800ee9f56e2a" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38202 10.244.1.16:80 10.244.1.1:42440 - -
[2024-05-01T03:22:21.714Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 41 40 "10.244.1.1" "curl/8.4.0" "3290e354-32b4-4db5-9afc-286b345970cb" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38202 10.244.1.16:80 10.244.1.1:16488 - -
[2024-05-01T03:22:22.795Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 5036 5035 "10.244.1.1" "curl/8.4.0" "13303974-1727-417a-9d51-6cfd3a6cc71a" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38208 10.244.1.16:80 10.244.1.1:5793 - -
[2024-05-01T03:22:29.913Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 26 25 "10.244.1.1" "curl/8.4.0" "602497e3-4ad3-4d6d-b869-f05b45c27675" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38208 10.244.1.16:80 10.244.1.1:28066 - -
[2024-05-01T03:22:30.975Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 35 34 "10.244.1.1" "curl/8.4.0" "28b0d0f1-971f-4ca2-9260-71e47b034f14" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38208 10.244.1.16:80 10.244.1.1:42628 - -
[2024-05-01T03:22:33.067Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 30 30 "10.244.1.1" "curl/8.4.0" "e46fe4ea-fa49-4f8c-83ce-15312b5d792b" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38202 10.244.1.16:80 10.244.1.1:41927 - -
[2024-05-01T03:22:36.194Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 5030 5030 "10.244.1.1" "curl/8.4.0" "22cb297d-53cb-4c7b-b2b4-f735c687d9c6" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38202 10.244.1.16:80 10.244.1.1:48881 - -
[2024-05-01T03:22:42.255Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 27 27 "10.244.1.1" "curl/8.4.0" "7eb28903-c849-4d6b-bc8d-b70a816d3578" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38208 10.244.1.16:80 10.244.1.1:44310 - -
[2024-05-01T03:22:43.322Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 38 37 "10.244.1.1" "curl/8.4.0" "3be6d61b-0c99-47d5-80ed-aad4f08a09cc" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38202 10.244.1.16:80 10.244.1.1:53152 - -
```

If you tail the logs of the `default` gateway revision there shouldn't be any requests logged since the activation of the telemetry API. Also there shouldn't be any traffic loss in this process but evaluate if your traffic flow requires scaling up replicas of your gateway pods. Now that we now the new ingress gateway is working as expected, remove the label we added to the Istio Gateway configuration to match the new instance and scale down the old `default` deployment:

```
kubectl edit gateways.networking.istio.io bookinfo-gateway
```

Remove the `canary-true` label.

All traffic should flow back to the `default` revision gateway.

- Now scale down the non-canary deployment so the existing service uses the new pod:

```
kubectl scale deployment -n istio-ingress istio-ingress --replicas=0
```

There shouldn't be any loss of traffic. All traffic should flow to the `canary` ingress instance.

- Verify the `canary` logs again to make sure the traffic is being logged:

```
kubectl logs -n istio-ingress istio-ingress-canary-6877b466fb-nj7k8 -f
```

Expect:

```
[2024-05-01T03:29:59.052Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 5041 5041 "10.244.1.1" "curl/8.4.0" "3867c9b3-10ca-45f6-b4b3-93db72166f65" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38202 10.244.1.16:80 10.244.1.1:34461 - -
[2024-05-01T03:30:05.121Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 34 34 "10.244.1.1" "curl/8.4.0" "870419d7-1bac-4053-b1c4-5a8e6d6da17f" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38202 10.244.1.16:80 10.244.1.1:46714 - -
[2024-05-01T03:30:06.183Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 27 27 "10.244.1.1" "curl/8.4.0" "374c7e40-6d0f-48de-9943-248ed8e51849" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38202 10.244.1.16:80 10.244.1.1:18608 - -
[2024-05-01T03:30:07.248Z] "HEAD /productpage HTTP/1.1" 200 - via_upstream - "-" 0 0 28 28 "10.244.1.1" "curl/8.4.0" "b55839bb-1f1d-4b0e-a991-137ef9efa0d4" "bookinfo.example.com" "10.244.1.5:9080" outbound|9080||productpage.default.svc.cluster.local 10.244.1.16:38202 10.244.1.16:80 10.244.1.1:29390 - -
```

- This is how or Zero downtime migration is looking so far:

```
istioctl proxy-status
```

Expect:

```
NAME                                                    CLUSTER        CDS        LDS        EDS        RDS        ECDS         ISTIOD                             VERSION
details-v1-698d88b-dwn6n.default                        Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-6fc46dcbd4-wb4mt            1.20.4
istio-ingress-canary-6877b466fb-nj7k8.istio-ingress     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-canary-7655d86d7c-k6zmb     1.21.2
productpage-v1-675fc69cf-4v4xq.default                  Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-6fc46dcbd4-wb4mt            1.20.4
ratings-v1-6484c4d9bb-92bvr.default                     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-6fc46dcbd4-wb4mt            1.20.4
reviews-v1-5b5d6494f4-rs697.default                     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-6fc46dcbd4-wb4mt            1.20.4
reviews-v2-5b667bcbf8-gnrdf.default                     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-6fc46dcbd4-wb4mt            1.20.4
reviews-v3-5b9bd44f4-975fb.default                      Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-6fc46dcbd4-wb4mt            1.20.4
```

Note how now we only have the `canary` ingress with version 1.21.2.

- Time to migrate the sidecars with zero downtime. First lets take a look at our existing MutatingWebHooks:

```
kubectl get mutatingwebhookconfigurations.admissionregistration.k8s.io --show-labels=true -A
```

Expect:

```
NAME                            WEBHOOKS   AGE     LABELS
istio-sidecar-injector          4          7h15m   app.kubernetes.io/managed-by=Helm,app=sidecar-injector,install.operator.istio.io/owning-resource=unknown,istio.io/rev=default,operator.istio.io/component=Pilot,release=istiod
istio-sidecar-injector-canary   2          6h26m   app.kubernetes.io/managed-by=Helm,app=sidecar-injector,install.operator.istio.io/owning-resource=unknown,istio.io/rev=canary,operator.istio.io/component=Pilot,release=istiod-canary
```

- Remove the istio-injection label from the default namespace:

```
kubectl label ns default istio-injection-
```

Expect:

```
namespace/default unlabeled
```

This will prevent the MutatingWebHook to inject sidecars from the default revision, no worries the existing sidecars are still there:

```
kubectl get pods -n default
```

Expect:

```
NAME                             READY   STATUS    RESTARTS   AGE
details-v1-698d88b-dwn6n         2/2     Running   0          7h5m
productpage-v1-675fc69cf-4v4xq   2/2     Running   0          7h5m
ratings-v1-6484c4d9bb-92bvr      2/2     Running   0          7h5m
reviews-v1-5b5d6494f4-rs697      2/2     Running   0          7h5m
reviews-v2-5b667bcbf8-gnrdf      2/2     Running   0          7h5m
reviews-v3-5b9bd44f4-975fb       2/2     Running   0          7h5m
```

- Label the default namespace with the matching revision:

```
kubectl label ns default istio.io/rev=canary
```

Expect:

```
namespace/default labeled
```

The pods are still using the `default` sidecar revision, we need to rollout a restart to get them to the `canary` revision, for this canary test we are going to use the `details-v1` service:

```
kubectl rollout restart deployment -n default details-v1
```

Expect:

```
deployment.apps/details-v1 restarted
```

Check the pods:

```
kubectl get pods -n default
```

Expect:

```
NAME                             READY   STATUS    RESTARTS   AGE
details-v1-7f8c77d5d7-db5xd      2/2     Running   0          58s
productpage-v1-675fc69cf-4v4xq   2/2     Running   0          7h11m
ratings-v1-6484c4d9bb-92bvr      2/2     Running   0          7h11m
reviews-v1-5b5d6494f4-rs697      2/2     Running   0          7h11m
reviews-v2-5b667bcbf8-gnrdf      2/2     Running   0          7h11m
reviews-v3-5b9bd44f4-975fb       2/2     Running   0          7h11m
```

Now check the proxy config status again:

```
istioctl proxy-status
```

Expect:

```
NAME                                                    CLUSTER        CDS        LDS        EDS        RDS        ECDS         ISTIOD                             VERSION
details-v1-7f8c77d5d7-db5xd.default                     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-canary-7655d86d7c-k6zmb     1.21.2
istio-ingress-canary-6877b466fb-nj7k8.istio-ingress     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-canary-7655d86d7c-k6zmb     1.21.2
productpage-v1-675fc69cf-4v4xq.default                  Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-6fc46dcbd4-wb4mt            1.20.4
ratings-v1-6484c4d9bb-92bvr.default                     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-6fc46dcbd4-wb4mt            1.20.4
reviews-v1-5b5d6494f4-rs697.default                     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-6fc46dcbd4-wb4mt            1.20.4
reviews-v2-5b667bcbf8-gnrdf.default                     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-6fc46dcbd4-wb4mt            1.20.4
reviews-v3-5b9bd44f4-975fb.default                      Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-6fc46dcbd4-wb4mt            1.20.4
```

Note how the `details-v1` proxy is now using the `canary` revision for version 1.21.2. Check the proxy logs to validate the canary test:

```
kubectl logs -n default details-v1-7f8c77d5d7-db5xd -c istio-proxy -f
```

Expect:

```
[2024-05-01T03:46:47.756Z] "GET /details/0 HTTP/1.1" 200 - via_upstream - "-" 0 178 1 1 "-" "curl/8.4.0" "32b7022f-89cc-4590-ad3f-ea186af009b2" "details:9080" "10.244.1.17:9080" inbound|9080|| 127.0.0.6:57339 10.244.1.17:9080 10.244.1.5:41858 outbound_.9080_._.details.default.svc.cluster.local default
[2024-05-01T03:46:53.848Z] "GET /details/0 HTTP/1.1" 200 - via_upstream - "-" 0 178 2 2 "-" "curl/8.4.0" "e730d0f4-0f2d-4431-86e6-cd147a644d73" "details:9080" "10.244.1.17:9080" inbound|9080|| 127.0.0.6:57339 10.244.1.17:9080 10.244.1.5:41858 outbound_.9080_._.details.default.svc.cluster.local default
[2024-05-01T03:46:54.910Z] "GET /details/0 HTTP/1.1" 200 - via_upstream - "-" 0 178 3 2 "-" "curl/8.4.0" "ac533966-4f40-48cd-8222-c18d95809ed9" "details:9080" "10.244.1.17:9080" inbound|9080|| 127.0.0.6:57339 10.244.1.17:9080 10.244.1.5:41854 outbound_.9080_._.details.default.svc.cluster.local default
```

This concludes the proxy is working as expected with the new `canary` revision establishing the `canary` istiod is being able to configure the proxy as expected, so now you can rollout a restart for the rest of the deployments:

```
kubectl rollout restart deployment -n default
```

Expect:

```
deployment.apps/details-v1 restarted
deployment.apps/productpage-v1 restarted
deployment.apps/ratings-v1 restarted
deployment.apps/reviews-v1 restarted
deployment.apps/reviews-v2 restarted
deployment.apps/reviews-v3 restarted
```

Validate the configuration once again:

```
istioctl proxy-status
```

Expect:

```
NAME                                                    CLUSTER        CDS        LDS        EDS        RDS        ECDS         ISTIOD                             VERSION
details-v1-84984f6bdd-vk8g7.default                     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-canary-7655d86d7c-k6zmb     1.21.2
istio-ingress-canary-6877b466fb-nj7k8.istio-ingress     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-canary-7655d86d7c-k6zmb     1.21.2
productpage-v1-6f58445988-9zfph.default                 Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-canary-7655d86d7c-k6zmb     1.21.2
ratings-v1-6d849748-jl9qd.default                       Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-canary-7655d86d7c-k6zmb     1.21.2
reviews-v1-7df64b96bb-5kqhm.default                     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-canary-7655d86d7c-k6zmb     1.21.2
reviews-v2-68fb5c9f84-qb8rl.default                     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-canary-7655d86d7c-k6zmb     1.21.2
reviews-v3-85b5bd8df4-nttx5.default                     Kubernetes     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-canary-7655d86d7c-k6zmb     1.21.2
```

Notice how now all the sidecar proxies and ingress gateway are using the `canary` revision for version 1.21.2. During all this procedure there shouldn't be any loss of traffic to the ingress from the loop we ran before.

> **_Disclaimer:_** If doing a canary test to a live workload like the `details-v1` is deemed as too risky, there is always the possibility to perform this same procedure on a test namespace containing all the workload replicas from the `default` namespace in order to mitigate the risk of affecting the main live ones. This has routing considerations when flowing inbound traffic into the mesh.

## Cleanup

Finally is considered good practice not to leave the `canary` revision as canary because there are going to be subsequent upgrades that require a `default` revision as a baseline.

- Check the charts installed:

```
helm ls -A
```

Expect:

```
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
istio-base              istio-system    1               2024-04-30 15:18:39.107697 -0500 -05    deployed        base-1.20.4     1.20.4
istio-ingress           istio-ingress   1               2024-04-30 21:35:19.150856 -0500 -05    deployed        gateway-1.20.4  1.20.4
istio-ingress-canary    istio-ingress   1               2024-04-30 21:38:28.844922 -0500 -05    deployed        gateway-1.21.2  1.21.2
istiod                  istio-system    1               2024-04-30 15:21:12.440376 -0500 -05    deployed        istiod-1.20.4   1.20.4
istiod-canary           istio-system    1               2024-04-30 16:10:21.099893 -0500 -05    deployed        istiod-1.21.2   1.21.2
```

- Delete the `default` istiod revision:

```
helm uninstall istiod -n istio-system
```

Expect:

```
release "istiod" uninstalled
```

> **_Disclaimer:_** Do not uninstall the default gateway revision because that one is the baseline for the k8s service and deleting it will cause an outage

- Upgrade the Istio base chart, making the new revision the default:

```
helm upgrade istio-base istio/base --set defaultRevision=canary -n istio-system --skip-crds --version=1.21.2
```

Expect:

```
Release "istio-base" has been upgraded. Happy Helming!
NAME: istio-base
LAST DEPLOYED: Tue Apr 30 23:04:50 2024
NAMESPACE: istio-system
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
Istio base successfully installed!

To learn more about the release, try:
  $ helm status istio-base
  $ helm get all istio-base
```

- Check the revisions:

```
helm ls -A
```

Expect:

```
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
istio-base              istio-system    2               2024-04-30 23:04:50.891672 -0500 -05    deployed        base-1.21.2     1.21.2
istio-ingress           istio-ingress   1               2024-04-30 21:35:19.150856 -0500 -05    deployed        gateway-1.20.4  1.20.4
istio-ingress-canary    istio-ingress   1               2024-04-30 21:38:28.844922 -0500 -05    deployed        gateway-1.21.2  1.21.2
istiod-canary           istio-system    1               2024-04-30 16:10:21.099893 -0500 -05    deployed        istiod-1.21.2   1.21.2
```