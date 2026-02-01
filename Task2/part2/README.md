## Часть 2. Динамическая маршрутизация на основании показателей количества запросов в секунду

# Задание 2. Динамическая маршрутизация на основании показателей количества запросов в секунду

## Настройка кластера

```bash
# Установите Prometheus в вашем кластере
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-operator prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```
Получаем вот такой ответ:
```bash
NAME: prometheus-operator
LAST DEPLOYED: Sun Feb  1 18:35:48 2026
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=prometheus-operator"

Get Grafana 'admin' user password by running:

  kubectl --namespace monitoring get secrets prometheus-operator-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

Access Grafana local instance:

  export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=prometheus-operator" -oname)
  kubectl --namespace monitoring port-forward $POD_NAME 3000

Get your grafana admin user password by running:

  kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo


Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.

```
Продолжаем настройку:

```bash
# Проверить
kubectl get pods -n monitoring

# Применить манифесты
kubectl apply -f ./Task2/part2/app/k8s/deployment.yaml
kubectl apply -f ./Task2/part2/app/k8s/service-metric.yaml

# Установить Prometheus Adapter
helm install prometheus-adapter prometheus-community/prometheus-adapter --namespace monitoring

helm upgrade prometheus-adapter prometheus-community/prometheus-adapter \
  -n monitoring \
  --set prometheus.url=http://prometheus-kube-prometheus-prometheus.monitoring.svc \
  --set prometheus.port=9090

# Применить configMap и prometheus-adapter
kubectl apply -f ./Task2/part2/app/k8s/podmonitor.yaml 
kubectl apply -f ./Task2/part2/app/k8s/prometheus-adapter-config.yaml

# Открыть Prometeus UI
kubectl port-forward svc/prometheus-kube-prometheus-prometheus  9090:9090 -n monitoring
```

Проверки:
```bash
# Должен вывести непустой resources[]
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1"

# Должен вывести непустой resources[]
kubectl get --raw \
"/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second"
```

Применить HPA
```bash
kubectl apply -f ./Task2/part2/app/k8s/hpa-rps.yaml
```

Наблюдение
```bash
kubectl get hpa -w
```

## Решение