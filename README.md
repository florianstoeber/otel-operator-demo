# otel-operator-demo
## 0. ToDos
Not working: 
- Metrics exporting to Dynatrace
- Instrument the missing services
  - missing:
    - cart
    - checkout
    - frontend
    - productcatalog
    - shipping
## 1. Requirements
- Kubernetes Cluster
- Destination to send data (Demo is using Dynatrace)
### 1.1 Create a Kuberentes cluster
As we need a Kubernetes cluster, we will initiliaze a GKE cluster with enabled cluster-autoscaler. In case you have no access to a Google Cloud project, feel free to adapt this to another cloud provider. The following steps are cloud-agnostic:
```bash
gcloud container clusters create otel-operator-demo --enable-autoscaling --total-min-nodes=3 --total-max-nodes=10 --zone us-central1-c
gcloud container clusters get-credentials otel-operator-demo --zone us-central1-c --project $PROJECT_ID
```
### 1.2 Install webapplication to use in this demo
```bash
helm upgrade onlineboutique -n boutique --create-namespace oci://us-docker.pkg.dev/online-boutique-ci/charts/onlineboutique \
    --install
```
## 2. Setup the OpenTelemetry Operator
At first we will install a cert-manager as this is a dependeny for the OpenTelemtry Operator. If you already have one on your cluster, you can utilize that one.
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
```
Next we can install the Operator with the official Helm-Chart
```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
kubectl create ns opentelemetry-operator-system
helm install --namespace opentelemetry-operator-system opentelemetry-operator open-telemetry/opentelemetry-operator
```

Now we will have two new CRDs. Both of them are Namespaced, so that teams can install these resources in their namespaces.
- opentelemetrycollectors.opentelemetry.io  
- instrumentations.opentelemetry.io

## 3. Create a set of collectors
We will utilize a combination of two different collector types. It is possible to deploy collectors in 4 modes:
- Deployment 
- Daemonset
- Statefulset
- Sidecar

We will use on the hand the sidecar to automatically inject a collector container in each Pod. This will lead to an easily adaptable collector for our users on the clusters, as they just have to send to a specific port on localhost. This sidecar will then send all the metrics to a collector which is deployed as a Deployment in the same Namespace. So it is possible that all teams configure their own collectors and the team which is operating the cluster does not need to handle this.

At first we will define a sidecar-collector:
```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-sidecar
  namespace: boutique
spec:
  mode: sidecar
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    processors:
      batch:

    exporters:
      logging:
      otlp:
        endpoint: "http://otel-gateway-collector:4317"
        tls:
          insecure: true

    service:
      telemetry:
        logs:
          level: "debug"
      pipelines:
        traces: 
          receivers: [otlp]
          processors: []
          exporters: [logging, otlp]
        metrics:
          receivers: [otlp]
          processors: []
          exporters: [logging, otlp]
        logs:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlp]
EOF
```

Additionally we need a Gateway in the same Namespace to send the data to. We will apply it with this manifest:
```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-gateway
  namespace: boutique
  labels:
    app: opentelemetry
    component: otel-collector
spec:
  mode: deployment
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    processors:
      batch:

    exporters:
      logging:
      # OTLPHTTP used for Dynatrace
      # otlphttp:
      #   endpoint: "https://$tenant.live.dynatrace.com/api/v2/otlp"
      #   headers: {"Authorization": "Api-Token $token"}
      # OTLP used for other Backends. This example is NewRelic
      otlp:
        endpoint: https://otlp.eu01.nr-data.net:4317
        headers:
          api-key: $NEWRELIC_LICENSEKEY

    service:
      telemetry:
        logs:
          level: "debug"
      pipelines:
        traces: 
          receivers: [otlp]
          processors: []
          #exporters: [logging, otlphttp]
          exporters: [logging, otlp]
        metrics:
          receivers: [otlp]
          processors: []
          #exporters: [logging, otlphttp]
          exporters: [logging, otlp]
        logs:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlp]
EOF
```

### 4. Create an Instrumentation for our applications
Next we will have to create an Instrumentation. These objects are used to add OpenTelemetry capabilities to Pods, that have not implemented the capabilities in code. So we will inject tehe capabilities while running the Pod.
```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: instrumentation
  namespace: boutique
spec:
  resource:
    addK8sUIDAttributes: true
  propagators:
  - tracecontext
  - baggage
  - b3
EOF
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: instrumentation-python
  namespace: boutique
spec:
  resource:
    addK8sUIDAttributes: true
  propagators:
  - tracecontext
  - baggage
EOF
```

### 5. Adding annotations to the services we want to observe
Add the two following annotations to the Pod Mannifest in the emailservice, recommendationservice Deployment:
```yaml
instrumentation.opentelemetry.io/inject-python: instrumentation-python  # This will add the defined instrumentation so that Python code can be instrumented
sidecar.opentelemetry.io/inject: otel-sidecar # This will add the sidecar collector we defined before
```
Add the two following annotations to the Pod Mannifest in the paymentservice, currencyservice Deployment:
```yaml
instrumentation.opentelemetry.io/inject-nodejs: instrumentation  # This will add the defined instrumentation so that NodeJS code can be instrumented
sidecar.opentelemetry.io/inject: otel-sidecar # This will add the sidecar collector we defined before
```

Add the two following annotations to the Pod Mannifest in the adservice Deployment:
```yaml
instrumentation.opentelemetry.io/inject-java: instrumentation  # This will add the defined instrumentation so that Java code can be instrumented
sidecar.opentelemetry.io/inject: otel-sidecar # This will add the sidecar collector we defined before
```
