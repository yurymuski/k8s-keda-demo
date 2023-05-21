# k8s-keda-demo

### How to scale k8s deployments depending on custom prometheus metric using KEDA

Alternative way is not use `prometheus-adapter` and `HPA` [demo-repo](https://github.com/yurymuski/k8s-hpa-demo)

---
## setup kube-prometheus-stack
https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# deploy kube-prometheus-stack
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 45.28.0 -f kube-prometheus-stack/values.yaml

# access grafana
kubectl port-forward service/kube-prometheus-stack-grafana 8080:80

```

---
## setup app with metrics (bitnami nginx as example)
https://github.com/bitnami/charts/blob/main/bitnami/nginx/values.yaml

```sh
# deploy app
helm upgrade --install test-app oci://registry-1.docker.io/bitnamicharts/nginx --version 14.2.1 -f app/values.yaml
```
App will create public loadbalancer
```sh
# access app by IP
kubectl get svc --namespace default test-app-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```
Generate some load and check metric `sum(rate(nginx_http_requests_total[1m]))` at grafana

---
## setup KEDA
https://github.com/kedacore/charts/blob/main/keda/values.yaml

```sh
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

# deploy KEDA
helm upgrade --install keda kedacore/keda --version 2.10.2

# create keda scaledobject
kubectl apply -f keda/scaledobject.yaml

# check that scaledobject is Ready and Active
kubectl get scaledobjects.keda.sh

# check k8s API for s0-prometheus-nginx_http_requests_total
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1 | jq .

# check for current metric value
# https://keda.sh/docs/2.10/operate/metrics-server/
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/default/s0-prometheus-nginx_http_requests_total?labelSelector=scaledobject.keda.sh%2Fname%3Dkeda-prometheus-test-app-nginx"

```

---
## Generate some load and check pod scaling

```sh
# generate load
plow http://LB_IP_ADDRESS/ --rate 50/s -c 10


# watch scaling status
kubectl get hpa  --watch
```

---
## Clean up

```sh
helm uninstall test-app
kubectl delete -f keda/scaledobject.yaml
helm uninstall keda
helm uninstall kube-prometheus-stack # NOTE: helm uninstall does not remove pvc
kubectl delete pvc -l app.kubernetes.io/name=prometheus
```
