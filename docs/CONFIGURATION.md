# Pub/Sub configuration

`slogcppubsub` provides helpers for carrying trace context across Pub/Sub
boundaries via `pubsub.Message.Attributes`, and for deriving message-scoped
loggers in pull subscribers (so `slogcp.Logger(ctx)` works inside receive
handlers). Use `Inject` before publishing and `WrapReceiveHandler` when
receiving. `Extract` / `ExtractAttributes` and `InfoFromContext` are available
when you want manual control or need to inspect message metadata.

| Option | Default | Description |
| --- | --- | --- |
| `WithLogger(*slog.Logger)` | `slog.Default()` | Base logger used to derive per-message loggers (used by `WrapReceiveHandler`). |
| `WithProjectID(string)` | detected at runtime | Project used when formatting Cloud Logging trace correlation fields and `gcp.project_id` span attributes. |
| `WithSubscription(*pubsub.Subscriber)` | (unset) | Captures a trimmed subscription ID from `sub.ID()` for span/logger enrichment. |
| `WithSubscriptionID(string)` | (unset) | Sets the trimmed subscription ID used for span/logger enrichment. |
| `WithTopic(*pubsub.Publisher)` | (unset) | Captures a trimmed topic ID from `topic.ID()` for span/logger enrichment. |
| `WithTopicID(string)` | (unset) | Sets the trimmed topic ID used for span/logger enrichment. |
| `WithOTel(bool)` | `true` | Enables/disables creation of an application-level consumer span around message processing. |
| `WithSpanStrategy(slogcppubsub.SpanStrategy)` | `SpanStrategyAlways` | Controls when consumer spans are started (`SpanStrategyAuto` skips when a local span is already active. `SpanStrategyAlways` always starts). |
| `WithTracerProvider(trace.TracerProvider)` | global tracer provider | Tracer provider used when consumer spans are created. |
| `WithSpanName(string)` | `pubsub.process` | Span name used when creating consumer spans. |
| `WithSpanAttributes(attribute.KeyValue...)` | (none) | Additional OpenTelemetry attributes appended to consumer spans. |
| `WithPublicEndpoint(bool)` | `false` | Treats producers as untrusted and starts a new root trace and links extracted remote context instead of parenting. |
| `WithRemoteTrace(bool)` | `false` | When `WithPublicEndpoint(true)` and no local span is created (for example, `WithOTel(false)`), controls whether logs correlate to extracted remote trace context. Also reads `SLOGCP_TRUST_REMOTE_TRACE` when unset. |
| `WithPropagators(propagation.TextMapPropagator)` | `otel.GetTextMapPropagator()` | Propagator used for attribute injection/extraction. Passing `nil` is treated the same as omitting the option (use the global propagator). Disable propagation with `WithTracePropagation(false)`. |
| `WithTracePropagation(bool)` | `true` | Enables/disables trace context extraction and injection via message attributes. |
| `WithBaggagePropagation(bool)` | `false` | Enables/disables baggage injection/extraction via message attributes when the propagator supports it. |
| `WithCaseInsensitiveExtraction(bool)` | `false` | Enables case-insensitive lookup for propagation keys during extraction. |
| `WithInjectOnlyIfSpanPresent(bool)` | `false` | Only injects when a valid span context is present on `ctx`. |
| `WithLogMessageID(bool)` | `false` | Adds `messaging.message.id` to derived loggers (high-cardinality). |
| `WithLogOrderingKey(bool)` | `false` | Adds `messaging.gcp_pubsub.message.ordering_key` to derived loggers (can be high-cardinality). |
| `WithLogDeliveryAttempt(bool)` | `true` | Adds `messaging.gcp_pubsub.message.delivery_attempt` to derived loggers when present. |
| `WithLogPublishTime(bool)` | `false` | Adds `pubsub.message.publish_time` to derived loggers. |
| `WithAttrEnricher(func(context.Context, *pubsub.Message, *MessageInfo) []slog.Attr)` | (none) | Appends additional attributes to the derived message logger. |
| `WithAttrTransformer(func(context.Context, []slog.Attr, *pubsub.Message, *MessageInfo) []slog.Attr)` | (none) | Mutates/redacts the derived message logger attribute slice before it is applied. |
| `WithGoogClientCompat(bool)` | `false` | Enables both extraction fallback and injection compatibility via `googclient_`-prefixed keys. |
| `WithGoogClientExtraction(bool)` | `false` | Enables extraction from `googclient_`-prefixed keys when standard keys are absent. |
| `WithGoogClientInjection(bool)` | `false` | Enables injection of `googclient_`-prefixed keys in addition to standard keys. |
