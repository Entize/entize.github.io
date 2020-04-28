---
layout: post
title: '如何完成数据校验'
date: 2020-04-28
author: entize
tags: java etl validate
---

### 问题

`etl`中数据校验是个麻烦的问题,如何做好数据校验,做好数据清洗呢?
现在的处理方案基本是依据人力去处理数据,开发给出数据要求,实施通过`etl`工具从各个不同的数据库将数据抽取

在这个过程中,一些约定的数据在不经意的情况下很有可能有不符合要求的数据会被同步过来
而如果到了项目或者产品中则会出现一些异常信息导致用户体验很差

那么就需要一个预校验数据的功能,通过一些校验的规则来处理数据,能够给出预处理的结果,找出不符合的数据以及原因
通过对出错数据的记录和原因的描述来做出对应的修改以减少因为数据集成而导致的体验问题

### 想法
通过对表的字段设置一些校验规则,预设规则构想如下
- **非空** *不能为空 其他的校验除非是有自带的非空限制否则默认可空*
- **字段长度** *如果是单个数值表示最大的长度需求如果是以逗号分隔的2个数字则认为是`[min,max]`的长度区间*
- **数值区间** *如果是单个数值表示最大的数值需求如果是以逗号分隔的2个数字则认为是`[min,max]`的数值区间*
- **枚举值** *逗号分隔的预设值,如果字段的值出现了枚举以外的值则认为是非法的,如果非空没有设置则是允许为空值*
- **外联字段** *如果`B`表的`col`字段需要依赖`A`表的`col`字段则可以做`A.col`的设置,设置完成会校验`B`的`col`是否在`A.col`存在*

### 实现

#### 非空
做出字段的非空判断即可
```java
public boolean validNotEmpty(String str) {
    return str != null && str.trim().length() > 0;
}
```

#### 字段长度
存在字段长度则校验字段长度检查是否有上下限的限制,如果只有上限则判断字段长度是否<=限制\[0,max],否则检查是否在\[min,max]区间内
```java
public boolean validStrLen(String str) {
    String[] len = limit.split(",")
    if (len.length == 1) {
        return str.length <= len[0];
    } else {
        return str.length >= len[0] && str.length <= len[1];
    }
}
```

#### 数值区间
和字段长度类似,如果数值只有上限限制则检查字段的值是否在限制内即字段的数值<=限制`[0,max]`,否则检查字段的数值是否在`[min,max]`区间内
```java
public boolean validLimit(String str) {
    String[] limits = limit.split(",");
    if (limits.length == 1) {
        return Double.parseDouble(str) <= Double.parseDouble(limits[0]);
    } else {
        return Double.parseDouble(str) >= Double.parseDouble(limits[0]) && Double.parseDouble(str) <= Double.parseDouble(limits[1]);
    }
}
```

#### 枚举值
设置一些值,字段的值只能在这些值中间选择,是否可空由非空校验完成
```java
public static boolean validNotEmpty(String str) {
    String[] enumsArr = enums.split("[,，]");
    if (str == null || str.isEmpty()) return true;
    return Arrays.asList(enumsArr).contains(str);
}
```

#### 外联字段
外联字段需要找到目标表的字段的值合集,判断当前的值是否在目标表字段中,如果表数据量比较大做`IN`或者`EXISTS`查询都是比较麻烦的
而且在表中做`sql`的关联没办法很好的实现异库表字段对比,所以实现使用代码判断的方案
使用`Set`做`contains`判断,基于`hash`的`set`做包含的判断效率要高于`list`的判断
```java
public static boolean validContains(String str, Set<String> set) {
    return set.contains(str);
}
```