# System Design Learnings

## How do you handle image optimization for different device types and network conditions to balance quality and performance?

**Context:** [Design a News Aggregator](https://www.hellointerview.com/learn/system-design/problem-breakdowns/google-news)

**Relevant Topics:**

- Web optimization
- Content negotiation

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

## How do you ensure that swipe actions are processed both consistently and rapidly, so that when a user likes someone who has already liked them—even if only moments before—they are immediately notified of the match?

**Context:** [Design Tinder](https://www.hellointerview.com/learn/system-design/problem-breakdowns/tinder)

**Relevant Topics:**

- Redis
- Concurrency

### Approach 1: Using sets in Redis to track swipes and detect matches

When a user swipes on a target user

```javascript
const isMatch = await redis.eval(
  luaScript, // script
  2, // num of keys
  userA, // KEYS[1]
  targetUser // KEYS[2]
);
```

We perform the following operations in a lua script because we need atomicity. Since redis is single-threaded, the lua script runs atomically.

```lua
local user_id = KEYS[1]
local target_id = KEYS[2]

-- add the swipe to a redis set that tracks the likes for a particular user
redis.call('SADD' 'user:' .. user_id .. ':likes', target_id)

-- check if user exists in set for
local isMatch = redis.call('SISMEMBER', 'user:' .. target_id .. ':likes', user_id)

if is_match == 1 then
  -- if we have a match, add to a separate set that tracks matches
  redis.call('SADD', 'matches:', .. user_id, target_id)
  redis.call('SADD', 'matches:', .. target_id, user_id)
end

return is_match
```

What happens

- When user A swipes on user B, user B will be added to user A's set, the command finishes
- Subsequently, user B swipes on user A. User A will be added to user B's set.
- We check if user B exists in user A's set. There is a match
- We proceed to store the match and notify both users

**Pros:**

- ✅ Fast as it is using redis
- ✅ No concurrency issues because Redis is single threaded

**Cons:**

- ❌ We need to setup additional measures to persist the data, such as using CDC or streams in Redis
- ❌ We need to handle node failures in Redis - what consistency and guarantees can be maintained in such events
