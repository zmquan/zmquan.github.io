---
title: Helm 安装prometheus
date: 2023-05-08 18:48:06
categories: 
- 技术文档
tags: 
- k8s
- prometheus
---
|软件|版本|
|:-|:-|
|prometheus|Chart Version:22.2.0 <br> App Version:v2.43.0|
|k8s|1.20.11|
|helm|3.11.1|

## 下载指定版本prometheus
```bash
$ sudo helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ sudo helm repo update
$ sudo helm fetch prometheus-community/prometheus --version 22.2.0
$ sudo tar xf prometheus-22.2.0.tgz
```
<!--more-->

<details>
  <summary>展开查看目录结构</summary>

```bash
$ tree prometheus
prometheus
├── Chart.lock
├── charts
│   ├── alertmanager
│   │   ├── Chart.yaml
│   │   ├── ci
│   │   │   └── config-reload-values.yaml
│   │   ├── README.md
│   │   ├── templates
│   │   │   ├── configmap.yaml
│   │   │   ├── _helpers.tpl
│   │   │   ├── ingress.yaml
│   │   │   ├── NOTES.txt
│   │   │   ├── pdb.yaml
│   │   │   ├── serviceaccount.yaml
│   │   │   ├── services.yaml
│   │   │   ├── statefulset.yaml
│   │   │   └── tests
│   │   │       └── test-connection.yaml
│   │   └── values.yaml
│   ├── kube-state-metrics
│   │   ├── Chart.yaml
│   │   ├── README.md
│   │   ├── templates
│   │   │   ├── clusterrolebinding.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── _helpers.tpl
│   │   │   ├── kubeconfig-secret.yaml
│   │   │   ├── NOTES.txt
│   │   │   ├── pdb.yaml
│   │   │   ├── podsecuritypolicy.yaml
│   │   │   ├── psp-clusterrolebinding.yaml
│   │   │   ├── psp-clusterrole.yaml
│   │   │   ├── rbac-configmap.yaml
│   │   │   ├── rolebinding.yaml
│   │   │   ├── role.yaml
│   │   │   ├── serviceaccount.yaml
│   │   │   ├── servicemonitor.yaml
│   │   │   ├── service.yaml
│   │   │   ├── stsdiscovery-rolebinding.yaml
│   │   │   ├── stsdiscovery-role.yaml
│   │   │   └── verticalpodautoscaler.yaml
│   │   └── values.yaml
│   ├── prometheus-node-exporter
│   │   ├── Chart.yaml
│   │   ├── ci
│   │   │   └── port-values.yaml
│   │   ├── README.md
│   │   ├── templates
│   │   │   ├── daemonset.yaml
│   │   │   ├── endpoints.yaml
│   │   │   ├── _helpers.tpl
│   │   │   ├── NOTES.txt
│   │   │   ├── psp-clusterrolebinding.yaml
│   │   │   ├── psp-clusterrole.yaml
│   │   │   ├── psp.yaml
│   │   │   ├── serviceaccount.yaml
│   │   │   ├── servicemonitor.yaml
│   │   │   ├── service.yaml
│   │   │   └── verticalpodautoscaler.yaml
│   │   └── values.yaml
│   └── prometheus-pushgateway
│       ├── Chart.yaml
│       ├── README.md
│       ├── templates
│       │   ├── deployment.yaml
│       │   ├── _helpers.tpl
│       │   ├── ingress.yaml
│       │   ├── networkpolicy.yaml
│       │   ├── NOTES.txt
│       │   ├── pdb.yaml
│       │   ├── pushgateway-pvc.yaml
│       │   ├── serviceaccount.yaml
│       │   ├── servicemonitor.yaml
│       │   ├── service.yaml
│       │   └── statefulset.yaml
│       └── values.yaml
├── Chart.yaml
├── README.md
├── templates
│   ├── clusterrolebinding.yaml
│   ├── clusterrole.yaml
│   ├── cm.yaml
│   ├── deploy.yaml
│   ├── extra-manifests.yaml
│   ├── headless-svc.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── network-policy.yaml
│   ├── NOTES.txt
│   ├── pdb.yaml
│   ├── psp.yaml
│   ├── pvc.yaml
│   ├── rolebinding.yaml
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   ├── sts.yaml
│   └── vpa.yaml
├── values.schema.json
└── values.yaml

13 directories, 86 files
```
</details>


## 查看values.yaml
```bash
$ cat  prometheus/values.yaml
```

<details>
  <summary>展开查看values.yaml</summary>

```yaml
rbac:
  create: true

podSecurityPolicy:
  enabled: false

imagePullSecrets: []
# - name: "image-pull-secret"

## Define serviceAccount names for components. Defaults to component's fully qualified name.
##
serviceAccounts:
  server:
    create: true
    name: ""
    annotations: {}

## Monitors ConfigMap changes and POSTs to a URL
## Ref: https://github.com/prometheus-operator/prometheus-operator/tree/main/cmd/prometheus-config-reloader
##
configmapReload:
  prometheus:
    ## If false, the configmap-reload container will not be deployed
    ##
    enabled: true

    ## configmap-reload container name
    ##
    name: configmap-reload

    ## configmap-reload container image
    ##
    image:
      repository: quay.io/prometheus-operator/prometheus-config-reloader
      tag: v0.63.0
      # When digest is set to a non-empty value, images will be pulled by digest (regardless of tag value).
      digest: ""
      pullPolicy: IfNotPresent

    # containerPort: 9533

    ## Additional configmap-reload container arguments
    ##
    extraArgs: {}
    ## Additional configmap-reload volume directories
    ##
    extraVolumeDirs: []


    ## Additional configmap-reload mounts
    ##
    extraConfigmapMounts: []
      # - name: prometheus-alerts
      #   mountPath: /etc/alerts.d
      #   subPath: ""
      #   configMap: prometheus-alerts
      #   readOnly: true

    ## Security context to be added to configmap-reload container
    containerSecurityContext: {}

    ## configmap-reload resource requests and limits
    ## Ref: http://kubernetes.io/docs/user-guide/compute-resources/
    ##
    resources: {}

server:
  ## Prometheus server container name
  ##
  name: server

  ## Use a ClusterRole (and ClusterRoleBinding)
  ## - If set to false - we define a RoleBinding in the defined namespaces ONLY
  ##
  ## NB: because we need a Role with nonResourceURL's ("/metrics") - you must get someone with Cluster-admin privileges to define this role for you, before running with this setting enabled.
  ##     This makes prometheus work - for users who do not have ClusterAdmin privs, but wants prometheus to operate on their own namespaces, instead of clusterwide.
  ##
  ## You MUST also set namespaces to the ones you have access to and want monitored by Prometheus.
  ##
  # useExistingClusterRoleName: nameofclusterrole

  ## namespaces to monitor (instead of monitoring all - clusterwide). Needed if you want to run without Cluster-admin privileges.
  # namespaces:
  #   - yournamespace

  # sidecarContainers - add more containers to prometheus server
  # Key/Value where Key is the sidecar `- name: <Key>`
  # Example:
  #   sidecarContainers:
  #      webserver:
  #        image: nginx
  sidecarContainers: {}

  # sidecarTemplateValues - context to be used in template for sidecarContainers
  # Example:
  #   sidecarTemplateValues: *your-custom-globals
  #   sidecarContainers:
  #     webserver: |-
  #       {{ include "webserver-container-template" . }}
  # Template for `webserver-container-template` might looks like this:
  #   image: "{{ .Values.server.sidecarTemplateValues.repository }}:{{ .Values.server.sidecarTemplateValues.tag }}"
  #   ...
  #
  sidecarTemplateValues: {}

  ## Prometheus server container image
  ##
  image:
    repository: quay.io/prometheus/prometheus
    # if not set appVersion field from Chart.yaml is used
    tag: ""
    # When digest is set to a non-empty value, images will be pulled by digest (regardless of tag value).
    digest: ""
    pullPolicy: IfNotPresent

  ## Prometheus server command
  ##
  command: []

  ## prometheus server priorityClassName
  ##
  priorityClassName: ""

  ## EnableServiceLinks indicates whether information about services should be injected
  ## into pod's environment variables, matching the syntax of Docker links.
  ## WARNING: the field is unsupported and will be skipped in K8s prior to v1.13.0.
  ##
  enableServiceLinks: true

  ## The URL prefix at which the container can be accessed. Useful in the case the '-web.external-url' includes a slug
  ## so that the various internal URLs are still able to access as they are in the default case.
  ## (Optional)
  prefixURL: ""

  ## External URL which can access prometheus
  ## Maybe same with Ingress host name
  baseURL: ""

  ## Additional server container environment variables
  ##
  ## You specify this manually like you would a raw deployment manifest.
  ## This means you can bind in environment variables from secrets.
  ##
  ## e.g. static environment variable:
  ##  - name: DEMO_GREETING
  ##    value: "Hello from the environment"
  ##
  ## e.g. secret environment variable:
  ## - name: USERNAME
  ##   valueFrom:
  ##     secretKeyRef:
  ##       name: mysecret
  ##       key: username
  env: []

  # List of flags to override default parameters, e.g:
  # - --enable-feature=agent
  # - --storage.agent.retention.max-time=30m
  defaultFlagsOverride: []

  extraFlags:
    - web.enable-lifecycle
    ## web.enable-admin-api flag controls access to the administrative HTTP API which includes functionality such as
    ## deleting time series. This is disabled by default.
    # - web.enable-admin-api
    ##
    ## storage.tsdb.no-lockfile flag controls BD locking
    # - storage.tsdb.no-lockfile
    ##
    ## storage.tsdb.wal-compression flag enables compression of the write-ahead log (WAL)
    # - storage.tsdb.wal-compression

  ## Path to a configuration file on prometheus server container FS
  configPath: /etc/config/prometheus.yml

  ### The data directory used by prometheus to set --storage.tsdb.path
  ### When empty server.persistentVolume.mountPath is used instead
  storagePath: ""

  global:
    ## How frequently to scrape targets by default
    ##
    scrape_interval: 1m
    ## How long until a scrape request times out
    ##
    scrape_timeout: 10s
    ## How frequently to evaluate rules
    ##
    evaluation_interval: 1m
  ## https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write
  ##
  remoteWrite: []
  ## https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_read
  ##
  remoteRead: []

  ## https://prometheus.io/docs/prometheus/latest/configuration/configuration/#tsdb
  ##
  tsdb: {}
    # out_of_order_time_window: 0s

  ## https://prometheus.io/docs/prometheus/latest/configuration/configuration/#exemplars
  ## Must be enabled via --enable-feature=exemplar-storage
  ##
  exemplars: {}
    # max_exemplars: 100000

  ## Custom HTTP headers for Liveness/Readiness/Startup Probe
  ##
  ## Useful for providing HTTP Basic Auth to healthchecks
  probeHeaders: []
    # - name: "Authorization"
    #   value: "Bearer ABCDEabcde12345"

  ## Additional Prometheus server container arguments
  ##
  extraArgs: {}

  ## Additional InitContainers to initialize the pod
  ##
  extraInitContainers: []

  ## Additional Prometheus server Volume mounts
  ##
  extraVolumeMounts: []

  ## Additional Prometheus server Volumes
  ##
  extraVolumes: []

  ## Additional Prometheus server hostPath mounts
  ##
  extraHostPathMounts: []
    # - name: certs-dir
    #   mountPath: /etc/kubernetes/certs
    #   subPath: ""
    #   hostPath: /etc/kubernetes/certs
    #   readOnly: true

  extraConfigmapMounts: []
    # - name: certs-configmap
    #   mountPath: /prometheus
    #   subPath: ""
    #   configMap: certs-configmap
    #   readOnly: true

  ## Additional Prometheus server Secret mounts
  # Defines additional mounts with secrets. Secrets must be manually created in the namespace.
  extraSecretMounts: []
    # - name: secret-files
    #   mountPath: /etc/secrets
    #   subPath: ""
    #   secretName: prom-secret-files
    #   readOnly: true

  ## ConfigMap override where fullname is {{.Release.Name}}-{{.Values.server.configMapOverrideName}}
  ## Defining configMapOverrideName will cause templates/server-configmap.yaml
  ## to NOT generate a ConfigMap resource
  ##
  configMapOverrideName: ""

  ## Extra labels for Prometheus server ConfigMap (ConfigMap that holds serverFiles)
  extraConfigmapLabels: {}

  ingress:
    ## If true, Prometheus server Ingress will be created
    ##
    enabled: false

    # For Kubernetes >= 1.18 you should specify the ingress-controller via the field ingressClassName
    # See https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/#specifying-the-class-of-an-ingress
    # ingressClassName: nginx

    ## Prometheus server Ingress annotations
    ##
    annotations: {}
    #   kubernetes.io/ingress.class: nginx
    #   kubernetes.io/tls-acme: 'true'

    ## Prometheus server Ingress additional labels
    ##
    extraLabels: {}

    ## Prometheus server Ingress hostnames with optional path
    ## Must be provided if Ingress is enabled
    ##
    hosts: []
    #   - prometheus.domain.com
    #   - domain.com/prometheus

    path: /

    # pathType is only for k8s >= 1.18
    pathType: Prefix

    ## Extra paths to prepend to every host configuration. This is useful when working with annotation based services.
    extraPaths: []
    # - path: /*
    #   backend:
    #     serviceName: ssl-redirect
    #     servicePort: use-annotation

    ## Prometheus server Ingress TLS configuration
    ## Secrets must be manually created in the namespace
    ##
    tls: []
    #   - secretName: prometheus-server-tls
    #     hosts:
    #       - prometheus.domain.com

  ## Server Deployment Strategy type
  strategy:
    type: Recreate

  ## hostAliases allows adding entries to /etc/hosts inside the containers
  hostAliases: []
  #   - ip: "127.0.0.1"
  #     hostnames:
  #       - "example.com"

  ## Node tolerations for server scheduling to nodes with taints
  ## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
  ##
  tolerations: []
    # - key: "key"
    #   operator: "Equal|Exists"
    #   value: "value"
    #   effect: "NoSchedule|PreferNoSchedule|NoExecute(1.6 only)"

  ## Node labels for Prometheus server pod assignment
  ## Ref: https://kubernetes.io/docs/user-guide/node-selection/
  ##
  nodeSelector: {}

  ## Pod affinity
  ##
  affinity: {}

  ## Pod topology spread constraints
  ## ref. https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/
  topologySpreadConstraints: []

  ## PodDisruptionBudget settings
  ## ref: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
  ##
  podDisruptionBudget:
    enabled: false
    maxUnavailable: 1

  ## Use an alternate scheduler, e.g. "stork".
  ## ref: https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/
  ##
  # schedulerName:

  persistentVolume:
    ## If true, Prometheus server will create/use a Persistent Volume Claim
    ## If false, use emptyDir
    ##
    enabled: true

    ## If set it will override the name of the created persistent volume claim
    ## generated by the stateful set.
    ##
    statefulSetNameOverride:

    ## Prometheus server data Persistent Volume access modes
    ## Must match those of existing PV or dynamic provisioner
    ## Ref: http://kubernetes.io/docs/user-guide/persistent-volumes/
    ##
    accessModes:
      - ReadWriteMany

    ## Prometheus server data Persistent Volume labels
    ##
    labels: {}

    ## Prometheus server data Persistent Volume annotations
    ##
    annotations: {}

    ## Prometheus server data Persistent Volume existing claim name
    ## Requires server.persistentVolume.enabled: true
    ## If defined, PVC must be created manually before volume will be bound
    existingClaim: ""

    ## Prometheus server data Persistent Volume mount root path
    ##
    mountPath: /data

    ## Prometheus server data Persistent Volume size
    ##
    size: 8Gi

    ## Prometheus server data Persistent Volume Storage Class
    ## If defined, storageClassName: <storageClass>
    ## If set to "-", storageClassName: "", which disables dynamic provisioning
    ## If undefined (the default) or set to null, no storageClassName spec is
    ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
    ##   GKE, AWS & OpenStack)
    ##
    # storageClass: "-"

    ## Prometheus server data Persistent Volume Binding Mode
    ## If defined, volumeBindingMode: <volumeBindingMode>
    ## If undefined (the default) or set to null, no volumeBindingMode spec is
    ##   set, choosing the default mode.
    ##
    # volumeBindingMode: ""

    ## Subdirectory of Prometheus server data Persistent Volume to mount
    ## Useful if the volume's root directory is not empty
    ##
    subPath: ""

    ## Persistent Volume Claim Selector
    ## Useful if Persistent Volumes have been provisioned in advance
    ## Ref: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#selector
    ##
    # selector:
    #  matchLabels:
    #    release: "stable"
    #  matchExpressions:
    #    - { key: environment, operator: In, values: [ dev ] }

    ## Persistent Volume Name
    ## Useful if Persistent Volumes have been provisioned in advance and you want to use a specific one
    ##
    # volumeName: ""

  emptyDir:
    ## Prometheus server emptyDir volume size limit
    ##
    sizeLimit: ""

  ## Annotations to be added to Prometheus server pods
  ##
  podAnnotations: {}
    # iam.amazonaws.com/role: prometheus

  ## Labels to be added to Prometheus server pods
  ##
  podLabels: {}

  ## Prometheus AlertManager configuration
  ##
  alertmanagers: []

  ## Specify if a Pod Security Policy for node-exporter must be created
  ## Ref: https://kubernetes.io/docs/concepts/policy/pod-security-policy/
  ##
  podSecurityPolicy:
    annotations: {}
      ## Specify pod annotations
      ## Ref: https://kubernetes.io/docs/concepts/policy/pod-security-policy/#apparmor
      ## Ref: https://kubernetes.io/docs/concepts/policy/pod-security-policy/#seccomp
      ## Ref: https://kubernetes.io/docs/concepts/policy/pod-security-policy/#sysctl
      ##
      # seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
      # seccomp.security.alpha.kubernetes.io/defaultProfileName: 'docker/default'
      # apparmor.security.beta.kubernetes.io/defaultProfileName: 'runtime/default'

  ## Use a StatefulSet if replicaCount needs to be greater than 1 (see below)
  ##
  replicaCount: 1

  ## Annotations to be added to deployment
  ##
  deploymentAnnotations: {}

  statefulSet:
    ## If true, use a statefulset instead of a deployment for pod management.
    ## This allows to scale replicas to more than 1 pod
    ##
    enabled: false

    annotations: {}
    labels: {}
    podManagementPolicy: OrderedReady

    ## Alertmanager headless service to use for the statefulset
    ##
    headless:
      annotations: {}
      labels: {}
      servicePort: 80
      ## Enable gRPC port on service to allow auto discovery with thanos-querier
      gRPC:
        enabled: false
        servicePort: 10901
        # nodePort: 10901

  ## Prometheus server readiness and liveness probe initial delay and timeout
  ## Ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
  ##
  tcpSocketProbeEnabled: false
  probeScheme: HTTP
  readinessProbeInitialDelay: 30
  readinessProbePeriodSeconds: 5
  readinessProbeTimeout: 4
  readinessProbeFailureThreshold: 3
  readinessProbeSuccessThreshold: 1
  livenessProbeInitialDelay: 30
  livenessProbePeriodSeconds: 15
  livenessProbeTimeout: 10
  livenessProbeFailureThreshold: 3
  livenessProbeSuccessThreshold: 1
  startupProbe:
    enabled: false
    periodSeconds: 5
    failureThreshold: 30
    timeoutSeconds: 10

  ## Prometheus server resource requests and limits
  ## Ref: http://kubernetes.io/docs/user-guide/compute-resources/
  ##
  resources: {}
    # limits:
    #   cpu: 500m
    #   memory: 512Mi
    # requests:
    #   cpu: 500m
    #   memory: 512Mi

  # Required for use in managed kubernetes clusters (such as AWS EKS) with custom CNI (such as calico),
  # because control-plane managed by AWS cannot communicate with pods' IP CIDR and admission webhooks are not working
  ##
  hostNetwork: false

  # When hostNetwork is enabled, this will set to ClusterFirstWithHostNet automatically
  dnsPolicy: ClusterFirst

  # Use hostPort
  # hostPort: 9090

  ## Vertical Pod Autoscaler config
  ## Ref: https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler
  verticalAutoscaler:
    ## If true a VPA object will be created for the controller (either StatefulSet or Deployemnt, based on above configs)
    enabled: false
    # updateMode: "Auto"
    # containerPolicies:
    # - containerName: 'prometheus-server'

  # Custom DNS configuration to be added to prometheus server pods
  dnsConfig: {}
    # nameservers:
    #   - 1.2.3.4
    # searches:
    #   - ns1.svc.cluster-domain.example
    #   - my.dns.search.suffix
    # options:
    #   - name: ndots
    #     value: "2"
  #   - name: edns0

  ## Security context to be added to server pods
  ##
  securityContext:
    runAsUser: 65534
    runAsNonRoot: true
    runAsGroup: 65534
    fsGroup: 65534

  ## Security context to be added to server container
  ##
  containerSecurityContext: {}

  service:
    ## If false, no Service will be created for the Prometheus server
    ##
    enabled: true

    annotations: {}
    labels: {}
    clusterIP: ""

    ## List of IP addresses at which the Prometheus server service is available
    ## Ref: https://kubernetes.io/docs/concepts/services-networking/service/#external-ips
    ##
    externalIPs: []

    loadBalancerIP: ""
    loadBalancerSourceRanges: []
    servicePort: 80
    sessionAffinity: None
    type: ClusterIP

    ## Enable gRPC port on service to allow auto discovery with thanos-querier
    gRPC:
      enabled: false
      servicePort: 10901
      # nodePort: 10901

    ## If using a statefulSet (statefulSet.enabled=true), configure the
    ## service to connect to a specific replica to have a consistent view
    ## of the data.
    statefulsetReplica:
      enabled: false
      replica: 0

  ## Prometheus server pod termination grace period
  ##
  terminationGracePeriodSeconds: 300

  ## Prometheus data retention period (default if not specified is 15 days)
  ##
  retention: "15d"

## Prometheus server ConfigMap entries for rule files (allow prometheus labels interpolation)
ruleFiles: {}

## Prometheus server ConfigMap entries
##
serverFiles:
  ## Alerts configuration
  ## Ref: https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
  alerting_rules.yml: {}
  # groups:
  #   - name: Instances
  #     rules:
  #       - alert: InstanceDown
  #         expr: up == 0
  #         for: 5m
  #         labels:
  #           severity: page
  #         annotations:
  #           description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.'
  #           summary: 'Instance {{ $labels.instance }} down'
  ## DEPRECATED DEFAULT VALUE, unless explicitly naming your files, please use alerting_rules.yml
  alerts: {}

  ## Records configuration
  ## Ref: https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/
  recording_rules.yml: {}
  ## DEPRECATED DEFAULT VALUE, unless explicitly naming your files, please use recording_rules.yml
  rules: {}

  prometheus.yml:
    rule_files:
      - /etc/config/recording_rules.yml
      - /etc/config/alerting_rules.yml
    ## Below two files are DEPRECATED will be removed from this default values file
      - /etc/config/rules
      - /etc/config/alerts

    scrape_configs:
      - job_name: prometheus
        static_configs:
          - targets:
            - localhost:9090

      # A scrape configuration for running Prometheus on a Kubernetes cluster.
      # This uses separate scrape configs for cluster components (i.e. API server, node)
      # and services to allow each to use different authentication configs.
      #
      # Kubernetes labels will be added as Prometheus labels on metrics via the
      # `labelmap` relabeling action.

      # Scrape config for API servers.
      #
      # Kubernetes exposes API servers as endpoints to the default/kubernetes
      # service so this uses `endpoints` role and uses relabelling to only keep
      # the endpoints associated with the default/kubernetes service using the
      # default named port `https`. This works for single API server deployments as
      # well as HA API server deployments.
      - job_name: 'kubernetes-apiservers'

        kubernetes_sd_configs:
          - role: endpoints

        # Default to scraping over https. If required, just disable this or change to
        # `http`.
        scheme: https

        # This TLS & bearer token file config is used to connect to the actual scrape
        # endpoints for cluster components. This is separate to discovery auth
        # configuration because discovery & scraping are two separate concerns in
        # Prometheus. The discovery auth config is automatic if Prometheus runs inside
        # the cluster. Otherwise, more config options have to be provided within the
        # <kubernetes_sd_config>.
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          # If your node certificates are self-signed or use a different CA to the
          # master CA, then disable certificate verification below. Note that
          # certificate verification is an integral part of a secure infrastructure
          # so this should only be disabled in a controlled environment. You can
          # disable certificate verification by uncommenting the line below.
          #
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        # Keep only the default/kubernetes service endpoints for the https port. This
        # will add targets for each API server which Kubernetes adds an endpoint to
        # the default/kubernetes service.
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'

        # Default to scraping over https. If required, just disable this or change to
        # `http`.
        scheme: https

        # This TLS & bearer token file config is used to connect to the actual scrape
        # endpoints for cluster components. This is separate to discovery auth
        # configuration because discovery & scraping are two separate concerns in
        # Prometheus. The discovery auth config is automatic if Prometheus runs inside
        # the cluster. Otherwise, more config options have to be provided within the
        # <kubernetes_sd_config>.
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          # If your node certificates are self-signed or use a different CA to the
          # master CA, then disable certificate verification below. Note that
          # certificate verification is an integral part of a secure infrastructure
          # so this should only be disabled in a controlled environment. You can
          # disable certificate verification by uncommenting the line below.
          #
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
          - role: node

        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/$1/proxy/metrics


      - job_name: 'kubernetes-nodes-cadvisor'

        # Default to scraping over https. If required, just disable this or change to
        # `http`.
        scheme: https

        # This TLS & bearer token file config is used to connect to the actual scrape
        # endpoints for cluster components. This is separate to discovery auth
        # configuration because discovery & scraping are two separate concerns in
        # Prometheus. The discovery auth config is automatic if Prometheus runs inside
        # the cluster. Otherwise, more config options have to be provided within the
        # <kubernetes_sd_config>.
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          # If your node certificates are self-signed or use a different CA to the
          # master CA, then disable certificate verification below. Note that
          # certificate verification is an integral part of a secure infrastructure
          # so this should only be disabled in a controlled environment. You can
          # disable certificate verification by uncommenting the line below.
          #
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
          - role: node

        # This configuration will work only on kubelet 1.7.3+
        # As the scrape endpoints for cAdvisor have changed
        # if you are using older version you need to change the replacement to
        # replacement: /api/v1/nodes/$1:4194/proxy/metrics
        # more info here https://github.com/coreos/prometheus-operator/issues/633
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor

        # Metric relabel configs to apply to samples before ingestion.
        # [Metric Relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#metric_relabel_configs)
        # metric_relabel_configs:
        # - action: labeldrop
        #   regex: (kubernetes_io_hostname|failure_domain_beta_kubernetes_io_region|beta_kubernetes_io_os|beta_kubernetes_io_arch|beta_kubernetes_io_instance_type|failure_domain_beta_kubernetes_io_zone)

      # Scrape config for service endpoints.
      #
      # The relabeling allows the actual service scrape endpoint to be configured
      # via the following annotations:
      #
      # * `prometheus.io/scrape`: Only scrape services that have a value of
      # `true`, except if `prometheus.io/scrape-slow` is set to `true` as well.
      # * `prometheus.io/scheme`: If the metrics endpoint is secured then you will need
      # to set this to `https` & most likely set the `tls_config` of the scrape config.
      # * `prometheus.io/path`: If the metrics path is not `/metrics` override this.
      # * `prometheus.io/port`: If the metrics are exposed on a different port to the
      # service then set this appropriately.
      # * `prometheus.io/param_<parameter>`: If the metrics endpoint uses parameters
      # then you can set any parameter
      - job_name: 'kubernetes-service-endpoints'
        honor_labels: true

        kubernetes_sd_configs:
          - role: endpoints

        relabel_configs:
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape_slow]
            action: drop
            regex: true
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
            action: replace
            target_label: __scheme__
            regex: (https?)
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
            action: replace
            target_label: __address__
            regex: (.+?)(?::\d+)?;(\d+)
            replacement: $1:$2
          - action: labelmap
            regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
            replacement: __param_$1
          - action: labelmap
            regex: __meta_kubernetes_service_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: namespace
          - source_labels: [__meta_kubernetes_service_name]
            action: replace
            target_label: service
          - source_labels: [__meta_kubernetes_pod_node_name]
            action: replace
            target_label: node

      # Scrape config for slow service endpoints; same as above, but with a larger
      # timeout and a larger interval
      #
      # The relabeling allows the actual service scrape endpoint to be configured
      # via the following annotations:
      #
      # * `prometheus.io/scrape-slow`: Only scrape services that have a value of `true`
      # * `prometheus.io/scheme`: If the metrics endpoint is secured then you will need
      # to set this to `https` & most likely set the `tls_config` of the scrape config.
      # * `prometheus.io/path`: If the metrics path is not `/metrics` override this.
      # * `prometheus.io/port`: If the metrics are exposed on a different port to the
      # service then set this appropriately.
      # * `prometheus.io/param_<parameter>`: If the metrics endpoint uses parameters
      # then you can set any parameter
      - job_name: 'kubernetes-service-endpoints-slow'
        honor_labels: true

        scrape_interval: 5m
        scrape_timeout: 30s

        kubernetes_sd_configs:
          - role: endpoints

        relabel_configs:
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape_slow]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
            action: replace
            target_label: __scheme__
            regex: (https?)
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
            action: replace
            target_label: __address__
            regex: (.+?)(?::\d+)?;(\d+)
            replacement: $1:$2
          - action: labelmap
            regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
            replacement: __param_$1
          - action: labelmap
            regex: __meta_kubernetes_service_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: namespace
          - source_labels: [__meta_kubernetes_service_name]
            action: replace
            target_label: service
          - source_labels: [__meta_kubernetes_pod_node_name]
            action: replace
            target_label: node

      - job_name: 'prometheus-pushgateway'
        honor_labels: true

        kubernetes_sd_configs:
          - role: service

        relabel_configs:
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
            action: keep
            regex: pushgateway

      # Example scrape config for probing services via the Blackbox Exporter.
      #
      # The relabeling allows the actual service scrape endpoint to be configured
      # via the following annotations:
      #
      # * `prometheus.io/probe`: Only probe services that have a value of `true`
      - job_name: 'kubernetes-services'
        honor_labels: true

        metrics_path: /probe
        params:
          module: [http_2xx]

        kubernetes_sd_configs:
          - role: service

        relabel_configs:
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
            action: keep
            regex: true
          - source_labels: [__address__]
            target_label: __param_target
          - target_label: __address__
            replacement: blackbox
          - source_labels: [__param_target]
            target_label: instance
          - action: labelmap
            regex: __meta_kubernetes_service_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
          - source_labels: [__meta_kubernetes_service_name]
            target_label: service

      # Example scrape config for pods
      #
      # The relabeling allows the actual pod scrape endpoint to be configured via the
      # following annotations:
      #
      # * `prometheus.io/scrape`: Only scrape pods that have a value of `true`,
      # except if `prometheus.io/scrape-slow` is set to `true` as well.
      # * `prometheus.io/scheme`: If the metrics endpoint is secured then you will need
      # to set this to `https` & most likely set the `tls_config` of the scrape config.
      # * `prometheus.io/path`: If the metrics path is not `/metrics` override this.
      # * `prometheus.io/port`: Scrape the pod on the indicated port instead of the default of `9102`.
      - job_name: 'kubernetes-pods'
        honor_labels: true

        kubernetes_sd_configs:
          - role: pod

        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape_slow]
            action: drop
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
            action: replace
            regex: (https?)
            target_label: __scheme__
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_ip]
            action: replace
            regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
            replacement: '[$2]:$1'
            target_label: __address__
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_ip]
            action: replace
            regex: (\d+);((([0-9]+?)(\.|$)){4})
            replacement: $2:$1
            target_label: __address__
          - action: labelmap
            regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
            replacement: __param_$1
          - action: labelmap
            regex: __meta_kubernetes_pod_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: pod
          - source_labels: [__meta_kubernetes_pod_phase]
            regex: Pending|Succeeded|Failed|Completed
            action: drop

      # Example Scrape config for pods which should be scraped slower. An useful example
      # would be stackriver-exporter which queries an API on every scrape of the pod
      #
      # The relabeling allows the actual pod scrape endpoint to be configured via the
      # following annotations:
      #
      # * `prometheus.io/scrape-slow`: Only scrape pods that have a value of `true`
      # * `prometheus.io/scheme`: If the metrics endpoint is secured then you will need
      # to set this to `https` & most likely set the `tls_config` of the scrape config.
      # * `prometheus.io/path`: If the metrics path is not `/metrics` override this.
      # * `prometheus.io/port`: Scrape the pod on the indicated port instead of the default of `9102`.
      - job_name: 'kubernetes-pods-slow'
        honor_labels: true

        scrape_interval: 5m
        scrape_timeout: 30s

        kubernetes_sd_configs:
          - role: pod

        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape_slow]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
            action: replace
            regex: (https?)
            target_label: __scheme__
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_ip]
            action: replace
            regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
            replacement: '[$2]:$1'
            target_label: __address__
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_ip]
            action: replace
            regex: (\d+);((([0-9]+?)(\.|$)){4})
            replacement: $2:$1
            target_label: __address__
          - action: labelmap
            regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
            replacement: __param_$1
          - action: labelmap
            regex: __meta_kubernetes_pod_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: pod
          - source_labels: [__meta_kubernetes_pod_phase]
            regex: Pending|Succeeded|Failed|Completed
            action: drop

# adds additional scrape configs to prometheus.yml
# must be a string so you have to add a | after extraScrapeConfigs:
# example adds prometheus-blackbox-exporter scrape config
extraScrapeConfigs: ""
  # - job_name: 'prometheus-blackbox-exporter'
  #   metrics_path: /probe
  #   params:
  #     module: [http_2xx]
  #   static_configs:
  #     - targets:
  #       - https://example.com
  #   relabel_configs:
  #     - source_labels: [__address__]
  #       target_label: __param_target
  #     - source_labels: [__param_target]
  #       target_label: instance
  #     - target_label: __address__
  #       replacement: prometheus-blackbox-exporter:9115

# Adds option to add alert_relabel_configs to avoid duplicate alerts in alertmanager
# useful in H/A prometheus with different external labels but the same alerts
alertRelabelConfigs: {}
  # alert_relabel_configs:
  # - source_labels: [dc]
  #   regex: (.+)\d+
  #   target_label: dc

networkPolicy:
  ## Enable creation of NetworkPolicy resources.
  ##
  enabled: false

# Force namespace of namespaced resources
forceNamespace: ""

# Extra manifests to deploy as an array
extraManifests: []
  # - apiVersion: v1
  #   kind: ConfigMap
  #   metadata:
  #   labels:
  #     name: prometheus-extra
  #   data:
  #     extra-data: "value"

# Configuration of subcharts defined in Chart.yaml

## alertmanager sub-chart configurable values
## Please see https://github.com/prometheus-community/helm-charts/tree/main/charts/alertmanager
##
alertmanager:
  ## If false, alertmanager will not be installed
  ##
  enabled: true

  persistence:
    size: 2Gi

  podSecurityContext:
    runAsUser: 65534
    runAsNonRoot: true
    runAsGroup: 65534
    fsGroup: 65534

## kube-state-metrics sub-chart configurable values
## Please see https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-state-metrics
##
kube-state-metrics:
  ## If false, kube-state-metrics sub-chart will not be installed
  ##
  enabled: true

## promtheus-node-exporter sub-chart configurable values
## Please see https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-node-exporter
##
prometheus-node-exporter:
  ## If false, node-exporter will not be installed
  ##
  enabled: true

  rbac:
    pspEnabled: false

  containerSecurityContext:
    allowPrivilegeEscalation: false

## pprometheus-pushgateway sub-chart configurable values
## Please see https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-pushgateway
##
prometheus-pushgateway:
  ## If false, pushgateway will not be installed
  ##
  enabled: false

  # Optional service annotations
  serviceAnnotations:
    prometheus.io/probe: pushgateway
```
</details>

## 设置server的pvc

```bash
$ sudo vim prometheus/values.yaml
```

```yaml
server:
  ## Prometheus server container name
  ##
  name: server
...
  persistentVolume:
    ## If true, Prometheus server will create/use a Persistent Volume Claim
    ## If false, use emptyDir
    ##
    enabled: true

    ## If set it will override the name of the created persistent volume claim
    ## generated by the stateful set.
    ##
    statefulSetNameOverride:

    ## Prometheus server data Persistent Volume access modes
    ## Must match those of existing PV or dynamic provisioner
    ## Ref: http://kubernetes.io/docs/user-guide/persistent-volumes/
    ##
    accessModes:
      - ReadWriteMany

    ## Prometheus server data Persistent Volume labels
    ##
    labels: {}

    ## Prometheus server data Persistent Volume annotations
    ##
    annotations: {}

    ## Prometheus server data Persistent Volume existing claim name
    ## Requires server.persistentVolume.enabled: true
    ## If defined, PVC must be created manually before volume will be bound
    existingClaim: ""   # 1. 如果有已存在的pvc，需要填上pvc的名称

    ## Prometheus server data Persistent Volume mount root path
    ##
    mountPath: /data

    ## Prometheus server data Persistent Volume size
    ##
    size: 8Gi

    ## Prometheus server data Persistent Volume Storage Class
    ## If defined, storageClassName: <storageClass>
    ## If set to "-", storageClassName: "", which disables dynamic provisioning
    ## If undefined (the default) or set to null, no storageClassName spec is
    ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
    ##   GKE, AWS & OpenStack)
    ##
    # storageClass: "-"     # 2. 如果没有默认存储类， 那么设置自己的存储类
    storageClass: "nas-sc" 
    ## Prometheus server data Persistent Volume Binding Mode
    ## If defined, volumeBindingMode: <volumeBindingMode>
    ## If undefined (the default) or set to null, no volumeBindingMode spec is
    ##   set, choosing the default mode.
    ##
    # volumeBindingMode: ""

    ## Subdirectory of Prometheus server data Persistent Volume to mount
    ## Useful if the volume's root directory is not empty
    ##
    subPath: ""

    ## Persistent Volume Claim Selector
    ## Useful if Persistent Volumes have been provisioned in advance
    ## Ref: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#selector
    ##
    # selector:
    #  matchLabels:
    #    release: "stable"
    #  matchExpressions:
    #    - { key: environment, operator: In, values: [ dev ] }

    ## Persistent Volume Name
    ## Useful if Persistent Volumes have been provisioned in advance and you want to use a specific one
    ##
    # volumeName: ""

  emptyDir:
    ## Prometheus server emptyDir volume size limit
    ##
    sizeLimit: ""
```
## 去掉pushgateway和alertmanager

```bash
$ sudo vim prometheus/values.yaml
```

```yaml
alertmanager:
  ## If false, alertmanager will not be installed
  ##
  enabled: false  # 1. 改成false

  persistence:
    size: 2Gi

  podSecurityContext:
    runAsUser: 65534
    runAsNonRoot: true
    runAsGroup: 65534
    fsGroup: 65534
...

prometheus-pushgateway:
  ## If false, pushgateway will not be installed
  ##
  enabled: false # 2. 改成false

  # Optional service annotations
  serviceAnnotations:
    prometheus.io/probe: pushgateway

```

## 部署prometheus

```bash
$ sudo helm install --create-namespace --namespace monitoring  prometheus ./prometheus --debug
install.go:194: [debug] Original chart version: ""
install.go:211: [debug] CHART PATH: /mnt/prometheus

client.go:133: [debug] creating 1 resource(s)
client.go:133: [debug] creating 15 resource(s)
NAME: prometheus
LAST DEPLOYED: Mon May  8 19:06:26 2023
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}
```

<details>
  <summary>展开查看debug信息</summary>

```yaml
COMPUTED VALUES:
alertRelabelConfigs: {}
alertmanager:
  additionalPeers: []
  affinity: {}
  automountServiceAccountToken: true
  command: []
  config:
    global: {}
    receivers:
    - name: default-receiver
    route:
      group_interval: 5m
      group_wait: 10s
      receiver: default-receiver
      repeat_interval: 3h
    templates:
    - /etc/alertmanager/*.tmpl
  configAnnotations: {}
  configmapReload:
    enabled: false
    image:
      pullPolicy: IfNotPresent
      repository: jimmidyson/configmap-reload
      tag: v0.8.0
    name: configmap-reload
    resources: {}
  dnsConfig: {}
  enabled: false
  extraArgs: {}
  extraEnv: []
  extraInitContainers: []
  extraSecretMounts: []
  extraVolumeMounts: []
  extraVolumes: []
  fullnameOverride: ""
  global: {}
  hostAliases: {}
  image:
    pullPolicy: IfNotPresent
    repository: quay.io/prometheus/alertmanager
    tag: ""
  imagePullSecrets: []
  ingress:
    annotations: {}
    className: ""
    enabled: false
    hosts:
    - host: alertmanager.domain.com
      paths:
      - path: /
        pathType: ImplementationSpecific
    tls: []
  livenessProbe:
    httpGet:
      path: /
      port: http
  nameOverride: ""
  nodeSelector: {}
  persistence:
    accessModes:
    - ReadWriteOnce
    enabled: true
    size: 2Gi
  podAnnotations: {}
  podAntiAffinity: ""
  podAntiAffinityTopologyKey: kubernetes.io/hostname
  podDisruptionBudget: {}
  podLabels: {}
  podSecurityContext:
    fsGroup: 65534
    runAsGroup: 65534
    runAsNonRoot: true
    runAsUser: 65534
  priorityClassName: ""
  readinessProbe:
    httpGet:
      path: /
      port: http
  replicaCount: 1
  resources: {}
  securityContext:
    runAsGroup: 65534
    runAsNonRoot: true
    runAsUser: 65534
  service:
    annotations: {}
    clusterPort: 9094
    loadBalancerIP: ""
    loadBalancerSourceRanges: []
    port: 9093
    type: ClusterIP
  serviceAccount:
    annotations: {}
    create: true
    name: null
  statefulSet:
    annotations: {}
  templates: {}
  testFramework:
    annotations:
      helm.sh/hook: test-success
    enabled: false
  tolerations: []
  topologySpreadConstraints: []
configmapReload:
  prometheus:
    containerSecurityContext: {}
    enabled: true
    extraArgs: {}
    extraConfigmapMounts: []
    extraVolumeDirs: []
    image:
      digest: ""
      pullPolicy: IfNotPresent
      repository: quay.io/prometheus-operator/prometheus-config-reloader
      tag: v0.63.0
    name: configmap-reload
    resources: {}
extraManifests: []
extraScrapeConfigs: ""
forceNamespace: ""
imagePullSecrets: []
kube-state-metrics:
  affinity: {}
  annotations: {}
  autosharding:
    enabled: false
  collectors:
  - certificatesigningrequests
  - configmaps
  - cronjobs
  - daemonsets
  - deployments
  - endpoints
  - horizontalpodautoscalers
  - ingresses
  - jobs
  - leases
  - limitranges
  - mutatingwebhookconfigurations
  - namespaces
  - networkpolicies
  - nodes
  - persistentvolumeclaims
  - persistentvolumes
  - poddisruptionbudgets
  - pods
  - replicasets
  - replicationcontrollers
  - resourcequotas
  - secrets
  - services
  - statefulsets
  - storageclasses
  - validatingwebhookconfigurations
  - volumeattachments
  containerSecurityContext: {}
  customLabels: {}
  enabled: true
  extraArgs: []
  global:
    imagePullSecrets: []
  hostNetwork: false
  image:
    pullPolicy: IfNotPresent
    repository: registry.k8s.io/kube-state-metrics/kube-state-metrics
    sha: ""
    tag: ""
  imagePullSecrets: []
  kubeRBACProxy:
    containerSecurityContext: {}
    enabled: false
    extraArgs: []
    image:
      pullPolicy: IfNotPresent
      repository: quay.io/brancz/kube-rbac-proxy
      sha: ""
      tag: v0.14.0
    resources: {}
  kubeTargetVersionOverride: ""
  kubeconfig:
    enabled: false
  metricAllowlist: []
  metricAnnotationsAllowList: []
  metricDenylist: []
  metricLabelsAllowlist: []
  namespaceOverride: ""
  namespaces: ""
  namespacesDenylist: ""
  nodeSelector: {}
  podAnnotations: {}
  podDisruptionBudget: {}
  podSecurityPolicy:
    additionalVolumes: []
    annotations: {}
    enabled: false
  prometheus:
    monitor:
      additionalLabels: {}
      enabled: false
      honorLabels: false
      interval: ""
      jobLabel: ""
      labelLimit: 0
      labelNameLengthLimit: 0
      labelValueLengthLimit: 0
      metricRelabelings: []
      namespace: ""
      podTargetLabels: []
      proxyUrl: ""
      relabelings: []
      sampleLimit: 0
      scheme: ""
      scrapeTimeout: ""
      selectorOverride: {}
      targetLabels: []
      targetLimit: 0
      tlsConfig: {}
  prometheusScrape: true
  rbac:
    create: true
    extraRules: []
    useClusterRole: true
  releaseLabel: false
  releaseNamespace: false
  replicas: 1
  resources: {}
  securityContext:
    enabled: true
    fsGroup: 65534
    runAsGroup: 65534
    runAsUser: 65534
  selectorOverride: {}
  selfMonitor:
    enabled: false
  service:
    annotations: {}
    clusterIP: ""
    loadBalancerIP: ""
    loadBalancerSourceRanges: []
    nodePort: 0
    port: 8080
    type: ClusterIP
  serviceAccount:
    annotations: {}
    create: true
    imagePullSecrets: []
  tolerations: []
  topologySpreadConstraints: []
  verticalPodAutoscaler:
    controlledResources: []
    enabled: false
    maxAllowed: {}
    minAllowed: {}
  volumeMounts: []
  volumes: []
networkPolicy:
  enabled: false
podSecurityPolicy:
  enabled: false
prometheus-node-exporter:
  affinity: {}
  configmaps: []
  containerSecurityContext:
    allowPrivilegeEscalation: false
  daemonsetAnnotations: {}
  dnsConfig: {}
  enabled: true
  endpoints: []
  env: {}
  extraArgs: []
  extraHostVolumeMounts: []
  extraInitContainers: []
  global: {}
  hostNetwork: true
  hostPID: true
  hostRootFsMount:
    enabled: true
    mountPropagation: HostToContainer
  image:
    pullPolicy: IfNotPresent
    repository: quay.io/prometheus/node-exporter
    sha: ""
    tag: ""
  imagePullSecrets: []
  livenessProbe:
    failureThreshold: 3
    httpGet:
      httpHeaders: []
      scheme: http
    initialDelaySeconds: 0
    periodSeconds: 10
    successThreshold: 1
    timeoutSeconds: 1
  namespaceOverride: ""
  nodeSelector: {}
  podAnnotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
  podLabels: {}
  prometheus:
    monitor:
      additionalLabels: {}
      apiVersion: ""
      basicAuth: {}
      enabled: false
      interval: ""
      jobLabel: ""
      labelLimit: 0
      labelNameLengthLimit: 0
      labelValueLengthLimit: 0
      metricRelabelings: []
      namespace: ""
      proxyUrl: ""
      relabelings: []
      sampleLimit: 0
      scheme: http
      scrapeTimeout: 10s
      selectorOverride: {}
      targetLimit: 0
      tlsConfig: {}
  rbac:
    create: true
    pspAnnotations: {}
    pspEnabled: false
  readinessProbe:
    failureThreshold: 3
    httpGet:
      httpHeaders: []
      scheme: http
    initialDelaySeconds: 0
    periodSeconds: 10
    successThreshold: 1
    timeoutSeconds: 1
  releaseLabel: false
  resources: {}
  secrets: []
  securityContext:
    fsGroup: 65534
    runAsGroup: 65534
    runAsNonRoot: true
    runAsUser: 65534
  service:
    annotations:
      prometheus.io/scrape: "true"
    listenOnAllInterfaces: true
    port: 9100
    portName: metrics
    targetPort: 9100
    type: ClusterIP
  serviceAccount:
    annotations: {}
    automountServiceAccountToken: false
    create: true
    imagePullSecrets: []
  sidecarHostVolumeMounts: []
  sidecarVolumeMount: []
  sidecars: []
  tolerations:
  - effect: NoSchedule
    operator: Exists
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
  verticalPodAutoscaler:
    controlledResources: []
    enabled: false
    maxAllowed: {}
    minAllowed: {}
prometheus-pushgateway:
  affinity: {}
  containerSecurityContext: {}
  enabled: false
  extraArgs: []
  extraContainers: []
  extraInitContainers: []
  extraVars: []
  extraVolumeMounts: []
  extraVolumes: []
  fullnameOverride: ""
  global: {}
  image:
    pullPolicy: IfNotPresent
    repository: prom/pushgateway
    tag: ""
  imagePullSecrets: []
  ingress:
    className: ""
    enabled: false
    extraPaths: []
    path: /
    pathType: ImplementationSpecific
  liveness:
    enabled: true
    probe:
      httpGet:
        path: /-/ready
        port: 9091
      initialDelaySeconds: 10
      timeoutSeconds: 10
  nameOverride: ""
  networkPolicy: {}
  nodeSelector: {}
  persistentVolume:
    accessModes:
    - ReadWriteOnce
    annotations: {}
    enabled: false
    existingClaim: ""
    mountPath: /data
    size: 2Gi
    subPath: ""
  persistentVolumeLabels: {}
  podAnnotations: {}
  podDisruptionBudget: {}
  podLabels: {}
  priorityClassName: null
  readiness:
    enabled: true
    probe:
      httpGet:
        path: /-/ready
        port: 9091
      initialDelaySeconds: 10
      timeoutSeconds: 10
  replicaCount: 1
  resources: {}
  runAsStatefulSet: false
  securityContext:
    fsGroup: 65534
    runAsNonRoot: true
    runAsUser: 65534
  service:
    clusterIP: ""
    loadBalancerIP: ""
    loadBalancerSourceRanges: []
    port: 9091
    targetPort: 9091
    type: ClusterIP
  serviceAccount:
    create: true
    name: null
  serviceAccountLabels: {}
  serviceAnnotations:
    prometheus.io/probe: pushgateway
  serviceLabels: {}
  serviceMonitor:
    additionalLabels: {}
    enabled: false
    honorLabels: true
    metricRelabelings: []
    namespace: monitoring
    relabelings: []
  strategy:
    type: Recreate
  tolerations: {}
  topologySpreadConstraints: []
rbac:
  create: true
ruleFiles: {}
server:
  affinity: {}
  alertmanagers: []
  baseURL: ""
  command: []
  configMapOverrideName: ""
  configPath: /etc/config/prometheus.yml
  containerSecurityContext: {}
  defaultFlagsOverride: []
  deploymentAnnotations: {}
  dnsConfig: {}
  dnsPolicy: ClusterFirst
  emptyDir:
    sizeLimit: ""
  enableServiceLinks: true
  env: []
  exemplars: {}
  extraArgs: {}
  extraConfigmapLabels: {}
  extraConfigmapMounts: []
  extraFlags:
  - web.enable-lifecycle
  extraHostPathMounts: []
  extraInitContainers: []
  extraSecretMounts: []
  extraVolumeMounts: []
  extraVolumes: []
  global:
    evaluation_interval: 1m
    scrape_interval: 1m
    scrape_timeout: 10s
  hostAliases: []
  hostNetwork: false
  image:
    digest: ""
    pullPolicy: IfNotPresent
    repository: quay.io/prometheus/prometheus
    tag: ""
  ingress:
    annotations: {}
    enabled: false
    extraLabels: {}
    extraPaths: []
    hosts: []
    path: /
    pathType: Prefix
    tls: []
  livenessProbeFailureThreshold: 3
  livenessProbeInitialDelay: 30
  livenessProbePeriodSeconds: 15
  livenessProbeSuccessThreshold: 1
  livenessProbeTimeout: 10
  name: server
  nodeSelector: {}
  persistentVolume:
    accessModes:
    - ReadWriteMany
    annotations: {}
    enabled: true
    existingClaim: ""
    labels: {}
    mountPath: /data
    size: 8Gi
    statefulSetNameOverride: null
    storageClass: nas-sc
    subPath: ""
  podAnnotations: {}
  podDisruptionBudget:
    enabled: false
    maxUnavailable: 1
  podLabels: {}
  podSecurityPolicy:
    annotations: {}
  prefixURL: ""
  priorityClassName: ""
  probeHeaders: []
  probeScheme: HTTP
  readinessProbeFailureThreshold: 3
  readinessProbeInitialDelay: 30
  readinessProbePeriodSeconds: 5
  readinessProbeSuccessThreshold: 1
  readinessProbeTimeout: 4
  remoteRead: []
  remoteWrite: []
  replicaCount: 1
  resources: {}
  retention: 15d
  securityContext:
    fsGroup: 65534
    runAsGroup: 65534
    runAsNonRoot: true
    runAsUser: 65534
  service:
    annotations: {}
    clusterIP: ""
    enabled: true
    externalIPs: []
    gRPC:
      enabled: false
      servicePort: 10901
    labels: {}
    loadBalancerIP: ""
    loadBalancerSourceRanges: []
    servicePort: 80
    sessionAffinity: None
    statefulsetReplica:
      enabled: false
      replica: 0
    type: ClusterIP
  sidecarContainers: {}
  sidecarTemplateValues: {}
  startupProbe:
    enabled: false
    failureThreshold: 30
    periodSeconds: 5
    timeoutSeconds: 10
  statefulSet:
    annotations: {}
    enabled: false
    headless:
      annotations: {}
      gRPC:
        enabled: false
        servicePort: 10901
      labels: {}
      servicePort: 80
    labels: {}
    podManagementPolicy: OrderedReady
  storagePath: ""
  strategy:
    type: Recreate
  tcpSocketProbeEnabled: false
  terminationGracePeriodSeconds: 300
  tolerations: []
  topologySpreadConstraints: []
  tsdb: {}
  verticalAutoscaler:
    enabled: false
serverFiles:
  alerting_rules.yml: {}
  alerts: {}
  prometheus.yml:
    rule_files:
    - /etc/config/recording_rules.yml
    - /etc/config/alerting_rules.yml
    - /etc/config/rules
    - /etc/config/alerts
    scrape_configs:
    - job_name: prometheus
      static_configs:
      - targets:
        - localhost:9090
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-apiservers
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: default;kubernetes;https
        source_labels:
        - __meta_kubernetes_namespace
        - __meta_kubernetes_service_name
        - __meta_kubernetes_endpoint_port_name
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-nodes
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - replacement: kubernetes.default.svc:443
        target_label: __address__
      - regex: (.+)
        replacement: /api/v1/nodes/$1/proxy/metrics
        source_labels:
        - __meta_kubernetes_node_name
        target_label: __metrics_path__
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-nodes-cadvisor
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - replacement: kubernetes.default.svc:443
        target_label: __address__
      - regex: (.+)
        replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
        source_labels:
        - __meta_kubernetes_node_name
        target_label: __metrics_path__
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - honor_labels: true
      job_name: kubernetes-service-endpoints
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (.+?)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: service
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
    - honor_labels: true
      job_name: kubernetes-service-endpoints-slow
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (.+?)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: service
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
      scrape_interval: 5m
      scrape_timeout: 30s
    - honor_labels: true
      job_name: prometheus-pushgateway
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - action: keep
        regex: pushgateway
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
    - honor_labels: true
      job_name: kubernetes-services
      kubernetes_sd_configs:
      - role: service
      metrics_path: /probe
      params:
        module:
        - http_2xx
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
      - source_labels:
        - __address__
        target_label: __param_target
      - replacement: blackbox
        target_label: __address__
      - source_labels:
        - __param_target
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - source_labels:
        - __meta_kubernetes_service_name
        target_label: service
    - honor_labels: true
      job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
        replacement: '[$2]:$1'
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: replace
        regex: (\d+);((([0-9]+?)(\.|$)){4})
        replacement: $2:$1
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
      - action: drop
        regex: Pending|Succeeded|Failed|Completed
        source_labels:
        - __meta_kubernetes_pod_phase
    - honor_labels: true
      job_name: kubernetes-pods-slow
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
        replacement: '[$2]:$1'
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: replace
        regex: (\d+);((([0-9]+?)(\.|$)){4})
        replacement: $2:$1
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
      - action: drop
        regex: Pending|Succeeded|Failed|Completed
        source_labels:
        - __meta_kubernetes_pod_phase
      scrape_interval: 5m
      scrape_timeout: 30s
  recording_rules.yml: {}
  rules: {}
serviceAccounts:
  server:
    annotations: {}
    create: true
    name: ""

HOOKS:
MANIFEST:
---
# Source: prometheus/charts/kube-state-metrics/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:    
    helm.sh/chart: kube-state-metrics-4.30.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: kube-state-metrics
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "2.8.0"
  name: prometheus-kube-state-metrics
  namespace: monitoring
imagePullSecrets:
---
# Source: prometheus/charts/prometheus-node-exporter/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-prometheus-node-exporter
  namespace: monitoring
  labels:
    helm.sh/chart: prometheus-node-exporter-4.8.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: prometheus-node-exporter
    app.kubernetes.io/name: prometheus-node-exporter
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "1.5.0"
---
# Source: prometheus/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
  namespace: monitoring
  annotations:
    {}
---
# Source: prometheus/templates/cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
  namespace: monitoring
data:
  allow-snippet-annotations: "false"
  alerting_rules.yml: |
    {}
  alerts: |
    {}
  prometheus.yml: |
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
    - job_name: prometheus
      static_configs:
      - targets:
        - localhost:9090
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-apiservers
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: default;kubernetes;https
        source_labels:
        - __meta_kubernetes_namespace
        - __meta_kubernetes_service_name
        - __meta_kubernetes_endpoint_port_name
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-nodes
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - replacement: kubernetes.default.svc:443
        target_label: __address__
      - regex: (.+)
        replacement: /api/v1/nodes/$1/proxy/metrics
        source_labels:
        - __meta_kubernetes_node_name
        target_label: __metrics_path__
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-nodes-cadvisor
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - replacement: kubernetes.default.svc:443
        target_label: __address__
      - regex: (.+)
        replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
        source_labels:
        - __meta_kubernetes_node_name
        target_label: __metrics_path__
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - honor_labels: true
      job_name: kubernetes-service-endpoints
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (.+?)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: service
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
    - honor_labels: true
      job_name: kubernetes-service-endpoints-slow
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (.+?)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: service
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
      scrape_interval: 5m
      scrape_timeout: 30s
    - honor_labels: true
      job_name: prometheus-pushgateway
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - action: keep
        regex: pushgateway
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
    - honor_labels: true
      job_name: kubernetes-services
      kubernetes_sd_configs:
      - role: service
      metrics_path: /probe
      params:
        module:
        - http_2xx
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
      - source_labels:
        - __address__
        target_label: __param_target
      - replacement: blackbox
        target_label: __address__
      - source_labels:
        - __param_target
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - source_labels:
        - __meta_kubernetes_service_name
        target_label: service
    - honor_labels: true
      job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
        replacement: '[$2]:$1'
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: replace
        regex: (\d+);((([0-9]+?)(\.|$)){4})
        replacement: $2:$1
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
      - action: drop
        regex: Pending|Succeeded|Failed|Completed
        source_labels:
        - __meta_kubernetes_pod_phase
    - honor_labels: true
      job_name: kubernetes-pods-slow
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
        replacement: '[$2]:$1'
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: replace
        regex: (\d+);((([0-9]+?)(\.|$)){4})
        replacement: $2:$1
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
      - action: drop
        regex: Pending|Succeeded|Failed|Completed
        source_labels:
        - __meta_kubernetes_pod_phase
      scrape_interval: 5m
      scrape_timeout: 30s
  recording_rules.yml: |
    {}
  rules: |
    {}
---
# Source: prometheus/templates/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: "nas-sc"
  resources:
    requests:
      storage: "8Gi"
---
# Source: prometheus/charts/kube-state-metrics/templates/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:    
    helm.sh/chart: kube-state-metrics-4.30.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: kube-state-metrics
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "2.8.0"
  name: prometheus-kube-state-metrics
rules:

- apiGroups: ["certificates.k8s.io"]
  resources:
  - certificatesigningrequests
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["list", "watch"]

- apiGroups: ["batch"]
  resources:
  - cronjobs
  verbs: ["list", "watch"]

- apiGroups: ["extensions", "apps"]
  resources:
  - daemonsets
  verbs: ["list", "watch"]

- apiGroups: ["extensions", "apps"]
  resources:
  - deployments
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - endpoints
  verbs: ["list", "watch"]

- apiGroups: ["autoscaling"]
  resources:
  - horizontalpodautoscalers
  verbs: ["list", "watch"]

- apiGroups: ["extensions", "networking.k8s.io"]
  resources:
  - ingresses
  verbs: ["list", "watch"]

- apiGroups: ["batch"]
  resources:
  - jobs
  verbs: ["list", "watch"]

- apiGroups: ["coordination.k8s.io"]
  resources:
  - leases
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - limitranges
  verbs: ["list", "watch"]

- apiGroups: ["admissionregistration.k8s.io"]
  resources:
    - mutatingwebhookconfigurations
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - namespaces
  verbs: ["list", "watch"]

- apiGroups: ["networking.k8s.io"]
  resources:
  - networkpolicies
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - nodes
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - persistentvolumeclaims
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - persistentvolumes
  verbs: ["list", "watch"]

- apiGroups: ["policy"]
  resources:
    - poddisruptionbudgets
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - pods
  verbs: ["list", "watch"]

- apiGroups: ["extensions", "apps"]
  resources:
  - replicasets
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - replicationcontrollers
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - resourcequotas
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - secrets
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - services
  verbs: ["list", "watch"]

- apiGroups: ["apps"]
  resources:
  - statefulsets
  verbs: ["list", "watch"]

- apiGroups: ["storage.k8s.io"]
  resources:
    - storageclasses
  verbs: ["list", "watch"]

- apiGroups: ["admissionregistration.k8s.io"]
  resources:
    - validatingwebhookconfigurations
  verbs: ["list", "watch"]

- apiGroups: ["storage.k8s.io"]
  resources:
    - volumeattachments
  verbs: ["list", "watch"]
---
# Source: prometheus/templates/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
rules:
  - apiGroups:
      - ""
    resources:
      - nodes
      - nodes/proxy
      - nodes/metrics
      - services
      - endpoints
      - pods
      - ingresses
      - configmaps
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
      - ingresses
    verbs:
      - get
      - list
      - watch
  - nonResourceURLs:
      - "/metrics"
    verbs:
      - get
---
# Source: prometheus/charts/kube-state-metrics/templates/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:    
    helm.sh/chart: kube-state-metrics-4.30.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: kube-state-metrics
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "2.8.0"
  name: prometheus-kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-kube-state-metrics
subjects:
- kind: ServiceAccount
  name: prometheus-kube-state-metrics
  namespace: monitoring
---
# Source: prometheus/templates/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
subjects:
  - kind: ServiceAccount
    name: prometheus-server
    namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-server
---
# Source: prometheus/charts/kube-state-metrics/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-kube-state-metrics
  namespace: monitoring
  labels:    
    helm.sh/chart: kube-state-metrics-4.30.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: kube-state-metrics
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "2.8.0"
  annotations:
    prometheus.io/scrape: 'true'
spec:
  type: "ClusterIP"
  ports:
  - name: "http"
    protocol: TCP
    port: 8080
    targetPort: 8080
  
  selector:    
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/instance: prometheus
---
# Source: prometheus/charts/prometheus-node-exporter/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-prometheus-node-exporter
  namespace: monitoring
  labels:
    helm.sh/chart: prometheus-node-exporter-4.8.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: prometheus-node-exporter
    app.kubernetes.io/name: prometheus-node-exporter
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "1.5.0"
  annotations:
    prometheus.io/scrape: "true"
spec:
  type: ClusterIP
  ports:
    - port: 9100
      targetPort: 9100
      protocol: TCP
      name: metrics
  selector:
    app.kubernetes.io/name: prometheus-node-exporter
    app.kubernetes.io/instance: prometheus
---
# Source: prometheus/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
  namespace: monitoring
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 9090
  selector:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
  sessionAffinity: None
  type: "ClusterIP"
---
# Source: prometheus/charts/prometheus-node-exporter/templates/daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: prometheus-prometheus-node-exporter
  namespace: monitoring
  labels:
    helm.sh/chart: prometheus-node-exporter-4.8.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: prometheus-node-exporter
    app.kubernetes.io/name: prometheus-node-exporter
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "1.5.0"
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: prometheus-node-exporter
      app.kubernetes.io/instance: prometheus
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
      labels:
        helm.sh/chart: prometheus-node-exporter-4.8.1
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/component: metrics
        app.kubernetes.io/part-of: prometheus-node-exporter
        app.kubernetes.io/name: prometheus-node-exporter
        app.kubernetes.io/instance: prometheus
        app.kubernetes.io/version: "1.5.0"
    spec:
      automountServiceAccountToken: false
      securityContext:
        fsGroup: 65534
        runAsGroup: 65534
        runAsNonRoot: true
        runAsUser: 65534
      serviceAccountName: prometheus-prometheus-node-exporter
      containers:
        - name: node-exporter
          image: quay.io/prometheus/node-exporter:v1.5.0
          imagePullPolicy: IfNotPresent
          args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
            - --path.rootfs=/host/root
            - --web.listen-address=[$(HOST_IP)]:9100
          securityContext:
            allowPrivilegeEscalation: false
          env:
            - name: HOST_IP
              value: 0.0.0.0
          ports:
            - name: metrics
              containerPort: 9100
              protocol: TCP
          livenessProbe:
            failureThreshold: 3
            httpGet:
              httpHeaders:
              path: /
              port: 9100
              scheme: HTTP
            initialDelaySeconds: 0
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              httpHeaders:
              path: /
              port: 9100
              scheme: HTTP
            initialDelaySeconds: 0
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly:  true
            - name: sys
              mountPath: /host/sys
              readOnly: true
            - name: root
              mountPath: /host/root
              mountPropagation: HostToContainer
              readOnly: true
      hostNetwork: true
      hostPID: true
      tolerations:
        - effect: NoSchedule
          operator: Exists
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: root
          hostPath:
            path: /
---
# Source: prometheus/charts/kube-state-metrics/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-kube-state-metrics
  namespace: monitoring
  labels:    
    helm.sh/chart: kube-state-metrics-4.30.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: kube-state-metrics
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "2.8.0"
spec:
  selector:
    matchLabels:      
      app.kubernetes.io/name: kube-state-metrics
      app.kubernetes.io/instance: prometheus
  replicas: 1
  template:
    metadata:
      labels:        
        helm.sh/chart: kube-state-metrics-4.30.0
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/component: metrics
        app.kubernetes.io/part-of: kube-state-metrics
        app.kubernetes.io/name: kube-state-metrics
        app.kubernetes.io/instance: prometheus
        app.kubernetes.io/version: "2.8.0"
    spec:
      hostNetwork: false
      serviceAccountName: prometheus-kube-state-metrics
      securityContext:
        fsGroup: 65534
        runAsGroup: 65534
        runAsUser: 65534
      containers:
      - name: kube-state-metrics
        args:
        - --port=8080
        - --resources=certificatesigningrequests,configmaps,cronjobs,daemonsets,deployments,endpoints,horizontalpodautoscalers,ingresses,jobs,leases,limitranges,mutatingwebhookconfigurations,namespaces,networkpolicies,nodes,persistentvolumeclaims,persistentvolumes,poddisruptionbudgets,pods,replicasets,replicationcontrollers,resourcequotas,secrets,services,statefulsets,storageclasses,validatingwebhookconfigurations,volumeattachments
        imagePullPolicy: IfNotPresent
        image: "registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.8.0"
        ports:
        - containerPort: 8080
          name: "http"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
---
# Source: prometheus/templates/deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: server
      app.kubernetes.io/name: prometheus
      app.kubernetes.io/instance: prometheus
  replicas: 1
  strategy:
    type: Recreate
    rollingUpdate: null
  template:
    metadata:
      labels:
        app.kubernetes.io/component: server
        app.kubernetes.io/name: prometheus
        app.kubernetes.io/instance: prometheus
        app.kubernetes.io/version: v2.43.0
        helm.sh/chart: prometheus-22.2.0
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/part-of: prometheus
    spec:
      enableServiceLinks: true
      serviceAccountName: prometheus-server
      containers:
        - name: prometheus-server-configmap-reload
          image: "quay.io/prometheus-operator/prometheus-config-reloader:v0.63.0"
          imagePullPolicy: "IfNotPresent"
          args:
            - --watched-dir=/etc/config
            - --reload-url=http://127.0.0.1:9090/-/reload
          resources:
            {}
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true

        - name: prometheus-server
          image: "quay.io/prometheus/prometheus:v2.43.0"
          imagePullPolicy: "IfNotPresent"
          args:
            - --storage.tsdb.retention.time=15d
            - --config.file=/etc/config/prometheus.yml
            - --storage.tsdb.path=/data
            - --web.console.libraries=/etc/prometheus/console_libraries
            - --web.console.templates=/etc/prometheus/consoles
            - --web.enable-lifecycle
          ports:
            - containerPort: 9090
          readinessProbe:
            httpGet:
              path: /-/ready
              port: 9090
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 5
            timeoutSeconds: 4
            failureThreshold: 3
            successThreshold: 1
          livenessProbe:
            httpGet:
              path: /-/healthy
              port: 9090
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 15
            timeoutSeconds: 10
            failureThreshold: 3
            successThreshold: 1
          resources:
            {}
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
            - name: storage-volume
              mountPath: /data
              subPath: ""
      dnsPolicy: ClusterFirst
      securityContext:
        fsGroup: 65534
        runAsGroup: 65534
        runAsNonRoot: true
        runAsUser: 65534
      terminationGracePeriodSeconds: 300
      volumes:
        - name: config-volume
          configMap:
            name: prometheus-server
        - name: storage-volume
          persistentVolumeClaim:
            claimName: prometheus-server

NOTES:
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.monitoring.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9090


#################################################################################
######   WARNING: Pod Security Policy has been disabled by default since    #####
######            it deprecated after k8s 1.25+. use                        #####
######            (index .Values "prometheus-node-exporter" "rbac"          #####
###### .          "pspEnabled") with (index .Values                         #####
######            "prometheus-node-exporter" "rbac" "pspAnnotations")       #####
######            in case you still need it.                                #####
#################################################################################



For more information on running Prometheus, visit:
https://prometheus.io/

```
</details>

## 登录prometheus

```bash
#1. 通过如下做一下svc的映射： kubectl port-forward svc/[service-name] -n [namespace] [external-port]:[internal-port]
$ sudo kubectl port-forward svc/prometheus-server -n monitoring 80:80 --address='0.0.0.0'

# 2. 通过本地IP+80即可访问
```

## 查看manifest

```bash
$ sudo helm get manifest prometheus --namespace monitoring
```
<details>
  <summary>点击展开查看manifest</summary>

```yaml
---
# Source: prometheus/charts/kube-state-metrics/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:    
    helm.sh/chart: kube-state-metrics-4.30.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: kube-state-metrics
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "2.8.0"
  name: prometheus-kube-state-metrics
  namespace: monitoring
imagePullSecrets:
---
# Source: prometheus/charts/prometheus-node-exporter/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-prometheus-node-exporter
  namespace: monitoring
  labels:
    helm.sh/chart: prometheus-node-exporter-4.8.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: prometheus-node-exporter
    app.kubernetes.io/name: prometheus-node-exporter
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "1.5.0"
---
# Source: prometheus/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
  namespace: monitoring
  annotations:
    {}
---
# Source: prometheus/templates/cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
  namespace: monitoring
data:
  allow-snippet-annotations: "false"
  alerting_rules.yml: |
    {}
  alerts: |
    {}
  prometheus.yml: |
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
    - job_name: prometheus
      static_configs:
      - targets:
        - localhost:9090
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-apiservers
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: default;kubernetes;https
        source_labels:
        - __meta_kubernetes_namespace
        - __meta_kubernetes_service_name
        - __meta_kubernetes_endpoint_port_name
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-nodes
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - replacement: kubernetes.default.svc:443
        target_label: __address__
      - regex: (.+)
        replacement: /api/v1/nodes/$1/proxy/metrics
        source_labels:
        - __meta_kubernetes_node_name
        target_label: __metrics_path__
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-nodes-cadvisor
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - replacement: kubernetes.default.svc:443
        target_label: __address__
      - regex: (.+)
        replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
        source_labels:
        - __meta_kubernetes_node_name
        target_label: __metrics_path__
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
    - honor_labels: true
      job_name: kubernetes-service-endpoints
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (.+?)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: service
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
    - honor_labels: true
      job_name: kubernetes-service-endpoints-slow
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (.+?)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: service
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
      scrape_interval: 5m
      scrape_timeout: 30s
    - honor_labels: true
      job_name: prometheus-pushgateway
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - action: keep
        regex: pushgateway
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
    - honor_labels: true
      job_name: kubernetes-services
      kubernetes_sd_configs:
      - role: service
      metrics_path: /probe
      params:
        module:
        - http_2xx
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
      - source_labels:
        - __address__
        target_label: __param_target
      - replacement: blackbox
        target_label: __address__
      - source_labels:
        - __param_target
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - source_labels:
        - __meta_kubernetes_service_name
        target_label: service
    - honor_labels: true
      job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
        replacement: '[$2]:$1'
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: replace
        regex: (\d+);((([0-9]+?)(\.|$)){4})
        replacement: $2:$1
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
      - action: drop
        regex: Pending|Succeeded|Failed|Completed
        source_labels:
        - __meta_kubernetes_pod_phase
    - honor_labels: true
      job_name: kubernetes-pods-slow
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
        replacement: '[$2]:$1'
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: replace
        regex: (\d+);((([0-9]+?)(\.|$)){4})
        replacement: $2:$1
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
      - action: drop
        regex: Pending|Succeeded|Failed|Completed
        source_labels:
        - __meta_kubernetes_pod_phase
      scrape_interval: 5m
      scrape_timeout: 30s
  recording_rules.yml: |
    {}
  rules: |
    {}
---
# Source: prometheus/templates/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: "nas-sc"
  resources:
    requests:
      storage: "8Gi"
---
# Source: prometheus/charts/kube-state-metrics/templates/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:    
    helm.sh/chart: kube-state-metrics-4.30.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: kube-state-metrics
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "2.8.0"
  name: prometheus-kube-state-metrics
rules:

- apiGroups: ["certificates.k8s.io"]
  resources:
  - certificatesigningrequests
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["list", "watch"]

- apiGroups: ["batch"]
  resources:
  - cronjobs
  verbs: ["list", "watch"]

- apiGroups: ["extensions", "apps"]
  resources:
  - daemonsets
  verbs: ["list", "watch"]

- apiGroups: ["extensions", "apps"]
  resources:
  - deployments
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - endpoints
  verbs: ["list", "watch"]

- apiGroups: ["autoscaling"]
  resources:
  - horizontalpodautoscalers
  verbs: ["list", "watch"]

- apiGroups: ["extensions", "networking.k8s.io"]
  resources:
  - ingresses
  verbs: ["list", "watch"]

- apiGroups: ["batch"]
  resources:
  - jobs
  verbs: ["list", "watch"]

- apiGroups: ["coordination.k8s.io"]
  resources:
  - leases
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - limitranges
  verbs: ["list", "watch"]

- apiGroups: ["admissionregistration.k8s.io"]
  resources:
    - mutatingwebhookconfigurations
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - namespaces
  verbs: ["list", "watch"]

- apiGroups: ["networking.k8s.io"]
  resources:
  - networkpolicies
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - nodes
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - persistentvolumeclaims
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - persistentvolumes
  verbs: ["list", "watch"]

- apiGroups: ["policy"]
  resources:
    - poddisruptionbudgets
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - pods
  verbs: ["list", "watch"]

- apiGroups: ["extensions", "apps"]
  resources:
  - replicasets
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - replicationcontrollers
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - resourcequotas
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - secrets
  verbs: ["list", "watch"]

- apiGroups: [""]
  resources:
  - services
  verbs: ["list", "watch"]

- apiGroups: ["apps"]
  resources:
  - statefulsets
  verbs: ["list", "watch"]

- apiGroups: ["storage.k8s.io"]
  resources:
    - storageclasses
  verbs: ["list", "watch"]

- apiGroups: ["admissionregistration.k8s.io"]
  resources:
    - validatingwebhookconfigurations
  verbs: ["list", "watch"]

- apiGroups: ["storage.k8s.io"]
  resources:
    - volumeattachments
  verbs: ["list", "watch"]
---
# Source: prometheus/templates/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
rules:
  - apiGroups:
      - ""
    resources:
      - nodes
      - nodes/proxy
      - nodes/metrics
      - services
      - endpoints
      - pods
      - ingresses
      - configmaps
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
      - ingresses
    verbs:
      - get
      - list
      - watch
  - nonResourceURLs:
      - "/metrics"
    verbs:
      - get
---
# Source: prometheus/charts/kube-state-metrics/templates/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:    
    helm.sh/chart: kube-state-metrics-4.30.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: kube-state-metrics
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "2.8.0"
  name: prometheus-kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-kube-state-metrics
subjects:
- kind: ServiceAccount
  name: prometheus-kube-state-metrics
  namespace: monitoring
---
# Source: prometheus/templates/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
subjects:
  - kind: ServiceAccount
    name: prometheus-server
    namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-server
---
# Source: prometheus/charts/kube-state-metrics/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-kube-state-metrics
  namespace: monitoring
  labels:    
    helm.sh/chart: kube-state-metrics-4.30.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: kube-state-metrics
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "2.8.0"
  annotations:
    prometheus.io/scrape: 'true'
spec:
  type: "ClusterIP"
  ports:
  - name: "http"
    protocol: TCP
    port: 8080
    targetPort: 8080
  
  selector:    
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/instance: prometheus
---
# Source: prometheus/charts/prometheus-node-exporter/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-prometheus-node-exporter
  namespace: monitoring
  labels:
    helm.sh/chart: prometheus-node-exporter-4.8.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: prometheus-node-exporter
    app.kubernetes.io/name: prometheus-node-exporter
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "1.5.0"
  annotations:
    prometheus.io/scrape: "true"
spec:
  type: ClusterIP
  ports:
    - port: 9100
      targetPort: 9100
      protocol: TCP
      name: metrics
  selector:
    app.kubernetes.io/name: prometheus-node-exporter
    app.kubernetes.io/instance: prometheus
---
# Source: prometheus/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
  namespace: monitoring
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 9090
  selector:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
  sessionAffinity: None
  type: "ClusterIP"
---
# Source: prometheus/charts/prometheus-node-exporter/templates/daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: prometheus-prometheus-node-exporter
  namespace: monitoring
  labels:
    helm.sh/chart: prometheus-node-exporter-4.8.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: prometheus-node-exporter
    app.kubernetes.io/name: prometheus-node-exporter
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "1.5.0"
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: prometheus-node-exporter
      app.kubernetes.io/instance: prometheus
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
      labels:
        helm.sh/chart: prometheus-node-exporter-4.8.1
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/component: metrics
        app.kubernetes.io/part-of: prometheus-node-exporter
        app.kubernetes.io/name: prometheus-node-exporter
        app.kubernetes.io/instance: prometheus
        app.kubernetes.io/version: "1.5.0"
    spec:
      automountServiceAccountToken: false
      securityContext:
        fsGroup: 65534
        runAsGroup: 65534
        runAsNonRoot: true
        runAsUser: 65534
      serviceAccountName: prometheus-prometheus-node-exporter
      containers:
        - name: node-exporter
          image: quay.io/prometheus/node-exporter:v1.5.0
          imagePullPolicy: IfNotPresent
          args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
            - --path.rootfs=/host/root
            - --web.listen-address=[$(HOST_IP)]:9100
          securityContext:
            allowPrivilegeEscalation: false
          env:
            - name: HOST_IP
              value: 0.0.0.0
          ports:
            - name: metrics
              containerPort: 9100
              protocol: TCP
          livenessProbe:
            failureThreshold: 3
            httpGet:
              httpHeaders:
              path: /
              port: 9100
              scheme: HTTP
            initialDelaySeconds: 0
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              httpHeaders:
              path: /
              port: 9100
              scheme: HTTP
            initialDelaySeconds: 0
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly:  true
            - name: sys
              mountPath: /host/sys
              readOnly: true
            - name: root
              mountPath: /host/root
              mountPropagation: HostToContainer
              readOnly: true
      hostNetwork: true
      hostPID: true
      tolerations:
        - effect: NoSchedule
          operator: Exists
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: root
          hostPath:
            path: /
---
# Source: prometheus/charts/kube-state-metrics/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-kube-state-metrics
  namespace: monitoring
  labels:    
    helm.sh/chart: kube-state-metrics-4.30.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: metrics
    app.kubernetes.io/part-of: kube-state-metrics
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: "2.8.0"
spec:
  selector:
    matchLabels:      
      app.kubernetes.io/name: kube-state-metrics
      app.kubernetes.io/instance: prometheus
  replicas: 1
  template:
    metadata:
      labels:        
        helm.sh/chart: kube-state-metrics-4.30.0
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/component: metrics
        app.kubernetes.io/part-of: kube-state-metrics
        app.kubernetes.io/name: kube-state-metrics
        app.kubernetes.io/instance: prometheus
        app.kubernetes.io/version: "2.8.0"
    spec:
      hostNetwork: false
      serviceAccountName: prometheus-kube-state-metrics
      securityContext:
        fsGroup: 65534
        runAsGroup: 65534
        runAsUser: 65534
      containers:
      - name: kube-state-metrics
        args:
        - --port=8080
        - --resources=certificatesigningrequests,configmaps,cronjobs,daemonsets,deployments,endpoints,horizontalpodautoscalers,ingresses,jobs,leases,limitranges,mutatingwebhookconfigurations,namespaces,networkpolicies,nodes,persistentvolumeclaims,persistentvolumes,poddisruptionbudgets,pods,replicasets,replicationcontrollers,resourcequotas,secrets,services,statefulsets,storageclasses,validatingwebhookconfigurations,volumeattachments
        imagePullPolicy: IfNotPresent
        image: "registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.8.0"
        ports:
        - containerPort: 8080
          name: "http"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
---
# Source: prometheus/templates/deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/version: v2.43.0
    helm.sh/chart: prometheus-22.2.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: prometheus
  name: prometheus-server
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: server
      app.kubernetes.io/name: prometheus
      app.kubernetes.io/instance: prometheus
  replicas: 1
  strategy:
    type: Recreate
    rollingUpdate: null
  template:
    metadata:
      labels:
        app.kubernetes.io/component: server
        app.kubernetes.io/name: prometheus
        app.kubernetes.io/instance: prometheus
        app.kubernetes.io/version: v2.43.0
        helm.sh/chart: prometheus-22.2.0
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/part-of: prometheus
    spec:
      enableServiceLinks: true
      serviceAccountName: prometheus-server
      containers:
        - name: prometheus-server-configmap-reload
          image: "quay.io/prometheus-operator/prometheus-config-reloader:v0.63.0"
          imagePullPolicy: "IfNotPresent"
          args:
            - --watched-dir=/etc/config
            - --reload-url=http://127.0.0.1:9090/-/reload
          resources:
            {}
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true

        - name: prometheus-server
          image: "quay.io/prometheus/prometheus:v2.43.0"
          imagePullPolicy: "IfNotPresent"
          args:
            - --storage.tsdb.retention.time=15d
            - --config.file=/etc/config/prometheus.yml
            - --storage.tsdb.path=/data
            - --web.console.libraries=/etc/prometheus/console_libraries
            - --web.console.templates=/etc/prometheus/consoles
            - --web.enable-lifecycle
          ports:
            - containerPort: 9090
          readinessProbe:
            httpGet:
              path: /-/ready
              port: 9090
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 5
            timeoutSeconds: 4
            failureThreshold: 3
            successThreshold: 1
          livenessProbe:
            httpGet:
              path: /-/healthy
              port: 9090
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 15
            timeoutSeconds: 10
            failureThreshold: 3
            successThreshold: 1
          resources:
            {}
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
            - name: storage-volume
              mountPath: /data
              subPath: ""
      dnsPolicy: ClusterFirst
      securityContext:
        fsGroup: 65534
        runAsGroup: 65534
        runAsNonRoot: true
        runAsUser: 65534
      terminationGracePeriodSeconds: 300
      volumes:
        - name: config-volume
          configMap:
            name: prometheus-server
        - name: storage-volume
          persistentVolumeClaim:
            claimName: prometheus-server
```
</details>