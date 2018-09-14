---
title: snowflake-分布式唯一ID生成器
date: 2018-09-13 14:10:04
tags:
---

# 为什么要了解snowflake

1. 数据库分表以后，数据库自增id无法满足全局唯一的性质

2. uuid作为主键无法保证id递增

# snowflake算法原理

{% asset_img snowflake.jpg snowflake %}

首先我们需要的是一个int64的id，可以通过对这64位bit划分命名空间，分别用来表示 时间戳，机器等来实现id唯一性。

41-bit的时间可以表示（1L<<41）/(1000L*3600*24*365)=69年的时间

10-bit机器可以分别表示1024台机器（5个bit是数据中心，5个bit的机器ID），这种5-5的划分实际上是可以自定义的。

12个自增序列号表示同一时间戳，同一机器下的自增流水号.

上面的设计保证了理论上snowflake方案的QPS约为409.6w/s（2^12*1000）。

# snowflake实现

直接参考github上的一个snowflake实现，代码略有删减

    var (
        // 起始时间戳 这个可以根据实际情况定义，如果项目是从2018-01-01运行，就可以设置为2018-01-01的时间戳，可以使用到2087年
        Epoch int64 = 1288834974657

        // 机器标识位
        NodeBits uint8 = 10

        // 自增序列号
        StepBits uint8 = 12

        // 机器标识最大值，实例化node时不能大于nodeMax
        nodeMax   int64 = -1 ^ (-1 << NodeBits)

        // 用于step循环的一个标识
        stepMask  int64 = -1 ^ (-1 << StepBits)

        // snowflake需要的一个参数
        timeShift uint8 = NodeBits + StepBits

        // 相当于自增序列号
        nodeShift uint8 = StepBits
    )

    // 生成id的节点服务
    type Node struct {
        mu   sync.Mutex
        time int64 // 最近使用的时间戳
        node int64 // 机器标识
        step int64 // 自增序列
    }

    // 实例化一个节点服务
    func NewNode(node int64) (*Node, error) {
        if node < 0 || node > nodeMax {
            return nil, errors.New("Node number must be between 0 and " + strconv.FormatInt(nodeMax, 10))
        }

        return &Node{
            time: 0,
            node: node,
            step: 0,
        }, nil
    }

    // 生成id
    func (n *Node) Generate() ID {

        n.mu.Lock()

        now := time.Now().UnixNano() / 1000000

        if n.time == now {
            n.step = (n.step + 1) & stepMask

            // 如果某一时间戳下的自增序列用完了，则切换时间戳
            if n.step == 0 {
                for now <= n.time {
                    now = time.Now().UnixNano() / 1000000
                }
            }
        } else {
            n.step = 0
        }

        n.time = now

        // snowflake算法
        r := ID((now-Epoch)<<timeShift |
            (n.node << nodeShift) |
            (n.step),
        )

        n.mu.Unlock()
        return r
    }

# 如何保证多个snowflake节点生成的id不重复

试想一下，如果部署两个snowflake节点，初始化的时候node值都为1，那么在同一毫秒内两个请求分别打在了两个节点上，那么有可能将会获取两个同样的id（time,node,step全部相同）。所以问题的关键是保证部署的时候，初始化workId(node)值不能重复。

其中一种解决方案是，通过etcd存储node列表

- 服务初始化时，连接etcd服务，获取指定key下的node列表 list
- 遍历list，取出一个最小的已过期的key值作为workId返回
- 定时刷新该key值的过期时间，如果服务挂掉，超过过期时间后该key值可能会被其他服务占用

# 唯一ID的其他解决方案

- UUID
- 数据库自增长序列或字段
- 步长+数据库自增序列

# FAQ

## 什么是趋势递增

趋势递增指的是同一节点下产生的id是递增的。

在多个snowflake节点的情况下，因为节点的workId不同，同一时间下获取的id有可能是多个节点产生的，此时就无法保证全局递增

## 如果etcd挂了怎么办？

可以在每次获取workId后，将workId存到本地。

## 解决时钟回退问题

时间回流的原因一般是因为服务器做时间同步，此时可能出现生成重复id的情况。

首先判断node.time是否大于当前服务器时间，如果大于说明发生了时间回流，返回错误并报警

每隔一段时间(3s)上报自身系统时间写入

# 参考资料

[Leaf——美团点评分布式ID生成系统](https://tech.meituan.com/MT_Leaf.html)
[分布式唯一id：snowflake算法思考](https://juejin.im/post/5a7f9176f265da4e721c73a8)