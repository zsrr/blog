---
title: Redis会话管理
date: 2017-09-23 16:15:46
tags: [Redis,NoSQL]
categories: Redis

---
本篇介绍基于Redis作后端会话管理，会话管理可以交由Tomcat服务器的Manager来进行管理，但是这样达不到我们自己对于会话的控制。参考博客: [基于Spring及Redis的Token鉴权](http://www.scienjus.com/restful-token-authorization/)

<!--more-->

# 大体设计思路
主要完成的目标是用户登录之后保存登录的状态，这样下次用户登录就不用每次传递密码来进行身份认证。此登录状态应该有有效期限制，如果在很长一段时间内用户没有登录的话，登录状态失效并要求用户重新登录。很简单的功能，Redis提供的键值数据库以及可以通过参数来设置键的过期时间的功能正好能完成这个需求，实现会话管理解耦，下面来看一下代码如何实现。

# 实现
## 配置Jedis客户端
要对Redis进行连接，推荐在框架中直接集成Jedis而不是Spring Data Redis，后者经过读者的测验后发现它有很多蜜汁bug……而且似乎对于集群的支持不怎么好，直接用Jedis一了百了：

    @Bean
    public JedisPool jedisPool() {
        JedisPoolConfig config = new JedisPoolConfig();
        config.setBlockWhenExhausted(true);
        config.setEvictionPolicyClassName("org.apache.commons.pool2.impl.DefaultEvictionPolicy");
        config.setMaxTotal(32);
        config.setMaxIdle(32);
        config.setMaxWaitMillis(300 * 1000);
        config.setMinEvictableIdleTimeMillis(15 * 60 * 1000);
        return new JedisPool(config, "127.0.0.1", 6379, 20 * 1000);
    }
这样在方法中用Jedis的时候直接从连接池获取，用完之后调用close方法，Jedis连接就会返回Jedis连接池，一个好的使用方法利用AutoCloseable特性直接在try语句中使用：

    @Override
    public TokenModel createToken(Long userId) {
        try(Jedis jedis = jedisPool.getResource()) {
            ...
        }
    }

## 生成session
生成一个随机的字符串来标记用户身份，这里算法有很多，可以直接使用jdk的UUID生成算法，为了之后验证功能的设计，这里session前面还要加上用户的id:

    @Override
    public TokenModel createToken(Long userId) {
        try(Jedis jedis = jedisPool.getResource()) {
            final String token = userId + "-" + UUID.randomUUID().toString().replace("-", "");
            jedis.set(getTokenKey(userId), token);
            // 设置有效期，一个星期
            jedis.expire(getTokenKey(userId), 7 * 24 * 60 * 60);
            return new TokenModel(userId, token);
        }
    }

TokenModel:

    public class TokenModel {
        private Long userId;
        private String token;

        public TokenModel(Long userId, String token) {
            this.userId = userId;
            this.token = token;
        }

        public Long getUserId() {
            return userId;
        }

        public void setUserId(Long userId) {
            this.userId = userId;
        }

        public String getToken() {
            return token;
        }

        public void setToken(String token) {
            this.token = token;
        }
    }

## 验证身份
验证就比较简单了：

    @Override
    public boolean checkToken(TokenModel tokenModel) {
        try (Jedis jedis = jedisPool.getResource()) {
            if (tokenModel == null ||
                    tokenModel.getUserId() == null ||
                    tokenModel.getToken() == null) {
                return false;
            }
            String token = jedis.get(getTokenKey(tokenModel.getUserId()));
            if (token == null || !token.equals(tokenModel.getToken())) {
                return false;
            }
            // 每一次验证重新设置过期时间
            jedis.expire(getTokenKey(tokenModel.getUserId()), 7 * 24 * 60 * 60);
            return true;
        }
    }

## 登出
直接删除键值：

    @Override
    public void deleteToken(Long userId) {
        try(Jedis jedis = jedisPool.getResource()) {
            jedis.del(getTokenKey(userId));
        }
    }

