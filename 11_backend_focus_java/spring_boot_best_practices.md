# Spring Boot Best Practices

Spring Boot has become the de facto standard for building Java microservices and web applications. This guide covers essential best practices for building production-ready, scalable, and maintainable Spring Boot applications, with implementation examples and performance optimization techniques.

## Application Configuration

### 1. Externalized Configuration

#### Profile-Based Configuration
```java
@Configuration
@PropertySource("classpath:application.properties")
public class AppConfig {
    
    @Value("${app.name}")
    private String appName;
    
    @Value("${app.version}")
    private String appVersion;
    
    @Bean
    @Profile("development")
    public DataSource developmentDataSource() {
        return createHikariDataSource("dev");
    }
    
    @Bean
    @Profile("production")
    public DataSource productionDataSource() {
        return createHikariDataSource("prod");
    }
    
    @Bean
    @Profile("test")
    public DataSource testDataSource() {
        return createHikariDataSource("test");
    }
    
    private HikariDataSource createHikariDataSource(String env) {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(getProperty("spring.datasource.url." + env));
        dataSource.setUsername(getProperty("spring.datasource.username." + env));
        dataSource.setPassword(getProperty("spring.datasource.password." + env));
        dataSource.setMaximumPoolSize(getIntProperty("spring.datasource.hikari.maximum-pool-size", 10));
        dataSource.setMinimumIdle(getIntProperty("spring.datasource.hikari.minimum-idle", 5));
        return dataSource;
    }
}
```

#### Configuration Properties Classes
```java
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {
    
    @NotBlank
    private String name;
    
    @NotBlank
    private String version;
    
    @Valid
    private Security security = new Security();
    
    @Valid
    private Cache cache = new Cache();
    
    @Valid
    private Metrics metrics = new Metrics();
    
    // Getters and setters
    
    public static class Security {
        @NotBlank
        private String jwtSecret;
        
        @Min(300)
        private int jwtExpiration = 3600; // seconds
        
        private boolean enableCsrf = true;
        
        // Getters and setters
    }
    
    public static class Cache {
        private boolean enabled = true;
        
        @Min(60)
        private int defaultTtl = 3600; // seconds
        
        @Min(10)
        private int maxEntries = 1000;
        
        // Getters and setters
    }
    
    public static class Metrics {
        private boolean enabled = true;
        private String endpoint = "/actuator/metrics";
        private List<String> includeMetrics = Arrays.asList("jvm.*", "http.*", "system.*");
        
        // Getters and setters
    }
}
```

### 2. Environment-Specific Properties

#### application.yml with Profiles
```yaml
# Common configuration
spring:
  application:
    name: user-service
  profiles:
    active: development

logging:
  level:
    root: INFO
    com.example: DEBUG

app:
  name: User Service
  version: 1.0.0

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized

---

# Development profile
spring:
  config:
    activate:
      on-profile: development
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
  h2:
    console:
      enabled: true

logging:
  level:
    org.springframework.security: DEBUG
    org.hibernate.SQL: DEBUG

app:
  cache:
    enabled: false

---

# Production profile
spring:
  config:
    activate:
      on-profile: production
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

logging:
  level:
    root: WARN
    com.example: INFO

app:
  cache:
    enabled: true
    default-ttl: 7200  # 2 hours
    max-entries: 5000
```

## Dependency Management

### 1. BOM (Bill of Materials)

#### Custom BOM for Dependency Management
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.example</groupId>
    <artifactId>company-bom</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    
    <name>Company BOM</name>
    <description>Bill of Materials for company projects</description>
    
    <properties>
        <java.version>17</java.version>
        <spring-boot.version>3.1.0</spring-boot.version>
        <junit.version>5.10.0</junit.version>
        <mockito.version>5.2.0</mockito.version>
    </properties>
    
    <dependencyManagement>
        <dependencies>
            <!-- Spring Boot BOM -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            
            <!-- Company internal dependencies -->
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>common-lib</artifactId>
                <version>2.1.0</version>
            </dependency>
            
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>security-lib</artifactId>
                <version>1.5.0</version>
            </dependency>
            
            <!-- Test dependencies -->
            <dependency>
                <groupId>org.junit.jupiter</groupId>
                <artifactId>junit-jupiter</artifactId>
                <version>${junit.version}</version>
                <scope>test</scope>
            </dependency>
            
            <dependency>
                <groupId>org.mockito</groupId>
                <artifactId>mockito-core</artifactId>
                <version>${mockito.version}</version>
                <scope>test</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring-boot.version}</version>
                    <executions>
                        <execution>
                            <goals>
                                <goal>repackage</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
                
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.11.0</version>
                    <configuration>
                        <source>${java.version}</source>
                        <target>${java.version}</target>
                        <encoding>UTF-8</encoding>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

#### Service POM using BOM
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>com.example</groupId>
        <artifactId>company-parent</artifactId>
        <version>1.0.0</version>
    </parent>
    
    <artifactId>user-service</artifactId>
    <version>1.0.0</version>
    
    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
        <!-- Company internal dependencies -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>common-lib</artifactId>
        </dependency>
        
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>security-lib</artifactId>
        </dependency>
        
        <!-- Database -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
        
        <!-- Test dependencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

## Layered Architecture

### 1. Controller Layer

#### REST Controller Best Practices
```java
@RestController
@RequestMapping("/api/v1/users")
@Validated
@Tag(name = "User Management", description = "User management API")
public class UserController {
    
    private final UserService userService;
    private final UserMapper userMapper;
    
    public UserController(UserService userService, UserMapper userMapper) {
        this.userService = userService;
        this.userMapper = userMapper;
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(summary = "Create a new user", description = "Creates a new user account")
    public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
        User user = userService.createUser(request);
        return userMapper.toResponse(user);
    }
    
    @GetMapping("/{userId}")
    @Operation(summary = "Get user by ID", description = "Retrieves user information by user ID")
    public UserResponse getUser(@PathVariable @Positive Long userId) {
        User user = userService.getUserById(userId);
        return userMapper.toResponse(user);
    }
    
    @GetMapping
    @Operation(summary = "Search users", description = "Search users with pagination")
    public Page<UserResponse> searchUsers(
            @RequestParam(required = false) String query,
            @RequestParam(defaultValue = "0") @Min(0) int page,
            @RequestParam(defaultValue = "20") @Min(1) @Max(100) int size,
            @RequestParam(defaultValue = "username") String sortBy,
            @RequestParam(defaultValue = "asc") String sortDirection) {
        
        Pageable pageable = PageRequest.of(page, size, 
            Sort.by(Sort.Direction.fromString(sortDirection), sortBy));
        
        Page<User> users = userService.searchUsers(query, pageable);
        return users.map(userMapper::toResponse);
    }
    
    @PutMapping("/{userId}")
    @Operation(summary = "Update user", description = "Updates user information")
    public UserResponse updateUser(@PathVariable @Positive Long userId, 
                                  @Valid @RequestBody UpdateUserRequest request) {
        User user = userService.updateUser(userId, request);
        return userMapper.toResponse(user);
    }
    
    @DeleteMapping("/{userId}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    @Operation(summary = "Delete user", description = "Deletes a user account")
    public void deleteUser(@PathVariable @Positive Long userId) {
        userService.deleteUser(userId);
    }
    
    @PostMapping("/{userId}/follow/{targetUserId}")
    @Operation(summary = "Follow user", description = "Follow another user")
    public void followUser(@PathVariable @Positive Long userId, 
                          @PathVariable @Positive Long targetUserId) {
        userService.followUser(userId, targetUserId);
    }
    
    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleUserNotFound(UserNotFoundException e) {
        return new ErrorResponse("USER_NOT_FOUND", e.getMessage());
    }
    
    @ExceptionHandler(ValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidationError(ValidationException e) {
        return new ErrorResponse("VALIDATION_ERROR", e.getMessage());
    }
}
```

### 2. Service Layer

#### Business Logic Implementation
```java
@Service
@Transactional(readOnly = true)
public class UserServiceImpl implements UserService {
    
    private final UserRepository userRepository;
    private final FollowRepository followRepository;
    private final PasswordEncoder passwordEncoder;
    private final EventPublisher eventPublisher;
    private final CacheManager cacheManager;
    
    public UserServiceImpl(UserRepository userRepository,
                          FollowRepository followRepository,
                          PasswordEncoder passwordEncoder,
                          EventPublisher eventPublisher,
                          CacheManager cacheManager) {
        this.userRepository = userRepository;
        this.followRepository = followRepository;
        this.passwordEncoder = passwordEncoder;
        this.eventPublisher = eventPublisher;
        this.cacheManager = cacheManager;
    }
    
    @Override
    @Transactional
    public User createUser(CreateUserRequest request) {
        // Validate username uniqueness
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new UsernameAlreadyExistsException(request.getUsername());
        }
        
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new EmailAlreadyExistsException(request.getEmail());
        }
        
        // Create user
        User user = new User();
        user.setUsername(request.getUsername());
        user.setEmail(request.getEmail());
        user.setFullName(request.getFullName());
        user.setPasswordHash(passwordEncoder.encode(request.getPassword()));
        user.setStatus(UserStatus.ACTIVE);
        user.setCreatedAt(Instant.now());
        user.setUpdatedAt(Instant.now());
        
        User savedUser = userRepository.save(user);
        
        // Publish domain event
        eventPublisher.publish(new UserCreatedEvent(savedUser.getId(), 
                                                   savedUser.getUsername(), 
                                                   savedUser.getEmail()));
        
        // Cache user profile
        cacheUserProfile(savedUser);
        
        return savedUser;
    }
    
    @Override
    public User getUserById(Long userId) {
        // Try cache first
        Cache cache = cacheManager.getCache("userProfiles");
        String cacheKey = "user:" + userId;
        
        User cachedUser = cache.get(cacheKey, User.class);
        if (cachedUser != null) {
            return cachedUser;
        }
        
        // Load from database
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        // Cache the result
        cache.put(cacheKey, user);
        
        return user;
    }
    
    @Override
    public Page<User> searchUsers(String query, Pageable pageable) {
        if (query == null || query.trim().isEmpty()) {
            return userRepository.findAll(pageable);
        }
        
        return userRepository.searchByUsernameOrFullName(query, pageable);
    }
    
    @Override
    @Transactional
    public User updateUser(Long userId, UpdateUserRequest request) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        // Update fields
        if (request.getFullName() != null) {
            user.setFullName(request.getFullName());
        }
        
        if (request.getBio() != null) {
            user.setBio(request.getBio());
        }
        
        user.setUpdatedAt(Instant.now());
        
        User savedUser = userRepository.save(user);
        
        // Invalidate cache
        Cache cache = cacheManager.getCache("userProfiles");
        cache.evict("user:" + userId);
        
        return savedUser;
    }
    
    @Override
    @Transactional
    public void deleteUser(Long userId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        // Soft delete
        user.setStatus(UserStatus.DELETED);
        user.setUpdatedAt(Instant.now());
        
        userRepository.save(user);
        
        // Publish event
        eventPublisher.publish(new UserDeletedEvent(userId));
        
        // Invalidate cache
        Cache cache = cacheManager.getCache("userProfiles");
        cache.evict("user:" + userId);
    }
    
    @Override
    @Transactional
    public void followUser(Long followerId, Long targetUserId) {
        // Validate users exist
        User follower = getUserById(followerId);
        User target = getUserById(targetUserId);
        
        // Check not following self
        if (followerId.equals(targetUserId)) {
            throw new InvalidFollowException("Cannot follow yourself");
        }
        
        // Check not already following
        if (followRepository.existsByFollowerIdAndFolloweeId(followerId, targetUserId)) {
            throw new AlreadyFollowingException(followerId, targetUserId);
        }
        
        // Create follow relationship
        Follow follow = new Follow();
        follow.setFollowerId(followerId);
        follow.setFolloweeId(targetUserId);
        follow.setCreatedAt(Instant.now());
        
        followRepository.save(follow);
        
        // Update follower counts (cache invalidation will handle this)
        invalidateUserProfileCache(followerId);
        invalidateUserProfileCache(targetUserId);
        
        // Publish event
        eventPublisher.publish(new UserFollowedEvent(followerId, targetUserId));
    }
    
    private void cacheUserProfile(User user) {
        Cache cache = cacheManager.getCache("userProfiles");
        cache.put("user:" + user.getId(), user);
    }
    
    private void invalidateUserProfileCache(Long userId) {
        Cache cache = cacheManager.getCache("userProfiles");
        cache.evict("user:" + userId);
    }
}
```

### 3. Repository Layer

#### JPA Repository with Custom Queries
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    boolean existsByUsername(String username);
    
    boolean existsByEmail(String email);
    
    Optional<User> findByUsername(String username);
    
    Optional<User> findByEmail(String email);
    
    @Query("SELECT u FROM User u WHERE u.status = :status")
    List<User> findByStatus(@Param("status") UserStatus status);
    
    @Query("SELECT u FROM User u WHERE u.createdAt >= :since")
    List<User> findUsersCreatedAfter(@Param("since") Instant since);
    
    @Query("""
        SELECT u FROM User u 
        WHERE (LOWER(u.username) LIKE LOWER(CONCAT('%', :query, '%')) 
               OR LOWER(u.fullName) LIKE LOWER(CONCAT('%', :query, '%')))
        AND u.status = :status
        """)
    Page<User> searchByUsernameOrFullName(@Param("query") String query, 
                                         @Param("status") UserStatus status, 
                                         Pageable pageable);
    
    @Query("SELECT COUNT(f) FROM Follow f WHERE f.followerId = :userId")
    Long countFollowing(@Param("userId") Long userId);
    
    @Query("SELECT COUNT(f) FROM Follow f WHERE f.followeeId = :userId")
    Long countFollowers(@Param("userId") Long userId);
    
    @Modifying
    @Query("UPDATE User u SET u.lastLoginAt = :lastLoginAt WHERE u.id = :userId")
    void updateLastLogin(@Param("userId") Long userId, @Param("lastLoginAt") Instant lastLoginAt);
}

@Repository
public interface FollowRepository extends JpaRepository<Follow, Long> {
    
    boolean existsByFollowerIdAndFolloweeId(Long followerId, Long followeeId);
    
    @Query("SELECT f FROM Follow f WHERE f.followerId = :followerId ORDER BY f.createdAt DESC")
    List<Follow> findByFollowerId(@Param("followerId") Long followerId);
    
    @Query("SELECT f FROM Follow f WHERE f.followeeId = :followeeId ORDER BY f.createdAt DESC")
    List<Follow> findByFolloweeId(@Param("followeeId") Long followeeId);
    
    @Query("SELECT f.followeeId FROM Follow f WHERE f.followerId = :followerId")
    List<Long> findFollowingIdsByUserId(@Param("followerId") Long followerId);
    
    @Query("SELECT f.followerId FROM Follow f WHERE f.followeeId = :followeeId")
    List<Long> findFollowerIdsByUserId(@Param("followeeId") Long followeeId);
    
    @Query("SELECT COUNT(f) FROM Follow f WHERE f.followeeId = :userId")
    Long countByFolloweeId(@Param("userId") Long userId);
    
    @Query("SELECT COUNT(f) FROM Follow f WHERE f.followerId = :userId")
    Long countByFollowerId(@Param("userId") Long userId);
    
    @Modifying
    @Query("DELETE FROM Follow f WHERE f.followerId = :followerId AND f.followeeId = :followeeId")
    void deleteByFollowerIdAndFolloweeId(@Param("followerId") Long followerId, 
                                        @Param("followeeId") Long followeeId);
}
```

## Error Handling and Validation

### 1. Global Exception Handler
```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleUserNotFound(UserNotFoundException e) {
        log.warn("User not found: {}", e.getMessage());
        return new ErrorResponse("USER_NOT_FOUND", e.getMessage());
    }
    
    @ExceptionHandler(ValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidationException(ValidationException e) {
        log.warn("Validation error: {}", e.getMessage());
        return new ErrorResponse("VALIDATION_ERROR", e.getMessage());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ValidationErrorResponse handleValidationErrors(MethodArgumentNotValidException e) {
        log.warn("Validation errors: {}", e.getMessage());
        
        Map<String, String> errors = new HashMap<>();
        e.getBindingResult().getFieldErrors().forEach(error -> 
            errors.put(error.getField(), error.getDefaultMessage()));
        
        return new ValidationErrorResponse("VALIDATION_ERROR", "Invalid request data", errors);
    }
    
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ValidationErrorResponse handleConstraintViolations(ConstraintViolationException e) {
        log.warn("Constraint violations: {}", e.getMessage());
        
        Map<String, String> errors = new HashMap<>();
        e.getConstraintViolations().forEach(violation -> 
            errors.put(violation.getPropertyPath().toString(), violation.getMessage()));
        
        return new ValidationErrorResponse("VALIDATION_ERROR", "Constraint violations", errors);
    }
    
    @ExceptionHandler(DataIntegrityViolationException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleDataIntegrityViolation(DataIntegrityViolationException e) {
        log.error("Data integrity violation", e);
        return new ErrorResponse("DATA_INTEGRITY_ERROR", "Data integrity constraint violated");
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGenericException(Exception e) {
        log.error("Unexpected error", e);
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
    }
    
    @ExceptionHandler(HttpMessageNotReadableException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleMalformedJson(HttpMessageNotReadableException e) {
        log.warn("Malformed JSON request", e);
        return new ErrorResponse("MALFORMED_REQUEST", "Request body is not valid JSON");
    }
    
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    @ResponseStatus(HttpStatus.METHOD_NOT_ALLOWED)
    public ErrorResponse handleMethodNotAllowed(HttpRequestMethodNotSupportedException e) {
        log.warn("Method not allowed: {}", e.getMessage());
        return new ErrorResponse("METHOD_NOT_ALLOWED", e.getMessage());
    }
    
    @ExceptionHandler(MissingServletRequestParameterException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleMissingParameter(MissingServletRequestParameterException e) {
        log.warn("Missing required parameter: {}", e.getParameterName());
        return new ErrorResponse("MISSING_PARAMETER", 
                               String.format("Required parameter '%s' is missing", e.getParameterName()));
    }
    
    // Response classes
    public static class ErrorResponse {
        private final String errorCode;
        private final String message;
        private final Instant timestamp;
        
        public ErrorResponse(String errorCode, String message) {
            this.errorCode = errorCode;
            this.message = message;
            this.timestamp = Instant.now();
        }
        
        // Getters
    }
    
    public static class ValidationErrorResponse extends ErrorResponse {
        private final Map<String, String> fieldErrors;
        
        public ValidationErrorResponse(String errorCode, String message, Map<String, String> fieldErrors) {
            super(errorCode, message);
            this.fieldErrors = fieldErrors;
        }
        
        // Getters
    }
}
```

### 2. Custom Validation Annotations
```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = StrongPasswordValidator.class)
@Documented
public @interface StrongPassword {
    
    String message() default "Password must be at least 8 characters long and contain at least one uppercase letter, one lowercase letter, one number, and one special character";
    
    Class<?>[] groups() default {};
    
    Class<? extends Payload>[] payload() default {};
}

public class StrongPasswordValidator implements ConstraintValidator<StrongPassword, String> {
    
    private static final Pattern PASSWORD_PATTERN = 
        Pattern.compile("^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$");
    
    @Override
    public void initialize(StrongPassword constraintAnnotation) {
        // No initialization needed
    }
    
    @Override
    public boolean isValid(String password, ConstraintValidatorContext context) {
        if (password == null) {
            return false;
        }
        
        return PASSWORD_PATTERN.matcher(password).matches();
    }
}

@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UsernameValidator.class)
@Documented
public @interface ValidUsername {
    
    String message() default "Username must be 3-20 characters long and contain only letters, numbers, and underscores";
    
    Class<?>[] groups() default {};
    
    Class<? extends Payload>[] payload() default {};
}

public class UsernameValidator implements ConstraintValidator<ValidUsername, String> {
    
    private static final Pattern USERNAME_PATTERN = Pattern.compile("^[a-zA-Z0-9_]{3,20}$");
    
    @Override
    public void initialize(ValidUsername constraintAnnotation) {
        // No initialization needed
    }
    
    @Override
    public boolean isValid(String username, ConstraintValidatorContext context) {
        if (username == null) {
            return false;
        }
        
        return USERNAME_PATTERN.matcher(username).matches();
    }
}

// Usage in DTOs
public class CreateUserRequest {
    
    @NotBlank(message = "Username is required")
    @ValidUsername
    private String username;
    
    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    private String email;
    
    @NotBlank(message = "Full name is required")
    @Size(min = 2, max = 100, message = "Full name must be between 2 and 100 characters")
    private String fullName;
    
    @NotBlank(message = "Password is required")
    @StrongPassword
    private String password;
    
    // Getters and setters
}
```

## Performance Optimization

### 1. Caching Strategy
```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheManagerBuilder builder = RedisCacheManager.builder(redisConnectionFactory);
        
        // Configure different caches with different TTLs
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        
        // User profiles - longer TTL
        cacheConfigurations.put("userProfiles", 
            RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(2))
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new Jackson2JsonRedisSerializer<>(User.class))));
        
        // Search results - shorter TTL
        cacheConfigurations.put("searchResults",
            RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10)));
        
        // Session data - medium TTL
        cacheConfigurations.put("sessions",
            RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(24)));
        
        return builder
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(30)))
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

@Service
public class CacheableUserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Cacheable(value = "userProfiles", key = "'user:' + #userId")
    public User getUserById(Long userId) {
        return userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
    }
    
    @Cacheable(value = "searchResults", key = "'users:search:' + #query + ':' + #pageable.pageNumber + ':' + #pageable.pageSize")
    public Page<User> searchUsers(String query, Pageable pageable) {
        return userRepository.searchByUsernameOrFullName(query, UserStatus.ACTIVE, pageable);
    }
    
    @CacheEvict(value = "userProfiles", key = "'user:' + #userId")
    @Transactional
    public User updateUser(Long userId, UpdateUserRequest request) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        // Update user logic
        user.setFullName(request.getFullName());
        user.setUpdatedAt(Instant.now());
        
        return userRepository.save(user);
    }
    
    @CacheEvict(value = "userProfiles", key = "'user:' + #userId")
    @Transactional
    public void deleteUser(Long userId) {
        userRepository.deleteById(userId);
    }
    
    @Caching(evict = {
        @CacheEvict(value = "userProfiles", key = "'user:' + #followerId"),
        @CacheEvict(value = "userProfiles", key = "'user:' + #targetUserId")
    })
    @Transactional
    public void followUser(Long followerId, Long targetUserId) {
        // Follow logic that updates follower/following counts
    }
}
```

### 2. Connection Pool Optimization
```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @ConfigurationProperties("spring.datasource.hikari")
    public HikariDataSource dataSource() {
        HikariDataSource dataSource = DataSourceBuilder.create()
            .type(HikariDataSource.class)
            .build();
        
        // Optimize for production workloads
        dataSource.setMaximumPoolSize(50);
        dataSource.setMinimumIdle(10);
        dataSource.setConnectionTimeout(30000); // 30 seconds
        dataSource.setIdleTimeout(600000); // 10 minutes
        dataSource.setMaxLifetime(1800000); // 30 minutes
        dataSource.setLeakDetectionThreshold(60000); // 1 minute
        
        // Enable metrics
        dataSource.setMetricRegistry(Metrics.globalRegistry);
        dataSource.setPoolName("HikariPool-user-service");
        
        return dataSource;
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
        
        // Enable fetch size optimization for large result sets
        jdbcTemplate.setFetchSize(100);
        
        // Enable query timeout
        jdbcTemplate.setQueryTimeout(30); // 30 seconds
        
        return jdbcTemplate;
    }
    
    @Bean
    public JpaVendorAdapter jpaVendorAdapter() {
        HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        adapter.setShowSql(false);
        adapter.setGenerateDdl(false);
        
        return adapter;
    }
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource, JpaVendorAdapter jpaVendorAdapter) {
        
        LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
        factory.setDataSource(dataSource);
        factory.setJpaVendorAdapter(jpaVendorAdapter);
        factory.setPackagesToScan("com.example.user.domain");
        
        Properties jpaProperties = new Properties();
        jpaProperties.setProperty("hibernate.dialect", "org.hibernate.dialect.MySQL8Dialect");
        jpaProperties.setProperty("hibernate.jdbc.batch_size", "25");
        jpaProperties.setProperty("hibernate.order_inserts", "true");
        jpaProperties.setProperty("hibernate.order_updates", "true");
        jpaProperties.setProperty("hibernate.jdbc.batch_versioned_data", "true");
        jpaProperties.setProperty("hibernate.connection.provider_disables_autocommit", "true");
        
        // Second level cache
        jpaProperties.setProperty("hibernate.cache.use_second_level_cache", "true");
        jpaProperties.setProperty("hibernate.cache.region.factory_class", 
            "org.hibernate.cache.jcache.JCacheRegionFactory");
        jpaProperties.setProperty("hibernate.javax.cache.provider", 
            "com.github.benmanes.caffeine.jcache.configuration.CaffeineConfiguration");
        
        factory.setJpaProperties(jpaProperties);
        
        return factory;
    }
}
```

### 3. Asynchronous Processing
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        
        return executor;
    }
}

@Service
public class AsyncUserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EmailService emailService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Async
    @Transactional
    public CompletableFuture<User> createUserAsync(CreateUserRequest request) {
        try {
            User user = createUser(request);
            
            // Send welcome email asynchronously
            sendWelcomeEmailAsync(user);
            
            // Send notifications asynchronously
            sendUserCreatedNotificationsAsync(user);
            
            return CompletableFuture.completedFuture(user);
            
        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }
    
    @Async
    public void sendWelcomeEmailAsync(User user) {
        try {
            emailService.sendWelcomeEmail(user.getEmail(), user.getFullName());
        } catch (Exception e) {
            // Log error but don't fail the user creation
            log.error("Failed to send welcome email to user {}", user.getId(), e);
        }
    }
    
    @Async
    public void sendUserCreatedNotificationsAsync(User user) {
        try {
            // Send internal notifications
            notificationService.notifyAdminNewUser(user);
            
            // Update analytics
            analyticsService.recordNewUser(user);
            
        } catch (Exception e) {
            log.error("Failed to send user created notifications for user {}", user.getId(), e);
        }
    }
    
    @Async
    public CompletableFuture<List<User>> getUsersBatchAsync(List<Long> userIds) {
        return CompletableFuture.supplyAsync(() -> 
            userRepository.findAllById(userIds), taskExecutor);
    }
    
    @Async
    public void updateUserStatsAsync() {
        // Heavy computation done asynchronously
        List<User> users = userRepository.findAll();
        
        for (User user : users) {
            long followerCount = followRepository.countByFolloweeId(user.getId());
            long followingCount = followRepository.countByFollowerId(user.getId());
            
            userRepository.updateFollowerStats(user.getId(), followerCount, followingCount);
        }
    }
}
```

## Security Best Practices

### 1. Authentication and Authorization
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    @Autowired
    private JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    
    @Autowired
    private JwtRequestFilter jwtRequestFilter;
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/actuator/info").permitAll()
                .anyRequest().authenticated()
            )
            .exceptionHandling().authenticationEntryPoint(jwtAuthenticationEntryPoint)
            .and()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
        
        http.addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration authenticationConfiguration) throws Exception {
        return authenticationConfiguration.getAuthenticationManager();
    }
}

@Service
public class JwtUserDetailsService implements UserDetailsService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
        
        return new org.springframework.security.core.userdetails.User(
            user.getUsername(),
            user.getPasswordHash(),
            user.getStatus() == UserStatus.ACTIVE,
            true, true, true,
            getAuthorities(user)
        );
    }
    
    private Collection<? extends GrantedAuthority> getAuthorities(User user) {
        List<GrantedAuthority> authorities = new ArrayList<>();
        
        if (user.getRole() != null) {
            authorities.add(new SimpleGrantedAuthority("ROLE_" + user.getRole().name()));
        }
        
        return authorities;
    }
}

@Service
public class JwtTokenService {
    
    @Value("${jwt.secret}")
    private String jwtSecret;
    
    @Value("${jwt.expiration}")
    private int jwtExpiration;
    
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("sub", userDetails.getUsername());
        claims.put("iat", new Date().getTime());
        claims.put("exp", new Date(System.currentTimeMillis() + jwtExpiration * 1000).getTime());
        
        return Jwts.builder()
            .setClaims(claims)
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }
    
    public String getUsernameFromToken(String token) {
        return getClaimsFromToken(token).getSubject();
    }
    
    public boolean validateToken(String token, UserDetails userDetails) {
        String username = getUsernameFromToken(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }
    
    private Claims getClaimsFromToken(String token) {
        return Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(token).getBody();
    }
    
    private boolean isTokenExpired(String token) {
        Date expiration = getClaimsFromToken(token).getExpiration();
        return expiration.before(new Date());
    }
}

@RestController
@RequestMapping("/api/auth")
public class AuthenticationController {
    
    @Autowired
    private AuthenticationManager authenticationManager;
    
    @Autowired
    private JwtTokenService jwtTokenService;
    
    @Autowired
    private JwtUserDetailsService userDetailsService;
    
    @PostMapping("/login")
    public ResponseEntity<JwtResponse> login(@RequestBody LoginRequest request) {
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.getUsername(), request.getPassword()));
        
        SecurityContextHolder.getContext().setAuthentication(authentication);
        
        UserDetails userDetails = userDetailsService.loadUserByUsername(request.getUsername());
        String token = jwtTokenService.generateToken(userDetails);
        
        return ResponseEntity.ok(new JwtResponse(token));
    }
    
    @PostMapping("/refresh")
    public ResponseEntity<JwtResponse> refreshToken(HttpServletRequest request) {
        String oldToken = extractToken(request);
        
        if (oldToken != null && jwtTokenService.validateToken(oldToken, 
            userDetailsService.loadUserByUsername(jwtTokenService.getUsernameFromToken(oldToken)))) {
            
            String username = jwtTokenService.getUsernameFromToken(oldToken);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            String newToken = jwtTokenService.generateToken(userDetails);
            
            return ResponseEntity.ok(new JwtResponse(newToken));
        }
        
        throw new InvalidTokenException("Invalid or expired token");
    }
    
    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            return header.substring(7);
        }
        return null;
    }
}

@Component
public class JwtRequestFilter extends OncePerRequestFilter {
    
    @Autowired
    private JwtUserDetailsService userDetailsService;
    
    @Autowired
    private JwtTokenService jwtTokenService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, 
                                  FilterChain chain) throws ServletException, IOException {
        
        String token = extractToken(request);
        
        if (token != null) {
            String username = jwtTokenService.getUsernameFromToken(token);
            
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                
                if (jwtTokenService.validateToken(token, userDetails)) {
                    UsernamePasswordAuthenticationToken authentication = 
                        new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                    
                    SecurityContextHolder.getContext().setAuthentication(authentication);
                }
            }
        }
        
        chain.doFilter(request, response);
    }
    
    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            return header.substring(7);
        }
        return null;
    }
}
```

### 2. Input Validation and Sanitization
```java
@Service
public class InputSanitizationService {
    
    private static final Pattern SQL_INJECTION_PATTERN = 
        Pattern.compile(".*(\\bUNION\\b|\\bSELECT\\b|\\bINSERT\\b|\\bUPDATE\\b|\\bDELETE\\b|\\bDROP\\b|\\bCREATE\\b|\\bALTER\\b).*", 
                       Pattern.CASE_INSENSITIVE);
    
    private static final Pattern XSS_PATTERN = 
        Pattern.compile(".*(<script|javascript:|vbscript:|onload=|onerror=|onclick=).*", 
                       Pattern.CASE_INSENSITIVE);
    
    public String sanitizeInput(String input) {
        if (input == null) {
            return null;
        }
        
        // Trim whitespace
        String sanitized = input.trim();
        
        // Check for SQL injection patterns
        if (SQL_INJECTION_PATTERN.matcher(sanitized).matches()) {
            throw new SecurityException("Potential SQL injection detected");
        }
        
        // Check for XSS patterns
        if (XSS_PATTERN.matcher(sanitized).matches()) {
            throw new SecurityException("Potential XSS attack detected");
        }
        
        // Escape HTML characters
        sanitized = StringEscapeUtils.escapeHtml4(sanitized);
        
        // Limit length
        if (sanitized.length() > 10000) {
            throw new ValidationException("Input too long");
        }
        
        return sanitized;
    }
    
    public String sanitizeHtml(String html) {
        if (html == null) {
            return null;
        }
        
        // Use OWASP AntiSamy or similar for HTML sanitization
        Policy policy = Policy.getInstance("antisamy-slashdot.xml");
        AntiSamy antiSamy = new AntiSamy();
        CleanResults cleanResults = antiSamy.scan(html, policy);
        
        return cleanResults.getCleanHTML();
    }
    
    public String sanitizeFilename(String filename) {
        if (filename == null) {
            return null;
        }
        
        // Remove path traversal attempts
        filename = filename.replaceAll("\\.\\.", "");
        filename = filename.replaceAll("[/\\\\]", "");
        
        // Allow only alphanumeric characters, dots, and hyphens
        filename = filename.replaceAll("[^a-zA-Z0-9.\\-]", "_");
        
        // Limit length
        if (filename.length() > 255) {
            filename = filename.substring(0, 255);
        }
        
        return filename;
    }
}

@Aspect
@Component
public class InputValidationAspect {
    
    @Autowired
    private InputSanitizationService sanitizationService;
    
    @Around("@within(org.springframework.web.bind.annotation.RestController)")
    public Object sanitizeControllerInputs(ProceedingJoinPoint joinPoint) throws Throwable {
        Object[] args = joinPoint.getArgs();
        
        // Sanitize string parameters
        for (int i = 0; i < args.length; i++) {
            if (args[i] instanceof String) {
                args[i] = sanitizationService.sanitizeInput((String) args[i]);
            } else if (args[i] instanceof CreateUserRequest) {
                sanitizeUserRequest((CreateUserRequest) args[i]);
            }
            // Add more types as needed
        }
        
        return joinPoint.proceed(args);
    }
    
    private void sanitizeUserRequest(CreateUserRequest request) {
        request.setUsername(sanitizationService.sanitizeInput(request.getUsername()));
        request.setFullName(sanitizationService.sanitizeInput(request.getFullName()));
        request.setBio(sanitizationService.sanitizeHtml(request.getBio()));
    }
}
```

## Testing Best Practices

### 1. Unit Testing
```java
@SpringBootTest
@Testcontainers
public class UserServiceTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Test
    @DisplayName("Should create user successfully")
    void shouldCreateUserSuccessfully() {
        // Given
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername("testuser");
        request.setEmail("test@example.com");
        request.setFullName("Test User");
        request.setPassword("password123");
        
        // When
        User user = userService.createUser(request);
        
        // Then
        assertThat(user).isNotNull();
        assertThat(user.getUsername()).isEqualTo("testuser");
        assertThat(user.getEmail()).isEqualTo("test@example.com");
        assertThat(user.getStatus()).isEqualTo(UserStatus.ACTIVE);
        
        // Verify in database
        User savedUser = userRepository.findById(user.getId()).orElse(null);
        assertThat(savedUser).isNotNull();
        assertThat(savedUser.getUsername()).isEqualTo("testuser");
    }
    
    @Test
    @DisplayName("Should throw exception for duplicate username")
    void shouldThrowExceptionForDuplicateUsername() {
        // Given
        User existingUser = createTestUser("existinguser", "existing@example.com");
        entityManager.persist(existingUser);
        
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername("existinguser"); // Same username
        request.setEmail("new@example.com");
        request.setFullName("New User");
        request.setPassword("password123");
        
        // When & Then
        assertThatThrownBy(() -> userService.createUser(request))
            .isInstanceOf(UsernameAlreadyExistsException.class)
            .hasMessageContaining("existinguser");
    }
    
    @Test
    @DisplayName("Should search users by query")
    void shouldSearchUsersByQuery() {
        // Given
        createTestUser("john_doe", "john@example.com");
        createTestUser("jane_doe", "jane@example.com");
        createTestUser("bob_smith", "bob@example.com");
        
        Pageable pageable = PageRequest.of(0, 10);
        
        // When
        Page<User> results = userService.searchUsers("doe", pageable);
        
        // Then
        assertThat(results.getTotalElements()).isEqualTo(2);
        assertThat(results.getContent())
            .extracting(User::getUsername)
            .containsExactlyInAnyOrder("john_doe", "jane_doe");
    }
    
    private User createTestUser(String username, String email) {
        User user = new User();
        user.setUsername(username);
        user.setEmail(email);
        user.setFullName("Test User");
        user.setPasswordHash("hashedpassword");
        user.setStatus(UserStatus.ACTIVE);
        user.setCreatedAt(Instant.now());
        return user;
    }
}

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
public class UserControllerIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @DisplayName("Should create user via REST API")
    void shouldCreateUserViaRestApi() throws Exception {
        // Given
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername("apiuser");
        request.setEmail("api@example.com");
        request.setFullName("API User");
        request.setPassword("password123");
        
        // When
        MvcResult result = mockMvc.perform(post("/api/v1/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andReturn();
        
        // Then
        UserResponse response = objectMapper.readValue(
            result.getResponse().getContentAsString(), UserResponse.class);
        
        assertThat(response.getUsername()).isEqualTo("apiuser");
        assertThat(response.getEmail()).isEqualTo("api@example.com");
        
        // Verify in database
        User savedUser = userRepository.findByUsername("apiuser").orElse(null);
        assertThat(savedUser).isNotNull();
    }
    
    @Test
    @DisplayName("Should return 400 for invalid request")
    void shouldReturn400ForInvalidRequest() throws Exception {
        // Given - invalid request (missing required fields)
        CreateUserRequest request = new CreateUserRequest();
        // Missing all required fields
        
        // When & Then
        mockMvc.perform(post("/api/v1/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errorCode").value("VALIDATION_ERROR"));
    }
}
```

### 2. Integration Testing
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@DirtiesContext
public class UserServiceIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Container
    static RedisContainer redis = new RedisContainer(RedisContainer.DEFAULT_IMAGE_NAME)
        .withExposedPorts(6379);
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", redis::getFirstMappedPort);
    }
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private FollowRepository followRepository;
    
    @Test
    @DisplayName("Should handle complete user lifecycle")
    void shouldHandleCompleteUserLifecycle() {
        // Create user
        CreateUserRequest createRequest = new CreateUserRequest();
        createRequest.setUsername("lifecycleuser");
        createRequest.setEmail("lifecycle@example.com");
        createRequest.setFullName("Lifecycle User");
        createRequest.setPassword("password123");
        
        User createdUser = userService.createUser(createRequest);
        assertThat(createdUser).isNotNull();
        assertThat(createdUser.getStatus()).isEqualTo(UserStatus.ACTIVE);
        
        Long userId = createdUser.getId();
        
        // Update user
        UpdateUserRequest updateRequest = new UpdateUserRequest();
        updateRequest.setFullName("Updated Lifecycle User");
        updateRequest.setBio("Updated bio");
        
        User updatedUser = userService.updateUser(userId, updateRequest);
        assertThat(updatedUser.getFullName()).isEqualTo("Updated Lifecycle User");
        assertThat(updatedUser.getBio()).isEqualTo("Updated bio");
        
        // Follow another user
        User targetUser = createTestUser("targetuser", "target@example.com");
        
        userService.followUser(userId, targetUser.getId());
        
        // Verify follow relationship
        assertThat(followRepository.existsByFollowerIdAndFolloweeId(userId, targetUser.getId()))
            .isTrue();
        
        // Delete user
        userService.deleteUser(userId);
        
        // Verify soft delete
        User deletedUser = userRepository.findById(userId).orElse(null);
        assertThat(deletedUser).isNotNull();
        assertThat(deletedUser.getStatus()).isEqualTo(UserStatus.DELETED);
    }
    
    private User createTestUser(String username, String email) {
        User user = new User();
        user.setUsername(username);
        user.setEmail(email);
        user.setFullName("Test User");
        user.setPasswordHash("hashedpassword");
        user.setStatus(UserStatus.ACTIVE);
        user.setCreatedAt(Instant.now());
        return userRepository.save(user);
    }
}
```

## Monitoring and Observability

### 1. Application Metrics
```java
@Configuration
public class MetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCustomizer() {
        return registry -> {
            registry.config()
                .commonTags("application", "user-service")
                .commonTags("version", getApplicationVersion());
        };
    }
    
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
    
    private String getApplicationVersion() {
        return getClass().getPackage().getImplementationVersion() != null ?
               getClass().getPackage().getImplementationVersion() : "unknown";
    }
}

@Service
public class MetricsService {
    
    private final MeterRegistry meterRegistry;
    
    // Business metrics
    private final Counter usersCreated = Counter.builder("users_created_total")
        .description("Total number of users created")
        .register(meterRegistry);
    
    private final Counter usersDeleted = Counter.builder("users_deleted_total")
        .description("Total number of users deleted")
        .register(meterRegistry);
    
    private final Timer userCreationTime = Timer.builder("user_creation_duration")
        .description("Time taken to create users")
        .register(meterRegistry);
    
    private final DistributionSummary userSearchResults = DistributionSummary.builder("user_search_results")
        .description("Number of results returned by user searches")
        .register(meterRegistry);
    
    // System metrics
    private final Gauge activeConnections = Gauge.builder("db_connections_active")
        .description("Number of active database connections")
        .register(meterRegistry);
    
    public void recordUserCreated() {
        usersCreated.increment();
    }
    
    public void recordUserDeleted() {
        usersDeleted.increment();
    }
    
    public Timer.Sample startUserCreationTimer() {
        return Timer.start(meterRegistry);
    }
    
    public void recordUserCreationTime(Timer.Sample sample) {
        sample.stop(userCreationTime);
    }
    
    public void recordUserSearchResults(int resultCount) {
        userSearchResults.record(resultCount);
    }
    
    public void updateActiveConnections(int count) {
        // This would be updated by a background task monitoring the connection pool
        activeConnections.set(count);
    }
}

@Aspect
@Component
public class MetricsAspect {
    
    @Autowired
    private MetricsService metricsService;
    
    @AfterReturning("execution(* com.example.user.UserService.createUser(..))")
    public void recordUserCreation() {
        metricsService.recordUserCreated();
    }
    
    @AfterReturning("execution(* com.example.user.UserService.deleteUser(..))")
    public void recordUserDeletion() {
        metricsService.recordUserDeleted();
    }
    
    @Around("execution(* com.example.user.UserService.searchUsers(..))")
    public Object recordSearchMetrics(ProceedingJoinPoint joinPoint) throws Throwable {
        Object result = joinPoint.proceed();
        
        if (result instanceof Page) {
            Page<?> page = (Page<?>) result;
            metricsService.recordUserSearchResults((int) page.getNumberOfElements());
        }
        
        return result;
    }
    
    @Around("execution(* com.example.user.UserService.createUser(..))")
    public Object recordUserCreationTime(ProceedingJoinPoint joinPoint) throws Throwable {
        Timer.Sample sample = metricsService.startUserCreationTimer();
        
        try {
            return joinPoint.proceed();
        } finally {
            metricsService.recordUserCreationTime(sample);
        }
    }
}
```

### 2. Health Checks
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection connection = dataSource.getConnection()) {
            // Test database connectivity
            try (PreparedStatement statement = connection.prepareStatement("SELECT 1")) {
                statement.executeQuery();
            }
            
            // Check connection pool status
            if (dataSource instanceof HikariDataSource) {
                HikariDataSource hikari = (HikariDataSource) dataSource;
                HikariPoolMXBean poolBean = hikari.getHikariPoolMXBean();
                
                return Health.up()
                    .withDetail("activeConnections", poolBean.getActiveConnections())
                    .withDetail("idleConnections", poolBean.getIdleConnections())
                    .withDetail("totalConnections", poolBean.getTotalConnections())
                    .withDetail("threadsAwaitingConnection", poolBean.getThreadsAwaitingConnection())
                    .build();
            }
            
            return Health.up().build();
            
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

@Component
public class ExternalServiceHealthIndicator implements HealthIndicator {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Value("${external.service.health.url}")
    private String healthCheckUrl;
    
    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = restTemplate.getForEntity(healthCheckUrl, String.class);
            
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("statusCode", response.getStatusCodeValue())
                    .withDetail("responseTime", "OK")
                    .build();
            } else {
                return Health.down()
                    .withDetail("statusCode", response.getStatusCodeValue())
                    .withDetail("error", "Non-2xx response")
                    .build();
            }
            
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

@Component
public class CacheHealthIndicator implements HealthIndicator {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Override
    public Health health() {
        try {
            // Test Redis connectivity
            redisTemplate.opsForValue().set("health-check", "ok", Duration.ofSeconds(10));
            String value = (String) redisTemplate.opsForValue().get("health-check");
            
            if ("ok".equals(value)) {
                return Health.up()
                    .withDetail("redis", "connected")
                    .build();
            } else {
                return Health.down()
                    .withDetail("redis", "write/read test failed")
                    .build();
            }
            
        } catch (Exception e) {
            return Health.down()
                .withDetail("redis", "connection failed")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

## Conclusion

Spring Boot best practices provide a solid foundation for building production-ready, scalable, and maintainable applications. By following these guidelines, teams can create applications that are:

- **Well-configured** with proper environment management
- **Properly layered** with clear separation of concerns
- **Thoroughly tested** with comprehensive test suites
- **Secure** with proper authentication and input validation
- **Performant** with effective caching and optimization
- **Observable** with proper monitoring and health checks
- **Maintainable** with clean code and proper error handling

Key principles to remember:
- **Configuration over convention** when needed
- **Dependency injection** for loose coupling
- **Aspect-oriented programming** for cross-cutting concerns
- **Comprehensive testing** at all levels
- **Security by design** from the start
- **Monitoring and observability** built-in
- **Performance optimization** as a continuous process

These practices, when applied consistently, result in applications that are reliable, scalable, and easy to maintain in production environments.
