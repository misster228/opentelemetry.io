---
title: Instrumentation
linkTitle: Instrumentation
aliases: [manual]
weight: 30
description: Instrumentation for OpenTelemetry C++
cSpell:ignore: decltype labelkv nostd nullptr
---

<!-- markdownlint-disable no-duplicate-heading -->

{{% include instrumentation-intro.md %}}

{{% alert title="Note" %}}

OpenTelemetry C++ doesn't support automatic instrumentation when the source code
of the library you want to instrument isn't available.

{{% /alert %}}

## Setup

Follow the instructions in the
[Getting Started Guide](/docs/languages/cpp/getting-started/) to build
OpenTelemetry C++.

## Traces

### Initialize tracing

```cpp
auto provider = opentelemetry::trace::Provider::GetTracerProvider();
auto tracer = provider->GetTracer("foo_library", "1.0.0");
```

The `TracerProvider` acquired in the first step is a singleton object that is
usually provided by the OpenTelemetry C++ SDK. It is used to provide specific
implementations for API interfaces. In case no SDK is used, the API provides a
default no-op implementation of a `TracerProvider`.

The `Tracer` acquired in the second step is needed to create and start Spans.

### Start a span

```cpp
auto span = tracer->StartSpan("HandleRequest");
```

This creates a span, sets its name to `"HandleRequest"`, and sets its start time
to the current time. Refer to the API documentation for other operations that
are available to enrich spans with additional data.

### Mark a span as active

```cpp
auto scope = tracer->WithActiveSpan(span);
```

This marks a span as active and returns a `Scope` object. The scope object
controls how long a span is active. The span remains active for the lifetime of
the scope object.

The concept of an active span is important, as any span that is created without
explicitly specifying a parent is parented to the currently active span. A span
without a parent is called root span.

### Create nested spans

```cpp
auto outer_span = tracer->StartSpan("Outer operation");
auto outer_scope = tracer->WithActiveSpan(outer_span);
{
    auto inner_span = tracer->StartSpan("Inner operation");
    auto inner_scope = tracer->WithActiveSpan(inner_span);
    // ... perform inner operation
    inner_span->End();
}
// ... perform outer operation
outer_span->End();
```

Spans can be nested, and have a parent-child relationship with other spans. When
a given span is active, the newly created span inherits the active span’s trace
ID, and other context attributes.

### Context propagation

```cpp
// set global propagator
opentelemetry::context::propagation::GlobalTextMapPropagator::SetGlobalPropagator(
    nostd::shared_ptr<opentelemetry::context::propagation::TextMapPropagator>(
        new opentelemetry::trace::propagation::HttpTraceContext()));

// get global propagator
HttpTextMapCarrier<opentelemetry::ext::http::client::Headers> carrier;
auto propagator =
    opentelemetry::context::propagation::GlobalTextMapPropagator::GetGlobalPropagator();

//inject context to headers
auto current_ctx = opentelemetry::context::RuntimeContext::GetCurrent();
propagator->Inject(carrier, current_ctx);

//Extract headers to context
auto current_ctx = opentelemetry::context::RuntimeContext::GetCurrent();
auto new_context = propagator->Extract(carrier, current_ctx);
auto remote_span = opentelemetry::trace::propagation::GetSpan(new_context);
```

`Context` contains the metadata of the currently active Span including Span ID,
Trace ID, and flags. Context Propagation is an important mechanism in
distributed tracing to transfer this Context across service boundary often
through HTTP headers. OpenTelemetry provides a text-based approach to propagate
context to remote services using the W3C Trace Context HTTP headers.

### Further reading

- [Traces API](https://opentelemetry-cpp.readthedocs.io/en/latest/otel_docs/namespace_opentelemetry__trace.html)
- [Traces SDK](https://opentelemetry-cpp.readthedocs.io/en/latest/otel_docs/namespace_opentelemetry__sdk__trace.html)
- [Simple Metrics Example](https://github.com/open-telemetry/opentelemetry-cpp/tree/main/examples/metrics_simple)

## Metrics

### Initialize exporter and reader

Initialize an exporter and a reader. In this case, you initialize an OStream
Exporter which prints to stdout by default. The reader periodically collects
metrics from the Aggregation Store and exports them.

```cpp
std::unique_ptr<opentelemetry::sdk::metrics::MetricExporter> exporter{new opentelemetry::exporters::OStreamMetricExporter};
std::unique_ptr<opentelemetry::sdk::metrics::MetricReader> reader{
    new opentelemetry::sdk::metrics::PeriodicExportingMetricReader(std::move(exporter), options)};
```

### Initialize a meter provider

Initialize a MeterProvider and add the reader. Use this to obtain Meter objects
in the future.

```cpp
auto provider = std::shared_ptr<opentelemetry::metrics::MeterProvider>(new opentelemetry::sdk::metrics::MeterProvider());
auto p = std::static_pointer_cast<opentelemetry::sdk::metrics::MeterProvider>(provider);
p->AddMetricReader(std::move(reader));
```

### Create a counter

Create a Counter instrument from the Meter, and record the measurement. Every
Meter pointer returned by the MeterProvider points to the same Meter. This means
that the Meter can combine metrics captured from different functions without
having to constantly pass the Meter around the library.

```cpp
auto meter = provider->GetMeter(name, "1.2.0");
auto double_counter = meter->CreateDoubleCounter(counter_name);
// Create a label set which annotates metric values
std::map<std::string, std::string> labels = {{"key", "value"}};
auto labelkv = common::KeyValueIterableView<decltype(labels)>{labels};
double_counter->Add(val, labelkv);
```

### Create a histogram

Create a histogram instrument from the meter, and record the measurement.

```cpp
auto meter = provider->GetMeter(name, "1.2.0");
auto histogram_counter = meter->CreateDoubleHistogram("histogram_name");
histogram_counter->Record(val, labelkv);
```

### Create an observable counter

Create an observable counter instrument from the meter, and add a callback. The
callback is used to record the measurement during metrics collection. Ensure to
keep the Instrument object active for the lifetime of collection.

```cpp
auto meter = provider->GetMeter(name, "1.2.0");
auto counter = meter->CreateDoubleObservableCounter(counter_name);
counter->AddCallback(MeasurementFetcher::Fetcher, nullptr);
```

### Create views

#### Map the counter instrument to sum aggregation

Create a view to map the Counter Instrument to Sum Aggregation. Add this view to
provider. View creation is optional unless you want to add custom aggregation
config, and attribute processor. Metrics SDK creates a missing view with default
mapping between Instrument and Aggregation.

```cpp
std::unique_ptr<opentelemetry::sdk::metrics::InstrumentSelector> instrument_selector{
    new opentelemetry::sdk::metrics::InstrumentSelector(opentelemetry::sdk::metrics::InstrumentType::kCounter, "counter_name")};
std::unique_ptr<opentelemetry::sdk::metrics::MeterSelector> meter_selector{
    new opentelemetry::sdk::metrics::MeterSelector(name, version, schema)};
std::unique_ptr<opentelemetry::sdk::metrics::View> sum_view{
    new opentelemetry::sdk::metrics::View{name, "description", opentelemetry::sdk::metrics::AggregationType::kSum}};
p->AddView(std::move(instrument_selector), std::move(meter_selector), std::move(sum_view));
```

#### Map the histogram instrument to histogram aggregation

```cpp
std::unique_ptr<opentelemetry::sdk::metrics::InstrumentSelector> histogram_instrument_selector{
    new opentelemetry::sdk::metrics::InstrumentSelector(opentelemetry::sdk::metrics::InstrumentType::kHistogram, "histogram_name")};
std::unique_ptr<opentelemetry::sdk::metrics::MeterSelector> histogram_meter_selector{
    new opentelemetry::sdk::metrics::MeterSelector(name, version, schema)};
std::unique_ptr<opentelemetry::sdk::metrics::View> histogram_view{
    new opentelemetry::sdk::metrics::View{name, "description", opentelemetry::sdk::metrics::AggregationType::kHistogram}};
p->AddView(std::move(histogram_instrument_selector), std::move(histogram_meter_selector),
    std::move(histogram_view));
```

#### Map the observable counter instrument to sum aggregation

```cpp
std::unique_ptr<opentelemetry::sdk::metrics::InstrumentSelector> observable_instrument_selector{
    new opentelemetry::sdk::metrics::InstrumentSelector(opentelemetry::sdk::metrics::InstrumentType::kObservableCounter,
                                     "observable_counter_name")};
std::unique_ptr<opentelemetry::sdk::metrics::MeterSelector> observable_meter_selector{
  new opentelemetry::sdk::metrics::MeterSelector(name, version, schema)};
std::unique_ptr<opentelemetry::sdk::metrics::View> observable_sum_view{
  new opentelemetry::sdk::metrics::View{name, "description", opentelemetry::sdk::metrics::AggregationType::kSum}};
p->AddView(std::move(observable_instrument_selector), std::move(observable_meter_selector),
         std::move(observable_sum_view));
```

### Further reading

- [Metrics API](https://opentelemetry-cpp.readthedocs.io/en/latest/otel_docs/namespace_opentelemetry__metrics.html#)
- [Metrics SDK](https://opentelemetry-cpp.readthedocs.io/en/latest/otel_docs/namespace_opentelemetry__sdk__metrics.html)
- [Simple Metrics Example](https://github.com/open-telemetry/opentelemetry-cpp/tree/main/examples/metrics_simple)

## Logs

The documentation for the logs API & SDK is missing, you can help make it
available by
[editing this page](https://github.com/open-telemetry/opentelemetry.io/edit/main/content/en/docs/languages/cpp/instrumentation.md).

## Next steps

You’ll also want to configure an appropriate exporter to
[export your telemetry data](/docs/languages/cpp/exporters) to one or more
telemetry backends.
