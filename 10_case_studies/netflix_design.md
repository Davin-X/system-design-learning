# Netflix System Design

Netflix is a subscription-based streaming service that offers a vast library of movies, TV shows, and original content to over 270 million subscribers worldwide. This case study examines Netflix's architecture evolution, content delivery challenges, and technical decisions that enable seamless streaming at global scale with personalized recommendations.

## Overview

Netflix allows subscribers to stream content on-demand across multiple devices, with features like personalized recommendations, offline viewing, and multiple concurrent streams. The platform processes petabytes of data daily while maintaining high-quality streaming and sub-second content startup times.

### Key Statistics
- **270+ million subscribers worldwide**
- **15+ billion hours of content watched in 2023**
- **Over 17,000 titles in library**
- **1+ billion hours of content streamed daily**
- **Peak concurrent streams**: 100+ million
- **Data processed daily**: 500+ PB
- **Global infrastructure**: 1,000+ cities in 190+ countries

## System Requirements

### Functional Requirements
- **Content Streaming**: High-quality video playback on all devices
- **Personalization**: AI-powered content recommendations
- **User Management**: Multi-profile accounts, parental controls
- **Device Management**: Concurrent streams, offline downloads
- **Content Discovery**: Search, browse by categories/genres
- **Playback Features**: Variable quality, subtitles, audio descriptions
- **Social Features**: My List, ratings, reviews

### Non-Functional Requirements
- **Global Scale**: Fast content delivery worldwide
- **High Availability**: 99.99% uptime with minimal outages
- **Low Latency**: Sub-second content startup times
- **Adaptive Quality**: Automatic quality adjustment based on network
- **Cost Efficiency**: Optimize storage and bandwidth costs
- **Security**: Content protection and user privacy
- **Real-time Analytics**: Instant viewer metrics and A/B testing

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Client Applications                            │
│  Smart TVs • Mobile Apps • Web Players • Gaming Consoles • Roku     │
└─────────────────┬───────────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────────┐
│                   Edge Services & CDN                              │
│  • Open Connect CDN • API Gateways • Load Balancers • DNS          │
└─────────────────┬───────────────────────────────────────────────────┘
                  │
    ┌─────────────▼─────────────┐
    │    Application Layer     │
    │  • Playback Service      │
    │  • Recommendation Engine │
    │  • User Service          │
    │  • Content Service       │
    │  • Search Service        │
    └─────────────┬─────────────┘
                  │
    ┌─────────────▼─────────────┐
    │     Data Layer           │
    │  • EVCache (Caching)     │
    │  • Cassandra (Metadata)  │
    │  • S3 (Video Storage)    │
    │  • Elasticsearch (Search)│
    │  • Redshift (Analytics)  │
    └─────────────┬─────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────────┐
│                 Video Processing & AI                              │
│  • Video Encoding • ML Training • Real-time Analytics • A/B Tests │
└─────────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Playback Service

#### Adaptive Bitrate Streaming
```java
@Service
public class PlaybackService {
    
    @Autowired
    private ContentRepository contentRepository;
    
    @Autowired
    private UserSessionRepository sessionRepository;
    
    @Autowired
    private BandwidthEstimator bandwidthEstimator;
    
    @Autowired
    private CDNService cdnService;
    
    public PlaybackManifest getPlaybackManifest(String contentId, String userId, String deviceType) {
        // Validate content access
        Content content = contentRepository.findById(contentId)
            .orElseThrow(() -> new ContentNotFoundException(contentId));
        
        if (!hasAccess(userId, content)) {
            throw new AccessDeniedException("User does not have access to this content");
        }
        
        // Get user session info
        UserSession session = sessionRepository.findByUserId(userId);
        
        // Determine available bitrates based on device capabilities
        List<BitrateOption> availableBitrates = getAvailableBitrates(content, deviceType);
        
        // Estimate current bandwidth
        long estimatedBandwidth = bandwidthEstimator.estimateBandwidth(session);
        
        // Select optimal initial bitrate
        BitrateOption initialBitrate = selectInitialBitrate(availableBitrates, estimatedBandwidth);
        
        // Generate manifest with CDN URLs
        PlaybackManifest manifest = new PlaybackManifest();
        manifest.setContentId(contentId);
        manifest.setDuration(content.getDuration());
        manifest.setAvailableBitrates(availableBitrates);
        manifest.setInitialBitrate(initialBitrate);
        manifest.setSegments(generateSegmentUrls(content, cdnService));
        manifest.setSubtitles(getSubtitleUrls(content, session.getPreferredLanguage()));
        manifest.setAudioTracks(getAudioTrackUrls(content, session.getPreferredLanguage()));
        
        // Add DRM protection info
        if (content.isProtected()) {
            manifest.setDrmInfo(generateDrmInfo(content, userId));
        }
        
        return manifest;
    }
    
    public void recordPlaybackEvent(PlaybackEvent event) {
        // Store playback metrics for analytics
        playbackMetricsRepository.save(event);
        
        // Update user preferences based on viewing behavior
        updateUserPreferences(event);
        
        // Trigger recommendations refresh if needed
        if (event.getEventType() == PlaybackEventType.COMPLETED) {
            recommendationService.refreshRecommendations(event.getUserId());
        }
    }
    
    private boolean hasAccess(String userId, Content content) {
        // Check subscription level, regional restrictions, parental controls
        return subscriptionService.hasAccess(userId, content) &&
               !content.isRegionBlocked(getUserRegion(userId)) &&
               parentalControlsService.isAllowed(userId, content.getRating());
    }
    
    private List<BitrateOption> getAvailableBitrates(Content content, String deviceType) {
        // Return bitrates supported by the content and device
        return content.getAvailableBitrates().stream()
            .filter(bitrate -> isSupportedByDevice(bitrate, deviceType))
            .collect(Collectors.toList());
    }
    
    private BitrateOption selectInitialBitrate(List<BitrateOption> bitrates, long bandwidth) {
        // Select highest bitrate that fits within estimated bandwidth
        return bitrates.stream()
            .filter(bitrate -> bitrate.getBitsPerSecond() <= bandwidth)
            .max(Comparator.comparing(BitrateOption::getBitsPerSecond))
            .orElse(bitrates.get(0)); // Fallback to lowest bitrate
    }
    
    private List<SegmentUrl> generateSegmentUrls(Content content, CDNService cdnService) {
        // Generate signed CDN URLs for video segments
        return content.getSegments().stream()
            .map(segment -> new SegmentUrl(
                segment.getSequenceNumber(),
                cdnService.generateSignedUrl(content.getId(), segment.getFileName()),
                segment.getDuration()
            ))
            .collect(Collectors.toList());
    }
    
    private void updateUserPreferences(PlaybackEvent event) {
        // Update viewing history
        viewingHistoryService.addToHistory(event.getUserId(), event.getContentId());
        
        // Update genre preferences
        Content content = contentRepository.findById(event.getContentId()).orElse(null);
        if (content != null) {
            for (String genre : content.getGenres()) {
                userPreferencesService.incrementGenrePreference(event.getUserId(), genre);
            }
        }
        
        // Update device preferences
        devicePreferencesService.updatePreferredQuality(event.getUserId(), event.getSelectedBitrate());
    }
}
```

#### Content Delivery Network (Open Connect)
```java
@Service
public class OpenConnectCDNService {
    
    @Autowired
    private CDNRepository cdnRepository;
    
    @Autowired
    private ContentDistributionService distributionService;
    
    public String generateSignedUrl(String contentId, String fileName) {
        // Find optimal CDN server for current request
        CDNLocation optimalLocation = findOptimalCDNLocation();
        
        // Generate signed URL with expiration
        String baseUrl = optimalLocation.getBaseUrl();
        String signature = generateSignature(contentId, fileName, optimalLocation.getSecretKey());
        long expiration = System.currentTimeMillis() + TimeUnit.HOURS.toMillis(4);
        
        return String.format("%s/content/%s/%s?signature=%s&expires=%d",
                           baseUrl, contentId, fileName, signature, expiration);
    }
    
    public void distributeContent(String contentId, Content content) {
        // Distribute new content to CDN locations
        List<CDNLocation> activeLocations = cdnRepository.findActiveLocations();
        
        for (CDNLocation location : activeLocations) {
            // Queue content distribution task
            distributionService.queueDistribution(contentId, content, location);
        }
        
        // Monitor distribution progress
        monitorDistribution(contentId, activeLocations);
    }
    
    public CDNHealthStatus getHealthStatus() {
        List<CDNLocation> locations = cdnRepository.findAllLocations();
        
        CDNHealthStatus status = new CDNHealthStatus();
        status.setTotalLocations(locations.size());
        status.setHealthyLocations(countHealthyLocations(locations));
        status.setTotalCapacity(calculateTotalCapacity(locations));
        status.setUsedCapacity(calculateUsedCapacity(locations));
        
        return status;
    }
    
    private CDNLocation findOptimalCDNLocation() {
        // Use geo-DNS or anycast to route to optimal location
        // Based on user location, server load, network latency
        
        String userRegion = getUserRegion();
        List<CDNLocation> regionalLocations = cdnRepository.findLocationsByRegion(userRegion);
        
        // Select location with lowest load
        return regionalLocations.stream()
            .min(Comparator.comparing(CDNLocation::getCurrentLoad))
            .orElse(regionalLocations.get(0));
    }
    
    private String generateSignature(String contentId, String fileName, String secretKey) {
        try {
            String data = contentId + "/" + fileName;
            Mac mac = Mac.getInstance("HmacSHA256");
            SecretKeySpec secretKeySpec = new SecretKeySpec(secretKey.getBytes(), "HmacSHA256");
            mac.init(secretKeySpec);
            byte[] hash = mac.doFinal(data.getBytes());
            return Base64.getEncoder().encodeToString(hash);
        } catch (Exception e) {
            throw new RuntimeException("Failed to generate signature", e);
        }
    }
    
    private void monitorDistribution(String contentId, List<CDNLocation> locations) {
        // Start monitoring task
        Executors.newSingleThreadScheduledExecutor().scheduleAtFixedRate(() -> {
            int completedCount = countCompletedDistributions(contentId, locations);
            double progress = (double) completedCount / locations.size();
            
            if (progress >= 1.0) {
                // Distribution completed
                contentRepository.markDistributionComplete(contentId);
                // Cancel this monitoring task
            } else {
                // Log progress
                logger.info("Content {} distribution progress: {:.2f}%", contentId, progress * 100);
            }
        }, 30, 30, TimeUnit.SECONDS); // Check every 30 seconds
    }
    
    private int countCompletedDistributions(String contentId, List<CDNLocation> locations) {
        return (int) locations.stream()
            .filter(location -> distributionService.isDistributionComplete(contentId, location))
            .count();
    }
    
    private int countHealthyLocations(List<CDNLocation> locations) {
        return (int) locations.stream()
            .filter(CDNLocation::isHealthy)
            .count();
    }
    
    private long calculateTotalCapacity(List<CDNLocation> locations) {
        return locations.stream()
            .mapToLong(CDNLocation::getTotalCapacity)
            .sum();
    }
    
    private long calculateUsedCapacity(List<CDNLocation> locations) {
        return locations.stream()
            .mapToLong(CDNLocation::getUsedCapacity)
            .sum();
    }
}
```

### 2. Recommendation Engine

#### Collaborative Filtering & Machine Learning
```java
@Service
public class RecommendationEngine {
    
    @Autowired
    private ViewingHistoryRepository viewingHistoryRepository;
    
    @Autowired
    private UserSimilarityService similarityService;
    
    @Autowired
    private ContentMetadataService metadataService;
    
    @Autowired
    private ABTestingService abTestingService;
    
    public List<ContentRecommendation> generateRecommendations(String userId) {
        // Get user's viewing history
        List<ViewingHistory> history = viewingHistoryRepository.findByUserId(userId);
        
        if (history.isEmpty()) {
            // New user - return popular content
            return getPopularContent();
        }
        
        // Apply different recommendation algorithms
        List<ContentRecommendation> recommendations = new ArrayList<>();
        
        // Collaborative filtering
        recommendations.addAll(generateCollaborativeFilteringRecommendations(userId, history));
        
        // Content-based filtering
        recommendations.addAll(generateContentBasedRecommendations(userId, history));
        
        // Trending/popular content
        recommendations.addAll(generateTrendingRecommendations(userId));
        
        // Social recommendations (friends' favorites)
        recommendations.addAll(generateSocialRecommendations(userId));
        
        // Remove duplicates and already viewed content
        recommendations = deduplicateAndFilter(recommendations, history);
        
        // Rank and limit results
        return rankAndLimit(recommendations, 50);
    }
    
    private List<ContentRecommendation> generateCollaborativeFilteringRecommendations(
            String userId, List<ViewingHistory> history) {
        
        // Find similar users
        List<String> similarUsers = similarityService.findSimilarUsers(userId, 100);
        
        // Get content highly rated by similar users but not viewed by current user
        Set<String> viewedContentIds = history.stream()
            .map(ViewingHistory::getContentId)
            .collect(Collectors.toSet());
        
        Map<String, Double> contentScores = new HashMap<>();
        
        for (String similarUser : similarUsers) {
            List<UserRating> ratings = userRatingRepository.findByUserId(similarUser);
            
            for (UserRating rating : ratings) {
                if (!viewedContentIds.contains(rating.getContentId())) {
                    double similarity = similarityService.getSimilarity(userId, similarUser);
                    contentScores.merge(rating.getContentId(), 
                                      rating.getRating() * similarity, Double::sum);
                }
            }
        }
        
        return contentScores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(20)
            .map(entry -> new ContentRecommendation(entry.getKey(), 
                                                  RecommendationType.COLLABORATIVE_FILTERING,
                                                  entry.getValue()))
            .collect(Collectors.toList());
    }
    
    private List<ContentRecommendation> generateContentBasedRecommendations(
            String userId, List<ViewingHistory> history) {
        
        // Extract user preferences from viewing history
        Map<String, Double> genrePreferences = calculateGenrePreferences(history);
        Map<String, Double> actorPreferences = calculateActorPreferences(history);
        
        // Find content matching preferences
        List<Content> candidateContent = contentRepository.findByGenres(
            genrePreferences.keySet(), 100);
        
        return candidateContent.stream()
            .map(content -> {
                double score = calculateContentScore(content, genrePreferences, actorPreferences);
                return new ContentRecommendation(content.getId(), 
                                               RecommendationType.CONTENT_BASED,
                                               score);
            })
            .sorted(Comparator.comparing(ContentRecommendation::getScore).reversed())
            .limit(15)
            .collect(Collectors.toList());
    }
    
    private Map<String, Double> calculateGenrePreferences(List<ViewingHistory> history) {
        Map<String, Double> preferences = new HashMap<>();
        
        for (ViewingHistory item : history) {
            Content content = contentRepository.findById(item.getContentId()).orElse(null);
            if (content != null) {
                double weight = item.getWatchTimePercentage(); // 0.0 to 1.0
                for (String genre : content.getGenres()) {
                    preferences.merge(genre, weight, Double::sum);
                }
            }
        }
        
        // Normalize preferences
        double maxPreference = preferences.values().stream().max(Double::compare).orElse(1.0);
        preferences.replaceAll((k, v) -> v / maxPreference);
        
        return preferences;
    }
    
    private double calculateContentScore(Content content, 
                                       Map<String, Double> genrePreferences,
                                       Map<String, Double> actorPreferences) {
        
        double score = 0.0;
        
        // Genre matching
        for (String genre : content.getGenres()) {
            score += genrePreferences.getOrDefault(genre, 0.0) * 0.7;
        }
        
        // Actor matching
        for (String actor : content.getActors()) {
            score += actorPreferences.getOrDefault(actor, 0.0) * 0.3;
        }
        
        return score;
    }
    
    private List<ContentRecommendation> getPopularContent() {
        // Get most viewed content in the last 30 days
        List<ContentViewCount> popularContent = contentAnalyticsRepository
            .findMostViewedContent(Instant.now().minus(Duration.ofDays(30)), 20);
        
        return popularContent.stream()
            .map(view -> new ContentRecommendation(view.getContentId(), 
                                                 RecommendationType.POPULAR,
                                                 view.getViewCount().doubleValue()))
            .collect(Collectors.toList());
    }
    
    private List<ContentRecommendation> deduplicateAndFilter(
            List<ContentRecommendation> recommendations, List<ViewingHistory> history) {
        
        Set<String> viewedContentIds = history.stream()
            .map(ViewingHistory::getContentId)
            .collect(Collectors.toSet());
        
        return recommendations.stream()
            .filter(rec -> !viewedContentIds.contains(rec.getContentId()))
            .distinct() // Remove duplicates
            .collect(Collectors.toList());
    }
    
    private List<ContentRecommendation> rankAndLimit(List<ContentRecommendation> recommendations, int limit) {
        // Apply A/B testing for ranking algorithm
        String experimentGroup = abTestingService.getExperimentGroup("recommendation_ranking");
        
        if ("algorithm_v2".equals(experimentGroup)) {
            return applyRankingAlgorithmV2(recommendations).stream()
                .limit(limit)
                .collect(Collectors.toList());
        } else {
            return recommendations.stream()
                .sorted(Comparator.comparing(ContentRecommendation::getScore).reversed())
                .limit(limit)
                .collect(Collectors.toList());
        }
    }
    
    private List<ContentRecommendation> applyRankingAlgorithmV2(List<ContentRecommendation> recommendations) {
        // Apply freshness boost, diversity, and other factors
        return recommendations.stream()
            .map(this::applyRankingBoosts)
            .sorted(Comparator.comparing(ContentRecommendation::getScore).reversed())
            .collect(Collectors.toList());
    }
    
    private ContentRecommendation applyRankingBoosts(ContentRecommendation rec) {
        Content content = contentRepository.findById(rec.getContentId()).orElse(null);
        if (content == null) return rec;
        
        double boostedScore = rec.getScore();
        
        // Freshness boost for new content
        long daysSinceRelease = ChronoUnit.DAYS.between(content.getReleaseDate(), Instant.now());
        if (daysSinceRelease <= 30) {
            boostedScore *= (1.0 + (30 - daysSinceRelease) / 100.0); // Up to 30% boost
        }
        
        // Diversity penalty for similar content in results
        // (Implementation would check for genre/actor overlap)
        
        rec.setScore(boostedScore);
        return rec;
    }
}
```

#### Machine Learning Pipeline
```java
@Service
public class MLTrainingPipeline {
    
    @Autowired
    private ViewingHistoryRepository historyRepository;
    
    @Autowired
    private UserProfileRepository profileRepository;
    
    @Autowired
    private ContentMetadataRepository contentRepository;
    
    @Autowired
    private MLModelRepository modelRepository;
    
    @Scheduled(cron = "0 0 2 * * *") // Run daily at 2 AM
    public void trainRecommendationModel() {
        logger.info("Starting daily ML model training");
        
        try {
            // Gather training data
            TrainingData data = gatherTrainingData();
            
            // Train collaborative filtering model
            MLModel cfModel = trainCollaborativeFilteringModel(data);
            
            // Train content-based model
            MLModel cbModel = trainContentBasedModel(data);
            
            // Train hybrid model
            MLModel hybridModel = trainHybridModel(cfModel, cbModel, data);
            
            // Evaluate models
            ModelEvaluation evaluation = evaluateModels(cfModel, cbModel, hybridModel, data);
            
            // Deploy best performing model
            deployBestModel(evaluation);
            
            logger.info("ML model training completed successfully");
            
        } catch (Exception e) {
            logger.error("ML model training failed", e);
            // Send alert and fallback to previous model
            alertService.sendAlert("ML Training Failed", 
                                 "ML model training failed: " + e.getMessage());
        }
    }
    
    private TrainingData gatherTrainingData() {
        // Gather data from last 90 days
        Instant ninetyDaysAgo = Instant.now().minus(Duration.ofDays(90));
        
        List<ViewingHistory> viewingHistory = historyRepository.findByViewedAfter(ninetyDaysAgo);
        List<UserProfile> userProfiles = profileRepository.findAll();
        List<ContentMetadata> contentMetadata = contentRepository.findAll();
        
        return new TrainingData(viewingHistory, userProfiles, contentMetadata);
    }
    
    private MLModel trainCollaborativeFilteringModel(TrainingData data) {
        // Implement matrix factorization or neural collaborative filtering
        // This would use TensorFlow, PyTorch, or similar ML framework
        
        // For this example, we'll simulate the training
        logger.info("Training collaborative filtering model");
        
        // Create user-item interaction matrix
        Map<String, Map<String, Double>> userItemMatrix = buildUserItemMatrix(data);
        
        // Train model (actual implementation would use ML library)
        MLModel model = new MLModel();
        model.setType(ModelType.COLLABORATIVE_FILTERING);
        model.setTrainingDate(Instant.now());
        model.setParameters(Map.of("latent_factors", "64", "learning_rate", "0.001"));
        
        return model;
    }
    
    private Map<String, Map<String, Double>> buildUserItemMatrix(TrainingData data) {
        Map<String, Map<String, Double>> matrix = new HashMap<>();
        
        for (ViewingHistory history : data.getViewingHistory()) {
            matrix.computeIfAbsent(history.getUserId(), k -> new HashMap<>())
                  .put(history.getContentId(), calculateRating(history));
        }
        
        return matrix;
    }
    
    private double calculateRating(ViewingHistory history) {
        // Calculate implicit rating based on watch time percentage
        double watchPercentage = history.getWatchTimePercentage();
        
        if (watchPercentage >= 0.9) return 5.0;  // Completed
        if (watchPercentage >= 0.7) return 4.0;  // Mostly watched
        if (watchPercentage >= 0.5) return 3.0;  // Half watched
        if (watchPercentage >= 0.2) return 2.0;  // Started watching
        return 1.0;  // Barely watched
    }
    
    private ModelEvaluation evaluateModels(MLModel cfModel, MLModel cbModel, 
                                         MLModel hybridModel, TrainingData data) {
        
        // Split data into training and test sets
        TrainingDataSplit split = splitDataForEvaluation(data);
        
        // Evaluate each model
        double cfPrecision = evaluatePrecision(cfModel, split.getTestData());
        double cbPrecision = evaluatePrecision(cbModel, split.getTestData());
        double hybridPrecision = evaluatePrecision(hybridModel, split.getTestData());
        
        ModelEvaluation evaluation = new ModelEvaluation();
        evaluation.setCfModelPrecision(cfPrecision);
        evaluation.setCbModelPrecision(cbPrecision);
        evaluation.setHybridModelPrecision(hybridPrecision);
        
        return evaluation;
    }
    
    private double evaluatePrecision(MLModel model, List<ViewingHistory> testData) {
        // Calculate precision@10 for each user
        Map<String, List<String>> userRecommendations = new HashMap<>();
        
        // Generate recommendations for test users
        for (String userId : getUniqueUserIds(testData)) {
            List<String> recommendations = generateRecommendations(model, userId, 10);
            userRecommendations.put(userId, recommendations);
        }
        
        // Calculate average precision
        double totalPrecision = 0.0;
        int userCount = 0;
        
        for (ViewingHistory testItem : testData) {
            List<String> recommendations = userRecommendations.get(testItem.getUserId());
            if (recommendations != null && recommendations.contains(testItem.getContentId())) {
                int rank = recommendations.indexOf(testItem.getContentId()) + 1;
                totalPrecision += 1.0 / rank; // Precision contribution
            }
            userCount++;
        }
        
        return totalPrecision / userCount;
    }
    
    private void deployBestModel(ModelEvaluation evaluation) {
        MLModel bestModel;
        
        if (evaluation.getHybridModelPrecision() >= evaluation.getCfModelPrecision() &&
            evaluation.getHybridModelPrecision() >= evaluation.getCbModelPrecision()) {
            bestModel = modelRepository.findByType(ModelType.HYBRID);
        } else if (evaluation.getCfModelPrecision() >= evaluation.getCbModelPrecision()) {
            bestModel = modelRepository.findByType(ModelType.COLLABORATIVE_FILTERING);
        } else {
            bestModel = modelRepository.findByType(ModelType.CONTENT_BASED);
        }
        
        // Mark model as active
        bestModel.setActive(true);
        modelRepository.save(bestModel);
        
        // Update recommendation service to use new model
        recommendationService.setActiveModel(bestModel);
        
        logger.info("Deployed best performing model: {}", bestModel.getType());
    }
}
```

### 3. Content Encoding Pipeline

#### Video Processing at Scale
```java
@Service
public class VideoEncodingService {
    
    @Autowired
    private ContentRepository contentRepository;
    
    @Autowired
    private EncodingJobRepository jobRepository;
    
    @Autowired
    private S3Client s3Client;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void startEncodingPipeline(String contentId, String sourceFileKey) {
        // Create encoding jobs for different bitrates and formats
        List<EncodingJob> jobs = createEncodingJobs(contentId, sourceFileKey);
        
        // Save jobs to database
        jobs.forEach(jobRepository::save);
        
        // Queue jobs for processing
        for (EncodingJob job : jobs) {
            kafkaTemplate.send("encoding-jobs", "job-created", job);
        }
        
        logger.info("Started encoding pipeline for content {} with {} jobs", contentId, jobs.size());
    }
    
    private List<EncodingJob> createEncodingJobs(String contentId, String sourceFileKey) {
        List<EncodingJob> jobs = new ArrayList<>();
        
        // Define encoding profiles
        List<EncodingProfile> profiles = Arrays.asList(
            new EncodingProfile("240p", 400, 240, 24),
            new EncodingProfile("360p", 800, 360, 25),
            new EncodingProfile("480p", 1200, 480, 25),
            new EncodingProfile("720p", 2400, 720, 25),
            new EncodingProfile("1080p", 4800, 1080, 25),
            new EncodingProfile("4K", 12000, 2160, 25)
        );
        
        for (EncodingProfile profile : profiles) {
            EncodingJob job = new EncodingJob();
            job.setId(generateJobId());
            job.setContentId(contentId);
            job.setSourceFileKey(sourceFileKey);
            job.setProfile(profile);
            job.setStatus(EncodingStatus.QUEUED);
            job.setCreatedAt(Instant.now());
            
            jobs.add(job);
        }
        
        return jobs;
    }
    
    @KafkaListener(topics = "encoding-jobs", groupId = "video-encoding", concurrency = "10")
    public void processEncodingJob(String jobJson) {
        try {
            EncodingJob job = objectMapper.readValue(jobJson, EncodingJob.class);
            
            // Update job status
            job.setStatus(EncodingStatus.PROCESSING);
            job.setStartedAt(Instant.now());
            jobRepository.save(job);
            
            // Download source file
            GetObjectRequest getRequest = GetObjectRequest.builder()
                .bucket("netflix-source-content")
                .key(job.getSourceFileKey())
                .build();
            
            Path tempFile = Files.createTempFile("source", ".mp4");
            try (InputStream inputStream = s3Client.getObject(getRequest);
                 FileOutputStream outputStream = new FileOutputStream(tempFile.toFile())) {
                
                // Download file
                byte[] buffer = new byte[8192];
                int bytesRead;
                while ((bytesRead = inputStream.read(buffer)) != -1) {
                    outputStream.write(buffer, 0, bytesRead);
                }
            }
            
            // Encode video
            String outputFileName = encodeVideo(tempFile, job.getProfile());
            
            // Upload encoded file
            String outputKey = "encoded/" + job.getContentId() + "/" + job.getProfile().getName() + ".mp4";
            s3Client.putObject(PutObjectRequest.builder()
                .bucket("netflix-encoded-content")
                .key(outputKey)
                .build(), Paths.get(outputFileName));
            
            // Update job status
            job.setStatus(EncodingStatus.COMPLETED);
            job.setCompletedAt(Instant.now());
            job.setOutputFileKey(outputKey);
            jobRepository.save(job);
            
            // Check if all jobs for content are completed
            checkContentEncodingCompletion(job.getContentId());
            
            // Clean up temp files
            Files.deleteIfExists(tempFile);
            Files.deleteIfExists(Paths.get(outputFileName));
            
        } catch (Exception e) {
            logger.error("Failed to process encoding job", e);
            
            // Update job status to failed
            EncodingJob job = objectMapper.readValue(jobJson, EncodingJob.class);
            job.setStatus(EncodingStatus.FAILED);
            job.setErrorMessage(e.getMessage());
            jobRepository.save(job);
        }
    }
    
    private String encodeVideo(Path inputFile, EncodingProfile profile) throws Exception {
        // Use FFmpeg for video encoding
        String outputFileName = "encoded_" + profile.getName() + "_" + System.nanoTime() + ".mp4";
        
        ProcessBuilder pb = new ProcessBuilder(
            "ffmpeg",
            "-i", inputFile.toString(),
            "-c:v", "libx264",
            "-b:v", profile.getBitrate() + "k",
            "-maxrate", profile.getBitrate() + "k",
            "-bufsize", (profile.getBitrate() * 2) + "k",
            "-vf", "scale=-2:" + profile.getHeight(),
            "-c:a", "aac",
            "-b:a", "128k",
            "-r", String.valueOf(profile.getFrameRate()),
            "-y", // Overwrite output files
            outputFileName
        );
        
        pb.redirectErrorStream(true);
        Process process = pb.start();
        
        // Wait for completion with timeout
        if (!process.waitFor(30, TimeUnit.MINUTES)) {
            process.destroyForcibly();
            throw new EncodingException("Encoding timeout");
        }
        
        if (process.exitValue() != 0) {
            throw new EncodingException("Encoding failed with exit code: " + process.exitValue());
        }
        
        return outputFileName;
    }
    
    private void checkContentEncodingCompletion(String contentId) {
        List<EncodingJob> jobs = jobRepository.findByContentId(contentId);
        
        boolean allCompleted = jobs.stream()
            .allMatch(job -> job.getStatus() == EncodingStatus.COMPLETED);
        
        if (allCompleted) {
            // Mark content as encoding complete
            Content content = contentRepository.findById(contentId).orElse(null);
            if (content != null) {
                content.setEncodingStatus(ContentEncodingStatus.COMPLETED);
                contentRepository.save(content);
                
                // Notify CDN to start distribution
                kafkaTemplate.send("content-events", "encoding-completed", 
                    new ContentEncodingCompletedEvent(contentId));
            }
        }
    }
    
    private String generateJobId() {
        return "ENC-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
}
```

## Scaling Challenges & Solutions

### 1. Global Content Delivery

#### Open Connect CDN Architecture
```java
@Configuration
public class OpenConnectConfig {
    
    @Bean
    public CDNManager openConnectManager() {
        List<CDNAppliance> appliances = Arrays.asList(
            // North America
            new CDNAppliance("oc-na-001", "us-east-1", 100, 80),
            new CDNAppliance("oc-na-002", "us-west-2", 100, 85),
            new CDNAppliance("oc-na-003", "us-central-1", 100, 75),
            
            // Europe
            new CDNAppliance("oc-eu-001", "eu-west-1", 100, 90),
            new CDNAppliance("oc-eu-002", "eu-central-1", 100, 70),
            
            // Asia Pacific
            new CDNAppliance("oc-ap-001", "ap-southeast-1", 100, 65),
            new CDNAppliance("oc-ap-002", "ap-northeast-1", 100, 80),
            new CDNAppliance("oc-ap-003", "ap-south-1", 100, 60)
        );
        
        return new OpenConnectManager(appliances);
    }
}

public class OpenConnectManager {
    
    private final List<CDNAppliance> appliances;
    private final Map<String, List<CDNAppliance>> regionalAppliances;
    
    public OpenConnectManager(List<CDNAppliance> appliances) {
        this.appliances = appliances;
        this.regionalAppliances = appliances.stream()
            .collect(Collectors.groupingBy(CDNAppliance::getRegion));
    }
    
    public CDNAppliance selectOptimalAppliance(String contentId, String userRegion, 
                                             String userNetwork) {
        
        List<CDNAppliance> candidates = regionalAppliances.getOrDefault(userRegion, appliances);
        
        // Filter by content availability
        candidates = candidates.stream()
            .filter(appliance -> appliance.hasContent(contentId))
            .collect(Collectors.toList());
        
        if (candidates.isEmpty()) {
            // Fallback to any appliance with content
            candidates = appliances.stream()
                .filter(appliance -> appliance.hasContent(contentId))
                .collect(Collectors.toList());
        }
        
        if (candidates.isEmpty()) {
            throw new ContentNotAvailableException(contentId);
        }
        
        // Select appliance with lowest load
        return candidates.stream()
            .min(Comparator.comparing(CDNAppliance::getCurrentLoad))
            .orElse(candidates.get(0));
    }
    
    public void distributeContent(String contentId, byte[] contentData) {
        // Distribute content to all appliances
        for (CDNAppliance appliance : appliances) {
            // Queue distribution task
            appliance.queueContentDistribution(contentId, contentData);
        }
        
        // Monitor distribution progress
        monitorDistribution(contentId);
    }
    
    private void monitorDistribution(String contentId) {
        AtomicInteger completedCount = new AtomicInteger(0);
        
        for (CDNAppliance appliance : appliances) {
            appliance.addDistributionCallback(contentId, () -> {
                int completed = completedCount.incrementAndGet();
                logger.info("Content {} distribution progress: {}/{}", 
                          contentId, completed, appliances.size());
                
                if (completed == appliances.size()) {
                    logger.info("Content {} distribution completed", contentId);
                    // Notify other services
                }
            });
        }
    }
}

public class CDNAppliance {
    
    private final String id;
    private final String region;
    private final int totalCapacity; // TB
    private final AtomicInteger currentLoad = new AtomicInteger(0);
    private final Set<String> availableContent = ConcurrentHashMap.newKeySet();
    private final Map<String, Runnable> distributionCallbacks = new ConcurrentHashMap<>();
    
    public CDNAppliance(String id, String region, int totalCapacity, int initialLoadPercent) {
        this.id = id;
        this.region = region;
        this.totalCapacity = totalCapacity;
        this.currentLoad.set((totalCapacity * initialLoadPercent) / 100);
    }
    
    public boolean hasContent(String contentId) {
        return availableContent.contains(contentId);
    }
    
    public void queueContentDistribution(String contentId, byte[] contentData) {
        // Simulate async distribution
        Executors.newSingleThreadExecutor().submit(() -> {
            try {
                // Simulate network transfer time
                Thread.sleep(ThreadLocalRandom.current().nextInt(1000, 5000));
                
                availableContent.add(contentId);
                
                // Update load
                currentLoad.addAndGet(1); // Assume 1 unit of load per content
                
                // Execute callback
                Runnable callback = distributionCallbacks.remove(contentId);
                if (callback != null) {
                    callback.run();
                }
                
            } catch (Exception e) {
                logger.error("Failed to distribute content {} to appliance {}", contentId, id, e);
            }
        });
    }
    
    public void addDistributionCallback(String contentId, Runnable callback) {
        distributionCallbacks.put(contentId, callback);
    }
    
    public double getCurrentLoad() {
        return (double) currentLoad.get() / totalCapacity;
    }
    
    // Getters
    public String getId() { return id; }
    public String getRegion() { return region; }
    public int getTotalCapacity() { return totalCapacity; }
}
```

### 2. Personalization at Scale

#### Real-time Recommendation Serving
```java
@Service
public class RealTimeRecommendationService {
    
    @Autowired
    private UserActivityStreamProcessor activityProcessor;
    
    @Autowired
    private RecommendationCache cache;
    
    @Autowired
    private FallbackRecommendationService fallbackService;
    
    public List<String> getRealTimeRecommendations(String userId, int count, 
                                                  Map<String, Object> context) {
        
        try {
            // Check cache first
            List<String> cached = cache.getRecommendations(userId, context);
            if (cached != null && !cached.isEmpty()) {
                return cached.subList(0, Math.min(count, cached.size()));
            }
            
            // Generate real-time recommendations
            List<String> recommendations = generateRealTimeRecommendations(userId, context);
            
            if (recommendations.isEmpty()) {
                // Fallback to pre-computed recommendations
                recommendations = fallbackService.getFallbackRecommendations(userId, count);
            }
            
            // Cache results
            cache.putRecommendations(userId, recommendations, context);
            
            return recommendations.subList(0, Math.min(count, recommendations.size()));
            
        } catch (Exception e) {
            logger.error("Failed to generate real-time recommendations for user {}", userId, e);
            return fallbackService.getFallbackRecommendations(userId, count);
        }
    }
    
    private List<String> generateRealTimeRecommendations(String userId, Map<String, Object> context) {
        // Get recent user activity
        List<UserActivity> recentActivity = activityProcessor.getRecentActivity(userId, 
            Instant.now().minus(Duration.ofMinutes(30)));
        
        // Extract signals from recent activity
        Set<String> recentGenres = extractGenresFromActivity(recentActivity);
        Set<String> recentActors = extractActorsFromActivity(recentActivity);
        String currentMood = inferMoodFromActivity(recentActivity);
        
        // Query content index for recommendations
        List<String> candidates = contentIndex.searchByAttributes(Map.of(
            "genres", recentGenres,
            "actors", recentActors,
            "mood", currentMood,
            "recency", "high" // Prefer newer content
        ));
        
        // Apply collaborative filtering boost
        Map<String, Double> collaborativeScores = getCollaborativeScores(userId, candidates);
        
        // Rank candidates
        return candidates.stream()
            .sorted((a, b) -> {
                double scoreA = calculateRealTimeScore(a, recentGenres, recentActors, collaborativeScores);
                double scoreB = calculateRealTimeScore(b, recentGenres, recentActors, collaborativeScores);
                return Double.compare(scoreB, scoreA);
            })
            .collect(Collectors.toList());
    }
    
    private Set<String> extractGenresFromActivity(List<UserActivity> activities) {
        return activities.stream()
            .filter(activity -> activity.getType() == ActivityType.WATCH_STARTED || 
                              activity.getType() == ActivityType.WATCH_COMPLETED)
            .flatMap(activity -> getContentGenres(activity.getContentId()).stream())
            .collect(Collectors.toSet());
    }
    
    private String inferMoodFromActivity(List<UserActivity> activities) {
        // Simple mood inference based on content ratings and genres
        boolean hasComedy = activities.stream()
            .anyMatch(activity -> getContentGenres(activity.getContentId()).contains("Comedy"));
        
        boolean hasDrama = activities.stream()
            .anyMatch(activity -> getContentGenres(activity.getContentId()).contains("Drama"));
        
        if (hasComedy && !hasDrama) return "happy";
        if (hasDrama && !hasComedy) return "serious";
        return "neutral";
    }
    
    private Map<String, Double> getCollaborativeScores(String userId, List<String> contentIds) {
        // Get collaborative filtering scores for content items
        // This would query a pre-computed collaborative filtering model
        return contentIds.stream()
            .collect(Collectors.toMap(
                contentId -> contentId,
                contentId -> 0.5 // Default neutral score
            ));
    }
    
    private double calculateRealTimeScore(String contentId, Set<String> recentGenres, 
                                        Set<String> recentActors, 
                                        Map<String, Double> collaborativeScores) {
        
        double score = 0.0;
        
        // Genre relevance
        Set<String> contentGenres = getContentGenres(contentId);
        long genreMatches = contentGenres.stream()
            .filter(recentGenres::contains)
            .count();
        score += genreMatches * 0.4;
        
        // Actor relevance
        Set<String> contentActors = getContentActors(contentId);
        long actorMatches = contentActors.stream()
            .filter(recentActors::contains)
            .count();
        score += actorMatches * 0.3;
        
        // Collaborative score
        score += collaborativeScores.getOrDefault(contentId, 0.5) * 0.3;
        
        return score;
    }
    
    private Set<String> getContentGenres(String contentId) {
        // Query content metadata
        return contentMetadataCache.getGenres(contentId);
    }
    
    private Set<String> getContentActors(String contentId) {
        // Query content metadata
        return contentMetadataCache.getActors(contentId);
    }
}
```

## Performance Optimizations

### 1. Caching Strategy

#### Multi-Level Caching Architecture
```java
@Configuration
public class NetflixCachingConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheManagerBuilder builder = RedisCacheManager.builder(redisConnectionFactory);
        
        // Configure different TTLs for different data types
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        
        // User data - long TTL
        cacheConfigurations.put("userProfiles", 
            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofHours(24)));
        
        // Content metadata - medium TTL
        cacheConfigurations.put("contentMetadata", 
            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofHours(6)));
        
        // Recommendations - short TTL
        cacheConfigurations.put("recommendations", 
            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(30)));
        
        // Playback manifests - very short TTL
        cacheConfigurations.put("playbackManifests", 
            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(5)));
        
        return builder.withInitialCacheConfigurations(cacheConfigurations).build();
    }
    
    @Bean
    public CaffeineCacheManager caffeineCacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        
        // L1 cache for hot data
        cacheManager.setCacheNames(Arrays.asList("hotContent", "popularTitles", "trending"));
        
        return cacheManager;
    }
}

@Service
public class IntelligentCacheService {
    
    @Autowired
    private CacheManager redisCacheManager;
    
    @Autowired
    private CaffeineCacheManager caffeineCacheManager;
    
    @Autowired
    private ContentPopularityService popularityService;
    
    public <T> T getWithMultiLevelCache(String key, Class<T> type, Duration redisTtl) {
        // Try L1 cache first
        Cache caffeineCache = caffeineCacheManager.getCache("hotContent");
        T l1Result = caffeineCache.get(key, () -> null);
        
        if (l1Result != null) {
            return l1Result;
        }
        
        // Try L2 cache
        Cache redisCache = redisCacheManager.getCache("contentMetadata");
        T l2Result = redisCache.get(key, () -> null);
        
        if (l2Result != null) {
            // Promote to L1 cache
            caffeineCache.put(key, l2Result);
            return l2Result;
        }
        
        return null; // Cache miss
    }
    
    public void putWithMultiLevelCache(String key, Object value, String cacheType) {
        // Determine TTL based on content popularity
        Duration ttl = determineTTL(key, cacheType);
        
        // Store in Redis with appropriate TTL
        Cache redisCache = getRedisCacheForType(cacheType);
        if (redisCache != null) {
            redisCache.put(key, value);
        }
        
        // Store in Caffeine if it's hot content
        if (isHotContent(key)) {
            Cache caffeineCache = caffeineCacheManager.getCache("hotContent");
            caffeineCache.put(key, value);
        }
    }
    
    private Duration determineTTL(String key, String cacheType) {
        switch (cacheType) {
            case "userProfiles":
                return Duration.ofHours(24);
            case "contentMetadata":
                // Vary TTL based on popularity
                if (popularityService.isVeryPopular(key)) {
                    return Duration.ofHours(12);
                } else if (popularityService.isPopular(key)) {
                    return Duration.ofHours(6);
                } else {
                    return Duration.ofHours(2);
                }
            case "recommendations":
                return Duration.ofMinutes(30);
            case "playbackManifests":
                return Duration.ofMinutes(5);
            default:
                return Duration.ofMinutes(10);
        }
    }
    
    private Cache getRedisCacheForType(String cacheType) {
        // Map cache types to Redis cache names
        switch (cacheType) {
            case "userProfiles":
                return redisCacheManager.getCache("userProfiles");
            case "contentMetadata":
                return redisCacheManager.getCache("contentMetadata");
            case "recommendations":
                return redisCacheManager.getCache("recommendations");
            case "playbackManifests":
                return redisCacheManager.getCache("playbackManifests");
            default:
                return null;
        }
    }
    
    private boolean isHotContent(String key) {
        // Check if content is in top 1% of requests
        return popularityService.isHotContent(key);
    }
    
    // Cache warming for popular content
    @Scheduled(fixedRate = 300000) // Every 5 minutes
    public void warmHotContentCache() {
        List<String> hotContentIds = popularityService.getHotContentIds(1000);
        
        Cache caffeineCache = caffeineCacheManager.getCache("hotContent");
        Cache redisCache = redisCacheManager.getCache("contentMetadata");
        
        for (String contentId : hotContentIds) {
            // Pre-load from Redis to Caffeine
            Object content = redisCache.get(contentId, () -> null);
            if (content != null) {
                caffeineCache.put(contentId, content);
            }
        }
        
        logger.info("Warmed cache with {} hot content items", hotContentIds.size());
    }
}
```

### 2. Database Optimizations

#### EVCache (Eventually Consistent Cache)
```java
@Service
public class EVCacheService {
    
    @Autowired
    private EVCacheClient evCacheClient;
    
    @Autowired
    private DatabaseFallbackService fallbackService;
    
    public <T> T get(String key, Class<T> type) {
        try {
            String cachedValue = evCacheClient.get(key);
            if (cachedValue != null) {
                return objectMapper.readValue(cachedValue, type);
            }
        } catch (Exception e) {
            logger.warn("EVCache get failed for key {}, falling back to database", key, e);
        }
        
        // Fallback to database
        return fallbackService.getFromDatabase(key, type);
    }
    
    public void put(String key, Object value, Duration ttl) {
        try {
            String serializedValue = objectMapper.writeValueAsString(value);
            evCacheClient.set(key, serializedValue, (int) ttl.getSeconds());
        } catch (Exception e) {
            logger.error("EVCache put failed for key {}", key, e);
            // Don't fail the operation, just log
        }
    }
    
    public void delete(String key) {
        try {
            evCacheClient.delete(key);
        } catch (Exception e) {
            logger.error("EVCache delete failed for key {}", key, e);
        }
    }
    
    // Asynchronous operations for better performance
    public CompletableFuture<Void> putAsync(String key, Object value, Duration ttl) {
        return CompletableFuture.runAsync(() -> put(key, value, ttl));
    }
    
    public CompletableFuture<Void> deleteAsync(String key) {
        return CompletableFuture.runAsync(() -> delete(key));
    }
    
    // Batch operations
    public void putBatch(Map<String, Object> keyValuePairs, Duration ttl) {
        for (Map.Entry<String, Object> entry : keyValuePairs.entrySet()) {
            putAsync(entry.getKey(), entry.getValue(), ttl);
        }
    }
    
    public Map<String, Object> getBatch(Set<String> keys, Class<?> type) {
        Map<String, Object> results = new HashMap<>();
        
        // Try to get from cache first
        Map<String, String> cachedValues = evCacheClient.getBulk(keys);
        
        for (String key : keys) {
            String cachedValue = cachedValues.get(key);
            if (cachedValue != null) {
                try {
                    results.put(key, objectMapper.readValue(cachedValue, type));
                } catch (Exception e) {
                    logger.warn("Failed to deserialize cached value for key {}", key, e);
                }
            }
        }
        
        // Get missing values from database
        Set<String> missingKeys = new HashSet<>(keys);
        missingKeys.removeAll(results.keySet());
        
        if (!missingKeys.isEmpty()) {
            Map<String, Object> dbResults = fallbackService.getBatchFromDatabase(missingKeys, type);
            results.putAll(dbResults);
            
            // Cache the database results asynchronously
            for (Map.Entry<String, Object> entry : dbResults.entrySet()) {
                putAsync(entry.getKey(), entry.getValue(), Duration.ofHours(1));
            }
        }
        
        return results;
    }
}

@Configuration
public class EVCacheConfig {
    
    @Bean
    public EVCacheClient evCacheClient() {
        EVCacheClientBuilder builder = new EVCacheClientBuilder()
            .setAppName("netflix-app")
            .setDefaultTTL(3600) // 1 hour default
            .setMaxReadQueueSize(100)
            .setMaxWriteQueueSize(100)
            .addServerGroup("server-group-1", Arrays.asList(
                "evcache-001:11211",
                "evcache-002:11211",
                "evcache-003:11211"
            ))
            .addServerGroup("server-group-2", Arrays.asList(
                "evcache-004:11211",
                "evcache-005:11211",
                "evcache-006:11211"
            ));
        
        return builder.build();
    }
}
```

## Monitoring & Analytics

### Real-time Analytics Pipeline
```java
@Service
public class RealTimeAnalyticsService {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private StreamingAnalyticsProcessor processor;
    
    public void recordPlaybackEvent(PlaybackEvent event) {
        // Send to analytics pipeline
        kafkaTemplate.send("playback-events", "event-recorded", event);
        
        // Process real-time metrics
        processor.processPlaybackEvent(event);
    }
    
    public void recordUserAction(UserActionEvent event) {
        kafkaTemplate.send("user-action-events", "action-recorded", event);
        processor.processUserAction(event);
    }
    
    public void recordContentMetrics(ContentMetricsEvent event) {
        kafkaTemplate.send("content-metrics-events", "metrics-recorded", event);
        processor.processContentMetrics(event);
    }
}

@Service
public class StreamingAnalyticsProcessor {
    
    @Autowired
    private WindowedMetricsStore metricsStore;
    
    @Autowired
    private RealTimeDashboardService dashboardService;
    
    public void processPlaybackEvent(PlaybackEvent event) {
        // Update real-time metrics
        metricsStore.incrementCounter("total_watch_time", event.getWatchTimeMs());
        metricsStore.incrementCounter("content_views:" + event.getContentId(), 1);
        metricsStore.recordGauge("concurrent_streams", getCurrentConcurrentStreams());
        
        // Update user metrics
        metricsStore.updateUserMetrics(event.getUserId(), event);
        
        // Check for alerting conditions
        checkPlaybackAlerts(event);
        
        // Update dashboard
        dashboardService.updatePlaybackMetrics(event);
    }
    
    public void processUserAction(UserActionEvent event) {
        // Track user engagement
        metricsStore.incrementCounter("user_actions:" + event.getActionType(), 1);
        metricsStore.updateUserEngagement(event.getUserId(), event);
        
        // Update recommendation effectiveness
        if (event.getActionType().equals("content_selected_from_recommendation")) {
            metricsStore.incrementCounter("recommendation_clicks", 1);
            metricsStore.recordRecommendationSuccess(event.getRecommendationId());
        }
    }
    
    public void processContentMetrics(ContentMetricsEvent event) {
        // Update content performance
        metricsStore.updateContentMetrics(event.getContentId(), event);
        
        // Calculate engagement rate
        long views = metricsStore.getCounter("content_views:" + event.getContentId());
        long completions = metricsStore.getCounter("content_completions:" + event.getContentId());
        
        if (views > 0) {
            double completionRate = (double) completions / views;
            metricsStore.recordGauge("content_completion_rate:" + event.getContentId(), completionRate);
        }
    }
    
    private void checkPlaybackAlerts(PlaybackEvent event) {
        // Check for unusual patterns
        double errorRate = metricsStore.getErrorRateLast5Minutes();
        if (errorRate > 0.05) { // 5% error rate
            alertService.sendAlert("High Playback Error Rate", 
                                 "Playback error rate is " + String.format("%.2f%%", errorRate * 100));
        }
        
        // Check for content popularity spikes
        long viewRate = metricsStore.getContentViewRate(event.getContentId(), Duration.ofMinutes(5));
        if (viewRate > 10000) { // 10k views per 5 minutes
            alertService.sendAlert("Viral Content Detected", 
                                 "Content " + event.getContentId() + " has " + viewRate + " views in 5 minutes");
        }
    }
    
    private long getCurrentConcurrentStreams() {
        // Estimate concurrent streams (simplified)
        return metricsStore.getActiveStreamCount();
    }
}

@Service
public class RealTimeDashboardService {
    
    @Autowired
    private WebSocketService webSocketService;
    
    @Autowired
    private MetricsAggregator aggregator;
    
    public void updatePlaybackMetrics(PlaybackEvent event) {
        // Aggregate metrics for dashboard
        DashboardMetrics metrics = aggregator.aggregateRealTimeMetrics();
        
        // Send to connected dashboard clients
        webSocketService.broadcastToChannel("dashboard", metrics);
        
        // Update specific content metrics
        ContentMetrics contentMetrics = aggregator.getContentMetrics(event.getContentId());
        webSocketService.broadcastToChannel("content:" + event.getContentId(), contentMetrics);
    }
    
    // Send periodic updates
    @Scheduled(fixedRate = 5000) // Every 5 seconds
    public void sendPeriodicUpdates() {
        DashboardMetrics metrics = aggregator.aggregateRealTimeMetrics();
        webSocketService.broadcastToChannel("dashboard", metrics);
    }
}
```

## Lessons Learned

### Architectural Decisions
1. **Build for Scale**: Netflix designed systems to handle 10x growth from day one
2. **Accept Eventual Consistency**: Prioritized availability over strong consistency
3. **Optimize for Reads**: Heavy read workload drove sophisticated caching strategies
4. **Embrace Microservices**: Decomposition enabled independent scaling and deployment

### Technical Innovations
1. **Open Connect CDN**: Custom CDN for efficient global content delivery
2. **Chaos Monkey**: Failure injection testing for resilience
3. **EVCache**: Eventually consistent caching layer
4. **Titus**: Container management system predating Kubernetes

### Operational Excellence
1. **Data-Driven Culture**: Every decision backed by A/B testing and analytics
2. **Global Infrastructure**: Multi-region deployment for worldwide performance
3. **Continuous Deployment**: Multiple deployments per day
4. **Security First**: Comprehensive security measures for content protection

### Business Insights
1. **Personalization Pays**: Recommendation system drives significant engagement
2. **Quality over Quantity**: Focus on high-quality original content
3. **Global Expansion**: Localized content and recommendations for different markets
4. **Mobile First**: Mobile streaming experience drives user growth

## Future Challenges

### Emerging Technologies
- **4K and HDR**: Higher quality streaming with larger files
- **Interactive Content**: Choose-your-own-adventure style programming
- **Offline Viewing**: Improved offline experience across devices
- **AI-Generated Content**: Machine learning for content creation

### Technical Challenges
- **Video Codec Optimization**: More efficient compression for bandwidth savings
- **Edge Computing**: Content processing closer to users
- **5G Integration**: Adaptive streaming for variable network conditions
- **Privacy Regulations**: GDPR, CCPA compliance with personalization

Netflix's architecture demonstrates how to build a truly global, highly personalized streaming service that can handle massive scale while delivering exceptional user experience. The combination of innovative technology, data-driven decision making, and relentless focus on performance has made Netflix the world's leading streaming platform.
