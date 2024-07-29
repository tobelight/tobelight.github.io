---
layout: post
title: 基于 Opentelemetry + SigNoz 的监控系统（一）
subtitle: otel-1
comments: true
tags:
  - "otel"
  - "java"
  - "go"
  - "trace"
  - "apm"
category:
  - "tech"
  - "otel"
  - "instrumentation"
# header-img: 2024/otel-1/banner.jpg
date: 2024-07-29 22:44:04
updated: 2024-07-29 22:44:04
---

本篇文章主要介绍在 go，java/kotlin 下，使用 opentelemetry 探针进行 metric，tracing 数据收集。

## 介绍

Opentelemetry 由 OpenCensus（主要负责收集指标 metric 信息）和 OpenTracing（主要负责收集链路 tracing 信息）结合而成。主要提供了**指标监控**，**链路追踪**，**日志**的可观测性的数据的收集和存储。

官方网站：
<https://opentelemetry.io>
<https://opentracing.io>
<https://opencensus.io>

## 开源仓库

<https://github.com/open-telemetry>

### java/kotlin 使用 Opentelemetry 探针

开源仓库：<https://github.com/open-telemetry/opentelemetry-java-instrumentation>
需要从这个地方下载最新 release 的 opentelemetry-javaagent.jar。

#### 使用方法

Java/kotlin 系列的应用基本上都支持使用 JVM 参数 `-javaagent:` 进行探针的注入。

实际使用时，我们可以配置 `JAVA_TOOL_OPTIONS` 环境变量为 `-javaagent:/opentelemetry/opentelemetry-javaagent.jar`。这样实际执行时，就相当于执行了 `java -javaagent:/opentelemetry/opentelemetry-javaagent.jar app.jar` 命令来启动我们的应用。

关于 javaagent 的功能，实际上是 java 支持一个叫做 `premain` 的方法，在 `main` 方法之前执行。所以 java 探针就可以在这个时间点执行反射操作，从 jvm 中获取到需要代理的类，在代理切面之前之后注入监测点，收集数据发送给 opentelemetry-collector 服务。

#### 配置信息

<https://opentelemetry.io/docs/zero-code/java/agent/getting-started/>

官方提供了两种配置方式，来为探针添加配置信息。

1. 使用 java 命令中 `-D配置名称=配置值` 的方式，例如 `-Dotel.service.name=your-service-name`
2. 使用环境变量的方式，例如 `export OTEL_SERVICE_NAME="your-service-name"`

我们可以看出对于配置项 `a.b.c` 我们可以用 `A_B_C`的方式配置环境变量，这两种方式是等价的。

我们主要需要配置的参数是 `OTEL_EXPORTER_OTLP_ENDPOINT` 即将收集到的数据使用 [OTLP](https://opentelemetry.io/docs/specs/otlp/) 协议发送给数据收集管理的服务, 我们使用的是 [opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)。

可选的配置参数是 `OTEL_SERVICE_NAME` 和 `OTEL_SERVICE_NAME`。
虽然探针会自动获取到服务名称，不过我们还是尽可能手动配置 service name 会更稳定一些，避免探针自动获取到的我们无法确定的服务名称。
而 OTEL_RESOURCE_ATTRIBUTES 则支持了可扩展的 trace 配置项，我们可以在这个配置项上配置多个 key=value 来增加 trace 的信息。
例如添加 `OTEL_RESOURCE_ATTRIBUTES=deployment.environment=stg` 就会在发送的每一条 trace 数据中带上环境信息，这在监控显示中很有用。

#### 特定问题

1. 收集不到 trace 数据，用户请求的/数据库调用的/中间件发出的信息收集不到数据。

目前支持的探针列表在：<https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/supported-libraries.md>。
自动化探针优先处理这个列表中最新版本的 lib，如果是旧版本或者不在支持列表中的 lib 就会出现无法自动收集到监控数据的情况。这种情况下需要手动处理或者升级 lib 来支持。
我主要遇到了 Ktor 和 Vert.x SQL Client 的问题，项目中使用的是旧版本导致收集不到数据，在更新到最新的大版本之后就可以收集到数据了。

### go 使用 Opentelemetry 探针

开源仓库：
<https://github.com/open-telemetry/opentelemetry-go>
<https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation>

#### opentelemetry-go 的使用方法

go 语言不像 java/kotlin 一样有虚拟机那么方便的东西，大部分探针都需要我们手动写代码来添加。
开源提供了两个代码仓库，opentelemetry-go 提供了基础的功能如 trace，span 的创建，获取，传输等；opentelemetry-go-contrib 中则提供了具体的探针实现，如针对 gRPC，http 等具体探针的实现。

一般我们需要在 main.go 下添加 initTraceProvider 的逻辑，来初始化 opentelemetry-go，让其他具体的探针可以从 context 中获取到整条链路的 trace 信息。

```go main.go
import (

 "go.opentelemetry.io/otel"
 "go.opentelemetry.io/otel/attribute"
 "go.opentelemetry.io/otel/exporters/otlp/otlptrace"
 "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
 "go.opentelemetry.io/otel/propagation"
 "go.opentelemetry.io/otel/sdk/resource"
 sdktrace "go.opentelemetry.io/otel/sdk/trace"
)


const (
 serviceName  = "serviceName"
 deployEnv    = "stg"
 collectorURL = "collectorURL:xxxx"
 insecure     = "true"
)

func initTracer() func(context.Context) error {
 var secureOption otlptracegrpc.Option
 if strings.ToLower(insecure) == "false" || insecure == "0" || strings.ToLower(insecure) == "f" {
  secureOption = otlptracegrpc.WithTLSCredentials(credentials.NewClientTLSFromCert(nil, ""))
 } else {
  secureOption = otlptracegrpc.WithInsecure()
 }

 exporter, err := otlptrace.New(
  context.Background(),
  otlptracegrpc.NewClient(
   secureOption,
   otlptracegrpc.WithEndpoint(collectorURL),
  ),
 )

 if err != nil {
  log.Fatalf("Failed to create exporter: %v", err)
 }
 resources, err := resource.New(
  context.Background(),
  resource.WithAttributes(
   attribute.String("service.name", serviceName),
   attribute.String("library.language", "go"),
   attribute.String("deployment.environment", deployEnv),
  ),
 )
 if err != nil {
  log.Fatalf("Could not set resources: %v", err)
 }

 otel.SetTracerProvider(
  sdktrace.NewTracerProvider(
   sdktrace.WithSampler(sdktrace.AlwaysSample()),
   sdktrace.WithBatcher(exporter),
   sdktrace.WithResource(resources),
  ),
 )

 otel.SetTextMapPropagator(
  propagation.NewCompositeTextMapPropagator(
   propagation.TraceContext{},
   propagation.Baggage{},
  ),
 )

 return exporter.Shutdown
}
```

然后再依据我们使用的 lib 来添加具体的探针代码，基本上每个探针都会有对应的 example 代码，我们参考下例子中的模版，对应到自己的实现上即可。
官方支持的探针列表：<https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation>

1. 服务器 grpc： <https://github.com/open-telemetry/opentelemetry-go-contrib/blob/main/instrumentation/google.golang.org/grpc/otelgrpc/example/server/main.go>
2. 服务器 gin：<https://github.com/open-telemetry/opentelemetry-go-contrib/blob/main/instrumentation/github.com/gin-gonic/gin/otelgin/example/server.go>
3. 客户端 http: <https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation/net/http>
   客户端 http 会把探针添加在 Transport 接口上，通过 `otelhttp.NewTransport()` 创建。
   大部分 go 语言的 client 仓库如 go-resty, go-swagger 等都支持传入 Transport 参数来初始化 client。通过这种方式创建的 client 就会自动支持 otel 探针数据收集了。
4. db client: <https://github.com/uptrace/opentelemetry-go-extra>
5. redis client：<https://github.com/redis/go-redis/tree/master/example/otel>
6. kafka client: 目前没有官方支持，也没有很好的社区支持。可以手动创建 trace 进行跟踪。trace 属性添加 MessagingSystem 来代表是消息队列中间件的 trace 数据。

   ```go kafka.go
   package kafka
   import {
    "github.com/segmentio/kafka-go"
    "go.opentelemetry.io/otel"
    semconv "go.opentelemetry.io/otel/semconv/v1.20.0"
   }
   func worker(ctx context.Context, reader *kafka.Reader, handler MsgHandler) {
    var err error
    tp := otel.GetTracerProvider()
    trace := tp.Tracer("github.com/tobelight/oteldemo/kafka") // 通常是当前包全路径名称
    for {
     msg, err := reader.FetchMessage(ctx)
     if err != nil {
      break
     }

     ctx, span := trace.Start(ctx, fmt.Sprintf("Fetched kafka message from %s", msg.Topic))
     span.SetAttributes(
      semconv.MessagingOperationProcess,
      semconv.MessagingSystem("kafka"),
      semconv.MessagingDestinationName(msg.Topic),
      semconv.MessagingKafkaMessageKey(string(msg.Key)),
      semconv.MessagingKafkaConsumerGroup(reader.Config().GroupID),
      semconv.MessagingKafkaClientID(reader.Stats().ClientID),
      semconv.MessagingKafkaDestinationPartition(msg.Partition),
      semconv.MessagingKafkaMessageOffset(int(msg.Offset)),
     )

     if err = handler(ctx, msg); err != nil {
      span.End()
      break
     }

     if err = reader.CommitMessages(ctx, msg); err != nil {
      span.End()
      break
     }
     span.End()
    }
   }

   type SubscriptionRequest struct {
    Topic      string
    Group      string
    MsgHandler MsgHandler
   }

   type MsgHandler func(ctx context.Context, msg kafka.Message) error
   ```

#### opentelemetry-go 的配置信息

使用 go-flags 从环境变量中获取，然后在 initTracer 中手动注入相关配置。详细代码见上面的 main.go 中。

## 题外话

之前用过 skywalking 来 trace web-flux 的项目，数据收集的不准就算了，还要添加一大堆额外的 lib 来支持，界面还贼难看找不到重点。
本来想用 grafana 来优化界面，发现 dashboard 基本上都是 metrics 类型的，对于 trace 没什么拿来即用的模版。
当然如果仅仅看 trace，jeager 的 UI 还是蛮不错的，不过要弄个统一的 APM，仅仅是 trace 还是不够的。
最终看下来，使用 opentelemetry 来做数据的采集和存储控制，用 signoz 来做 UI 的方式可能确实不错，毕竟 signoz 既有 trace 也有 metric 还有告警功能。
