---
title: golang的OpenTracing和jaeger使用
date: 2018-06-12 16:23:13
tags:
---

## OpenTracing是什么？
当代分布式跟踪系统（例如，Zipkin, Dapper, HTrace, X-Trace等）旨在解决这些问题，但是他们使用不兼容的API来实现各自的应用需求。尽管这些分布式追踪系统有着相似的API语法，但各种语言的开发人员依然很难将他们各自的系统（使用不同的语言和技术）和特定的分布式追踪系统进行整合.

OpenTracing通过提供平台无关、厂商无关的API，使得开发人员能够方便的添加（或更换）追踪系统的实现。 OpenTracing提供了用于运营支撑系统的和针对特定平台的辅助程序库。

已经实现OpenTracing协议的项目有：
- Zipkin
- Jaeger
- Appdash



## jaeger和OpenTracing
jaeger实现了OpenTracing，而且后端存储支持memry(默认)，elasticsearch,cassandra

## jaeger使用

创建一个支持serve模式和client模式的web服务

    // main.go
    package main
    import (
        "flag"
        "log"

        jaegerClientConfig "github.com/uber/jaeger-client-go/config"
    )

    var (
        serverPort = flag.String("port", "8000", "server port")
        // 默认为服务模式
        actorKind  = flag.String("actor", "server", "server or client")
    )

    const (
        server = "server"
        client = "client"
    )

    func main() {
        flag.Parse()

        if *actorKind != server && *actorKind != client {
            log.Fatal("Please specify '-actor server' or '-actor client'")
        }

        cfg := jaegerClientConfig.Configuration{
            Sampler: &jaegerClientConfig.SamplerConfig{
                Type:  "const",
                Param: 1.0, // sample all traces
            },
        }
        // jaeger.NewRemoteReporter(transport)
        tracer, closer, _ := cfg.New(*actorKind)
        defer closer.Close()

        if *actorKind == server {
            runServer(tracer)
            return
        }

        runClient(tracer)

        // Close the tracer to guarantee that all spans that could
        // be still buffered in memory are sent to the tracing backend
        closer.Close()
    }

服务器模式

    // server.go
    package main
    
    import (
        "fmt"
        "io"
        "log"
        "net/http"
        "time"

        "github.com/opentracing-contrib/go-stdlib/nethttp"
        "github.com/opentracing/opentracing-go"
    )

    func getTime(w http.ResponseWriter, r *http.Request) {
        log.Print("Received getTime request")
        t := time.Now()
        ts := t.Format("Mon Jan _2 15:04:05 2006")
        io.WriteString(w, fmt.Sprintf("The time is %s", ts))
    }

    func redirect(w http.ResponseWriter, r *http.Request) {
        http.Redirect(w, r,
            fmt.Sprintf("http://localhost:%s/gettime", *serverPort), 301)
    }

    func runServer(tracer opentracing.Tracer) {
        http.HandleFunc("/gettime", getTime)
        http.HandleFunc("/", redirect)
        log.Printf("Starting server on port %s", *serverPort)
        err := http.ListenAndServe(
            fmt.Sprintf(":%s", *serverPort),
            // use nethttp.Middleware to enable OpenTracing for server
            nethttp.Middleware(tracer, http.DefaultServeMux))
        if err != nil {
            log.Fatalf("Cannot start server: %s", err)
        }
    }

客户端模式

    // client.go
    package main

    import (
        "fmt"
        "io/ioutil"
        "log"
        "net/http"

        "github.com/opentracing-contrib/go-stdlib/nethttp"
        "github.com/opentracing/opentracing-go"
        "github.com/opentracing/opentracing-go/ext"
        otlog "github.com/opentracing/opentracing-go/log"
        "golang.org/x/net/context"
    )

    func runClient(tracer opentracing.Tracer) {
        // nethttp.Transport from go-stdlib will do the tracing
        c := &http.Client{Transport: &nethttp.Transport{}}

        // create a top-level span to represent full work of the client
        span := tracer.StartSpan(client)
        span.SetTag(string(ext.Component), client)
        defer span.Finish()
        ctx := opentracing.ContextWithSpan(context.Background(), span)

        req, err := http.NewRequest(
            "GET",
            fmt.Sprintf("http://localhost:%s/", *serverPort),
            nil,
        )
        if err != nil {
            onError(span, err)
            return
        }

        req = req.WithContext(ctx)
        // wrap the request in nethttp.TraceRequest
        req, ht := nethttp.TraceRequest(tracer, req)
        defer ht.Finish()

        res, err := c.Do(req)
        if err != nil {
            onError(span, err)
            return
        }
        defer res.Body.Close()
        body, err := ioutil.ReadAll(res.Body)
        if err != nil {
            onError(span, err)
            return
        }
        fmt.Printf("Received result: %s\n", string(body))
    }

    func onError(span opentracing.Span, err error) {
        // handle errors by recording them in the span
        span.SetTag(string(ext.Error), true)
        span.LogKV(otlog.Error(err))
        log.Print(err)
    }

编译
go build

执行客户端请求代码
    ./opentracing-go-nethttp-demo

运行jaeger的docker镜像
    docker run -d -p5775:5775/udp -p16686:16686 jaegertracing/all-in-one:latest

执行客户端代码
    ./opentracing-go-nethttp-demo -actor client

此时访问 http://localhost:16686就可以看到jaeger上的记录了。

{% asset_img jaeger1.png %}

但是记录好像少了点，只有jaeger-query的，client和server的信息都没记录，查了下资料发现是因为启动jaeger的时候有些端口没开放。

    docker run \
    -p 5775:5775/udp \
    -p 16686:16686 \
    -p 6831:6831/udp \
    -p 6832:6832/udp \
    -p 5778:5778 \
    -p 14268:14268 \
    jaegertracing/all-in-one:latest



## 封装http请求
上面的例子就是对http请求的trace

## 将数据存储在es中
暂时还没成功，后面会更新。这里先把目前的操作记录下来。

在本地用docker启动了elasticsearch

    docker run -d -p 9200:9200 -e "http.host=0.0.0.0" -e "transport.host=127.0.0.1" docker.elastic.co/elasticsearch/elasticsearch:5.4.0

在本地用docker-compose启动jaeger

doker-compose.yaml:

    jaegertracing:
    image: jaegertracing/all-in-one:latest
    ports:
        - "5775:5775/udp"
        - "6831:6831/udp"
        - "6832:6832/udp"
        - "5778:5778"
        - "16686:16686"
        - "14268:14268"
    command:
        - "/go/bin/standalone-linux"
        - "--span-storage.type=elasticsearch"
        - "--query.static-files=/go/src/jaeger-ui-build/build/"
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch

报错：
    {"level":"fatal","ts":1528814826.4817529,"caller":"standalone/main.go:106","msg":"Failed to init storage factory","error":"health check timeout: no Elasticsearch node available","errorVerbose":"no Elasticsearch node available...
    
