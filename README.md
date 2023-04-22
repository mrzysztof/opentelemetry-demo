# OpenTele-17 - OpenTelemetry Case Study

Authors: Szymon Musiał, Paweł Kruczkiewicz, Marcin Kroczek, Krzysztof Maliszewski

2023, Implementation of Network Services

## Table of Contents
1. [Introduction](#introduction)
2. [Theoretical background](#theoretical-background)
3. [Case study concept description](#case-study-concept-description)
4. [Solution architecture](#solution-architecture)
5. [Environment configuration](#environment-configuration)
6. [Installation method](#installation-method)
7. [How to reproduce](#how-to-reproduce)
8. [Demo deployment](#demo-deployment)
9. [Summary](#summary)
10. [References](#references)
    - [Helm charts repo](#helm-charts-repo)

## Introduction
This project serves as a case study of observability using OpenTelemetry on an existing EmotoAGH application.
## Theoretical background

[OpenTelemetry](https://opentelemetry.io/) is is a vendor-neutral open-source [Observability](https://opentelemetry.io/docs/concepts/observability-primer/#what-is-observability) framework for instrumenting, generating, collecting, and exporting telemetry data such as traces, metrics, logs. OpenTelemetry’s goal is to provide a set of standardized vendor-agnostic SDKs, APIs, and tools for ingesting, transforming, and sending data to an Observability back-end (e.g. Jaeger,  Zipkin).

OpenTelemetry facilitates whole application observability, including:
### [Instrumenting](https://opentelemetry.io/docs/concepts/instrumenting/)
In order to make a system observable, code from the system’s components must emit traces, metrics, and logs. OTel provides two types of instrumentation:
- Automatic Instrumentation, that do not require the end-user to modify application’s source code,
- Manual Instrumentation, by coding against the OpenTelemetry API to collect telemetry from end-user

### [Data Collection](https://opentelemetry.io/docs/concepts/data-collection/)
The OpenTelemetry project facilitates the collection of telemetry data via the OpenTelemetry Collector. Collector offers a vendor-agnostic implementation on how to receive, process, and export telemetry data.
![Collector diagram](https://opentelemetry.io/docs/collector/img/otel-collector.svg)

## Case study concept description

The proposed demonstration of using OpenTelemetry will consist of using a set of these tools in the EmotoAGH project's web application. The mentioned application is responsible for:

- Collecting measurements from motorcycles created in the project,
- Aggregating, sorting, filtering measurements,
- Securing access based on the RBAC concept,
- A sufficiently universal API used by the website and another team of mobile applications.


It is publicly available, and its availability options are based on the user roles declared.
The application is available on bare-metal Kubernetes on Oracle OCI machines. We plan to use the mentioned project to observe its behaviors, events, logs, metrics and potential errors by deploying appropriate OpenTelemetry tools.


## Solution architecture

```mermaid
flowchart LR

App>Web Application]
Nginx["Nginx
ingress controller"]

K8["Kubernetes cluster"]

Prom[("Prometheus
operator
(metrics)")]

Tempo[("Grafana Tempo
(traces)")]

Loki[("Grafana Loki
(logs)")]

Grafana["Grafana
dashboards
(Web UI)"]

subgraph Otlp[Opentelemetry]
    direction TB
    o(Operator) --- c(Collector)
end

K8 <--->|K8 operator| Otlp
K8 <--->|K8 operator| Prom
K8 ----->|Promtail| Loki

Otlp --->|"Prometheus
remote write"| Prom
Otlp --->|Otlp| Tempo
Otlp -->|Api| Loki

Prom ---> Grafana
Tempo ---> Grafana
Loki ---> Grafana

App -->|Otlp| Otlp
Nginx -->|Jaeger| Otlp
```
One can notice the use Kubernetes operators. They are a way to automate the management of complex applications on Kubernetes clusters. An Operator is essentially a piece of software that uses Kubernetes APIs and custom resources to manage applications and infrastructure in a more intelligent way than traditional automation tools. Operators can be thought of as an extension of Kubernetes controllers, which are responsible for monitoring the state of Kubernetes objects and taking actions to bring them into the desired state. However, Operators are designed to handle more complex tasks than controllers, such as deploying and scaling stateful applications, managing databases, and handling backup and restore operations. Operators use custom resources to represent and manage the application or infrastructure they are responsible for. Custom resources are extensions of the Kubernetes API that can be used to define new types of objects that are specific to an application or infrastructure. These custom resources can be used to define the desired state of the application or infrastructure and provide a way for the Operator to monitor and manage it.

## Environment configuration

The configuration of the environment depends on the location where the applications are deployed. This can be a local version of Kubernetes, such as [k3d](https://k3d.io) or [kind](https://kind.sigs.k8s.io/). In our case, a ready-made cluster was used, which was configured using kubespray.


The configuration of the observed application involves providing the appropriate address, which is the endpoint of the OpenTelemetry Receiver.

In our case web application contains code shown below

```c#
public static IServiceCollection AddAppOpenTelemetryMetricsAndTracing(this IServiceCollection services, AppConfiguration appConfiguration)
{
    if (appConfiguration.PanelEmotoAgh_OtlpExporterEndpoint is null)
    {
        return services;
    }
    var resourceBuilder = CreateAppOtelResource(appConfiguration);

    services.AddOpenTelemetry()
        .WithMetrics(metricsProviderBuilder => metricsProviderBuilder
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation()
            .AddRuntimeInstrumentation()
            .AddOtlpExporter(e => e.Endpoint = new Uri(appConfiguration.PanelEmotoAgh_OtlpExporterEndpoint)))
        .WithTracing(tracerProviderBuilder => tracerProviderBuilder
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation()
            .AddEntityFrameworkCoreInstrumentation(o =>
            {
                o.SetDbStatementForText = true;
                o.SetDbStatementForStoredProcedure = true;
            })
            .AddOtlpExporter(e => e.Endpoint = new Uri(appConfiguration.PanelEmotoAgh_OtlpExporterEndpoint)));

    return services;
}

public static ILoggingBuilder AddAppOpenTelemetryLogs(this ILoggingBuilder loggingBuilder, AppConfiguration appConfiguration)
{
    if (appConfiguration.PanelEmotoAgh_OtlpExporterEndpoint is null)
    {
        return loggingBuilder;
    }

    return loggingBuilder.AddOpenTelemetry(options => options
        .SetResourceBuilder(CreateAppOtelResource(appConfiguration))
        .AddOtlpExporter(o => o.Endpoint = new Uri(appConfiguration.PanelEmotoAgh_OtlpExporterEndpoint))
    );
}

static internal ResourceBuilder CreateAppOtelResource(AppConfiguration appConfiguration)
{
    var resourceBuilder = ResourceBuilder.CreateDefault();
    resourceBuilder.AddService(
       serviceName: appConfiguration.PanelEmotoAgh_OtlpServiceName,
       serviceInstanceId: Environment.MachineName); // Pod name in k8 cluster
    return resourceBuilder;
}
```
This extension is used at application starting point

```c#
builder.Services.AddAppOpenTelemetryMetricsAndTracing(appConfiguration);
builder.Logging.AddAppOpenTelemetryLogs(appConfiguration);
```

Web application use global error handling so every exception will be passed via this part of code, what is responsible for append exception information into trace

```c#
internal static void RecordExceptionToTrace(Exception? exception)
{
    var activity = Activity.Current;

    if (activity is null && exception is null)
    {
        return;
    }

    // Add App domain error codes
    if (exception is AppException appException)
    {
        activity?.AddTag(nameof(AppException.AppErrorCodes), string.Join(", ", appException.AppErrorCodes));
    }

    while (activity != null)
    {
        activity.RecordException(exception);
        activity.Dispose();
        activity = activity.Parent;
    }
}
```

## Installation method

In the example, we were observing a running application. To carry out the demo, adding the appropriate deployments is required. Therefore, on the computer that will be configuring the cluster, installing and configuring kubectl, as well as the helm deployment repositories, is necessary.

Helm is a package manager for Kubernetes, which simplifies the deployment and management of complex applications on Kubernetes clusters. Helm Charts are packages of pre-configured Kubernetes resources, such as deployments, services, and ingress rules, that can be easily installed on a Kubernetes cluster using Helm. Helm Charts allow developers and DevOps teams to package their applications and infrastructure as a single unit, making it easier to share, deploy, and manage these resources across multiple environments. They provide a simple, repeatable, and automated way to deploy applications and infrastructure on Kubernetes, enabling faster and more reliable software delivery. Helm Charts also come with versioning and rollback features, which allow you to track changes to your application and infrastructure over time and revert to a previous version if needed. This makes it easier to manage and maintain the state of your Kubernetes cluster and ensure that your applications are running smoothly.

## How to reproduce

Perform deployment explained below.

## Demo deployment

   
First we should add required [repositories](#helm-charts-repo)

```sh
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Finally, we should have similar output
```sh
$ helm repo list
NAME            URL
prometheus      https://prometheus-community.github.io/helm-charts
open-telemetry  https://open-telemetry.github.io/opentelemetry-helm-charts
grafana         https://grafana.github.io/helm-charts
```

## Prometheus deployment

```sh
helm upgrade \
    --debug \
    --install kube-prometheus-stack \
    prometheus/kube-prometheus-stack \
    -f kube_prometheus_stack.yaml \
    --namespace observability \
    --create-namespace
```
Used chart values: [`kube_prometheus_stack.yaml`](./kube_prometheus_stack.yaml)

```yaml
prometheus:
  prometheusSpec:
    enableRemoteWriteReceiver: true

grafana:
  enabled: false
```

The Prometheus operator leverages the ability to read the cluster's configuration to obtain information about the installed components and gather metrics from them.

Also we enabled `enableRemoteWriteReceiver` to allow sending metrics from OpenTelemetry exporter to prometheus. It's push based approach.

### Grafana Tempo 

To save traces we deploy Grafana Tempo using monolithic mode. OpenTelemetry Collector is High availability (Deamon set), so we don't need microservice mode.

[In config file](./kube_prometheus_stack.yaml) we use remote write to send metrics to Prometheus to see ex. error spans in grafana dashboard and navigate to span which generated it.

```sh
helm upgrade \
    --install tempo \
    grafana/tempo \
    -f grafana_tempo.yaml \
    --namespace observability \
    --create-namespace
```

Used chart values: [`grafana_tempo.yaml`](./grafana_tempo.yaml)

```yaml
storage:
  trace:
    backend: local # can be AWS s3
    # ...

tempo:
  metricsGenerator:
    enabled: true
    remoteWriteUrl: http://kube-prometheus-stack-prometheus.observability.svc:9090/api/v1/write

tempoQuery:
  enabled: false
```

### Grafana Tempo 

To save logs we deploy Grafana Loki using monolithic mode. Loki deploy also provide Promtail, this is an agent which ships the contents of local logs to a Grafana Loki instance

```sh
helm upgrade \
   --install loki \
  grafana/loki-stack \
  -f ./grafana_loki.yaml \
  --namespace observability \
  --create-namespace
```

Used chart values: [`grafana_loki.yaml`](./grafana_loki.yaml)

```yaml
loki:
  commonConfig:
    replication_factor: 1
  storage:
    type: 'filesystem'
singleBinary:
  replicas: 1
```
### Grafana dashboards

We don't use the deployment of Grafana dashboards in the Prometheus stack because this deployment has fewer configuration options. The deployment of dashboards is done independently.

```sh
 helm upgrade \
    --install grafana  \
    grafana/grafana  \
    -f ./grafana_dashboards.yaml \
    --namespace observability \
    --create-namespace
```

Used chart values: [`grafana_dashboards.yaml`](./grafana_dashboards.yaml)

The mentioned Helm values include the definition of an ingress along with a certificate and the configuration of data sources that were deployed earlier.

```yaml
ingress:
  enabled: true
  ....
datasources:
  datasources.yaml:
   apiVersion: 1
   datasources:
    - name: Prometheus
      ....
    - name: Tempo
      ....
    - name: Loki
      ....
```

### OpenTelemetry

Next we should have installed Cert manager, the OpenTelemetry operator dependency

```sh
wget https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
kubectl apply -f ./cert-manager.yaml
```

Now we add OpenTelemetry Operator deployment
```sh
helm upgrade \
    --install opentelemetry-operator \
    open-telemetry/opentelemetry-operator \
    --namespace observability \
    --create-namespace
```

We are ready to deploy OpenTelemetry Collector. We can achieve this using few different kinds of deployments. To keep short route between pods and OpenTelemety receivers we use `DeamonSet`. So we have one pod with collector on every node and it's carry about process and export telemetry

To do that we apply [DeamonSet definition](./OpenTelemetryCollector.yaml) shown below

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: my-collector
  namespace: observability
spec:
  mode: daemonset
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
      jaeger:
        protocols:
          grpc:
          thrift_compact:
    processors:

    exporters:
      logging:
        loglevel: debug
      otlp:
        endpoint: tempo.observability.svc:4317
        tls:
          insecure: true
      prometheusremotewrite:
        endpoint: http://kube-prometheus-stack-prometheus.observability.svc:9090/api/v1/write

    service:
      pipelines:
        traces:
          receivers: [otlp, jaeger]
          processors: []
          exporters: [otlp]
        metrics:
          receivers: [otlp]
          processors: []
          exporters: [prometheusremotewrite]
        logs:
          receivers: [otlp]
          processors: []
          exporters: [logging]
```

Also we specify 2 receivers protocols: otlp, jaeger.

We save traces into Grafana Tempo using otlp protocol, and metric into Prometheus using `remote write receiver`


Finally we can use `http://my-collector-collector.observability.svc:4317` endpoint to pass into web app configuration and observe.
In our case we pass this variable using gitlab ci/cd variable section.
For demo purposes, we didn't use [.Net automatic Instrumentation](https://opentelemetry.io/docs/instrumentation/net/automatic/)


Additionally for presentation we configure Nginx ingress controller to send tracing. This can be done by modify [ingress config maps](./Configmap-ingress-nginx.yaml)

```yaml
apiVersion: v1
data:
  enable-opentracing: "true"
  jaeger-collector-host: my-collector-collector.observability.svc
  jaeger-collector-port: "6831"
  jaeger-propagation-format: w3c
  jaeger-service-name: nignx-ingresss-controller
  jaeger-trace-context-header-name: request-id
  opentracing-trust-incoming-span: "true"
kind: ConfigMap
# ....
```

Opentracing was used because cluster have installed 1.3.1 version of nginx ingress controller. Opentelemetry support was added starting from 1.4.0.

## Summary

https://grafana.emotoagh.eu.org/

## References

- [OpenTelemetry](https://opentelemetry.io)
- [OpenTelemetry in ASP.Net web application](https://opentelemetry.io/docs/instrumentation/net/libraries/)
- [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)
- [OpenTelemetry Operator for Kubernetes](https://opentelemetry.io/docs/k8s-operator)
- [Cert manager](https://cert-manager.io/docs/installation)
- [Grafana Tempo](https://grafana.com/oss/tempo)

### Helm charts repo:

- [Prometheus Helm](https://github.com/prometheus-community/helm-charts)
- [OpenTelemetry Operator](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-operator)
- [Grafana Tempo](https://github.com/grafana/helm-charts/tree/main/charts/tempo)

