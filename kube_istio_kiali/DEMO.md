### Minikube (v0.28.2)

Start minikube:

```shell
minikube start --kubernetes-version=v1.10.0 --vm-driver=kvm2 --memory=8192
```

We need to enable ingress addon in order to get Kiali running out of the box:

```shell
minikube addons enable ingress
```

### Helm (v2.9.1)

```shell
kubectl create -f install/kubernetes/helm/helm-service-account.yaml
helm init --service-account tiller
```

### Istio (0.8.0 using Helm)

```shell
helm install install/kubernetes/helm/istio --name istio --namespace istio-system --set tracing.enabled=true
kubectl apply -f install/kubernetes/addons/grafana.yaml -n istio-system
kubectl label namespace default istio-injection=enabled
```

### Bookinfo

Install bookinfo sample app:

```shell
kubectl apply -f samples/bookinfo/kube/bookinfo.yaml
istioctl create -f samples/bookinfo/routing/bookinfo-gateway.yaml
```

Useful variables to have:

```shell
export INGRESS_HOST=$(minikube ip)
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

echo "Ingress gateway: http://${GATEWAY_URL}"
```

### Kiali

```shell
curl https://raw.githubusercontent.com/kiali/kiali/master/deploy/kubernetes/kiali-configmap.yaml | VERSION_LABEL=master envsubst | kubectl create -n istio-system -f -
curl https://raw.githubusercontent.com/kiali/kiali/master/deploy/kubernetes/kiali-secrets.yaml | VERSION_LABEL=master envsubst | kubectl create -n istio-system -f -
curl https://raw.githubusercontent.com/kiali/kiali/master/deploy/kubernetes/kiali.yaml | IMAGE_NAME=kiali/kiali IMAGE_VERSION=latest NAMESPACE=istio-system VERSION_LABEL=master VERBOSE_MODE=4 envsubst | kubectl create -n istio-system -f -
```

```shell
echo "Kiali URL: http://${INGRESS_HOST}"
```

Create an infinite loop to generate data:

```shell
while true; do curl -o /dev/null -s -w "%{http_code}\n" http://${GATEWAY_URL}/productpage; done
```

### Jaeger UI

```shell
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686
```

Access Jaeger UI through [http://localhost:16686](http://localhost:16686)

### Prometheus UI

```shell
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090
```

Access Prometheus UI through [http://localhost:9090](http://localhost:9090)

### Grafana UI

```shell
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000
```

Access Prometheus UI through [http://localhost:3000](http://localhost:3000)


### Scripts for the demo

1. Create a rule to route every request to Reviews v1.

   ```shell
   istioctl create -f samples/bookinfo/routing/route-rule-all-v1.yaml
   ```

2. Create a rule where user Jason goes routes to v2 and everyone else routes to v3.

   ```shell
   istioctl replace -f route-rule-reviews-jason-v2-v3.yaml
   ```

3. Clean it up:

   ```shell
   istioctl delete -f samples/bookinfo/routing/route-rule-all-v1.yaml
   ```

### Prometheus queries to try

1. All istio requests

  ```text
  istio_request_count
  ```

2. All requests to the reviews service

  ```text
  istio_request_count{destination_service="reviews.default.svc.cluster.local"}
  ```

3. All requests to the reviews service version v2

  ```text
  istio_request_count{destination_service="reviews.default.svc.cluster.local",destination_version="v2"}
  ```

4. Rate of requests over the past 5 minutes