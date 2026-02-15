# Effective Modern Redis with Lettuce

A comprehensive guide to mastering Redis with Lettuce client, Spring Data Redis, caching patterns, and performance optimization.

---

## Table of Contents

1. [Redis Fundamentals](#redis-fundamentals)
2. [Lettuce Client Basics](#lettuce-client-basics)
3. [Redis Data Structures](#redis-data-structures)
4. [Spring Data Redis](#spring-data-redis)
5. [Caching Patterns](#caching-patterns)
6. [Pub/Sub and Streams](#pubsub-and-streams)
7. [Distributed Locks](#distributed-locks)
8. [Redis Transactions](#redis-transactions)
9. [Pipeline and Batch Operations](#pipeline-and-batch-operations)
10. [Performance Optimization](#performance-optimization)
11. [High Availability](#high-availability)
12. [Best Practices](#best-practices)

---

## Redis Fundamentals

### Why Redis?

```
Redis Use Cases:
┌─────────────────────────────────────┐
│ Caching                             │
│ - Application cache                 │
│ - Session storage                   │
│ - API response cache                │
├─────────────────────────────────────┤
│ Real-time Analytics                 │
│ - Counters, metrics                 │
│ - Leaderboards                      │
│ - Rate limiting                     │
├─────────────────────────────────────┤
│ Message Queue                       │
│ - Pub/Sub messaging                 │
│ - Task queues                       │
│ - Event streaming                   │
├─────────────────────────────────────┤
│ Distributed Systems                 │
│ - Distributed locks                 │
│ - Session sharing                   │
│ - Configuration management          │
└─────────────────────────────────────┘

Key Characteristics:
- In-memory data store (very fast)
- Supports persistence (RDB, AOF)
- Rich data structures
- Atomic operations
- Pub/Sub messaging
- Lua scripting
```

### Maven Dependencies

```xml
<dependencies>
    <!-- Lettuce (Reactive Redis client) -->
    <dependency>
        <groupId>io.lettuce</groupId>
        <artifactId>lettuce-core</artifactId>
        <version>6.3.0.RELEASE</version>
    </dependency>
    
    <!-- Spring Data Redis (includes Lettuce) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    
    <!-- Spring Cache -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
    
    <!-- Jackson for JSON serialization -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
    
    <!-- Redis Session (optional) -->
    <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-data-redis</artifactId>
    </dependency>
</dependencies>
```

---

## Lettuce Client Basics

### 1. **Standalone Connection**

```java
import io.lettuce.core.*;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.api.sync.RedisCommands;

public class LettuceBasics {
    
    // Synchronous client
    public void synchronousExample() {
        // Create client
        RedisClient client = RedisClient.create("redis://localhost:6379");
        
        // Connect
        StatefulRedisConnection<String, String> connection = client.connect();
        RedisCommands<String, String> commands = connection.sync();
        
        // Execute commands
        commands.set("key", "value");
        String value = commands.get("key");
        
        System.out.println("Value: " + value);
        
        // Cleanup
        connection.close();
        client.shutdown();
    }
    
    // Asynchronous client
    public void asyncExample() throws Exception {
        RedisClient client = RedisClient.create("redis://localhost:6379");
        StatefulRedisConnection<String, String> connection = client.connect();
        
        RedisAsyncCommands<String, String> async = connection.async();
        
        // Non-blocking operations
        RedisFuture<String> setFuture = async.set("key", "value");
        RedisFuture<String> getFuture = async.get("key");
        
        // Wait for completion
        setFuture.get();
        String value = getFuture.get();
        
        System.out.println("Value: " + value);
        
        connection.close();
        client.shutdown();
    }
    
    // Reactive client
    public void reactiveExample() {
        RedisClient client = RedisClient.create("redis://localhost:6379");
        StatefulRedisConnection<String, String> connection = client.connect();
        
        RedisReactiveCommands<String, String> reactive = connection.reactive();
        
        // Reactive operations
        reactive.set("key", "value")
            .flatMap(result -> reactive.get("key"))
            .subscribe(value -> System.out.println("Value: " + value));
        
        // Let reactive operations complete
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        connection.close();
        client.shutdown();
    }
    
    // Connection pooling (recommended for production)
    public void connectionPooling() {
        RedisClient client = RedisClient.create("redis://localhost:6379");
        
        // Get connection from pool
        StatefulRedisConnection<String, String> connection = client.connect();
        
        // Use connection
        RedisCommands<String, String> commands = connection.sync();
        commands.set("key", "value");
        
        // Connection is reused, not closed
        // Cleanup only on application shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            connection.close();
            client.shutdown();
        }));
    }
}
```

### 2. **Configuration Options**

```java
public class LettuceConfiguration {
    
    public RedisClient createConfiguredClient() {
        // Client options
        ClientOptions options = ClientOptions.builder()
            .autoReconnect(true)
            .pingBeforeActivateConnection(true)
            .disconnectedBehavior(ClientOptions.DisconnectedBehavior.REJECT_COMMANDS)
            .build();
        
        // Connection timeout
        RedisURI uri = RedisURI.builder()
            .withHost("localhost")
            .withPort(6379)
            .withTimeout(Duration.ofSeconds(5))
            .withDatabase(0)
            .withPassword("your-password".toCharArray())
            .build();
        
        RedisClient client = RedisClient.create(uri);
        client.setOptions(options);
        
        return client;
    }
    
    // SSL/TLS connection
    public RedisClient createSslClient() {
        RedisURI uri = RedisURI.builder()
            .withHost("redis.example.com")
            .withPort(6380)
            .withSsl(true)
            .withVerifyPeer(true)
            .build();
        
        return RedisClient.create(uri);
    }
}
```

---

## Redis Data Structures

### 1. **Strings**

```java
public class RedisStrings {
    
    private final RedisCommands<String, String> redis;
    
    public void stringOperations() {
        // SET/GET
        redis.set("user:1:name", "John Doe");
        String name = redis.get("user:1:name");
        
        // SET with expiration
        redis.setex("session:abc123", 3600, "user-data");  // Expires in 1 hour
        
        // SET if not exists
        Boolean wasSet = redis.setnx("lock:resource", "locked");
        
        // MSET/MGET - Multiple keys
        redis.mset(Map.of(
            "user:1:name", "John",
            "user:1:email", "john@example.com",
            "user:1:age", "30"
        ));
        
        List<KeyValue<String, String>> values = redis.mget("user:1:name", "user:1:email");
        
        // Increment/Decrement
        Long counter = redis.incr("page:views");
        redis.incrby("score", 10);
        redis.decr("inventory:item1");
        
        // Append
        redis.append("log", "New entry\n");
        
        // Get and Set (atomic)
        String oldValue = redis.getset("config:version", "2.0");
        
        // Get range
        String substring = redis.getrange("user:1:name", 0, 3);  // "John"
        
        // String length
        Long length = redis.strlen("user:1:name");
    }
    
    // JSON storage (as string)
    public void jsonStorage() throws JsonProcessingException {
        ObjectMapper mapper = new ObjectMapper();
        
        User user = new User("123", "John Doe", "john@example.com");
        String json = mapper.writeValueAsString(user);
        
        redis.set("user:123", json);
        
        String retrieved = redis.get("user:123");
        User deserializedUser = mapper.readValue(retrieved, User.class);
    }
}
```

### 2. **Hashes**

```java
public class RedisHashes {
    
    private final RedisCommands<String, String> redis;
    
    public void hashOperations() {
        // HSET/HGET - Single field
        redis.hset("user:1", "name", "John Doe");
        redis.hset("user:1", "email", "john@example.com");
        String name = redis.hget("user:1", "name");
        
        // HMSET - Multiple fields
        redis.hmset("user:1", Map.of(
            "name", "John Doe",
            "email", "john@example.com",
            "age", "30",
            "city", "New York"
        ));
        
        // HMGET - Multiple fields
        List<KeyValue<String, String>> values = redis.hmget("user:1", "name", "email");
        
        // HGETALL - All fields
        Map<String, String> user = redis.hgetall("user:1");
        
        // HEXISTS - Check if field exists
        Boolean hasEmail = redis.hexists("user:1", "email");
        
        // HDEL - Delete field
        redis.hdel("user:1", "age");
        
        // HINCRBY - Increment field
        redis.hincrby("user:1", "loginCount", 1);
        
        // HKEYS - Get all keys
        List<String> fields = redis.hkeys("user:1");
        
        // HVALS - Get all values
        List<String> values2 = redis.hvals("user:1");
        
        // HLEN - Number of fields
        Long fieldCount = redis.hlen("user:1");
    }
    
    // Use case: User profile
    public class UserProfile {
        
        public void saveUser(String userId, Map<String, String> profile) {
            redis.hmset("user:" + userId, profile);
            redis.expire("user:" + userId, 86400);  // Expire in 24 hours
        }
        
        public Map<String, String> getUser(String userId) {
            return redis.hgetall("user:" + userId);
        }
        
        public void updateField(String userId, String field, String value) {
            redis.hset("user:" + userId, field, value);
        }
    }
}
```

### 3. **Lists**

```java
public class RedisLists {
    
    private final RedisCommands<String, String> redis;
    
    public void listOperations() {
        // LPUSH/RPUSH - Add to left/right
        redis.lpush("queue:tasks", "task1", "task2", "task3");
        redis.rpush("queue:tasks", "task4");
        
        // LPOP/RPOP - Remove from left/right
        String task = redis.lpop("queue:tasks");  // FIFO queue
        String lastTask = redis.rpop("queue:tasks");  // LIFO stack
        
        // LRANGE - Get range
        List<String> tasks = redis.lrange("queue:tasks", 0, -1);  // All elements
        List<String> firstThree = redis.lrange("queue:tasks", 0, 2);
        
        // LLEN - Length
        Long length = redis.llen("queue:tasks");
        
        // LINDEX - Get by index
        String firstTask = redis.lindex("queue:tasks", 0);
        
        // LSET - Set by index
        redis.lset("queue:tasks", 0, "updated-task");
        
        // LREM - Remove elements
        redis.lrem("queue:tasks", 2, "task1");  // Remove 2 occurrences
        
        // LTRIM - Keep only range
        redis.ltrim("queue:tasks", 0, 99);  // Keep first 100 elements
        
        // RPOPLPUSH - Atomic move between lists
        String item = redis.rpoplpush("source:list", "dest:list");
        
        // Blocking operations
        KeyValue<String, String> result = redis.blpop(5, "queue:tasks");  // Wait 5 seconds
    }
    
    // Use case: Task queue
    public class TaskQueue {
        
        public void enqueue(String task) {
            redis.rpush("tasks", task);
        }
        
        public String dequeue() {
            return redis.lpop("tasks");
        }
        
        public String blockingDequeue(int timeoutSeconds) {
            KeyValue<String, String> result = redis.blpop(timeoutSeconds, "tasks");
            return result != null ? result.getValue() : null;
        }
        
        public List<String> getTasks() {
            return redis.lrange("tasks", 0, -1);
        }
    }
    
    // Use case: Recent activity feed
    public class ActivityFeed {
        
        private static final int MAX_ACTIVITIES = 100;
        
        public void addActivity(String userId, String activity) {
            String key = "activity:" + userId;
            redis.lpush(key, activity);
            redis.ltrim(key, 0, MAX_ACTIVITIES - 1);  // Keep only recent 100
        }
        
        public List<String> getRecentActivities(String userId, int count) {
            return redis.lrange("activity:" + userId, 0, count - 1);
        }
    }
}
```

### 4. **Sets**

```java
public class RedisSets {
    
    private final RedisCommands<String, String> redis;
    
    public void setOperations() {
        // SADD - Add members
        redis.sadd("tags:post1", "java", "redis", "tutorial");
        
        // SMEMBERS - Get all members
        Set<String> tags = redis.smembers("tags:post1");
        
        // SISMEMBER - Check membership
        Boolean hasJava = redis.sismember("tags:post1", "java");
        
        // SCARD - Count members
        Long count = redis.scard("tags:post1");
        
        // SREM - Remove members
        redis.srem("tags:post1", "tutorial");
        
        // SPOP - Remove random member
        String randomTag = redis.spop("tags:post1");
        
        // SRANDMEMBER - Get random member (without removing)
        String random = redis.srandmember("tags:post1");
        List<String> randomTags = redis.srandmember("tags:post1", 2);
        
        // Set operations
        redis.sadd("set1", "a", "b", "c");
        redis.sadd("set2", "b", "c", "d");
        
        // SINTER - Intersection
        Set<String> intersection = redis.sinter("set1", "set2");  // {b, c}
        
        // SUNION - Union
        Set<String> union = redis.sunion("set1", "set2");  // {a, b, c, d}
        
        // SDIFF - Difference
        Set<String> diff = redis.sdiff("set1", "set2");  // {a}
        
        // Store results
        redis.sinterstore("result", "set1", "set2");
        redis.sunionstore("result", "set1", "set2");
    }
    
    // Use case: User followers/following
    public class SocialGraph {
        
        public void follow(String userId, String followeeId) {
            redis.sadd("following:" + userId, followeeId);
            redis.sadd("followers:" + followeeId, userId);
        }
        
        public void unfollow(String userId, String followeeId) {
            redis.srem("following:" + userId, followeeId);
            redis.srem("followers:" + followeeId, userId);
        }
        
        public Set<String> getFollowers(String userId) {
            return redis.smembers("followers:" + userId);
        }
        
        public Set<String> getFollowing(String userId) {
            return redis.smembers("following:" + userId);
        }
        
        public Set<String> getMutualFollowers(String user1, String user2) {
            return redis.sinter("followers:" + user1, "followers:" + user2);
        }
        
        public Long getFollowerCount(String userId) {
            return redis.scard("followers:" + userId);
        }
    }
    
    // Use case: Tags and filtering
    public class TagFilter {
        
        public void tagItem(String itemId, String... tags) {
            redis.sadd("tags:" + itemId, tags);
            for (String tag : tags) {
                redis.sadd("items:tag:" + tag, itemId);
            }
        }
        
        public Set<String> findItemsWithAllTags(String... tags) {
            String[] keys = Arrays.stream(tags)
                .map(tag -> "items:tag:" + tag)
                .toArray(String[]::new);
            
            return redis.sinter(keys);
        }
        
        public Set<String> findItemsWithAnyTag(String... tags) {
            String[] keys = Arrays.stream(tags)
                .map(tag -> "items:tag:" + tag)
                .toArray(String[]::new);
            
            return redis.sunion(keys);
        }
    }
}
```

### 5. **Sorted Sets**

```java
public class RedisSortedSets {
    
    private final RedisCommands<String, String> redis;
    
    public void sortedSetOperations() {
        // ZADD - Add with score
        redis.zadd("leaderboard", 100, "player1");
        redis.zadd("leaderboard", 85, "player2");
        redis.zadd("leaderboard", 92, "player3");
        
        // Multiple adds
        redis.zadd("leaderboard", 
            ScoredValue.just(75, "player4"),
            ScoredValue.just(88, "player5")
        );
        
        // ZRANGE - Get by rank (ascending)
        List<String> topPlayers = redis.zrange("leaderboard", 0, 2);
        
        // ZREVRANGE - Get by rank (descending)
        List<String> topPlayersDesc = redis.zrevrange("leaderboard", 0, 2);
        
        // With scores
        List<ScoredValue<String>> withScores = redis.zrangeWithScores("leaderboard", 0, -1);
        
        // ZRANK/ZREVRANK - Get rank
        Long rank = redis.zrank("leaderboard", "player1");  // 0-based
        Long revRank = redis.zrevrank("leaderboard", "player1");
        
        // ZSCORE - Get score
        Double score = redis.zscore("leaderboard", "player1");
        
        // ZINCRBY - Increment score
        Double newScore = redis.zincrby("leaderboard", 10, "player1");
        
        // ZCARD - Count members
        Long count = redis.zcard("leaderboard");
        
        // ZCOUNT - Count in score range
        Long countInRange = redis.zcount("leaderboard", 80, 95);
        
        // ZRANGEBYSCORE - Get by score range
        List<String> inRange = redis.zrangebyscore("leaderboard", 80, 95);
        
        // ZREM - Remove members
        redis.zrem("leaderboard", "player1");
        
        // ZREMRANGEBYRANK - Remove by rank
        redis.zremrangebyrank("leaderboard", 0, 2);  // Remove bottom 3
        
        // ZREMRANGEBYSCORE - Remove by score
        redis.zremrangebyscore("leaderboard", 0, 50);  // Remove scores 0-50
    }
    
    // Use case: Leaderboard
    public class Leaderboard {
        
        private static final String KEY_PREFIX = "leaderboard:";
        
        public void addScore(String gameId, String playerId, double score) {
            redis.zadd(KEY_PREFIX + gameId, score, playerId);
        }
        
        public void incrementScore(String gameId, String playerId, double points) {
            redis.zincrby(KEY_PREFIX + gameId, points, playerId);
        }
        
        public List<ScoredValue<String>> getTopPlayers(String gameId, int count) {
            return redis.zrevrangeWithScores(KEY_PREFIX + gameId, 0, count - 1);
        }
        
        public Long getPlayerRank(String gameId, String playerId) {
            Long rank = redis.zrevrank(KEY_PREFIX + gameId, playerId);
            return rank != null ? rank + 1 : null;  // Convert to 1-based
        }
        
        public Double getPlayerScore(String gameId, String playerId) {
            return redis.zscore(KEY_PREFIX + gameId, playerId);
        }
    }
    
    // Use case: Time-series events with auto-cleanup
    public class TimeSeries {
        
        public void addEvent(String key, String eventId, Instant timestamp) {
            redis.zadd(key, timestamp.toEpochMilli(), eventId);
            
            // Clean up old events (older than 24 hours)
            long cutoff = Instant.now().minus(24, ChronoUnit.HOURS).toEpochMilli();
            redis.zremrangebyscore(key, 0, cutoff);
        }
        
        public List<String> getRecentEvents(String key, Duration duration) {
            long cutoff = Instant.now().minus(duration).toEpochMilli();
            return redis.zrangebyscore(key, cutoff, Double.MAX_VALUE);
        }
        
        public Long countRecentEvents(String key, Duration duration) {
            long cutoff = Instant.now().minus(duration).toEpochMilli();
            return redis.zcount(key, cutoff, Double.MAX_VALUE);
        }
    }
    
    // Use case: Rate limiting
    public class RateLimiter {
        
        public boolean isAllowed(String userId, int maxRequests, Duration window) {
            String key = "ratelimit:" + userId;
            long now = System.currentTimeMillis();
            long windowStart = now - window.toMillis();
            
            // Remove old requests
            redis.zremrangebyscore(key, 0, windowStart);
            
            // Count requests in window
            Long requestCount = redis.zcard(key);
            
            if (requestCount < maxRequests) {
                redis.zadd(key, now, UUID.randomUUID().toString());
                redis.expire(key, window.getSeconds());
                return true;
            }
            
            return false;
        }
    }
}
```

---

## Spring Data Redis

### 1. **Configuration**

```java
@Configuration
@EnableCaching
public class RedisConfig {
    
    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName("localhost");
        config.setPort(6379);
        config.setPassword("password");
        config.setDatabase(0);
        
        // Lettuce client configuration
        LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
            .commandTimeout(Duration.ofSeconds(5))
            .shutdownTimeout(Duration.ofMillis(100))
            .build();
        
        return new LettuceConnectionFactory(config, clientConfig);
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            LettuceConnectionFactory connectionFactory) {
        
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        
        // Jackson serializer for values
        Jackson2JsonRedisSerializer<Object> serializer = 
            new Jackson2JsonRedisSerializer<>(Object.class);
        
        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.activateDefaultTyping(
            mapper.getPolymorphicTypeValidator(),
            ObjectMapper.DefaultTyping.NON_FINAL
        );
        serializer.setObjectMapper(mapper);
        
        // String serializer for keys
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        
        // JSON serializer for values
        template.setValueSerializer(serializer);
        template.setHashValueSerializer(serializer);
        
        template.afterPropertiesSet();
        return template;
    }
    
    @Bean
    public CacheManager cacheManager(LettuceConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new StringRedisSerializer()
                )
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()
                )
            );
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .build();
    }
}
```

### 2. **RedisTemplate Usage**

```java
@Service
public class RedisService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    public RedisService(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }
    
    // String operations
    public void setString(String key, String value, Duration ttl) {
        redisTemplate.opsForValue().set(key, value, ttl);
    }
    
    public String getString(String key) {
        return (String) redisTemplate.opsForValue().get(key);
    }
    
    // Object operations
    public void saveUser(User user) {
        redisTemplate.opsForValue().set(
            "user:" + user.getId(),
            user,
            Duration.ofHours(24)
        );
    }
    
    public User getUser(String userId) {
        return (User) redisTemplate.opsForValue().get("user:" + userId);
    }
    
    // Hash operations
    public void saveUserProfile(String userId, Map<String, String> profile) {
        redisTemplate.opsForHash().putAll("profile:" + userId, profile);
        redisTemplate.expire("profile:" + userId, Duration.ofDays(7));
    }
    
    public Map<Object, Object> getUserProfile(String userId) {
        return redisTemplate.opsForHash().entries("profile:" + userId);
    }
    
    // List operations
    public void addToQueue(String queueName, String item) {
        redisTemplate.opsForList().rightPush(queueName, item);
    }
    
    public String popFromQueue(String queueName) {
        return (String) redisTemplate.opsForList().leftPop(queueName);
    }
    
    // Set operations
    public void addToSet(String key, String... values) {
        redisTemplate.opsForSet().add(key, (Object[]) values);
    }
    
    public Set<Object> getSet(String key) {
        return redisTemplate.opsForSet().members(key);
    }
    
    // Sorted set operations
    public void addToLeaderboard(String leaderboard, String player, double score) {
        redisTemplate.opsForZSet().add(leaderboard, player, score);
    }
    
    public Set<Object> getTopPlayers(String leaderboard, int count) {
        return redisTemplate.opsForZSet().reverseRange(leaderboard, 0, count - 1);
    }
    
    // Key operations
    public Boolean hasKey(String key) {
        return redisTemplate.hasKey(key);
    }
    
    public Boolean delete(String key) {
        return redisTemplate.delete(key);
    }
    
    public Boolean expire(String key, Duration timeout) {
        return redisTemplate.expire(key, timeout);
    }
}
```

---

## Caching Patterns

### 1. **Spring Cache Abstraction**

```java
@Service
public class UserService {
    
    private final UserRepository userRepository;
    
    // Cache result
    @Cacheable(value = "users", key = "#id")
    public User getUserById(String id) {
        System.out.println("Fetching user from database: " + id);
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    // Cache with condition
    @Cacheable(value = "users", key = "#id", condition = "#id.length() < 10")
    public User getUserIfShortId(String id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    // Cache unless specific condition
    @Cacheable(value = "users", key = "#id", unless = "#result.isGuest()")
    public User getUserUnlessGuest(String id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    // Update cache
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
    
    // Evict cache
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(String id) {
        userRepository.deleteById(id);
    }
    
    // Evict all entries
    @CacheEvict(value = "users", allEntries = true)
    public void deleteAllUsers() {
        userRepository.deleteAll();
    }
    
    // Multiple cache operations
    @Caching(
        cacheable = @Cacheable(value = "users", key = "#id"),
        put = @CachePut(value = "usersByEmail", key = "#result.email")
    )
    public User getUserWithMultiCache(String id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

### 2. **Custom Cache Configuration**

```java
@Configuration
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(LettuceConnectionFactory connectionFactory) {
        // Default configuration
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .disableCachingNullValues();
        
        // Custom configurations per cache
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        
        // Users cache - 24 hours TTL
        cacheConfigurations.put("users",
            RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(24))
                .prefixCacheNameWith("app::")
        );
        
        // Sessions cache - 30 minutes TTL
        cacheConfigurations.put("sessions",
            RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(30))
        );
        
        // Products cache - 1 hour TTL, max 1000 entries
        cacheConfigurations.put("products",
            RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1))
        );
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigurations)
            .build();
    }
}
```

### 3. **Cache-Aside Pattern**

```java
@Service
public class CacheAsideService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final UserRepository userRepository;
    
    public User getUser(String userId) {
        String cacheKey = "user:" + userId;
        
        // Try cache first
        User cachedUser = (User) redisTemplate.opsForValue().get(cacheKey);
        if (cachedUser != null) {
            return cachedUser;
        }
        
        // Cache miss - fetch from database
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        // Store in cache
        redisTemplate.opsForValue().set(
            cacheKey,
            user,
            Duration.ofHours(24)
        );
        
        return user;
    }
    
    public User updateUser(User user) {
        // Update database
        User updated = userRepository.save(user);
        
        // Update cache
        redisTemplate.opsForValue().set(
            "user:" + user.getId(),
            updated,
            Duration.ofHours(24)
        );
        
        return updated;
    }
    
    public void deleteUser(String userId) {
        // Delete from database
        userRepository.deleteById(userId);
        
        // Evict from cache
        redisTemplate.delete("user:" + userId);
    }
}
```

### 4. **Write-Through Cache**

```java
@Service
public class WriteThroughCacheService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final UserRepository userRepository;
    
    public User saveUser(User user) {
        // Write to database first
        User saved = userRepository.save(user);
        
        // Then write to cache
        redisTemplate.opsForValue().set(
            "user:" + saved.getId(),
            saved,
            Duration.ofHours(24)
        );
        
        return saved;
    }
}
```

### 5. **Cache Stampede Prevention**

```java
@Service
public class CacheStampedePreventionService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final UserRepository userRepository;
    
    public User getUser(String userId) {
        String cacheKey = "user:" + userId;
        String lockKey = "lock:user:" + userId;
        
        // Try cache
        User user = (User) redisTemplate.opsForValue().get(cacheKey);
        if (user != null) {
            return user;
        }
        
        // Try to acquire lock
        Boolean lockAcquired = redisTemplate.opsForValue().setIfAbsent(
            lockKey,
            "locked",
            Duration.ofSeconds(10)
        );
        
        if (Boolean.TRUE.equals(lockAcquired)) {
            try {
                // This thread loads the data
                user = userRepository.findById(userId)
                    .orElseThrow(() -> new UserNotFoundException(userId));
                
                redisTemplate.opsForValue().set(
                    cacheKey,
                    user,
                    Duration.ofHours(24)
                );
                
                return user;
            } finally {
                redisTemplate.delete(lockKey);
            }
        } else {
            // Other threads wait and retry
            try {
                Thread.sleep(100);
                return getUser(userId);  // Retry
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            }
        }
    }
}
```

---

## Pub/Sub and Streams

### 1. **Pub/Sub Pattern**

```java
@Configuration
public class RedisPubSubConfig {
    
    @Bean
    public RedisMessageListenerContainer messageListenerContainer(
            LettuceConnectionFactory connectionFactory,
            MessageListener messageListener) {
        
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        
        // Subscribe to channel
        container.addMessageListener(
            messageListener,
            new PatternTopic("events:*")
        );
        
        return container;
    }
    
    @Bean
    public MessageListener messageListener() {
        return (message, pattern) -> {
            String channel = new String(message.getChannel());
            String body = new String(message.getBody());
            
            System.out.println("Received message on " + channel + ": " + body);
        };
    }
}

@Service
public class EventPublisher {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    public void publishEvent(String channel, String message) {
        redisTemplate.convertAndSend(channel, message);
    }
    
    public void publishUserEvent(String userId, String eventType, Object data) {
        Map<String, Object> event = Map.of(
            "userId", userId,
            "eventType", eventType,
            "data", data,
            "timestamp", Instant.now().toString()
        );
        
        redisTemplate.convertAndSend("events:users", event);
    }
}
```

### 2. **Redis Streams**

```java
@Service
public class RedisStreamsService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    public String addToStream(String streamKey, Map<String, String> data) {
        StringRecord record = StreamRecords.string(data)
            .withStreamKey(streamKey);
        
        RecordId recordId = redisTemplate.opsForStream().add(record);
        return recordId.getValue();
    }
    
    public List<MapRecord<String, Object, Object>> readFromStream(
            String streamKey,
            String consumerId) {
        
        return redisTemplate.opsForStream().read(
            Consumer.from("my-group", consumerId),
            StreamReadOptions.empty().count(10),
            StreamOffset.create(streamKey, ReadOffset.lastConsumed())
        );
    }
    
    public void createConsumerGroup(String streamKey, String groupName) {
        try {
            redisTemplate.opsForStream().createGroup(streamKey, groupName);
        } catch (Exception e) {
            // Group might already exist
        }
    }
    
    public void acknowledge(String streamKey, String groupName, String... recordIds) {
        redisTemplate.opsForStream().acknowledge(
            streamKey,
            groupName,
            recordIds
        );
    }
}

// Event-driven example
@Service
public class OrderEventProducer {
    
    private final RedisStreamsService streamsService;
    
    public void publishOrderCreated(Order order) {
        Map<String, String> event = Map.of(
            "eventType", "ORDER_CREATED",
            "orderId", order.getId(),
            "customerId", order.getCustomerId(),
            "amount", order.getAmount().toString(),
            "timestamp", Instant.now().toString()
        );
        
        streamsService.addToStream("orders:events", event);
    }
}

@Component
public class OrderEventConsumer {
    
    private final RedisStreamsService streamsService;
    
    @Scheduled(fixedDelay = 1000)
    public void consumeEvents() {
        List<MapRecord<String, Object, Object>> records = 
            streamsService.readFromStream("orders:events", "consumer-1");
        
        for (MapRecord<String, Object, Object> record : records) {
            processEvent(record.getValue());
            streamsService.acknowledge(
                "orders:events",
                "my-group",
                record.getId().getValue()
            );
        }
    }
    
    private void processEvent(Map<Object, Object> event) {
        String eventType = (String) event.get("eventType");
        System.out.println("Processing: " + eventType);
    }
}
```

---

## Distributed Locks

### 1. **Simple Lock Implementation**

```java
@Service
public class RedisLockService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    public boolean acquireLock(String lockKey, String requestId, Duration timeout) {
        Boolean success = redisTemplate.opsForValue().setIfAbsent(
            lockKey,
            requestId,
            timeout
        );
        
        return Boolean.TRUE.equals(success);
    }
    
    public boolean releaseLock(String lockKey, String requestId) {
        // Lua script for atomic check-and-delete
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(lockKey),
            requestId
        );
        
        return result != null && result == 1;
    }
    
    public <T> T executeWithLock(
            String lockKey,
            Duration timeout,
            Supplier<T> task) {
        
        String requestId = UUID.randomUUID().toString();
        
        if (!acquireLock(lockKey, requestId, timeout)) {
            throw new LockAcquisitionException("Failed to acquire lock: " + lockKey);
        }
        
        try {
            return task.get();
        } finally {
            releaseLock(lockKey, requestId);
        }
    }
}

// Usage example
@Service
public class InventoryService {
    
    private final RedisLockService lockService;
    
    public void decrementInventory(String productId, int quantity) {
        String lockKey = "lock:inventory:" + productId;
        
        lockService.executeWithLock(
            lockKey,
            Duration.ofSeconds(10),
            () -> {
                // Critical section
                int current = getInventory(productId);
                if (current >= quantity) {
                    setInventory(productId, current - quantity);
                    return true;
                } else {
                    throw new InsufficientInventoryException();
                }
            }
        );
    }
}
```

### 2. **Redisson Lock (Production-Ready)**

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.25.0</version>
</dependency>
```

```java
@Configuration
public class RedissonConfig {
    
    @Bean(destroyMethod = "shutdown")
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
            .setAddress("redis://localhost:6379")
            .setPassword("password")
            .setConnectionPoolSize(50)
            .setConnectionMinimumIdleSize(10);
        
        return Redisson.create(config);
    }
}

@Service
public class RedissonLockService {
    
    private final RedissonClient redissonClient;
    
    public void executeWithLock(String lockKey, Runnable task) {
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // Try to acquire lock (wait up to 10 seconds, auto-release after 30 seconds)
            boolean acquired = lock.tryLock(10, 30, TimeUnit.SECONDS);
            
            if (acquired) {
                try {
                    task.run();
                } finally {
                    if (lock.isHeldByCurrentThread()) {
                        lock.unlock();
                    }
                }
            } else {
                throw new LockAcquisitionException("Failed to acquire lock");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
    
    // Fair lock (FIFO order)
    public void executeWithFairLock(String lockKey, Runnable task) {
        RLock fairLock = redissonClient.getFairLock(lockKey);
        
        fairLock.lock(30, TimeUnit.SECONDS);
        try {
            task.run();
        } finally {
            fairLock.unlock();
        }
    }
    
    // Read-write lock
    public void executeWithReadLock(String lockKey, Runnable task) {
        RReadWriteLock rwLock = redissonClient.getReadWriteLock(lockKey);
        RLock readLock = rwLock.readLock();
        
        readLock.lock(30, TimeUnit.SECONDS);
        try {
            task.run();
        } finally {
            readLock.unlock();
        }
    }
}
```

---

## Redis Transactions

### 1. **Basic Transactions (MULTI/EXEC)**

```java
@Service
public class RedisTransactionService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    public void executeTransaction() {
        redisTemplate.execute(new SessionCallback<Object>() {
            @Override
            public Object execute(RedisOperations operations) {
                operations.multi();  // Start transaction
                
                operations.opsForValue().set("key1", "value1");
                operations.opsForValue().set("key2", "value2");
                operations.opsForValue().increment("counter");
                
                return operations.exec();  // Execute transaction
            }
        });
    }
    
    // Transfer between accounts
    public void transfer(String fromAccount, String toAccount, double amount) {
        redisTemplate.execute(new SessionCallback<Object>() {
            @Override
            public Object execute(RedisOperations operations) {
                String fromKey = "account:" + fromAccount;
                String toKey = "account:" + toAccount;
                
                operations.multi();
                
                operations.opsForValue().increment(fromKey, -amount);
                operations.opsForValue().increment(toKey, amount);
                
                return operations.exec();
            }
        });
    }
}
```

### 2. **Optimistic Locking with WATCH**

```java
@Service
public class OptimisticLockingService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    public boolean incrementWithOptimisticLock(String key) {
        return redisTemplate.execute(new SessionCallback<Boolean>() {
            @Override
            public Boolean execute(RedisOperations operations) {
                while (true) {
                    operations.watch(key);  // Watch for changes
                    
                    Object value = operations.opsForValue().get(key);
                    int currentValue = value != null ? (Integer) value : 0;
                    
                    operations.multi();
                    operations.opsForValue().set(key, currentValue + 1);
                    
                    List<Object> result = operations.exec();
                    
                    if (result != null) {
                        // Transaction succeeded
                        return true;
                    }
                    
                    // Transaction failed (key was modified), retry
                    // Could add retry limit here
                }
            }
        });
    }
}
```

---

## Pipeline and Batch Operations

### 1. **Pipeline**

```java
@Service
public class RedisPipelineService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    public void pipelineExample() {
        List<Object> results = redisTemplate.executePipelined(
            new SessionCallback<Object>() {
                @Override
                public Object execute(RedisOperations operations) {
                    for (int i = 0; i < 1000; i++) {
                        operations.opsForValue().set("key:" + i, "value:" + i);
                    }
                    return null;  // Results collected automatically
                }
            }
        );
        
        System.out.println("Pipeline executed " + results.size() + " commands");
    }
    
    // Bulk user save
    public void saveUsers(List<User> users) {
        redisTemplate.executePipelined(new SessionCallback<Object>() {
            @Override
            public Object execute(RedisOperations operations) {
                for (User user : users) {
                    operations.opsForValue().set(
                        "user:" + user.getId(),
                        user,
                        Duration.ofHours(24)
                    );
                }
                return null;
            }
        });
    }
    
    // Bulk get with pipeline
    public Map<String, User> getUsers(List<String> userIds) {
        List<Object> results = redisTemplate.executePipelined(
            new SessionCallback<Object>() {
                @Override
                public Object execute(RedisOperations operations) {
                    for (String userId : userIds) {
                        operations.opsForValue().get("user:" + userId);
                    }
                    return null;
                }
            }
        );
        
        Map<String, User> users = new HashMap<>();
        for (int i = 0; i < userIds.size(); i++) {
            if (results.get(i) != null) {
                users.put(userIds.get(i), (User) results.get(i));
            }
        }
        
        return users;
    }
}
```

---

## Performance Optimization

### Best Practices

```java
/**
 * Redis Performance Optimization Guide
 */

// 1. Use connection pooling (Lettuce does this by default)
@Bean
public LettuceConnectionFactory connectionFactory() {
    GenericObjectPoolConfig poolConfig = new GenericObjectPoolConfig();
    poolConfig.setMaxTotal(50);
    poolConfig.setMaxIdle(20);
    poolConfig.setMinIdle(10);
    poolConfig.setMaxWaitMillis(2000);
    
    LettucePoolingClientConfiguration clientConfig = 
        LettucePoolingClientConfiguration.builder()
            .poolConfig(poolConfig)
            .commandTimeout(Duration.ofSeconds(5))
            .build();
    
    return new LettuceConnectionFactory(redisStandaloneConfiguration, clientConfig);
}

// 2. Use appropriate data structures
// ❌ BAD: Storing JSON as string
redis.set("user:123", "{\"name\":\"John\",\"age\":30}");

// ✅ GOOD: Using hashes
redis.hmset("user:123", Map.of("name", "John", "age", "30"));

// 3. Set expiration on keys
// ✅ GOOD: Always set TTL
redis.setex("session:" + sessionId, 3600, sessionData);

// ❌ BAD: Keys without expiration
redis.set("temp:data", value);  // Will stay forever!

// 4. Use pipelining for bulk operations
// ❌ BAD: Individual commands
for (String key : keys) {
    redis.get(key);  // Network round trip per key
}

// ✅ GOOD: Pipeline
redisTemplate.executePipelined(operations -> {
    for (String key : keys) {
        operations.opsForValue().get(key);
    }
    return null;
});

// 5. Use appropriate serialization
// JSON for human-readable data
// Protocol Buffers/Avro for binary efficiency

// 6. Monitor key sizes
// ❌ BAD: Large values
redis.set("big:data", largeJsonString);  // > 1MB

// ✅ GOOD: Split or compress
compressAndSet("big:data", largeJsonString);

// 7. Use scan instead of keys
// ❌ BAD: Blocks Redis
Set<String> keys = redis.keys("user:*");

// ✅ GOOD: Non-blocking scan
ScanOptions options = ScanOptions.scanOptions().match("user:*").count(100).build();
Cursor<byte[]> cursor = redisTemplate.executeWithStickyConnection(
    connection -> connection.scan(options)
);

// 8. Avoid hot keys
// Distribute load across multiple keys
String shardKey = "counter:" + (userId.hashCode() % 10);
```

---

## High Availability

### 1. **Redis Sentinel Configuration**

```java
@Configuration
public class RedisSentinelConfig {
    
    @Bean
    public LettuceConnectionFactory sentinelConnectionFactory() {
        RedisSentinelConfiguration sentinelConfig = 
            new RedisSentinelConfiguration()
                .master("mymaster")
                .sentinel("sentinel1", 26379)
                .sentinel("sentinel2", 26379)
                .sentinel("sentinel3", 26379);
        
        sentinelConfig.setPassword("password");
        
        return new LettuceConnectionFactory(sentinelConfig);
    }
}
```

### 2. **Redis Cluster Configuration**

```java
@Configuration
public class RedisClusterConfig {
    
    @Bean
    public LettuceConnectionFactory clusterConnectionFactory() {
        RedisClusterConfiguration clusterConfig = 
            new RedisClusterConfiguration()
                .clusterNode("node1", 7000)
                .clusterNode("node2", 7001)
                .clusterNode("node3", 7002)
                .clusterNode("node4", 7003)
                .clusterNode("node5", 7004)
                .clusterNode("node6", 7005);
        
        clusterConfig.setPassword("password");
        clusterConfig.setMaxRedirects(3);
        
        return new LettuceConnectionFactory(clusterConfig);
    }
}
```

---

## Best Practices

### ✅ DO

1. **Set expiration on keys** - Prevent memory bloat
2. **Use connection pooling** - Reuse connections
3. **Use pipelining** - Reduce network round trips
4. **Choose right data structures** - Hash > JSON string
5. **Monitor memory usage** - Set maxmemory policy
6. **Use Lua scripts** - Atomic complex operations
7. **Set appropriate TTLs** - Based on use case
8. **Use scan, not keys** - Non-blocking iteration
9. **Enable persistence** - RDB + AOF for durability
10. **Monitor slow queries** - Use Redis slowlog

### ❌ DON'T

1. **Don't use Redis as primary database** - It's a cache
2. **Don't store large values** - Keep under 1MB
3. **Don't use blocking operations in production** - BLPOP carefully
4. **Don't use KEYS in production** - Blocks server
5. **Don't forget to handle connection failures** - Retry logic
6. **Don't store sensitive data unencrypted** - Use encryption
7. **Don't ignore memory limits** - Set maxmemory
8. **Don't use SELECT** (multiple DBs) - Use key prefixes
9. **Don't forget to clean up** - Set expiration
10. **Don't use Redis for complex queries** - Use proper database

---

## Conclusion

**Key Takeaways:**

1. **Lettuce** - Modern, reactive Redis client for Java
2. **Data Structures** - Choose wisely (String, Hash, List, Set, Sorted Set)
3. **Caching** - Cache-aside, write-through, stampede prevention
4. **Distributed Systems** - Locks, pub/sub, streams
5. **Performance** - Pipelining, connection pooling, proper serialization
6. **High Availability** - Sentinel for HA, Cluster for horizontal scaling

**Redis Strengths:**
- Blazing fast (in-memory)
- Rich data structures
- Atomic operations
- Pub/Sub messaging
- Simple yet powerful

**Remember**: Redis is not a one-size-fits-all solution. Use it where it excels: caching, real-time analytics, message queuing, and session storage!
