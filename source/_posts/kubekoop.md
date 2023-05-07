---
title: 快速安装阿里开源的kubeskoop，配合Prometheus+Grafana
date: 2023-05-07 18:36:41
categories: 
- 技术文档
tags:
- Prometheus
- Grafana
- kubeskoop
---
## 前言
> 目的是介绍一下默认安装方式进行快速安装kubeskoop，细节可以参考安装地址官网介绍, 作用也不做过多介绍了

当然社区有kubeskoop+Prometheus+Grafana的整个的bundle的manifests，直接apply就可以看到效果，然后import json到Grafana : [kubeskoop dashboard](https://github.com/alibaba/kubeskoop/blob/main/deploy/resource/kubeskoop-exporter-dashboard.json).

```
kubectl create -f  \
  https://raw.githubusercontent.com/alibaba/kubeskoop/main/deploy/skoopbundle.yaml
```

> 以下是解耦的方式安装

<!--more-->

## 软件版本
|软件|版本|安装地址|
|:-|:-|:-|
|kubeskoop|Helm 0.1.1 / App: 0.1.1|[kubeskoop](https://github.com/alibaba/kubeskoop/tree/main/deploy/helm)|
|Prometheus|Helm 22.2.0 / App: v2.43.0|[promtheus](https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus/README.md)|
|Grafana|Helm 6.56.1 / App: 9.5.0|[Grafana](https://github.com/grafana/helm-charts/blob/main/charts/grafana/README.md)|
|helm3|v3.11.1|[Helm](https://github.com/helm/helm/releases/tag/v3.11.1)|
|Kubernetes|v1.20.11|-|

## Helm
可能用到的`helm` 命令
```bash
# 添加仓库
$ sudo  helm repo add $myRepoName  https://xxx

# 更新repo源
$ sudo helm repo update

# 查询repo有那些charts和那些版本
$ sudo helm search repo $myRepoName --versions

# 安装应用
$ sudo helm install $myApp $myRepoName/$chartName --version xxx

# 安装本地charts
$ sudo helm install $myApp ./$chartName-$version.tgz

# 卸载
$ sudo helm uninstall m$myApp

# 万物皆可--help
$ sudo helm --help
```

## kubeskoop
> [kubeskoop](https://github.com/alibaba/kubeskoop/tree/main/deploy/helm)

### 下载

repo源和charts包有点问题，那就先克隆下来本地安装
```
$ sudo git clone https://github.com/alibaba/kubeskoop.git
```
### 安装

```bash
$sudo  helm install --create-namespace --namespace kubeskoop  kubeskoop  ./kubeskoop/deploy/helm/  --debug

install.go:194: [debug] Original chart version: ""
install.go:211: [debug] CHART PATH: /root/skoop/kubeskoop/deploy/helm

client.go:133: [debug] creating 1 resource(s)
client.go:133: [debug] creating 2 resource(s)
NAME: kubeskoop
LAST DEPLOYED: Sun May  7 17:35:31 2023
NAMESPACE: kubeskoop
STATUS: deployed
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
appName: kubeskoop-exporter
config:
  enableEventServer: true
  enableMetricServer: true
  eventProbes:
  - tcpreset
  - packetloss
  eventServerPort: 19102
  metricCacheInterval: 15
  metricLabelVerbose: true
  metricProbes:
  - netdev
  - io
  - socketlatency
  - packetloss
  - net_softirq
  - tcpext
  - tcpsummary
  - tcp
  - sock
  - softnet
  - udp
  - virtcmdlatency
  - kernellatency
  metricServerPort: 9102
  remoteLokiAddress: loki-service.skoop.svc.cluster.local
debugMode: false
expose_labels:
- source: ip
  type: meta
- replace: component
  source: app
  type: label
image:
  pullPolicy: IfNotPresent
  repository: registry.cn-hangzhou.aliyuncs.com/acs/kubeskoop-exporter
  repositoryPre: registry.cn-hangzhou.aliyuncs.com/build-test/kubeskoop-exporter
  tag: v0.0.1-g857c8be-aliyun
initcontainer:
  pullPolicy: Always
  repository: registry.cn-hangzhou.aliyuncs.com/acs/btfhack
  tag: latest
name: kubeskoop-exporter
namespace: kubeskoop
nodeSelector: {}
preRelease: false
resources:
  limits:
    cpu: 500m
    memory: 1024Mi
  requests:
    cpu: 500m
    memory: 1024Mi
runtimeEndpoint: /run/containerd/containerd.sock

HOOKS:
MANIFEST:
---
# Source: kubeskoop-exporter/templates/configMap.yaml
apiVersion: v1
data:
  config.yaml: |-
    debugmode: false
    metric_config:
      interval: 15
      port: 9102
      verbose: true
      expose_labels:
        - source: ip
          type: meta
        - replace: component
          source: app
          type: label
      probes:
      - netdev
      - io
      - socketlatency
      - packetloss
      - net_softirq
      - tcpext
      - tcpsummary
      - tcp
      - sock
      - softnet
      - udp
      - virtcmdlatency
      - kernellatency
    event_config:
      port: 19102
      loki_enable: true
      loki_address: loki-service.skoop.svc.cluster.local
      probes:
      - tcpreset
      - packetloss
kind: ConfigMap
metadata:
  name: kubeskoop-config
  namespace: kubeskoop
---
# Source: kubeskoop-exporter/templates/deamonSet.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kubeskoop-exporter
  namespace: kubeskoop
  labels:
    app: kubeskoop-exporter
spec:
  selector:
    matchLabels:
      app: kubeskoop-exporter
  template:
    metadata:
      labels:
        app: kubeskoop-exporter
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "9102"
        prometheus.io/scheme: http
        prometheus.io/scrape: "true"
      name: kubeskoop-exporter
    spec:
      hostNetwork: true
      hostPID: true
      hostIPC: true
      dnsPolicy: ClusterFirstWithHostNet
      initContainers:
        - name: preparels

          image: "registry.cn-hangzhou.aliyuncs.com/acs/btfhack:latest"
          volumeMounts:
            - name: bpfdir
              mountPath: /etc/net-exporter/btf
            - mountPath: /boot/
              name: boot
          command: [btfhack, discover,-p ,/etc/net-exporter/btf/]
      containers:
      - image:  "registry.cn-hangzhou.aliyuncs.com/acs/kubeskoop-exporter:v0.0.1-g857c8be-aliyun"
        name: inspector
        env:
        - name: INSPECTOR_NODENAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
          - name: configvolume
            mountPath: /etc/config/
          - name: bpfdir
            mountPath: /etc/net-exporter/btf
          - name: procfs
            mountPath: /proc
          - mountPath: /run/containerd/containerd.sock
            name: runtimeendpoint
          - mountPath: /var/run/
            name: rundir
          - mountPath: /sys/fs/bpf
            name: bpfmap
            mountPropagation: HostToContainer
          - mountPath: /sys/kernel/debug
            name: bpfdebugfs
            mountPropagation: HostToContainer
          - mountPath: /etc/node-hostname
            name: hostname
        command: [/bin/inspector,server]
        securityContext:
          privileged: true
        resources:
          limits:
            cpu: 500m
            memory: 1024Mi
          requests:
            cpu: 500m
            memory: 1024Mi
      volumes:
        - name: procfs
          hostPath:
            path: /proc
        - name: runtimeendpoint
          hostPath:
            path: /run/containerd/containerd.sock
        - name: boot
          hostPath:
            path: /boot/
        - name: rundir
          hostPath:
            path: /var/run/
        - hostPath:
            path: /sys/fs/bpf
            type: DirectoryOrCreate
          name: bpfmap
        - hostPath:
            path: /sys/kernel/debug
          name: bpfdebugfs
        - name: hostname
          hostPath:
            path: /etc/hostname
            type: FileOrCreate
        - name: configvolume
          configMap:
            name: kubeskoop-config
        - name: bpfdir
          emptyDir: {}
```
### 查看
正常运行就ok了

```bash
$ sudo kubectl  get deployment,pod,cm -n kubeskoop
NAME                           READY   STATUS    RESTARTS   AGE
pod/kubeskoop-exporter-87bzw   1/1     Running   0          3m21s
pod/kubeskoop-exporter-t9hbn   1/1     Running   0          3m21s

NAME                         DATA   AGE
configmap/kubeskoop-config   1      3m21s
```

## Prometheus
### Helm安装
> This chart bootstraps a Prometheus deployment on a Kubernetes cluster using the Helm package manager.

安装地址: [Promtheus](https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus/README.md)

```bash
$ sudo helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ sudo helm repo update
$ sudo helm install myprom prometheus-community/prometheus --namespace kubeskoop
```

### 配置configmap
安装后再configmap的prometheus.yml中的scrape_configs里追加如下一个job
```yaml
apiVersion: v1
data:
  alerting_rules.yml: ...
  alerts: ..
  allow-snippet-annotations: 'false'
  prometheus.yml: |-
    global:
      evaluation_interval: 1m
      scrape_interval: 1m
      scrape_timeout: 10s
    rule_files:
    - /etc/config/recording_rules.yml
    - /etc/config/alerting_rules.yml
    - /etc/config/rules
    - /etc/config/alerts
    scrape_configs:
    - job_name: ...
    - job_name: kubeskoop     # kubeskoop的job配置抓取这里追加
      honor_timestamps: true
      scrape_interval: 30s
      scrape_timeout: 30s
      metrics_path: /metrics
      scheme: http
      follow_redirects: true
      relabel_configs:
      - source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape
        separator: ;
        regex: 'true'
        replacement: $1
        action: keep
      - source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        separator: ;
        regex: '9102'
        replacement: $1
        action: keep
      - source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        separator: ;
        regex: (.+)
        target_label: __metrics_path__
        replacement: $1
        action: replace
      - source_labels:
        - __address__
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        separator: ;
        regex: ([^:]+)(?::\d+)?;(\d+)
        target_label: __address__
        replacement: $1:$2
        action: replace
      - separator: ;
        regex: __meta_kubernetes_pod_label_(.+)
        replacement: $1
        action: labelmap
      - source_labels:
        - __meta_kubernetes_namespace
        separator: ;
        regex: (.*)
        target_label: kubernetes_namespace
        replacement: $1
        action: replace
      - source_labels:
        - __meta_kubernetes_pod_name
        separator: ;
        regex: (.*)
        target_label: kubernetes_pod_name
        replacement: $1
        action: replace
      kubernetes_sd_configs:
      - role: pod
        follow_redirects: true
        namespaces:
          names:
          - kubeskoop    # 这里可以加上名称空间kubeskoop的pod所在的ns
    # ...
```

### 查询日志
查询prometheus-server的pod输出，有如下日志证明reload成功了。

```
# prometheus-server-configmap-reload容器输出:

level=info ts=2023-05-07T09:48:42.804818624Z caller=reloader.go:374 msg="Reload triggered" cfg_in= cfg_out= watched_dirs=/etc/config

# prometheus-server 容器输出: 
ts=2023-05-07T08:09:18.741Z caller=main.go:1246 level=info msg="Completed loading of configuration file" filename=/etc/config/prometheus.yml totalDuration=3.276064ms db_storage=1.652µs remote_storage=1.007µs web_handler=444ns query_engine=1.173µs scrape=662.222µs scrape_sd=739.882µs notify=69.047µs notify_sd=191.773µs rules=132.322µs tracing=2.817µs
```

### 访问prom查看Targets
查看是否识别target
```
# 通过如下做一下svc的映射： kubectl port-forward svc/[service-name] -n [namespace] [external-port]:[internal-port]

$ sudo kubectl port-forward svc/myprom-prometheus-server -n kubeskoop 80:80 --address='0.0.0.0'

# 通过本机的IP+80就可以访问
# 页面看到如下即可
Targets
kubeskoop (2/2 up)
Endpoint	                    State	 
http://192.168.0.6:9102/metrics	UP	
http://192.168.0.5:9102/metrics	UP	
```


## Grafana
> How to install and run Grafana on Kubernetes (K8S). It uses Kubernetes manifests for the setup:  [Grafana](https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/)

### Helm安装
```shell
$ sudo helm repo add grafana https://grafana.github.io/helm-charts
$ sudo helm repo update
$ sudo helm install mygrafana grafana/grafana --namespace kubeskoop
## helm install mygrafana grafana/grafana --namespace kubeskoop --debug
install.go:194: [debug] Original chart version: ""
install.go:211: [debug] CHART PATH: /root/.cache/helm/repository/grafana-6.56.1.tgz

client.go:133: [debug] creating 9 resource(s)
NAME: mygrafana
LAST DEPLOYED: Sun May  7 18:30:34 2023
NAMESPACE: kubeskoop
STATUS: deployed
REVISION: 1
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
admin:
  existingSecret: ""
  passwordKey: admin-password
  userKey: admin-user
adminUser: admin
affinity: {}
alerting: {}
autoscaling:
  behavior: {}
  enabled: false
  maxReplicas: 5
  minReplicas: 1
  targetCPU: "60"
  targetMemory: ""
containerSecurityContext:
  capabilities:
    drop:
    - ALL
  seccompProfile:
    type: RuntimeDefault
createConfigmap: true
dashboardProviders: {}
dashboards: {}
dashboardsConfigMaps: {}
datasources: {}
deploymentStrategy:
  type: RollingUpdate
downloadDashboards:
  env: {}
  envFromSecret: ""
  envValueFrom: {}
  resources: {}
  securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop:
      - ALL
    seccompProfile:
      type: RuntimeDefault
downloadDashboardsImage:
  pullPolicy: IfNotPresent
  repository: docker.io/curlimages/curl
  sha: ""
  tag: 7.85.0
enableKubeBackwardCompatibility: false
enableServiceLinks: true
env: {}
envFromConfigMaps: []
envFromSecret: ""
envFromSecrets: []
envRenderSecret: {}
envValueFrom: {}
extraConfigmapMounts: []
extraContainerVolumes: []
extraContainers: ""
extraEmptyDirMounts: []
extraExposePorts: []
extraInitContainers: []
extraLabels: {}
extraObjects: []
extraSecretMounts: []
extraVolumeMounts: []
global:
  imagePullSecrets: []
gossipPortName: gossip
grafana.ini:
  analytics:
    check_for_updates: true
  grafana_net:
    url: https://grafana.net
  log:
    mode: console
  paths:
    data: /var/lib/grafana/
    logs: /var/log/grafana
    plugins: /var/lib/grafana/plugins
    provisioning: /etc/grafana/provisioning
  server:
    domain: '{{ if (and .Values.ingress.enabled .Values.ingress.hosts) }}{{ .Values.ingress.hosts
      | first }}{{ else }}''''{{ end }}'
headlessService: false
hostAliases: []
image:
  pullPolicy: IfNotPresent
  pullSecrets: []
  repository: docker.io/grafana/grafana
  sha: ""
  tag: ""
imageRenderer:
  affinity: {}
  autoscaling:
    behavior: {}
    enabled: false
    maxReplicas: 5
    minReplicas: 1
    targetCPU: "60"
    targetMemory: ""
  containerSecurityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop:
      - ALL
    readOnlyRootFilesystem: true
    seccompProfile:
      type: RuntimeDefault
  deploymentStrategy: {}
  enabled: false
  env:
    HTTP_HOST: 0.0.0.0
  grafanaProtocol: http
  grafanaSubPath: ""
  hostAliases: []
  image:
    pullPolicy: Always
    repository: docker.io/grafana/grafana-image-renderer
    sha: ""
    tag: latest
  networkPolicy:
    extraIngressSelectors: []
    limitEgress: false
    limitIngress: true
  nodeSelector: {}
  podPortName: http
  priorityClassName: ""
  replicas: 1
  resources: {}
  revisionHistoryLimit: 10
  securityContext: {}
  service:
    appProtocol: ""
    enabled: true
    port: 8081
    portName: http
    targetPort: 8081
  serviceAccountName: ""
  serviceMonitor:
    enabled: false
    interval: 1m
    labels: {}
    path: /metrics
    relabelings: []
    scheme: http
    scrapeTimeout: 30s
    targetLabels: []
    tlsConfig: {}
  tolerations: []
ingress:
  annotations: {}
  enabled: false
  extraPaths: []
  hosts:
  - chart-example.local
  labels: {}
  path: /
  pathType: Prefix
  tls: []
initChownData:
  enabled: true
  image:
    pullPolicy: IfNotPresent
    repository: docker.io/library/busybox
    sha: ""
    tag: 1.31.1
  resources: {}
  securityContext:
    capabilities:
      add:
      - CHOWN
    runAsNonRoot: false
    runAsUser: 0
    seccompProfile:
      type: RuntimeDefault
ldap:
  config: ""
  enabled: false
  existingSecret: ""
lifecycleHooks: {}
livenessProbe:
  failureThreshold: 10
  httpGet:
    path: /api/health
    port: 3000
  initialDelaySeconds: 60
  timeoutSeconds: 30
namespaceOverride: ""
networkPolicy:
  allowExternal: true
  egress:
    enabled: false
    ports: []
  enabled: false
  explicitNamespacesSelector: {}
  ingress: true
nodeSelector: {}
notifiers: {}
persistence:
  accessModes:
  - ReadWriteOnce
  enabled: false
  extraPvcLabels: {}
  finalizers:
  - kubernetes.io/pvc-protection
  inMemory:
    enabled: false
  size: 10Gi
  type: pvc
plugins: []
podDisruptionBudget: {}
podPortName: grafana
rbac:
  create: true
  extraClusterRoleRules: []
  extraRoleRules: []
  namespaced: false
  pspEnabled: false
  pspUseAppArmor: false
readinessProbe:
  httpGet:
    path: /api/health
    port: 3000
replicas: 1
resources: {}
revisionHistoryLimit: 10
securityContext:
  fsGroup: 472
  runAsGroup: 472
  runAsNonRoot: true
  runAsUser: 472
service:
  annotations: {}
  appProtocol: ""
  enabled: true
  labels: {}
  port: 80
  portName: service
  targetPort: 3000
  type: ClusterIP
serviceAccount:
  autoMount: true
  create: true
  labels: {}
  name: null
  nameTest: null
serviceMonitor:
  enabled: false
  interval: 1m
  labels: {}
  path: /metrics
  relabelings: []
  scheme: http
  scrapeTimeout: 30s
  targetLabels: []
  tlsConfig: {}
sidecar:
  alerts:
    enabled: false
    env: {}
    label: grafana_alert
    labelValue: ""
    reloadURL: http://localhost:3000/api/admin/provisioning/alerting/reload
    resource: both
    script: null
    searchNamespace: null
    sizeLimit: {}
    skipReload: false
    watchMethod: WATCH
  dashboards:
    SCProvider: true
    defaultFolderName: null
    enabled: false
    env: {}
    extraMounts: []
    folder: /tmp/dashboards
    folderAnnotation: null
    label: grafana_dashboard
    labelValue: ""
    provider:
      allowUiUpdates: false
      disableDelete: false
      folder: ""
      foldersFromFilesStructure: false
      name: sidecarProvider
      orgid: 1
      type: file
    reloadURL: http://localhost:3000/api/admin/provisioning/dashboards/reload
    resource: both
    script: null
    searchNamespace: null
    sizeLimit: {}
    skipReload: false
    watchMethod: WATCH
  datasources:
    enabled: false
    env: {}
    initDatasources: false
    label: grafana_datasource
    labelValue: ""
    reloadURL: http://localhost:3000/api/admin/provisioning/datasources/reload
    resource: both
    script: null
    searchNamespace: null
    sizeLimit: {}
    skipReload: false
    watchMethod: WATCH
  enableUniqueFilenames: false
  image:
    repository: quay.io/kiwigrid/k8s-sidecar
    sha: ""
    tag: 1.22.0
  imagePullPolicy: IfNotPresent
  livenessProbe: {}
  notifiers:
    enabled: false
    env: {}
    initNotifiers: false
    label: grafana_notifier
    labelValue: ""
    reloadURL: http://localhost:3000/api/admin/provisioning/notifications/reload
    resource: both
    script: null
    searchNamespace: null
    sizeLimit: {}
    skipReload: false
    watchMethod: WATCH
  plugins:
    enabled: false
    env: {}
    initPlugins: false
    label: grafana_plugin
    labelValue: ""
    reloadURL: http://localhost:3000/api/admin/provisioning/plugins/reload
    resource: both
    script: null
    searchNamespace: null
    sizeLimit: {}
    skipReload: false
    watchMethod: WATCH
  readinessProbe: {}
  resources: {}
  securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop:
      - ALL
    seccompProfile:
      type: RuntimeDefault
smtp:
  existingSecret: ""
  passwordKey: password
  userKey: user
testFramework:
  enabled: true
  image: docker.io/bats/bats
  imagePullPolicy: IfNotPresent
  securityContext: {}
  tag: v1.4.1
tolerations: []
topologySpreadConstraints: []
useStatefulSet: false

HOOKS:
---
# Source: grafana/templates/tests/test-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
  name: mygrafana-test
  namespace: kubeskoop
  annotations:
    "helm.sh/hook": test-success
    "helm.sh/hook-delete-policy": "before-hook-creation,hook-succeeded"
---
# Source: grafana/templates/tests/test-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mygrafana-test
  namespace: kubeskoop
  annotations:
    "helm.sh/hook": test-success
    "helm.sh/hook-delete-policy": "before-hook-creation,hook-succeeded"
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
data:
  run.sh: |-
    @test "Test Health" {
      url="http://mygrafana/api/health"

      code=$(wget --server-response --spider --timeout 90 --tries 10 ${url} 2>&1 | awk '/^  HTTP/{print $2}')
      [ "$code" == "200" ]
    }
---
# Source: grafana/templates/tests/test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mygrafana-test
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
  annotations:
    "helm.sh/hook": test-success
    "helm.sh/hook-delete-policy": "before-hook-creation,hook-succeeded"
  namespace: kubeskoop
spec:
  serviceAccountName: mygrafana-test
  containers:
    - name: mygrafana-test
      image: "docker.io/bats/bats:v1.4.1"
      imagePullPolicy: "IfNotPresent"
      command: ["/opt/bats/bin/bats", "-t", "/tests/run.sh"]
      volumeMounts:
        - mountPath: /tests
          name: tests
          readOnly: true
  volumes:
    - name: tests
      configMap:
        name: mygrafana-test
  restartPolicy: Never
MANIFEST:
---
# Source: grafana/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
  name: mygrafana
  namespace: kubeskoop
---
# Source: grafana/templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mygrafana
  namespace: kubeskoop
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
type: Opaque
data:
  admin-user: "YWRtaW4="
  admin-password: "MVlnSkJUVXlyWWt2cHllNXYzT1FYQjk4d3NaQWY1SGNmR0tLVEp0WA=="
  ldap-toml: ""
---
# Source: grafana/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mygrafana
  namespace: kubeskoop
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
data:
  grafana.ini: |
    [analytics]
    check_for_updates = true
    [grafana_net]
    url = https://grafana.net
    [log]
    mode = console
    [paths]
    data = /var/lib/grafana/
    logs = /var/log/grafana
    plugins = /var/lib/grafana/plugins
    provisioning = /etc/grafana/provisioning
    [server]
    domain = ''
---
# Source: grafana/templates/clusterrole.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
  name: mygrafana-clusterrole
rules: []
---
# Source: grafana/templates/clusterrolebinding.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: mygrafana-clusterrolebinding
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
subjects:
  - kind: ServiceAccount
    name: mygrafana
    namespace: kubeskoop
roleRef:
  kind: ClusterRole
  name: mygrafana-clusterrole
  apiGroup: rbac.authorization.k8s.io
---
# Source: grafana/templates/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: mygrafana
  namespace: kubeskoop
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
rules: []
---
# Source: grafana/templates/rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: mygrafana
  namespace: kubeskoop
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: mygrafana
subjects:
- kind: ServiceAccount
  name: mygrafana
  namespace: kubeskoop
---
# Source: grafana/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mygrafana
  namespace: kubeskoop
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
spec:
  type: ClusterIP
  ports:
    - name: service
      port: 80
      protocol: TCP
      targetPort: 3000
  selector:
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
---
# Source: grafana/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mygrafana
  namespace: kubeskoop
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: mygrafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/name: grafana
      app.kubernetes.io/instance: mygrafana
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app.kubernetes.io/name: grafana
        app.kubernetes.io/instance: mygrafana
      annotations:
        checksum/config: 938a3b8ef4586b3acdd138be82579706b3a9add5932b72146fc706d5aed54e5c
        checksum/dashboards-json-config: 01ba4719c80b6fe911b091a7c05124b64eeece964e09c058ef8f9805daca546b
        checksum/sc-dashboard-provider-config: 01ba4719c80b6fe911b091a7c05124b64eeece964e09c058ef8f9805daca546b
        checksum/secret: 97fe1479d0bdebb904f71d7c7a2ecc2684b35db8f77c6086731bd4abd3301557
        kubectl.kubernetes.io/default-container: grafana
    spec:

      serviceAccountName: mygrafana
      automountServiceAccountToken: true
      securityContext:
        fsGroup: 472
        runAsGroup: 472
        runAsNonRoot: true
        runAsUser: 472
      enableServiceLinks: true
      containers:
        - name: grafana
          image: "docker.io/grafana/grafana:9.5.1"
          imagePullPolicy: IfNotPresent
          securityContext:
            capabilities:
              drop:
              - ALL
            seccompProfile:
              type: RuntimeDefault
          volumeMounts:
            - name: config
              mountPath: "/etc/grafana/grafana.ini"
              subPath: grafana.ini
            - name: storage
              mountPath: "/var/lib/grafana"
          ports:
            - name: grafana
              containerPort: 3000
              protocol: TCP
            - name: gossip-tcp
              containerPort: 9094
              protocol: TCP
            - name: gossip-udp
              containerPort: 9094
              protocol: UDP
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: GF_SECURITY_ADMIN_USER
              valueFrom:
                secretKeyRef:
                  name: mygrafana
                  key: admin-user
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mygrafana
                  key: admin-password
            - name: GF_PATHS_DATA
              value: /var/lib/grafana/
            - name: GF_PATHS_LOGS
              value: /var/log/grafana
            - name: GF_PATHS_PLUGINS
              value: /var/lib/grafana/plugins
            - name: GF_PATHS_PROVISIONING
              value: /etc/grafana/provisioning
          livenessProbe:
            failureThreshold: 10
            httpGet:
              path: /api/health
              port: 3000
            initialDelaySeconds: 60
            timeoutSeconds: 30
          readinessProbe:
            httpGet:
              path: /api/health
              port: 3000
      volumes:
        - name: config
          configMap:
            name: mygrafana
        - name: storage
          emptyDir: {}

NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace kubeskoop mygrafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo


2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   mygrafana.kubeskoop.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:
     export POD_NAME=$(kubectl get pods --namespace kubeskoop -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=mygrafana" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace kubeskoop port-forward $POD_NAME 3000

3. Login with the password from step 1 and the username: admin
#################################################################################
######   WARNING: Persistence is disabled!!! You will lose your data when   #####
######            the Grafana pod is terminated.                            #####
#################################################################################

```

### 获取grafana的svc映射访问

```bash
# 通过如下做一下svc的映射： kubectl port-forward svc/[service-name] -n [namespace] [external-port]:[internal-port]
$ sudo kubectl port-forward svc/mygrafana -n kubeskoop 80:80 --address='0.0.0.0'

获取初始密码
NOTES:
Get your 'admin' user password by running:

   kubectl get secret --namespace kubeskoop mygrafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

```

### 添加prom数据源
```
地址： 
http://myprom-prometheus-server.kubeskoop.svc
```

### 导入kubeskoop的dashboard

```
下载导入dashboard: 
https://raw.githubusercontent.com/alibaba/kubeskoop/main/deploy/resource/kubeskoop-exporter-dashboard.json
```