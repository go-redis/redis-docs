# go-redis joins Redis org on GitHub

Today the go-redis team is thrilled to release go-redis v9, which adds support for RESP3 protocol,
introduces new hooks API, improves pipelines retries. and allows to monitor performance using
OpenTelemetry tracing and metrics.

With this release, we also took a chance to move the [go-redis](https://github.com/redis/go-redis)
repository under the Redis org on GitHub, which means that you should use a new import path:

```go
// old, deprecated
import "github.com/go-redis/redis/v9"

// new, recommended
import "go.redis.io/redis/v9"
```

Other than that, the API is the same:

```go
package main

import (
	"context"
	"go.redis.io/redis/v9"
)

func main() {
	ctx := context.Background()

	rdb := redis.NewClient(&redis.Options{
		Addr:	  "localhost:6379",
		Password: "", // no password set
		DB:		  0,  // use default DB
	})

	err := rdb.Set(ctx, "key", "value", 0).Err()
	if err != nil {
		panic(err)
	}

	val, err := rdb.Get(ctx, "key").Result()
	if err != nil {
		panic(err)
	}
	fmt.Println("key", val)
}
```

## RESP3

[@monkey92t](https://github.com/monkey92t) did all the hard work to seamlessly support
[RESP3](https://github.com/antirez/RESP3/blob/master/spec.md) protocol in go-redis.

RESP3 is an updated version of RESP v2, which is the protocol used in Redis. It supports more
[data types](https://github.com/antirez/RESP3/blob/master/spec.md#resp3-types) which allows to
implement [client side caching](https://redis.io/docs/manual/client-side-caching/), which go-redis
does not support yet.

## Improved hooks

v9 comes with the simplified design of execution hooks which can be used to instrument Redis client
operations:

- `DialHook` is called to establish new connection.
- `ProcessHook` is called to process a Redis command.
- `ProcessPipelineHook` is called to process Redis commands in a pipeline.

The hooks API now looks like this:

```go
import "go.redis.io/redis/v9"

type redisHook struct{}

var _ redis.Hook = redisHook{}

func (redisHook) DialHook(hook redis.DialHook) redis.DialHook {
	return func(ctx context.Context, network, addr string) (net.Conn, error) {
		fmt.Printf("dialing %s %s\n", network, addr)
		conn, err := hook(ctx, network, addr)
		fmt.Printf("finished dialing %s %s\n", network, addr)
		return conn, err
	}
}

func (redisHook) ProcessHook(hook redis.ProcessHook) redis.ProcessHook {
	return func(ctx context.Context, cmd redis.Cmder) error {
		fmt.Printf("starting processing: <%s>\n", cmd)
		err := hook(ctx, cmd)
		fmt.Printf("finished processing: <%s>\n", cmd)
		return err
	}
}

func (redisHook) ProcessPipelineHook(hook redis.ProcessPipelineHook) redis.ProcessPipelineHook {
	return func(ctx context.Context, cmds []redis.Cmder) error {
		fmt.Printf("pipeline starting processing: %v\n", cmds)
		err := hook(ctx, cmds)
		fmt.Printf("pipeline finished processing: %v\n", cmds)
		return err
	}
}
```

## OpenTelemetry

[OpenTelemetry](https://uptrace.dev/opentelemetry/) is an open source and vendor-neutral API for
distributed tracing, logs, and metrics. go-redis comes with an OpenTelemetry instrumentation called
[redisotel](https://github.com/go-redis/redis/tree/master/extra/redisotel) that uses hooks API to
instrument go-redis clients.

In v9, the redisotel package was fully reworked to support both
[tracing](https://uptrace.dev/opentelemetry/distributed-tracing.html) and
[metrics](https://uptrace.dev/opentelemetry/metrics.html) like this:

```go
import (
	"go.redis.io/redis/extra/redisotel/v9"
	"go.redis.io/redis/v9"
)

rdb := redis.NewClient(...)
// rdb := redis.NewClusterClient(...)

// Enable tracing instrumentation.
if err := redisotel.InstrumentTracing(rdb); err != nil {
	panic(err)
}

// Enable metrics instrumentation.
if err := redisotel.InstrumentMetrics(rdb); err != nil {
	panic(err)
}
```

You can run the [example](https://github.com/go-redis/redis/tree/master/example/otel) yourself to
get a trace like this:

**screnshoot goes here**

See [Monitoring Go Redis Performance](https://redis.uptrace.dev/guide/go-redis-monitoring.html) for
details.

## Redis Metrics

Starting with v9, redisotel reports the following metrics:

- `db.client.connections.usage` - the number of connections that are currently in state described by
  the `state` attribute.
- `db.client.connections.timeouts` - the number of connection timeouts that have occurred trying to
  obtain a connection from the pool.
- `db.client.connections.create_time` - the time it took to create a new connection.
- `db.client.connections.use_time` - the time between borrowing a connection and returning it to the
  pool.

You can visualize and monitor those metrics using
[open source APM](https://uptrace.dev/get/open-source-apm.html) tool called Uptrace which is
developed by go-redis authors

Uptrace is an
[open-source DataDog competitor](https://uptrace.dev/get/compare/datadog-competitors.html) with an
intuitive query builder, rich dashboards, automatic alerts, and integrations for most languages and
frameworks.

After [installing Uptrace](https://uptrace.dev/get/install.html), it will automatically create the
following dashboard for you:

**screnshoot goes here**

In addition to monitoring go-redis client, you can also
[monitor Redis Server performance](https://uptrace.dev/opentelemetry/redis-performance-monitoring.html)
using OpenTelemetry Collector Agent.

## Prometheus

You can also send OpenTelemetry metrics to Prometheus using
[OpenTelemetry Prometheus exporter](https://uptrace.dev/opentelemetry/prometheus-metrics.html).

## ParseClusterURL

In v8, you could use `ParseURL` to connect to a Redis Server like this:

```go
options, err := redis.ParseURL("redis://<user>:<pass>@localhost:6379/<db>")
if err != nil {
	panic(err)
}
rdb := redis.NewClient(options)
```

In v9, we added `ParseClusterURL` that allows to connect to a Redis cluster:

```go
options, err := redis.("redis://user:password@localhost:6789?dial_timeout=3&read_timeout=6s&addr=localhost:6790&addr=localhost:6791")
if err != nil {
	panic(err)
}
rdb := redis.NewClusterClient(options)
```
