# HTTP and HTTPS Protocols

HTTP (HyperText Transfer Protocol) and HTTPS are the foundational protocols of the World Wide Web, enabling communication between clients and servers.

## HTTP Basics

### What is HTTP?
HTTP is an application-layer protocol for transmitting hypermedia documents (like HTML) over the internet. It follows a client-server model where:

- **Clients** (browsers, mobile apps) send requests
- **Servers** respond with resources or status information

### HTTP Characteristics
- **Stateless**: Each request is independent, no memory of previous requests
- **Request-Response**: Client initiates, server responds
- **Text-based**: Human-readable protocol
- **Extensible**: Headers allow custom metadata

### HTTP Versions

#### HTTP/1.0 (1996)
- One request per connection
- Connection closed after response
- Simple but inefficient for multiple resources

#### HTTP/1.1 (1997) - Most Common
- Persistent connections (keep-alive)
- Pipelining (multiple requests without waiting)
- Chunked transfer encoding
- Host header for virtual hosting
- Improved caching mechanisms

#### HTTP/2 (2015)
- Binary protocol (not text-based)
- Multiplexing (multiple requests/responses over single connection)
- Header compression (HPACK)
- Server push capability
- Prioritization of requests

#### HTTP/3 (2022)
- Uses QUIC instead of TCP
- Built-in encryption
- Faster connection establishment
- Better performance on poor networks

## HTTP Request Structure

### Request Line
```
METHOD URI HTTP/VERSION
GET /api/users/123 HTTP/1.1
```

**Common HTTP Methods:**
- **GET**: Retrieve resource
- **POST**: Create new resource
- **PUT**: Update entire resource
- **PATCH**: Partial update
- **DELETE**: Remove resource
- **HEAD**: Get headers only
- **OPTIONS**: Describe communication options

### Headers
Key-value pairs providing metadata:

```
Host: api.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36
Accept: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
Content-Length: 42
```

**Important Headers:**
- **Host**: Domain name (required in HTTP/1.1)
- **User-Agent**: Client information
- **Accept**: Acceptable response formats
- **Authorization**: Authentication credentials
- **Content-Type**: Request body format
- **Content-Length**: Body size in bytes

### Request Body
Optional data sent with request (mainly for POST/PUT/PATCH):

```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30
}
```

## HTTP Response Structure

### Status Line
```
HTTP/VERSION STATUS_CODE STATUS_TEXT
HTTP/1.1 200 OK
```

**Status Code Categories:**
- **1xx Informational**: Request received, continuing process
- **2xx Success**: Request successfully received, understood, accepted
- **3xx Redirection**: Further action needed to complete request
- **4xx Client Error**: Request contains bad syntax or cannot be fulfilled
- **5xx Server Error**: Server failed to fulfill valid request

**Common Status Codes:**
- **200 OK**: Request succeeded
- **201 Created**: Resource created
- **301 Moved Permanently**: Resource moved permanently
- **302 Found**: Resource moved temporarily
- **400 Bad Request**: Invalid request syntax
- **401 Unauthorized**: Authentication required
- **403 Forbidden**: Access denied
- **404 Not Found**: Resource not found
- **500 Internal Server Error**: Server error

### Response Headers
Similar to request headers:

```
Content-Type: application/json
Content-Length: 123
Cache-Control: max-age=3600
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT
```

**Caching Headers:**
- **Cache-Control**: Directives for caching behavior
- **ETag**: Entity tag for conditional requests
- **Last-Modified**: When resource was last modified

### Response Body
The actual content returned to client:

```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2023-12-01T10:00:00Z"
}
```

## HTTPS - Secure HTTP

### What is HTTPS?
HTTPS (HTTP Secure) is HTTP over TLS/SSL encryption. It provides:

- **Confidentiality**: Data encrypted in transit
- **Integrity**: Data cannot be modified without detection
- **Authentication**: Server identity verification

### TLS/SSL Handshake Process

1. **Client Hello**: Client sends supported cipher suites and TLS version
2. **Server Hello**: Server responds with chosen cipher suite and certificate
3. **Certificate Verification**: Client validates server certificate
4. **Key Exchange**: Client and server establish shared secret
5. **Finished**: Encrypted communication begins

### Certificate Authorities (CAs)
- Trusted third parties that issue digital certificates
- Verify website ownership and identity
- Browser trusts certificates from known CAs

### Types of Certificates
- **Domain Validated (DV)**: Basic validation, shows domain ownership
- **Organization Validated (OV)**: Company identity verified
- **Extended Validation (EV)**: Strict validation, highest trust level

## HTTP in System Design

### REST APIs
Representational State Transfer - architectural style for web services:

**Principles:**
- **Stateless**: Each request contains all necessary information
- **Client-Server**: Clear separation between client and server
- **Cacheable**: Responses can be cached
- **Uniform Interface**: Consistent way to interact with resources
- **Layered System**: Client doesn't know about intermediate layers

**RESTful Resource Design:**
```
/api/users          # Collection
/api/users/123      # Individual resource
/api/users/123/posts # Related resources
```

### API Versioning
Handling API evolution without breaking clients:

**Methods:**
- **URL Path**: `/api/v1/users`, `/api/v2/users`
- **Query Parameter**: `/api/users?version=1`
- **Header**: `Accept: application/vnd.api.v1+json`
- **Content Negotiation**: `Accept` header with custom media types

### Rate Limiting
Controlling request frequency to prevent abuse:

**Algorithms:**
- **Fixed Window**: Allow N requests per time window
- **Sliding Window**: Rolling time window
- **Token Bucket**: Tokens added at fixed rate, consumed by requests
- **Leaky Bucket**: Requests processed at constant rate

**Implementation:**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1638360000
```

### Content Negotiation
Server selects response format based on client preferences:

```
Accept: application/json, text/html;q=0.9
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
```

## HTTP Caching

### Browser Caching
- **Memory Cache**: Fastest, cleared when tab closes
- **Disk Cache**: Persistent across sessions
- **Service Worker Cache**: Programmatic control

### HTTP Cache Headers

**Cache-Control Directives:**
- `max-age=3600`: Cache for 1 hour
- `no-cache`: Revalidate before using
- `no-store`: Don't cache
- `public`: Cacheable by any cache
- `private`: Cacheable by browser only

**Conditional Requests:**
- `If-Modified-Since`: Check if resource changed since date
- `If-None-Match`: Check if ETag matches
- Server responds with `304 Not Modified` if unchanged

### CDN Caching
- **Edge Locations**: Cache content closer to users
- **Cache Invalidation**: Purge outdated content
- **Origin Shield**: Reduce load on origin server

## HTTP Performance Optimization

### Connection Optimization
- **Connection Reuse**: Keep-alive connections
- **Connection Pooling**: Reuse connections across requests
- **HTTP/2 Multiplexing**: Multiple requests over single connection

### Content Optimization
- **Compression**: gzip, brotli compression
- **Minification**: Remove whitespace from code
- **Image Optimization**: WebP format, responsive images
- **Resource Hints**: preload, prefetch, preconnect

### Caching Strategies
- **Browser Caching**: Cache static assets
- **CDN Caching**: Global content distribution
- **API Response Caching**: Cache expensive computations
- **Edge Computing**: Process near users

## HTTP Security

### Common Vulnerabilities
- **Injection Attacks**: SQL injection, XSS
- **Broken Authentication**: Weak session management
- **Sensitive Data Exposure**: Unencrypted data transmission
- **XML External Entities (XXE)**: External entity processing
- **Broken Access Control**: Improper authorization

### Security Headers
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
```

### HTTPS Best Practices
- **Redirect HTTP to HTTPS**: Automatic upgrade
- **HSTS**: Force HTTPS connections
- **Certificate Pinning**: Prevent MITM attacks
- **Perfect Forward Secrecy**: Unique keys per session

## HTTP in Java

### Spring Boot HTTP Handling

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(user);
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        User created = userService.create(user);
        return ResponseEntity
            .created(URI.create("/api/users/" + created.getId()))
            .body(created);
    }
}
```

### HTTP Client in Java

```java
@Service
public class ApiClient {
    private final WebClient webClient;
    
    public ApiClient() {
        this.webClient = WebClient.builder()
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
    
    public Mono<User> getUser(Long id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class);
    }
}
```

### HTTP Interceptors

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoggingInterceptor());
    }
}

@Component
public class LoggingInterceptor extends HandlerInterceptorAdapter {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler) {
        logger.info("Request: {} {}", request.getMethod(), request.getRequestURI());
        return true;
    }
}
```

## Monitoring HTTP Traffic

### Key Metrics
- **Request Rate**: Requests per second
- **Response Time**: Average, percentiles (p50, p95, p99)
- **Error Rate**: 4xx and 5xx responses
- **Throughput**: Data transferred per second

### Tools
- **Browser DevTools**: Network tab analysis
- **Postman**: API testing and debugging
- **Wireshark**: Packet-level analysis
- **cURL**: Command-line HTTP client
- **Application Metrics**: Prometheus, Grafana

## Common HTTP Interview Questions

### Q: Explain HTTP status codes
**A:** 
- 1xx: Informational (100 Continue)
- 2xx: Success (200 OK, 201 Created)
- 3xx: Redirection (301 Moved Permanently, 302 Found)
- 4xx: Client Error (400 Bad Request, 401 Unauthorized, 404 Not Found)
- 5xx: Server Error (500 Internal Server Error, 502 Bad Gateway)

### Q: Difference between PUT and PATCH?
**A:** PUT replaces entire resource, PATCH applies partial update. PUT is idempotent, PATCH may not be.

### Q: How does HTTPS work?
**A:** HTTPS uses TLS to encrypt HTTP traffic. Client and server perform handshake to establish shared secret, then encrypt all communication.

### Q: What is CORS?
**A:** Cross-Origin Resource Sharing - mechanism allowing restricted resources on web page from different domain. Uses headers like `Access-Control-Allow-Origin`.

HTTP and HTTPS form the backbone of modern web communication. Understanding these protocols is essential for designing scalable, secure, and performant web applications.
