# Dapr PubSub with Dynatrace OpenTelemetry Tracing

## Prerequisites

- Set up the [PubSub quickstart for Kubernetes](https://github.com/dapr/quickstarts/tree/master/tutorials/pub-sub#run-in-kubernetes)
- Access to a Dynatrace tenant and an API token with `openTelemetryTrace.ingest`, `metrics.ingest`, and `logs.ingest` scopes

## Set up Dynatrace OpenTelemetry Collector

1. **Create Kubernetes secret with Dynatrace credentials**

   ```bash
   kubectl create secret generic dynatrace-otelcol-dt-api-credentials \
     --from-literal=DT_ENDPOINT=https://YOUR_TENANT.live.dynatrace.com/api/v2/otlp \
     --from-literal=DT_API_TOKEN=dt0s01.YOUR_TOKEN_HERE
   ```

   Replace `YOUR_TENANT` with your Dynatrace tenant ID and `YOUR_TOKEN_HERE` with your Dynatrace API token.

2. **Create required namespaces**

   ```bash
   kubectl create namespace dynatrace
   kubectl create namespace otel-demo
   ```

3. **Deploy Dynatrace Operator (if using DynaKube)**

   ```bash
   kubectl apply -f ./dynatrace/dynakube.yaml    
   ```

4. **Deploy OpenTelemetry Collector**

   ```bash
   helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
   helm repo update
   helm upgrade -i dynatrace-collector-gateway open-telemetry/opentelemetry-collector \
     -f ./dynatrace/collector-agent.yaml \
     --namespace dynatrace
   ```

5. **Configure Dapr tracing**

   The Dapr configuration is already set up in `appconfig.yaml` to send traces to the Dynatrace OpenTelemetry Collector:

   ```yaml
   apiVersion: dapr.io/v1alpha1
   kind: Configuration
   metadata:
     name: appconfig
   spec:
     features:
     - enabled: true
       name: HotReload
     tracing:
       otel:
         endpointAddress: "dynatrace-collector-opentelemetry-collector.dynatrace.svc.cluster.local:4318"
         isSecure: false
         protocol: "http"
       samplingRate: "1"
   ```

   Apply the configuration:

   ```bash
   kubectl apply -f appconfig.yaml
   ```

6. **Deploy PubSub components**

   ```bash
   # Deploy PubSub component
   kubectl apply -f pubsub.yaml
   
   # Deploy Redis state store  
   kubectl apply -f redis.yaml
   ```

7. **Deploy application components**

   ```bash
   # Deploy all subscribers
   kubectl apply -f csharp-subscriber.yaml
   kubectl apply -f node-subscriber.yaml
   kubectl apply -f python-subscriber.yaml
   
   # Deploy React form (publisher)
   kubectl apply -f react-form.yaml
   ```

## Verify deployment

Check that all pods are running:

```bash
kubectl get pods
```

Access the React form locally:

```bash
kubectl port-forward service/react-form 8000:80
```

Then open `http://localhost:8000` in your browser to access the form.

## View traces in Dynatrace

1. Navigate to your Dynatrace tenant
2. Go to **Distributed traces**
3. Look for traces from your Dapr applications
4. The traces should show the full request flow from the React form through the PubSub system to all subscribers

## Scope and Limitations

**This example covers only distributed tracing.** For a complete observability setup with logs, metrics, and traces, you'll need additional configuration.

## Complete Observability with Dynatrace

For full observability beyond just traces, refer to these resources:

### Logs
- **[Direct log ingestion](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/collector/use-cases/kubernetes/k8s-podlogs)** - Kubernetes pod logs via OpenTelemetry Collector
- **[IsItObservable Dapr example](https://github.com/isItObservable/Dapr)** - Complete logs/metrics/tracing setup with collectors

### Metrics  
- **[Prometheus configuration](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/collector/use-cases/prometheus)** - Scraping Prometheus metrics via collector
- **[Dapr Prometheus config](https://docs.dapr.io/operations/observability/metrics/metrics-overview/#prometheus-endpoint)** - Enabling Dapr metrics endpoint

### Advanced Integration
- **[Dynatrace Collector documentation](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/collector)** - Complete collector capabilities
- **[OneAgent with sidecar technologies](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/integrations/envoy)** - Guidance for other sidecar deployments

## Troubleshooting

- Ensure the OpenTelemetry Collector service name matches the `endpointAddress` in `appconfig.yaml`
- Check collector logs: `kubectl logs -l app.kubernetes.io/name=opentelemetry-collector -n dynatrace`
- Verify Dapr sidecar logs: `kubectl logs <pod-name> -c daprd`
