# Instagram System Design

Instagram is a photo and video sharing social networking service with over 1 billion monthly active users. This case study examines Instagram's architecture, scaling challenges, and technical decisions that enable handling massive amounts of content and user interactions.

## Overview

Instagram allows users to share photos and videos, follow other users, and interact with content through likes, comments, and direct messages. The platform processes billions of photos and videos daily while maintaining low latency and high availability.

### Key Statistics
- **1 billion+ monthly active users**
- **500 million+ daily active users**
- **50 billion+ photos stored**
- **4+ million hours of video watched daily**
- **95 million+ photos/videos uploaded daily**
- **Peak QPS**: Millions of requests per second

## System Requirements

### Functional Requirements
- **User Management**: Registration, authentication, profiles
- **Content Upload**: Photo/video upload and processing
- **Timeline**: Personalized content feed
- **Social Interactions**: Follow, like, comment, share
- **Search**: User and content discovery
- **Direct Messaging**: Private messaging between users
- **Notifications**: Push notifications for interactions

### Non-Functional Requirements
- **High Availability**: 99.9% uptime
- **Low Latency**: <200ms for feed loading
- **Scalability**: Handle millions of concurrent users
- **Data Durability**: Never lose user content
- **Global Reach**: Fast access worldwide
- **Cost Efficiency**: Optimize storage and compute costs

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Client Applications                          │
│  Mobile Apps (iOS/Android) • Web App • Third-party APIs         │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│                   API Gateway & Load Balancer                   │
│  • Request routing • Rate limiting • Authentication • Caching   │
└─────────────────┬───────────────────────────────────────────────┘
                  │
    ┌─────────────▼─────────────┐
    │    Application Layer     │
    │  • User Service          │
    │  • Feed Service          │
    │  • Media Service         │
    │  • Search Service        │
    │  • Notification Service  │
    └─────────────┬─────────────┘
                  │
    ┌─────────────▼─────────────┐
    │     Data Layer           │
    │  • PostgreSQL (Users)    │
    │  • Cassandra (Feed)      │
    │  • Redis (Cache)         │
    │  • S3 (Media Storage)    │
    │  • Elasticsearch (Search)│
    └─────────────┬─────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│                 Infrastructure                                  │
│  • AWS Global Network • CDN • Message Queues • Monitoring       │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. User Service

#### Architecture
```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private FollowRepository followRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public User createUser(CreateUserRequest request) {
        // Validate username uniqueness
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new UsernameAlreadyExistsException(request.getUsername());
        }
        
        // Create user
        User user = new User();
        user.setUsername(request.getUsername());
        user.setEmail(request.getEmail());
        user.setFullName(request.getFullName());
        user.setCreatedAt(Instant.now());
        
        // Hash password
        user.setPasswordHash(passwordEncoder.encode(request.getPassword()));
        
        User savedUser = userRepository.save(user);
        
        // Publish user created event
        kafkaTemplate.send("user-events", "user-created", 
            new UserCreatedEvent(savedUser.getId(), savedUser.getUsername()));
        
        // Cache user profile
        cacheUserProfile(savedUser);
        
        return savedUser;
    }
    
    public UserProfile getUserProfile(Long userId) {
        String cacheKey = "user:profile:" + userId;
        
        // Try cache first
        UserProfile cached = (UserProfile) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        // Load from database
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        // Get follower/following counts
        long followerCount = followRepository.countByFolloweeId(userId);
        long followingCount = followRepository.countByFollowerId(userId);
        
        UserProfile profile = UserProfile.from(user, followerCount, followingCount);
        
        // Cache for 30 minutes
        redisTemplate.opsForValue().set(cacheKey, profile, Duration.ofMinutes(30));
        
        return profile;
    }
    
    public void followUser(Long followerId, Long followeeId) {
        // Validate users exist
        validateUserExists(followerId);
        validateUserExists(followeeId);
        
        // Check not already following
        if (followRepository.existsByFollowerIdAndFolloweeId(followerId, followeeId)) {
            throw new AlreadyFollowingException(followerId, followeeId);
        }
        
        // Create follow relationship
        Follow follow = new Follow();
        follow.setFollowerId(followerId);
        follow.setFolloweeId(followeeId);
        follow.setCreatedAt(Instant.now());
        
        followRepository.save(follow);
        
        // Update cache
        updateFollowCounts(followerId, followeeId);
        
        // Publish follow event
        kafkaTemplate.send("social-events", "user-followed", 
            new UserFollowedEvent(followerId, followeeId));
    }
    
    private void cacheUserProfile(User user) {
        String key = "user:profile:" + user.getId();
        long followerCount = followRepository.countByFolloweeId(user.getId());
        long followingCount = followRepository.countByFollowerId(user.getId());
        
        UserProfile profile = UserProfile.from(user, followerCount, followingCount);
        redisTemplate.opsForValue().set(key, profile, Duration.ofMinutes(30));
    }
    
    private void updateFollowCounts(Long followerId, Long followeeId) {
        // Invalidate follower count for follower
        redisTemplate.delete("user:profile:" + followerId);
        
        // Invalidate following count for followee
        redisTemplate.delete("user:profile:" + followeeId);
    }
}
```

#### Data Storage
- **PostgreSQL**: User profiles, follow relationships, authentication data
- **Redis**: User profile cache, follower/following counts
- **Cassandra**: User activity logs, follow timelines

### 2. Media Service

#### Photo/Video Upload Flow
```java
@Service
public class MediaService {
    
    @Autowired
    private S3Client s3Client;
    
    @Autowired
    private MediaMetadataRepository metadataRepository;
    
    @Autowired
    private ImageProcessingService imageProcessor;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public MediaUploadResult uploadMedia(MediaUploadRequest request) {
        String mediaId = generateMediaId();
        String userId = request.getUserId();
        
        try {
            // Upload original file to S3
            String originalKey = "media/" + userId + "/original/" + mediaId;
            s3Client.putObject(PutObjectRequest.builder()
                .bucket("instagram-media")
                .key(originalKey)
                .contentType(request.getContentType())
                .build(), request.getFile().getInputStream());
            
            // Store metadata
            MediaMetadata metadata = new MediaMetadata();
            metadata.setId(mediaId);
            metadata.setUserId(userId);
            metadata.setOriginalKey(originalKey);
            metadata.setContentType(request.getContentType());
            metadata.setFileSize(request.getFileSize());
            metadata.setUploadedAt(Instant.now());
            metadata.setStatus(MediaStatus.UPLOADED);
            
            metadataRepository.save(metadata);
            
            // Publish processing event
            kafkaTemplate.send("media-events", "media-uploaded", 
                new MediaUploadedEvent(mediaId, userId, request.getContentType()));
            
            return new MediaUploadResult(mediaId, MediaStatus.UPLOADED);
            
        } catch (Exception e) {
            logger.error("Failed to upload media for user {}", userId, e);
            throw new MediaUploadException("Upload failed", e);
        }
    }
}

@Service
public class ImageProcessingService {
    
    @Autowired
    private S3Client s3Client;
    
    @Autowired
    private MediaMetadataRepository metadataRepository;
    
    @KafkaListener(topics = "media-events", groupId = "image-processing")
    public void processImage(String eventJson) {
        try {
            MediaUploadedEvent event = objectMapper.readValue(eventJson, MediaUploadedEvent.class);
            
            if (!isImage(event.getContentType())) {
                return; // Skip non-image processing
            }
            
            MediaMetadata metadata = metadataRepository.findById(event.getMediaId());
            if (metadata == null) return;
            
            // Download original image
            GetObjectRequest getRequest = GetObjectRequest.builder()
                .bucket("instagram-media")
                .key(metadata.getOriginalKey())
                .build();
            
            try (InputStream originalStream = s3Client.getObject(getRequest)) {
                BufferedImage originalImage = ImageIO.read(originalStream);
                
                // Generate thumbnails
                generateThumbnails(metadata, originalImage);
                
                // Apply filters if requested
                // applyFilters(metadata, originalImage);
                
                // Update metadata status
                metadata.setStatus(MediaStatus.PROCESSED);
                metadata.setProcessedAt(Instant.now());
                metadataRepository.save(metadata);
                
                // Publish completion event
                kafkaTemplate.send("media-events", "media-processed", 
                    new MediaProcessedEvent(event.getMediaId(), event.getUserId()));
                
            }
            
        } catch (Exception e) {
            logger.error("Failed to process image {}", event.getMediaId(), e);
            // Update status to failed and retry logic
        }
    }
    
    private void generateThumbnails(MediaMetadata metadata, BufferedImage original) {
        // Generate multiple thumbnail sizes
        int[] sizes = {150, 240, 320, 480, 640};
        
        for (int size : sizes) {
            BufferedImage thumbnail = createThumbnail(original, size);
            
            // Upload thumbnail to S3
            String thumbnailKey = "media/" + metadata.getUserId() + "/thumb/" + size + "/" + metadata.getId();
            
            ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
            ImageIO.write(thumbnail, "JPEG", outputStream);
            
            s3Client.putObject(PutObjectRequest.builder()
                .bucket("instagram-media")
                .key(thumbnailKey)
                .contentType("image/jpeg")
                .build(), new ByteArrayInputStream(outputStream.toByteArray()));
            
            // Store thumbnail URL in metadata
            metadata.addThumbnailUrl(size, getCdnUrl(thumbnailKey));
        }
    }
    
    private BufferedImage createThumbnail(BufferedImage original, int maxSize) {
        int width = original.getWidth();
        int height = original.getHeight();
        
        if (width <= maxSize && height <= maxSize) {
            return original;
        }
        
        double scale = Math.min((double) maxSize / width, (double) maxSize / height);
        int newWidth = (int) (width * scale);
        int newHeight = (int) (height * scale);
        
        BufferedImage thumbnail = new BufferedImage(newWidth, newHeight, BufferedImage.TYPE_INT_RGB);
        Graphics2D g2d = thumbnail.createGraphics();
        g2d.drawImage(original.getScaledInstance(newWidth, newHeight, Image.SCALE_SMOOTH), 0, 0, null);
        g2d.dispose();
        
        return thumbnail;
    }
    
    private boolean isImage(String contentType) {
        return contentType != null && contentType.startsWith("image/");
    }
    
    private String getCdnUrl(String s3Key) {
        // Return CDN URL for the S3 object
        return "https://cdn.instagram.com/" + s3Key;
    }
}
```

#### Storage Architecture
- **S3**: Original media files and processed thumbnails
- **CloudFront CDN**: Global content delivery
- **PostgreSQL**: Media metadata and relationships
- **Redis**: Media URL cache, processing status

### 3. Feed Service

#### Timeline Generation
```java
@Service
public class FeedService {
    
    @Autowired
    private PostRepository postRepository;
    
    @Autowired
    private FollowRepository followRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private FeedCacheService feedCache;
    
    public List<Post> getUserFeed(Long userId, int page, int size) {
        String cacheKey = "feed:" + userId + ":" + page;
        
        // Try cache first
        @SuppressWarnings("unchecked")
        List<Post> cachedFeed = (List<Post>) redisTemplate.opsForValue().get(cacheKey);
        if (cachedFeed != null) {
            return cachedFeed;
        }
        
        // Generate feed from database
        List<Post> feed = generateFeed(userId, page, size);
        
        // Cache for 5 minutes
        redisTemplate.opsForValue().set(cacheKey, feed, Duration.ofMinutes(5));
        
        return feed;
    }
    
    private List<Post> generateFeed(Long userId, int page, int size) {
        // Get users that this user follows
        List<Long> followingIds = followRepository.findFollowingIdsByUserId(userId);
        followingIds.add(userId); // Include own posts
        
        // Get recent posts from followed users
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        List<Post> posts = postRepository.findByUserIdInAndStatusOrderByCreatedAtDesc(
            followingIds, PostStatus.PUBLISHED, pageable);
        
        // Apply ranking algorithm
        return rankPosts(posts, userId);
    }
    
    private List<Post> rankPosts(List<Post> posts, Long userId) {
        return posts.stream()
            .sorted((p1, p2) -> {
                // Ranking factors:
                // 1. Recency (newer posts first)
                // 2. Engagement (likes + comments)
                // 3. User affinity (posts from frequently interacted users)
                // 4. Content type preference
                
                double score1 = calculatePostScore(p1, userId);
                double score2 = calculatePostScore(p2, userId);
                
                return Double.compare(score2, score1); // Higher score first
            })
            .collect(Collectors.toList());
    }
    
    private double calculatePostScore(Post post, Long userId) {
        double score = 0.0;
        
        // Recency score (exponential decay over time)
        long hoursSincePosted = ChronoUnit.HOURS.between(post.getCreatedAt(), Instant.now());
        score += Math.exp(-hoursSincePosted / 24.0); // Half-life of 24 hours
        
        // Engagement score
        int engagement = post.getLikeCount() + post.getCommentCount() * 2;
        score += Math.log(engagement + 1) * 0.5;
        
        // User affinity score
        double affinity = calculateUserAffinity(userId, post.getUserId());
        score += affinity * 0.3;
        
        return score;
    }
    
    private double calculateUserAffinity(Long viewerId, Long posterId) {
        // Calculate based on interaction history
        // - How often the viewer likes the poster's content
        // - How often the viewer comments on the poster's content
        // - Recency of interactions
        
        String affinityKey = "affinity:" + viewerId + ":" + posterId;
        Double affinity = (Double) redisTemplate.opsForValue().get(affinityKey);
        
        if (affinity == null) {
            // Calculate from interaction history (simplified)
            affinity = 0.5; // Default affinity
            
            // Cache for 24 hours
            redisTemplate.opsForValue().set(affinityKey, affinity, Duration.ofHours(24));
        }
        
        return affinity;
    }
    
    @KafkaListener(topics = "post-events", groupId = "feed-updates")
    public void handlePostEvent(String eventJson) {
        try {
            PostEvent event = objectMapper.readValue(eventJson, PostEvent.class);
            
            if ("post-published".equals(event.getEventType())) {
                // Invalidate feeds for users who follow the poster
                invalidateFollowerFeeds(event.getUserId());
                
                // Update user affinity scores
                updateAffinityScores(event);
            }
            
        } catch (Exception e) {
            logger.error("Failed to process post event", e);
        }
    }
    
    private void invalidateFollowerFeeds(Long posterId) {
        // Get all followers
        List<Long> followerIds = followRepository.findFollowerIdsByUserId(posterId);
        
        // Invalidate feed cache for all followers
        for (Long followerId : followerIds) {
            String pattern = "feed:" + followerId + ":*";
            Set<String> keys = redisTemplate.keys(pattern);
            if (keys != null && !keys.isEmpty()) {
                redisTemplate.delete(keys);
            }
        }
    }
    
    private void updateAffinityScores(PostEvent event) {
        // This would update affinity scores based on engagement
        // Implementation would analyze likes, comments, etc.
    }
}
```

#### Feed Caching Strategy
- **Redis**: User feed cache with 5-minute TTL
- **Cassandra**: Long-term feed storage for analytics
- **Write-through cache**: Updates propagate to cache immediately

### 4. Search Service

#### Elasticsearch Implementation
```java
@Service
public class SearchService {
    
    @Autowired
    private RestHighLevelClient elasticsearchClient;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PostRepository postRepository;
    
    public SearchResult searchUsers(String query, int page, int size) {
        SearchRequest searchRequest = new SearchRequest("users");
        
        // Multi-match query for username, full name, and bio
        MultiMatchQueryBuilder queryBuilder = QueryBuilders.multiMatchQuery(query)
            .field("username", 3.0f)    // Higher weight for username
            .field("fullName", 2.0f)    // Medium weight for full name
            .field("bio", 1.0f)         // Lower weight for bio
            .type(MultiMatchQueryBuilder.Type.BEST_FIELDS)
            .fuzziness(Fuzziness.AUTO);
        
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder()
            .query(queryBuilder)
            .from(page * size)
            .size(size)
            .sort("_score", SortOrder.DESC)
            .sort("followerCount", SortOrder.DESC); // Secondary sort by popularity
        
        searchRequest.source(sourceBuilder);
        
        try {
            SearchResponse response = elasticsearchClient.search(searchRequest, RequestOptions.DEFAULT);
            
            List<UserSearchResult> users = Arrays.stream(response.getHits().getHits())
                .map(this::mapToUserResult)
                .collect(Collectors.toList());
            
            return new SearchResult(users, response.getHits().getTotalHits().value);
            
        } catch (IOException e) {
            logger.error("Failed to search users", e);
            throw new SearchException("User search failed", e);
        }
    }
    
    public SearchResult searchPosts(String query, Long userId, int page, int size) {
        SearchRequest searchRequest = new SearchRequest("posts");
        
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        
        // Text search in caption and hashtags
        MultiMatchQueryBuilder textQuery = QueryBuilders.multiMatchQuery(query)
            .field("caption", 2.0f)
            .field("hashtags", 3.0f)  // Higher weight for hashtags
            .type(MultiMatchQueryBuilder.Type.BEST_FIELDS)
            .fuzziness(Fuzziness.AUTO);
        
        boolQuery.must(textQuery);
        
        // Filter by user's followings if specified
        if (userId != null) {
            List<Long> followingIds = getFollowingIds(userId);
            followingIds.add(userId); // Include own posts
            
            TermsQueryBuilder followingFilter = QueryBuilders.termsQuery("userId", followingIds);
            boolQuery.filter(followingFilter);
        }
        
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder()
            .query(boolQuery)
            .from(page * size)
            .size(size)
            .sort("_score", SortOrder.DESC)
            .sort("createdAt", SortOrder.DESC); // Secondary sort by recency
        
        searchRequest.source(sourceBuilder);
        
        try {
            SearchResponse response = elasticsearchClient.search(searchRequest, RequestOptions.DEFAULT);
            
            List<PostSearchResult> posts = Arrays.stream(response.getHits().getHits())
                .map(this::mapToPostResult)
                .collect(Collectors.toList());
            
            return new SearchResult(posts, response.getHits().getTotalHits().value);
            
        } catch (IOException e) {
            logger.error("Failed to search posts", e);
            throw new SearchException("Post search failed", e);
        }
    }
    
    @KafkaListener(topics = "user-events", groupId = "search-indexing")
    public void indexUser(String eventJson) {
        try {
            UserEvent event = objectMapper.readValue(eventJson, UserEvent.class);
            
            if ("user-created".equals(event.getEventType()) || 
                "user-updated".equals(event.getEventType())) {
                
                User user = userRepository.findById(event.getUserId()).orElse(null);
                if (user != null) {
                    indexUser(user);
                }
            }
            
        } catch (Exception e) {
            logger.error("Failed to index user", e);
        }
    }
    
    @KafkaListener(topics = "post-events", groupId = "search-indexing")
    public void indexPost(String eventJson) {
        try {
            PostEvent event = objectMapper.readValue(eventJson, PostEvent.class);
            
            if ("post-published".equals(event.getEventType())) {
                Post post = postRepository.findById(event.getPostId()).orElse(null);
                if (post != null) {
                    indexPost(post);
                }
            }
            
        } catch (Exception e) {
            logger.error("Failed to index post", e);
        }
    }
    
    private void indexUser(User user) {
        try {
            Map<String, Object> userDoc = Map.of(
                "id", user.getId(),
                "username", user.getUsername(),
                "fullName", user.getFullName(),
                "bio", user.getBio(),
                "followerCount", getFollowerCount(user.getId()),
                "followingCount", getFollowingCount(user.getId()),
                "isVerified", user.isVerified(),
                "createdAt", user.getCreatedAt()
            );
            
            IndexRequest indexRequest = new IndexRequest("users")
                .id(user.getId().toString())
                .source(userDoc, XContentType.JSON);
            
            elasticsearchClient.index(indexRequest, RequestOptions.DEFAULT);
            
        } catch (IOException e) {
            logger.error("Failed to index user {}", user.getId(), e);
        }
    }
    
    private void indexPost(Post post) {
        try {
            List<String> hashtags = extractHashtags(post.getCaption());
            
            Map<String, Object> postDoc = Map.of(
                "id", post.getId(),
                "userId", post.getUserId(),
                "caption", post.getCaption(),
                "hashtags", hashtags,
                "likeCount", post.getLikeCount(),
                "commentCount", post.getCommentCount(),
                "mediaUrls", post.getMediaUrls(),
                "location", post.getLocation(),
                "createdAt", post.getCreatedAt()
            );
            
            IndexRequest indexRequest = new IndexRequest("posts")
                .id(post.getId().toString())
                .source(postDoc, XContentType.JSON);
            
            elasticsearchClient.index(indexRequest, RequestOptions.DEFAULT);
            
        } catch (IOException e) {
            logger.error("Failed to index post {}", post.getId(), e);
        }
    }
    
    private List<String> extractHashtags(String caption) {
        if (caption == null) return Collections.emptyList();
        
        Pattern hashtagPattern = Pattern.compile("#\\w+");
        Matcher matcher = hashtagPattern.matcher(caption);
        
        List<String> hashtags = new ArrayList<>();
        while (matcher.find()) {
            hashtags.add(matcher.group().substring(1)); // Remove # prefix
        }
        
        return hashtags;
    }
    
    private List<Long> getFollowingIds(Long userId) {
        // Implementation to get following user IDs
        return followRepository.findFollowingIdsByUserId(userId);
    }
    
    private UserSearchResult mapToUserResult(SearchHit hit) {
        Map<String, Object> source = hit.getSourceAsMap();
        return new UserSearchResult(
            Long.valueOf(hit.getId()),
            (String) source.get("username"),
            (String) source.get("fullName"),
            ((Number) source.get("followerCount")).longValue()
        );
    }
    
    private PostSearchResult mapToPostResult(SearchHit hit) {
        Map<String, Object> source = hit.getSourceAsMap();
        return new PostSearchResult(
            Long.valueOf(hit.getId()),
            (Long) source.get("userId"),
            (String) source.get("caption"),
            ((Number) source.get("likeCount")).intValue(),
            (List<String>) source.get("mediaUrls")
        );
    }
}
```

## Scaling Challenges & Solutions

### 1. Database Scaling

#### User Data
- **PostgreSQL**: Primary user data with read replicas
- **Sharding**: Users sharded by user ID ranges
- **Caching**: Heavy use of Redis for user profiles and relationships

#### Content Data
- **Cassandra**: Distributed storage for posts, comments, likes
- **Time-based partitioning**: Data partitioned by time windows
- **Eventual consistency**: Acceptable for social features

### 2. Media Storage & Delivery

#### S3 + CloudFront Strategy
- **Multi-region S3**: Content replicated across regions
- **CloudFront CDN**: Global edge locations for fast delivery
- **Progressive loading**: Images load progressively for better UX
- **WebP format**: Modern image format for smaller file sizes

### 3. Feed Generation

#### Timeline Challenges
- **Massive scale**: Billions of posts, trillions of relationships
- **Real-time updates**: New content must appear immediately
- **Personalization**: Each user sees different content

#### Solutions
- **Write-through caching**: Updates propagate to all follower caches
- **Background processing**: Heavy computation done offline
- **Machine learning**: Personalized ranking algorithms

### 4. Search & Discovery

#### Search Architecture
- **Elasticsearch clusters**: Distributed search with sharding
- **Real-time indexing**: Kafka-based indexing pipeline
- **Query optimization**: Complex scoring algorithms
- **Caching**: Popular search results cached in Redis

## Performance Optimizations

### 1. Caching Strategy

#### Multi-Level Caching
```java
@Configuration
public class CacheConfiguration {
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))  // Default TTL
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new Jackson2JsonRedisSerializer(Object.class)));
        
        // Custom TTL for different caches
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        cacheConfigurations.put("userProfiles", config.entryTtl(Duration.ofMinutes(30)));
        cacheConfigurations.put("feeds", config.entryTtl(Duration.ofMinutes(5)));
        cacheConfigurations.put("search", config.entryTtl(Duration.ofMinutes(15)));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .withInitialCacheConfigurations(cacheConfigurations)
            .build();
    }
    
    @Bean
    public CaffeineCacheManager caffeineCacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        
        // L1 cache for hot data
        cacheManager.setCacheNames(Arrays.asList("hotData", "config"));
        
        return cacheManager;
    }
}
```

### 2. Database Optimizations

#### Read/Write Splitting
```java
@Configuration
public class DataSourceConfiguration {
    
    @Bean
    @Primary
    public DataSource dataSource() {
        // Master datasource for writes
        HikariDataSource master = createDataSource("master-url");
        
        // Read replicas
        HikariDataSource replica1 = createDataSource("replica1-url");
        HikariDataSource replica2 = createDataSource("replica2-url");
        
        // Routing datasource
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("master", master);
        targetDataSources.put("replica", replica1);
        targetDataSources.put("replica2", replica2);
        
        RoutingDataSource routingDataSource = new RoutingDataSource();
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(master);
        
        return routingDataSource;
    }
    
    private HikariDataSource createDataSource(String url) {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(url);
        dataSource.setUsername("username");
        dataSource.setPassword("password");
        dataSource.setMaximumPoolSize(20);
        return dataSource;
    }
}

public class ReadWriteRoutingDataSource extends RoutingDataSource {
    
    @Override
    protected Object determineCurrentLookupKey() {
        // Determine if this is a read or write operation
        if (isReadOperation()) {
            return "replica";  // Route to read replica
        } else {
            return "master";   // Route to master
        }
    }
    
    private boolean isReadOperation() {
        // Check if current transaction is read-only
        TransactionDefinition definition = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
        return definition != null && TransactionDefinition.ISOLATION_READ_COMMITTED == definition;
        
        // Alternative: check method name or annotation
    }
}
```

### 3. API Optimizations

#### Response Compression
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new CompressionInterceptor());
    }
}

public class CompressionInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                           Object handler) throws Exception {
        
        // Enable gzip compression for responses
        if (acceptsGzip(request)) {
            response.setHeader("Content-Encoding", "gzip");
            // Spring Boot handles compression automatically when enabled
        }
        
        return true;
    }
    
    private boolean acceptsGzip(HttpServletRequest request) {
        String acceptEncoding = request.getHeader("Accept-Encoding");
        return acceptEncoding != null && acceptEncoding.contains("gzip");
    }
}
```

#### Pagination & Cursor-Based Navigation
```java
@RestController
@RequestMapping("/api/feed")
public class FeedController {
    
    @Autowired
    private FeedService feedService;
    
    @GetMapping
    public ResponseEntity<FeedResponse> getFeed(
            @RequestParam Long userId,
            @RequestParam(required = false) String cursor,
            @RequestParam(defaultValue = "20") int limit) {
        
        FeedPage page = feedService.getFeedPage(userId, cursor, limit);
        
        FeedResponse response = new FeedResponse();
        response.setPosts(page.getPosts());
        response.setNextCursor(page.getNextCursor());
        response.setHasMore(page.hasMore());
        
        return ResponseEntity.ok(response);
    }
}

@Service
public class FeedService {
    
    public FeedPage getFeedPage(Long userId, String cursor, int limit) {
        // Parse cursor (format: timestamp_postId)
        Instant startTime = Instant.now();
        Long startPostId = null;
        
        if (cursor != null) {
            String[] parts = cursor.split("_");
            startTime = Instant.ofEpochMilli(Long.parseLong(parts[0]));
            startPostId = Long.parseLong(parts[1]);
        }
        
        // Query with cursor-based pagination
        List<Post> posts = postRepository.findPostsAfter(
            getFollowingIds(userId), startTime, startPostId, limit + 1);
        
        boolean hasMore = posts.size() > limit;
        List<Post> pagePosts = hasMore ? posts.subList(0, limit) : posts;
        
        // Generate next cursor
        String nextCursor = null;
        if (hasMore && !pagePosts.isEmpty()) {
            Post lastPost = pagePosts.get(pagePosts.size() - 1);
            nextCursor = lastPost.getCreatedAt().toEpochMilli() + "_" + lastPost.getId();
        }
        
        return new FeedPage(pagePosts, nextCursor, hasMore);
    }
}
```

## Monitoring & Observability

### Key Metrics to Monitor

#### System Metrics
- **Request latency**: P50, P95, P99 response times
- **Error rates**: 4xx and 5xx error percentages
- **Throughput**: Requests per second
- **Resource utilization**: CPU, memory, disk, network

#### Business Metrics
- **User engagement**: Daily active users, session duration
- **Content metrics**: Posts uploaded, stories created
- **Interaction rates**: Likes, comments, shares per post
- **User growth**: New user registrations, retention rates

#### Infrastructure Metrics
- **Database performance**: Query latency, connection pool utilization
- **Cache hit rates**: Redis hit/miss ratios
- **Storage utilization**: S3 storage growth, transfer costs
- **CDN performance**: Cache hit rates, latency by region

### Alerting Strategy

#### Critical Alerts
- **Service downtime**: Any service with >99.9% uptime SLA
- **Data loss**: Any indication of lost user content
- **Security incidents**: Unusual access patterns or breaches
- **Performance degradation**: P95 latency >500ms

#### Warning Alerts
- **High resource utilization**: CPU >80%, memory >85%
- **Error rate spikes**: Error rate >5% for 5+ minutes
- **Queue backlog**: Message queues with growing backlog
- **Storage capacity**: Storage utilization >80%

## Lessons Learned

### Architectural Decisions
1. **Start with MVP**: Instagram began with simple LAMP stack and evolved architecture as user base grew
2. **Optimize for reads**: Heavy read workload led to sophisticated caching and replication strategies
3. **Embrace eventual consistency**: Accepted for better performance and availability
4. **Build for scale**: Designed systems to handle 10x growth from day one

### Technical Insights
1. **Photo processing at scale**: Asynchronous processing pipeline for media transformations
2. **Feed ranking complexity**: ML-driven algorithms for personalized content discovery
3. **Global distribution**: CDN and multi-region architecture for worldwide performance
4. **Data denormalization**: Pre-computed views and caches to reduce database load

### Operational Excellence
1. **Monitoring culture**: Comprehensive metrics and alerting from early stages
2. **Incident response**: Well-defined processes for handling outages and issues
3. **Cost optimization**: Efficient storage and compute resource utilization
4. **Security focus**: Multi-layered security approach for user data protection

## Future Considerations

### Emerging Technologies
- **Machine Learning**: Enhanced content moderation, recommendation algorithms
- **Edge Computing**: Faster content processing and delivery
- **Blockchain**: Content ownership and digital rights management
- **AR/VR**: Immersive content experiences

### Scalability Challenges
- **Video dominance**: Increasing video content and processing requirements
- **Real-time features**: Live streaming, real-time messaging at scale
- **Global expansion**: Supporting diverse languages and cultures
- **Privacy regulations**: GDPR, CCPA compliance at massive scale

Instagram's architecture demonstrates how to build a highly scalable, user-centric platform that can handle massive growth while maintaining excellent user experience. The combination of carefully chosen technologies, thoughtful architectural patterns, and relentless focus on performance has made Instagram one of the world's most successful social platforms.
