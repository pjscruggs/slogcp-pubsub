# slogcp-pubsub

`github.com/pjscruggs/slogcp-pubsub` provides Pub/Sub helpers for
[slogcp](https://github.com/pjscruggs/slogcp).

- `Inject` / `Extract` for trace context propagation via
  `pubsub.Message.Attributes`.
- `WrapReceiveHandler` to derive a per-message `*slog.Logger`, attach it to the
  handler context (so `slogcp.Logger(ctx)` works), and optionally start an
  application-level consumer span.

The helpers enrich application logs. Your receive callback chooses when to log
message processing and completion.

## Install

```bash
go get github.com/pjscruggs/slogcp-pubsub
```

```go
import (
    slogcppubsub "github.com/pjscruggs/slogcp-pubsub"
    "github.com/pjscruggs/slogcp/v2"
)
```

The package name remains `slogcppubsub`. Applications migrating from the
previous bundled package update their import path and use the same core module
throughout their logger setup and context helpers. Existing Pub/Sub options and
callbacks keep the same API.

## Publisher (inject)

```go
msg := &pubsub.Message{Data: []byte("hello")}
slogcppubsub.Inject(ctx, msg) // injects trace context

res := topic.Publish(ctx, msg)
_ = res
```

By default, slogcppubsub injects W3C trace context only. To also propagate
baggage, enable it explicitly (and avoid propagating sensitive/high-cardinality
data because Pub/Sub attributes are size-limited).

```go
slogcppubsub.Inject(ctx, msg, slogcppubsub.WithBaggagePropagation(true))
```

Enable `WithGoogClientInjection` to also write the Go client's
`googclient_` trace keys.

```go
slogcppubsub.Inject(ctx, msg, slogcppubsub.WithGoogClientInjection(true))
```

## Subscriber (pull)

```go
handler, err := slogcp.NewHandler(os.Stdout)
if err != nil {
	log.Fatal(err)
}
logger := slog.New(handler)

err = sub.Receive(ctx, slogcppubsub.WrapReceiveHandler(
	func(ctx context.Context, msg *pubsub.Message) {
		slogcp.Logger(ctx).Info("processing message", "id", msg.ID)
		msg.Ack()
	},
	slogcppubsub.WithLogger(logger),
	slogcppubsub.WithSubscription(sub),
	// Accept googclient_traceparent when present.
	slogcppubsub.WithGoogClientExtraction(true),
))
```

## Trust boundary (public endpoint)

If you do not trust producer trace IDs, enable public endpoint mode to start a
new root trace and link to the extracted remote context instead of parenting.

```go
wrapped := slogcppubsub.WrapReceiveHandler(handler, slogcppubsub.WithPublicEndpoint(true))
```

When public endpoint mode is enabled and an upstream trace context is present,
derived loggers also include `pubsub.remote.traceparent` for debugging without
using it for Cloud Logging trace correlation.

Enable `WithInjectOnlyIfSpanPresent` to restrict injection to contexts that
contain a span.

```go
slogcppubsub.Inject(ctx, msg, slogcppubsub.WithInjectOnlyIfSpanPresent(true))
```

When public endpoint mode is enabled and `WithOTel(false)` is used, logs do not
correlate to extracted remote trace context by default. Enable `WithRemoteTrace`
or set `SLOGCP_TRUST_REMOTE_TRACE=true` to allow that correlation.

```go
wrapped := slogcppubsub.WrapReceiveHandler(
	handler,
	slogcppubsub.WithPublicEndpoint(true),
	slogcppubsub.WithOTel(false),
	slogcppubsub.WithRemoteTrace(true),
)
```

## Logger Cardinality

Derived loggers avoid high-cardinality fields by default. Enable them when
needed for debugging.

```go
wrapped := slogcppubsub.WrapReceiveHandler(
	handler,
	slogcppubsub.WithLogMessageID(true),
	slogcppubsub.WithLogOrderingKey(true),
	slogcppubsub.WithLogPublishTime(true),
)
```

## Push Subscriptions

For push subscriptions delivering to HTTP endpoints, Pub/Sub does not send
message attributes as HTTP headers unless you configure payload unwrapping and
metadata writing. To preserve end-to-end trace continuity, inject standard W3C
keys (`traceparent`/`tracestate`) into message attributes on publish and ensure
your push configuration forwards them as headers so your HTTP middleware can
extract them.

## More

- [Configuration reference](docs/CONFIGURATION.md)
- [Runnable example](.examples/pubsub/main.go)
- [Preserve existing OpenTelemetry instrumentation](https://github.com/pjscruggs/slogcp/blob/main/docs/recipes/pubsub-existing-otel.md)

## License

[Apache 2.0](LICENSE)
