## Часть 2. Динамическая маршрутизация на основании показателей количества запросов в секунду

# Задание 2. Динамическая маршрутизация на основании показателей количества запросов в секунду

## Настройка кластера
Установите Prometheus в вашем кластере
```bash
# Установите Prometheus в вашем кластере
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.enabled=true \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false
```
Получаем вот такой ответ:
```bash
NAME: prometheus-stack
LAST DEPLOYED: Mon Feb  2 16:20:08 2026
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=prometheus-stack"

Get Grafana 'admin' user password by running:

  kubectl --namespace monitoring get secrets prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

Access Grafana local instance:

  export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=prometheus-stack" -oname)
  kubectl --namespace monitoring port-forward $POD_NAME 3000

Get your grafana admin user password by running:

  kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo


Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.

```
Проверяем установку Prometheus:
```bash
[slava@altlinux-vm-1 k8s]$ kubectl --namespace monitoring get pods -l "release=prometheus-stack"
NAME                                                   READY   STATUS    RESTARTS   AGE
prometheus-stack-kube-prom-operator-6779d49c95-gb5qj   1/1     Running   0          2m6s
prometheus-stack-kube-state-metrics-85df8cf9b4-pp8jk   1/1     Running   0          2m6s
prometheus-stack-prometheus-node-exporter-kq7h9        1/1     Running   0          2m6s
```

Продолжаем настройку. Установим приложение и ServiceMonitor.

```bash

# Применить манифесты
# Установка приложения
kubectl apply -f ./Task2/part2/app/k8s/deployment.yaml
kubectl apply -f ./Task2/part2/app/k8s/service.yaml

# Установить ServiceMonitor
kubectl apply -f ./Task2/part2/app/k8s/servicemonitor.yaml

##
```
Проверка работы
 - Проверяем targets в Prometheus
```bash
# Пробрасываем порт Prometheus
kubectl port-forward -n monitoring svc/prometheus-stack-kube-prom-prometheus 9090:9090
```
Откройте http://localhost:9090/targets
Должны увидеть:
```bash
serviceMonitor/monitoring/myapp-service-monitor/0 в состоянии UP
```

Теперь установим Prometheus Adapter, через который K8s сможет видеть метрики и использовать их для HPA.
```
# Установить Prometheus Adapter
helm install prometheus-adapter prometheus-community/prometheus-adapter -f ./Task2/part2/app/k8s/values.yaml -n monitoring

#NAME: prometheus-adapter
#LAST DEPLOYED: Sun Feb  1 19:48:08 2026
#NAMESPACE: monitoring
#STATUS: deployed
#REVISION: 1
#TEST SUITE: None
#NOTES:
#prometheus-adapter has been deployed.
#In a few minutes you should be able to list metrics using the following command(s):
#
#  kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1

# Открыть Prometeus UI
kubectl port-forward svc/prometheus-operator-kube-p-prometheus  9090:9090 -n monitoring
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

## Результат - метрик нет

minikube сдох
попатки переустановить prometheus не увенчались успехом
Убил несколько часов на эти переустановки.

```bash
[slava@altlinux-vm-1 k8s]$ helm install prometheus-operator prometheus-community/kube-prometheus-stack -n monitoring
Error: INSTALLATION FAILED: create: failed to create: namespaces "monitoring" not found
[slava@altlinux-vm-1 k8s]$ kubectl create namespace monitoring
namespace/monitoring created
[slava@altlinux-vm-1 k8s]$ helm install prometheus-operator prometheus-community/kube-prometheus-stack -n monitoring
Error: INSTALLATION FAILED: failed to create resource: Internal error occurred: failed calling webhook "prometheusrulevalidate.monitoring.coreos.com": failed to call webhook: Post "https://prometheus-operator-kube-p-operator.monitoring.svc:443/admission-prometheusrules/validate?timeout=10s": dial tcp 10.108.90.123:443: connect: connection refused
[slava@altlinux-vm-1 k8s]$ helm search repo prometheus-community/kube-prometheus-stack
NAME                                            CHART VERSION   APP VERSION     DESCRIPTION                                       
prometheus-community/kube-prometheus-stack      81.4.2          v0.88.1         kube-prometheus-stack collects Kubernetes manif...
[slava@altlinux-vm-1 k8s]$ helm install prometheus-operator prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
Error: INSTALLATION FAILED: failed to create resource: Internal error occurred: failed calling webhook "prometheusrulevalidate.monitoring.coreos.com": failed to call webhook: Post "https://prometheus-operator-kube-p-operator.monitoring.svc:443/admission-prometheusrules/validate?timeout=10s": dial tcp 10.99.18.189:443: connect: connection refused
[slava@altlinux-vm-1 k8s]$ 

```

# Выводы
Должно работать, но протестировать не получилось