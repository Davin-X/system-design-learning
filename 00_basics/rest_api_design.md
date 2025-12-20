# REST API Design

REST (Representational State Transfer) is an architectural style for designing networked applications. RESTful APIs have become the standard for web services due to their simplicity, scalability, and stateless nature.

## REST Principles

### 1. Stateless
Each request from client to server must contain all necessary information to understand and process the request. No client context is stored on the server between requests.

**Benefits:**
- **Scalability**: Any server can handle any request
- **Reliability**: No session state to lose
- **Simplicity**: Easier to debug and test

**Implementation:**
```http
# Bad - relies on server session
POST /login
POST /transfer-money?amount=100  # Uses session

# Good - self-contained requests
POST /transfers
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "from_account": "12345",
  "to_account": "67890", 
  "amount": 100.00,
  "currency": "USD"
}
```

### 2. Client-Server Architecture
Clear separation between client and server concerns. Clients handle user interface and user state, servers handle data storage and business logic.

**Benefits:**
- **Separation of Concerns**: Independent evolution
- **Scalability**: Different teams can work on client/server
- **Portability**: Same server can serve different clients

### 3. Uniform Interface
All resources are accessed through a uniform and consistent interface.

**Key Constraints:**
- **Resource Identification**: Resources identified in requests (URIs)
- **Resource Manipulation through Representations**: Resources manipulated via representations
- **Self-descriptive Messages**: Each message contains enough information
- **Hypermedia as Engine of Application State (HATEOAS)**: Responses contain links to related resources

### 4. Cacheable
Responses must be cacheable when appropriate. Caching can reduce latency and network traffic.

**Cacheable Responses:**
```http
GET /products/123

Response:
Cache-Control: max-age=3600
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT
```

### 5. Layered System
Client cannot tell whether connected directly to server or through intermediaries.

**Benefits:**
- **Security**: Intermediaries can enforce security policies
- **Load Balancing**: Requests can be distributed
- **Caching**: Intermediaries can cache responses

## Resource Design

### Resource Naming
Resources should be nouns, not verbs. Use plural nouns for collections.

**Good Examples:**
```
GET /users         # Get all users
GET /users/123     # Get specific user
POST /users        # Create new user
PUT /users/123     # Update user
DELETE /users/123  # Delete user
```

**Bad Examples:**
```
GET /getUsers      # Verb in URL
POST /createUser   # Verb in URL
GET /users/getById?id=123  # Verb + query params
```

### Resource Relationships
Use sub-resources to represent relationships.

**Examples:**
```
GET /users/123/posts       # User's posts
GET /users/123/followers   # User's followers
GET /posts/456/comments    # Post's comments
GET /orders/789/items      # Order items
```

### Resource Representations
Resources can have multiple representations (JSON, XML, HTML).

**Content Negotiation:**
```http
GET /users/123
Accept: application/json

# or
Accept: application/xml
```

## HTTP Methods

### GET - Retrieve Resource
- **Safe**: No side effects
- **Idempotent**: Multiple calls = same result
- **Cacheable**: Can be cached
- **Use**: Retrieve data

```http
GET /users/123
GET /users?page=2&limit=50
GET /users?search=john&sort=name
```

### POST - Create Resource
- **Not Safe**: Has side effects
- **Not Idempotent**: Multiple calls create multiple resources
- **Not Cacheable**
- **Use**: Create new resources

```http
POST /users
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com"
}
```

### PUT - Update Entire Resource
- **Not Safe**: Has side effects
- **Idempotent**: Multiple calls = same result
- **Not Cacheable**
- **Use**: Replace entire resource

```http
PUT /users/123
Content-Type: application/json

{
  "id": 123,
  "name": "John Smith",
  "email": "johnsmith@example.com"
}
```

### PATCH - Partial Update
- **Not Safe**: Has side effects
- **Not necessarily Idempotent**: Depends on implementation
- **Not Cacheable**
- **Use**: Modify part of resource

```http
PATCH /users/123
Content-Type: application/json

{
  "email": "newemail@example.com"
}
```

### DELETE - Remove Resource
- **Not Safe**: Has side effects
- **Idempotent**: Multiple calls = same result (resource gone)
- **Not Cacheable**
- **Use**: Delete resource

```http
DELETE /users/123
```

## Status Codes

### Success Codes (2xx)
- **200 OK**: Request succeeded
- **201 Created**: Resource created successfully
- **202 Accepted**: Request accepted for processing
- **204 No Content**: Success but no response body

### Client Error Codes (4xx)
- **400 Bad Request**: Invalid request syntax
- **401 Unauthorized**: Authentication required
- **403 Forbidden**: Authentication succeeded but authorization failed
- **404 Not Found**: Resource doesn't exist
- **409 Conflict**: Request conflicts with current state
- **422 Unprocessable Entity**: Validation errors

### Server Error Codes (5xx)
- **500 Internal Server Error**: Unexpected server error
- **502 Bad Gateway**: Invalid response from upstream server
- **503 Service Unavailable**: Server temporarily unable to handle request

## API Versioning

### URL Path Versioning
Include version in URL path.

```http
/api/v1/users
/api/v2/users
```

**Pros:** Explicit, easy to understand
**Cons:** URL pollution, caching issues

### Query Parameter Versioning
Pass version as query parameter.

```http
/api/users?version=1
/api/users?version=2
```

**Pros:** Single URL, flexible
**Cons:** Not RESTful, caching issues

### Header Versioning
Use custom headers.

```http
GET /api/users
Accept: application/vnd.myapp.v1+json
```

**Pros:** Clean URLs, content negotiation
**Cons:** Less visible, harder to test

### Media Type Versioning
Use Accept header with vendor media types.

```http
GET /api/users
Accept: application/vnd.myapp.v1+json
```

**Pros:** Standards-compliant, flexible
**Cons:** Complex, less common

## Pagination

### Offset-Based Pagination
Use offset and limit parameters.

```http
GET /users?offset=100&limit=50
```

**Response:**
```json
{
  "data": [...],
  "pagination": {
    "offset": 100,
    "limit": 50,
    "total": 1000
  }
}
```

### Cursor-Based Pagination
Use cursor for next page.

```http
GET /users?cursor=eyJpZCI6MTIzfQ&limit=50
```

**Response:**
```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTczfQ",
    "has_more": true
  }
}
```

**Pros:** Consistent performance, handles deletions
**Cons:** Complex implementation, not intuitive

## Filtering, Sorting, Searching

### Filtering
```http
GET /users?status=active&role=admin
GET /products?price_min=10&price_max=100
GET /orders?created_after=2023-01-01
```

### Sorting
```http
GET /users?sort=name:asc,created_at:desc
GET /products?sort=price:asc
```

### Searching
```http
GET /users?search=john
GET /products?q=laptop&category=electronics
```

## Error Handling

### Error Response Format
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "details": {
      "field": "email",
      "value": "invalid-email",
      "reason": "must contain @ symbol"
    }
  }
}
```

### Common Error Patterns
- **Validation Errors**: 400 with field-specific details
- **Authentication Errors**: 401 with auth challenge
- **Authorization Errors**: 403 with permission details
- **Not Found Errors**: 404 with resource type
- **Rate Limit Errors**: 429 with retry-after header

## Security

### Authentication
- **JWT**: Stateless token-based auth
- **OAuth 2.0**: Delegated authorization
- **API Keys**: Simple key-based auth

### Authorization
- **Role-Based Access Control (RBAC)**
- **Attribute-Based Access Control (ABAC)**
- **Scope-based permissions**

### Security Headers
```http
X-API-Key: abc123def456
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
X-Request-ID: req-12345
```

## Rate Limiting

### Implementation
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1638360000
Retry-After: 60
```

### Strategies
- **User-based**: Per user limits
- **IP-based**: Per IP address limits
- **Endpoint-based**: Different limits per endpoint
- **Tier-based**: Different limits for different user tiers

## Documentation

### OpenAPI Specification
Standard for REST API documentation.

```yaml
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
paths:
  /users:
    get:
      summary: Get all users
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
      responses:
        '200':
          description: Success
```

### Tools
- **Swagger/OpenAPI**: Specification and tooling
- **Postman**: API testing and documentation
- **Insomnia**: API client and documentation
- **Redoc/Apiary**: Documentation generators

## REST API Design in Java (Spring Boot)

### Controller Design
```java
@RestController
@RequestMapping("/api/v1/users")
@Validated
public class UserController {
    
    @GetMapping
    public ResponseEntity<Page<UserDto>> getUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String search) {
        
        Page<UserDto> users = userService.getUsers(page, size, search);
        return ResponseEntity.ok(users);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        UserDto user = userService.getUserById(id);
        return ResponseEntity.ok(user);
    }
    
    @PostMapping
    public ResponseEntity<UserDto> createUser(@Valid @RequestBody CreateUserRequest request) {
        UserDto user = userService.createUser(request);
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(user.getId())
            .toUri();
        
        return ResponseEntity.created(location).body(user);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<UserDto> updateUser(
            @PathVariable Long id, 
            @Valid @RequestBody UpdateUserRequest request) {
        
        UserDto user = userService.updateUser(id, request);
        return ResponseEntity.ok(user);
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Global Exception Handling
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            "USER_NOT_FOUND", 
            ex.getMessage()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationErrors(MethodArgumentNotValidException ex) {
        Map<String, String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                FieldError::getDefaultMessage
            ));
        
        ErrorResponse error = new ErrorResponse("VALIDATION_ERROR", "Validation failed", errors);
        return ResponseEntity.badRequest().body(error);
    }
}
```

### Validation
```java
public class CreateUserRequest {
    
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
    private String name;
    
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;
    
    @NotNull(message = "Age is required")
    @Min(value = 18, message = "Must be at least 18 years old")
    @Max(value = 120, message = "Invalid age")
    private Integer age;
    
    // getters and setters
}
```

## Testing REST APIs

### Unit Testing Controllers
```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void shouldCreateUser() throws Exception {
        String userJson = """
            {
                "name": "John Doe",
                "email": "john@example.com",
                "age": 30
            }
            """;
        
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(userJson))
                .andExpect(status().isCreated())
                .andExpect(header().exists("Location"))
                .andExpect(jsonPath("$.name").value("John Doe"));
    }
}
```

### Integration Testing
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserApiIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void shouldGetUserById() {
        ResponseEntity<UserDto> response = restTemplate
            .withBasicAuth("user", "password")
            .getForEntity("/api/v1/users/1", UserDto.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isNotNull();
    }
}
```

## Common REST API Anti-Patterns

### 1. Using GET for State Changes
```http
# Bad
GET /users/123/activate

# Good  
PUT /users/123/status
Content-Type: application/json
{"status": "active"}
```

### 2. Ignoring HTTP Semantics
```http
# Bad - Using POST for retrieval
POST /search
{"query": "john"}

# Good
GET /users?search=john
```

### 3. Tight Coupling
```http
# Bad - Client knows internal structure
GET /users/123/internal-details

# Good - Expose domain concepts
GET /users/123/profile
```

### 4. Ignoring HATEOAS
```json
// Bad - No links
{
  "id": 123,
  "name": "John Doe"
}

// Good - With links
{
  "id": 123,
  "name": "John Doe",
  "_links": {
    "self": {"href": "/users/123"},
    "posts": {"href": "/users/123/posts"}
  }
}
```

REST API design requires careful consideration of resources, HTTP semantics, and client needs. Well-designed REST APIs are intuitive, consistent, and evolve gracefully over time.
