---
title: "网关限流的策略"
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

API 密钥限流是用户侧限流的第一道防线，针对已认证用户进行资源分配和使用控制。

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

IP 限流主要用于防止恶意攻击和保护未认证请求路径，是系统安全的重要保障。

#### 自适应阈值

我们实现了基于历史流量模式的自适应阈值：

1. **基础限制**：所有 IP 都有默认的基础请求限制
2. **信誉系统**：根据 IP 的历史行为动态调整限制
3. **异常检测**：识别异常流量模式，对可疑 IP 实施更严格限制

#### 代码实现

```rust
pub struct IpRateLimiter {
    redis_client: Arc<RedisClient>,
    suspicious_ip_bloom: BloomFilter<String>,
    ip_reputation: HashMap<String, f32>,
}

impl IpRateLimiter {
    pub fn check_limit(&self, ip_address: &str, limit: u32) -> Result<bool, Error> {
        // 预筛选：检查是否为可疑 IP（根据历史数据）
        if self.suspicious_ip_bloom.might_contain(ip_address) {
            // 对可疑 IP 使用更严格的限制
            let strict_limit = limit / 2;
            return self.check_rate(ip_address, strict_limit);
        }

        // 根据 IP 信誉调整限制
        let reputation_factor = self.ip_reputation.get(ip_address).unwrap_or(&1.0);
        let adjusted_limit = (limit as f32 * reputation_factor) as u32;

        self.check_rate(ip_address, adjusted_limit)
    }
}
```

### 请求特征限流

请求特征限流是 AI 网关特有的限流维度，针对请求参数和资源消耗进行精细控制。

#### 差异化限流策略

我们基于以下特征实施差异化限流：

1. **模型差异化限流**：不同模型设置不同 QPS 限制（如 GPT-4 限制为 5 QPS，GPT-3.5 为 20 QPS）
2. **参数敏感限流**：大 context_length 请求（如 16K tokens）设置更严格限制
3. **端点差异化**：不同 API 端点（如聊天补全、嵌入）设置不同限制

#### 代码实现

```rust
pub struct RequestFeatureLimiter {
    model_limits: HashMap<String, ModelLimits>,
    endpoint_limits: HashMap<String, u32>,
}

impl RequestFeatureLimiter {
    pub fn check_limit(&self,
                      api_key: &str,
                      model: &str,
                      endpoint: &str,
                      context_length: usize) -> LimitResult {
        // 1. 检查模型特定限制
        let model_limit = self.model_limits.get(model)
            .unwrap_or(&self.model_limits["default"]);

        // 2. 检查上下文长度限制
        let length_factor = if context_length > 8192 {
            0.5  // 大上下文请求限制更严格
        } else if context_length > 4096 {
            0.75
        } else {
            1.0
        };

        // 3. 检查端点限制
        let endpoint_limit = self.endpoint_limits.get(endpoint)
            .unwrap_or(&self.endpoint_limits["default"]);

        // 综合判断
        let effective_limit = (model_limit.qps as f32 * length_factor) as u32;
        if effective_limit < *endpoint_limit {
            // 使用更严格的限制
            self.check_rate(api_key, model, effective_limit)
        } else {
            self.check_rate(api_key, endpoint, *endpoint_limit)
        }
    }
}
```

### 智能排队机制

为了提高用户体验，我们实现了智能排队机制，在短期速率超限但资源允许的情况下，不直接拒绝请求而是进行排队处理。

#### 排队策略

1. **优先级排队**：基于用户等级和请求重要性确定队列优先级
2. **超时控制**：设置最大等待时间，避免资源无限占用
3. **动态调整**：根据系统负载动态调整队列处理速度

#### 代码实现

```rust
pub fn handle_request_burst(&self, user_id: &str, request: Request) -> Response {
    // 检查用户当前使用量
    let usage = self.get_current_usage(user_id);
    let limit = self.get_user_limit(user_id);

    if usage > limit {
        // 完全超限，直接拒绝
        return Response::rate_limited(429, "Rate limit exceeded");
    } else if usage > limit * 0.8 {
        // 接近限制，尝试排队
        let priority = self.get_user_priority(user_id);
        let wait_time = self.queue_manager.enqueue(request, priority);

        if wait_time > MAX_WAIT_TIME {
            // 等待时间过长，拒绝请求
            return Response::rate_limited(429, "Queue too long");
        }

        // 请求进入队列，等待处理
        return Response::queued(202, wait_time);
    }

    // 未超限，正常处理
    self.process_request(request)
}
```

## 服务侧限流

服务侧限流是管理发往后端 AI 服务提供商的请求频率，不同服务提供商的资源消耗和响应时间不同，需要不同的限流策略。

## 多实例协同

在分布式部署环境中，多个网关实例需要协同工作，确保限流的一致性和准确性。

#### 协同机制

1. **计数同步**：使用 Redis 作为中央存储，确保计数一致性
2. **配置共享**：限流配置统一存储，所有实例实时获取最新配置
3. **状态广播**：关键状态变更（如紧急限流）通过发布订阅机制广播

通过以上机制，我们实现了高性能、高可靠的用户侧限流系统，既保障了系统的稳定性，又提供了良好的用户体验。
