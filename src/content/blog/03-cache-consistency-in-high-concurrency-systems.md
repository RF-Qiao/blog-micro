---
title: "高并发系统中的缓存一致性实战：从 Cache Aside 到可靠补偿"
description: "系统梳理 Cache Aside、延迟双删、消息驱动失效、热点保护与版本控制策略，帮助后端工程师在性能与一致性之间做出可维护的工程决策。"
date: "2026-06-28"
tags:
  - JAVA
  - Redis
  - 架构设计
  - 高并发
---

在后端系统里，缓存几乎是绕不过去的话题。只要业务进入增长期，数据库迟早会扛不住所有读流量；而只要你引入缓存，就一定会遇到另一个更难的问题: **缓存和数据库到底怎样才能尽量保持一致？**

很多线上事故并不是因为“没有缓存”，而是因为“缓存用了，但一致性策略设计得不完整”。典型现象包括:

- 用户刚修改的数据查出来还是旧值
- 订单状态已经更新，但页面还显示处理中
- 热点 key 在失效瞬间把数据库打穿
- 多实例并发写入后，缓存反而比不加缓存更乱

这篇文章不讲概念堆砌，而是从真实工程视角出发，把缓存一致性问题拆成三个层次:

1. 缓存模式怎么选
2. 写路径怎么控风险
3. 高并发和故障场景下如何兜底

如果你正在做 Java、Spring Boot、Redis 相关系统，这些内容基本都会直接用得上。

在进入正文之前，可以先给这篇文章定一个阅读预期: 我们讨论的不是“教科书里最标准的唯一答案”，而是线上系统真正会遇到的取舍问题。因为缓存一致性从来不是某一个 API 的选择题，而是一整条读写链路的工程平衡。

## 一、先统一一个认知: 缓存一致性很少是“绝对一致”

在讨论方案之前，我们先把目标讲清楚。大多数业务系统里的 Redis 缓存，并不是为了替代数据库，而是为了降低延迟和保护存储层。因此，缓存一致性通常追求的是:

- **读写后尽快收敛**
- **大部分时间返回正确结果**
- **在异常情况下可恢复、可观测、可补偿**

也就是说，工程上更常见的目标其实是**最终一致性**，而不是分布式事务级别的强一致性。

原因很简单:

- Redis 和 MySQL 通常不是一个事务系统
- 网络抖动、进程重启、消息延迟都会制造时间窗口
- 你越追求绝对一致，系统复杂度和写延迟就越高

真正成熟的设计，不是幻想“完全没有不一致”，而是要明确:

- 不一致可能发生在哪些窗口
- 这些窗口能持续多久
- 发生后系统如何自动恢复
- 业务能否接受这个级别的风险

很多团队在评审方案时会直接问一句: “这个方案能保证一致吗？” 更好的问法其实应该是:

- 正常情况下多久能收敛？
- 极端情况下最坏会不一致多久？
- 是单个 key 风险，还是一类业务整体风险？
- 出现问题后靠什么机制恢复？

当你开始这样提问，缓存设计才算真正进入工程讨论阶段。

## 二、最常见的模式: Cache Aside

绝大多数互联网业务，最终都会回到 `Cache Aside`，也叫**旁路缓存模式**。

它的基本流程很简单。

### 读流程

```text
先读缓存 -> 缓存命中直接返回
          -> 缓存未命中则查数据库 -> 回填缓存 -> 返回结果
```

### 写流程

```text
先更新数据库 -> 再删除缓存
```

对应代码大概像这样:

```java
public UserDTO getUser(Long userId) {
    String key = "user:" + userId;
    String cached = redisTemplate.opsForValue().get(key);
    if (cached != null) {
        return JsonUtils.toObject(cached, UserDTO.class);
    }

    User user = userMapper.selectById(userId);
    if (user == null) {
        redisTemplate.opsForValue().set(key, "NULL", Duration.ofMinutes(2));
        return null;
    }

    UserDTO dto = UserDTO.from(user);
    redisTemplate.opsForValue().set(key, JsonUtils.toJson(dto), Duration.ofMinutes(30));
    return dto;
}

@Transactional
public void updateUser(UserUpdateCommand command) {
    userMapper.update(command);
    redisTemplate.delete("user:" + command.userId());
}
```

这个模式流行，不是因为它完美，而是因为它在复杂度、性能和可维护性之间比较平衡。

### 它为什么比“先删缓存再更新数据库”更常见？

因为“先删缓存，再更新数据库”有一个经典并发问题:

1. 线程 A 删除缓存
2. 线程 A 还没来得及更新数据库
3. 线程 B 读缓存未命中，去查数据库，读到旧值
4. 线程 B 把旧值重新写回缓存
5. 线程 A 更新数据库成功

结果就是: **数据库新了，缓存却旧了。**

所以更常见的建议是:

- **更新数据库**
- **删除缓存**

这样即使后续有读请求进来，也会从数据库读到新值并回填缓存。

## 三、为什么“先更新数据库，再删缓存”仍然不是完美方案？

很多人学到这里会觉得问题已经结束了，但真正的麻烦才刚开始。

看下面这个场景:

1. 线程 A 更新数据库成功
2. 线程 A 删除缓存时失败了
3. 缓存里旧值继续存活到 TTL 结束

这时，你仍然会持续读到脏数据。

也就是说，`更新 DB + 删除缓存` 只是把不一致窗口缩小了，但没有彻底消灭。真正的问题变成了:

- 删缓存失败怎么办？
- 并发读刚好落在删除前后怎么办？
- 同一个 key 被高频更新时怎么办？

这才是缓存一致性设计的核心。

换句话说，缓存一致性讨论到最后，真正比拼的不是“谁知道更多名词”，而是谁能把这些失败场景提前设计进去。

## 四、延迟双删: 不是银弹，但在一些场景下有价值

延迟双删的思路是:

1. 更新数据库
2. 删除缓存
3. 等待一小段时间
4. 再删一次缓存

示意代码:

```java
@Transactional
public void updateUser(UserUpdateCommand command) {
    userMapper.update(command);
    String key = "user:" + command.userId();
    redisTemplate.delete(key);

    cacheInvalidationExecutor.submit(() -> {
        try {
            Thread.sleep(500);
            redisTemplate.delete(key);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    });
}
```

它要解决的是这种窗口:

1. 线程 A 更新数据库
2. 线程 B 在缓存删除前发起读请求
3. 线程 B 读到旧缓存，或者查库后把旧值重新回填
4. 第二次删除把这个旧缓存清掉

### 延迟双删的优点

- 实现成本低
- 对已有 Cache Aside 方案改动小
- 能降低并发窗口下旧值回灌的问题

### 它的局限也非常明显

- 延迟多久很难精确选择
- 线程池任务丢失会导致第二次删除失效
- 对跨服务、多机房、消息堆积场景帮助有限
- 它本质上还是补偿手段，不是严格保证

所以我的建议是: **延迟双删可以用，但不要把它当成主方案，只能把它看成删缓存链路上的一次增强。**

## 五、工程上更稳的做法: 让“删缓存”可重试、可补偿、可追踪

如果你的业务已经比较核心，只靠应用线程里顺手 `delete(key)` 往往不够。更成熟的设计应该把“缓存失效”当成一条正式链路来治理。

一个更可靠的思路是:

1. 业务事务先提交数据库
2. 记录一条缓存失效事件
3. 异步消费者执行删除缓存
4. 删除失败自动重试
5. 对超过阈值的失败事件落库告警

这类设计通常会结合:

- 本地消息表
- MQ 异步投递
- 定时补偿任务
- 幂等删除逻辑

### 一个常见落地模型: 事务消息/本地消息表

```text
业务更新数据库
    -> 同事务写入 message/outbox 表
    -> 事务提交成功
    -> 异步任务扫描 outbox
    -> 投递“删除缓存”消息
    -> 消费者执行 delete(key)
    -> 成功后回写消息状态
```

这样设计的好处是:

- 业务数据变更和“待失效事件”在一个数据库事务里
- 即使应用实例在提交后立刻挂掉，补偿任务仍然能继续推进
- 你可以统计失败次数、积压量、平均延迟，具备可观测性

这类方案比“在业务代码里直接删一下缓存”重，但如果是订单、库存、资产、计费这类核心域，通常是值得的。

很多团队一开始会觉得这套方案“有点重”，但只要线上真的出现过一次删缓存失败引发的数据错误，你就会发现: 相比事故后的排查、补数和业务解释，这点治理成本通常非常划算。

## 六、不要只盯着一致性，还要一起处理三个高频问题

缓存系统在线上真正难的地方，从来不是某一个理论模型，而是多类问题交织在一起。除了数据一致性，以下三个问题往往会同时出现。

### 1. 缓存穿透

查询一个根本不存在的数据，每次都会打到数据库。

常见处理方式:

- 缓存空对象
- 布隆过滤器
- 接口层参数校验

示例:

```java
public ProductDTO getProduct(Long productId) {
    String key = "product:" + productId;
    String value = redisTemplate.opsForValue().get(key);
    if ("NULL".equals(value)) {
        return null;
    }
    if (value != null) {
        return JsonUtils.toObject(value, ProductDTO.class);
    }

    Product product = productMapper.selectById(productId);
    if (product == null) {
        redisTemplate.opsForValue().set(key, "NULL", Duration.ofMinutes(3));
        return null;
    }

    ProductDTO dto = ProductDTO.from(product);
    redisTemplate.opsForValue().set(key, JsonUtils.toJson(dto), Duration.ofMinutes(20));
    return dto;
}
```

### 2. 缓存击穿

某个热点 key 失效瞬间，大量请求同时回源数据库。

常见处理方式:

- 分布式锁
- 本地 singleflight/请求合并
- 热点数据永不过期，改为后台异步刷新

示例:

```java
public ProductDTO getHotProduct(Long productId) {
    String key = "hot:product:" + productId;
    String cached = redisTemplate.opsForValue().get(key);
    if (cached != null) {
        return JsonUtils.toObject(cached, ProductDTO.class);
    }

    String lockKey = "lock:" + key;
    boolean locked = Boolean.TRUE.equals(
        redisTemplate.opsForValue().setIfAbsent(lockKey, "1", Duration.ofSeconds(10))
    );

    if (!locked) {
        try {
            Thread.sleep(50);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return getHotProduct(productId);
    }

    try {
        Product product = productMapper.selectById(productId);
        ProductDTO dto = ProductDTO.from(product);
        redisTemplate.opsForValue().set(key, JsonUtils.toJson(dto), Duration.ofMinutes(30));
        return dto;
    } finally {
        redisTemplate.delete(lockKey);
    }
}
```

### 3. 缓存雪崩

大量 key 在同一时间失效，导致流量集体回源。

常见处理方式:

- TTL 增加随机抖动
- 不同业务 key 分批过期
- 多级缓存
- 服务降级与限流

例如:

```java
int baseTtlSeconds = 1800;
int randomSeconds = ThreadLocalRandom.current().nextInt(0, 300);
redisTemplate.opsForValue().set(key, value, Duration.ofSeconds(baseTtlSeconds + randomSeconds));
```

## 七、更新缓存还是删除缓存？大多数情况下优先删

这是一个经常被问到的问题。

很多同学的第一反应是: “数据库都更新了，为什么不直接把新值同步写到缓存里？”

理论上可以，但工程上通常优先**删除缓存**，原因有三个。

### 1. 你未必知道缓存里存的最终视图是什么

数据库表结构和缓存对象结构往往不是一一对应的。缓存可能是:

- 多表 join 结果
- 聚合视图
- 带权限裁剪的结果
- 经过排序、过滤、字段转换后的 DTO

如果你在写请求路径里直接更新缓存，就必须重复一遍这套组装逻辑，复杂度会明显上升。

### 2. 删除比更新更天然幂等

`delete(key)` 执行多次几乎没有副作用，而“更新缓存”需要保证:

- 数据版本正确
- 并发写顺序正确
- 部分字段更新不会覆盖新值

### 3. 删除更适合异步化和补偿

删除缓存是一种很容易做成事件驱动的动作，而更新缓存往往需要额外查数、额外计算，失败补偿也更重。

所以除非是特别明确的场景，例如:

- 实时排行榜
- 固定格式的计数器
- 强依赖写后立即读且结构简单的数据

否则优先推荐: **更新数据库，删除缓存，让读请求自然回填。**

## 八、如果业务对一致性要求更高，可以引入版本号

当一个 key 被高频并发写入时，仅靠“删缓存”有时还不够稳。此时可以考虑引入**数据版本号**或**更新时间戳**，避免旧值覆盖新值。

典型做法是:

- 数据库记录 `version` 字段
- 写入成功后版本号递增
- 缓存对象也携带 version
- 回填缓存时做版本比较

伪代码示意:

```java
public void rebuildCache(Long userId) {
    User user = userMapper.selectById(userId);
    CacheUser cacheUser = CacheUser.from(user);

    String key = "user:" + userId;
    CacheUser oldValue = redisClient.getObject(key, CacheUser.class);

    if (oldValue == null || cacheUser.version() >= oldValue.version()) {
        redisClient.setObject(key, cacheUser, Duration.ofMinutes(30));
    }
}
```

这不能解决所有问题，但能显著减少“旧回填覆盖新缓存”的概率，尤其适合:

- 高并发资料更新
- 状态机型业务
- 多服务共同写同一类缓存

## 九、缓存一致性方案怎么选？给一个实用决策表

| 业务场景 | 推荐方案 | 说明 |
| --- | --- | --- |
| 普通资料查询 | Cache Aside + 更新 DB 后删缓存 | 成本低，适合绝大多数 CRUD 系统 |
| 写入频率不高但读很多 | Cache Aside + TTL 抖动 + 空值缓存 | 兼顾性能和稳定性 |
| 热点 key 明显 | Cache Aside + singleflight/锁 + 后台刷新 | 重点解决击穿 |
| 核心交易数据 | DB 更新 + Outbox/MQ 删除缓存 + 重试补偿 | 提高删除链路可靠性 |
| 多服务并发写入 | 删除缓存 + 版本号控制 + 异步补偿 | 降低旧值覆盖风险 |
| 极端强一致诉求 | 减少缓存参与范围，必要时直查数据库 | 不要硬把 Redis 做成事务系统 |

工程上最怕的不是“方案不高级”，而是**方案和业务重要性不匹配**。对一个后台配置页面上价值不高的数据，可能简单删缓存就够了；但对库存可售量这种核心数据，链路治理就必须更重。

所以做方案选型时，建议始终把业务分级带上:

- 普通资料类数据，优先低复杂度
- 高频热点数据，优先防击穿和削峰
- 核心交易数据，优先补偿和可靠性
- 极端强一致场景，优先减少缓存参与范围

## 十、线上落地时，建议重点监控这些指标

如果没有监控，再好的设计也只是“感觉可靠”。

建议至少监控以下指标:

- 缓存命中率
- 缓存平均延迟、P99 延迟
- 缓存删除失败次数
- 删除补偿积压量
- 热点 key QPS
- 数据库回源流量
- 空值缓存占比
- 锁等待时间与获取失败率

如果你用了消息驱动删除缓存，还应该额外看:

- 消息投递延迟
- 消费失败重试次数
- 死信队列数量
- outbox 表未处理记录数

只有这些指标可见，你才能真正回答两个问题:

1. 当前一致性窗口是否可接受？
2. 系统是否正在朝着异常状态滑落？

## 十一、给一套我更推荐的实践组合

如果你希望在大多数中后台系统里找到一个“足够稳、实现成本也合理”的平衡点，我更推荐下面这套组合:

1. 读路径使用 Cache Aside
2. 写路径坚持“更新数据库后删除缓存”
3. 为缓存 TTL 加随机抖动
4. 对不存在数据做短 TTL 空值缓存
5. 对热点 key 做 singleflight 或分布式锁保护
6. 对核心业务引入 outbox + MQ 的异步失效补偿
7. 对高并发更新对象增加 version 字段
8. 为整条链路补齐监控、告警和定时补偿任务

这套方案不追求理论最强，但在真实业务里通常更容易维护、更容易排障，也更容易被团队长期执行。

## 总结

缓存一致性没有“放之四海而皆准”的银弹，真正重要的是你能不能把问题拆开:

- **模式层**: 选对 Cache Aside 这样的主模式
- **链路层**: 让删缓存具备重试和补偿能力
- **并发层**: 同时处理穿透、击穿、雪崩和旧值回灌
- **治理层**: 用监控和告警把风险显性化

如果只记一个结论，我会建议你记住这句:

> 大多数业务里，最实用的方案不是“强行做绝对一致”，而是“用简单主链路加可靠补偿，把不一致窗口控制在业务可接受范围内”。

这，才是缓存一致性在工程世界里的真正答案。
