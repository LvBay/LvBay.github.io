---
title: postgresql性能优化笔记
date: 2018-09-28 20:03:36
tags:
---

# pg_stat_activity

1. pg_stat_activity.state 状态解析

2. update语句导致idle in transaction

3. query 字段为COMMIT

4. query 字段为BEGIN READ WRITE

5. query 为 SELECT 某些较长的sql展示为这个

6. client_addr 有些为192.168

7. realtime left join base VS base left join realtime

# explain

1. start_up_cost all_cost

# json字段优化

1. 使用json_extract_path_text(effects,'Consumption'))，使用的时候比较麻烦
2. 修改为jsonb，使用gin索引
    create index idx_resource_20180926_consum on resource_20180926 USING gin (effects);

3. 直接使用json字段索引
    CREATE INDEX ON resource_20180926((effects->>'Consumption'));

# left join

1. 如何让left join走索引

     SELECT A
        .user_id
    FROM
        base.unit
        A LEFT JOIN base.resource b ON A.user_id = b.user_id 
        AND A.auth_id = b.auth_id 
        AND A.campaign_id = b.campaign_id 
        AND A.unit_id = b.unit_id
        LEFT JOIN base.target C ON A.user_id = C.user_id 
        AND A.auth_id = C.auth_id 
        AND A.campaign_id = C.campaign_id 
        AND A.unit_id = C.unit_id
        LEFT JOIN base.creative d ON A.user_id = d.user_id 
        AND A.auth_id = d.auth_id 
        AND A.campaign_id = d.campaign_id 
        AND A.unit_id = d.unit_id 
    WHERE
        ( A.deleted_at IS NULL AND b.deleted_at IS NULL AND C.deleted_at IS NULL AND d.deleted_at IS NULL ) 

        AND (
            A.platform = 2
        AND A.user_id = '998825031364526080' 
        AND A.auth_id = '1044508644000931840')

上面sql的golang代码

    units := []base.Unit{}
    err := resource.DB.
        Select("a.user_id, a.auth_id, a.campaign_id, a.unit_id, b.meta_resource_id, a.unit_name, a.config_status, a.passive_status, a.work_time, a.start_time, a.end_time").
        Table("base.unit a").
        Joins("left join base.resource b on a.user_id = b.user_id and a.auth_id = b.auth_id and a.campaign_id = b.campaign_id and a.unit_id = b.unit_id").
        Joins("left join base.target c on a.user_id = c.user_id and a.auth_id = c.auth_id and a.campaign_id = c.campaign_id and a.unit_id = c.unit_id").
        Joins("left join base.creative d on a.user_id = d.user_id and a.auth_id = d.auth_id and a.campaign_id = d.campaign_id and a.unit_id = d.unit_id").
        Where("a.deleted_at is null and b.deleted_at is null and c.deleted_at is null and d.deleted_at is null").
        Where("a.platform = ? and a.user_id = ? and a.auth_id = ?", 2, "998825031364526080", "1044508644000931840").
        Limit(1).
        Scan(&units).Error
    fmt.Println("err:", err)

# cpu利用率飙高

消息堆积


# 联合索引

## 只有带user_id才能走索引

    EXPLAIN ANALYSE SELECT * FROM "base"."resource" WHERE "base"."resource"."deleted_at" IS NULL AND (( auth_id = '1044508644000931840' ) 
    AND ( resource_id = 'sc_213523592969230286' ) 
    AND ( user_id = '998825031364526080' ) 
    );

## 索引不仅仅需要命中

explain ANALYSE SELECT * FROM "base"."resource" WHERE "base"."resource"."deleted_at" IS NULL AND ((user_id = '1046367009123614720') AND (auth_id = '1046367009182326784') AND (resource_id = 'sc_-1866255835007264483') AND (platform = '2')
and campaign_id = '112455583' and unit_id = '128115026' )

VS

explain ANALYSE SELECT * FROM "base"."resource" WHERE "base"."resource"."deleted_at" IS NULL AND ((user_id = '1046367009123614720') AND (auth_id = '1046367009182326784') AND (resource_id = 'sc_-1866255835007264483') AND (platform = '2')
)

# 写在join中还是where中？

# gin索引对json字段性能提升