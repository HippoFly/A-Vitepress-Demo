

# 缓存不一致问题

在学习 Redis 的时候，可能只是简单写了一些 Demo，并没有去关注缓存的读写策略，或者说压根不知道这回事。

但是，搞懂 3 种常见的缓存读写策略对于实际工作中使用缓存以及面试中被问到缓存都是非常有帮助的！

下面介绍到的三种模式各有优劣，不存在最佳模式，根据具体的业务场景选择适合自己的缓存读写模式。

## 1.缓存更新策略

### Cache Aside Pattern（旁路缓存模式）

#### 适合场景

Cache Aside Pattern 是我们平时使用比较多的一个缓存读写模式，比较适合**读请求比较多**的场景。

Cache Aside Pattern 中服务端需要同时维系 db 和 cache，并且是以 db 的结果为准。

#### 使用

下面我们来看一下这个策略模式下的缓存读写步骤。

**写**：

- 先更新 db
- 然后直接删除 cache 。

简单画了一张图帮助大家理解写的步骤。

![](https://oss.javaguide.cn/github/javaguide/database/redis/cache-aside-write.png)

**读** :

- 从 cache 中读取数据，读取到就直接返回
- cache 中读取不到的话，就从 db 中读取数据返回
- 再把数据放到 cache 中。

简单画了一张图帮助大家理解读的步骤。

![](https://img2.imgtp.com/2024/05/29/SrHLAyWH.png)

你仅仅了解了上面这些内容的话是远远不够的，我们还要搞懂其中的原理。

比如说面试官很可能会追问：“**在写数据的过程中，可以先删除 cache ，后更新 db 么？**”

**答案：** 那肯定是不行的！因为这样可能会造成 **数据库（db）和缓存（Cache）数据不一致**的问题。

举例：请求 1 先写数据 A，请求 2 随后读数据 A 的话，就很有可能产生数据不一致性的问题。

这个过程可以简单描述为：

> 请求 1 先把 cache 中的 A 数据删除 -> 请求 2 从 db 中读取数据（缓存没有更新的数据）->请求 1 再把 db 中的 A 数据更新

当你这样回答之后，面试官可能会紧接着就追问：“**在写数据的过程中，先更新 db，后删除 cache 就没有问题了么？**”

**答案：** 理论上来说还是可能会出现数据不一致性的问题，不过概率非常小，因为缓存的写入速度是比数据库的写入速度快很多。

举例：请求 1 先读数据 A，请求 2 随后写数据 A，并且数据 A 在请求 1 请求之前不在缓存中的话，也有可能产生数据不一致性的问题。

这个过程可以简单描述为：

> 请求 1 从 db 读数据 A-> 请求 2 更新 db 中的数据 A（此时缓存中无数据 A ，故不用执行删除缓存操作 ） -> 请求 1 将数据 A 写入 cache

#### 缺陷

**缺陷 1：首次请求数据一定不在 cache 的问题**

解决办法：可以将热点数据可以提前放入 cache 中。

**缺陷 2：写操作比较频繁的话导致 cache 中的数据会被频繁被删除，这样会影响缓存命中率 。**

解决办法：

- 数据库和缓存数据强一致场景：更新 db 的时候同样更新 cache，不过我们需要加一个锁/分布式锁来保证更新 cache 的时候不存在线程安全问题。
- 可以短暂地允许数据库和缓存数据不一致的场景：更新 db 的时候同样更新 cache，但是给缓存加一个比较短的过期时间，这样的话就可以保证即使数据不一致的话影响也比较小。

### Read/Write Through Pattern（读写穿透）

#### 使用场景

Read/Write Through Pattern 中服务端把 cache 视为主要数据存储，从中读取数据并将数据写入其中。cache 服务负责将此数据读取和写入 db，从而减轻了应用程序的职责。

这种缓存读写策略小伙伴们应该也发现了在平时在开发过程中非常少见。抛去性能方面的影响，大概率是因为我们经常使用的分布式缓存 Redis 并没有提供 cache 将数据写入 db 的功能。

#### 使用

**写（Write Through）：**

- 先查 cache，cache 中不存在，直接更新 db。
- cache 中存在，则先更新 cache，然后 cache 服务自己更新 db（**同步更新 cache 和 db**）。

简单画了一张图帮助大家理解写的步骤。

![](https://oss.javaguide.cn/github/javaguide/database/redis/write-through.png)

**读(Read Through)：**

- 从 cache 中读取数据，读取到就直接返回 。
- 读取不到的话，先从 db 加载，写入到 cache 后返回响应。

简单画了一张图帮助大家理解读的步骤。

![](https://oss.javaguide.cn/github/javaguide/database/redis/read-through.png)

Read-Through Pattern 实际只是在 Cache-Aside Pattern 之上进行了封装。在 Cache-Aside Pattern 下，发生读请求的时候，如果 cache 中不存在对应的数据，是由客户端自己负责把数据写入 cache，而 Read Through Pattern 则是 cache 服务自己来写入缓存的，这对客户端是透明的。

和 Cache Aside Pattern 一样， Read-Through Pattern 也有首次请求数据一定不再 cache 的问题，对于热点数据可以提前放入缓存中。

### Write Behind Pattern（异步缓存写入）

#### 使用场景

Write Behind Pattern 和 Read/Write Through Pattern 很相似，两者都是由 cache 服务来负责 cache 和 db 的读写。

但是，两个又有很大的不同：**Read/Write Through 是同步更新 cache 和 db，而 Write Behind 则是只更新缓存，不直接更新 db，而是改为异步批量的方式来更新 db。**

很明显，这种方式对数据一致性带来了更大的挑战，比如 cache 数据可能还没异步更新 db 的话，cache 服务可能就就挂掉了。

这种策略在我们平时开发过程中也非常非常少见，但是不代表它的应用场景少，比如消息队列中消息的异步写入磁盘、MySQL 的 Innodb Buffer Pool 机制都用到了这种策略。

Write Behind Pattern 下 db 的写性能非常高，非常适合一些数据经常变化又对数据一致性要求没那么高的场景，比如浏览量、点赞量。

<!-- @include: @article-footer.snippet.md -->

## 2. 分布式锁

使用分布式锁确保在更新缓存和数据库时的操作是原子的。常用的分布式锁实现有：

- **Redis分布式锁**：通过SETNX和EXPIRE命令实现分布式锁，确保在同一时间只有一个客户端可以执行缓存和数据库的更新操作。
- **Zookeeper分布式锁**：Zookeeper提供了一种基于临时有序节点的分布式锁机制。

## 3. 双写一致性

在某些场景下，应用程序需要同时更新缓存和数据库。可以通过以下几种方法解决一致性问题：

- **事务控制**：使用数据库事务确保缓存和数据库的更新在一个事务中执行，保证操作的原子性。
- **消息队列**：将数据库更新操作和缓存更新操作分开，通过消息队列将数据库更新操作发送到一个队列中，消费者读取队列消息并更新缓存。

## 4. 数据变更通知

使用数据库触发器或者变更数据捕获（Change Data Capture，CDC）工具监控数据库的变化，并实时更新缓存。例如：

- **MySQL的binlog**：通过解析MySQL的binlog日志，捕获数据变更并更新缓存。
- **Debezium**：一个开源的CDC平台，支持多种数据库（如MySQL、PostgreSQL等），可以捕获数据变更并发送到Kafka等消息系统，然后消费者更新缓存。

## 5. 缓存失效策略

确保缓存中的数据在数据库更新时失效，常见的方法有：

- **主动失效**：在数据库更新时，主动删除或更新对应的缓存数据。
- **被动失效**：通过设置合理的缓存过期时间，确保缓存中的数据不会长期过期。

## 6. 使用一致性哈希

在分布式缓存环境中，使用一致性哈希算法将数据分布在多个缓存节点上，减少缓存重新分布的次数，减少缓存不一致的概率。
