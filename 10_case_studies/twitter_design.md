# Twitter (X) System Design

Twitter (now X) is a real-time social media platform that enables users to post short messages called "tweets" and follow other users. This case study examines Twitter's architecture evolution, real-time messaging challenges, and scaling decisions that enable handling hundreds of millions of tweets daily with millisecond latency requirements.

## Overview

Twitter allows users to post tweets (up to 280 characters), follow other users, and engage with content through likes, retweets, and replies. The platform processes billions of tweets annually while maintaining real-time delivery and high availability.

### Key Statistics
- **300+ million monthly active users**
- **500 million+ tweets per day**
- **1.3 billion+ tweets annually**
- **Peak tweets per second**: 300,000+ during major events
- **Timeline reads**: 150,000 per second
- **Data processed daily**: 4+ PB

## System Requirements

### Functional Requirements
- **Tweeting**: Post text, images, videos, links
- **Timeline**: Personalized feed of followed users' tweets
- **Search**: Real-time tweet and user search
- **Engagement**: Like, retweet, reply, quote tweet
- **Direct Messages**: Private messaging between users
- **Notifications**: Real-time push notifications
- **Trends**: Real-time trending topics and hashtags

### Non-Functional Requirements
- **Real-time Delivery**: Tweets appear instantly in followers' timelines
- **High Availability**: 99.99% uptime
- **Low Latency**: <100ms for tweet delivery
- **Durability**: Never lose tweets once posted
- **Global Scale**: Worldwide content delivery
- **Event Handling**: Support for breaking news and major events

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Client Applications                          │
│  Web App • Mobile Apps (iOS/Android) • Third-party APIs • Embeddings │
└─────────────────┬───────────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────────┐
│                   Edge Services & CDN                              │
│  • Global CDN (Akamai/Cloudflare) • API Gateways • Rate Limiting   │
└─────────────────┬───────────────────────────────────────────────────┘
                  │
    ┌─────────────▼─────────────┐
    │    Application Layer     │
    │  • Tweet Service         │
    │  • Timeline Service      │
    │  • User Service          │
    │  • Search Service        │
    │  • Notification Service  │
    └─────────────┬─────────────┘
                  │
    ┌─────────────▼─────────────┐
    │     Data Layer           │
    │  • MySQL (Users/Tweets)  │
    │  • Redis (Cache/Timelines)│
    │  • Cassandra (Analytics) │
    │  • Manhattan (Timelines) │
    │  • Earlybird (Search)    │
    └─────────────┬─────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────────┐
│                 Real-time Processing                              │
│  • Event Streaming (Kafka) • Real-time Analytics • Push Service   │
└─────────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Tweet Service

#### Tweet Creation and Storage
```java
@Service
public class TweetService {
    
    @Autowired
    private TweetRepository tweetRepository;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private FanoutService fanoutService;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private MetricsService metricsService;
    
    public Tweet createTweet(CreateTweetRequest request) {
        Long userId = request.getUserId();
        
        // Validate user exists and is not suspended
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        if (user.isSuspended()) {
            throw new UserSuspendedException(userId);
        }
        
        // Validate tweet content
        validateTweetContent(request.getContent());
        
        // Create tweet
        Tweet tweet = new Tweet();
        tweet.setId(generateTweetId());
        tweet.setUserId(userId);
        tweet.setContent(request.getContent());
        tweet.setCreatedAt(Instant.now());
        tweet.setHashtags(extractHashtags(request.getContent()));
        tweet.setMentions(extractMentions(request.getContent()));
        
        // Handle media attachments
        if (request.getMediaIds() != null && !request.getMediaIds().isEmpty()) {
            tweet.setMediaIds(request.getMediaIds());
            tweet.setMediaType(determineMediaType(request.getMediaIds()));
        }
        
        // Save tweet
        Tweet savedTweet = tweetRepository.save(tweet);
        
        // Update user tweet count
        userRepository.incrementTweetCount(userId);
        
        // Publish tweet created event
        kafkaTemplate.send("tweet-events", "tweet-created", 
            new TweetCreatedEvent(savedTweet.getId(), userId, savedTweet.getContent()));
        
        // Start fanout process
        fanoutService.fanoutTweet(savedTweet);
        
        // Record metrics
        metricsService.recordTweetCreated();
        
        return savedTweet;
    }
    
    public Tweet getTweet(Long tweetId) {
        return tweetRepository.findById(tweetId)
            .orElseThrow(() -> new TweetNotFoundException(tweetId));
    }
    
    public void deleteTweet(Long tweetId, Long userId) {
        Tweet tweet = tweetRepository.findById(tweetId)
            .orElseThrow(() -> new TweetNotFoundException(tweetId));
        
        // Verify ownership
        if (!tweet.getUserId().equals(userId)) {
            throw new UnauthorizedException("Cannot delete tweet owned by another user");
        }
        
        // Mark as deleted (soft delete)
        tweet.setDeleted(true);
        tweet.setDeletedAt(Instant.now());
        tweetRepository.save(tweet);
        
        // Publish delete event
        kafkaTemplate.send("tweet-events", "tweet-deleted", 
            new TweetDeletedEvent(tweetId, userId));
        
        // Remove from timelines (handled by fanout service)
        fanoutService.removeTweetFromTimelines(tweet);
    }
    
    private void validateTweetContent(String content) {
        if (content == null || content.trim().isEmpty()) {
            throw new InvalidTweetException("Tweet content cannot be empty");
        }
        
        if (content.length() > 280) {
            throw new InvalidTweetException("Tweet content exceeds 280 characters");
        }
        
        // Check for spam patterns, banned words, etc.
        if (containsSpam(content)) {
            throw new SpamTweetException("Tweet contains spam content");
        }
    }
    
    private List<String> extractHashtags(String content) {
        Pattern hashtagPattern = Pattern.compile("#(\\w+)");
        Matcher matcher = hashtagPattern.matcher(content);
        
        List<String> hashtags = new ArrayList<>();
        while (matcher.find()) {
            hashtags.add(matcher.group(1));
        }
        
        return hashtags;
    }
    
    private List<Long> extractMentions(String content) {
        Pattern mentionPattern = Pattern.compile("@(\\w+)");
        Matcher matcher = mentionPattern.matcher(content);
        
        List<Long> mentions = new ArrayList<>();
        while (matcher.find()) {
            String username = matcher.group(1);
            // Resolve username to user ID (would need user service call)
            Long userId = resolveUsernameToId(username);
            if (userId != null) {
                mentions.add(userId);
            }
        }
        
        return mentions;
    }
    
    private Long generateTweetId() {
        // Twitter uses Snowflake-like ID generation
        // 41 bits timestamp, 5 bits datacenter, 5 bits worker, 12 bits sequence
        return SnowflakeIdGenerator.generateId();
    }
    
    private boolean containsSpam(String content) {
        // Implement spam detection logic
        // Could use ML models, keyword filtering, etc.
        return false; // Simplified
    }
    
    private Long resolveUsernameToId(String username) {
        // Would call user service to resolve username to ID
        return null; // Simplified
    }
}
```

### 2. Fanout Service (Timeline Generation)

#### Push vs Pull Model
Twitter uses a hybrid approach: push for small followings, pull for large followings.

```java
@Service
public class FanoutService {
    
    @Autowired
    private FollowRepository followRepository;
    
    @Autowired
    private TimelineRepository timelineRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    private static final int SMALL_FOLLOWING_THRESHOLD = 1000;
    private static final int LARGE_FOLLOWING_THRESHOLD = 10000;
    
    public void fanoutTweet(Tweet tweet) {
        Long authorId = tweet.getUserId();
        
        // Get follower count
        long followerCount = followRepository.countFollowers(authorId);
        
        if (followerCount <= SMALL_FOLLOWING_THRESHOLD) {
            // Push to all followers
            fanoutToFollowers(tweet, authorId);
        } else if (followerCount <= LARGE_FOLLOWING_THRESHOLD) {
            // Push to active followers, others pull
            fanoutToActiveFollowers(tweet, authorId);
        } else {
            // Large accounts - followers pull from cache
            cacheTweetForLargeAccount(tweet, authorId);
        }
    }
    
    private void fanoutToFollowers(Tweet tweet, Long authorId) {
        List<Long> followerIds = followRepository.findFollowerIds(authorId);
        
        // Batch followers for efficient processing
        List<List<Long>> followerBatches = Lists.partition(followerIds, 100);
        
        for (List<Long> batch : followerBatches) {
            // Add tweet to each follower's home timeline
            for (Long followerId : batch) {
                timelineRepository.addTweetToTimeline(followerId, tweet.getId());
                
                // Cache timeline update
                String timelineKey = "timeline:home:" + followerId;
                redisTemplate.opsForList().leftPush(timelineKey, tweet.getId());
                
                // Trim timeline to recent tweets only
                redisTemplate.opsForList().trim(timelineKey, 0, 799); // Keep 800 tweets
            }
            
            // Publish timeline update events
            kafkaTemplate.send("timeline-events", "timeline-updated", 
                new TimelineUpdatedEvent(batch, tweet.getId()));
        }
    }
    
    private void fanoutToActiveFollowers(Tweet tweet, Long authorId) {
        // Get only active followers (recently engaged users)
        List<Long> activeFollowerIds = followRepository.findActiveFollowerIds(authorId, 
            Instant.now().minus(Duration.ofDays(30))); // Active in last 30 days
        
        // Push to active followers
        for (Long followerId : activeFollowerIds) {
            timelineRepository.addTweetToTimeline(followerId, tweet.getId());
            
            String timelineKey = "timeline:home:" + followerId;
            redisTemplate.opsForList().leftPush(timelineKey, tweet.getId());
            redisTemplate.opsForList().trim(timelineKey, 0, 799);
        }
        
        // Cache tweet for inactive followers to pull later
        String authorTweetsKey = "tweets:user:" + authorId;
        redisTemplate.opsForList().leftPush(authorTweetsKey, tweet.getId());
        redisTemplate.opsForList().trim(authorTweetsKey, 0, 199); // Keep 200 recent tweets
    }
    
    private void cacheTweetForLargeAccount(Tweet tweet, Long authorId) {
        // Only cache tweet for pull-based access
        String authorTweetsKey = "tweets:user:" + authorId;
        redisTemplate.opsForList().leftPush(authorTweetsKey, tweet.getId());
        redisTemplate.opsForList().trim(authorTweetsKey, 0, 199);
        
        // Mark that this is a large account requiring pull-based timeline
        String largeAccountKey = "large_accounts";
        redisTemplate.opsForSet().add(largeAccountKey, authorId.toString());
    }
    
    public List<Long> getHomeTimeline(Long userId, Long cursor, int count) {
        String timelineKey = "timeline:home:" + userId;
        
        // Try to get from cache first
        List<Object> cachedTweets = redisTemplate.opsForList()
            .range(timelineKey, 0, count - 1);
        
        if (cachedTweets != null && !cachedTweets.isEmpty()) {
            return cachedTweets.stream()
                .map(obj -> Long.valueOf(obj.toString()))
                .collect(Collectors.toList());
        }
        
        // Fall back to database
        return timelineRepository.getTimelineTweets(userId, cursor, count);
    }
    
    public void removeTweetFromTimelines(Tweet tweet) {
        Long tweetId = tweet.getId();
        Long authorId = tweet.getUserId();
        
        // Remove from cached timelines
        String timelinePattern = "timeline:home:*";
        Set<String> timelineKeys = redisTemplate.keys(timelinePattern);
        
        if (timelineKeys != null) {
            for (String timelineKey : timelineKeys) {
                redisTemplate.opsForList().remove(timelineKey, 0, tweetId);
            }
        }
        
        // Remove from database timelines
        timelineRepository.removeTweetFromAllTimelines(tweetId);
        
        // Remove from author's tweet cache
        String authorTweetsKey = "tweets:user:" + authorId;
        redisTemplate.opsForList().remove(authorTweetsKey, 0, tweetId);
    }
    
    public void rebuildTimeline(Long userId) {
        // Rebuild timeline from scratch (expensive operation)
        List<Long> followingIds = followRepository.findFollowingIdsByUserId(userId);
        
        // Get recent tweets from followed users
        List<Long> recentTweetIds = timelineRepository.getRecentTweetsFromUsers(
            followingIds, Instant.now().minus(Duration.ofDays(7))); // Last 7 days
        
        // Update cached timeline
        String timelineKey = "timeline:home:" + userId;
        redisTemplate.delete(timelineKey);
        
        // Add tweets in reverse chronological order
        Collections.reverse(recentTweetIds);
        for (Long tweetId : recentTweetIds) {
            redisTemplate.opsForList().rightPush(timelineKey, tweetId);
        }
        
        // Trim to 800 tweets
        redisTemplate.opsForList().trim(timelineKey, 0, 799);
    }
}
```

### 3. Search Service (Earlybird)

#### Real-time Search Architecture
```java
@Service
public class SearchService {
    
    @Autowired
    private EarlybirdClient earlybirdClient;
    
    @Autowired
    private TweetRepository tweetRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public SearchResults searchTweets(String query, Long userId, SearchFilters filters) {
        // Check cache for popular queries
        String cacheKey = generateCacheKey(query, filters);
        @SuppressWarnings("unchecked")
        SearchResults cachedResults = (SearchResults) redisTemplate.opsForValue().get(cacheKey);
        
        if (cachedResults != null) {
            return cachedResults;
        }
        
        // Perform real-time search using Earlybird
        SearchRequest searchRequest = new SearchRequest(query);
        searchRequest.setFilters(filters);
        
        if (userId != null) {
            // Include search history and personalization
            searchRequest.setUserId(userId);
            searchRequest.setPersonalizationEnabled(true);
        }
        
        SearchResults results = earlybirdClient.search(searchRequest);
        
        // Cache results for popular queries (TTL based on result freshness)
        if (isPopularQuery(query)) {
            Duration ttl = determineCacheTTL(results);
            redisTemplate.opsForValue().set(cacheKey, results, ttl);
        }
        
        return results;
    }
    
    public List<String> getTrendingTopics() {
        // Get trending topics from cache or compute
        String trendingKey = "trending:topics";
        @SuppressWarnings("unchecked")
        List<String> trending = (List<String>) redisTemplate.opsForValue().get(trendingKey);
        
        if (trending == null) {
            // Compute trending topics
            trending = computeTrendingTopics();
            redisTemplate.opsForValue().set(trendingKey, trending, Duration.ofMinutes(5));
        }
        
        return trending;
    }
    
    public List<String> getSuggestedSearches(String partialQuery) {
        String suggestionsKey = "search:suggestions:" + partialQuery.toLowerCase();
        
        @SuppressWarnings("unchecked")
        List<String> suggestions = (List<String>) redisTemplate.opsForValue().get(suggestionsKey);
        
        if (suggestions == null) {
            suggestions = computeSearchSuggestions(partialQuery);
            redisTemplate.opsForValue().set(suggestionsKey, suggestions, Duration.ofHours(1));
        }
        
        return suggestions;
    }
    
    private String generateCacheKey(String query, SearchFilters filters) {
        return "search:" + query.hashCode() + ":" + filters.hashCode();
    }
    
    private boolean isPopularQuery(String query) {
        // Check if query has been searched frequently recently
        String popularKey = "popular:queries";
        Double score = redisTemplate.opsForZSet().score(popularKey, query);
        return score != null && score > 100; // Searched >100 times recently
    }
    
    private Duration determineCacheTTL(SearchResults results) {
        // Cache popular results longer, fresh results shorter
        Instant newestTweet = results.getTweets().stream()
            .map(Tweet::getCreatedAt)
            .max(Instant::compareTo)
            .orElse(Instant.now());
        
        long minutesSinceNewest = ChronoUnit.MINUTES.between(newestTweet, Instant.now());
        
        if (minutesSinceNewest < 5) {
            return Duration.ofSeconds(30); // Very fresh results
        } else if (minutesSinceNewest < 60) {
            return Duration.ofMinutes(2);  // Recent results
        } else {
            return Duration.ofMinutes(10); // Older results
        }
    }
    
    private List<String> computeTrendingTopics() {
        // Aggregate hashtag frequencies from recent tweets
        // This would use a streaming computation engine like Heron or Flink
        return Arrays.asList("#BreakingNews", "#Technology", "#Sports", "#Politics");
    }
    
    private List<String> computeSearchSuggestions(String partialQuery) {
        // Find popular searches that start with the partial query
        String suggestionsKey = "search:suggestions";
        return redisTemplate.opsForZSet()
            .rangeByLex(suggestionsKey, 
                Range.range().gte(partialQuery).lt(partialQuery + Character.MAX_VALUE),
                RedisZSetCommands.Limit.limit().count(10))
            .stream()
            .map(Object::toString)
            .collect(Collectors.toList());
    }
    
    // Record search query for analytics and suggestions
    public void recordSearchQuery(String query, Long userId) {
        // Add to popular queries
        String popularKey = "popular:queries";
        redisTemplate.opsForZSet().incrementScore(popularKey, query, 1);
        
        // Add to suggestions
        String suggestionsKey = "search:suggestions";
        redisTemplate.opsForZSet().add(suggestionsKey, query, 0);
        
        // Clean up old entries periodically
        redisTemplate.opsForZSet().removeRange(popularKey, 0, -1001); // Keep top 1000
        redisTemplate.opsForZSet().removeRange(suggestionsKey, 0, -10001); // Keep top 10000
    }
}
```

### 4. Notification Service

#### Real-time Push Notifications
```java
@Service
public class NotificationService {
    
    @Autowired
    private NotificationRepository notificationRepository;
    
    @Autowired
    private PushNotificationService pushService;
    
    @Autowired
    private EmailService emailService;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @KafkaListener(topics = "tweet-events", groupId = "notifications")
    public void handleTweetEvents(String eventJson) {
        try {
            TweetEvent event = objectMapper.readValue(eventJson, TweetEvent.class);
            
            switch (event.getEventType()) {
                case "tweet-created":
                    handleTweetCreated((TweetCreatedEvent) event);
                    break;
                case "tweet-liked":
                    handleTweetLiked((TweetLikedEvent) event);
                    break;
                case "user-mentioned":
                    handleUserMentioned((UserMentionedEvent) event);
                    break;
            }
            
        } catch (Exception e) {
            logger.error("Failed to process tweet event for notifications", e);
        }
    }
    
    private void handleTweetCreated(TweetCreatedEvent event) {
        Long authorId = event.getUserId();
        
        // Get users who follow the author and want notifications
        List<Long> followerIds = followRepository.findFollowerIdsWithNotificationsEnabled(authorId);
        
        // Create notifications for followers
        for (Long followerId : followerIds) {
            Notification notification = new Notification();
            notification.setId(generateNotificationId());
            notification.setUserId(followerId);
            notification.setType(NotificationType.NEW_TWEET);
            notification.setActorId(authorId);
            notification.setTweetId(event.getTweetId());
            notification.setCreatedAt(Instant.now());
            notification.setRead(false);
            
            notificationRepository.save(notification);
            
            // Send push notification
            pushService.sendPushNotification(followerId, 
                "New tweet from @" + event.getUsername(), 
                event.getContentPreview());
        }
    }
    
    private void handleTweetLiked(TweetLikedEvent event) {
        // Notify tweet author about the like
        Long tweetAuthorId = event.getTweetAuthorId();
        Long likerId = event.getLikerId();
        
        // Check if author wants like notifications
        if (userPreferencesService.wantsLikeNotifications(tweetAuthorId)) {
            Notification notification = new Notification();
            notification.setId(generateNotificationId());
            notification.setUserId(tweetAuthorId);
            notification.setType(NotificationType.TWEET_LIKED);
            notification.setActorId(likerId);
            notification.setTweetId(event.getTweetId());
            notification.setCreatedAt(Instant.now());
            
            notificationRepository.save(notification);
            
            pushService.sendPushNotification(tweetAuthorId, 
                "@" + event.getLikerUsername() + " liked your tweet", 
                event.getTweetPreview());
        }
    }
    
    private void handleUserMentioned(UserMentionedEvent event) {
        Long mentionedUserId = event.getMentionedUserId();
        Long mentionerId = event.getMentionerId();
        
        Notification notification = new Notification();
        notification.setId(generateNotificationId());
        notification.setUserId(mentionedUserId);
        notification.setType(NotificationType.USER_MENTIONED);
        notification.setActorId(mentionerId);
        notification.setTweetId(event.getTweetId());
        notification.setCreatedAt(Instant.now());
        
        notificationRepository.save(notification);
        
        pushService.sendPushNotification(mentionedUserId, 
            "@" + event.getMentionerUsername() + " mentioned you", 
            event.getTweetPreview());
    }
    
    public List<Notification> getUserNotifications(Long userId, Long cursor, int limit) {
        return notificationRepository.findByUserIdOrderByCreatedAtDesc(userId, cursor, limit);
    }
    
    public void markNotificationsAsRead(Long userId, List<Long> notificationIds) {
        notificationRepository.markAsRead(userId, notificationIds);
    }
    
    public NotificationSettings getUserNotificationSettings(Long userId) {
        return notificationSettingsRepository.findByUserId(userId)
            .orElse(new NotificationSettings(userId)); // Default settings
    }
    
    public void updateNotificationSettings(Long userId, NotificationSettings settings) {
        settings.setUserId(userId);
        notificationSettingsRepository.save(settings);
    }
    
    // Email digest for users who prefer email notifications
    @Scheduled(cron = "0 0 * * * *") // Run hourly
    public void sendEmailDigests() {
        // Find users who want email digests
        List<Long> usersToNotify = userRepository.findUsersWithEmailNotificationsEnabled();
        
        for (Long userId : usersToNotify) {
            List<Notification> unreadNotifications = 
                notificationRepository.findUnreadByUserId(userId, Instant.now().minus(Duration.ofHours(24)));
            
            if (!unreadNotifications.isEmpty()) {
                emailService.sendNotificationDigest(userId, unreadNotifications);
                
                // Mark notifications as emailed
                List<Long> notificationIds = unreadNotifications.stream()
                    .map(Notification::getId)
                    .collect(Collectors.toList());
                
                notificationRepository.markAsEmailed(userId, notificationIds);
            }
        }
    }
    
    private Long generateNotificationId() {
        return SnowflakeIdGenerator.generateId();
    }
}
```

## Scaling Challenges & Solutions

### 1. Timeline Generation at Scale

#### Fanout Challenges
- **Massive scale**: Millions of tweets per day, billions of follower relationships
- **Real-time requirements**: Tweets must appear instantly
- **Large followings**: Celebrities with millions of followers

#### Solutions Implemented
- **Hybrid fanout model**: Push for small followings, pull for large accounts
- **Timeline caching**: Redis for fast timeline access
- **Background processing**: Async fanout for large accounts
- **Timeline reconstruction**: Periodic rebuild for consistency

### 2. Search at Real-time Scale

#### Earlybird Search Engine
```java
@Configuration
public class EarlybirdConfiguration {
    
    @Bean
    public EarlybirdClient earlybirdClient() {
        EarlybirdClientConfig config = new EarlybirdClientConfig();
        config.setHosts(Arrays.asList("earlybird-1:2898", "earlybird-2:2898", "earlybird-3:2898"));
        config.setTimeout(Duration.ofMillis(100));
        config.setMaxConnections(50);
        config.setRetryCount(2);
        
        return new EarlybirdClient(config);
    }
}

@Service
public class EarlybirdIndexingService {
    
    @Autowired
    private EarlybirdClient earlybirdClient;
    
    @KafkaListener(topics = "tweet-events", groupId = "earlybird-indexing")
    public void indexTweet(String eventJson) {
        try {
            TweetCreatedEvent event = objectMapper.readValue(eventJson, TweetCreatedEvent.class);
            
            // Create search document
            SearchDocument doc = new SearchDocument();
            doc.setId(event.getTweetId());
            doc.setUserId(event.getUserId());
            doc.setContent(event.getContent());
            doc.setHashtags(event.getHashtags());
            doc.setMentions(event.getMentions());
            doc.setCreatedAt(event.getTimestamp());
            
            // Add metadata
            doc.setLanguage(detectLanguage(event.getContent()));
            doc.setLocation(event.getLocation());
            doc.setReplyToId(event.getReplyToId());
            
            // Index in Earlybird
            earlybirdClient.index(doc);
            
            logger.debug("Indexed tweet {} in Earlybird", event.getTweetId());
            
        } catch (Exception e) {
            logger.error("Failed to index tweet in Earlybird", e);
            // Could send to dead letter queue for retry
        }
    }
    
    private String detectLanguage(String content) {
        // Use language detection library
        return "en"; // Default to English
    }
}
```

### 3. Real-time Event Processing

#### Event Streaming with Kafka
```java
@Configuration
public class KafkaConfiguration {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-1:9092,kafka-2:9092,kafka-3:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        configProps.put(ProducerConfig.ACKS_CONFIG, "all");
        configProps.put(ProducerConfig.RETRIES_CONFIG, 3);
        configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-1:9092,kafka-2:9092,kafka-3:9092");
        configProps.put(ConsumerConfig.GROUP_ID_CONFIG, "twitter-services");
        configProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        configProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        configProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        
        return new DefaultKafkaConsumerFactory<>(configProps);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3); // 3 threads per listener
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}
```

### 4. Database Scaling

#### MySQL Sharding Strategy
```java
@Configuration
public class DatabaseConfiguration {
    
    @Bean
    @Primary
    public DataSource dataSource() {
        // Sharded data source
        ShardedDataSource shardedDataSource = new ShardedDataSource();
        
        // User shards (by user ID)
        Map<String, DataSource> userShards = createUserShards();
        shardedDataSource.addShardGroup("users", userShards, new UserShardStrategy());
        
        // Tweet shards (by tweet ID)
        Map<String, DataSource> tweetShards = createTweetShards();
        shardedDataSource.addShardGroup("tweets", tweetShards, new TweetShardStrategy());
        
        return shardedDataSource;
    }
    
    private Map<String, DataSource> createUserShards() {
        Map<String, DataSource> shards = new HashMap<>();
        
        // Create 256 user shards (for ~4 billion users)
        for (int i = 0; i < 256; i++) {
            HikariDataSource shardDataSource = new HikariDataSource();
            shardDataSource.setJdbcUrl("jdbc:mysql://user-shard-" + i + ":3306/twitter_users");
            shardDataSource.setUsername("twitter");
            shardDataSource.setPassword("password");
            shardDataSource.setMaximumPoolSize(10);
            
            shards.put("user-shard-" + i, shardDataSource);
        }
        
        return shards;
    }
    
    private Map<String, DataSource> createTweetShards() {
        Map<String, DataSource> shards = new HashMap<>();
        
        // Create 512 tweet shards
        for (int i = 0; i < 512; i++) {
            HikariDataSource shardDataSource = new HikariDataSource();
            shardDataSource.setJdbcUrl("jdbc:mysql://tweet-shard-" + i + ":3306/twitter_tweets");
            shardDataSource.setUsername("twitter");
            shardDataSource.setPassword("password");
            shardDataSource.setMaximumPoolSize(15);
            
            shards.put("tweet-shard-" + i, shardDataSource);
        }
        
        return shards;
    }
}

public class UserShardStrategy implements ShardStrategy {
    
    @Override
    public String getShardKey(Object entity) {
        if (entity instanceof User) {
            Long userId = ((User) entity).getId();
            return "user-shard-" + (userId % 256);
        }
        throw new IllegalArgumentException("Unsupported entity type: " + entity.getClass());
    }
}

public class TweetShardStrategy implements ShardStrategy {
    
    @Override
    public String getShardKey(Object entity) {
        if (entity instanceof Tweet) {
            Long tweetId = ((Tweet) entity).getId();
            return "tweet-shard-" + (tweetId % 512);
        }
        throw new IllegalArgumentException("Unsupported entity type: " + entity.getClass());
    }
}
```

## Performance Optimizations

### 1. Timeline Caching Strategy

#### Multi-Level Timeline Cache
```java
@Service
public class TimelineCacheService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private CaffeineCacheManager caffeineCacheManager;
    
    // L1 cache for hot timelines (in-memory)
    private final Cache<String, List<Long>> l1TimelineCache = 
        Caffeine.newBuilder()
            .maximumSize(10000) // 10k timelines
            .expireAfterWrite(Duration.ofMinutes(2))
            .build();
    
    public List<Long> getTimeline(Long userId, int page, int size) {
        String cacheKey = "timeline:" + userId + ":" + page + ":" + size;
        
        // Try L1 cache first
        List<Long> l1Result = l1TimelineCache.getIfPresent(cacheKey);
        if (l1Result != null) {
            return l1Result;
        }
        
        // Try L2 cache (Redis)
        String redisKey = "timeline:" + userId;
        List<Object> redisResult = redisTemplate.opsForList().range(redisKey, 
            page * size, (page + 1) * size - 1);
        
        if (redisResult != null && !redisResult.isEmpty()) {
            List<Long> timeline = redisResult.stream()
                .map(obj -> Long.valueOf(obj.toString()))
                .collect(Collectors.toList());
            
            // Update L1 cache
            l1TimelineCache.put(cacheKey, timeline);
            return timeline;
        }
        
        return null; // Cache miss
    }
    
    public void updateTimeline(Long userId, Long tweetId) {
        String redisKey = "timeline:" + userId;
        
        // Add to Redis timeline
        redisTemplate.opsForList().leftPush(redisKey, tweetId);
        
        // Trim to keep only recent tweets
        redisTemplate.opsForList().trim(redisKey, 0, 999); // Keep 1000 tweets
        
        // Invalidate L1 cache for this user
        l1TimelineCache.invalidateAll(key -> key.startsWith("timeline:" + userId + ":"));
        
        // Set expiration on Redis key
        redisTemplate.expire(redisKey, Duration.ofHours(24));
    }
    
    public void invalidateTimeline(Long userId) {
        // Invalidate both L1 and L2 cache
        l1TimelineCache.invalidateAll(key -> key.startsWith("timeline:" + userId + ":"));
        
        String redisKey = "timeline:" + userId;
        redisTemplate.delete(redisKey);
    }
    
    // Warm up cache for active users
    @Scheduled(fixedRate = 300000) // Every 5 minutes
    public void warmCacheForActiveUsers() {
        List<Long> activeUserIds = getActiveUserIds();
        
        for (Long userId : activeUserIds) {
            // Pre-load timeline into cache
            List<Long> timeline = getTimelineFromDatabase(userId, 0, 50);
            
            String redisKey = "timeline:" + userId;
            redisTemplate.delete(redisKey); // Clear existing
            
            // Add tweets to Redis
            for (Long tweetId : timeline) {
                redisTemplate.opsForList().rightPush(redisKey, tweetId);
            }
            
            redisTemplate.expire(redisKey, Duration.ofHours(24));
        }
    }
    
    private List<Long> getActiveUserIds() {
        // Get users who were active in the last hour
        return userActivityRepository.findActiveUserIds(Instant.now().minus(Duration.ofHours(1)));
    }
    
    private List<Long> getTimelineFromDatabase(Long userId, int page, int size) {
        // Query database for timeline
        return timelineRepository.getTimelineTweets(userId, null, size);
    }
}
```

### 2. Rate Limiting & Abuse Prevention

#### Multi-Layer Rate Limiting
```java
@Service
public class RateLimitService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private final Map<String, RateLimitRule> rateLimitRules = new HashMap<>();
    
    @PostConstruct
    public void initializeRules() {
        // Tweet posting limits
        rateLimitRules.put("tweet:create", new RateLimitRule(300, Duration.ofHours(3))); // 300 tweets per 3 hours
        rateLimitRules.put("tweet:create:daily", new RateLimitRule(2400, Duration.ofDays(1))); // 2400 tweets per day
        
        // API call limits
        rateLimitRules.put("api:read", new RateLimitRule(60000, Duration.ofHours(1))); // 60k reads per hour
        rateLimitRules.put("api:write", new RateLimitRule(300, Duration.ofHours(1))); // 300 writes per hour
        
        // Search limits
        rateLimitRules.put("search:query", new RateLimitRule(1800, Duration.ofHours(1))); // 1800 searches per hour
    }
    
    public boolean allowRequest(String key, String operation) {
        RateLimitRule rule = rateLimitRules.get(operation);
        if (rule == null) {
            return true; // No limit for this operation
        }
        
        String redisKey = "ratelimit:" + operation + ":" + key;
        
        // Use Redis sorted set to track requests
        long currentTime = System.currentTimeMillis();
        long windowStart = currentTime - rule.getWindow().toMillis();
        
        // Remove old requests outside the window
        redisTemplate.opsForZSet().removeRangeByScore(redisKey, 0, windowStart);
        
        // Count requests in current window
        Long requestCount = redisTemplate.opsForZSet().zCard(redisKey);
        
        if (requestCount >= rule.getLimit()) {
            return false; // Rate limit exceeded
        }
        
        // Add current request
        redisTemplate.opsForZSet().add(redisKey, UUID.randomUUID().toString(), currentTime);
        
        // Set expiration on the key
        redisTemplate.expire(redisKey, rule.getWindow().multipliedBy(2));
        
        return true;
    }
    
    public RateLimitStatus getRateLimitStatus(String key, String operation) {
        RateLimitRule rule = rateLimitRules.get(operation);
        if (rule == null) {
            return new RateLimitStatus(Integer.MAX_VALUE, Integer.MAX_VALUE, Duration.ZERO);
        }
        
        String redisKey = "ratelimit:" + operation + ":" + key;
        Long requestCount = redisTemplate.opsForZSet().zCard(redisKey);
        
        long remaining = rule.getLimit() - (requestCount != null ? requestCount : 0);
        long resetTime = calculateResetTime(redisKey, rule);
        
        return new RateLimitStatus(rule.getLimit(), remaining, Duration.ofMillis(resetTime));
    }
    
    private long calculateResetTime(String redisKey, RateLimitRule rule) {
        // Find the oldest request in the current window
        Set<Object> oldestRequests = redisTemplate.opsForZSet().range(redisKey, 0, 0);
        
        if (oldestRequests != null && !oldestRequests.isEmpty()) {
            // Get score (timestamp) of oldest request
            Double oldestTimestamp = redisTemplate.opsForZSet().score(redisKey, oldestRequests.iterator().next());
            if (oldestTimestamp != null) {
                long resetTime = oldestTimestamp.longValue() + rule.getWindow().toMillis();
                return Math.max(0, resetTime - System.currentTimeMillis());
            }
        }
        
        return rule.getWindow().toMillis();
    }
    
    public static class RateLimitRule {
        private final int limit;
        private final Duration window;
        
        public RateLimitRule(int limit, Duration window) {
            this.limit = limit;
            this.window = window;
        }
        
        public int getLimit() { return limit; }
        public Duration getWindow() { return window; }
    }
    
    public static class RateLimitStatus {
        private final long limit;
        private final long remaining;
        private final Duration resetTime;
        
        public RateLimitStatus(long limit, long remaining, Duration resetTime) {
            this.limit = limit;
            this.remaining = remaining;
            this.resetTime = resetTime;
        }
        
        public long getLimit() { return limit; }
        public long getRemaining() { return remaining; }
        public Duration getResetTime() { return resetTime; }
    }
}
```

## Monitoring & Observability

### Key Metrics to Track

#### System Metrics
- **Tweets per second**: Peak TPS during major events
- **Timeline latency**: P95 timeline loading time
- **Fanout lag**: Delay between tweet creation and timeline appearance
- **Search latency**: P95 search response time

#### Business Metrics
- **User engagement**: Tweets viewed, liked, retweeted per user
- **Content reach**: Average impressions per tweet
- **Real-time events**: Breaking news tweet processing
- **API usage**: Requests per minute by endpoint

#### Infrastructure Metrics
- **Database performance**: Query latency, connection utilization
- **Cache hit rates**: Redis hit/miss ratios by data type
- **Message queue lag**: Kafka consumer lag by topic
- **CDN performance**: Global content delivery metrics

### Alerting Strategy

#### Critical Alerts
- **Service downtime**: Any service with >99.9% uptime SLA
- **Timeline delivery lag**: Tweets not appearing within 5 seconds
- **Search unavailability**: Search service down or degraded
- **Data loss**: Any indication of lost tweets or user data

#### Performance Alerts
- **High latency**: P95 latency > 500ms for critical paths
- **Low throughput**: TPS drops below 80% of normal
- **Resource saturation**: CPU > 85%, memory > 90%
- **Queue backup**: Message queues with growing backlog

## Lessons Learned

### Architectural Evolution
1. **From monolithic to SOA**: Gradual decomposition into services
2. **Hybrid fanout model**: Push for small accounts, pull for large accounts
3. **Real-time search**: Custom search engine for real-time indexing
4. **Global infrastructure**: Multi-region deployment for worldwide reach

### Technical Innovations
1. **Snowflake IDs**: Globally unique, time-ordered ID generation
2. **Earlybird search**: Real-time full-text search at scale
3. **Twemcache**: Custom caching layer for timeline data
4. **Gizzard**: Sharding framework for database scaling

### Operational Excellence
1. **Chaos engineering**: Regular failure injection testing
2. **Capacity planning**: Proactive scaling based on growth predictions
3. **Incident response**: Well-documented runbooks and automated remediation
4. **Security focus**: Multi-layered security with encryption and access controls

## Future Challenges

### Emerging Requirements
- **Video dominance**: Increased video content and processing
- **Live streaming**: Real-time video streaming at scale
- **Longer content**: Extended character limits for tweets
- **Monetization**: Advertising and premium features

### Technical Challenges
- **Real-time ML**: Machine learning inferences for content moderation
- **Edge computing**: Content processing closer to users
- **Privacy regulations**: GDPR, CCPA compliance at global scale
- **Content moderation**: AI-powered detection of harmful content

Twitter's architecture demonstrates how to build a real-time, globally distributed system that can handle massive scale while maintaining millisecond-level performance. The combination of innovative algorithms, carefully chosen technologies, and relentless focus on performance has made Twitter the world's primary real-time communication platform.
