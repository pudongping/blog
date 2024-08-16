---
title: Go语言中如何扫描Redis中大量的key
author: Alex
top: false
hide: false
cover: false
toc: true
mathjax: false
categories: Go
tags:
  - Go
  - Golang
  - Redis
abbrlink: 9b127c3
date: 2024-08-16 22:16:56
img:
coverImg:
password:
summary:
---

在 Redis 中，当我们需要遍历大量的键时，直接使用 `KEYS` 命令会面临性能瓶颈，尤其是在键数量非常多的情况下。

`KEYS` 命令会一次性返回所有匹配的键，这可能导致 Redis 阻塞，严重影响线上服务的稳定性。为了解决这个问题，Redis 提供了 `SCAN` 命令，用于分批次迭代键，避免一次性返回所有数据。

今天，我们将通过两个示例代码，详细讲解如何在 Go 语言中使用 `SCAN` 命令遍历 Redis 键。

这里我们用到的是 `github.com/go-redis/redis` 包，先创建一个 redis 链接

```go
package redis_demo

import (
	"github.com/go-redis/redis"
)

func RDBClient() (*redis.Client, error) {
	// 创建一个 Redis 客户端
	// 也可以使用数据源名称（DSN）来创建
	// redis://<user>:<pass>@localhost:6379/<db>
	opt, err := redis.ParseURL("redis://localhost:6379/0")
	if err != nil {
		return nil, err
	}
	client := redis.NewClient(opt)

	// 通过 cient.Ping() 来检查是否成功连接到了 redis 服务器
	_, err = client.Ping().Result()
	if err != nil {
		return nil, err
	}

	return client, nil
}
```

## 代码示例 1：使用 `SCAN` 命令的基本迭代方式

首先来看第一个示例代码，这段代码展示了如何通过 `SCAN` 命令遍历 Redis 数据库中的所有键。

```go
package redis_demo

import (
	"fmt"
)

func scanKeysDemo1() {
	var cursor uint64
	rdb, err := RDBClient()
	if err != nil {
		panic(err)
	}

	for {
		var keys []string
		var err error
		// Scan 命令用于迭代数据库中的数据库键。
		keys, cursor, err = rdb.Scan(cursor, "*", 0).Result()
		if err != nil {
			panic(err)
		}

		// 处理 keys
		for _, key := range keys {
			fmt.Printf("key: %s\n", key)
		}

		// 如果 cursor 为 0，说明已经遍历完成，退出循环
		if cursor == 0 {
			break
		}
	}

}
```

### 代码详解：

1. **RDBClient() 函数**: 这段代码假设 `RDBClient()` 是一个返回 Redis 客户端实例的函数，用于连接 Redis 数据库。如果连接失败，程序会直接 panic 终止。

2. **Scan 命令**: `rdb.Scan(cursor, "*", 0).Result()` 是 `SCAN` 命令的核心部分。这里的 `cursor` 用于记录当前扫描的游标位置，`*` 表示匹配所有键，`0` 表示每次扫描返回所有匹配键。在第一次调用时，`cursor` 必须为 `0`，之后 Redis 会返回新的 `cursor`，直到 `cursor` 再次为 `0` 表示迭代结束。

3. **循环扫描**: 使用 `for` 循环不断调用 `SCAN` 命令，每次返回一批键并更新 `cursor`。当 `cursor` 为 `0` 时，退出循环。

4. **键处理**: `for _, key := range keys` 用于遍历当前批次的所有键，并对每个键进行处理（如打印出来）。

这个方法相对直观，但如果 Redis 中的键数量巨大，手动处理游标的方式可能显得繁琐。这时候，可以考虑使用更简便的 `Iterator` 方法。

## 代码示例 2：使用 `Iterator` 简化迭代过程

接下来是第二个示例代码，它展示了如何使用 `Iterator` 方法简化键的遍历过程。

```go
package redis_demo

import (
	"fmt"
)

func scanKeysDemo2() {
	rdb, err := RDBClient()
	if err != nil {
		panic(err)
	}

	// 针对这种需要遍历大量 key 的场景，go-redis 提供了一个更简单的方法 Iterator
	iter := rdb.Scan(0, "*", 50).Iterator()
	for iter.Next() {
		fmt.Printf("key: %s\n", iter.Val())
	}
	if err := iter.Err(); err != nil {
		panic(err)
	}

	// 此外，对于 redis 中的 set、hash、zset 等类型，也可以使用 Iterator 进行遍历
	// 例如：
	// iter := rdb.SScan("set_key", 0, "*", 50).Iterator()
	// iter := rdb.HScan("hash_key", 0, "*", 50).Iterator()
	// iter := rdb.ZScan("zset_key", 0, "*", 50).Iterator()
}
```

### 代码详解：

1. **使用 Iterator**: 与前一个示例不同，这里使用了 `Iterator` 迭代器。`rdb.Scan(0, "*", 50).Iterator()` 创建了一个迭代器，每次返回 50 个匹配的键。这样无需手动处理 `cursor`，简化了遍历过程。

2. **迭代与处理**: `for iter.Next()` 是一个简洁的循环，用于遍历所有匹配的键。当 `iter.Next()` 返回 `false` 时，表示遍历结束。`iter.Val()` 返回当前键的值。

3. **错误处理**: 在循环结束后，检查 `iter.Err()` 是否为 `nil`，以确保遍历过程中没有出现错误。

4. **扩展功能**: 此外，`Iterator` 方法不仅适用于遍历键，也可用于遍历 Redis 中的集合（`Set`）、哈希（`Hash`）、有序集合（`ZSet`）等数据结构。通过将 `Scan` 换成 `SScan`、`HScan` 或 `ZScan`，就能遍历对应的数据结构。

## 总结

这篇文章介绍了如何在 Go 语言中使用 `SCAN` 命令遍历 Redis 键，并比较了手动处理 `cursor` 和使用 `Iterator` 的两种方式。对于 Redis 新手来说，了解 `SCAN` 命令的用法非常重要，它不仅帮助你避免了使用 `KEYS` 命令可能带来的性能问题，还让你能够更高效地遍历 Redis 数据。

如果你觉得文章有帮助，欢迎点赞、转发，让更多人掌握 Redis 的这些实用技巧！😊