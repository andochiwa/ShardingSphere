# ShardingSphere简介

[官方文档](https://shardingsphere.apache.org/document/current/cn/overview/)

Apache ShardingSphere 是一套开源的分布式数据库解决方案组成的生态圈，它由 JDBC、Proxy 和 Sidecar（规划中）这 3 款既能够独立部署，又支持混合部署配合使用的产品组成。 它们均提供标准化的数据水平扩展、分布式事务和分布式治理等功能，可适用于如 Java 同构、异构语言、云原生等各种多样化的应用场景。

Apache ShardingSphere 旨在充分合理地在分布式的场景下利用关系型数据库的计算和存储能力，而并非实现一个全新的关系型数据库。 关系型数据库当今依然占有巨大市场份额，是企业核心系统的基石，未来也难于撼动，我们更加注重在原有基础上提供增量，而非颠覆。

Apache ShardingSphere 5.x 版本开始致力于可插拔架构，项目的功能组件能够灵活的以可插拔的方式进行扩展。 目前，数据分片、读写分离、数据加密、影子库压测等功能，以及 MySQL、PostgreSQL、SQLServer、Oracle 等 SQL 与协议的支持，均通过插件的方式织入项目。 开发者能够像使用积木一样定制属于自己的独特系统。Apache ShardingSphere 目前已提供数十个 SPI 作为系统的扩展点，仍在不断增加中

**总结：**

1. 一套开源的分布式数据库中间件的解决方案
2. 有Sharding-JDBC, Sharding-Proxy, Sharding-Sidecar三个产品
3. 定位为关系型数据库中间件，合理在分布式环境下使用关系型数据库操作

## ShardingSphere-JDBC

定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。 它使用客户端直连数据库，以 jar 包形式提供服务，无需额外部署和依赖，可理解为增强版的 JDBC 驱动，完全兼容 JDBC 和各种 ORM 框架。

- 适用于任何基于 JDBC 的 ORM 框架，如：JPA, Hibernate, Mybatis, Spring JDBC Template 或直接使用 JDBC。
- 支持任何第三方的数据库连接池，如：DBCP, C3P0, BoneCP, Druid, HikariCP 等。
- 支持任意实现 JDBC 规范的数据库，目前支持 MySQL，Oracle，SQLServer，PostgreSQL 以及任何遵循 SQL92 标准的数据库。

## ShardingSphere-Proxy

定位为透明化的数据库代理端，提供封装了数据库二进制协议的服务端版本，用于完成对异构语言的支持。 目前提供 MySQL 和 PostgreSQL 版本，它可以使用任何兼容 MySQL/PostgreSQL 协议的访问客户端(如：MySQL Command Client, MySQL Workbench, Navicat 等)操作数据，对 DBA 更加友好。

- 向应用程序完全透明，可直接当做 MySQL/PostgreSQL 使用。
- 适用于任何兼容 MySQL/PostgreSQL 协议的的客户端。

# 分库分表

## 为什么要分库分表

数据库的数据量是不可控的，随着时间和业务的发展，造成表里的数据越来越多，如果再去对数据库表 CRUD，就可能会造成性能问题

分库分表就是为了解决数据量过大而导致数据库性能降低问题，将原来独立的数据库拆分成若干数据库而成，将数据大表拆分成若干数据表组成，使得单一数据库、单一数据表的数据量变小，从而达到提升数据库性能的目的

## 分库分表的方式

分库分表有两种分式：**垂直切分**和**水平切分**

## 垂直切分

### 1. 垂直分表

**将一个表按照字段分成多个表，每个表存储其中一部分字段**

它带来的提升是：

1.为了避免IO争抢并减少锁表的几率，各个表的字段互不影响

2.充分发挥热门数据的操作效率，对某个字段的操作的高效率不会被另一个字段的低效率所拖累。

> 为什么大字段IO效率低：第一是由于数据量本身大，需要更长的读取时间；第二是跨页，页是数据库存储单位，很多查找及定位操作都是以页为单位，单页内的数据行越多数据库整体性能越好，而大字段占用空间大，单页内存储行数少，因此IO效率较低。第三，数据库以行为单位将数据加载到内存中，这样表中字段长度较短且访问频率较高，内存能加载更多的数据，命中率更高，减少了磁盘IO，从而提升了数据库性能。

一般来说，某业务实体中的各个数据项的访问频次是不一样的，部分数据项可能是占用存储空间比较大的BLOB或是TEXT。所以，当表数据量很大时，可以将表按字段切开，将热门字段、冷门字段分开放置在不同库中，这些库可以放在不同的存储设备上，避免IO争抢。垂直切分带来的性能提升主要集中在热门数据的操作效率上，而且磁盘争用情况减少。

通常我们按以下原则进行垂直拆分:

* 把不常用的字段单独放在一张表
* 把text，blob等大字段拆分出来放在附表中
* 经常组合查询的列放在一张表中;

### 2. 垂直分库

**垂直分库是指按照业务将表进行分类，分布到不同的数据库上面，每个库可以放在不同的服务器上，它的核心理念是专库专用**

它带来的提升是：

* 解决业务层面的耦合，业务清晰
* 能对不同业务的数据进行分级管理、维护、监控、扩展等
* 高并发场景下，垂直分库一定程度的提升IO、数据库连接数、降低单机硬件资源的瓶颈
* 垂直分库通过将表按业务分类，然后分布在不同数据库，并且可以将这些数据库部署在不同服务器上，从而达到多个服务器共同分摊压力的效果，但是依然没有解决单表数据量过大的问题。

## 水平切分

### 1. 水平分库

**水平分库是把同一个表的数据按一定规则拆到不同的数据库中，每个库可以放在不同的服务器上**

> 垂直分库是把不同表拆到不同数据库中，它是对数据行的拆分，不影响表结构

它带来的提升是：

- 解决了单库大数据，高并发的性能瓶颈。
- 提高了系统的稳定性及可用性。

> 稳定性体现在IO冲突减少，锁定减少，可用性指某个库出问题，部分可用

当一个应用难以再细粒度的垂直切分，或切分后数据量行数巨大，存在单库读写、存储性能瓶颈，这时候就需要进行水平分库了，经过水平切分的优化，往往能解决单库存储量及性能瓶颈。但由于同一个表被分配在不同的数据库，需要额外进行数据操作的路由工作，因此大大提升了系统复杂度

### 2. 水平分表

**水平分表是在同一个数据库内，把同一个表的数据按一定规则拆到多个表中**

它带来的提升是：

- 优化单一表数据量过大而产生的性能问题
- 避免IO争抢并减少锁表的几率库内的水平分表，解决了单一表数据量过大的问题，分出来的小表中只包含一部分数据，从而使得单个表的数据量变小，提高检索性能。

## 总结

**垂直分表**：可以把一个宽表的字段按访问频次、是否是大字段的原则拆分为多个表，这样既能使业务清晰，还能提升部分性能。拆分后，尽量从业务角度避免联查，否则性能方面将得不偿失。

**垂直分库**：可以把多个表按业务耦合松紧归类，分别存放在不同的库，这些库可以分布在不同服务器，从而使访问压力被多服务器负载，大大提升性能，同时能提高整体架构的业务清晰度，不同的业务库可根据自身情况定制优化方案。但是它需要解决跨库带来的所有复杂问题。

**水平分库**：可以把一个表的数据(按数据行)分到多个不同的库，每个库只有这个表的部分数据，这些库可以分布在不同服务器，从而使访问压力被多服务器负载，大大提升性能。它不仅需要解决跨库带来的所有复杂问题，还要解决数据路由的问题(数据路由问题后边介绍)。

**水平分表**：可以把一个表的数据(按数据行)分到多个同一个数据库的多张表中，每个表只有这个表的部分数据，这样做能小幅提升性能，它仅仅作为水平分库的一个补充优化。

一般来说，在系统设计阶段就应该根据业务耦合松紧来确定垂直分库，垂直分表方案，在数据量及访问压力不是特别大的情况，首先考虑缓存、读写分离、索引技术等方案。若数据量极大，且持续增长，再考虑水平分库水平分表方案。

# 读写分离

面对日益增加的系统访问量，数据库的吞吐量面临着巨大的瓶颈。对于同一时刻有大量并发读操作和较少的写操作类型的应用来说，将数据库拆分成主库和从库，主库负责处理事务的增删改查操作，从库负责处理查询操作，能够有效地避免由数据更新导致的行锁，使得整个系统的查询性能得到提升

通过一主多从的配置方式，可以将查询请求均匀分散到多个数据副本中，能够进一步地提升系统的处理能力。使用多主多从的方式，不但能提升系统的吞吐量，还可以提升系统的可用性，可以达到任何一个数据库宕机甚至物理损坏时都不影响使用

sharding配置方式可以参照官方文档 master-slave 配置
