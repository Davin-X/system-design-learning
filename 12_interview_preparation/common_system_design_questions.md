# Common System Design Interview Questions

This guide covers frequently asked system design interview questions, organized by complexity level, with detailed solutions and analysis. Each question includes requirements analysis, high-level design, component breakdowns, and trade-off discussions.

## Beginner Level Questions

### 1. Design a URL Shortening Service (TinyURL)

#### Requirements Analysis
- **Functional Requirements:**
  - Generate short URLs from long URLs
  - Redirect short URLs to original URLs
  - Custom short URLs (optional)
  - URL expiration (optional)

- **Non-Functional Requirements:**
  - High availability (99.9% uptime)
  - Low latency (<200ms read, <500ms write)
  - Scalability (handle millions of requests)
  - Uniqueness (no URL collisions)

- **Scale Estimates:**
  - 100 million URLs created daily
  - 1 billion redirects daily
  - URLs should not expire for 10 years

#### High-Level Design

```
┌─────────────┐    ┌─────────────────┐    ┌─────────────┐
│   Client    │────│   API Gateway   │────│   Web App   │
└─────────────┘    └─────────────────┘    └─────────────┘
                                              │
                                              ▼
┌─────────────┐    ┌─────────────────┐    ┌─────────────┐
│   Cache     │────│   Application   │────│  Database   │
│ (Redis)     │    │   Service       │    │ (MySQL)     │
└─────────────┘    └─────────────────┘    └─────────────┘
```

#### Component Design

##### URL Generation Algorithm
```java
@Service
public class UrlShorteningService {
    
    private static final String BASE62 = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    private static final int SHORT_URL_LENGTH = 7;
    
    @Autowired
    private UrlRepository urlRepository;
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public String shortenUrl(String originalUrl, String customAlias) {
        // Validate original URL
        if (!isValidUrl(originalUrl)) {
            throw new InvalidUrlException("Invalid URL format");
        }
        
        // Check if URL already exists
        Url existing = urlRepository.findByOriginalUrl(originalUrl);
        if (existing != null) {
            return existing.getShortCode();
        }
        
        String shortCode;
        if (customAlias != null && !customAlias.isEmpty()) {
            // Use custom alias if provided and available
            if (urlRepository.existsByShortCode(customAlias)) {
                throw new AliasAlreadyExistsException("Custom alias already in use");
            }
            shortCode = customAlias;
        } else {
            // Generate unique short code
            shortCode = generateUniqueShortCode();
        }
        
        // Save to database
        Url url = new Url();
        url.setOriginalUrl(originalUrl);
        url.setShortCode(shortCode);
        url.setCreatedAt(Instant.now());
        url.setExpiresAt(calculateExpiration());
        
        urlRepository.save(url);
        
        // Cache the mapping
        redisTemplate.opsForValue().set("url:" + shortCode, originalUrl, 
                                       Duration.ofDays(30));
        
        return shortCode;
    }
    
    public String getOriginalUrl(String shortCode) {
        // Try cache first
        String cachedUrl = redisTemplate.opsForValue().get("url:" + shortCode);
        if (cachedUrl != null) {
            // Update access stats asynchronously
            updateAccessStatsAsync(shortCode);
            return cachedUrl;
        }
        
        // Load from database
        Url url = urlRepository.findByShortCode(shortCode);
        if (url == null || url.getExpiresAt().isBefore(Instant.now())) {
            throw new UrlNotFoundException("URL not found or expired");
        }
        
        // Cache for future requests
        redisTemplate.opsForValue().set("url:" + shortCode, url.getOriginalUrl(), 
                                       Duration.ofDays(30));
        
        // Update access stats
        updateAccessStats(url);
        
        return url.getOriginalUrl();
    }
    
    private String generateUniqueShortCode() {
        String shortCode;
        do {
            // Generate random 7-character string
            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < SHORT_URL_LENGTH; i++) {
                sb.append(BASE62.charAt(ThreadLocalRandom.current().nextInt(BASE62.length())));
            }
            shortCode = sb.toString();
        } while (urlRepository.existsByShortCode(shortCode));
        
        return shortCode;
    }
    
    private Instant calculateExpiration() {
        // URLs expire after 2 years by default
        return Instant.now().plus(730, ChronoUnit.DAYS);
    }
    
    private boolean isValidUrl(String url) {
        try {
            new URL(url);
            return true;
        } catch (MalformedURLException e) {
            return false;
        }
    }
    
    @Async
    private void updateAccessStatsAsync(String shortCode) {
        // Update access statistics
        redisTemplate.opsForValue().increment("stats:access:" + shortCode);
    }
    
    private void updateAccessStats(Url url) {
        url.setAccessCount(url.getAccessCount() + 1);
        url.setLastAccessedAt(Instant.now());
        urlRepository.save(url);
    }
}
```

##### Database Schema
```sql
CREATE TABLE urls (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    short_code VARCHAR(10) NOT NULL UNIQUE,
    original_url TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP,
    access_count BIGINT DEFAULT 0,
    last_accessed_at TIMESTAMP,
    user_id BIGINT,
    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);

CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

#### Scaling Considerations
- **Database Sharding:** Shard by short_code hash for even distribution
- **Caching Strategy:** Cache URL mappings with TTL, popular URLs in memory
- **Rate Limiting:** Prevent abuse with request limits per IP/user
- **Analytics:** Track click rates, geographic distribution, referral sources

---

### 2. Design a Rate Limiter

#### Requirements Analysis
- **Functional Requirements:**
  - Limit requests per user/IP/time window
  - Support multiple rate limit types (sliding window, fixed window, token bucket)
  - Return appropriate HTTP status codes (429 Too Many Requests)
  - Allow bursting for occasional high traffic

- **Non-Functional Requirements:**
  - Low latency (<10ms overhead)
  - High throughput (handle millions of requests/second)
  - Distributed (work across multiple instances)
  - Configurable limits per endpoint/user

#### Implementation
```java
@Service
public class RateLimiterService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private final Map<String, RateLimitRule> rules = new ConcurrentHashMap<>();
    
    public RateLimiterService() {
        // Initialize default rules
        rules.put("/api/users", new RateLimitRule(100, Duration.ofMinutes(1))); // 100 req/min
        rules.put("/api/search", new RateLimitRule(10, Duration.ofSeconds(1))); // 10 req/sec
    }
    
    public RateLimitResult checkLimit(String key, String endpoint) {
        RateLimitRule rule = rules.getOrDefault(endpoint, 
            new RateLimitRule(60, Duration.ofMinutes(1))); // Default: 60 req/min
        
        return checkLimit(key, rule);
    }
    
    public RateLimitResult checkLimit(String key, RateLimitRule rule) {
        long currentTime = System.currentTimeMillis();
        long windowStart = currentTime - rule.getWindow().toMillis();
        
        String redisKey = "ratelimit:" + key;
        
        // Remove old requests outside the window
        redisTemplate.opsForZSet().removeRangeByScore(redisKey, 0, windowStart);
        
        // Count requests in current window
        Long requestCount = redisTemplate.opsForZSet().zCard(redisKey);
        
        RateLimitResult result = new RateLimitResult();
        result.setAllowed(requestCount < rule.getLimit());
        result.setCurrentRequests(requestCount);
        result.setLimit(rule.getLimit());
        result.setResetTime(currentTime + rule.getWindow().toMillis());
        
        if (result.isAllowed()) {
            // Add current request
            redisTemplate.opsForZSet().add(redisKey, UUID.randomUUID().toString(), currentTime);
            
            // Set expiration on the key (twice the window to allow for sliding)
            redisTemplate.expire(redisKey, rule.getWindow().multipliedBy(2));
        }
        
        return result;
    }
    
    public static class RateLimitRule {
        private final int limit;
        private final Duration window;
        
        public RateLimitRule(int limit, Duration window) {
            this.limit = limit;
            this.window = window;
        }
        
        // Getters
    }
    
    public static class RateLimitResult {
        private boolean allowed;
        private long currentRequests;
        private long limit;
        private long resetTime;
        
        // Getters and setters
    }
}

@RestController
@RequestMapping("/api")
public class ApiController {
    
    @Autowired
    private RateLimiterService rateLimiter;
    
    @Autowired
    private UserService userService;
    
    @GetMapping("/users")
    public ResponseEntity<?> getUsers(@RequestParam String apiKey, 
                                    HttpServletRequest request) {
        
        // Check rate limit
        String clientKey = apiKey != null ? apiKey : getClientIp(request);
        RateLimitResult limitResult = rateLimiter.checkLimit(clientKey, "/api/users");
        
        if (!limitResult.isAllowed()) {
            HttpHeaders headers = new HttpHeaders();
            headers.set("X-RateLimit-Limit", String.valueOf(limitResult.getLimit()));
            headers.set("X-RateLimit-Remaining", 
                       String.valueOf(Math.max(0, limitResult.getLimit() - limitResult.getCurrentRequests())));
            headers.set("X-RateLimit-Reset", String.valueOf(limitResult.getResetTime()));
            headers.set("Retry-After", String.valueOf(
                (limitResult.getResetTime() - System.currentTimeMillis()) / 1000));
            
            return ResponseEntity.status(429)
                               .headers(headers)
                               .body("Too Many Requests");
        }
        
        // Process request
        List<User> users = userService.getUsers();
        return ResponseEntity.ok(users);
    }
    
    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

---

## Intermediate Level Questions

### 3. Design a Notification Service

#### Requirements Analysis
- **Functional Requirements:**
  - Send notifications via email, SMS, push notifications
  - Support scheduled notifications
  - Handle user preferences (opt-in/opt-out)
  - Track delivery status and analytics

- **Non-Functional Requirements:**
  - High throughput (millions of notifications/day)
  - Low latency for immediate notifications
  - Reliable delivery (retry failed notifications)
  - Scalable (handle traffic spikes)

#### Architecture
```java
@Service
public class NotificationService {
    
    @Autowired
    private NotificationRepository notificationRepository;
    
    @Autowired
    private UserPreferencesRepository preferencesRepository;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private EmailService emailService;
    
    @Autowired
    private SmsService smsService;
    
    @Autowired
    private PushNotificationService pushService;
    
    public void sendNotification(NotificationRequest request) {
        // Validate request
        validateRequest(request);
        
        // Check user preferences
        if (!userWantsNotification(request.getUserId(), request.getType())) {
            return; // User has opted out
        }
        
        // Create notification record
        Notification notification = new Notification();
        notification.setId(generateId());
        notification.setUserId(request.getUserId());
        notification.setType(request.getType());
        notification.setTitle(request.getTitle());
        notification.setMessage(request.getMessage());
        notification.setStatus(NotificationStatus.PENDING);
        notification.setCreatedAt(Instant.now());
        
        if (request.getScheduledTime() != null) {
            notification.setScheduledTime(request.getScheduledTime());
            notification.setStatus(NotificationStatus.SCHEDULED);
        }
        
        notificationRepository.save(notification);
        
        if (request.getScheduledTime() == null) {
            // Send immediately
            sendImmediateNotification(notification);
        } else {
            // Schedule for later
            scheduleNotification(notification);
        }
    }
    
    private void sendImmediateNotification(Notification notification) {
        // Publish to appropriate channel based on type
        switch (notification.getType()) {
            case EMAIL:
                kafkaTemplate.send("email-notifications", "send", notification);
                break;
            case SMS:
                kafkaTemplate.send("sms-notifications", "send", notification);
                break;
            case PUSH:
                kafkaTemplate.send("push-notifications", "send", notification);
                break;
        }
    }
    
    private void scheduleNotification(Notification notification) {
        kafkaTemplate.send("scheduled-notifications", "schedule", notification);
    }
    
    private boolean userWantsNotification(Long userId, NotificationType type) {
        UserPreferences prefs = preferencesRepository.findByUserId(userId);
        if (prefs == null) {
            return true; // Default to allowing notifications
        }
        
        switch (type) {
            case EMAIL: return prefs.isEmailEnabled();
            case SMS: return prefs.isSmsEnabled();
            case PUSH: return prefs.isPushEnabled();
            default: return false;
        }
    }
}

@Service
public class EmailNotificationProcessor {
    
    @Autowired
    private NotificationRepository notificationRepository;
    
    @Autowired
    private EmailService emailService;
    
    @KafkaListener(topics = "email-notifications", groupId = "email-processor")
    public void processEmailNotification(String notificationJson) {
        try {
            Notification notification = objectMapper.readValue(notificationJson, Notification.class);
            
            // Send email
            boolean sent = emailService.sendEmail(
                getUserEmail(notification.getUserId()),
                notification.getTitle(),
                notification.getMessage()
            );
            
            // Update notification status
            notification.setStatus(sent ? NotificationStatus.SENT : NotificationStatus.FAILED);
            notification.setSentAt(sent ? Instant.now() : null);
            notificationRepository.save(notification);
            
        } catch (Exception e) {
            // Handle failure, could implement retry logic
            log.error("Failed to process email notification", e);
        }
    }
    
    @KafkaListener(topics = "scheduled-notifications", groupId = "scheduled-processor")
    public void processScheduledNotification(String notificationJson) {
        try {
            Notification notification = objectMapper.readValue(notificationJson, Notification.class);
            
            // Check if it's time to send
            if (notification.getScheduledTime().isBefore(Instant.now())) {
                sendImmediateNotification(notification);
                
                // Remove from scheduled queue
                // Implementation would remove from scheduled storage
            } else {
                // Re-queue for later processing
                kafkaTemplate.send("scheduled-notifications", "schedule", notification);
            }
            
        } catch (Exception e) {
            log.error("Failed to process scheduled notification", e);
        }
    }
}
```

---

### 4. Design a Search Service (Autocomplete)

#### Requirements Analysis
- **Functional Requirements:**
  - Provide autocomplete suggestions as user types
  - Search across multiple data types (users, posts, hashtags)
  - Support fuzzy matching and typo tolerance
  - Return results ranked by relevance

- **Non-Functional Requirements:**
  - Sub-100ms response time
  - Handle high QPS (thousands per second)
  - Support multiple languages
  - Scalable data ingestion

#### Implementation
```java
@Service
public class AutocompleteService {
    
    @Autowired
    private ElasticsearchClient elasticsearchClient;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PostRepository postRepository;
    
    public List<AutocompleteResult> getSuggestions(String query, int limit, String userId) {
        if (query == null || query.trim().isEmpty()) {
            return getTrendingSuggestions(limit);
        }
        
        // Check cache first
        String cacheKey = "autocomplete:" + query.toLowerCase() + ":" + limit;
        @SuppressWarnings("unchecked")
        List<AutocompleteResult> cached = (List<AutocompleteResult>) 
            redisTemplate.opsForValue().get(cacheKey);
        
        if (cached != null) {
            return cached;
        }
        
        // Perform search
        List<AutocompleteResult> results = performSearch(query, limit, userId);
        
        // Cache results for 5 minutes
        redisTemplate.opsForValue().set(cacheKey, results, Duration.ofMinutes(5));
        
        return results;
    }
    
    private List<AutocompleteResult> performSearch(String query, int limit, String userId) {
        List<AutocompleteResult> results = new ArrayList<>();
        
        // Search users
        results.addAll(searchUsers(query, limit / 3));
        
        // Search posts/hashtags
        results.addAll(searchPosts(query, limit / 3));
        
        // Search hashtags
        results.addAll(searchHashtags(query, limit / 3));
        
        // Rank and limit results
        return results.stream()
            .sorted(Comparator.comparing(AutocompleteResult::getScore).reversed())
            .limit(limit)
            .collect(Collectors.toList());
    }
    
    private List<AutocompleteResult> searchUsers(String query, int limit) {
        // Elasticsearch query for users
        SearchRequest searchRequest = new SearchRequest("users");
        
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        
        // Multi-match query on username, full name, bio
        MultiMatchQueryBuilder multiMatch = QueryBuilders.multiMatchQuery(query)
            .field("username", 3.0f)
            .field("fullName", 2.0f)
            .field("bio", 1.0f)
            .type(MultiMatchQueryBuilder.Type.BEST_FIELDS)
            .fuzziness(Fuzziness.AUTO);
        
        boolQuery.must(multiMatch);
        
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder()
            .query(boolQuery)
            .size(limit)
            .sort("_score", SortOrder.DESC);
        
        searchRequest.source(sourceBuilder);
        
        try {
            SearchResponse response = elasticsearchClient.search(searchRequest, RequestOptions.DEFAULT);
            
            return Arrays.stream(response.getHits().getHits())
                .map(hit -> {
                    Map<String, Object> source = hit.getSourceAsMap();
                    return new AutocompleteResult(
                        "user",
                        (String) source.get("username"),
                        (String) source.get("fullName"),
                        (Double) hit.getScore()
                    );
                })
                .collect(Collectors.toList());
                
        } catch (IOException e) {
            log.error("Failed to search users", e);
            return Collections.emptyList();
        }
    }
    
    private List<AutocompleteResult> searchPosts(String query, int limit) {
        // Similar implementation for posts
        return Collections.emptyList(); // Simplified
    }
    
    private List<AutocompleteResult> searchHashtags(String query, int limit) {
        // Search hashtags with prefix matching
        String hashtagQuery = query.startsWith("#") ? query : "#" + query;
        
        // Redis sorted set with hashtag frequencies
        Set<String> hashtags = redisTemplate.opsForZSet()
            .rangeByLex("hashtags", 
                Range.range().gte(hashtagQuery).lt(hashtagQuery + Character.MAX_VALUE),
                RedisZSetCommands.Limit.limit().count(limit));
        
        return hashtags.stream()
            .map(hashtag -> new AutocompleteResult("hashtag", hashtag, null, 1.0))
            .collect(Collectors.toList());
    }
    
    private List<AutocompleteResult> getTrendingSuggestions(int limit) {
        // Get trending searches from Redis
        Set<String> trending = redisTemplate.opsForZSet()
            .reverseRange("trending:searches", 0, limit - 1);
        
        return trending.stream()
            .map(term -> new AutocompleteResult("trending", term, null, 1.0))
            .collect(Collectors.toList());
    }
    
    public void indexUser(User user) {
        // Index user in Elasticsearch
        Map<String, Object> userDoc = Map.of(
            "id", user.getId(),
            "username", user.getUsername(),
            "fullName", user.getFullName(),
            "bio", user.getBio(),
            "followerCount", getFollowerCount(user.getId())
        );
        
        IndexRequest indexRequest = new IndexRequest("users")
            .id(user.getId().toString())
            .source(userDoc, XContentType.JSON);
        
        try {
            elasticsearchClient.index(indexRequest, RequestOptions.DEFAULT);
        } catch (IOException e) {
            log.error("Failed to index user {}", user.getId(), e);
        }
    }
    
    public void trackSearchQuery(String query, String userId) {
        // Track search queries for trending suggestions
        redisTemplate.opsForZSet().incrementScore("trending:searches", query, 1);
        
        // Clean up old trending searches periodically
        redisTemplate.opsForZSet().removeRange("trending:searches", 0, -1001); // Keep top 1000
    }
    
    private long getFollowerCount(Long userId) {
        // Implementation to get follower count
        return 0; // Simplified
    }
    
    public static class AutocompleteResult {
        private final String type;
        private final String text;
        private final String subtitle;
        private final double score;
        
        public AutocompleteResult(String type, String text, String subtitle, double score) {
            this.type = type;
            this.text = text;
            this.subtitle = subtitle;
            this.score = score;
        }
        
        // Getters
    }
}
```

---

## Advanced Level Questions

### 5. Design a Social Media Feed (Timeline)

#### Requirements Analysis
- **Functional Requirements:**
  - Generate personalized feed for each user
  - Support different feed types (chronological, algorithmic)
  - Handle millions of posts and billions of relationships
  - Real-time updates for new content

- **Non-Functional Requirements:**
  - Sub-200ms response time for feed loading
  - Handle peak loads during major events
  - Support 100M+ concurrent users
  - 99.9% availability

#### Fanout-on-Write Architecture
```java
@Service
public class FeedGenerationService {
    
    @Autowired
    private FollowRepository followRepository;
    
    @Autowired
    private PostRepository postRepository;
    
    @Autowired
    private FeedRepository feedRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    // Fanout thresholds
    private static final int SMALL_FOLLOWER_THRESHOLD = 1000;
    private static final int LARGE_FOLLOWER_THRESHOLD = 100000;
    
    public void publishPost(Post post) {
        Long authorId = post.getUserId();
        
        // Get follower count
        long followerCount = followRepository.countFollowers(authorId);
        
        if (followerCount <= SMALL_FOLLOWER_THRESHOLD) {
            // Fanout to all followers immediately
            fanoutToFollowers(post, authorId);
        } else if (followerCount <= LARGE_FOLLOWER_THRESHOLD) {
            // Fanout to active followers, others pull
            fanoutToActiveFollowers(post, authorId);
        } else {
            // Large accounts - cache for pull-based access
            cachePostForLargeAccount(post, authorId);
        }
        
        // Index post for search
        kafkaTemplate.send("post-indexing", "index", post);
    }
    
    private void fanoutToFollowers(Post post, Long authorId) {
        List<Long> followerIds = followRepository.findFollowerIds(authorId);
        
        // Process in batches to avoid overwhelming the system
        List<List<Long>> batches = Lists.partition(followerIds, 100);
        
        for (List<Long> batch : batches) {
            for (Long followerId : batch) {
                // Add post to user's feed
                feedRepository.addPostToFeed(followerId, post.getId());
                
                // Cache in Redis for fast access
                String feedKey = "feed:" + followerId;
                redisTemplate.opsForList().leftPush(feedKey, post.getId().toString());
                
                // Keep only recent posts in cache
                redisTemplate.opsForList().trim(feedKey, 0, 999); // Keep 1000 posts
            }
            
            // Publish feed update events
            kafkaTemplate.send("feed-updates", "updated", 
                new FeedUpdateEvent(batch, post.getId()));
        }
    }
    
    public List<Post> getUserFeed(Long userId, FeedRequest request) {
        String feedKey = "feed:" + userId;
        
        // Try cache first
        List<Object> cachedPostIds = redisTemplate.opsForList()
            .range(feedKey, request.getOffset(), request.getOffset() + request.getLimit() - 1);
        
        if (cachedPostIds != null && !cachedPostIds.isEmpty()) {
            List<Long> postIds = cachedPostIds.stream()
                .map(id -> Long.valueOf(id.toString()))
                .collect(Collectors.toList());
            
            return postRepository.findByIdInOrderByCreatedAtDesc(postIds);
        }
        
        // Fall back to database
        List<Long> postIds = feedRepository.getFeedPostIds(userId, request.getOffset(), request.getLimit());
        
        if (postIds.isEmpty()) {
            return Collections.emptyList();
        }
        
        List<Post> posts = postRepository.findByIdInOrderByCreatedAtDesc(postIds);
        
        // Cache the results
        redisTemplate.delete(feedKey);
        postIds.forEach(id -> redisTemplate.opsForList().rightPush(feedKey, id.toString()));
        redisTemplate.expire(feedKey, Duration.ofHours(24));
        
        return posts;
    }
    
    public List<Post> getAlgorithmicFeed(Long userId, FeedRequest request) {
        // Get base chronological feed
        List<Post> chronologicalPosts = getUserFeed(userId, request);
        
        if (chronologicalPosts.isEmpty()) {
            return chronologicalPosts;
        }
        
        // Apply ranking algorithm
        return chronologicalPosts.stream()
            .map(post -> new PostWithScore(post, calculatePostScore(post, userId)))
            .sorted(Comparator.comparing(PostWithScore::getScore).reversed())
            .map(PostWithScore::getPost)
            .collect(Collectors.toList());
    }
    
    private double calculatePostScore(Post post, Long userId) {
        double score = 0.0;
        
        // Recency score (newer posts score higher)
        long hoursSincePosted = ChronoUnit.HOURS.between(post.getCreatedAt(), Instant.now());
        score += Math.exp(-hoursSincePosted / 24.0); // Half-life of 24 hours
        
        // Engagement score
        score += Math.log(post.getLikeCount() + post.getCommentCount() + 1) * 0.5;
        
        // User affinity score (based on interaction history)
        score += calculateUserAffinity(userId, post.getUserId()) * 0.3;
        
        // Content type preferences
        score += calculateContentTypeScore(post, userId) * 0.2;
        
        return score;
    }
    
    private double calculateUserAffinity(Long viewerId, Long posterId) {
        // Cache affinity scores
        String affinityKey = "affinity:" + viewerId + ":" + posterId;
        Double affinity = (Double) redisTemplate.opsForValue().get(affinityKey);
        
        if (affinity == null) {
            // Calculate based on interaction history
            affinity = calculateAffinityFromHistory(viewerId, posterId);
            redisTemplate.opsForValue().set(affinityKey, affinity, Duration.ofHours(24));
        }
        
        return affinity;
    }
    
    private double calculateAffinityFromHistory(Long viewerId, Long posterId) {
        // Calculate based on likes, comments, shares from viewer to poster
        // This would query interaction history
        return 0.5; // Default affinity
    }
    
    private double calculateContentTypeScore(Post post, Long userId) {
        // Calculate based on user's content preferences
        // e.g., if user likes videos, boost video posts
        return 0.0; // Simplified
    }
    
    private static class PostWithScore {
        private final Post post;
        private final double score;
        
        public PostWithScore(Post post, double score) {
            this.post = post;
            this.score = score;
        }
        
        // Getters
    }
}
```

---

### 6. Design a Distributed Cache

#### Requirements Analysis
- **Functional Requirements:**
  - Store key-value pairs with TTL
  - Support get, put, delete operations
  - Handle cache misses gracefully
  - Provide cache statistics and monitoring

- **Non-Functional Requirements:**
  - High availability and fault tolerance
  - Consistent hashing for even distribution
  - Sub-millisecond latency
  - Horizontal scalability

#### Distributed Cache Implementation
```java
@Service
public class DistributedCacheService {
    
    private final ConsistentHashRing ring;
    private final Map<String, CacheNode> nodes;
    private final LoadBalancer loadBalancer;
    
    public DistributedCacheService(List<CacheNode> cacheNodes) {
        this.nodes = cacheNodes.stream()
            .collect(Collectors.toMap(CacheNode::getId, node -> node));
        
        // Build consistent hash ring
        this.ring = new ConsistentHashRing();
        cacheNodes.forEach(node -> ring.addNode(node.getId()));
        
        this.loadBalancer = new LoadBalancer(cacheNodes);
    }
    
    public String get(String key) {
        // Find responsible node
        String nodeId = ring.getNode(key);
        CacheNode node = nodes.get(nodeId);
        
        if (node == null || !node.isHealthy()) {
            // Try fallback nodes
            return getFromFallbackNodes(key, nodeId);
        }
        
        try {
            return node.get(key);
        } catch (CacheException e) {
            // Try fallback nodes
            return getFromFallbackNodes(key, nodeId);
        }
    }
    
    public void put(String key, String value, Duration ttl) {
        // Find responsible node
        String primaryNodeId = ring.getNode(key);
        CacheNode primaryNode = nodes.get(primaryNodeId);
        
        // Also store on replica nodes for redundancy
        List<String> replicaNodeIds = ring.getReplicas(key, 2);
        
        // Store on primary
        if (primaryNode != null && primaryNode.isHealthy()) {
            primaryNode.put(key, value, ttl);
        }
        
        // Store on replicas
        for (String replicaId : replicaNodeIds) {
            CacheNode replica = nodes.get(replicaId);
            if (replica != null && replica.isHealthy()) {
                replica.put(key, value, ttl);
            }
        }
    }
    
    public void delete(String key) {
        // Find all nodes that might have this key
        List<String> nodeIds = new ArrayList<>();
        nodeIds.add(ring.getNode(key));
        nodeIds.addAll(ring.getReplicas(key, 2));
        
        // Delete from all nodes
        for (String nodeId : nodeIds) {
            CacheNode node = nodes.get(nodeId);
            if (node != null) {
                try {
                    node.delete(key);
                } catch (CacheException e) {
                    // Log error but continue
                }
            }
        }
    }
    
    private String getFromFallbackNodes(String key, String failedNodeId) {
        // Get other nodes that might have the data
        List<String> fallbackNodeIds = ring.getReplicas(key, 3).stream()
            .filter(id -> !id.equals(failedNodeId))
            .collect(Collectors.toList());
        
        for (String nodeId : fallbackNodeIds) {
            CacheNode node = nodes.get(nodeId);
            if (node != null && node.isHealthy()) {
                try {
                    String value = node.get(key);
                    if (value != null) {
                        return value;
                    }
                } catch (CacheException e) {
                    // Continue to next node
                }
            }
        }
        
        return null; // Cache miss
    }
    
    public CacheStatistics getStatistics() {
        CacheStatistics stats = new CacheStatistics();
        
        for (CacheNode node : nodes.values()) {
            if (node.isHealthy()) {
                NodeStatistics nodeStats = node.getStatistics();
                stats.addNodeStatistics(nodeStats);
            }
        }
        
        return stats;
    }
    
    // Handle node failures and recoveries
    public void handleNodeFailure(String nodeId) {
        CacheNode node = nodes.get(nodeId);
        if (node != null) {
            node.markAsUnhealthy();
            
            // Trigger data redistribution if needed
            redistributeDataFromFailedNode(nodeId);
        }
    }
    
    public void handleNodeRecovery(String nodeId) {
        CacheNode node = nodes.get(nodeId);
        if (node != null) {
            node.markAsHealthy();
            
            // Sync data to recovered node
            syncDataToRecoveredNode(nodeId);
        }
    }
    
    private void redistributeDataFromFailedNode(String failedNodeId) {
        // This would implement a background process to redistribute
        // data from the failed node to other nodes
    }
    
    private void syncDataToRecoveredNode(String recoveredNodeId) {
        // Sync data from other nodes to the recovered node
    }
}

public class ConsistentHashRing {
    
    private final SortedMap<Integer, String> ring = new TreeMap<>();
    private final int virtualNodesPerNode = 100;
    
    public void addNode(String nodeId) {
        for (int i = 0; i < virtualNodesPerNode; i++) {
            int hash = hash(nodeId + ":" + i);
            ring.put(hash, nodeId);
        }
    }
    
    public void removeNode(String nodeId) {
        for (int i = 0; i < virtualNodesPerNode; i++) {
            int hash = hash(nodeId + ":" + i);
            ring.remove(hash);
        }
    }
    
    public String getNode(String key) {
        if (ring.isEmpty()) {
            return null;
        }
        
        int hash = hash(key);
        SortedMap<Integer, String> tailMap = ring.tailMap(hash);
        
        return tailMap.isEmpty() ? ring.get(ring.firstKey()) : tailMap.get(tailMap.firstKey());
    }
    
    public List<String> getReplicas(String key, int count) {
        List<String> replicas = new ArrayList<>();
        Set<String> seenNodes = new HashSet<>();
        
        int hash = hash(key);
        SortedMap<Integer, String> tailMap = ring.tailMap(hash);
        
        // Add nodes from tail map
        for (Map.Entry<Integer, String> entry : tailMap.entrySet()) {
            if (!seenNodes.contains(entry.getValue())) {
                replicas.add(entry.getValue());
                seenNodes.add(entry.getValue());
                if (replicas.size() == count) break;
            }
        }
        
        // Wrap around to beginning if needed
        if (replicas.size() < count) {
            for (Map.Entry<Integer, String> entry : ring.entrySet()) {
                if (entry.getKey() >= hash) break; // Already processed this range
                
                if (!seenNodes.contains(entry.getValue())) {
                    replicas.add(entry.getValue());
                    seenNodes.add(entry.getValue());
                    if (replicas.size() == count) break;
                }
            }
        }
        
        return replicas;
    }
    
    private int hash(String key) {
        return Math.abs(key.hashCode());
    }
}
```

## Interview Tips

### Preparation Strategy
1. **Practice Common Questions:** URL shortener, rate limiter, notification service
2. **Master Fundamentals:** CAP theorem, database sharding, caching strategies
3. **Study Real Systems:** Netflix, Twitter, Instagram, Uber architectures
4. **Focus on Trade-offs:** Understand when to choose different solutions

### During the Interview
1. **Clarify Requirements:** Ask about scale, features, constraints
2. **Start High-Level:** Present high-level architecture first
3. **Drive Discussion:** Explain your choices and trade-offs
4. **Know Limitations:** Be ready to discuss what you didn't cover

### Common Mistakes to Avoid
1. **Jumping to Details:** Start with high-level design
2. **Ignoring Scale:** Always consider scalability requirements
3. **No Trade-off Discussion:** Explain why you chose certain approaches
4. **Database-only Solutions:** Consider caching, queues, CDNs

### Follow-up Questions
- How would you handle database failures?
- What about security and rate limiting?
- How do you monitor and alert?
- What's your deployment strategy?
- How do you handle data consistency?

Remember: System design interviews test your ability to think about complex, real-world problems. Focus on communication, trade-off analysis, and demonstrating deep understanding of distributed systems concepts.
