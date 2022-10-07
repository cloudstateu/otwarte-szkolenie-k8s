# Observability Prometheus, Grafana, Loki

## Instalacja nginx-ingress

```shell
kubectl create ns nginx-ingress
helm repo add nginx-stable https://helm.nginx.com/stable
helm install nginx-ingress nginx-stable/nginx-ingress -n nginx-ingress --set controller.enableLatencyMetrics=true --set prometheus.create=true --set controller.config.name=nginx-
```

Dostosuj adresy IP w konfiguracji obiektów `Ingress`

```shell
IP=$(kubectl get svc nginx-ingress-nginx-ingress -n nginx-ingress -ojson | jq -j '.status.loadBalancer.ingress[].ip')
sed -i '' -e "s/IP_TO_REPLACE/$IP/g" hipstershop/k8s-manifest.yaml
sed -i '' -e "s/IP_TO_REPLACE/$IP/g" grafana/ingress.yaml
```

Uruchom [nip.io](https://nip.io/):

```shell
cd nip.io
./build_and_run_docker.sh
cd ..
```

## Instalacja aplikacji

> Wdrażana aplikacja pochodzi z repozytorium: https://github.com/GoogleCloudPlatform/microservices-demo

```shell
kubectl create ns hipster-shop
kubectl -n hipster-shop create rolebinding default-view --clusterrole=view --serviceaccount=hipster-shop:default
kubectl -n hipster-shop apply -f hipstershop/k8s-manifest.yaml
```

Wyświetl aplikacje w przeglądarce używając adresu wygenerowanego komendą poniżej:

```
k get ing -n hipster-shop -ojson | jq '.items | first | .spec.rules | first | .host'
```

## Instalacja Prometheus + Grafana

```shell
kubectl create ns observability
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus -n observability prometheus-community/kube-prometheus-stack
```

Wyświetl stronę Grafana znajdującą się pod adresem:

```shell
kubectl get ing -n observability -ojson | jq '.items | first | .spec.rules | first | .host'
```

Login i hasło znajdziesz za pomocą poniższych komend:

```shell
kubectl get secret -n observability prometheus-grafana -o jsonpath="{.data.admin-user}" | base64 -d
kubectl get secret -n observability prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```

```shell
kubectl apply -f prometheus/servicemonitor.yaml
```

## Instalacja Grafana Loki

```shell
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install loki grafana/loki-stack
```

Zaktualizuj ConfigMap NGINX do logowania requestów zgodnie z nowym formatem:

```shell
data:
  log_format: $remote_addr [$time_local] $request $status $body_bytes_sent $request_time $upstream_addr $upstream_response_time $proxy_host $upstream_status $resource_name $resource_type $resource_namespace $service
```

Utwórz dashboard z logami:

```shell
{app="nginx-ingress-nginx-ingress"}
| pattern `<remote_addr> - - [<time_local>] "<method> <request> <_>" <status> <body_bytes_sent> "-" "<http_user_agent>" "-"`
```

### Dashboards

Ingress Health:

```shell
nginx_ingress_nginx_up
```

Total HTTP Requests:

```shell
rate(nginx_ingress_nginx_http_requests_total[30s])
```

Request by method:

```shell
sum by (method) (rate({app="nginx-ingress-nginx-ingress"}
| pattern `<remote_addr> - - [<time_local>] "<method> <request> <_>" <status> <body_bytes_sent> "-" "<http_user_agent>" "-"` | method!="" [1m]))
```

Bytes sent:

```shell
avg_over_time({app="nginx-ingress-nginx-ingress"}
| pattern `<remote_addr> - - [<time_local>] "<method> <request> <_>" <status> <body_bytes_sent> "-" "<http_user_agent>" "-"`
| unwrap body_bytes_sent
| __error__=""[30s]) by (app)

```