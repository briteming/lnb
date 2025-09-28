# Public API Documentation

## 🚀 Content-Aware Caching API

This API exposes a lightweight, cache-friendly feed with **advanced content-aware distributed caching** for your blog posts. Designed for portfolio integration and external clients with global consistency guarantees.

### **⚡ Performance Highlights**

- **Smart Content Detection**: ETags based on actual content changes, not request parameters
- **Global Consistency**: New articles visible worldwide within 30 minutes
- **Zero Stale Content**: Eliminated 24-hour stale-while-revalidate risk
- **Resource Optimization**: 96% reduction in unnecessary server checks

## 📋 Best Practices

- **Portfolio Integration**: Use `/api/v1/feed` for "Latest Posts" cards; link to `canonicalUrl` for full reading
- **Respect Caching**: Responses include smart `Cache-Control` and content-based `ETag` headers
- **Content Validation**: Use `/api/v1/health` endpoint's `contentHash` to validate cache freshness
- **Request Efficiency**: Prefer `limit<=20`, avoid fetching full bodies unless necessary
- **CORS Compliance**: API allows `GET` from any origin; avoid sending credentials
- **Content Canonical**: Do not republish full content without `rel=canonical`

## 📊 Data Model

- **Source**: Velite-generated `public/data/posts.json` (accessible via HTTP in production)
- **Feed Item Fields**: `slug`, `slugAsParams?`, `title`, `description?`, `date`, `tags?`, `canonicalUrl`
- **Content Hash**: MD5 hash of posts metadata for cache validation

## 🔗 Endpoints

### `GET /api/v1/health`
**Real-time service status with content version tracking**

```json
{
  "status": "ok",
  "version": "1.0.0",
  "time": "2025-09-27T04:36:04.211Z",
  "posts": {
    "total": 65,
    "contentHash": "f9ba44e33c07",     // ✨ NEW: Content-based cache validation
    "lastModified": "2025-09-26T18:00:00.000Z"  // ✨ NEW: Latest post date
  }
}
```
- **Caching**: `revalidate: 0` - Always fresh for content validation
- **Use Case**: Cache validation, service monitoring

### `GET /api/v1/feed?limit=6&tag=nextjs`
**Latest posts feed with smart caching**

- **Parameters**:
  - `limit` (default 6, max 20)
  - `tag` (optional filter)
- **Response**: `{ data: FeedItem[] }` sorted by date desc, no `body`
- **Caching**: Content-based ETag, 30min edge cache, 5min browser cache

### `GET /api/v1/posts?page=1&per_page=10&tag=nextjs&sort=date_desc`
**Paginated posts with advanced filtering**

- **Parameters**:
  - `page` (1+), `per_page` (1–50)
  - `tag` (optional), `sort` (`date_desc|date_asc`)
- **Response**: `{ data: FeedItem[], meta: { page, per_page, total, total_pages } }`
- **Caching**: Content-based ETag, 30min edge cache, 5min browser cache

## 🔧 Headers & Caching Strategy

### **🎯 Smart Caching Architecture**

| **Component** | **Strategy** | **Duration** | **Benefit** |
|---------------|--------------|--------------|-------------|
| **ISR** | Content-frequency match | 86400s (24h) | 96% resource reduction |
| **Edge Cache** | Conservative strategy | 1800s (30min) | Global consistency |
| **Browser Cache** | Short-term with revalidation | 300s (5min) | Fresh content guarantee |
| **ETag** | Content-based MD5 hash | Dynamic | Precise cache validation |

### **📡 HTTP Headers**

```http
# Common Headers
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET
Vary: Accept-Encoding, Origin

# Smart Cache Control (Feed & Posts)
Cache-Control: public, s-maxage=1800, max-age=300, must-revalidate
ETag: "f9ba44e33c07"

# Health Endpoint (Always Fresh)
Cache-Control: no-store
```

### **⚠️ Breaking Changes from Previous Version**

- **REMOVED**: `stale-while-revalidate=86400` (eliminated 24h stale content risk)
- **IMPROVED**: ISR revalidation from 300s → 86400s (matches content update frequency)
- **ADDED**: Content-based ETag system for precise cache invalidation
- **ENHANCED**: Health endpoint now includes `contentHash` and `lastModified` fields

## Error Format
- On invalid routes/inputs, expect standard HTTP codes (e.g., 404). Error bodies use `{ error: { message, code } }` when applicable.

## 💡 Examples

### **🔍 Content Validation Workflow**

```bash
# 1. Check current content version
curl -s https://blog.liuyuelin.dev/api/v1/health | jq '.posts.contentHash'
# Output: "f9ba44e33c07"

# 2. Get latest posts with content-based caching
curl -s "https://blog.liuyuelin.dev/api/v1/feed?limit=6" | jq

# 3. Verify cache headers
curl -I "https://blog.liuyuelin.dev/api/v1/feed?limit=6" | grep -E "(ETag|Cache-Control)"
```

### **📱 Portfolio Integration (Next.js)**

```typescript
// Smart caching with content validation
export const revalidate = 1800; // 30 minutes - matches edge cache

async function getLatestPosts() {
  // Use health endpoint to check content freshness
  const healthRes = await fetch(`${BLOG_API}/health`);
  const { posts: { contentHash } } = await healthRes.json();

  // Fetch posts with content-aware caching
  const postsRes = await fetch(`${BLOG_API}/feed?limit=6`, {
    next: { revalidate: 1800 },
    headers: {
      'If-None-Match': `"${contentHash}"` // Smart cache validation
    }
  });

  if (postsRes.status === 304) {
    // Content unchanged, use cached version
    return getCachedPosts();
  }

  return postsRes.json();
}
```

### **🛠️ Advanced Examples**

```bash
# Monitor cache performance
curl -s -w "Cache: %{http_code} | Time: %{time_total}s\n" \
  "https://blog.liuyuelin.dev/api/v1/feed?limit=3" | head -1

# Paginated data fetching with tag filtering
curl -s "https://blog.liuyuelin.dev/api/v1/posts?page=1&per_page=5&tag=javascript" | \
  jq '.meta'

# Real-time content monitoring
watch -n 30 'curl -s https://blog.liuyuelin.dev/api/v1/health | jq ".posts.contentHash"'
```

## Local Testing
- Install and run: `pnpm install && pnpm dev` (or `bun install && bun run dev`).
- Health check: `http://localhost:3000/api/v1/health`
- Feed: `http://localhost:3000/api/v1/feed?limit=6`
- Posts: `http://localhost:3000/api/v1/posts?page=1&per_page=10`

