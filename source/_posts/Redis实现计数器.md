---
title: Redis实现计数器
date: 2017-09-25 15:30:13
tags: [Redis,NoSQL]
categories: Redis

---
本篇概述Redis的使用场景及其代码实现，参考书籍[Redis实战](https://item.jd.com/11791607.html)

<!--more-->

# 计数器设计
有时可能要检测一段时间内后台的访问量，并根据趋势进行预测，若后台的访问频率非常高，例如20秒内有5000+次访问，那么采用Redis来做计数器是非常合适的(Redis高速读写特性令此时关系数据库显得笨拙)。

要达到之前的需求，实现对网站访问量的监控和预测，可以定义好多不同的频率的技术器，并在技术器的名称当中进行标注，例如分别记录5秒内访问量，1分钟内访问量……

计数器可以采用散列来实现，键值为Unix时间戳，值为在该时间片之内网站/后台的访问量。为了清理过期的数据，可以采用一个有序集合来保存计数器的信息，键值为计数器的名称，权值都设置成0，这样Redis在返回的时候便会按照计数器的名称来对返回结果进行排序。

# 实现
下面来看一下对上述功能的Java实现：

## 更新及得到计数器

    private static final int[] PRECISIONS = new int[]{ 5, 60, 300, 3600, 18000, 86400 };

    public void updateCounter() {
        long now = System.currentTimeMillis() / 1000;
        // 减少和服务器的直接通信，这里使用非事务型流水线来进行通信
        Pipeline pipeline = jedis.pipelined();

        for(int precision : PRECISIONS) {
            int pnow = ((int) now / precision) * precision;
            String hashKey = precision + ":hits";
            pipeline.zadd("known:", 0.0, hashKey);
            pipeline.hincrBy("count:" + hashKey, pnow + "", 1);
        }

        pipeline.sync();
    }

    public Map<Long, Long> getCounter(int precision) {
        // 检查参数的合法性
        checkPrecision(precision);
        Map<String, String> data = jedis.hgetAll("count:" + precision + ":hits");
        return transformData(data);
    }

    private static Map<Long, Long> transformData(Map<String, String> data) {
        // 排序方式，默认为升序
        Map<Long, Long> result = new TreeMap<>(Long::compareTo);
        for (Map.Entry<String, String> entry : data.entrySet()) {
            result.put(Long.parseLong(entry.getKey()), Long.parseLong(entry.getValue()));
        }
        return result;
    }

    // 检验参数的有效性
    private static void checkPrecision(int precision) {
        for (int p : PRECISIONS) {
            if (p == precision)
                return;
        }
        throw new IllegalArgumentException();
    }

## 清理过期数据
为了防止数据过大而导致内存吃紧，定期清理过期数据是十分有必要的，在这里需要思考几个问题：

- 不能使用EXPIRE特性，因为此特性只能应用于整个键值，并不能对内部数据应用。
- 任何时候有可能有新的计数器被添加进来。
- 在同一个时间内可能有多个清理操作同时运行。
- 对于更新频率为一天的计数器执行一分钟一次清理为垃圾操作。
- 如果一个计数器不包含任何数据，那么不应该对其执行清理。

可以采用守护进程来进行过期数据清理，对于一分钟更新一次的计数器一分钟清理一次，5分钟的5分钟清理一次……清理之后每个计数器最多维持120个有效信息：

    // 要保存的记录数量
    private static final int COUNT = 120;

    private volatile boolean quit = false;

    public void stopCleaning() {
        quit = true;
    }

    public void clean() {
        Pipeline pipeline = jedis.pipelined();
        int passes = 0;
        // 外层循环 一分钟一次
        while (!quit) {
            long start = System.currentTimeMillis() / 1000;
            int index = 0;
            // 遍历整个"known:"有序列表
            while (index < jedis.zcard("known:") && !quit) {
                Set<String> hashes = jedis.zrange("known", index, index);
                index += 1;
                // 此时没有计数器
                if (hashes == null || hashes.isEmpty()) {
                    break;
                }
                String hash = (String) hashes.toArray()[0];
                // 获取精度
                int prec = Integer.parseInt(hash.split(":")[0]);
                // 看此时是否需要进行清理
                int bprec = prec / 60 == 0 ? 1 : prec / 60;
                // 此时不需要进行清理，遍历下一个
                if (passes % bprec != 0) {
                    continue;
                }

                // 用来看哪些数据需要删除，在此时间之前的统统删除，最多保留COUNT个记录
                long cutoff = System.currentTimeMillis() / 1000 - COUNT * prec;
                String hkey = "count:" + hash;
                Set<String> keys = jedis.hkeys(hkey);
                Set<Long> times = transformToLong(keys);
                // 需要删除的
                Set<Long> removals = ((SortedSet<Long>) times).headSet(cutoff);
                
                if (!removals.isEmpty()) {
                    for (Long removal : removals) {
                        pipeline.hdel(hkey, removal.toString());
                    }
                    pipeline.sync();
                }
            }
            passes += 1;
            // 计算一共用了多长时间
            long duration = System.currentTimeMillis() / 1000 - start + 1;
            try {
                // 超过一分钟则只休眠一秒，否则休眠剩下的时间
                Thread.sleep(duration > 60 ? 1000 : (60 - duration) * 1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

最后，附上完整代码：

    public class Counter {

        private Jedis jedis;
        private volatile boolean quit = false;

        public Counter() {
            jedis = new Jedis("127.0.0.1", 6379);
        }

        // 精度
        private static final int[] PRECISIONS = new int[]{5, 60, 300, 3600, 18000, 86400};
        // 要保存的记录数量
        private static final int COUNT = 120;

        public void updateCounter() {
            long now = System.currentTimeMillis() / 1000;
            // 减少和服务器的直接通信，这里使用非事务型流水线来进行通信
            Pipeline pipeline = jedis.pipelined();

            for (int precision : PRECISIONS) {
                int pnow = ((int) now / precision) * precision;
                String hashKey = precision + ":hits";
                pipeline.zadd("known:", 0.0, hashKey);
                pipeline.hincrBy("count:" + hashKey, pnow + "", 1);
            }

            pipeline.sync();
        }

        public Map<Long, Long> getCounter(int precision) {
            // 检查参数的合法性
            checkPrecision(precision);
            Map<String, String> data = jedis.hgetAll("count:" + precision + ":hits");
            return transformData(data);
        }

        private static Map<Long, Long> transformData(Map<String, String> data) {
            Map<Long, Long> result = new TreeMap<>(Long::compareTo);
            for (Map.Entry<String, String> entry : data.entrySet()) {
                result.put(Long.parseLong(entry.getKey()), Long.parseLong(entry.getValue()));
            }
            return result;
        }

        private static void checkPrecision(int precision) {
            for (int p : PRECISIONS) {
                if (p == precision)
                    return;
            }
            throw new IllegalArgumentException();
        }

        public void stopCleaning() {
            quit = true;
        }

        public void clean() {
            Pipeline pipeline = jedis.pipelined();
            int passes = 0;
            while (!quit) {
                long start = System.currentTimeMillis() / 1000;
                int index = 0;
                while (index < jedis.zcard("known:") && !quit) {
                    Set<String> hashes = jedis.zrange("known", index, index);
                    index += 1;
                    // 此时没有计数器
                    if (hashes == null || hashes.isEmpty()) {
                        break;
                    }
                    String hash = (String) hashes.toArray()[0];
                    // 获取精度
                    int prec = Integer.parseInt(hash.split(":")[0]);
                    // 看此时是否需要进行清理
                    int bprec = prec / 60 == 0 ? 1 : prec / 60;
                    // 此时不需要进行清理，遍历下一个
                    if (passes % bprec != 0) {
                        continue;
                    }

                    // 用来看哪些数据需要删除，在此时间之前的统统删除，最多保留COUNT个记录
                    long cutoff = System.currentTimeMillis() / 1000 - COUNT * prec;
                    String hkey = "count:" + hash;
                    Set<String> keys = jedis.hkeys(hkey);
                    Set<Long> times = transformToLong(keys);
                    // 需要删除的
                    Set<Long> removals = ((SortedSet<Long>) times).headSet(cutoff);

                    if (!removals.isEmpty()) {
                        for (Long removal : removals) {
                            pipeline.hdel(hkey, removal.toString());
                        }
                        pipeline.sync();
                    }
                }
                passes += 1;
                long duration = System.currentTimeMillis() / 1000 - start + 1;
                try {
                    Thread.sleep(duration > 60 ? 1000 : (60 - duration) * 1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }

        private static Set<Long> transformToLong(Set<String> keys) {
            Set<Long> result = new TreeSet<>(Long::compareTo);
            for (String key : keys) {
                result.add(Long.parseLong(key));
            }

            return result;
        }

        public void close() {
            if (jedis != null) {
                jedis.close();
            }
        }

    }

## 与Java Web项目集成
下面来看一下怎么和刚才的计数器怎么和Java Web项目进行集成，首先将刚才的代码稍作修改：

    public class RedisAccessCounter implements Runnable {
        private Jedis jedis;
        private volatile boolean quit = false;
        private Thread cleanThread;

        public RedisAccessCounter(String host, int port) {
            jedis = new Jedis(host, port);
        }

        // 精度
        private static final int[] PRECISIONS = new int[]{5, 60, 300, 3600, 18000, 86400};
        // 要保存的记录数量
        private static final int COUNT = 120;

        public void updateCounter() {
            long now = System.currentTimeMillis() / 1000;
            // 减少和服务器的直接通信，这里使用非事务型流水线来进行通信
            Pipeline pipeline = jedis.pipelined();

            for (int precision : PRECISIONS) {
                int pnow = ((int) now / precision) * precision;
                String hashKey = precision + ":hits";
                pipeline.zadd("known:", 0.0, hashKey);
                pipeline.hincrBy("count:" + hashKey, pnow + "", 1);
            }

            pipeline.sync();
        }

        public Map<Long, Long> getCounter(int precision) {
            // 检查参数的合法性
            checkPrecision(precision);
            Map<String, String> data = jedis.hgetAll("count:" + precision + ":hits");
            return transformData(data);
        }

        private static Map<Long, Long> transformData(Map<String, String> data) {
            Map<Long, Long> result = new TreeMap<>(Long::compareTo);
            for (Map.Entry<String, String> entry : data.entrySet()) {
                result.put(Long.parseLong(entry.getKey()), Long.parseLong(entry.getValue()));
            }
            return result;
        }

        private static void checkPrecision(int precision) {
            for (int p : PRECISIONS) {
                if (p == precision)
                    return;
            }
            throw new IllegalArgumentException();
        }

        public void stopCleaning() {
            quit = true;
        }

        public void clean() {
            cleanThread = new Thread(this);
            cleanThread.setDaemon(true);
            cleanThread.start();
        }

        private static Set<Long> transformToLong(Set<String> keys) {
            Set<Long> result = new TreeSet<>(Long::compareTo);
            for (String key : keys) {
                result.add(Long.parseLong(key));
            }

            return result;
        }

        public void close() {
            stopCleaning();
            if (jedis != null) {
                jedis.close();
            }
        }

        @Override
        public void run() {
            Pipeline pipeline = jedis.pipelined();
            int passes = 0;
            while (!quit) {
                long start = System.currentTimeMillis() / 1000;
                int index = 0;
                while (index < jedis.zcard("known:") && !quit) {
                    Set<String> hashes = jedis.zrange("known", index, index);
                    index += 1;
                    // 此时没有计数器
                    if (hashes == null || hashes.isEmpty()) {
                        break;
                    }
                    String hash = (String) hashes.toArray()[0];
                    // 获取精度
                    int prec = Integer.parseInt(hash.split(":")[0]);
                    // 看此时是否需要进行清理
                    int bprec = prec / 60 == 0 ? 1 : prec / 60;
                    // 此时不需要进行清理，遍历下一个
                    if (passes % bprec != 0) {
                        continue;
                    }

                    // 用来看哪些数据需要删除，在此时间之前的统统删除，最多保留COUNT个记录
                    long cutoff = System.currentTimeMillis() / 1000 - COUNT * prec;
                    String hkey = "count:" + hash;
                    Set<String> keys = jedis.hkeys(hkey);
                    Set<Long> times = transformToLong(keys);
                    // 需要删除的
                    Set<Long> removals = ((SortedSet<Long>) times).headSet(cutoff);

                    if (!removals.isEmpty()) {
                        for (Long removal : removals) {
                            pipeline.hdel(hkey, removal.toString());
                        }
                        pipeline.sync();
                    }
                }
                passes += 1;
                long duration = System.currentTimeMillis() / 1000 - start + 1;
                try {
                    Thread.sleep(duration > 60 ? 1000 : (60 - duration) * 1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
最好的方法是定义一个Filter，然后每当访问指定页面的时候更新计数器：

    public class AccessCounterFilter implements Filter {

        private RedisAccessCounter counter;

        @Override
        public void init(FilterConfig filterConfig) throws ServletException {
            String host = filterConfig.getInitParameter("jedis.host");
            String port = filterConfig.getInitParameter("jedis.port");
            counter = new RedisAccessCounter(host, Integer.parseInt(port));
            counter.clean();
        }

        @Override
        public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
            counter.updateCounter();
            filterChain.doFilter(servletRequest, servletResponse);
        }

        @Override
        public void destroy() {
            counter.close();
        }
    }

web.xml:

    <filter>
        <filter-name>access-counter-filter</filter-name>
        <filter-class>com.stephen.zhihu.base.AccessCounterFilter</filter-class>
        <init-param>
            <param-name>jedis.host</param-name>
            <param-value>127.0.0.1</param-value>
        </init-param>
        <init-param>
            <param-name>jedis.port</param-name>
            <param-value>6379</param-value>
        </init-param>
    </filter>

    <filter-mapping>
        <filter-name>access-counter-filter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
