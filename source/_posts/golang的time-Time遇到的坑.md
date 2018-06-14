---
title: golang的time.Time遇到的坑
date: 2018-06-11 23:20:53
tags:
---

## 坑的起源
使用不同的时区操作

## 后端存储的时间戳数字

    // 入库操作
    now := time.Now().Unix()
    fmt.Println(now) // 2018-6-11 06:00:00 +0800 CST
    db.Save(now)

    // 查询操作
    // 前端从时间框控件筛选时间(2018-6-11)，然后传至后端
    startTime = 1528675200000
    fmt.Println(startTime) // 2018-06-11 08:00:00 +0800 CST
    // 错误出现 上面入库的数据因为是在6点(CST时区)入库，所以查询不到

## 后端存储的是datetime类型

    t,_ := time.Parse("2006-01-02 15:04:05","2018-06-11 06:00:00") // UTC时区
    db.Save(t)


## docker容器时区
docker容器修改

## time.UnixNano() 大于int64的最大值
这个问题，标准包中已经说明了

    // UnixNano returns t as a Unix time, the number of nanoseconds elapsed
    // since January 1, 1970 UTC. The result is undefined if the Unix time
    // in nanoseconds cannot be represented by an int64 (a date before the year
    // 1678 or after 2262). Note that this means the result of calling UnixNano
    // on the zero Time is undefined.
    func (t Time) UnixNano() int64 {
        return (t.unixSec())*1e9 + int64(t.nsec())
    }

## 总结
1. 团队统一使用一个时区
2. 不用time.Parse()，使用time.ParseInLocation()