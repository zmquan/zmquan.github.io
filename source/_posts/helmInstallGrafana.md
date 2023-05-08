---
title: Helm 安装grafana
date: 2023-05-08 11:24:47
categories: 
- 技术文档
tags: 
- k8s
- grafana
---
|软件|版本|
|:-|:-|
|grafana|Chart Version:6.56.1 <br> App Version:9.5.1|
|k8s|1.20.11|
|helm|3.11.1|

## 下载指定版本grafana
```bash
$ sudo helm repo add grafana https://grafana.github.io/helm-charts
$ sudo helm repo update
$ sudo helm fetch grafana/grafana --version 6.56.1
$ sudo tar xf  grafana-6.56.1.tgz
```
<!--more-->

<details>
  <summary>展开查看目录结构</summary>

```bash
$ tree grafana
grafana
├── Chart.yaml
├── ci
│   ├── default-values.yaml
│   ├── with-affinity-values.yaml
│   ├── with-dashboard-json-values.yaml
│   ├── with-dashboard-values.yaml
│   ├── with-extraconfigmapmounts-values.yaml
│   ├── with-image-renderer-values.yaml
│   └── with-persistence.yaml
├── dashboards
│   └── custom-dashboard.json
├── README.md
├── templates
│   ├── clusterrolebinding.yaml
│   ├── clusterrole.yaml
│   ├── configmap-dashboard-provider.yaml
│   ├── configmap.yaml
│   ├── dashboards-json-configmap.yaml
│   ├── deployment.yaml
│   ├── extra-manifests.yaml
│   ├── headless-service.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── image-renderer-deployment.yaml
│   ├── image-renderer-hpa.yaml
│   ├── image-renderer-network-policy.yaml
│   ├── image-renderer-servicemonitor.yaml
│   ├── image-renderer-service.yaml
│   ├── ingress.yaml
│   ├── networkpolicy.yaml
│   ├── NOTES.txt
│   ├── poddisruptionbudget.yaml
│   ├── podsecuritypolicy.yaml
│   ├── _pod.tpl
│   ├── pvc.yaml
│   ├── rolebinding.yaml
│   ├── role.yaml
│   ├── secret-env.yaml
│   ├── secret.yaml
│   ├── serviceaccount.yaml
│   ├── servicemonitor.yaml
│   ├── service.yaml
│   ├── statefulset.yaml
│   └── tests
│       ├── test-configmap.yaml
│       ├── test-podsecuritypolicy.yaml
│       ├── test-rolebinding.yaml
│       ├── test-role.yaml
│       ├── test-serviceaccount.yaml
│       └── test.yaml
└── values.yaml

4 directories, 47 files
```
</details>


## 查看values.yaml
```bash
$ cat  grafana/values.yaml
```

<details>
  <summary>展开查看values.yaml</summary>

```yaml
global:
  # To help compatibility with other charts which use global.imagePullSecrets.
  # Allow either an array of {name: pullSecret} maps (k8s-style), or an array of strings (more common helm-style).
  # Can be tempalted.
  # global:
  #   imagePullSecrets:
  #   - name: pullSecret1
  #   - name: pullSecret2
  # or
  # global:
  #   imagePullSecrets:
  #   - pullSecret1
  #   - pullSecret2
  imagePullSecrets: []

rbac:
  create: true
  ## Use an existing ClusterRole/Role (depending on rbac.namespaced false/true)
  # useExistingRole: name-of-some-(cluster)role
  pspEnabled: false
  pspUseAppArmor: false
  namespaced: false
  extraRoleRules: []
  # - apiGroups: []
  #   resources: []
  #   verbs: []
  extraClusterRoleRules: []
  # - apiGroups: []
  #   resources: []
  #   verbs: []
serviceAccount:
  create: true
  name:
  nameTest:
  ## ServiceAccount labels.
  labels: {}
## Service account annotations. Can be templated.
#  annotations:
#    eks.amazonaws.com/role-arn: arn:aws:iam::123456789000:role/iam-role-name-here
  autoMount: true

replicas: 1

## Create a headless service for the deployment
headlessService: false

## Create HorizontalPodAutoscaler object for deployment type
#
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 5
  targetCPU: "60"
  targetMemory: ""
  behavior: {}

## See `kubectl explain poddisruptionbudget.spec` for more
## ref: https://kubernetes.io/docs/tasks/run-application/configure-pdb/
podDisruptionBudget: {}
#  minAvailable: 1
#  maxUnavailable: 1

## See `kubectl explain deployment.spec.strategy` for more
## ref: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy
deploymentStrategy:
  type: RollingUpdate

readinessProbe:
  httpGet:
    path: /api/health
    port: 3000

livenessProbe:
  httpGet:
    path: /api/health
    port: 3000
  initialDelaySeconds: 60
  timeoutSeconds: 30
  failureThreshold: 10

## Use an alternate scheduler, e.g. "stork".
## ref: https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/
##
# schedulerName: "default-scheduler"

image:
  repository: docker.io/grafana/grafana
  # Overrides the Grafana image tag whose default is the chart appVersion
  tag: ""
  sha: ""
  pullPolicy: IfNotPresent

  ## Optionally specify an array of imagePullSecrets.
  ## Secrets must be manually created in the namespace.
  ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
  ## Can be templated.
  ##
  pullSecrets: []
  #   - myRegistrKeySecretName

testFramework:
  enabled: true
  image: docker.io/bats/bats
  tag: "v1.4.1"
  imagePullPolicy: IfNotPresent
  securityContext: {}

securityContext:
  runAsNonRoot: true
  runAsUser: 472
  runAsGroup: 472
  fsGroup: 472

containerSecurityContext:
  capabilities:
    drop:
    - ALL
  seccompProfile:
    type: RuntimeDefault

# Enable creating the grafana configmap
createConfigmap: true

# Extra configmaps to mount in grafana pods
# Values are templated.
extraConfigmapMounts: []
  # - name: certs-configmap
  #   mountPath: /etc/grafana/ssl/
  #   subPath: certificates.crt # (optional)
  #   configMap: certs-configmap
  #   readOnly: true


extraEmptyDirMounts: []
  # - name: provisioning-notifiers
  #   mountPath: /etc/grafana/provisioning/notifiers


# Apply extra labels to common labels.
extraLabels: {}

## Assign a PriorityClassName to pods if set
# priorityClassName:

downloadDashboardsImage:
  repository: docker.io/curlimages/curl
  tag: 7.85.0
  sha: ""
  pullPolicy: IfNotPresent

downloadDashboards:
  env: {}
  envFromSecret: ""
  resources: {}
  securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop:
      - ALL
    seccompProfile:
      type: RuntimeDefault
  envValueFrom: {}
  #  ENV_NAME:
  #    configMapKeyRef:
  #      name: configmap-name
  #      key: value_key

## Pod Annotations
# podAnnotations: {}

## Pod Labels
# podLabels: {}

podPortName: grafana
gossipPortName: gossip
## Deployment annotations
# annotations: {}

## Expose the grafana service to be accessed from outside the cluster (LoadBalancer service).
## or access it from within the cluster (ClusterIP service). Set the service type and the port to serve it.
## ref: http://kubernetes.io/docs/user-guide/services/
##
service:
  enabled: true
  type: ClusterIP
  port: 80
  targetPort: 3000
    # targetPort: 4181 To be used with a proxy extraContainer
  ## Service annotations. Can be templated.
  annotations: {}
  labels: {}
  portName: service
  # Adds the appProtocol field to the service. This allows to work with istio protocol selection. Ex: "http" or "tcp"
  appProtocol: ""

serviceMonitor:
  ## If true, a ServiceMonitor CRD is created for a prometheus operator
  ## https://github.com/coreos/prometheus-operator
  ##
  enabled: false
  path: /metrics
  #  namespace: monitoring  (defaults to use the namespace this chart is deployed to)
  labels: {}
  interval: 1m
  scheme: http
  tlsConfig: {}
  scrapeTimeout: 30s
  relabelings: []
  targetLabels: []

extraExposePorts: []
 # - name: keycloak
 #   port: 8080
 #   targetPort: 8080
 #   type: ClusterIP

# overrides pod.spec.hostAliases in the grafana deployment's pods
hostAliases: []
  # - ip: "1.2.3.4"
  #   hostnames:
  #     - "my.host.com"

ingress:
  enabled: false
  # For Kubernetes >= 1.18 you should specify the ingress-controller via the field ingressClassName
  # See https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/#specifying-the-class-of-an-ingress
  # ingressClassName: nginx
  # Values can be templated
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  labels: {}
  path: /

  # pathType is only for k8s >= 1.1=
  pathType: Prefix

  hosts:
    - chart-example.local
  ## Extra paths to prepend to every host configuration. This is useful when working with annotation based services.
  extraPaths: []
  # - path: /*
  #   backend:
  #     serviceName: ssl-redirect
  #     servicePort: use-annotation
  ## Or for k8s > 1.19
  # - path: /*
  #   pathType: Prefix
  #   backend:
  #     service:
  #       name: ssl-redirect
  #       port:
  #         name: use-annotation


  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

resources: {}
#  limits:
#    cpu: 100m
#    memory: 128Mi
#  requests:
#    cpu: 100m
#    memory: 128Mi

## Node labels for pod assignment
## ref: https://kubernetes.io/docs/user-guide/node-selection/
#
nodeSelector: {}

## Tolerations for pod assignment
## ref: https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/
##
tolerations: []

## Affinity for pod assignment (evaluated as template)
## ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity
##
affinity: {}

## Topology Spread Constraints
## ref: https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/
##
topologySpreadConstraints: []

## Additional init containers (evaluated as template)
## ref: https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
##
extraInitContainers: []

## Enable an Specify container in extraContainers. This is meant to allow adding an authentication proxy to a grafana pod
extraContainers: ""
# extraContainers: |
# - name: proxy
#   image: quay.io/gambol99/keycloak-proxy:latest
#   args:
#   - -provider=github
#   - -client-id=
#   - -client-secret=
#   - -github-org=<ORG_NAME>
#   - -email-domain=*
#   - -cookie-secret=
#   - -http-address=http://0.0.0.0:4181
#   - -upstream-url=http://127.0.0.1:3000
#   ports:
#     - name: proxy-web
#       containerPort: 4181

## Volumes that can be used in init containers that will not be mounted to deployment pods
extraContainerVolumes: []
#  - name: volume-from-secret
#    secret:
#      secretName: secret-to-mount
#  - name: empty-dir-volume
#    emptyDir: {}

## Enable persistence using Persistent Volume Claims
## ref: http://kubernetes.io/docs/user-guide/persistent-volumes/
##
persistence:
  type: pvc
  enabled: fasle
  # storageClassName: default
  accessModes:
    - ReadWriteOnce
  size: 10Gi
  # annotations: {}
  finalizers:
    - kubernetes.io/pvc-protection
  # selectorLabels: {}
  ## Sub-directory of the PV to mount. Can be templated.
  # subPath: ""
  ## Name of an existing PVC. Can be templated.
  # existingClaim:
  ## Extra labels to apply to a PVC.
  extraPvcLabels: {}

  ## If persistence is not enabled, this allows to mount the
  ## local storage in-memory to improve performance
  ##
  inMemory:
    enabled: false
    ## The maximum usage on memory medium EmptyDir would be
    ## the minimum value between the SizeLimit specified
    ## here and the sum of memory limits of all containers in a pod
    ##
    # sizeLimit: 300Mi

initChownData:
  ## If false, data ownership will not be reset at startup
  ## This allows the grafana-server to be run with an arbitrary user
  ##
  enabled: true

  ## initChownData container image
  ##
  image:
    repository: docker.io/library/busybox
    tag: "1.31.1"
    sha: ""
    pullPolicy: IfNotPresent

  ## initChownData resource requests and limits
  ## Ref: http://kubernetes.io/docs/user-guide/compute-resources/
  ##
  resources: {}
  #  limits:
  #    cpu: 100m
  #    memory: 128Mi
  #  requests:
  #    cpu: 100m
  #    memory: 128Mi
  securityContext:
    runAsNonRoot: false
    runAsUser: 0
    seccompProfile:
      type: RuntimeDefault
    capabilities:
      add:
        - CHOWN

# Administrator credentials when not using an existing secret (see below)
adminUser: admin
# adminPassword: strongpassword

# Use an existing secret for the admin user.
admin:
  ## Name of the secret. Can be templated.
  existingSecret: ""
  userKey: admin-user
  passwordKey: admin-password

## Define command to be executed at startup by grafana container
## Needed if using `vault-env` to manage secrets (ref: https://banzaicloud.com/blog/inject-secrets-into-pods-vault/)
## Default is "run.sh" as defined in grafana's Dockerfile
# command:
# - "sh"
# - "/run.sh"

## Optionally define args if command is used
## Needed if using `hashicorp/envconsul` to manage secrets
## By default no arguments are set
# args:
# - "-secret"
# - "secret/grafana"
# - "./grafana"

## Extra environment variables that will be pass onto deployment pods
##
## to provide grafana with access to CloudWatch on AWS EKS:
## 1. create an iam role of type "Web identity" with provider oidc.eks.* (note the provider for later)
## 2. edit the "Trust relationships" of the role, add a line inside the StringEquals clause using the
## same oidc eks provider as noted before (same as the existing line)
## also, replace NAMESPACE and prometheus-operator-grafana with the service account namespace and name
##
##  "oidc.eks.us-east-1.amazonaws.com/id/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX:sub": "system:serviceaccount:NAMESPACE:prometheus-operator-grafana",
##
## 3. attach a policy to the role, you can use a built in policy called CloudWatchReadOnlyAccess
## 4. use the following env: (replace 123456789000 and iam-role-name-here with your aws account number and role name)
##
## env:
##   AWS_ROLE_ARN: arn:aws:iam::123456789000:role/iam-role-name-here
##   AWS_WEB_IDENTITY_TOKEN_FILE: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
##   AWS_REGION: us-east-1
##
## 5. uncomment the EKS section in extraSecretMounts: below
## 6. uncomment the annotation section in the serviceAccount: above
## make sure to replace arn:aws:iam::123456789000:role/iam-role-name-here with your role arn

env: {}

## "valueFrom" environment variable references that will be added to deployment pods. Name is templated.
## ref: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#envvarsource-v1-core
## Renders in container spec as:
##   env:
##     ...
##     - name: <key>
##       valueFrom:
##         <value rendered as YAML>
envValueFrom: {}
  #  ENV_NAME:
  #    configMapKeyRef:
  #      name: configmap-name
  #      key: value_key

## The name of a secret in the same kubernetes namespace which contain values to be added to the environment
## This can be useful for auth tokens, etc. Value is templated.
envFromSecret: ""

## Sensible environment variables that will be rendered as new secret object
## This can be useful for auth tokens, etc
envRenderSecret: {}

## The names of secrets in the same kubernetes namespace which contain values to be added to the environment
## Each entry should contain a name key, and can optionally specify whether the secret must be defined with an optional key.
## Name is templated.
envFromSecrets: []
## - name: secret-name
##   optional: true

## The names of conifgmaps in the same kubernetes namespace which contain values to be added to the environment
## Each entry should contain a name key, and can optionally specify whether the configmap must be defined with an optional key.
## Name is templated.
## ref: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#configmapenvsource-v1-core
envFromConfigMaps: []
## - name: configmap-name
##   optional: true

# Inject Kubernetes services as environment variables.
# See https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/#environment-variables
enableServiceLinks: true

## Additional grafana server secret mounts
# Defines additional mounts with secrets. Secrets must be manually created in the namespace.
extraSecretMounts: []
  # - name: secret-files
  #   mountPath: /etc/secrets
  #   secretName: grafana-secret-files
  #   readOnly: true
  #   subPath: ""
  #
  # for AWS EKS (cloudwatch) use the following (see also instruction in env: above)
  # - name: aws-iam-token
  #   mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
  #   readOnly: true
  #   projected:
  #     defaultMode: 420
  #     sources:
  #       - serviceAccountToken:
  #           audience: sts.amazonaws.com
  #           expirationSeconds: 86400
  #           path: token
  #
  # for CSI e.g. Azure Key Vault use the following
  # - name: secrets-store-inline
  #  mountPath: /run/secrets
  #  readOnly: true
  #  csi:
  #    driver: secrets-store.csi.k8s.io
  #    readOnly: true
  #    volumeAttributes:
  #      secretProviderClass: "akv-grafana-spc"
  #    nodePublishSecretRef:                       # Only required when using service principal mode
  #       name: grafana-akv-creds                  # Only required when using service principal mode

## Additional grafana server volume mounts
# Defines additional volume mounts.
extraVolumeMounts: []
  # - name: extra-volume-0
  #   mountPath: /mnt/volume0
  #   readOnly: true
  #   existingClaim: volume-claim
  # - name: extra-volume-1
  #   mountPath: /mnt/volume1
  #   readOnly: true
  #   hostPath: /usr/shared/
  # - name: grafana-secrets
  #   csi: true
  #   data:
  #     driver: secrets-store.csi.k8s.io
  #     readOnly: true
  #     volumeAttributes:
  #       secretProviderClass: "grafana-env-spc"

## Container Lifecycle Hooks. Execute a specific bash command or make an HTTP request
lifecycleHooks: {}
  # postStart:
  #   exec:
  #     command: []

## Pass the plugins you want installed as a list.
##
plugins: []
  # - digrich-bubblechart-panel
  # - grafana-clock-panel
  ## You can also use other plugin download URL, as long as they are valid zip files,
  ## and specify the name of the plugin after the semicolon. Like this:
  # - https://grafana.com/api/plugins/marcusolsson-json-datasource/versions/1.3.2/download;marcusolsson-json-datasource

## Configure grafana datasources
## ref: http://docs.grafana.org/administration/provisioning/#datasources
##
datasources: {}
#  datasources.yaml:
#    apiVersion: 1
#    datasources:
#    - name: Prometheus
#      type: prometheus
#      url: http://prometheus-prometheus-server
#      access: proxy
#      isDefault: true
#    - name: CloudWatch
#      type: cloudwatch
#      access: proxy
#      uid: cloudwatch
#      editable: false
#      jsonData:
#        authType: default
#        defaultRegion: us-east-1

## Configure grafana alerting (can be templated)
## ref: http://docs.grafana.org/administration/provisioning/#alerting
##
alerting: {}
  # rules.yaml:
  #   apiVersion: 1
  #   groups:
  #     - orgId: 1
  #       name: '{{ .Chart.Name }}_my_rule_group'
  #       folder: my_first_folder
  #       interval: 60s
  #       rules:
  #         - uid: my_id_1
  #           title: my_first_rule
  #           condition: A
  #           data:
  #             - refId: A
  #               datasourceUid: '-100'
  #               model:
  #                 conditions:
  #                   - evaluator:
  #                       params:
  #                         - 3
  #                       type: gt
  #                     operator:
  #                       type: and
  #                     query:
  #                       params:
  #                         - A
  #                     reducer:
  #                       type: last
  #                     type: query
  #                 datasource:
  #                   type: __expr__
  #                   uid: '-100'
  #                 expression: 1==0
  #                 intervalMs: 1000
  #                 maxDataPoints: 43200
  #                 refId: A
  #                 type: math
  #           dashboardUid: my_dashboard
  #           panelId: 123
  #           noDataState: Alerting
  #           for: 60s
  #           annotations:
  #             some_key: some_value
  #           labels:
  #             team: sre_team_1
  # contactpoints.yaml:
  #   apiVersion: 1
  #   contactPoints:
  #     - orgId: 1
  #       name: cp_1
  #       receivers:
  #         - uid: first_uid
  #           type: pagerduty
  #           settings:
  #             integrationKey: XXX
  #             severity: critical
  #             class: ping failure
  #             component: Grafana
  #             group: app-stack
  #             summary: |
  #               {{ `{{ include "default.message" . }}` }}

## Configure notifiers
## ref: http://docs.grafana.org/administration/provisioning/#alert-notification-channels
##
notifiers: {}
#  notifiers.yaml:
#    notifiers:
#    - name: email-notifier
#      type: email
#      uid: email1
#      # either:
#      org_id: 1
#      # or
#      org_name: Main Org.
#      is_default: true
#      settings:
#        addresses: an_email_address@example.com
#    delete_notifiers:

## Configure grafana dashboard providers
## ref: http://docs.grafana.org/administration/provisioning/#dashboards
##
## `path` must be /var/lib/grafana/dashboards/<provider_name>
##
dashboardProviders: {}
#  dashboardproviders.yaml:
#    apiVersion: 1
#    providers:
#    - name: 'default'
#      orgId: 1
#      folder: ''
#      type: file
#      disableDeletion: false
#      editable: true
#      options:
#        path: /var/lib/grafana/dashboards/default

## Configure grafana dashboard to import
## NOTE: To use dashboards you must also enable/configure dashboardProviders
## ref: https://grafana.com/dashboards
##
## dashboards per provider, use provider name as key.
##
dashboards: {}
  # default:
  #   some-dashboard:
  #     json: |
  #       $RAW_JSON
  #   custom-dashboard:
  #     file: dashboards/custom-dashboard.json
  #   prometheus-stats:
  #     gnetId: 2
  #     revision: 2
  #     datasource: Prometheus
  #   local-dashboard:
  #     url: https://example.com/repository/test.json
  #     token: ''
  #   local-dashboard-base64:
  #     url: https://example.com/repository/test-b64.json
  #     token: ''
  #     b64content: true
  #   local-dashboard-gitlab:
  #     url: https://example.com/repository/test-gitlab.json
  #     gitlabToken: ''
  #   local-dashboard-bitbucket:
  #     url: https://example.com/repository/test-bitbucket.json
  #     bearerToken: ''
  #   local-dashboard-azure:
  #     url: https://example.com/repository/test-azure.json
  #     basic: ''
  #     acceptHeader: '*/*'

## Reference to external ConfigMap per provider. Use provider name as key and ConfigMap name as value.
## A provider dashboards must be defined either by external ConfigMaps or in values.yaml, not in both.
## ConfigMap data example:
##
## data:
##   example-dashboard.json: |
##     RAW_JSON
##
dashboardsConfigMaps: {}
#  default: ""

## Grafana's primary configuration
## NOTE: values in map will be converted to ini format
## ref: http://docs.grafana.org/installation/configuration/
##
grafana.ini:
  paths:
    data: /var/lib/grafana/
    logs: /var/log/grafana
    plugins: /var/lib/grafana/plugins
    provisioning: /etc/grafana/provisioning
  analytics:
    check_for_updates: true
  log:
    mode: console
  grafana_net:
    url: https://grafana.net
  server:
    domain: "{{ if (and .Values.ingress.enabled .Values.ingress.hosts) }}{{ .Values.ingress.hosts | first }}{{ else }}''{{ end }}"
## grafana Authentication can be enabled with the following values on grafana.ini
 # server:
      # The full public facing url you use in browser, used for redirects and emails
 #    root_url:
 # https://grafana.com/docs/grafana/latest/auth/github/#enable-github-in-grafana
 # auth.github:
 #    enabled: false
 #    allow_sign_up: false
 #    scopes: user:email,read:org
 #    auth_url: https://github.com/login/oauth/authorize
 #    token_url: https://github.com/login/oauth/access_token
 #    api_url: https://api.github.com/user
 #    team_ids:
 #    allowed_organizations:
 #    client_id:
 #    client_secret:
## LDAP Authentication can be enabled with the following values on grafana.ini
## NOTE: Grafana will fail to start if the value for ldap.toml is invalid
  # auth.ldap:
  #   enabled: true
  #   allow_sign_up: true
  #   config_file: /etc/grafana/ldap.toml

## Grafana's LDAP configuration
## Templated by the template in _helpers.tpl
## NOTE: To enable the grafana.ini must be configured with auth.ldap.enabled
## ref: http://docs.grafana.org/installation/configuration/#auth-ldap
## ref: http://docs.grafana.org/installation/ldap/#configuration
ldap:
  enabled: false
  # `existingSecret` is a reference to an existing secret containing the ldap configuration
  # for Grafana in a key `ldap-toml`.
  existingSecret: ""
  # `config` is the content of `ldap.toml` that will be stored in the created secret
  config: ""
  # config: |-
  #   verbose_logging = true

  #   [[servers]]
  #   host = "my-ldap-server"
  #   port = 636
  #   use_ssl = true
  #   start_tls = false
  #   ssl_skip_verify = false
  #   bind_dn = "uid=%s,ou=users,dc=myorg,dc=com"

## Grafana's SMTP configuration
## NOTE: To enable, grafana.ini must be configured with smtp.enabled
## ref: http://docs.grafana.org/installation/configuration/#smtp
smtp:
  # `existingSecret` is a reference to an existing secret containing the smtp configuration
  # for Grafana.
  existingSecret: ""
  userKey: "user"
  passwordKey: "password"

## Sidecars that collect the configmaps with specified label and stores the included files them into the respective folders
## Requires at least Grafana 5 to work and can't be used together with parameters dashboardProviders, datasources and dashboards
sidecar:
  image:
    repository: quay.io/kiwigrid/k8s-sidecar
    tag: 1.22.0
    sha: ""
  imagePullPolicy: IfNotPresent
  resources: {}
#   limits:
#     cpu: 100m
#     memory: 100Mi
#   requests:
#     cpu: 50m
#     memory: 50Mi
  securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop:
      - ALL
    seccompProfile:
      type: RuntimeDefault
  # skipTlsVerify Set to true to skip tls verification for kube api calls
  # skipTlsVerify: true
  enableUniqueFilenames: false
  readinessProbe: {}
  livenessProbe: {}
  # Log level default for all sidecars. Can be one of: DEBUG, INFO, WARN, ERROR, CRITICAL. Defaults to INFO
  # logLevel: INFO
  alerts:
    enabled: false
    # Additional environment variables for the alerts sidecar
    env: {}
    # Do not reprocess already processed unchanged resources on k8s API reconnect.
    # ignoreAlreadyProcessed: true
    # label that the configmaps with alert are marked with
    label: grafana_alert
    # value of label that the configmaps with alert are set to
    labelValue: ""
    # Log level. Can be one of: DEBUG, INFO, WARN, ERROR, CRITICAL.
    # logLevel: INFO
    # If specified, the sidecar will search for alert config-maps inside this namespace.
    # Otherwise the namespace in which the sidecar is running will be used.
    # It's also possible to specify ALL to search in all namespaces
    searchNamespace: null
    # Method to use to detect ConfigMap changes. With WATCH the sidecar will do a WATCH requests, with SLEEP it will list all ConfigMaps, then sleep for 60 seconds.
    watchMethod: WATCH
    # search in configmap, secret or both
    resource: both
    # watchServerTimeout: request to the server, asking it to cleanly close the connection after that.
    # defaults to 60sec; much higher values like 3600 seconds (1h) are feasible for non-Azure K8S
    # watchServerTimeout: 3600
    #
    # watchClientTimeout: is a client-side timeout, configuring your local socket.
    # If you have a network outage dropping all packets with no RST/FIN,
    # this is how long your client waits before realizing & dropping the connection.
    # defaults to 66sec (sic!)
    # watchClientTimeout: 60
    #
    # Endpoint to send request to reload alerts
    reloadURL: "http://localhost:3000/api/admin/provisioning/alerting/reload"
    # Absolute path to shell script to execute after a alert got reloaded
    script: null
    skipReload: false
    # Deploy the alert sidecar as an initContainer in addition to a container.
    # Sets the size limit of the alert sidecar emptyDir volume
    sizeLimit: {}
  dashboards:
    enabled: false
    # Additional environment variables for the dashboards sidecar
    env: {}
    # Do not reprocess already processed unchanged resources on k8s API reconnect.
    # ignoreAlreadyProcessed: true
    SCProvider: true
    # label that the configmaps with dashboards are marked with
    label: grafana_dashboard
    # value of label that the configmaps with dashboards are set to
    labelValue: ""
    # Log level. Can be one of: DEBUG, INFO, WARN, ERROR, CRITICAL.
    # logLevel: INFO
    # folder in the pod that should hold the collected dashboards (unless `defaultFolderName` is set)
    folder: /tmp/dashboards
    # The default folder name, it will create a subfolder under the `folder` and put dashboards in there instead
    defaultFolderName: null
    # Namespaces list. If specified, the sidecar will search for config-maps/secrets inside these namespaces.
    # Otherwise the namespace in which the sidecar is running will be used.
    # It's also possible to specify ALL to search in all namespaces.
    searchNamespace: null
    # Method to use to detect ConfigMap changes. With WATCH the sidecar will do a WATCH requests, with SLEEP it will list all ConfigMaps, then sleep for 60 seconds.
    watchMethod: WATCH
    # search in configmap, secret or both
    resource: both
    # If specified, the sidecar will look for annotation with this name to create folder and put graph here.
    # You can use this parameter together with `provider.foldersFromFilesStructure`to annotate configmaps and create folder structure.
    folderAnnotation: null
    # Endpoint to send request to reload alerts
    reloadURL: "http://localhost:3000/api/admin/provisioning/dashboards/reload"
    # Absolute path to shell script to execute after a configmap got reloaded
    script: null
    skipReload: false
    # watchServerTimeout: request to the server, asking it to cleanly close the connection after that.
    # defaults to 60sec; much higher values like 3600 seconds (1h) are feasible for non-Azure K8S
    # watchServerTimeout: 3600
    #
    # watchClientTimeout: is a client-side timeout, configuring your local socket.
    # If you have a network outage dropping all packets with no RST/FIN,
    # this is how long your client waits before realizing & dropping the connection.
    # defaults to 66sec (sic!)
    # watchClientTimeout: 60
    #
    # provider configuration that lets grafana manage the dashboards
    provider:
      # name of the provider, should be unique
      name: sidecarProvider
      # orgid as configured in grafana
      orgid: 1
      # folder in which the dashboards should be imported in grafana
      folder: ''
      # type of the provider
      type: file
      # disableDelete to activate a import-only behaviour
      disableDelete: false
      # allow updating provisioned dashboards from the UI
      allowUiUpdates: false
      # allow Grafana to replicate dashboard structure from filesystem
      foldersFromFilesStructure: false
    # Additional dashboard sidecar volume mounts
    extraMounts: []
    # Sets the size limit of the dashboard sidecar emptyDir volume
    sizeLimit: {}
  datasources:
    enabled: false
    # Additional environment variables for the datasourcessidecar
    env: {}
    # Do not reprocess already processed unchanged resources on k8s API reconnect.
    # ignoreAlreadyProcessed: true
    # label that the configmaps with datasources are marked with
    label: grafana_datasource
    # value of label that the configmaps with datasources are set to
    labelValue: ""
    # Log level. Can be one of: DEBUG, INFO, WARN, ERROR, CRITICAL.
    # logLevel: INFO
    # If specified, the sidecar will search for datasource config-maps inside this namespace.
    # Otherwise the namespace in which the sidecar is running will be used.
    # It's also possible to specify ALL to search in all namespaces
    searchNamespace: null
    # Method to use to detect ConfigMap changes. With WATCH the sidecar will do a WATCH requests, with SLEEP it will list all ConfigMaps, then sleep for 60 seconds.
    watchMethod: WATCH
    # search in configmap, secret or both
    resource: both
    # watchServerTimeout: request to the server, asking it to cleanly close the connection after that.
    # defaults to 60sec; much higher values like 3600 seconds (1h) are feasible for non-Azure K8S
    # watchServerTimeout: 3600
    #
    # watchClientTimeout: is a client-side timeout, configuring your local socket.
    # If you have a network outage dropping all packets with no RST/FIN,
    # this is how long your client waits before realizing & dropping the connection.
    # defaults to 66sec (sic!)
    # watchClientTimeout: 60
    #
    # Endpoint to send request to reload datasources
    reloadURL: "http://localhost:3000/api/admin/provisioning/datasources/reload"
    # Absolute path to shell script to execute after a datasource got reloaded
    script: null
    skipReload: false
    # Deploy the datasource sidecar as an initContainer in addition to a container.
    # This is needed if skipReload is true, to load any datasources defined at startup time.
    initDatasources: false
    # Sets the size limit of the datasource sidecar emptyDir volume
    sizeLimit: {}
  plugins:
    enabled: false
    # Additional environment variables for the plugins sidecar
    env: {}
    # Do not reprocess already processed unchanged resources on k8s API reconnect.
    # ignoreAlreadyProcessed: true
    # label that the configmaps with plugins are marked with
    label: grafana_plugin
    # value of label that the configmaps with plugins are set to
    labelValue: ""
    # Log level. Can be one of: DEBUG, INFO, WARN, ERROR, CRITICAL.
    # logLevel: INFO
    # If specified, the sidecar will search for plugin config-maps inside this namespace.
    # Otherwise the namespace in which the sidecar is running will be used.
    # It's also possible to specify ALL to search in all namespaces
    searchNamespace: null
    # Method to use to detect ConfigMap changes. With WATCH the sidecar will do a WATCH requests, with SLEEP it will list all ConfigMaps, then sleep for 60 seconds.
    watchMethod: WATCH
    # search in configmap, secret or both
    resource: both
    # watchServerTimeout: request to the server, asking it to cleanly close the connection after that.
    # defaults to 60sec; much higher values like 3600 seconds (1h) are feasible for non-Azure K8S
    # watchServerTimeout: 3600
    #
    # watchClientTimeout: is a client-side timeout, configuring your local socket.
    # If you have a network outage dropping all packets with no RST/FIN,
    # this is how long your client waits before realizing & dropping the connection.
    # defaults to 66sec (sic!)
    # watchClientTimeout: 60
    #
    # Endpoint to send request to reload plugins
    reloadURL: "http://localhost:3000/api/admin/provisioning/plugins/reload"
    # Absolute path to shell script to execute after a plugin got reloaded
    script: null
    skipReload: false
    # Deploy the datasource sidecar as an initContainer in addition to a container.
    # This is needed if skipReload is true, to load any plugins defined at startup time.
    initPlugins: false
    # Sets the size limit of the plugin sidecar emptyDir volume
    sizeLimit: {}
  notifiers:
    enabled: false
    # Additional environment variables for the notifierssidecar
    env: {}
    # Do not reprocess already processed unchanged resources on k8s API reconnect.
    # ignoreAlreadyProcessed: true
    # label that the configmaps with notifiers are marked with
    label: grafana_notifier
    # value of label that the configmaps with notifiers are set to
    labelValue: ""
    # Log level. Can be one of: DEBUG, INFO, WARN, ERROR, CRITICAL.
    # logLevel: INFO
    # If specified, the sidecar will search for notifier config-maps inside this namespace.
    # Otherwise the namespace in which the sidecar is running will be used.
    # It's also possible to specify ALL to search in all namespaces
    searchNamespace: null
    # Method to use to detect ConfigMap changes. With WATCH the sidecar will do a WATCH requests, with SLEEP it will list all ConfigMaps, then sleep for 60 seconds.
    watchMethod: WATCH
    # search in configmap, secret or both
    resource: both
    # watchServerTimeout: request to the server, asking it to cleanly close the connection after that.
    # defaults to 60sec; much higher values like 3600 seconds (1h) are feasible for non-Azure K8S
    # watchServerTimeout: 3600
    #
    # watchClientTimeout: is a client-side timeout, configuring your local socket.
    # If you have a network outage dropping all packets with no RST/FIN,
    # this is how long your client waits before realizing & dropping the connection.
    # defaults to 66sec (sic!)
    # watchClientTimeout: 60
    #
    # Endpoint to send request to reload notifiers
    reloadURL: "http://localhost:3000/api/admin/provisioning/notifications/reload"
    # Absolute path to shell script to execute after a notifier got reloaded
    script: null
    skipReload: false
    # Deploy the notifier sidecar as an initContainer in addition to a container.
    # This is needed if skipReload is true, to load any notifiers defined at startup time.
    initNotifiers: false
    # Sets the size limit of the notifier sidecar emptyDir volume
    sizeLimit: {}

## Override the deployment namespace
##
namespaceOverride: ""

## Number of old ReplicaSets to retain
##
revisionHistoryLimit: 10

## Add a seperate remote image renderer deployment/service
imageRenderer:
  deploymentStrategy: {}
  # Enable the image-renderer deployment & service
  enabled: false
  replicas: 1
  autoscaling:
    enabled: false
    minReplicas: 1
    maxReplicas: 5
    targetCPU: "60"
    targetMemory: ""
    behavior: {}
  image:
    # image-renderer Image repository
    repository: docker.io/grafana/grafana-image-renderer
    # image-renderer Image tag
    tag: latest
    # image-renderer Image sha (optional)
    sha: ""
    # image-renderer ImagePullPolicy
    pullPolicy: Always
  # extra environment variables
  env:
    HTTP_HOST: "0.0.0.0"
    # RENDERING_ARGS: --no-sandbox,--disable-gpu,--window-size=1280x758
    # RENDERING_MODE: clustered
    # IGNORE_HTTPS_ERRORS: true
  # image-renderer deployment serviceAccount
  serviceAccountName: ""
  # image-renderer deployment securityContext
  securityContext: {}
  # image-renderer deployment container securityContext
  containerSecurityContext:
    seccompProfile:
      type: RuntimeDefault
    capabilities:
      drop: ['ALL']
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
  # image-renderer deployment Host Aliases
  hostAliases: []
  # image-renderer deployment priority class
  priorityClassName: ''
  service:
    # Enable the image-renderer service
    enabled: true
    # image-renderer service port name
    portName: 'http'
    # image-renderer service port used by both service and deployment
    port: 8081
    targetPort: 8081
    # Adds the appProtocol field to the image-renderer service. This allows to work with istio protocol selection. Ex: "http" or "tcp"
    appProtocol: ""
  serviceMonitor:
    ## If true, a ServiceMonitor CRD is created for a prometheus operator
    ## https://github.com/coreos/prometheus-operator
    ##
    enabled: false
    path: /metrics
    #  namespace: monitoring  (defaults to use the namespace this chart is deployed to)
    labels: {}
    interval: 1m
    scheme: http
    tlsConfig: {}
    scrapeTimeout: 30s
    relabelings: []
    # See: https://doc.crds.dev/github.com/prometheus-operator/kube-prometheus/monitoring.coreos.com/ServiceMonitor/v1@v0.11.0#spec-targetLabels
    targetLabels: []
      # - targetLabel1
      # - targetLabel2
  # If https is enabled in Grafana, this needs to be set as 'https' to correctly configure the callback used in Grafana
  grafanaProtocol: http
  # In case a sub_path is used this needs to be added to the image renderer callback
  grafanaSubPath: ""
  # name of the image-renderer port on the pod
  podPortName: http
  # number of image-renderer replica sets to keep
  revisionHistoryLimit: 10
  networkPolicy:
    # Enable a NetworkPolicy to limit inbound traffic to only the created grafana pods
    limitIngress: true
    # Enable a NetworkPolicy to limit outbound traffic to only the created grafana pods
    limitEgress: false
    # Allow additional services to access image-renderer (eg. Prometheus operator when ServiceMonitor is enabled)
    extraIngressSelectors: []
  resources: {}
#   limits:
#     cpu: 100m
#     memory: 100Mi
#   requests:
#     cpu: 50m
#     memory: 50Mi
  ## Node labels for pod assignment
  ## ref: https://kubernetes.io/docs/user-guide/node-selection/
  #
  nodeSelector: {}

  ## Tolerations for pod assignment
  ## ref: https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/
  ##
  tolerations: []

  ## Affinity for pod assignment (evaluated as template)
  ## ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity
  ##
  affinity: {}

  ## Use an alternate scheduler, e.g. "stork".
  ## ref: https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/
  ##
  # schedulerName: "default-scheduler"

networkPolicy:
  ## @param networkPolicy.enabled Enable creation of NetworkPolicy resources. Only Ingress traffic is filtered for now.
  ##
  enabled: false
  ## @param networkPolicy.allowExternal Don't require client label for connections
  ## The Policy model to apply. When set to false, only pods with the correct
  ## client label will have network access to  grafana port defined.
  ## When true, grafana will accept connections from any source
  ## (with the correct destination port).
  ##
  ingress: true
  ## @param networkPolicy.ingress When true enables the creation
  ## an ingress network policy
  ##
  allowExternal: true
  ## @param networkPolicy.explicitNamespacesSelector A Kubernetes LabelSelector to explicitly select namespaces from which traffic could be allowed
  ## If explicitNamespacesSelector is missing or set to {}, only client Pods that are in the networkPolicy's namespace
  ## and that match other criteria, the ones that have the good label, can reach the grafana.
  ## But sometimes, we want the grafana to be accessible to clients from other namespaces, in this case, we can use this
  ## LabelSelector to select these namespaces, note that the networkPolicy's namespace should also be explicitly added.
  ##
  ## Example:
  ## explicitNamespacesSelector:
  ##   matchLabels:
  ##     role: frontend
  ##   matchExpressions:
  ##    - {key: role, operator: In, values: [frontend]}
  ##
  explicitNamespacesSelector: {}
  ##
  ##
  ##
  ##
  ##
  ##
  egress:
    ## @param networkPolicy.egress.enabled When enabled, an egress network policy will be
    ## created allowing grafana to connect to external data sources from kubernetes cluster.
    enabled: false
    ##
    ## @param networkPolicy.egress.ports Add individual ports to be allowed by the egress
    ports: []
    ## Add ports to the egress by specifying - port: <port number>
    ## E.X.
    ## ports:
      ## - port: 80
      ## - port: 443
  ##
  ##
  ##
  ##
  ##
  ##

# Enable backward compatibility of kubernetes where version below 1.13 doesn't have the enableServiceLinks option
enableKubeBackwardCompatibility: false
useStatefulSet: false
# Create a dynamic manifests via values:
extraObjects: []
  # - apiVersion: "kubernetes-client.io/v1"
  #   kind: ExternalSecret
  #   metadata:
  #     name: grafana-secrets
  #   spec:
  #     backendType: gcpSecretsManager
  #     data:
  #       - key: grafana-admin-password
  #         name: adminPassword
```
</details>

## 设置pvc

```bash
$ sudo vim grafana/values.yaml
...
persistence:
  type: pvc
  enabled: true             # 1. 改成true,  如果没有默认的存储类那么需要做第二步
  storageClassName: nas-sc  # 【自己选sc】2. 改成集群已经存在的storageClassName， 自动生成pvc
  # storageClassName: default
  accessModes:
    - ReadWriteOnce
  size: 10Gi
  # annotations: {}
  finalizers:
    - kubernetes.io/pvc-protection
  # selectorLabels: {}
  ## Sub-directory of the PV to mount. Can be templated.
  # subPath: ""
  ## Name of an existing PVC. Can be templated.
  # existingClaim:          # 【使用已有pvc】2. 如果有自己的PVC，去掉注释填上pvc的名称
  ## Extra labels to apply to a PVC.
  extraPvcLabels: {}

  ## If persistence is not enabled, this allows to mount the
  ## local storage in-memory to improve performance
  ##
  inMemory:
    enabled: false
    ## The maximum usage on memory medium EmptyDir would be
    ## the minimum value between the SizeLimit specified
    ## here and the sum of memory limits of all containers in a pod
    ##
    # sizeLimit: 300Mi
```

## 部署grafana

```bash
$ sudo helm install --create-namespace --namespace monitoring  grafana ./grafana --debug

NAME: grafana
LAST DEPLOYED: Mon May  8 11:01:51 2023
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
```

<details>
  <summary>展开查看debug信息</summary>

```yaml
install.go:194: [debug] Original chart version: ""
install.go:211: [debug] CHART PATH: /mnt/grafana
NAME: grafana
LAST DEPLOYED: Mon May  8 11:01:51 2023
NAMESPACE: monitoring
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
  enabled: true
  extraPvcLabels: {}
  finalizers:
  - kubernetes.io/pvc-protection
  inMemory:
    enabled: false
  size: 10Gi
  storageClassName: nas-sc
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
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
  name: grafana-test
  namespace: monitoring
  annotations:
    "helm.sh/hook": test-success
    "helm.sh/hook-delete-policy": "before-hook-creation,hook-succeeded"
---
# Source: grafana/templates/tests/test-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-test
  namespace: monitoring
  annotations:
    "helm.sh/hook": test-success
    "helm.sh/hook-delete-policy": "before-hook-creation,hook-succeeded"
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
data:
  run.sh: |-
    @test "Test Health" {
      url="http://grafana/api/health"

      code=$(wget --server-response --spider --timeout 90 --tries 10 ${url} 2>&1 | awk '/^  HTTP/{print $2}')
      [ "$code" == "200" ]
    }
---
# Source: grafana/templates/tests/test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: grafana-test
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
  annotations:
    "helm.sh/hook": test-success
    "helm.sh/hook-delete-policy": "before-hook-creation,hook-succeeded"
  namespace: monitoring
spec:
  serviceAccountName: grafana-test
  containers:
    - name: grafana-test
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
        name: grafana-test
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
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
  name: grafana
  namespace: monitoring
---
# Source: grafana/templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
type: Opaque
data:
  admin-user: "YWRtaW4="
  admin-password: "MTRaTGxWeXNLWHJrYnhyUW9DV3dYZjM5ZjlCQ3lsMU1RNmVMYVdnUw=="
  ldap-toml: ""
---
# Source: grafana/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
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
# Source: grafana/templates/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
  finalizers:
    - kubernetes.io/pvc-protection
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "10Gi"
  storageClassName: nas-sc
---
# Source: grafana/templates/clusterrole.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
  name: grafana-clusterrole
rules: []
---
# Source: grafana/templates/clusterrolebinding.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: grafana-clusterrolebinding
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
subjects:
  - kind: ServiceAccount
    name: grafana
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: grafana-clusterrole
  apiGroup: rbac.authorization.k8s.io
---
# Source: grafana/templates/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
rules: []
---
# Source: grafana/templates/rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: grafana
subjects:
- kind: ServiceAccount
  name: grafana
  namespace: monitoring
---
# Source: grafana/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
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
    app.kubernetes.io/instance: grafana
---
# Source: grafana/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/name: grafana
      app.kubernetes.io/instance: grafana
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app.kubernetes.io/name: grafana
        app.kubernetes.io/instance: grafana
      annotations:
        checksum/config: 19d2af365db97b8a27f9f902f631ebd5a0739c98c77e20c41f479c8789695773
        checksum/dashboards-json-config: 01ba4719c80b6fe911b091a7c05124b64eeece964e09c058ef8f9805daca546b
        checksum/sc-dashboard-provider-config: 01ba4719c80b6fe911b091a7c05124b64eeece964e09c058ef8f9805daca546b
        checksum/secret: e5bdf8d22f5bd1f9f151274a9673630e92a4dd2d29d4b390f0f0f61e466712c3
        kubectl.kubernetes.io/default-container: grafana
    spec:
      
      serviceAccountName: grafana
      automountServiceAccountToken: true
      securityContext:
        fsGroup: 472
        runAsGroup: 472
        runAsNonRoot: true
        runAsUser: 472
      initContainers:
        - name: init-chown-data
          image: "docker.io/library/busybox:1.31.1"
          imagePullPolicy: IfNotPresent
          securityContext:
            capabilities:
              add:
              - CHOWN
            runAsNonRoot: false
            runAsUser: 0
            seccompProfile:
              type: RuntimeDefault
          command:
            - chown
            - -R
            - 472:472
            - /var/lib/grafana
          volumeMounts:
            - name: storage
              mountPath: "/var/lib/grafana"
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
                  name: grafana
                  key: admin-user
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana
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
            name: grafana
        - name: storage
          persistentVolumeClaim:
            claimName: grafana

NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo


2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   grafana.monitoring.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:
     export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace monitoring port-forward $POD_NAME 3000

3. Login with the password from step 1 and the username: admin

```
</details>

## 登录grafana

```bash
#1. 通过如下做一下svc的映射： kubectl port-forward svc/[service-name] -n [namespace] [external-port]:[internal-port]
$ sudo kubectl port-forward svc/grafana -n monitoring 80:80 --address='0.0.0.0'

#2. 获取初始密码
NOTES:
Get your 'admin' user password by running:

   kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

# 3. 通过本地IP+80即可访问
```

## 查看manifest

```bash
$ sudo helm get manifest grafana --namespace monitoring
```

<details> 
  <summary>点击展开查看manifest</summary>

```yaml
---
# Source: grafana/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
  name: grafana
  namespace: monitoring
---
# Source: grafana/templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
type: Opaque
data:
  admin-user: "YWRtaW4="
  admin-password: "MTRaTGxWeXNLWHJrYnhyUW9DV3dYZjM5ZjlCQ3lsMU1RNmVMYVdnUw=="
  ldap-toml: ""
---
# Source: grafana/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
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
# Source: grafana/templates/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
  finalizers:
    - kubernetes.io/pvc-protection
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "10Gi"
  storageClassName: nas-sc
---
# Source: grafana/templates/clusterrole.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
  name: grafana-clusterrole
rules: []
---
# Source: grafana/templates/clusterrolebinding.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: grafana-clusterrolebinding
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
subjects:
  - kind: ServiceAccount
    name: grafana
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: grafana-clusterrole
  apiGroup: rbac.authorization.k8s.io
---
# Source: grafana/templates/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
rules: []
---
# Source: grafana/templates/rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: grafana
subjects:
- kind: ServiceAccount
  name: grafana
  namespace: monitoring
---
# Source: grafana/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
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
    app.kubernetes.io/instance: grafana
---
# Source: grafana/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
  labels:
    helm.sh/chart: grafana-6.56.1
    app.kubernetes.io/name: grafana
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/version: "9.5.1"
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/name: grafana
      app.kubernetes.io/instance: grafana
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app.kubernetes.io/name: grafana
        app.kubernetes.io/instance: grafana
      annotations:
        checksum/config: 19d2af365db97b8a27f9f902f631ebd5a0739c98c77e20c41f479c8789695773
        checksum/dashboards-json-config: 01ba4719c80b6fe911b091a7c05124b64eeece964e09c058ef8f9805daca546b
        checksum/sc-dashboard-provider-config: 01ba4719c80b6fe911b091a7c05124b64eeece964e09c058ef8f9805daca546b
        checksum/secret: e5bdf8d22f5bd1f9f151274a9673630e92a4dd2d29d4b390f0f0f61e466712c3
        kubectl.kubernetes.io/default-container: grafana
    spec:

      serviceAccountName: grafana
      automountServiceAccountToken: true
      securityContext:
        fsGroup: 472
        runAsGroup: 472
        runAsNonRoot: true
        runAsUser: 472
      initContainers:
        - name: init-chown-data
          image: "docker.io/library/busybox:1.31.1"
          imagePullPolicy: IfNotPresent
          securityContext:
            capabilities:
              add:
              - CHOWN
            runAsNonRoot: false
            runAsUser: 0
            seccompProfile:
              type: RuntimeDefault
          command:
            - chown
            - -R
            - 472:472
            - /var/lib/grafana
          volumeMounts:
            - name: storage
              mountPath: "/var/lib/grafana"
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
                  name: grafana
                  key: admin-user
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana
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
            name: grafana
        - name: storage
          persistentVolumeClaim:
            claimName: grafana
```
</details>