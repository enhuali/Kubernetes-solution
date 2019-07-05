# Prometheus使用

日常需要维护的内容有3种： metric、config、rules

mertic由crd prometheuses.monitoring.coreos.com 控制

config由crd servicemonitors.monitoring.coreos.com控制

rules由crd prometheusrules.monitoring.coreos.com控制



## redis-exporter

- 部署redis-exporter，获取监控指标

  ```
  // 进入prometheus charts目录
  cd /etc/ansible/extension/monitoring/prometheus-operator/charts
  
  // 获取redis-exporter包
  helm fetch stable/prometheus-redis-exporter --version 2.0.2
  
  // 添加需要监控的redis/redis集群
  vim prometheus-redis-exporter/values.yaml
  redisAddress: redis://10.216.91.142:6379,redis://redis.sim.svc:6379,redis://redis.monitoring.svc:6379
  // 需要注意的是在监控不同namespaces时，需要加上其namespace名称，且多节点监控时用逗号隔开在一行书写
  ```

- 添加自动发现即定义job_name

  ```
  // 进入operator工作目录
  cd /etc/ansible/extension/monitoring
  
  // 新增prome-servicemonitors.yaml文件
  vim prome-servicemonitors.yaml
  prometheus:
    additionalServiceMonitors:
      - name: redis-exporter
    #    jobLabel: xxx
        selector:
          matchLabels:
            app: prometheus-redis-exporter
            release: promethues-operator
        namespaceSelector:
          matchNames:
            - monitoring
        endpoints:
          - targetPort: 9121
          #- port: metrics
            path: /metrics
            interval: 10s
  ```

  `additionalServiceMonitors` 添加servicemonitor

  `matchLabels` 可以通过查看部署redis-exporter的yaml来获取其label

  `matchNames` exporter所在的namespace，如果是多namespace可设置为NAMESPACE（未验证）

  `endpoint` 必须字段，定义exporter的targetport

- 添加rules即报警规则

  ```
  // 进入operator工作目录
  cd /etc/ansible/extension/monitoring
  
  // 新增prome-rules.yaml文件
  vim prome-rules.yaml
  additionalPrometheusRules:
    - name: chengdan
      groups:
        - name: chengdan
          rules:
          - alert: chengdan2
            annotations:
              message: this is test
          #- record: chengdan2
            expr: abs(node_timex_offset_seconds{job="node-exporter"}) > 0.10
            for: 2m
            labels:
              severity: warning
  ```

  `expr` 规则设定，可在prometheus面板上进行测试验证后设立

- 加载以上配置文件

  ```
  helm upgrade  promethues-operator --namespace monitoring -f prome-settings.yaml -f prome-rules.yaml -f prome-servicemonitors.yaml prometheus-operator
  ```

  





# 初始定制化

rules和servicemonitor均可在安装operator时就进行集成，大致操作如下

**rules**

```
// 在/etc/ansible/extension/monitoring/prometheus-operator/templates/prometheus/rules下创建文件，比如lienhua

{{- if and .Values.defaultRules.create }}
apiVersion: {{ printf "%s/v1" (.Values.prometheusOperator.crdApiGroup | default "monitoring.coreos.com") }}
kind: PrometheusRule
metadata:
  name: {{ printf "%s-%s" (include "prometheus-operator.fullname" .) "lienhua" | trunc 63 | trimSuffix "-" }}
  labels:
    app: {{ template "prometheus-operator.name" . }}
{{ include "prometheus-operator.labels" . | indent 4 }}
{{- if .Values.defaultRules.labels }}
{{ toYaml .Values.defaultRules.labels | indent 4 }}
{{- end }}
{{- if .Values.defaultRules.annotations }}
  annotations:
{{ toYaml .Values.defaultRules.annotations | indent 4 }}
{{- end }}
spec:
  groups:
  - name: lienhua
    rules:
    - alert: lienhua
      annotations:
        message: Clock skew detected on node-exporter {{`{{ $labels.namespace }}`}}/{{`{{ $labels.pod }}`}}. Ensure NTP is configured correctly on this host.
      expr: abs(node_timex_offset_seconds{job="node-exporter"}) > 0.10
      for: 2m
      labels:
        severity: warning
{{- end }}
```



**servicemonitor**

```
// 在prome-settings.yaml中添加内容

redisExporter:
  enabled: true

  ## Use the value configured in prometheus-node-exporter.podLabels
  ##
  jobLabel: jobLabel

  serviceMonitor:
    ## Scrape interval. If not set, the Prometheus default scrape interval is used.
    ##
    interval: ""

    ##  metric relabel configs to apply to samples before ingestion.
    ##
    metricRelabelings: []
    # - sourceLabels: [__name__]
    #   separator: ;
    #   regex: ^node_mountstats_nfs_(event|operations|transport)_.+
    #   replacement: $1
    #   action: drop

    ##  relabel configs to apply to samples before ingestion.
    ##
    relabelings: []

// 然后在/etc/ansible/extension/monitoring/prometheus-operator/templates/exporters中创建redis-exporter目录下servicemonitor.yaml （主要修改有关redis的项）

{{- if .Values.redisExporter.enabled }}
apiVersion: {{ printf "%s/v1" (.Values.prometheusOperator.crdApiGroup | default "monitoring.coreos.com") }}
kind: ServiceMonitor
metadata:
  name: {{ template "prometheus-operator.fullname" . }}-redis-exporter
  labels:
    app: {{ template "prometheus-operator.name" . }}-redis-exporter
{{ include "prometheus-operator.labels" . | indent 4 }}
spec:
  jobLabel: {{ .Values.redisExporter.jobLabel }}
  selector:
    matchLabels:
      app: prometheus-redis-exporter
      release: {{ .Release.Name }}
  endpoints:
  - port: metrics
    {{- if .Values.redisExporter.serviceMonitor.interval }}
    interval: {{ .Values.redisExporter.serviceMonitor.interval }}
    {{- end }}
{{- if .Values.redisExporter.serviceMonitor.metricRelabelings }}
    metricRelabelings:
{{ toYaml .Values.redisExporter.serviceMonitor.metricRelabelings | indent 4 }}
{{- end }}
{{- if .Values.redisExporter.serviceMonitor.relabelings }}
    relabelings:
{{ toYaml .Values.redisExporter.serviceMonitor.relabelings | indent 4 }}
{{- end }}
{{- end }}
```



