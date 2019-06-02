背景
===

系统中有一批爬虫，每天都在从一些网络平台上获取数据储存到数据库中。随着时间的增长，数据库的数据越来越多。某些表的数据已经达到了数十G甚至数百G。已经到了风险很大的地步. 必须要做出一些改变了。

思考
---

既然决定了要进行改变就开始着手了。首先针对当前系统进行排查，入库语句和查询业务。进行逐步分析，因为业务主要以热点数据查询为主，更多的是以时间作为一个大条件。而针对某条或某项关联的某些条的查询是在时间之下。因此可以以时间作为一个跨度进行分表分区处理。

选型
---
在当前的系统中，数据量过大的表主要作为数据源。并且系统内部存在更多的平台级分片。而不同平台之间没有太复杂的联系。
与同事进行交流和讨论之后。暂时选型为针对大表进行分表+分区。


实施
===

首先针对平台进行数据拆表。不同平台的基础数据分开放置。然后针对某些数据量过大的平台表进行分区存储。


分区
---

```SQL
ALTER TABLE social_user_at PARTITION BY RANGE (TO_DAYS(create_time))(
    PARTITION p201901 VALUES LESS THAN (TO_DAYS('20190201')),
    PARTITION p201902 VALUES LESS THAN (TO_DAYS('20190301')),
    PARTITION p201903 VALUES LESS THAN (TO_DAYS('20190401')),
    PARTITION p201904 VALUES LESS THAN (TO_DAYS('20190501')),
    partition p201905 VALUES LESS THAN (TO_DAYS('20190601')),
    partition p201906 VALUES LESS THAN (TO_DAYS('20190701')),
    PARTITION p201907 VALUES LESS THAN (MAXVALUE)
)
```

在MySQL中创建一个 Event, 定时调用 响应的存储过程。进行自动分区增长

``` SQL
# 存储过程
CREATE PROCEDURE setNewPartitionForUserAt()
BEGIN
    # 查询出当前最大的分区的名称  20190701
    SELECT REPLACE(PARTITION_NAME, 'p', '') INTO @PCurName
    FROM information_schema.PARTITIONS
    WHERE `TABLE_SCHEMA` = 'TestDb'
      AND `table_name` = 'range_user_at'
    ORDER BY partition_ordinal_position DESC
    LIMIT 1;

    # 分区增加一个月
    SET @PNewDate = DATE(DATE_ADD(@PCurName), INTERVAL 1 MONTH) + 0;

    # 修改表，新加一个分区
    SET @addSql = CONCAT('ALTER TABLE range_user_at REORGANIZE PARTITION p', @PCurName, ' INTO (PARTITION p', @PCurName,
                         'VALUES LESS THAN (TO_DAYS(', @PCurName, ')),PARTITION p', @PNewDate,
                         'VALUES LESS THAN (MAXVALUE)', ')');
    PREPARE stmt FROM @addSql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;

    COMMIT;
END;

# 定时任务
CREATE EVENT auto_partition
    ON SCHEDULE
        EVERY 1 MONTH
            STARTS CURRENT_TIME
    DO CALL setNewPartitionForUserAt();
```
