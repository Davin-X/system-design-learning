# URL Shortener System Design

Design a URL shortening service like bit.ly or TinyURL that converts long URLs into short, shareable links.

## Requirements Analysis

### Functional Requirements
- **Shorten URLs**: Convert long URLs to short aliases
- **Redirect**: Short URLs redirect to original URLs
- **Custom URLs**: Allow custom short codes (optional)
- **Analytics**: Track click counts, geolocation, referrer data
- **Expiration**: URLs can have expiration dates

### Non-Functional Requirements
- **High Availability**: 99.9% uptime
- **Low Latency**: <100ms for redirects, <500ms for shortening
- **Scalability**: Handle millions of URLs and billions of redirects
- **Durability**: URLs persist indefinitely (or until expiration)
- **Security**: Prevent malicious URLs, rate limiting

### Constraints
- **Traffic**: 100 million URLs created per month
- **Read/Write Ratio**: 100:1 (mostly redirects)
- **Data Retention**: URLs stored permanently unless expired
- **Geographic Distribution**: Global service

## Capacity Estimation

### Storage
- **URLs per month**: 100 million
- **URL metadata**: 500 bytes per URL (original URL, short code, metadata)
- **Monthly storage**: 100M × 500 bytes = 50GB/month
- **Analytics data**: 10× clicks per URL = 1 billion clicks/month
- **Analytics storage**: 1B × 100 bytes = 100GB/month
- **Total storage**: ~150GB/month, 1.8TB/year

### Traffic
- **Write QPS**: 100M URLs/month ÷ 30 days ÷ 24 hours ÷ 3600 seconds ≈ 40 URLs/second
- **Read QPS**: 40 × 100 = 4,000 redirects/second (conservative estimate)
- **Peak QPS**: 3× average = 12,000 redirects/second

## API Design

### Shorten URL
```http
POST /api/shorten
Content-Type: application/json

{
    "url": "https://example.com/very/long/url/path?param1=value1&param2=value2",
    "custom_code": "optional-custom-code",
    "expires_in_days": 30
}
```

**Response:**
```json
{
    "short_url": "https://short.ly/abc123",
    "original_url": "https://example.com/very/long/url/path?param1=value1&param2=value2",
    "expires_at": "2024-01-30T00:00:00Z"
}
```

### Redirect
```http
GET /abc123
```

**Response:**
```
HTTP/1.1 302 Found
Location: https://example.com/very/long/url/path?param1=value1&param2=value2
```

### Analytics
```http
GET /api/analytics/abc123
```

**Response:**
```json
{
    "total_clicks": 1250,
    "clicks_today": 45,
    "clicks_by_country": {
        "US": 450,
        "IN": 320,
        "UK": 180
    },
    "referrers": {
        "twitter.com": 300,
        "facebook.com": 250,
        "direct": 200
    },
    "created_at": "2023-12-01T00:00:00Z",
    "expires_at": "2024-01-30T00:00:00Z"
}
```

## High-Level Architecture

### Components
1. **API Gateway**: Rate limiting, authentication, request routing
2. **URL Shortening Service**: Generate short codes, validate URLs, store mappings
3. **Redirect Service**: Handle redirects, collect analytics
4. **Analytics Service**: Process and aggregate click data
5. **Database**: Store URL mappings and metadata
6. **Cache**: Redis for fast lookups
7. **Message Queue**: Async analytics processing
8. **CDN**: Global distribution

### Data Flow
1. **Shortening Flow**:
   - User submits URL → API Gateway → URL Service
   - Generate short code → Store in DB + Cache → Return short URL

2. **Redirect Flow**:
   - User clicks short URL → CDN/Redirect Service → Lookup in Cache
   - If cache miss → Lookup in DB → Redirect + Queue analytics

3. **Analytics Flow**:
   - Click events → Message Queue → Analytics Service → Store in DB

## Database Design

### URL Table
```sql
CREATE TABLE urls (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    original_url VARCHAR(2048) NOT NULL,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NULL,
    user_id BIGINT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    
    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);
```

### Click Analytics Table
```sql
CREATE TABLE url_clicks (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    url_id BIGINT NOT NULL,
    clicked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address VARCHAR(45),
    user_agent TEXT,
    referrer VARCHAR(2048),
    country_code CHAR(2),
    city VARCHAR(100),
    
    FOREIGN KEY (url_id) REFERENCES urls(id),
    INDEX idx_url_id (url_id),
    INDEX idx_clicked_at (clicked_at),
    INDEX idx_country_code (country_code)
);
```

### Daily Analytics Summary
```sql
CREATE TABLE daily_analytics (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    url_id BIGINT NOT NULL,
    date DATE NOT NULL,
    total_clicks INT DEFAULT 0,
    unique_clicks INT DEFAULT 0,
    top_country CHAR(2),
    top_referrer VARCHAR(2048),
    
    UNIQUE KEY unique_url_date (url_id, date),
    INDEX idx_date (date)
);
```

## Short Code Generation

### Requirements
- **Uniqueness**: No collisions
- **Length**: 6-8 characters for readability
- **Characters**: Alphanumeric (a-z, A-Z, 0-9)
- **Pronounceable**: Avoid confusing characters (0/O, 1/I/l)

### Strategies

#### 1. Random Generation
- Generate random strings
- Check uniqueness in database
- Pros: Simple, no patterns
- Cons: Collision probability increases with scale

#### 2. Sequential with Base Conversion
- Use auto-incrementing IDs
- Convert to base62: A-Za-z0-9
- Pros: No collisions, deterministic
- Cons: Predictable, not random-looking

#### 3. Hash-based
- Hash original URL
- Take first N characters
- Pros: Deterministic (same URL → same short code)
- Cons: Collisions, not unique per URL

#### 4. Hybrid Approach
- Use timestamp + random + checksum
- Ensure uniqueness through database constraints
- Handle collisions gracefully

**Recommended**: Base62 encoding of auto-incrementing IDs
- 62^6 ≈ 57 billion possible URLs
- 62^7 ≈ 3.5 trillion possible URLs
- Efficient and collision-free

## Scalability Considerations

### Database Sharding
- **Shard by short_code**: Distribute URLs across shards
- **Consistent Hashing**: Minimize data movement when adding shards
- **Read Replicas**: Handle read-heavy redirect traffic

### Caching Strategy
- **Multi-level Caching**:
  - **L1**: In-memory cache (Redis) - hot URLs
  - **L2**: CDN edge locations - geographic distribution
  - **L3**: Database with read replicas

- **Cache Invalidation**:
  - TTL-based expiration
  - Event-driven invalidation for updates
  - Write-through for new URLs

### Load Balancing
- **Global Load Balancer**: Route users to nearest region
- **Application Load Balancer**: Distribute traffic across instances
- **Database Load Balancer**: Route reads to replicas

## Performance Optimization

### Redirect Performance
- **Cache Hit Rate**: Target 95%+ for popular URLs
- **CDN Integration**: Serve redirects from edge locations
- **Database Indexing**: Optimize short_code lookups
- **Connection Pooling**: Reuse database connections

### Analytics Performance
- **Async Processing**: Don't slow down redirects
- **Batch Updates**: Aggregate analytics data
- **Data Partitioning**: Partition by date/time
- **Compression**: Compress old analytics data

## Security Considerations

### Input Validation
- **URL Validation**: Check for valid HTTP/HTTPS URLs
- **Length Limits**: Prevent extremely long URLs
- **Blacklist**: Block malicious domains
- **Rate Limiting**: Prevent abuse

### Access Control
- **Authentication**: API key or OAuth for shortening
- **Authorization**: User-specific URL management
- **Expiration**: Automatic cleanup of expired URLs

### Privacy & Compliance
- **GDPR Compliance**: User data handling
- **Analytics Anonymization**: IP address masking
- **Data Retention**: Configurable retention policies

## Monitoring & Observability

### Key Metrics
- **Availability**: Uptime, error rates
- **Performance**: Response times, throughput
- **Usage**: URLs created, redirects served
- **Cache Hit Rate**: Cache effectiveness
- **Error Rates**: 4xx/5xx responses

### Alerts
- High error rates
- Cache miss rate spikes
- Database connection issues
- Queue backlog

### Logging
- Request logs with correlation IDs
- Error logs with stack traces
- Analytics events
- Security events

## Deployment Architecture

### Regional Deployment
- **Primary Region**: US East (Virginia)
- **Secondary Regions**: US West, EU West, Asia Pacific
- **Global CDN**: CloudFront with edge locations

### Service Architecture
- **Microservices**: Independent deployment
- **Containerization**: Docker containers
- **Orchestration**: Kubernetes for scaling
- **Service Mesh**: Istio for traffic management

### Database Architecture
- **Primary Database**: MySQL with sharding
- **Read Replicas**: Regional replicas for low latency
- **Backup**: Cross-region backups
- **Disaster Recovery**: Multi-region failover

## Cost Optimization

### Compute Costs
- **Auto-scaling**: Scale based on traffic
- **Spot Instances**: Use for non-critical workloads
- **Regional Distribution**: Route traffic efficiently

### Storage Costs
- **Data Lifecycle**: Move old data to cheaper storage
- **Compression**: Compress analytics data
- **Archiving**: Archive expired URLs

### Network Costs
- **CDN Optimization**: Reduce origin requests
- **Regional Routing**: Minimize cross-region traffic

## Implementation Considerations

### Technology Stack
- **Backend**: Java Spring Boot (your expertise)
- **Database**: MySQL with Vitess for sharding
- **Cache**: Redis Cluster
- **Queue**: Apache Kafka
- **CDN**: CloudFront
- **Monitoring**: Prometheus + Grafana

### Development Practices
- **API-First Design**: Design APIs before implementation
- **TDD**: Test-driven development
- **CI/CD**: Automated testing and deployment
- **Documentation**: OpenAPI specifications

## Common Interview Follow-ups

### Q: How do you handle duplicate URLs?
**A**: Check if URL already exists before creating new short code. Return existing short code for duplicates.

### Q: How do you prevent URL abuse?
**A**: Rate limiting per IP/user, URL validation, malware scanning, expiration policies.

### Q: How do you ensure high availability?
**A**: Multi-region deployment, database replication, CDN for global distribution, circuit breakers.

### Q: How do you handle expired URLs?
**A**: Soft delete (mark inactive), background job cleanup, return 404 for expired URLs.

### Q: How do you scale the analytics system?
**A**: Async processing with message queues, batch updates, data partitioning by time, aggregation.

This URL shortener design demonstrates key system design principles: scalability, performance, reliability, and cost optimization. The architecture handles real-world requirements while teaching fundamental distributed system concepts.
