---
title: golang-pprof实践
date: 2018-03-30 15:51:51
tags:
---


## 准备工作

在 golang官方博客的一篇[文章](https://blog.golang.org/profiling-go-programs)中看到了详细使用go tool pprof优化程序的过程.
于是我把[代码](https://code.google.com/archive/p/benchgraffiti/source/default/source)clone下来,在本地实践了一次.

    ➜  ~ go env
    GOARCH="amd64"
    GOBIN=""
    GOEXE=""
    GOHOSTARCH="amd64"
    GOHOSTOS="linux"
    GOOS="linux"
    GOPATH="/home/skt/mygo"
    GORACE=""
    GOROOT="/usr/local/go"
    GOTOOLDIR="/usr/local/go/pkg/tool/linux_amd64"
    GCCGO="gccgo"
    CC="gcc"
    GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build112976360=/tmp/go-build -gno-record-gcc-switches"
    CXX="g++"
    CGO_ENABLED="1"
    CGO_CFLAGS="-g -O2"
    CGO_CPPFLAGS=""
    CGO_CXXFLAGS="-g -O2"
    CGO_FFLAGS="-g -O2"
    CGO_LDFLAGS="-g -O2"
    PKG_CONFIG="pkg-config"

## havlak背景

项目havlak,在C++版本的运行时间是17.8s,花费了700M的内存, Go版本花费了25.20s,使用了1302M内存,接下来会使用go tool pprof工具,对Go程序进行优化

## 开始

### havlak1

切换到项目目录

    ➜  ~ cd mygo/src/benchgraffiti/havlak 
    ➜  havlak 

这里我直接用go build ,而不是博客中的make

因为我执行make havlak1.prof的时候报错,说我没有6g工具

原来是MakeFile中使用6g工具编译的,也许是生成博客的时间(2011.6.24)太久远了吧

    ➜  havlak go build havlak1.go

程序中给了-cpuprofile的接口,可以用来生成prof文件

    ➜  havlak ./havlak1 -cpuprofile=havlak1.prof
    # of loops: 76000 (including 1 artificial root node)

执行go tool pprof 二进制文件名 prof文件名,开始分析

    ➜  havlak go tool pprof havlak1 havlak1.prof
    File: havlak1
    Type: cpu
    Time: Mar 30, 2018 at 5:49pm (CST)
    Duration: 21.48s, Total samples = 26.63s (123.97%)
    Entering interactive mode (type "help" for commands, "o" for options)
    (pprof) top10
    Showing nodes accounting for 17790ms, 66.80% of 26630ms total
    Dropped 121 nodes (cum <= 133.15ms)
    Showing top 10 nodes out of 66
      flat  flat%   sum%        cum   cum%
    3500ms 13.14% 13.14%     6760ms 25.38%  runtime.scanobject /usr/local/go/src/runtime/mgcmark.go
    3120ms 11.72% 24.86%     3330ms 12.50%  runtime.mapaccess1_fast64 /usr/local/go/src/runtime/hashmap_fast.go
    2800ms 10.51% 35.37%    15770ms 59.22%  main.FindLoops /home/skt/mygo/src/benchgraffiti/havlak/havlak1.go
    1720ms  6.46% 41.83%     1720ms  6.46%  runtime.heapBitsForObject /usr/local/go/src/runtime/mbitmap.go
    1500ms  5.63% 47.47%     3600ms 13.52%  main.DFS /home/skt/mygo/src/benchgraffiti/havlak/havlak1.go
    1440ms  5.41% 52.87%     4670ms 17.54%  runtime.mapassign_fast64 /usr/local/go/src/runtime/hashmap_fast.go
    1170ms  4.39% 57.27%     5120ms 19.23%  runtime.mallocgc /usr/local/go/src/runtime/malloc.go
    1010ms  3.79% 61.06%     1020ms  3.83%  runtime.greyobject /usr/local/go/src/runtime/mgcmark.go
     780ms  2.93% 63.99%     1270ms  4.77%  runtime.evacuate /usr/local/go/src/runtime/hashmap.go
     750ms  2.82% 66.80%      750ms  2.82%  runtime.memclrNoHeapPointers /usr/local/go/src/runtime/memclr_amd64.s

这里对各个列名解释一下:

- flat 表示采样时,命中到该函数的时长,去除该函数中对其它函数的调用所花费的时间,也有些文档把这一列理解为函数被调用次数
- cum 表示采样时,命中到该函数的全部时长,包括函数中的其他调用所花费的时间
- cum >= flat,当cum远大于flat时,说明函数中有某个调用特别费时(比如上面的FindLoops,2800ms:15770ms)

上面展示的内容和官博中的文章比较,有些出入,初步估计是因为go版本不一样,编译方式不一样导致的.先忽略这个问题.

分析上面的top10结果可以看到top1是runtime包的scanobject,mapaccess1_fast64,前者是gc相关的标记函数,后者是map寻值的操作.但是这个貌似不能直接看到我的程序哪里出问题了.怎么办呢?按照博客文章的顺序,我们先来分析后者.

我想知道我的程序哪里调用了runtime.mapaccess1_fast64

    (pprof) web runtime.mapaccess1_fast64

{% asset_img havlak1_mapaccess1_fast64.svg 右键新标签页打开,可看大图 %}

可以看到主要有两个地方调用了mapaccess1_fast64,**DFS**和**FindLoops**

接着我想知道具体DFS或者FindLoops的哪一行导致问题的

    (pprof) list DFS
    Total: 26.63s
    ROUTINE ======================== main.DFS in /home/skt/mygo/src/benchgraffiti/havlak/havlak1.go
     1.50s      7.16s (flat, cum) 26.89% of Total
             .          .    240:func DFS(currentNode *BasicBlock, nodes []*UnionFindNode, number map[*BasicBlock]int, last []int, current int) int {
      10ms       10ms    241:	nodes[current].Init(currentNode, current)
      10ms      160ms    242:	number[currentNode] = current
         .          .    243:
         .          .    244:	lastid := current
     1.10s      1.10s    245:	for _, target := range currentNode.OutEdges {
     150ms      1.63s    246:		if number[target] == unvisited {
      40ms      3.60s    247:			lastid = DFS(target, nodes, number, last, lastid+1)
         .          .    248:		}
         .          .    249:	}
     120ms      590ms    250:	last[number[currentNode]] = lastid
      30ms       30ms    251:	return lastid
         .          .    252:}
         .          .    253:
         .          .    254:// FindLoops
         .          .    255://
         .          .    256:// Find loops and build loop forest using Havlak's algorithm, which

第一列是flat,第二列是cum,第三列是行数

可以看到最占时间的是247行,他本是是个递归,先不分析
接下来是246行,它是从map中取值,怎么会花费这么多时间呢,对了,前面就是在分析runtime.mapaccess1_fast64的调用情况,那应该就是他了. 在242,246,250行都使用了取值操作

这里引用官博中的一句话

>  For that particular lookup, a map is not the most efficient choice. Just as they would be in a compiler, the basic block structures have unique sequence numbers assigned to them.Instead of using a map[*BasicBlock]int we can use a []int, a slice indexed by the block number. There's no reason to use a map when an array or slice will do.

> 意思是对于某些特殊查询,map不是最有效的选择.就像它们在编译器中一样，基本块结构具有分配给它们的唯一序列号.使用切片[]int是一个更好的选择. 在array或者slice可以解决的情况下,没理由去使用map.(但是我觉得map方便啊...捂脸)

我的理解是因为数组是在内存中连续存储的,所以查询能在O(1)时间内完成,而map在数据量较大是,在对hash值进行查询的时候,性能可能会收到影响.

### havlak2

在对havlak1修改以后,形式好了很多.具体修改哪些,可以看[这里](https://github.com/rsc/benchgraffiti/commit/58ac27bcac3ffb553c29d0b3fb64745c91c95948)

看一下havlak2的pprof

    ➜  havlak go build havlak2.go
    ➜  havlak ./havlak2 -cpuprofile=havlak2.prof
    # of loops: 76000 (including 1 artificial root node)
    ➜  havlak go tool pprof havlak2 havlak2.prof
    File: havlak2
    Type: cpu
    Time: Mar 30, 2018 at 7:10pm (CST)
    Duration: 13.11s, Total samples = 17.55s (133.83%)
    Entering interactive mode (type "help" for commands, "o" for options)
    (pprof) top5  
    Showing nodes accounting for 8410ms, 47.92% of 17550ms total
    Dropped 93 nodes (cum <= 87.75ms)
    Showing top 5 nodes out of 72
        flat  flat%   sum%        cum   cum%
        2730ms 15.56% 15.56%     5630ms 32.08%  runtime.scanobject /usr/local/go/src/runtime/mgcmark.go
        1990ms 11.34% 26.89%    10290ms 58.63%  main.FindLoops /home/skt/mygo/src/benchgraffiti/havlak/havlak2.go
        1600ms  9.12% 36.01%     1600ms  9.12%  runtime.heapBitsForObject /usr/local/go/src/runtime/mbitmap.go
        1300ms  7.41% 43.42%     4370ms 24.90%  runtime.mallocgc /usr/local/go/src/runtime/malloc.go
        790ms  4.50% 47.92%      800ms  4.56%  runtime.greyobject /usr/local/go/src/runtime/mgcmark.go


哇哦! 之前的**DFS**和**runtime.mapaccess1_fast64**都不见了.现在被调用次数最多的是runtime.scanobject(15.56%)和main.FindLoops(11.34%).

**我试了一下list main.FindLoops,显示了FindLoops之外的代码.应该是因为没有函数内联导致的.**

先来分析runtime.scanobject,这个是和gc相关的扫描函数,有调度器调用,那么是什么导致调度器调用这个函数呢,应该是某些地方对象创建频繁,导致频繁gc.

涉及到内存分配,单从cpu-pprof是不好找原因的,那么从mem-pprof入手. 

### havlak3

havlak3提供了memprofile参数,可以生成mem-pprof. ([这里](https://github.com/rsc/benchgraffiti/commit/b78dac106bea1eb3be6bb3ca5dba57c130268232)可以看havlak3.go和havlak2.go的区别)

    ➜  havlak go build -gcflags '-N -l' havlak3.go //禁止内联优化
    ➜  havlak ./havlak3 -memprofile=havlak3.mprof
    ➜  havlak go tool pprof havlak3 havlak3.mprof
    File: havlak3
    Type: inuse_space
    Time: Mar 30, 2018 at 7:48pm (CST)
    Entering interactive mode (type "help" for commands, "o" for options)
    (pprof) top5
    Showing nodes accounting for 60.89MB, 100% of 60.89MB total
    Showing top 5 nodes out of 13
        flat  flat%   sum%        cum   cum%
    33.10MB 54.36% 54.36%    33.10MB 54.36%  main.FindLoops /home/skt/mygo/src/benchgraffiti/havlak/havlak3.go
        20MB 32.85% 87.21%       20MB 32.85%  main.NewBasicBlock /home/skt/mygo/src/benchgraffiti/havlak/havlak3.go
        3MB  4.93% 92.13%        3MB  4.93%  main.(*BasicBlock).AddInEdge /home/skt/mygo/src/benchgraffiti/havlak/havlak3.go
        2.50MB  4.11% 96.24%     2.50MB  4.11%  main.(*BasicBlock).AddOutEdge /home/skt/mygo/src/benchgraffiti/havlak/havlak3.go
        2.29MB  3.76%   100%    22.29MB 36.61%  main.(*CFG).CreateNode /home/skt/mygo/src/benchgraffiti/havlak/havlak3.go
    (pprof) 

main.FindLoops名列前茅,终究还是回到他这里了.

    (pprof) list FindLoops
    Total: 60.89MB
    ROUTINE ======================== main.FindLoops in /home/skt/mygo/src/benchgraffiti/havlak/havlak3.go
    33.10MB    33.10MB (flat, cum) 54.36% of Total
            .          .    263:		return
            .          .    264:	}
            .          .    265:
            .          .    266:	size := cfgraph.NumNodes()
            .          .    267:
        1.97MB     1.97MB   268:	nonBackPreds := make([]map[int]bool, size)
        5.77MB     5.77MB   269:	backPreds := make([][]int, size)
            .          .    270:
        1.97MB     1.97MB   271:	number := make([]int, size)
        1.97MB     1.97MB   272:	header := make([]int, size, size)
        1.97MB     1.97MB   273:	types := make([]int, size, size)
        1.97MB     1.97MB   274:	last := make([]int, size, size)
        1.97MB     1.97MB   275:	nodes := make([]*UnionFindNode, size, size)
            .          .    276:
            .          .    277:	for i := 0; i < size; i++ {
        5MB        5MB      278:		nodes[i] = new(UnionFindNode)
            .          .    279:	}
            .          .    280:
            .          .    281:	// Step a:
            .          .    282:	//   - initialize all nodes as unvisited.
            .          .    283:	//   - depth-first traversal and numbering.
            .          .    284:	//   - unreached BB's are marked as dead.
            .          .    285:	//
            .          .    286:	for i, bb := range cfgraph.Blocks {
            .          .    287:		number[bb.Name] = unvisited
        10.50MB    10.50MB  288:		nonBackPreds[i] = make(map[int]bool)
            .          .    289:	}
            .          .    290:
            .          .    291:	DFS(cfgraph.Start, nodes, number, last, 0)
            .          .    292:
            .          .    293:	// Step b:


第288行,又是因为可以用简单的数据结构切片或者数组时,却创建了map,且花费了10.5M的内存

另一方面,使用go tool pprof --inuse_objects havlak3 havlak3.mprof 可以显示创建内存的次数,而不是.这一点更符合寻找gc压力的目的.

    ➜  havlak go tool pprof --inuse_objects havlak3 havlak3.mprof
    File: havlak3
    Type: inuse_objects
    Time: Mar 30, 2018 at 7:48pm (CST)
    Entering interactive mode (type "help" for commands, "o" for options)
    (pprof) list FindLoops
    Total: 1245227
    ROUTINE ======================== main.FindLoops in /home/skt/mygo/src/benchgraffiti/havlak/havlak3.go
    393238     393238 (flat, cum) 31.58% of Total
         .          .    263:		return
         .          .    264:	}
         .          .    265:
         .          .    266:	size := cfgraph.NumNodes()
         .          .    267:
         1          1    268:	nonBackPreds := make([]map[int]bool, size)
         1          1    269:	backPreds := make([][]int, size)
         .          .    270:
         1          1    271:	number := make([]int, size)
         1          1    272:	header := make([]int, size, size)
         1          1    273:	types := make([]int, size, size)
         1          1    274:	last := make([]int, size, size)
         1          1    275:	nodes := make([]*UnionFindNode, size, size)
         .          .    276:
         .          .    277:	for i := 0; i < size; i++ {
    163845     163845    278:		nodes[i] = new(UnionFindNode)
         .          .    279:	}
         .          .    280:
         .          .    281:	// Step a:
         .          .    282:	//   - initialize all nodes as unvisited.
         .          .    283:	//   - depth-first traversal and numbering.
         .          .    284:	//   - unreached BB's are marked as dead.
         .          .    285:	//
         .          .    286:	for i, bb := range cfgraph.Blocks {
         .          .    287:		number[bb.Name] = unvisited
    229386     229386    288:		nonBackPreds[i] = make(map[int]bool)
         .          .    289:	}
         .          .    290:
         .          .    291:	DFS(cfgraph.Start, nodes, number, last, 0)
         .          .    292:
         .          .    293:	// Step b:

官博中写到
> That's reasonable when a map is being used to hold key-value pairs, but not when a map is being used as a stand-in for a simple set, as it is here.

> 在处理key-value数据时,用map是合适的,但是在处理简单的set集合类型时,map不一定是好的选择


### havlak4

在havlak4.go中使用了数组来替代map,[这里](https://github.com/rsc/benchgraffiti/commit/245d899f7b1a33b0c8148a4cd147cb3de5228c8a)可以看到和havlak3.go的区别

最开始我是想一个map和array差别真的就这么大么,有点不相信哎.

恰好在这个项目中执行xtime,可以输出二进制文件的耗时和内存占用情况

    ➜  havlak ./xtime ./havlak3
    # of loops: 76000 (including 1 artificial root node)
    19.33u 0.16s 13.70r 303024kB ./havlak3
    ➜  havlak ./xtime ./havlak4
    # of loops: 76000 (including 1 artificial root node)
    11.44u 0.06s 8.41r 166100kB ./havlak4
xtime 输出三个时间和一个内存占用

- usr_time 用户态时间
- sys_time 系统态时间
- real_time 实际花费时间
- memory 运行内存 

可以看到havlak4提高了将近40%的性能...捂脸

现在,比较**havlak4**和**havlak2**. 跳过**havlak3**版本,因为该版本只是加了一个memprof参数,

    ➜  havlak go build -gcflags '-N -l' havlak4.go
    ➜  havlak ./havlak4 -cpuprofile=havlak4.prof
    # of loops: 76000 (including 1 artificial root node)
    ➜  havlak go tool pprof havlak4 havlak4.prof
    File: havlak4
    Type: cpu
    Time: Mar 30, 2018 at 8:40pm (CST)
    Duration: 8.16s, Total samples = 10.62s (130.14%)
    Entering interactive mode (type "help" for commands, "o" for options)
    (pprof) top5
    Showing nodes accounting for 5990ms, 56.40% of 10620ms total
    Dropped 71 nodes (cum <= 53.10ms)
    Showing top 5 nodes out of 69
      flat  flat%   sum%        cum   cum%
    1920ms 18.08% 18.08%     3740ms 35.22%  runtime.scanobject /usr/local/go/src/runtime/mgcmark.go
    1720ms 16.20% 34.27%     5710ms 53.77%  main.FindLoops /home/skt/mygo/src/benchgraffiti/havlak/havlak4.go
    1010ms  9.51% 43.79%     1010ms  9.51%  runtime.heapBitsForObject /usr/local/go/src/runtime/mbitmap.go
     690ms  6.50% 50.28%      810ms  7.63%  main.DFS /home/skt/mygo/src/benchgraffiti/havlak/havlak4.go
     650ms  6.12% 56.40%     2550ms 24.01%  runtime.mallocgc /usr/local/go/src/runtime/malloc.go

看cum列:

                                    havlak2    havlak4

    - runtime.scanobject:           5630ms  -> 3740ms
    - main.FindLoops                10290ms -> 5710ms
    - runtime.heapBitsForObject     1600ms  -> 1010ms
    - runtime.mallocgc              4370ms  -> 2550ms

数值上,优化了不少.但是现在问题还是在main.FindLoops, runtime.scanobject, runtime.mallocgc上.FindLoops先不管,我想知道是什么导致了**scanobject**和**mallocgc**的调用.

先看scanobject

    (pprof) web runtime.scanobject

{% asset_img havlak4_scanobject.svg 右键新标签页打开,可看大图 %}

可以看到主要是FindLoops里面用了makeslice,从而导致mallocgc,然后导致scanobject.

再看mallocgc

    (pprof) web runtime.mallocgc

{% asset_img havlak4_mallocgc.svg 右键新标签页打开,可看大图 %}

mallocgc的调用关系有点复杂,但是通过web页面的方块颜色还是可以明显看出**FindLoops**调用了**newobjec**和**growslice**,从而导致**mallocgc**的调用.

那么问题就集中到FindLoops身上了

    (pprof) list FindLoops
    Total: 10.62s
    ROUTINE ======================== main.FindLoops in /home/skt/mygo/src/benchgraffiti/havlak/havlak4.go
     1.72s      5.71s (flat, cum) 53.77% of Total
         .          .    272:		return
         .          .    273:	}
         .          .    274:
         .          .    275:	size := cfgraph.NumNodes()
         .          .    276:
         .       30ms    277:	nonBackPreds := make([][]int, size)
         .      280ms    278:	backPreds := make([][]int, size)
         .          .    279:
         .      170ms    280:	number := make([]int, size)
         .       20ms    281:	header := make([]int, size, size)
         .       40ms    282:	types := make([]int, size, size)
         .       10ms    283:	last := make([]int, size, size)
         .       20ms    284:	nodes := make([]*UnionFindNode, size, size)
         .          .    285:
         .          .    286:	for i := 0; i < size; i++ {
      30ms      460ms    287:		nodes[i] = new(UnionFindNode)
         .          .    288:	}


下面是官博中的引用
> Every time FindLoops is called, it allocates some sizable bookkeeping structures. Since the benchmark calls FindLoops 50 times, these add up to a significant amount of garbage, so a significant amount of work for the garbage collector.

> Having a garbage-collected language doesn't mean you can ignore memory allocation issues. In this case, a simple solution is to introduce a cache so that each call to FindLoops reuses the previous call's storage when possible. (In fact, in Hundt's paper, he explains that the Java program needed just this change to get anything like reasonable performance, but he did not make the same change in the other garbage-collected implementations.)

每次调用**FindLoops**都会创建大量的对象(278,280),连续调用**FindLoops**后,就有可能
触发GC.

虽然golang是一门自带垃圾回收的语言,但是这不代表我们可以随意的创建对象.如果可以的话,应该考虑对象重用,或者创建一个cache缓存.

### havlak5

可以创建一个本地的cache对象

    var cache struct {
        size int
        nonBackPreds [][]int
        backPreds [][]int
        number []int
        header []int
        types []int
        last []int
        nodes []*UnionFindNode
    }

然后每次FindLoops去查询缓存,而不是直接创建对象

    if cache.size < size {
        cache.size = size
        cache.nonBackPreds = make([][]int, size)
        cache.backPreds = make([][]int, size)
        cache.number = make([]int, size)
        cache.header = make([]int, size)
        cache.types = make([]int, size)
        cache.last = make([]int, size)
        cache.nodes = make([]*UnionFindNode, size)
        for i := range cache.nodes {
            cache.nodes[i] = new(UnionFindNode)
        }
    }

    nonBackPreds := cache.nonBackPreds[:size]
    for i := range nonBackPreds {
        nonBackPreds[i] = nonBackPreds[i][:0]
    }
    backPreds := cache.backPreds[:size]
    for i := range nonBackPreds {
        backPreds[i] = backPreds[i][:0]
    }
    number := cache.number[:size]
    header := cache.header[:size]
    types := cache.types[:size]
    last := cache.last[:size]
    nodes := cache.nodes[:size]

这里有个疑问,重用cache的时候,难道不需要对cache值初始化吗,否则上次调用的产生的结果会对当前值产生影响吧.

[这里](https://github.com/rsc/benchgraffiti/commit/2d41d6d16286b8146a3f697dd4074deac60d12a4)可以看到havlak5和havlak4的区别

现在用xtime比较一下havlak5和havlak4的性能

    ➜  havlak ./xtime ./havlak4                   
    # of loops: 76000 (including 1 artificial root node)
    11.09u 0.08s 8.11r 165820kB ./havlak4
    ➜  havlak ./xtime ./havlak5
    # of loops: 76000 (including 1 artificial root node)
    7.27u 0.06s 5.48r 171320kB ./havlak5

运行时间从8.11s到5.48s,提升了%25的性能,但是不知道为什么,内存占用反而更多了.


### havlak6

官博中提到,havlak6主要是对迭代做了一些优化,因为具体代码我没仔细研究,所以这里就不介绍了

以及使用了更地道的Go风格,数据结构+方法的形式,这个对运行时影响不大

比较一下和上个版本的性能

    ➜  havlak ./xtime ./havlak5
    # of loops: 76000 (including 1 artificial root node)
    7.27u 0.06s 5.48r 171320kB ./havlak5
    ➜  havlak go build -gcflags="-N -l" havlak6.go
    ➜  havlak ./xtime ./havlak6                   
    # of loops: 76000 (including 1 artificial root node)
    2.90u 0.03s 2.71r 87524kB ./havlak6

运行时间从5.48s优化到2.71s

接下来,文章中用c++代码重写havlak6的代码,对比了一下不同语言之间的性能,go运行时间稍微大于c++

----
## 总结

### **优化思路**
topN中,如果排在前面的是runtime函数,可以用pprof web runtime.xxx被调用的原因,一直向上层寻找自己的代码

如果排在前面的是自己的代码,说明自己的这块代码被调用次数较多,如果函数对应的cum值也比较高,很有可能自己的这块代码有问题,可以用pprof list xxxx,查看具体是哪一行导致的问题

### **优化的时候,是看主要flat列还是cum列?**

可以先选择优化flat列,因为flat可以看成是函数被调用次数,所以对flat值比较高函数进行优化时,优化收益可以看成是被调用次数*本次优化值

### **cum值较高,可能的原因是什么?**
cum值表示整个函数(包括内部调用其它函数)的运行时间,如果cum值较高,有两种原因:

- 函数内部的其他函数比较耗时
- 函数中golang的内置函数比较费时,例如make,append,new,map寻值

### **什么时候用cpupprof,什么时候用mempprof**

可以先分析cpupprof,如果其中发现gc相关的代码占用时间较多的话,可以再去分析mempprof

### **如果在topN中显示标准包的一些函数占用时间较多,怎么去优化我的程序呢.**

先用pprof web runtime.xxx命令,找到是哪些程序调用了这个标准包函数,然后依次向上层寻找跟我的程序有关的函数名

### **已经定位到了某个函数有问提,怎么确定是哪一行?**

可以用pprof list xxx, 注意编译的时候禁止函数内联,否则list会展示其他函数名

### **平时写代码,需要注意哪些坑呢**

- 能用切片,数组解决的,就不要用map,一方面是可能查询会慢,另一方面map分配内存也更多
- 如果可以的话,尽量重用对象
- map,slice在创建的时候,尽量声明大小,较低扩容函数的调用

## 最后
阅读这篇博客+实践+记录,大概花费了我10个小时的时间.

虽然我的英语水平不是很好,但是这篇英文博客看下来竟然感觉非常的爽!推荐大家花几个小时照着博客也去实践一下

## 相关链接

[鸟窝博客-深入Go语言](http://colobu.com/2016/07/04/dive-into-go-11/)

[golang官博](https://blog.golang.org/profiling-go-programs)












    



    







