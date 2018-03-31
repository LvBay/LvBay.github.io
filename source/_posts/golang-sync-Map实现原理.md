---
title: golang-sync.Map实现原理
date: 2018-03-31 17:41:59
tags:
---

## 结构体

### sync.Map

    type Map struct {
        // 常用的锁,在操作dirty时会用到
        mu Mutex

        read atomic.Value // readOnly

        // dirty值要么为空,要么为全部键值对
        dirty map[interface{}]*entry

        // 在查询read时,命中失败的次数,当misses大于dirty长度时,read.m 会直接指向dirty
        misses int
    }

Map.read实际值是readOnly结构体,可以看成是比dirty多了一个amended字段的结构

注意这个并不是真正的只读,添加操作是通过直接将m字段指向整个dirty完成的,删除操作是通过修改entry为expunged完成的


### readOnly

        type readOnly struct {
            m       map[interface{}]*entry
            amended bool 
        }

 主要看下readOnly.amended的值变化情况

- 在Store方法中,添加新的键值对时,如果**amended==false**,会遍历read值,将其中未被标记为删除的记录,复制到dirty中,然后read.amended会被修改为true

- 在Load方法中,如果miss次数大于diry长度时,会将read.m直接指向dirty,且read.amended被置为false

- 如果**amended==true**,说明当前map时间节点处于新添加键值对之后,复制dirty到read之前,此时dirty不为空,且dirty包括所有Map中未被删除的数据,read中的数据可能少于dirty

- 如果**amended==false**,说明说明dirty为空

### entry

    type entry struct {
        p unsafe.Pointer // *interface{}
    }

实际是键值对中的value



## 关键操作

### 寻值

    func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
        read, _ := m.read.Load().(readOnly)
        e, ok := read.m[key]
        if !ok && read.amended {
            m.mu.Lock()
            read, _ = m.read.Load().(readOnly)
            e, ok = read.m[key]
            if !ok && read.amended {
                e, ok = m.dirty[key]
                m.missLocked() // 如果read中没有,查询dirty
            }
            m.mu.Unlock()
        }
        if !ok {
            return nil, false
        }
        return e.load()
    }

### 复制dirty到read,并清空dirty

    func (m *Map) missLocked() {
        m.misses++
        if m.misses < len(m.dirty) {
            return
        }
        m.read.Store(readOnly{m: m.dirty})
        m.dirty = nil
        m.misses = 0
    }

### 存值

    func (m *Map) Store(key, value interface{}) {
        read, _ := m.read.Load().(readOnly)
        if e, ok := read.m[key]; ok && e.tryStore(&value) {
            return
        }

        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        if e, ok := read.m[key]; ok {
            if e.unexpungeLocked() {
                m.dirty[key] = e
            }
            e.storeLocked(&value)
        } else if e, ok := m.dirty[key]; ok {
            e.storeLocked(&value)
        } else {
            if !read.amended {
                m.dirtyLocked()
                m.read.Store(readOnly{m: read.m, amended: true})
            }
            m.dirty[key] = newEntry(value)
        }
        m.mu.Unlock()
    }

## 场景分析

### map.Store(a,"a") -> map.Store(b,"b")

第一次map.Store("a","a")
1. 因为备份标记为false,执行m.dirtyLocked()
2. 复制read -> dirty
3. 将备份标记read.amended更新为true
4. 将a存在dirty中

第二次map.Store("b","b")
1. 跳过if !read.amended 判断
2. 向dirty中存储数据

### map.Store("a","a") ->map.Load("a") map.Store("b","b")

map.Store("a","a")
1. 因为备份标记为false,执行m.dirtyLocked()
2. 复制read -> dirty
3. 将read.amended更新为true
4. 将a存在dirty中

map.Load("a")
1. 从read中读取失败
2. 从dirty中读取
3. m.missLocked()
4. miss+1
5. 因为miss >= len(dirty),直接将m.dirty复制到m.read中
6. 备份标记m.read.amended = false
7. 清空dirty

map.Store("b","b")
1. 因为备份标记为false,执行m.dirtyLocked()
2. 复制read -> dirty
3. 将read.amended更新为true
4. 将a存在dirty中

## 总结

### sync.Map是如何保证性能比直接在map中加锁的性能好

当写入操作较多时,性能是无法保证的,因为每次都有可能要遍历read复制到dirty中

当读多写少时,read是atomic.Value类型, 读取时利用了atomic.Value.Load实现了原子操作,没有用到锁,所以性能有所提升

### 怎么理解read.amended

可以把amended理解为一个备份标记,从read中遍历数据,复制到dirty中,相当于完成备份,amended为true,dirty为nil时,说明未备份,amended为false

### Map.read 和 Map.dirty的关系

可以把read看成是缓存,当缓存命中失败次数过多时,会从dirty中复制数据到read中,如果dirty不为空,那么dirty的数据大于等于read

### 从dirty中复制数据到read中,是否会导致原来的read中的数据丢失

不会,每次dirty创建时,都是从read中读取未被标记删除的数据复制到dirty中,之后dirty中的数据只会多于read,所以在从dirty中复制数据到read中时,只是会丢失已被标记删除的数据,而不会丢失实际数据

----

## 相关链接

[golang sync.Map 原理](https://blog.yiz96.com/golang-sync-map/)

[Go 1.9 sync.Map揭秘](http://colobu.com/2017/07/11/dive-into-sync-Map/)
    