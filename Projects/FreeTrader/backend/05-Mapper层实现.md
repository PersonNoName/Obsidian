# Mapper 层实现

## 概述

Mapper 层负责数据库访问，通过继承 MyBatis-Plus 的 `BaseMapper` 获得基础 CRUD 能力。

## BaseMapper 提供的通用方法

```java
// 插入
int insert(T entity);

// 根据 ID 删除
int deleteById(Serializable id);

// 根据条件删除
int delete(QueryWrapper<T> queryWrapper);

// 更新
int updateById(T entity);
int update(T entity, QueryWrapper<T> queryWrapper);

// 查询
T selectById(Serializable id);
List<T> selectList(QueryWrapper<T> queryWrapper);
T selectOne(QueryWrapper<T> queryWrapper);
Long selectCount(QueryWrapper<T> queryWrapper);
```

## UserMapper - 用户 Mapper

**文件位置**: `mapper/UserMapper.java`

```java
@Mapper
public interface UserMapper extends BaseMapper<User> {
}
```

继承 `BaseMapper<User>`，获得用户表的所有 CRUD 方法。

## CategoryMapper - 板块分类 Mapper

**文件位置**: `mapper/CategoryMapper.java`

```java
@Mapper
public interface CategoryMapper extends BaseMapper<Category> {

    @Select("""
        SELECT
            c.cid AS id,
            c.name,
            c.description,
            c.item_count AS fundsCount,
            c.status,
            COALESCE(AVG(
                CASE
                    WHEN latest.adjusted_nav > 0
                    THEN (latest.adjusted_nav - earliest.adjusted_nav) / earliest.adjusted_nav * 100
                    ELSE 0
                END
            ), 0) AS change
        FROM category c
        LEFT JOIN etf_info e ON c.name = e.sector
        LEFT JOIN etf_netasset latest ON e.ths_code = latest.ths_code AND latest.time = #{latestDay}
        LEFT JOIN etf_netasset earliest ON e.ths_code = earliest.ths_code AND earliest.time = #{earliestDay}
        WHERE c.status = 1
        GROUP BY c.cid, c.name, c.description, c.item_count, c.status
        """)
    List<SectorDTO> findAllSectorsWithAvgChange(@Param("latestDay") String latestDay,
                                                 @Param("earliestDay") String earliestDay);
}
```

**自定义查询**：
- `findAllSectorsWithAvgChange` - 查询所有板块及平均涨跌幅

## EtfInfoMapper - ETF 信息 Mapper

**文件位置**: `mapper/EtfInfoMapper.java`

```java
@Mapper
public interface EtfInfoMapper extends BaseMapper<EtfInfo> {

    @Select("""
        SELECT
            e.ths_code AS name,
            e.chinese_name AS fullName,
            latest.net_asset_value AS price,
            COALESCE(
                (latest.adjusted_nav - earliest.adjusted_nav) / earliest.adjusted_nav * 100,
                0
            ) AS returns,
            COALESCE(
                (latest.adjusted_nav - earliest.adjusted_nav) / earliest.adjusted_nav * 100,
                0
            ) / 100 AS returnsPercent
        FROM etf_info e
        INNER JOIN etf_netasset latest ON e.ths_code = latest.ths_code
            AND latest.time = #{latestDay}
        INNER JOIN etf_netasset earliest ON e.ths_code = earliest.ths_code
            AND earliest.time = #{earliestDay}
        WHERE e.sector = #{sector}
        ORDER BY returns DESC
        LIMIT #{limit}
        """)
    List<EtfDTO> findTopEtfsBySector(@Param("sector") String sector,
                                      @Param("latestDay") String latestDay,
                                      @Param("earliestDay") String earliestDay,
                                      @Param("limit") int limit);
}
```

**自定义查询**：
- `findTopEtfsBySector` - 查询板块内收益率最高的 ETF

## CalendarMapper - 交易日历 Mapper

**文件位置**: `mapper/CalendarMapper.java`

```java
@Mapper
public interface CalendarMapper extends BaseMapper<Calendar> {

    @Select("""
        SELECT "Day" FROM calendar
        WHERE "Day" <= #{today}
        AND "IsTradingDay" = 1
        ORDER BY "Day" DESC
        LIMIT #{days}
        """)
    List<String> findLastNTradingDays(@Param("today") String today,
                                       @Param("days") int days);
}
```

**自定义查询**：
- `findLastNTradingDays` - 查询最近 N 个交易日

## UserCollectionMapper - 用户收藏 Mapper

**文件位置**: `mapper/UserCollectionMapper.java`

```java
@Mapper
public interface UserCollectionMapper extends BaseMapper<UserCollection> {

    @Select("SELECT cid FROM user_collection WHERE user_id = #{userId}")
    List<Integer> findFavoriteCidsByUserId(@Param("userId") Integer userId);
}
```

**自定义查询**：
- `findFavoriteCidsByUserId` - 查询用户收藏的所有板块 ID

## 相关笔记

- [[04-实体层实现]]
- [[06-Service层实现]]
