Rate Limiting Advanced Plugin - Field Reference & Guidance
1. Purpose
The Rate Limiting Advanced plugin controls how many requests can be made to an API within a defined time window. At FIS, this plugin is configured with Redis to ensure consistency across multiple Kong nodes. This avoids situations where a client could bypass limits by distributing requests across different nodes.

2. Step-by-Step Configuration with Explanations
🔑 Identifiers
Identifier = ip
 → We use the client IP address as the identifier. This ensures rate limits apply even if the caller is unauthenticated.
We are not using compound_identifier.  For global setup, we keep it simple and only use IP. Compound identifiers (like IP + header) are more complex and not required in our use case. If other teams configure this at the route level, they can extend, but our default guidance is IP-based.



📊 Rate Limits
Limit = 100 This number balances protection and usability. It prevents misuse while still allowing legitimate clients enough throughput.
Future service tiers (bronze, silver, gold) could align with different limits.


Window Size = 60 seconds
One minute is a standard and understandable window for clients.
If smaller windows (like 1 second) were used, APIs would reject bursts of traffic too aggressively, which is not desired for most use cases.


Window Type = sliding


Sliding windows ensures smoother enforcement by continuously counting requests. And to avoid “burst resets” (clients send a large burst right after a reset) which usually happens with Fixed window type.



🗂 Dictionaries & Namespacing
Dictionary Name = kong_rate_limiting_counters
Stores counters used by the plugin.
This dictionary is maintained internally and doesn’t need to be changed.


Lock Dictionary Name = kong_locks
Default is fine; no need to change unless instructed.


Namespace = dev / uat / prod
Currently its not used as we’re using a global setup for the same. And doesn;t need any segregation
If used, namespaces should follow environment tags (dev, uat, prod).



🔄 Redis Configuration
Strategy = redis


Always use Redis for distributed counters in multi-node FIS deployments.


Host = master.kong-redis.t1sxxx.use1.cache.amazonaws.com


AWS ElastiCache Redis endpoint.


Database = 1


Keeps counters separate from other Redis usage.


SSL = true


Secures traffic to Redis, which is required for production.


Timeouts (connect/read/send) = 2000 ms


Protects APIs from slow Redis responses. Clients will fail fast instead of hanging.


Keepalive Pool Size = 256


Supports high concurrency by reusing connections.


Cluster Max Redirections = 5


Handles Redis cluster redirections in case of failover.


Note: Without Redis, each Kong node would count requests separately, meaning a user could hit multiple nodes and bypass limits. Redis ensures one shared counter.

🚫 Error Handling
Error Code = 429
This is the HTTP standard for “Too Many Requests.”
Clients (SDKs, libraries, and apps) know how to handle this properly.


Error Message = "API rate limit exceeded"


Clear and human-readable message.


Disable Penalty = false


We do not allow “soft” limits. Once the limit is reached, the client must stop.


This prevents clients from abusing APIs beyond defined thresholds.


Retry After Jitter Max = 0


We don’t add random retry jitter by default. Clients must retry after the window.


If we see retry storms in future, this setting can be revisited.



👥 Consumer & Groups
Consumer Groups = Not used


We’re using a global setup currently as of that we don’t need to use Consumer and consumer groups.


If we needed tiered limits in future (bronze, silver, gold), consumer groups would be useful.


Enforce Consumer Groups = false


Since consumer groups are not used, enforcement is always disabled.



🧾 Headers
Header Name


By default, Kong adds headers like X-RateLimit-Limit, X-RateLimit-Remaining.


These help clients understand their current usage.


Hide Client Headers = false


We do not hide headers. Keeping them visible allows clients to adjust their request patterns responsibly.



🌐 Protocols
Protocols = http, https, grpc, grpcs


We include all supported protocols to ensure limits apply universally.


Even if most APIs use https, it’s best to configure broadly.



3. Example decK Configuration
plugins:
  - name: rate-limiting-advanced
    enabled: true
    config:
      identifier: ip               
      limit: [100]                 
      window_size: [60]            
      window_type: sliding         
      strategy: redis             
      redis:
        host: master.kong-redis.t1sxxx.use1.cache.amazonaws.com
        port: 6379
        database: 1
        ssl: true
        timeout: 2000
        keepalive_pool_size: 256
        cluster_max_redirections: 5
      error_code: 429
      error_message: "API rate limit exceeded"
      hide_client_headers: false
      enforce_consumer_groups: false


4. Key Guidance
Always keep the plugin enabled; a disabled plugin means no protection.


Use IP identifiers for global APIs unless business need dictates consumer-based limits.


Stick to the standard limit (100/min) unless approved otherwise.


Always configure the Redis backend in production for shared counters.


Do not hide headers; they help clients self-regulate traffic.


