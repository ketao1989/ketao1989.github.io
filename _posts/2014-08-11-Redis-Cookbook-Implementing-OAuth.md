---
layout: post  
categories: 
  - Redis
  - Redis Cookbook
title: "Redis Cookbook 之 基于Redis 实现Oauth协议"
date: 2014-08-11 22:21:35 +0800
comments: true
---

## <a id="Problem">问题</a>

在我们案例里，我们将实现一个数据模型和交互来支持Oauth v1.0a API。一般常常是基于MySQL或者其他的RDBMS实现，但是我们可以利用Redis的数据结构更高效的实现Oauth协议。

## <a id="Solution">解决方法</a>

我们不会去实现Oauth的API或者交互。这里我们仅仅感兴趣的是这种类型场景里所需要的数据。我们将在Redis里面存储五种类型的数据：

 - consumer keys
 - consumer secrets
 - request tokens
 - access tokens
 - nonces
 
 所以我们的需求如下：通过一对`key`和`secret`标识的应用（客户）。这些客户根据他们的需求会有许多次的请求和访问token，并且每个用户/时间戳对是唯一的。
 
根据他们的要求和交互情况，这种类型的数据可以被存储在`hashes`、`sets`和`strings`。

## <a id="Discussion">讨论</a>

###  初始设置

一开始，在客户发出一个请求之前，他们必须输入他们的数据。我们将这个数据和客户信息放入hash中。当他注册我们系统的时候，key是我们为特定的商户存储的数据之一：

<!-- more -->

    HMSET /consumers/key:dpf43f3p2l4k3l03 secret kd94hf93k423kf44 created_at 201103060000 redirect_url http://www.example.com/oauth_redirect name test_application
    
   如同命令所示，这个hash包含它的正常的数据，并且可以随着时间进行扩展。这样，和Memcache一样，或者以不同的keys来存储所有的值，或者存储像JSON一样的数据。
   
    HSET hash-name key value
        Sets a value on a hash with the given key. As with other Redis commands, if the hash doesn’t exist, it’s created.
        
    HMSET hash-name key1 value1 [key2 value2 ...]
        Allows you to set several values in a hash with a single command

###  获取一个请求token

为了获取一个请求token，客户发送他们的key、时间戳、唯一生成随机数、回调url和一个请求的签名（使用客户的secret对请求路径和参数hash计算结果）。

API提供者需要验证签名是否正确，检查随机数/时间戳在之前是否使用过，并且生成一个新的请求token。

为了做到这样，server需要获取客户的数据：

    HGETALL /consumers/key:dpf43f3p2l4k3l03

然后检查这个随机数是否以前使用过：

    SADD /nonces/key:dpf43f3p2l4k3l03/timestamp:20110306182600 dji430splmx33448
    





