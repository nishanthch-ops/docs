Rate Limiting Advanced – Field Reference & Guidance

This section provides field-level guidance for configuring the Rate Limiting Advanced plugin in Kong. Instead of listing all possible permutations, we narrow the choices to the specific FIS setup.

identifier (string)
We are using ip. This ensures rate limits apply even if the caller is unauthenticated.
Compound identifiers (like IP + header) are not required in our use case and add unnecessary complexity.

limit (array[integer])
We are using [100]. This balances protection and usability by preventing misuse while still allowing legitimate throughput.
In the future, different service tiers (bronze, silver, gold) could use different limits.

window_size (array[integer])
We are using [60] seconds. A one-minute window is standard and easy for clients to understand.
Shorter windows (like 1s) would reject bursts too aggressively, which is not desirable.

window_type (string)
We are using sliding. Sliding windows provide smoother enforcement and avoid burst resets that happen with fixed windows.

dictionary_name (string)
We are using kong_rate_limiting_counters. This is the default and does not need to be changed.

lock_dictionary_name (string)
We are using kong_locks. This is the default value and should not be updated unless specifically instructed.

namespace (string)
Currently not configured.
Since we use a global setup, segregation by namespace is not needed. If required in the future, namespaces should follow environment tags (dev, uat, prod).

strategy (string)
We are using redis. Redis ensures counters are centralized and shared across Kong nodes, preventing bypass when clients hit multiple nodes.

redis.host (string)
Example: master.kong-redis.t1sxxx.use1.cache.amazonaws.com (AWS ElastiCache endpoint). Must point to a valid Redis cluster for the plugin to work.

redis.port (integer)
Using default port 6379. This value is unchanged unless Redis is configured differently.

redis.database (integer)
We are using Database 1. Database 0 is often reserved for system use, so a separate DB ensures isolation. More DBs can be added if Redis is shared across environments.

redis.ssl (boolean) / redis.ssl_verify (boolean)
ssl = true ensures encryption in transit.
ssl_verify = false is acceptable in lower environments.
In production, enable ssl_verify = true with proper certificates.

redis.connect_timeout, redis.read_timeout, redis.send_timeout (ms)
We are using 2000 ms (2s) for each. This gives a good balance between responsiveness and reliability. Increasing the values could improve tolerance but would cause slower failovers.

redis.keepalive_pool_size (integer)
We are using 256. Supports many concurrent Redis connections in high-traffic scenarios. Increase this value if connection exhaustion is observed.

redis.cluster_max_redirections (integer)
We are using 5. This handles Redis cluster redirections in case of failover.

error_code (integer)
We are using 429. This is the HTTP standard for “Too Many Requests” and is properly handled by clients and SDKs.

error_message (string)
We are using "API rate limit exceeded". This is clear and easy for clients to understand.

disable_penalty (boolean)
We are keeping it false. This enforces hard limits. Once the threshold is reached, clients are blocked. This prevents silent overuse of the API.

retry_after_jitter_max (integer)
We are using 0. No random jitter is applied. Clients must retry after the window.
If retry storms are seen in the future, jitter can be introduced.

consumer_groups / enforce_consumer_groups
Not configured. We use a global setup, so consumer groups are not required. In the future, consumer groups can be used for tiered limits.

headers (boolean)
Default headers (X-RateLimit-Limit, X-RateLimit-Remaining) are enabled.
hide_client_headers = false keeps them visible so clients can adjust their request behavior responsibly.

protocols (array[string])
We are using [http, https, grpc, grpcs]. This ensures limits apply across all supported protocols, not just HTTPS.

Example FIS Standard Setup

Identifier: ip

Limit: 100 requests

Window Size: 60 seconds (sliding)

Dictionary: kong_rate_limiting_counters

Lock Dictionary: kong_locks

Redis Host: AWS ElastiCache endpoint

Port: 6379

Database: 1

SSL: true

Timeouts: 2000ms

Keepalive Pool Size: 256

Cluster Max Redirections: 5

Error Code: 429

Error Message: "API rate limit exceeded"

Headers: Enabled (not hidden)

Consumer Groups: Not used

Protocols: http, https, grpc, grpcs
