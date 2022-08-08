# Вступление

Сегодня мы поговорим о том как можно скейлить поды в Kubernetes при помощи внешних метрик на примере воркеров RabbitMQ
А так-же я расскажу о том с какими сложностями и подводными камнями можно столкнутся настраивая такую сборку
Так-как я свою очередь столкнувшись с этой проблемой не нашел рабочего и понятного мануала, решил что для себя задокументировать эту 

# Условия задачи
Предположим к нам пришел заказчик(разработчик, лид или егерь из лесу) и попросил сделать чтоб поды в кластере скейлились в зависимости от того сколько накопилось в очереди сообщений

# Как говорил Доктор Дью: Время начинать

Прежде всего нам нужно чтоб в кластере был установлен Prometheus

## Rabbitmq Exporter
Для того чтоб стягивать метрики с ребита нам нужен экспортер. К сожалению встроенный экспортер ребита не вытягивает нужные нам данные, а конкретно количество сообщений разбито по очередям, вхостам, и всему прочему.

Потому берём экспортер он многоуважаемого [kbudde](https://github.com/kbudde/rabbitmq_exporter), он соберает метрики из GUI ребита, потому в самом ребите никаких плагинов ставить не нужно.

<img src="https://img.icons8.com/emoji/16/000000/warning-emoji.png"/>Если морда ребита с SSL сертификатом то нужно будет закинуть сертификаты в контейнер мониторинга, или-же выключить проверку `SKIPVERIFY`

Деплоем чарт с екпортером
```bash

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade \
    --atomic \
    --wait \
    --namespace test-namespace \
    --install rabbitmq-exporter \
    prometheus-community/prometheus-rabbitmq-exporter \
    \
    --set fullnameOverride=rabbitmq-exporter \
    \
    --set prometheus.monitor.enabled=true \
    \
    --set rabbitmq.url=https://RABBITMQ_MANAGEMENT_HOST:RABBITMQ_MANAGEMENT_PORT \
    --set rabbitmq.user=guest \
    --set rabbitmq.password=guest \
    --set rabbitmq.skip_verify=true \
    --set loglevel=debug
```

Дальше проверяем собираются ли метрики самим экспортером
```bash 
kubectl --namespace test-namespace port-forward deployment/rabbitmq-exporter 9419:9419
curl -v http://localhost:9419/metrics | grep rabbitmq_up
```

Если в выхлопе курла мы видим `rabbitmq_up 1` значит экспортер собирает метрики.

### Проверяем метрики в Prometheus
Заходим в веб морду Prometheus в Service Discovery и проверяем есть ли наш таргет.
```
serviceMonitor/test-namespace/rabbitmq-exporter/0 (1 / 5 active targets)
```
И проверяем на всякий случай есть ли сами данные в Graph `rabbitmq_up`

## Настраиваем HorizontalPodAutoscaler

Далее нам нужно выбрать каким методом мы будем получать данные в наш hpa.
Существует 3 апишки для получения метрик:
 * metrics.k8s.io
 * custom.metrics.k8s.io
 * external.metrics.k8s.io
 
В нашем случае мы будем использовать custom метрики. Потому проверяем наличие в апи данных по этим метрикам
```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq .
```
Если данные возвращаются значит у вас стоит [адаптер](https://github.com/kubernetes-sigs/prometheus-adapter) и можно продолжать дальше.
Далее добавляем в чарт HPA
```
{{ if (eq $.Values.horizontalPodAutoscaler.enabled true) }}
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: {{ $.Release.Name }}
spec:
  groups:
    - name: {{ $.Release.Name }}-queue-size # Define the name of your rule
      rules:
        - record: {{ include "workers.hpa_metric_name" . }}_messages_waiting_in_queue # Уникальное имя метрики.
          expr: rabbitmq_queue_messages{queue="{{ $.Values.horizontalPodAutoscaler.rabbitmq.queue }}", vhost="{{ $.Values.horizontalPodAutoscaler.rabbitmq.vhost }}"} # Запрос в Prometheus c фильтрацией 
---
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ $.Release.Name }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ $.Release.Name }}
  minReplicas: {{ $.Values.horizontalPodAutoscaler.minReplicas }}
  maxReplicas: {{ $.Values.horizontalPodAutoscaler.maxReplicas }}
  metrics:
    - type: Object
      object:
        metric:
          name: {{ include "workers.hpa_metric_name" . }}_messages_waiting_in_queue
        describedObject:
          apiVersion: v1
          kind: Namespace
          name: {{ $.Release.Namespace }}
        target:
          type: Value
          value: {{ $.Values.horizontalPodAutoscaler.rabbitmq.value }}

{{- end }}
```

<img src="https://img.icons8.com/emoji/16/000000/warning-emoji.png"/>Важные моменты
 * метрика `{{ include "workers.hpa_metric_name" . }}_messages_waiting_in_queue` должна быть уникальна, чтоб система не конфликтовала если в неймспейсе есть другие hpa
 * Запрос к Prometheus должен отдавать одну запись. В случае когда запрос отдает несколько значений могут быть непредвиденное поведение 
 Полезные статьи по этой теме:
* https://stackoverflow.com/questions/66423005/hpa-with-different-namespaces
* https://itnext.io/horizontal-pod-autoscaling-with-custom-metric-from-different-namespace-f19f8446143b
