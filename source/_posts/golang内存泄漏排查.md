---
title: golang内存泄漏排查
date: 2020-01-20 17:51:17
tags: 内存泄漏
---

# golang内存泄漏排查

最近有一个项目主要是从pulsar中接收数据，再推送到其他topic中。在重启pulsar的时候会触发重连，同时也会出现内存泄漏。
因为这个泄漏比较缓慢，从docker stats可以发现每天泄漏10M左右的内存

# 主题

- 解决内存泄漏问题
- 理解golang-pprof的相关数据
- 理解golang的内存分配

# 复盘过程

之前我一直想从内存的pprof数据中查出内存泄漏的原因，事实证明这条路线不是很好，有以下原因：

1. heap-profiling它通过采样，比较直观的反映了内存分配情况。一般我们的注意力会放在内存占用较多的代码块，
但是内存泄漏不一定就意味着占用较多
2. heap-profiling是采样数据， 存在不确定性。有可能某段代码循环调用了100次，但是profiling只采集了50次
3. dave.cheney 在[博客](https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html#memory_profiling)中提到一个个人观点：memory profiling对内存泄漏排查作用不大

实际上我们完全可以换个方向，从goroutine-profiling入手

{% asset_img granfana.png %}

从图中可以看出在经历过项目初始化 -> 消息推送 -> 停止推送后，goroutine数量比初始化时高了一些，而且一直没有降下去。

通过收集go-prometheus上报的metrics数据，可以看到goroutine数量出现了上升且不释放的情况

通过对比两个时间点的gorourtine-profiling文件，找到新增的goroutine，然后再通过堆栈信息去/debug/pprof/goroutine?debug=2
返回的结果中查询，果不其然，有些goroutine被意外阻塞了。

例如send(ctx context.Context)，ack(ctx context.Context)这些函数本不该阻塞，它们之所以被阻塞住，是因为之前在做一些
封装的时候，随意传了一个不带超时时间的context对象

# runtime.Memstats解读

之前一直对golang的runtime.Memstats各个字段有些模糊的认知，这次我们来彻底把它们搞清楚！

## runtime.Memstats各字段的关系

具体每个字段代表什么意思，就不细说了。可以去读标准包的代码注释，或者[这里](https://colobu.com/2019/08/28/go-memory-leak-i-dont-think-so/)
我们主要理清楚各个字段之间的关系
{% asset_img runtime.Memstats.png %}

## debug/pprof/heap?debug=1 和 debug/pprof/heap的inuse-space数据不一致

{% asset_img inuse-space.png %}

{% asset_img inuse-space_proto.png %}

一个是61KB，另一个是5662KB

其实关键点就在runtime.MemProfileRate
在runtime/pprof/protomem.go

``` go
  // runtime/pprof/protomem.go

  // scaleHeapSample adjusts the data from a heap Sample to
  // account for its probability of appearing in the collected
  // data. heap profiles are a sampling of the memory allocations
  // requests in a program. We estimate the unsampled value by dividing
  // each collected sample by its probability of appearing in the
  // profile. heap profiles rely on a poisson process to determine
  // which samples to collect, based on the desired average collection
  // rate R. The probability of a sample of size S to appear in that
  // profile is 1-exp(-S/R).
  func scaleHeapSample(count, size, rate int64) (int64, int64) {
    if count == 0 || size == 0 {
      return 0, 0
    }

    if rate <= 1 {
      // if rate==1 all samples were collected so no adjustment is needed.
      // if rate<1 treat as unknown and skip scaling.
      return count, size
    }

    avgSize := float64(size) / float64(count)
    scale := 1 / (1 - math.Exp(-avgSize/float64(rate)))

    return int64(float64(count) * scale), int64(float64(size) * scale)
}

```

在protomem中会对数据根据runtime.MemProfileRate进行一次scale

当runtime.MemProfileRate=1时，两个profiling的inuse-space就是一致的

## inuse-space和runtime.Memstats.HeapAlloc字段不一致

因为inuse-space是一份采样数据，不是全量的内存分配数据。有些博客说是采样频率是1/1000而且可以配置，
这个我还没有找到具体出处。

## docker-stats, /proc/pid/status, ps, top

上面几个命令都是查看进程的内存占用情况

docker-stats 反应的是物理内存占用的一部分，和/proc/pid/status中RssAnon大致相等

``` bash
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
ca75e73e0bfe        pulsar-util-or      0.19%               42.8MiB / 1.952GiB    2.14%               1.24MB / 2.23MB     1.65MB / 0B         8
```

/proc/1/status

```  bash
/data # cat /proc/1/status
Name: pulsar-util
...
VmPeak:   115852 kB
VmSize:   115852 kB
VmLck:       0 kB
VmPin:       0 kB
VmHWM:   52568 kB
VmRSS:   52568 kB
RssAnon:   42584 kB
RssFile:    9984 kB
RssShmem:       0 kB
VmData:  103536 kB
VmStk:     132 kB
VmExe:    5516 kB
VmLib:       8 kB
VmPTE:     148 kB
VmPMD:      16 kB
VmSwap:       0 kB
...
```

ps命令

``` bash
/data # ps -o 'pid,rss,vsz'
PID   RSS  VSZ
    1  51m 113m
   12 1040 1592
   22    4 1516
```

top命令 S

``` bash

Mem total:2047036 anon:913928 map:103176 free:192520
 slab:62844 buf:31100 cache:816244 dirty:3856 write:0
Swap total:1048572 free:1011520
  PID   VSZ^VSZRW^  RSS (SHR) DIRTY (SHR) STACK COMMAND
    1  113m  101m 59600     4 39760     0   132 ./pulsar-util
   12  1596   228  1040   760   136     0   132 /bin/sh
   23  1528   160   824   760    60     0   132 top

```

## runtime.Memstats.Sys 和 top，ps命令不一致

top,ps 显示虚拟内存占用113M，runtime.Memstats.Sys却只有71M左右

目前我理解113M是进程初始化时申请的虚拟内存大小，早期版本的go程序在初始化时甚至会申请更多。
71M是当前程序实打实使用的虚拟内存

其实这个我暂时还没有找到源码或者官方解释。。。欢迎大家补充

# 总结

- 程序出现内存泄漏时，先做好数据收集工作，利用prometheus+granfana收集metrics
- heap-profiling数据并不能很好的帮助定位内存泄漏，可以优先从goroutine追查
- 内存泄漏可以分为暂时性泄漏和永久性泄漏，其中后者十有八九是由于goroutine泄漏导致
- 上述排查过程中对比两份profiling文件的工作，可以编写一个工具来实现，后续完善以后会分享出来

# 相关链接

1. [https://colobu.com/2019/08/28/go-memory-leak-i-dont-think-so/](https://colobu.com/2019/08/28/go-memory-leak-i-dont-think-so/)
2. [https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html](https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html)
3. [https://github.com/golang/go/issues/32284](https://github.com/golang/go/issues/32284)
4. [https://gfw.go101.org/article/memory-leaking.html](https://gfw.go101.org/article/memory-leaking.html)
