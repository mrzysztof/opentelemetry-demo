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

## Introduction
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
## Environment configuration
## Installation method
## How to reproduce
## Demo deployment
## Summary
## References
