---
title: 网关限流的策略
date: 2024-04-18T18:12:04+08:00
tags:
  - 网关
  - 系统设计
draft: false
---

最近在做一个基于 Pingora 的 AI 网关，限流是其中一个重要问题。在这里做一下记录

## 为什么网关需要限流

网关作为系统的入口，承担着连接客户端和后端服务的重要角色。限流机制在网关层面实现有以下几个关键原因：

1. **保护后端服务**：防止突发流量导致后端服务过载崩溃，确保系统稳定性
2. **资源合理分配**：在有限资源条件下，确保资源被公平合理地分配给各个用户
3. **防止恶意攻击**：限制单一来源的请求频率，有效防范 DDoS 等恶意攻击
4. **服务质量保障**：通过限制总体流量，保证系统响应时间和服务质量
5. **成本控制**：特别是对于 AI 服务，限流直接关系到 API 调用成本控制

## 常见限流算法

下面先解释几种常见的限流算法。固定窗口计数器，滑动窗口计数器，令牌桶算法，和 漏桶算法。

### 固定窗口计数器（Fixed Window Counter）

固定窗口计数器是最简单直观的限流算法：

- 将时间划分为固定长度的窗口（如 1 分钟、1 小时）
- 在每个窗口内维护一个计数器，记录请求数量
- 当计数器达到预设阈值时，拒绝后续请求
- 在新窗口开始时，计数器重置为零

给出一个实现的示例代码：

```rust
struct FixedWindowCounter {
    window_size_ms: u64,  // 窗口大小（毫秒）
    max_requests: u32,    // 窗口内最大请求数
    current_count: u32,   // 当前计数
    window_start: u64,    // 当前窗口开始时间
}

impl FixedWindowCounter {
    fn allow_request(&mut self, current_time_ms: u64) -> bool {
        // 检查是否需要重置窗口
        if current_time_ms >= self.window_start + self.window_size_ms {
            self.current_count = 0;
            self.window_start = current_time_ms;
        }

        // 检查是否允许请求
        if self.current_count < self.max_requests {
            self.current_count += 1;
            return true;
        }

        false
    }
}
```

**优点：** 实现简单，计算开销小，内存占用少，适合简单场景和单机部署。

**缺点：** 最大的缺点是存在边界突刺问题，即在窗口边界可能出现瞬间流量翻倍的情况。比如用户在 12:00:59 发送 100 个请求，然后在 12:01:01 又发送 100 个请求，实际上 2 秒内发送了 200 个请求，远超期望的速率。

### 滑动窗口计数器（Sliding Window）

滑动窗口算法通过更精细的时间划分，解决固定窗口的边界问题：

- 将时间窗口细分为多个小格（bucket）
- 每个格子记录各自时间段内的请求数
- 窗口随时间推移，动态计算当前有效时间范围内的请求总数
- 当总数超过阈值时拒绝请求

```rust
struct SlidingWindow {
    window_size_ms: u64,       // 窗口大小（毫秒）
    bucket_size_ms: u64,       // 每个桶的大小（毫秒）
    max_requests: u32,         // 窗口内最大请求数
    buckets: Vec<(u64, u32)>,  // (桶开始时间, 请求数)
}

impl SlidingWindow {
    fn allow_request(&mut self, current_time_ms: u64) -> bool {
        // 移除过期的桶
        let expire_time = current_time_ms - self.window_size_ms;
        self.buckets.retain(|(time, _)| *time >= expire_time);

        // 计算当前总请求数
        let total_requests: u32 = self.buckets.iter().map(|(_, count)| count).sum();

        // 检查是否允许请求
        if total_requests < self.max_requests {
            // 找到或创建当前时间对应的桶
            let current_bucket_time = current_time_ms / self.bucket_size_ms * self.bucket_size_ms;
            if let Some(idx) = self.buckets.iter().position(|(time, _)| *time == current_bucket_time) {
                self.buckets[idx].1 += 1;
            } else {
                self.buckets.push((current_bucket_time, 1));
            }
            return true;
        }

        false
    }
}
```

**优点：**: 避免边界突刺问题，能够较好地处理突发流量。

**缺点：**: 实现复杂度高于固定窗口，内存占用较大，特别是在分布式环境中同步复杂度增加。

### 令牌桶算法（Token Bucket）

令牌桶算法是一种更灵活的限流方式：

- 系统维护一个固定容量的令牌桶
- 以固定速率向桶中添加令牌（如每秒 10 个）
- 每个请求消耗一个或多个令牌
- 如果桶中令牌不足，请求被拒绝或等待
- 令牌桶可以存储一定数量的令牌，允许突发流量

```rust
struct TokenBucket {
    capacity: u32,           // 桶容量
    rate_per_ms: f64,        // 每毫秒生成的令牌数
    tokens: f64,             // 当前令牌数
    last_refill_time: u64,   // 上次填充令牌的时间
}

impl TokenBucket {
    fn allow_request(&mut self, current_time_ms: u64, tokens_required: u32) -> bool {
        // 计算从上次填充到现在应该添加的令牌数
        let time_passed = current_time_ms - self.last_refill_time;
        let new_tokens = time_passed as f64 * self.rate_per_ms;

        // 更新当前令牌数，不超过容量
        self.tokens = (self.tokens + new_tokens).min(self.capacity as f64);
        self.last_refill_time = current_time_ms;

        // 检查令牌是否足够
        if self.tokens >= tokens_required as f64 {
            self.tokens -= tokens_required as f64;
            return true;
        }

        false
    }
}
```

**优点：**: 能够较好地处理突发流量，提供缓冲能力。基于资源消耗的差异化限流。

**缺点：**: 实现复杂，时间精度要求高，影响限流精确性。分布式实现需要额外机制保证一致性。

### 漏桶算法（Leaky Bucket）

漏桶算法提供了一种更为严格的速率控制方式：

- 将请求比作水滴，放入上方开口的漏桶中
- 漏桶以固定速率出水（处理请求）
- 当桶满时，新请求被丢弃
- 无论输入速率如何变化，输出速率始终恒定

```rust
struct LeakyBucket {
    capacity: u32,           // 桶容量
    leak_rate_per_ms: f64,   // 每毫秒处理的请求数
    water_level: u32,        // 当前水位（请求数）
    last_leak_time: u64,     // 上次漏水时间
}

impl LeakyBucket {
    fn allow_request(&mut self, current_time_ms: u64) -> bool {
        // 计算从上次漏水到现在应该处理的请求数
        let time_passed = current_time_ms - self.last_leak_time;
        let leaked = (time_passed as f64 * self.leak_rate_per_ms) as u32;

        // 更新当前水位，不低于0
        self.water_level = self.water_level.saturating_sub(leaked);
        self.last_leak_time = current_time_ms;

        // 检查桶是否有空间
        if self.water_level < self.capacity {
            self.water_level += 1;
            return true;
        }

        false
    }
}
```

**优点：** 适合需要稳定出口速率的场景。

**缺点：** 不允许突发流量，可能导致资源浪费。请求可能需要等待较长时间。

### 算法选择和组合

在实际应用中，我们往往需要根据具体场景选择合适的限流算法，甚至将多种算法组合使用：

1. **固定窗口**：适用于简单场景和统计类需求
2. **滑动窗口**：适用于需要平滑控制且资源充足的场景
3. **令牌桶**：适用于允许突发流量但需要控制长期平均速率的场景
4. **漏桶**：适用于需要严格控制出口速率的场景

在 Charon AI 网关中，我们主要采用令牌桶和滑动窗口的组合策略，针对不同维度（用户、IP、服务）采用不同的算法，以实现最佳的限流效果。

## AI 网关相比传统网关的特殊性

AI 网关与传统 API 网关相比，具有一些独特的特性和挑战：

1. **资源消耗差异大**：AI 请求的资源消耗差异极大，例如大模型推理与小模型推理的计算资源需求可能相差 10 倍以上
2. **请求参数影响资源**：输入 token 数量、上下文长度等参数直接影响资源消耗，需要参数级别的限流
3. **响应时间不确定**：AI 模型的响应时间波动较大，需要更复杂的超时和重试策略
4. **多维度计费模型**：AI 服务通常基于多种指标计费（如 token 数、请求数），限流需要考虑多维度
5. **服务质量敏感**：用户对 AI 服务的响应时间和稳定性要求较高，需要更精细的限流策略

## AI 网关的限流策略

基于上面的分析，AI 网关的限流策略可以分为两个主要方向：

1. **用户侧限流**：控制来自客户端的请求频率和总量，在 AI 网关的特殊性下，需要更复杂的限流策略，针对用户、IP、服务等多个维度进行限流。
2. **服务侧限流**：管理发往后端 AI 服务提供商的请求频率，不同服务提供商的资源消耗和响应时间不同，需要不同的限流策略。比如 chatgpt 和 deepseek 的价格，响应时间，资源消耗都不同。

这种双向限流机制既避免 Token 被滥用，也确保了用户资源的合理分配。

## 用户侧限流

当一个请求进入 AI 网关时，会经历以下限流检查流程：

1. **请求预处理**：解析请求头，提取 API 密钥、IP 地址、请求路径等信息
2. **多维度限流检查**：
   - **API 密钥限流**：检查该密钥的请求频率和总量是否超限
   - **IP 限流**：检查源 IP 地址的请求频率是否超限,防止单一 IP 发送过多请求
   - **请求特征限流**：基于请求参数（如模型类型、上下文长度）进行限流判断,对某些高消耗的端点（如长上下文模型请求）设置更严格的限制
3. **限流决策**：
   - 如果任一维度超限，返回 429 状态码，并附带 Retry-After 头
   - 如果接近限制但未超限，可能进入排队机制
   - 如果未超限，请求继续处理
4. **计数更新**：请求通过限流检查后，更新相应维度的计数器
5. **请求处理**：将请求转发到后端 AI 服务

整个流程的核心是确保在请求实际处理前就完成限流决策，避免资源浪费。

### 分布式计数器系统

用户侧限流的核心是分布式计数器系统，它负责在分布式环境中准确追踪各维度的请求数量。

#### 架构设计

我们采用二级架构实现分布式计数器：

1. **Redis 作为共享计数存储**

   - 使用 Redis 的原子操作特性确保计数准确性
   - 键设计采用 `{user_id}:{resource_type}:{time_window}:{timestamp}` 格式, 比如 `user1:token:60:1708222400`
   - 实现自动过期机制，避免存储膨胀

2. **本地内存缓存层**
   - 每个网关实例维护本地计数缓存，减少网络请求
   - 定期与 Redis 同步，确保数据一致性
   - 使用 LRU 策略管理缓存大小

#### 滑动窗口实现

示例场景: 限制用户每分钟最多 100 次请求。传统固定窗口可能在窗口边界出现"突刺"，比如用户在 12:00:59 发送 100 个请求，然后在 12:01:01 又发送 100 个请求，实际上 2 秒内发送了 200 个请求，远超期望的速率。

我们的滑动窗口算法这样工作：

1. 把每分钟分成更小的时间桶，比如 6 个 10 秒的桶
2. 每个请求进来时，更新当前时间桶的计数
3. 计算窗口限制时，汇总当前时间往前推一个完整窗口内的所有桶的计数

具体代码逻辑类似这样：

```rust
pub fn check_and_increment(
    &self,
    user_id: &str,
    window_size_secs: u64,
    limit: u64
) -> Result<bool, Error> {
    let now = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs();

    // 计算当前桶的起始时间（以10秒为一个桶）
    let bucket_size = 10;
    let current_bucket = (now / bucket_size) * bucket_size;

    // 计算窗口内所有需要检查的桶
    let window_start = now - window_size_secs;
    let mut all_buckets = Vec::new();

    // 收集所有桶（整个窗口时间范围内）
    let mut bucket = current_bucket;
    while bucket >= window_start / bucket_size * bucket_size {
        all_buckets.push(bucket);
        if bucket >= bucket_size {
            bucket -= bucket_size;
        } else {
            break;
        }
    }

    // 使用Redis pipeline批量获取所有桶的计数
    let mut pipe = redis::pipe();
    let keys: Vec<String> = all_buckets.iter()
        .map(|b| format!("rate:{}:{}:{}", user_id, window_size_secs, b))
        .collect();

    for key in &keys {
        pipe.get(key);
    }

    let counts: Vec<Option<u64>> = pipe
        .query(&mut self.redis_conn)?;

    // 计算总请求数
    let total_count: u64 = counts.iter()
        .filter_map(|c| *c)
        .sum();

    // 判断是否超限
    if total_count >= limit {
        return Ok(false); // 已达到限制
    }

    // 未超限，递增当前桶的计数
    let current_key = format!("rate:{}:{}:{}", user_id, window_size_secs, current_bucket);

    // 原子递增并设置过期时间
    let _: () = redis::pipe()
        .atomic()
        .incr(&current_key, 1)
        .expire(&current_key, window_size_secs as usize + 60) // 窗口大小+额外余量
        .query(&mut self.redis_conn)?;

    Ok(true) // 未达到限制，允许请求
}
```

### API 密钥限流

API 密钥限流是用户侧限流的基础，针对已认证用户进行资源分配和使用控制。

#### 多级限流策略

我们实现了多级限流策略：

1. **短期速率限制**：控制每秒/每分钟的请求数量（QPS/QPM），使用上面的滑动窗口算法实现
2. **长期配额限制**：控制每日/每月的总请求量或 token 消耗量，使用上面的分布式计数器实现
3. **差异化模型限制**：不同模型类型设置不同的限制阈值，使用差异化的分布式计数器实现

```rust
pub struct ApiKeyRateLimiter {
    redis_client: Arc<RedisClient>,
    local_cache: Mutex<LruCache<String, u32>>,
    user_limits: HashMap<String, UserLimits>,
}

impl ApiKeyRateLimiter {
    pub fn check_limit(&self, api_key: &str, model: &str, request_size: usize) -> LimitResult {
        // 1. 检查短期速率限制（QPS）
        if !self.check_qps_limit(api_key, model).unwrap_or(false) {
            return LimitResult::RateLimited(5); // 建议 5 秒后重试
        }

        // 2. 检查长期配额限制（每日总量）
        if !self.check_daily_quota(api_key, request_size).unwrap_or(false) {
            return LimitResult::QuotaExceeded;
        }

        // 3. 检查模型特定限制
        if !self.check_model_specific_limit(api_key, model, request_size).unwrap_or(false) {
            return LimitResult::ModelLimited(10); // 建议 10 秒后重试
        }

        // 通过所有限制检查
        LimitResult::Allowed
    }
}

pub fn check_daily_quota(&self, api_key: &str, request_size: usize) -> Result<bool, Error> {
    // 1. 获取当前日期的时间戳（按天计算）
    let now = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs();
    let day_timestamp = (now / 86400) * 86400; // 取当天的起始时间戳（86400 = 24小时的秒数）

    // 2. 构造Redis键，格式为 "quota:{api_key}:daily:{day_timestamp}"
    let quota_key = format!("quota:{}:daily:{}", api_key, day_timestamp);

    // 3. 获取用户的每日配额限制
    let user_id = self.get_user_id_from_api_key(api_key)?;
    let daily_limit = self.user_limits
        .get(user_id)
        .map(|limits| limits.daily_quota)
        .unwrap_or(DEFAULT_DAILY_QUOTA);

    // 4. 原子操作：检查当前用量并增加请求大小
    let mut conn = self.redis_client.get_connection()?;
    let current_usage: u64 = conn.get(&quota_key).unwrap_or(0);

    // 5. 判断是否超出配额
    if current_usage + request_size as u64 > daily_limit {
        return Ok(false); // 配额已用尽
    }

    // 6. 未超出配额，增加用量并设置过期时间
    let _: () = redis::pipe()
        .atomic()
        .incr(&quota_key, request_size as u64)
        .expire(&quota_key, 86400 + 3600) // 设置为1天+1小时（额外余量防止边界问题）
        .query(&mut conn)?;

    Ok(true) // 未达到限制，允许请求
}

```

### IP 地址限流

IP 限流是用户侧限流的第一道防线，主要用于防止恶意攻击和保护未认证请求路径，是系统安全的重要保障。比如一个IP拿到多个密钥进行恶意请求, 可能挤占其他正常用户的请求。

这里对IP也做滑动窗口的限流，不过相比于api key来说单个IP的qps要高得多。
```rust
pub struct IpRateLimiter {
    redis_client: Arc<RedisClient>,
    // 滑动窗口大小（秒）
    window_sizes: Vec<u64>,
    // 对应窗口的限制次数
    limits: Vec<u64>,
    // 本地缓存，减少Redis访问
    local_cache: Mutex<LruCache<String, bool>>,
}

impl IpRateLimiter {
    pub fn new(redis_client: Arc<RedisClient>) -> Self {
        IpRateLimiter {
            redis_client,
            // 配置多个时间窗口：10秒、1分钟、10分钟
            window_sizes: vec![10, 60, 600],
            // 对应限制：20次/10秒，100次/分钟，500次/10分钟
            limits: vec![20, 100, 500],
            local_cache: Mutex::new(LruCache::new(1000)),
        }
    }
    
    pub fn check_ip(&self, ip: &str) -> Result<LimitResult, Error> {
        // 检查本地缓存中是否已标记为受限IP
        if let Some(&limited) = self.local_cache.lock().unwrap().get(ip) {
            if limited {
                return Ok(LimitResult::IpLimited(30)); // 建议30秒后重试
            }
        }
        
        // 对每个时间窗口进行检查
        for (i, &window_size) in self.window_sizes.iter().enumerate() {
            let limit = self.limits[i];
            
            // 使用滑动窗口检查IP请求频率
            if !self.check_window(ip, window_size, limit)? {
                // 限流触发，记录到本地缓存
                self.local_cache.lock().unwrap().put(ip.to_string(), true, Duration::from_secs(30));
                return Ok(LimitResult::IpLimited(window_size / 10)); // 建议等待时间
            }
        }
        
        Ok(LimitResult::Allowed)
    }
    
    fn check_window(&self, ip: &str, window_size: u64, limit: u64) -> Result<bool, Error> {
        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs();
            
        // 构造Redis键
        let key = format!("ip:{}:{}:{}", ip, window_size, now / window_size);
        
        let mut conn = self.redis_client.get_connection()?;
        let count: u64 = conn.get(&key).unwrap_or(0);
        
        if count >= limit {
            return Ok(false); // 已达到限制
        }
        
        // 原子递增并设置过期时间
        let _: () = redis::pipe()
            .atomic()
            .incr(&key, 1)
            .expire(&key, (window_size * 2) as usize) // 窗口大小的2倍作为过期时间
            .query(&mut conn)?;
            
        Ok(true) // 未达到限制
    }
}
```
另外还可以根据IP行为来做具体的限制，比如说IP的key有多个，认证多次错误，同一个时间并发请求更多.... 可以根据这些行为来做具体的评分进行限制。
## 服务侧限流

服务侧限流是AI网关架构中的另一个关键环节，专注于管理发往后端AI服务提供商的请求频率。与用户侧限流不同，服务侧限流需要考虑不同AI服务提供商的特性、限制和成本模型。在Charon AI网关中，我们主要采用令牌桶算法来实现服务侧的精细化流量控制。

### 令牌桶算法的应用

选择令牌桶算法作为服务侧限流的核心机制，主要基于以下几个考虑：

1. **突发流量处理能力**：令牌桶允许短期内的请求突发，这对于AI服务的不均匀负载分布尤为重要
2. **精确的速率控制**：可以通过调整令牌生成速率，精确控制对各个AI服务提供商的请求速率
3. **差异化资源分配**：支持基于请求特征（如模型类型、参数大小）进行差异化的令牌消耗策略

### 服务提供商特性适配

针对不同的AI服务提供商，我们的令牌桶配置有所不同：

```rust
pub struct ServiceRateLimiter {
    provider_buckets: HashMap<String, TokenBucket>,         // 服务提供商级别的令牌桶
    model_buckets: HashMap<String, HashMap<String, TokenBucket>>,  // 模型级别的令牌桶
    endpoint_buckets: HashMap<String, TokenBucket>,         // API端点级别的令牌桶
}

impl ServiceRateLimiter {
    pub fn new() -> Self {
        let mut limiter = ServiceRateLimiter {
            provider_buckets: HashMap::new(),
            model_buckets: HashMap::new(),
            endpoint_buckets: HashMap::new(),
        };
        
        // 配置不同服务提供商的令牌桶
        limiter.provider_buckets.insert(
            "openai".to_string(), 
            TokenBucket::new(100, 10.0)  // 容量100，每秒生成10个令牌
        );
        
        limiter.provider_buckets.insert(
            "anthropic".to_string(), 
            TokenBucket::new(80, 8.0)    // 容量80，每秒生成8个令牌
        );
        
        // 针对特定模型配置令牌桶
        let mut openai_models = HashMap::new();
        openai_models.insert(
            "gpt-4".to_string(),
            TokenBucket::new(50, 5.0)    // GPT-4限制更严格
        );
        
        openai_models.insert(
            "gpt-3.5-turbo".to_string(),
            TokenBucket::new(200, 20.0)  // GPT-3.5限制较宽松
        );
        
        limiter.model_buckets.insert("openai".to_string(), openai_models);
        
        // 配置特定端点的令牌桶
        limiter.endpoint_buckets.insert(
            "chat/completions".to_string(),
            TokenBucket::new(150, 15.0)
        );
        
        limiter
    }
}
```

### 动态令牌消耗策略

不同于传统API网关，AI请求的资源消耗差异非常大。我们根据请求特征动态计算令牌消耗量：

```rust
fn calculate_token_cost(&self, request: &Request) -> u32 {
    let base_cost = 1;  // 基础消耗1个令牌
    
    // 根据模型类型调整消耗
    let model_factor = match request.model.as_str() {
        "gpt-4" | "claude-3-opus" => 2.5,        // 大模型消耗更多
        "gpt-3.5-turbo" | "claude-instant" => 1.0,
        _ => 1.5,                                 // 默认消耗
    };
    
    // 根据输入token数量调整消耗
    let token_count = request.get_input_token_count();
    let token_factor = if token_count > 4000 {
        2.0  // 长文本消耗更多令牌
    } else if token_count > 2000 {
        1.5
    } else {
        1.0
    };
    
    // 根据请求类型调整消耗
    let request_factor = match request.endpoint.as_str() {
        "chat/completions" => 1.0,
        "embeddings" => 0.5,      // 嵌入请求消耗较少
        _ => 1.0,
    };
    
    // 计算最终令牌消耗
    (base_cost as f32 * model_factor * token_factor * request_factor) as u32
}
```

### 限流决策流程

当请求需要转发到后端服务时，限流器会执行一系列检查：

```rust
pub fn check_and_consume(&mut self, request: &Request) -> LimitResult {
    let provider = request.get_provider();
    let model = request.get_model();
    let endpoint = request.get_endpoint();
    
    // 计算此请求需要消耗的令牌数
    let token_cost = self.calculate_token_cost(request);
    let current_time_ms = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_millis() as u64;
    
    // 1. 检查服务提供商级别限制
    let provider_allowed = self.provider_buckets
        .get_mut(&provider)
        .map(|bucket| bucket.allow_request(current_time_ms, token_cost))
        .unwrap_or(true);  // 如果未配置，默认允许
    
    if !provider_allowed {
        return LimitResult::ProviderLimited(5);  // 建议5秒后重试
    }
    
    // 2. 检查模型级别限制
    let model_allowed = self.model_buckets
        .get_mut(&provider)
        .and_then(|models| models.get_mut(&model))
        .map(|bucket| bucket.allow_request(current_time_ms, token_cost))
        .unwrap_or(true);  // 如果未配置，默认允许
    
    if !model_allowed {
        return LimitResult::ModelLimited(3);  // 建议3秒后重试
    }
    
    // 3. 检查端点级别限制
    let endpoint_allowed = self.endpoint_buckets
        .get_mut(&endpoint)
        .map(|bucket| bucket.allow_request(current_time_ms, token_cost))
        .unwrap_or(true);  // 如果未配置，默认允许
    
    if !endpoint_allowed {
        return LimitResult::EndpointLimited(2);  // 建议2秒后重试
    }
    
    // 通过所有限制检查
    LimitResult::Allowed
}
```

### 自适应调整机制

为了应对AI服务提供商的动态变化（如错误率上升、响应时间延长），我们实现了令牌桶参数的自适应调整机制：

```rust
pub fn adjust_rate_based_on_metrics(&mut self, provider: &str, metrics: &ProviderMetrics) {
    if let Some(bucket) = self.provider_buckets.get_mut(provider) {
        // 根据错误率调整令牌生成速率
        if metrics.error_rate > 0.05 {  // 错误率超过5%
            let current_rate = bucket.get_rate_per_ms();
            let new_rate = current_rate * 0.8;  // 降低20%的速率
            bucket.set_rate_per_ms(new_rate);
            log::info!("Reducing rate for provider {} due to high error rate: {}", 
                       provider, metrics.error_rate);
        }
        
        // 根据响应时间调整令牌生成速率
        if metrics.avg_response_time_ms > metrics.baseline_response_time_ms * 1.5 {
            let current_rate = bucket.get_rate_per_ms();
            let new_rate = current_rate * 0.9;  // 降低10%的速率
            bucket.set_rate_per_ms(new_rate);
            log::info!("Reducing rate for provider {} due to increased latency: {}ms", 
                       provider, metrics.avg_response_time_ms);
        }
        
        // 如果指标恢复正常，逐步恢复速率
        if metrics.error_rate < 0.01 && 
           metrics.avg_response_time_ms < metrics.baseline_response_time_ms * 1.2 {
            let current_rate = bucket.get_rate_per_ms();
            let default_rate = self.get_default_rate(provider);
            if current_rate < default_rate {
                let new_rate = (current_rate * 1.1).min(default_rate);  // 增加10%但不超过默认值
                bucket.set_rate_per_ms(new_rate);
                log::info!("Restoring rate for provider {} as metrics improved", provider);
            }
        }
    }
}
```

### 指标收集与反馈

为了支持自适应调整，我们需要持续收集后端服务的性能指标：

```rust
pub struct ProviderMetrics {
    error_rate: f64,
    avg_response_time_ms: u64,
    baseline_response_time_ms: u64,
    request_success_count: u64,
    request_error_count: u64,
}

impl ServiceRateLimiter {
    pub fn update_metrics(&mut self, provider: &str, response: &Response, latency_ms: u64) {
        let metrics = self.metrics.entry(provider.to_string())
            .or_insert_with(|| ProviderMetrics {
                error_rate: 0.0,
                avg_response_time_ms: 0,
                baseline_response_time_ms: 200,  // 初始基准响应时间
                request_success_count: 0,
                request_error_count: 0,
            });
        
        // 更新请求计数
        if response.status_code >= 200 && response.status_code < 300 {
            metrics.request_success_count += 1;
        } else {
            metrics.request_error_count += 1;
        }
        
        // 更新错误率
        let total_requests = metrics.request_success_count + metrics.request_error_count;
        metrics.error_rate = metrics.request_error_count as f64 / total_requests as f64;
        
        // 更新平均响应时间（简单移动平均）
        metrics.avg_response_time_ms = (metrics.avg_response_time_ms * 9 + latency_ms) / 10;
        
        // 根据收集的指标调整令牌桶参数
        self.adjust_rate_based_on_metrics(provider, metrics);
    }
}
```

### 成本与公平性平衡

令牌桶算法还能够帮助我们平衡AI服务的成本与用户体验：

```rust
pub fn optimize_for_cost_and_fairness(&mut self) {
    // 根据当前时间段调整令牌生成速率
    let hour = Local::now().hour();
    
    for (provider, bucket) in &mut self.provider_buckets {
        let default_rate = self.get_default_rate(provider);
        
        // 高峰时段（工作时间）降低限制以控制成本
        if hour >= 9 && hour <= 17 {
            bucket.set_rate_per_ms(default_rate * 0.8);
        } 
        // 夜间时段放宽限制
        else if hour >= 22 || hour <= 6 {
            bucket.set_rate_per_ms(default_rate * 1.2);
        }
        // 其他时段使用默认限制
        else {
            bucket.set_rate_per_ms(default_rate);
        }
    }
}
```

### 集成与部署

在实际部署中，服务侧限流器与用户侧限流器相互配合，形成完整的流量控制系统：

```rust
pub fn process_request(&mut self, request: Request) -> Response {
    // 1. 用户侧限流检查
    let user_limit_result = self.user_limiter.check_limit(&request);
    if !user_limit_result.is_allowed() {
        return self.create_error_response(user_limit_result);
    }
    
    // 2. 服务侧限流检查
    let service_limit_result = self.service_limiter.check_and_consume(&request);
    if !service_limit_result.is_allowed() {
        return self.create_error_response(service_limit_result);
    }
    
    // 3. 转发请求到后端服务
    let start_time = Instant::now();
    let response = self.backend_client.send_request(request);
    let latency = start_time.elapsed().as_millis() as u64;
    
    // 4. 更新指标
    self.service_limiter.update_metrics(&request.get_provider(), &response, latency);
    
    response
}
```

## 多实例协同

在分布式部署环境中，多个网关实例需要协同工作，确保限流的一致性和准确性。
我们采用了云原生的部署模式，主要包括以下几个层次：

1. **负载均衡层**：通常使用云服务商提供的负载均衡器（如 AWS ALB/NLB）或 Kubernetes Ingress Controller 作为入口点
    
2. **网关集群**：多个无状态的 Charon 网关实例，水平部署在 Kubernetes 集群中
    - 每个实例完全相同，可独立处理请求
    - 使用 StatefulSet 或 Deployment 进行管理
    - 实例数量可以根据负载自动伸缩
3. **共享状态层**：
    - Redis 集群用于限流计数、缓存等共享状态
    - 配置中心（使用 ConfigMap 或专用配置服务）保存路由规则和限流策略
4. **监控与可观测性**：
    - Prometheus 指标收集
    - 分布式追踪（使用 OpenTelemetry）
    - 集中式日志收集

**典型部署案例**

一个中等规模的部署通常包括：

- 3-5 个 Charon 网关实例，每个实例配置 4 核 8GB 内存
- 3 节点 Redis 集群（主从架构）
- Kubernetes 作为编排平台，配置 HPA（Horizontal Pod Autoscaler）

这种架构能够处理每秒数千次 API 调用，同时保持高可用性和容错能力。

**请求调度流程**

当一个请求进入系统时，调度过程如下：

1. **入口路由**：
    - 请求首先到达负载均衡器
    - 负载均衡器根据简单的轮询或最少连接算法将请求分发到某个网关实例
2. **请求处理**：选中的网关实例接收请求后开始处理流程：
    - 解析请求，提取 API 密钥、目标模型等信息
    - 进行认证验证
    - 执行多层次限流检查（与 Redis 通信）
    - 根据路由规则确定目标 LLM 提供商
3. **智能路由决策**：
    - 检查目标服务的健康状态和历史延迟数据
    - 考虑当前负载和配额使用情况
    - 选择最优的目标端点（可能在多个提供商中选择）
4. **请求转发**：
    - 转换请求格式（如需）以适配目标 API
    - 添加必要的请求头和认证信息
    - 发送请求到目标 LLM 服务
5. **响应处理**：
    - 接收 LLM 服务响应
    - 格式转换（如需）
    - 记录使用量统计
    - 返回响应给客户端

**高可用性保障**

为确保系统高可用，我们实现了以下机制：

1. **实例冗余**：任何实例故障时，负载均衡器会自动将流量转移到健康实例
    
2. **健康检查**：
    - Kubernetes 的 liveness 和 readiness 探针监控实例健康
    - 实例内部自检机制，定期检查关键依赖（如 Redis 连接）
3. **熔断机制**：
    - 当检测到某个 LLM 提供商异常时自动熔断
    - 实现退避策略，避免雪崩效应
4. **降级策略**：
    - 当系统负载过高时，可以选择性地拒绝低优先级请求
    - 保障核心业务流程的稳定性

**实例间协调**

虽然每个实例独立工作，但它们通过共享状态实现协调：

1. **共享限流计数**：使用 Redis 存储限流计数器，确保全局一致性
    
2. **健康状态同步**：
    
    - 各实例将服务发现的结果写入共享存储
    - 通过记录目标服务的响应时间和成功率，帮助其他实例做出更好的路由决策
3. **配置动态更新**：
    
    - 配置变更通过配置中心下发
    - 实例定期拉取或通过 webhook 接收配置更新
