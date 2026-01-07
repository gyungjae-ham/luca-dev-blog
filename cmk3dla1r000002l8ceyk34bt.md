---
title: "Why HikariCP is the Default Choice: Analysis and Optimization Guide"
seoTitle: "HikariCP"
seoDescription: "HikariCP"
datePublished: Wed Jan 07 2026 02:04:43 GMT+0000 (Coordinated Universal Time)
cuid: cmk3dla1r000002l8ceyk34bt
slug: why-hikaricp-is-the-default-choice-analysis-and-optimization-guide
tags: hikari, hikaricp

---

In Spring Boot, HikariCP has been the default connection pool since version 2.0. As a backend developer, understanding "why" we use it and how to tune it for production is crucial for building scalable systems.

The Spring documentation explicitly states the preference order for connection pooling:

1. **HikariCP**: Preferred for its performance and concurrency. If available, it is always chosen.
    
2. **Tomcat Pooling**: The second choice if HikariCP is unavailable.
    
3. **Commons DBCP2**: The third alternative.
    
4. **Oracle UCP**: Used as a last resort.
    

Let’s dive into why HikariCP is considered the "gold standard" and how we can optimize it for live environments.

---

## Why Use a Connection Pool (DBCP) at All?

In raw JDBC, connecting to a database is an expensive operation. Every user request would require:

1. Loading the DB driver.
    
2. Establishing a **TCP/IP connection** (the infamous 3-way handshake).
    
3. Authentication (sending ID/PW) and creating a DB session.
    
4. Returning the connection object to the client.
    

Doing this for every request is extremely inefficient. A Connection Pool pre-allocates these connections so the application can simply "borrow" and "return" them, bypassing the heavy handshake and initialization overhead.

---

## Performance Analysis

The benchmark results from the HikariCP team show a staggering difference:

* **Connection Cycle (ops/ms):** HikariCP handles ~50,000 operations per millisecond, while its closest competitor (Vibur) manages only about 1/10th of that.
    
* **Statement Cycle (ops/ms):** The gap remains just as wide, proving HikariCP's dominance in executing SQL statements.
    

### The Secret Sauce: How HikariCP Achieves This

HikariCP goes "down the rabbit hole" with low-level optimizations:

1. **Bytecode-level Engineering**: The team studied JIT (Just-In-Time) compiler assembly output to ensure critical routines stay under the "Inline Threshold." By making methods inlineable, they eliminate the overhead of method calls.
    
2. **CPU Cache Optimization**: They optimized the code to fit within L1/L2 caches. By minimizing instructions, they ensure tasks complete within the OS scheduler's time slice, avoiding the performance hit of a "cache miss" when a thread is rescheduled to a different core.
    
3. **Custom Collections (FastList)**: They replaced `ArrayList` with a custom `FastList`.
    
    * **Removed Range Checks**: It skips index validation (e.g., checking if the index exceeds array size) to save cycles.
        
    * **Optimized Scans**: It performs removal scans from head to tail, specifically optimized for the way connection pools access statements.
        

---

## MySQL Performance Optimization Options

The HikariCP team suggests several `data-source-properties` to maximize MySQL throughput:

### 1\. PreparedStatement Caching

* `cachePrepStmts=true`: Enables client-side caching of `PreparedStatement` objects.
    
* `prepStmtCacheSize=250`: Number of statements to cache per connection. (Default: 25, Recommended: 250–500).
    
* `prepStmtCacheSqlLimit=2048`: Maximum length of a SQL string to cache. Essential for long ORM-generated queries.
    

### 2\. Server-Side & Protocol Optimization

* `useServerPrepStmts=true`: Instead of sending full SQL strings, it sends a template to the server and only passes parameters thereafter. This reduces network traffic and allows the DB to reuse execution plans.
    
* `rewriteBatchedStatements=true`: Optimizes bulk INSERT/UPDATE operations.
    
* `useLocalSessionState=true`: Tracks session state locally to avoid unnecessary round-trips to the server.
    

### 3\. Metadata & Stats

* `elideSetAutoCommits=true`: Eliminates redundant `setAutoCommit` calls.
    
* `maintainTimeStats=false`: Disables internal timing metrics to reduce overhead.
    
    * *Note: While this boosts performance, it makes troubleshooting harder as you lose metrics like connection acquisition time.*
        

---

## Recommended Configuration for Production

### Database Level (Reference)

Ensure your DB server is configured to handle the pool size.

**MySQL:**

```sql
max_connections = 1000
innodb_buffer_pool_size = 4G
wait_timeout = 28800
```

### Spring Boot `application.yml`

Here is a production-ready configuration focused on performance and stability.

```yaml
spring:
  datasource:
    url: jdbc:mysql://db-url:3306/dbname?rewriteBatchedStatements=true&characterEncoding=UTF-8&serverTimezone=Asia/Seoul&useSSL=true
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      pool-name: HikariCP-Primary
      # Sizing: (core_count * 2) + effective_spindle_count
      maximum-pool-size: 10  
      minimum-idle: 5
      idle-timeout: 300000     # 5 mins
      connection-timeout: 5000 # 5 secs (Fail fast)
      max-lifetime: 1200000    # 20 mins (Must be shorter than DB wait_timeout)
      auto-commit: true
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true
        useLocalSessionState: true
        rewriteBatchedStatements: true
        cacheResultSetMetadata: true
        cacheServerConfiguration: true
        elideSetAutoCommits: true
        maintainTimeStats: false

  jpa:
    database-platform: org.hibernate.dialect.MySQLDialect
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        show_sql: false
        format_sql: false
        jdbc:
          batch_size: 100
          order_inserts: true
          order_updates: true
        query:
          in_clause_parameter_padding: true
```

## Conclusion

HikariCP isn't just "fast"; it's engineered at the bytecode level to respect CPU architecture. By combining its lean design with the proper MySQL driver properties, you can significantly reduce latency in your backend services.