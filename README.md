# System Design Learnings

## How do you handle image optimization for different device types and network conditions to balance quality and performance?

**Context:** [Design a News Aggregator](https://www.hellointerview.com/learn/system-design/problem-breakdowns/google-news)

### Approach 1: Client Driven content negotiation: Use HTML element with srcset attributes

```html
<img
  src="image-800w.jpg"
  srcset="image-400w.jpg 400w, image-800w.jpg 800w, image-1200w.jpg 1200w"
  sizes="(max-width: 600px) 400px, 
            (max-width: 1000px) 800px, 
            1200px"
/>
```

**Pros:**

- ✅ caching logic is simple, by URL

**Cons:**

- ❌ Less control as decision of what image to fetch is by browser
- ❌ Need to maintain all the URLs for various image types

### Approach 2: Server Driven content negotiation: Use Accept header to decide what image to return

```
GET /api/image/123
Accept: image/webp,image/jpeg;q=0.8
User-Agent: Mobile Safari...

Response:
Vary: Accept, User-Agent // critical for caching, this tells caches that response varies based on Accept and User Agent headers
Content-Type: image/webp
Cache-Control: public, max-age=3600
```

**Pros:**

- ✅ Control is retained by server, allows for experimentation

**Cons:**

- ❌ Caching logic is more complicated, since there is only 1 URL
- ❌ Harder to debug, since it is the same URL and we need to inspect the cache

**Decision:** Client driven for simplicity, server driver for personalisation requirements
