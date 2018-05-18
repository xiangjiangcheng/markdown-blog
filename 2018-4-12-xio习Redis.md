## Redis入门到放弃（使用篇+代码实例）
---

### set（有序集合）
1. 认识：

2. ##命令：
    这里参数关键字都比较多，所以下面开始列举的命令，`关键字`都使用`大写`
    有序集合中,有key(键)、score（分数）、member（元素）三个参数！其中member为元素，score为member对应的分数。也就是说一个key里面有多个member，一个member又对应了一个score。后面我们根据分数的范围获取集合及其他操作（类似于筛选）。
    1、 新增元素 
    **ZADD key score member [score member ...]**
    ```js {.line-numbers}
    127.0.0.1:6379> zadd scoreboard 89 tom
    (integer) 1     //添加一个
    127.0.0.1:6379> zadd scoreboard 70 peter 100 david
    (integer) 2     //添加多个
    127.0.0.1:6379> zrange scoreboard 0 -1 withscores     
    1) "peter"      //带分数输出
    2) "70"
    3) "tom"
    4) "89"
    5) "david"
    6) "100"
    ```

    2、获得元素的分数
    **ZSCORE key member**
    ```js
    127.0.0.1:6379> zscore scoreboard peter
    "76"                                                    
    ```
    
    3、获得排名在某个范围的元素列表
    **ZRANGE key start stop [WITHSCORE]**
    **ZREVRANGE key start stop [WITHSCORE]**
    >ZRANGE命令会按照元素分数的从小到大顺序返回索引从start到stop之间所有的元素（包含两端）。ZRANGE与LRANGE命令相似，索引从0开始，负数一样代表从后向前查找（-1是最后一个）。WITHSCORE代表是否加上分数。

    4、获得指定分数范围的元素
    **ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]**
    >这个命令参数很多，但是都很好理解。这个命令用来获取指定分数范围的元素，min是最小值，max是最大值，WITHSCORE还是和上面介绍的一样，LIMIT是为了指定偏移量及数量的，和sql的有点像。offset是偏移量，count是数量。同时这些min和max都是包含的，如果要想不包含，需要使用“(”符号。

    5、增加某个元素的分数
    **ZINCRBY key incremnet member**

    6、获得集合中元素的数量
    **ZCARD key**
    >这个命令和SCARD类似，也就不多说了。

    7、获得指定分数范围的元素个数
    **ZCOUNT key min max**
    >这里就是获得min和max分数之间的元素数，当然这里也支持“(”符号。
    
    8、删除一个或多个元素
    **ZREM key member [member ...]**
    >返回值是成功删除的元素的个数。

    9、按照排名范围删除元素
    **ZREMRANGEBYRANK key start stop**
    >这个命令按照元素分数从小到大顺序删除指定范围内所有的元素（其实就是先排序，然后按照排好的序列的索引删除），并返回删除的元素的数量。

    10、按照分数范围删除元素
    **ZREMRANGEBYSCORE key min max**
    >这里就是直接删除分数范围的元素了，这里分数同样支持“(”符号，返回删除数量。

    11、获得元素的排名
    **ZRANK key member**
    **ZREVRANK key member**
    >ZRANK命令按照元素分数的从小到大的顺序获得制定元素的排名（第一个从0开始），ZREVRANK则相反。

3. 实际应用场景：
    最后我们举个实际应用的例子。

    >我们把wordpress的文章按点击率排序，关系数据库我们是遍历所有的文章排序点击数，如果使用Redis，我们需要一个posts:page.view键的有序集合类型，然后每个member为文章ID，score为文章的点击量。这样我们就可以用`ZREVRANGE`命令获取点击量排行榜。

    >还有一个实际的例子，我们用有序集合类型保存文章的发布时间（时间用UNIX时间及时间的毫秒数）与文章ID，这样我们可以很方便的按时间来查看文章列表，我们的文章列表应该是用文章发布时间排序而不应该用文章ID排序的。
4. 例子：
    ```java {.line-numbers}
    /**
        * 下面为校验密码输入错误次数 超过设置次数则锁定钱包
        * @param userId
        * @param request
        * @return
        */
        @Override
        public Map<String, Object> checkInputPWDCount(Integer userId, HttpServletRequest request) {
            Jedis jedis = RedisPool.getJedis();
            try {
                Map<String, Object> checkMap = getCheckTimeAndCount(jedis);
                if (null != checkMap) {
                    return checkMap;
                }
                String ip = null;
                // 获取到操作次数
                Double phoneScore = jedis.zscore(Rkey.REPEAT_REQUEST_CACHE, String.valueOf(userId));

                Long currentTime = System.currentTimeMillis();
                if (null != phoneScore && org.apache.commons.lang3.StringUtils.isNotEmpty(phoneScore.toString())) {
                    Map<String, Object> map = checkInputPWDRepeat(phoneScore, currentTime, jedis, String.valueOf(userId), userId);
                    if (map != null) {
                        return map;
                    }
                } else {
                    Double score = Double.valueOf(currentTime + "1");
                    jedis.zadd(Rkey.REPEAT_REQUEST_CACHE, score, String.valueOf(userId));
                    phoneScore = jedis.zscore(Rkey.REPEAT_REQUEST_CACHE, String.valueOf(userId));
                    Map<String, Object> map = checkInputPWDRepeat(phoneScore, currentTime, jedis, String.valueOf(userId), userId);
                    if (map != null) {
                        return map;
                    }
                }

                //获取IP地址
                if (request.getHeader("x-forwarded-for") == null) {
                    ip = request.getRemoteAddr();
                } else {
                    ip = request.getHeader("x-forwarded-for");
                }
                if (ip == null) {
                    return null;
                }
                //获取缓存数据
                Double ipScore = jedis.zscore(Rkey.REPEAT_REQUEST_CACHE, ip);
                //检测IP
                if (null != ipScore && org.apache.commons.lang3.StringUtils.isNotEmpty(ipScore.toString())) {
                    Map<String, Object> map = checkInputPWDRepeat(ipScore, currentTime, jedis, ip, userId);
                    if (map != null) {
                        return map;
                    }
                } else {
                    Double score = Double.valueOf(currentTime + "1");
                    jedis.zadd(Rkey.REPEAT_REQUEST_CACHE, score, ip);
                    phoneScore = jedis.zscore(Rkey.REPEAT_REQUEST_CACHE, String.valueOf(userId));
                    Map<String, Object> map = checkInputPWDRepeat(ipScore, currentTime, jedis, ip, userId);
                    if (map != null) {
                        return map;
                    }
                }
                return null;
            } finally {
                RedisPool.returnJedis(jedis);
            }
        }

        @Override
        public Map<String, Object> checkInputPWDRepeat(Double score, Long currentTime, Jedis jedis, String checkStr, Integer userId) {
            // 限制时间 默认1分钟
            Long timeLimit = 60000L;
            // 限制次数
            Integer countLimit = 5;
            // 读取redis的配置
            String x = jedis.get(Rkey.WALLET_LOCK_X);
            if (!StringUtils.isEmpty(x)) {
                timeLimit = Long.valueOf(x);
            }
            String y = jedis.get(Rkey.WALLET_LOCK_Y);
            if (!StringUtils.isEmpty(y)) {
                countLimit = Integer.valueOf(y);
            }
            Map<String, Object> returnMap = new HashMap<String, Object>();
            String str = String.valueOf(score.longValue());
            String previousTime = str.substring(0, 13);
            Integer requestCount = Integer.valueOf(str.substring(13));

            if (currentTime - Long.valueOf(previousTime) < timeLimit && requestCount >= countLimit) {
                returnMap.put("errorCode", E.INVALID_REQUEST);
                returnMap.put("msg", "密码输入次数太频繁，钱包功能已锁定！解锁后可正常使用！");
                returnMap.put("keepCount", 0);
                // 2018/4/18  锁定钱包
                updateIsLockByUserId(userId, Boolean.TRUE);
                return returnMap;
            } else {
                if (currentTime - Long.valueOf(previousTime) > timeLimit) {
                    Double newScore = Double.valueOf(currentTime + "1");
                    jedis.zadd(Rkey.REPEAT_REQUEST_CACHE, newScore, checkStr);
                } else {
                    jedis.zincrby(Rkey.REPEAT_REQUEST_CACHE, 1, checkStr);
                    long keepCount = countLimit - requestCount;
                    if (keepCount > 0) {
                        returnMap.put("keepCount", keepCount);
                        returnMap.put("errorCode", E.INVALID_REQUEST);
                        returnMap.put("msg", "支付密码错误，还有" + keepCount + "机会");
                        return returnMap;
                    }
                }
                return null;
            }
        }
    ```
### hash


