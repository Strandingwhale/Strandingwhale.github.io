---
title: "重大更新：数据库与protobuf"
date: 2026-10-25T2:00:00+08:00
draft: false
tags: ["C++", "游戏开发", "游戏项目"]
categories: ["游戏开发"]
description: "数据库，proyobuf"
---

# 前言

    今天我完成了数据库链接与protobuf创建，顺带把原本的fmt输出改成了日志输出。把账号、密码检测真正写入了代码中。我几乎重建了server.go和account.go的代码，花了很多很多的时间去改bug。挺掉头发的。payload在login里存的是（账号+密码）

# protobuf

    首先建立proto后最的文件，写入message结构，并自动运行生成.go文件，这一步还挺简单的。

```ProtoBuf
syntax = "proto3";
package api;
option go_package="./;api";

//心跳请求
message PingReq{
  int64 timestamp = 1;
}

//心跳回复
message PingResp{
  int64 timestamp = 1;
}
...
```

# 数据库

    首先MySQL用于保存长时间的数据，账号、密码之类的，Redis用于链接token，热数据缓存。

```Go
// InitDB 打开 MySQL 连接池并验证连通性。
// dsn 形如：root:root123@tcp(127.0.0.1:3306)/abyssal_city?parseTime=true
func InitDB(dsn string) (*sql.DB, error) {
	db, err := sql.Open("mysql", dsn)
	if err != nil {
		return nil, fmt.Errorf("open mysql: %w", err)
	}

	// 连接池参数：够用就行
	db.SetMaxOpenConns(20)           // 最多同时打开的连接数
	db.SetMaxIdleConns(10)           // 空闲时保留的连接数，避免频繁建连
	db.SetConnMaxLifetime(time.Hour) // 连接最长存活时间，防 MySQL 服务端超时踢掉

	// Open 只是配置，Ping 才真正建立连接
	if err := db.Ping(); err != nil {
		return nil, fmt.Errorf("ping mysql: %w", err)
	}
	return db, nil
}
```

```Go
// InitRedis 创建 Redis 客户端并验证连通性
func InitRedis(addr string) (*redis.Client, error) {
	rdb := redis.NewClient(&redis.Options{Addr: addr})

	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()
	if err := rdb.Ping(ctx).Err(); err != nil {
		return nil, fmt.Errorf("ping redis: %w", err)
	}
	return rdb, nil
}
```

    最主要的还是登陆相关的更改，首先定义token有效时间和回包错误码

```Go
const tokenTTL = 2 * time.Hour
const （
	codeOK          = 0
	codeBadParam    = 1 // 参数缺失/格式错误
	codeAccountDup  = 2 // 账号已存在
	codeServerError = 3 // 服务器内部错误
）
```

    issueToken 生成 32 位随机 token，以 token→uid 的形式存入 Redis，到期自动删除

```Go
func issueToken(uid uint64) (string, error) {
	buf := make([]byte, 16)
	if _, err := rand.Read(buf); err != nil {
		return "", err
	}
	token := hex.EncodeToString(buf)
	err := rdb.Set(ctx, "token:"+token, uid, tokenTTL).Err()
	return token, err
}
```

    handleLogin 查库校验密码（bcrypt 比对），成功则签发短期 token 存入 Redis。先传入payload并解码

```go
if err := proto.Unmarshal(payload, &req); err != nil {
		logger.Println("解析 LoginReq 失败:", err)
		return &api.LoginResp{Code: codeBadParam, Msg: "请求格式错误"}
	}
...
err := db.QueryRow("SELECT id, password_hash FROM account WHERE account = ?", req.Account).
	Scan(&uid, &hash)
```

    bcrypt 比对：不解密哈希，而是用同样的算法验证是否匹配

```go
if bcrypt.CompareHashAndPassword([]byte(hash), []byte(req.Password)) != nil {
		return &api.LoginResp{Code: codeBadParam, Msg: "账号密码错误"}
	}
```

    handleRegister 注册：bcrypt 哈希密码后入库，绝不存明文。也是先传入payload

```go
var req api.RegisterReq
	if err := proto.Unmarshal(payload, &req); err != nil {
		logger.Println("解析 RegisterReq 失败:", err)
		return &api.RegisterResp{Code: codeBadParam, Msg: "请求格式错误"}
	}
```

    bcrypt 哈希：每次结果不同（自带随机盐），相同密码也无法从哈希看出

```go
hash, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
	if err != nil {
		logger.Println("bcrypt 哈希失败:", err)
		return &api.RegisterResp{Code: codeServerError, Msg: "服务器繁忙"}
	}
```

    还要检测账号是否冲突

```Go
	_, err = db.Exec("INSERT INTO account (account, password_hash) VALUES (?, ?)", req.Account, string(hash))
	if err != nil {
		// 唯一索引冲突 = 账号已存在
		var me *mysql.MySQLError
		if errors.As(err, &me) && me.Number == 1062 {
			return &api.RegisterResp{Code: codeAccountDup, Msg: "账号已存在"}
		}
		logger.Println("插入账号失败:", err)
		return &api.RegisterResp{Code: codeServerError, Msg: "服务器繁忙"}
	}
```

ok今天到这里，下周继续。
