# Caching Strategy

> Authoritative caching standards for consistent, performant cache implementation across all application layers.

## Purpose

Define caching patterns, invalidation strategies, and operational requirements that reduce latency, lower infrastructure costs, and maintain data consistency across all services.

## Core Principles

1. **Cache deliberately** — Every cached value has a clear reason, TTL, and invalidation strategy
2. **Correctness over speed** — Stale data that causes incorrect behaviour is worse than a cache miss
3. **Layered caching** — Use the right cache at the right layer (CDN, reverse proxy, application, database)
4. **Measure everything** — Monitor hit rates, latency impact, and memory consumption
5. **Plan for failure** — The application must function (degraded, not broken) when any cache is unavailable

## Cache Layers

```
Client                    CDN           Reverse Proxy       Application Cache       Database
┌─────────┐         ┌──────────┐      ┌─────────────┐     ┌──────────────────┐    ┌──────────┐
│ Browser  │────────▶│ CDN Edge │─────▶│ Nginx/Caddy │────▶│ Redis/In-Memory  │───▶│ DB Query │
│ Cache    │         │ Cache    │      │ Cache       │     │ Cache            │    │ Cache    │
└─────────┘         └──────────┘      └─────────────┘     └──────────────────┘    └──────────┘
  SWR, SW            Static assets     Full responses      Business objects        Materialised
  localStorage       HTML pages        Rate limiting        Session data           views
```

### When to Use Each Layer

| Layer | Best For | TTL Range | Invalidation |
| ----- | -------- | --------- | ------------ |
| Browser | Static assets, API responses | Minutes to days | Cache-Control headers |
| CDN | Static assets, public HTML, images | Hours to days | Purge API, versioned URLs |
| Reverse proxy | Full API responses, rate limit counters | Seconds to minutes | TTL expiry, cache tags |
| Application (Redis) | Session data, computed results, rate limits | Seconds to hours | Event-driven, TTL |
| Database | Aggregations, expensive joins | Minutes to hours | Materialised view refresh |

## HTTP Caching

### Cache-Control Headers

```typescript
// Static assets (immutable, long cache)
res.setHeader('Cache-Control', 'public, max-age=31536000, immutable');

// API responses (short cache, revalidate)
res.setHeader('Cache-Control', 'private, max-age=60, stale-while-revalidate=300');

// Sensitive data (no caching)
res.setHeader('Cache-Control', 'no-store');

// Shared public data (CDN-cacheable)
res.setHeader('Cache-Control', 'public, max-age=300, s-maxage=600');
```

### Header Reference

| Directive | Purpose | Example |
| --------- | ------- | ------- |
| `public` | CDN/proxy can cache | Static assets, public pages |
| `private` | Only browser can cache | User-specific data |
| `max-age=N` | Browser cache duration (seconds) | `max-age=3600` |
| `s-maxage=N` | CDN/proxy cache duration | `s-maxage=86400` |
| `no-cache` | Must revalidate before using cached version | Frequently changing data |
| `no-store` | Never cache anywhere | Sensitive data, auth tokens |
| `immutable` | Content will never change at this URL | Hashed static assets |
| `stale-while-revalidate=N` | Serve stale while fetching fresh in background | API responses |

### ETag and Conditional Requests

```typescript
// NestJS ETag implementation
import { createHash } from 'crypto';

@Get('products/:id')
getProduct(@Param('id') id: string, @Req() req: Request, @Res() res: Response) {
  const product = this.productsService.findById(id);
  const etag = createHash('md5').update(JSON.stringify(product)).digest('hex');

  res.setHeader('ETag', `"${etag}"`);
  res.setHeader('Cache-Control', 'private, max-age=0, must-revalidate');

  if (req.headers['if-none-match'] === `"${etag}"`) {
    return res.status(304).end();
  }

  return res.json(product);
}
```

### Next.js Caching Patterns

```typescript
// Static page (ISR with revalidation)
export const revalidate = 60; // Revalidate every 60 seconds

// API route caching
export async function GET() {
  const data = await fetchData();
  return Response.json(data, {
    headers: {
      'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
    },
  });
}

// Dynamic page (no cache)
export const dynamic = 'force-dynamic';
```

## Application Caching (Redis)

### Cache-Aside Pattern (Default)

The application checks cache first, falls back to database, and populates cache on miss:

```typescript
// TypeScript (with ioredis)
import Redis from 'ioredis';

class CacheService {
  constructor(private redis: Redis) {}

  async getOrSet<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttlSeconds: number = 300
  ): Promise<T> {
    const cached = await this.redis.get(key);
    if (cached) {
      return JSON.parse(cached);
    }

    const value = await fetcher();
    await this.redis.setex(key, ttlSeconds, JSON.stringify(value));
    return value;
  }

  async invalidate(pattern: string): Promise<void> {
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }
}

// Usage
const user = await cache.getOrSet(
  `user:${userId}`,
  () => prisma.user.findUnique({ where: { id: userId } }),
  600 // 10 minutes
);
```

### .NET Cache-Aside

```csharp
public class CacheService
{
    private readonly IDistributedCache _cache;
    private readonly DistributedCacheEntryOptions _defaultOptions;

    public CacheService(IDistributedCache cache)
    {
        _cache = cache;
        _defaultOptions = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5),
            SlidingExpiration = TimeSpan.FromMinutes(2)
        };
    }

    public async Task<T?> GetOrSetAsync<T>(
        string key,
        Func<Task<T>> factory,
        DistributedCacheEntryOptions? options = null)
    {
        var cached = await _cache.GetStringAsync(key);
        if (cached is not null)
            return JsonSerializer.Deserialize<T>(cached);

        var value = await factory();
        var json = JsonSerializer.Serialize(value);
        await _cache.SetStringAsync(key, json, options ?? _defaultOptions);
        return value;
    }
}
```

### Python Cache-Aside

```python
import redis
import json
from functools import wraps

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def cached(key_prefix: str, ttl: int = 300):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            cache_key = f"{key_prefix}:{':'.join(str(a) for a in args)}"
            cached_value = r.get(cache_key)
            if cached_value:
                return json.loads(cached_value)

            result = await func(*args, **kwargs)
            r.setex(cache_key, ttl, json.dumps(result, default=str))
            return result
        return wrapper
    return decorator

# Usage
@cached("user", ttl=600)
async def get_user(user_id: str):
    return await db.users.find_one({"_id": user_id})
```

## Cache Invalidation Strategies

### Event-Driven Invalidation

Invalidate cache when the underlying data changes:

```typescript
// NestJS service with cache invalidation
@Injectable()
export class UsersService {
  constructor(
    private prisma: PrismaService,
    private cache: CacheService,
    private events: EventEmitter2,
  ) {}

  async updateUser(id: string, dto: UpdateUserDto) {
    const user = await this.prisma.user.update({
      where: { id },
      data: dto,
    });

    // Invalidate specific cache entries
    await this.cache.invalidate(`user:${id}`);
    await this.cache.invalidate(`user:${user.email}`);
    await this.cache.invalidate('users:list:*');

    // Emit event for other services
    this.events.emit('user.updated', { userId: id });

    return user;
  }
}
```

### Cache Key Design

```typescript
// Key format: <entity>:<identifier>:<variant>
const CACHE_KEYS = {
  user:       (id: string) => `user:${id}`,
  userByEmail:(email: string) => `user:email:${email}`,
  userList:   (page: number, size: number) => `users:list:${page}:${size}`,
  userCount:  () => 'users:count',
  settings:   (userId: string) => `settings:${userId}`,
};
```

**Rules:**
- Use colons (`:`) as delimiters
- Include all parameters that affect the cached value
- Keep keys human-readable for debugging
- Use a consistent prefix per entity/domain

### TTL Guidelines

| Data Type | TTL | Rationale |
| --------- | --- | --------- |
| User session | 15-30 min | Security, sliding window |
| User profile | 5-10 min | Changes infrequently |
| List/search results | 1-5 min | Multiple records, higher change rate |
| Configuration/settings | 10-30 min | Rarely changes |
| Static reference data | 1-24 hours | Countries, currencies, categories |
| Computed aggregations | 5-60 min | Expensive to recompute |

## Cache Stampede Prevention

When a popular cache key expires, many concurrent requests may hit the database simultaneously. Prevent this with:

### Locking (Mutex)

```typescript
async getOrSetWithLock<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttlSeconds: number
): Promise<T> {
  const cached = await this.redis.get(key);
  if (cached) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  const acquired = await this.redis.set(lockKey, '1', 'EX', 10, 'NX');

  if (acquired) {
    try {
      const value = await fetcher();
      await this.redis.setex(key, ttlSeconds, JSON.stringify(value));
      return value;
    } finally {
      await this.redis.del(lockKey);
    }
  }

  // Wait and retry if lock not acquired
  await new Promise(resolve => setTimeout(resolve, 100));
  return this.getOrSetWithLock(key, fetcher, ttlSeconds);
}
```

### Stale-While-Revalidate

```typescript
async getWithStaleRevalidate<T>(
  key: string,
  fetcher: () => Promise<T>,
  freshTtl: number,   // How long data is considered fresh
  staleTtl: number     // How long stale data can be served
): Promise<T> {
  const entry = await this.redis.get(key);
  if (entry) {
    const { value, expiresAt } = JSON.parse(entry);
    if (Date.now() > expiresAt) {
      // Stale — serve it but refresh in background
      this.refreshInBackground(key, fetcher, freshTtl, staleTtl);
    }
    return value;
  }

  // Cache miss — fetch synchronously
  const value = await fetcher();
  await this.setCacheEntry(key, value, freshTtl, staleTtl);
  return value;
}
```

## Frontend Caching

### React Query / TanStack Query

```typescript
import { useQuery, useQueryClient } from '@tanstack/react-query';

// Fetch with caching
function useUser(userId: string) {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: () => api.getUser(userId),
    staleTime: 5 * 60 * 1000,      // Fresh for 5 minutes
    gcTime: 30 * 60 * 1000,         // Keep in memory for 30 minutes
    refetchOnWindowFocus: false,
  });
}

// Invalidate on mutation
function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: UpdateUserDto) => api.updateUser(data),
    onSuccess: (_, variables) => {
      queryClient.invalidateQueries({ queryKey: ['user', variables.id] });
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}
```

### SWR Defaults

| Setting | Default | Recommended |
| ------- | ------- | ----------- |
| `staleTime` | 0 | 1-5 min for most data |
| `gcTime` | 5 min | 15-30 min |
| `refetchOnWindowFocus` | true | false for non-critical data |
| `refetchOnReconnect` | true | Keep true |
| `retry` | 3 | 2-3 for API calls |

## Monitoring

### Key Metrics

| Metric | Target | Alert Threshold |
| ------ | ------ | --------------- |
| Cache hit rate | > 80% | < 60% |
| Cache latency (p99) | < 5 ms | > 20 ms |
| Memory usage | < 80% of max | > 90% |
| Eviction rate | Low and stable | Sudden spike |
| Connection pool usage | < 80% | > 90% |

### Redis Monitoring

```bash
# Check memory usage
redis-cli INFO memory | grep used_memory_human

# Check hit rate
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"

# Check connected clients
redis-cli INFO clients | grep connected_clients

# Monitor slow commands
redis-cli SLOWLOG GET 10
```

## Checklist

### Implementation

- [ ] Cache-aside pattern used (not write-through unless specifically needed)
- [ ] TTLs defined for all cached values
- [ ] Cache key naming convention followed
- [ ] Invalidation strategy documented for each cached entity
- [ ] Stampede prevention implemented for high-traffic keys
- [ ] Application handles cache unavailability gracefully

### HTTP Caching

- [ ] Static assets have `Cache-Control: immutable` with versioned URLs
- [ ] API responses have appropriate `Cache-Control` directives
- [ ] Sensitive data uses `no-store`
- [ ] ETags implemented for conditional requests where appropriate

### Operations

- [ ] Cache hit rate monitored and alerted
- [ ] Memory usage monitored with eviction alerts
- [ ] Redis connection pooling configured
- [ ] Cache can be flushed without application restart
- [ ] Cache warming strategy for cold starts (if needed)

## References

- [HTTP Caching — MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [Redis Best Practices](https://redis.io/docs/management/optimization/)
- [TanStack Query — Caching](https://tanstack.com/query/latest/docs/framework/react/guides/caching)
- [Cache Stampede — Wikipedia](https://en.wikipedia.org/wiki/Cache_stampede)
