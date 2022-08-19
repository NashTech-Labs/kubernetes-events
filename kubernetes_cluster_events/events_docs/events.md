# Events

## Why we need events

Events are the first thing to look at for application, as well as infrastructure operations when something is not working as expected. Keeping them for a longer period is essential if the failure is the result of earlier events, or when conducting post-mortem analysis.

## Kubernetes Events

### **Export events with a kube-event-exporter**

**1. Service Account**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: monitoring
  name: event-exporter
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: event-exporter
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    namespace: monitoring
    name: event-exporter
```

**2. configmap**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: event-exporter-cfg
  namespace: monitoring
data:
  config.yaml: |
    logLevel: error
    logFormat: json
    route:
        # Main route
      routes:
        - match:
            - receiver: "dump"
    receivers:
      - name: "dump"
        stdout: {}
```

**3. Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-exporter
  namespace: monitoring
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: event-exporter
        version: v1
    spec:
      serviceAccountName: event-exporter
      containers:
        - name: event-exporter
          image: ghcr.io/opsgenie/kubernetes-event-exporter:v0.11
          imagePullPolicy: IfNotPresent
          args:
            - -conf=/data/config.yaml
          volumeMounts:
            - mountPath: /data
              name: cfg
      volumes:
        - name: cfg
          configMap:
            name: event-exporter-cfg
  selector:
    matchLabels:
      app: event-exporter
      version: v1
```


## Configuration

Configuration is done via a YAML file, when run in Kubernetes, ConfigMap. The tool watches all the events and
user has to option to filter out some events, according to their properties. Critical events can be routed to alerting
tools such as Opsgenie, or all events can be dumped to an Elasticsearch instance. You can use namespaces, labels on the
related object to route some Pod related events to owners via Slack. The final routing is a tree which allows
flexibility. It generally looks like following:

```yaml
route:
  # Main route
  routes:
    # This route allows dumping all events because it has no fields to match and no drop rules.
    - match:
        - receiver: dump
    # This starts another route, drops all the events in *test* namespaces and Normal events
    # for capturing critical events
    - drop:
        - namespace: "*test*"
        - type: "Normal"
      match:
        - receiver: "critical-events-queue"
    # This a final route for user messages
    - match:
        - kind: "Pod|Deployment|ReplicaSet"
          labels:
            version: "dev"
          receiver: "slack"
receivers:
# See below for configuring the receivers
```

* A `match` rule is exclusive, all conditions must be matched to the event.
* During processing a route, `drop` rules are executed first to filter out events.
* The `match` rules in a route are independent of each other. If an event matches a rule, it goes down it's subtree.
* If all the `match` rules are matched, the event is passed to the `receiver`.
* A route can have many sub-routes, forming a tree.
* Routing starts from the root route.



[JSON response](./events.json)

```
{"metadata":{"name":"event-exporter-d88d4f8f4-t4fb4.170510a7bf1b7667","namespace":"monitoring","uid":"c3922661-a3b0-4a6b-a5ea-7d85cd2e37c0","resourceVersion":"299648","creationTimestamp":"2022-07-25T11:59:10Z"},"reason":"Created","message":"Created container event-exporter","source":{"component":"kubelet","host":"minikube"},"firstTimestamp":"2022-07-25T11:59:10Z","lastTimestamp":"2022-07-25T11:59:10Z","count":1,"type":"Normal","eventTime":null,"reportingComponent":"","reportingInstance":"","involvedObject":{"kind":"Pod","namespace":"monitoring","name":"event-exporter-d88d4f8f4-t4fb4","uid":"f462458e-e479-4a67-9ffd-29b45a99c8ad","apiVersion":"v1","resourceVersion":"299621","fieldPath":"spec.containers{event-exporter}","labels":{"app":"event-exporter","pod-template-hash":"d88d4f8f4","version":"v1"}}}
{"metadata":{"name":"event-exporter-d88d4f8f4-t4fb4.170510a7a0dda3b7","namespace":"monitoring","uid":"ddfa9690-5b52-4a49-859a-a8dd9453e302","resourceVersion":"299647","creationTimestamp":"2022-07-25T11:59:09Z"},"reason":"Pulled","message":"Successfully pulled image \"ghcr.io/opsgenie/kubernetes-event-exporter:v0.11\" in 26.622057233s","source":{"component":"kubelet","host":"minikube"},"firstTimestamp":"2022-07-25T11:59:09Z","lastTimestamp":"2022-07-25T11:59:09Z","count":1,"type":"Normal","eventTime":null,"reportingComponent":"","reportingInstance":"","involvedObject":{"kind":"Pod","namespace":"monitoring","name":"event-exporter-d88d4f8f4-t4fb4","uid":"f462458e-e479-4a67-9ffd-29b45a99c8ad","apiVersion":"v1","resourceVersion":"299621","fieldPath":"spec.containers{event-exporter}","labels":{"app":"event-exporter","pod-template-hash":"d88d4f8f4","version":"v1"}}}
{"metadata":{"name":"event-exporter-d88d4f8f4-t4fb4.170510a7e1a437fa","namespace":"monitoring","uid":"bdc5e14d-5982-4e1c-8374-f6c790f8a471","resourceVersion":"299649","creationTimestamp":"2022-07-25T11:59:10Z"},"reason":"Started","message":"Started container event-exporter","source":{"component":"kubelet","host":"minikube"},"firstTimestamp":"2022-07-25T11:59:10Z","lastTimestamp":"2022-07-25T11:59:10Z","count":1,"type":"Normal","eventTime":null,"reportingComponent":"","reportingInstance":"","involvedObject":{"kind":"Pod","namespace":"monitoring","name":"event-exporter-d88d4f8f4-t4fb4","uid":"f462458e-e479-4a67-9ffd-29b45a99c8ad","apiVersion":"v1","resourceVersion":"299621","fieldPath":"spec.containers{event-exporter}","labels":{"app":"event-exporter","pod-template-hash":"d88d4f8f4","version":"v1"}}}

 ~/Desktop/knolcicd/kubernetes_cluster_events/event_exportern | doc/events ?1                                                                  minikube/monitoring kube 
> k get events -A
NAMESPACE    LAST SEEN   TYPE     REASON              OBJECT                                MESSAGE
monitoring   110s        Normal   Scheduled           pod/event-exporter-d88d4f8f4-t4fb4    Successfully assigned monitoring/event-exporter-d88d4f8f4-t4fb4 to minikube
monitoring   107s        Normal   Pulling             pod/event-exporter-d88d4f8f4-t4fb4    Pulling image "ghcr.io/opsgenie/kubernetes-event-exporter:v0.11"
monitoring   81s         Normal   Pulled              pod/event-exporter-d88d4f8f4-t4fb4    Successfully pulled image "ghcr.io/opsgenie/kubernetes-event-exporter:v0.11" in 26.622057233s
monitoring   80s         Normal   Created             pod/event-exporter-d88d4f8f4-t4fb4    Created container event-exporter
monitoring   80s         Normal   Started             pod/event-exporter-d88d4f8f4-t4fb4    Started container event-exporter
monitoring   110s        Normal   SuccessfulCreate    replicaset/event-exporter-d88d4f8f4   Created pod: event-exporter-d88d4f8f4-t4fb4
monitoring   110s        Normal   ScalingReplicaSet   deployment/event-exporter             Scaled up replica set event-exporter-d88d4f8f4 to 1
```