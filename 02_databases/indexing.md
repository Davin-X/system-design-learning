# Database Indexing

Database indexing is one of the most important concepts for database performance optimization. Proper indexing can reduce query time from minutes to milliseconds, while poor indexing can cause applications to crawl under load.

## What is Database Indexing?

### Definition
A database index is a data structure that improves the speed of data retrieval operations on a database table. It works similar to an index in a book - allowing quick lookup of information without scanning every page.

### How Indexes Work
- **Index Structure**: Organized data structure (B-tree, hash table, etc.)
- **Pointer System**: References to actual data locations
- **Sorted Order**: Data maintained in sorted order for quick access
- **Selective Access**: Avoids full table scans

### Index Components
- **Key**: Column(s) being indexed
- **Pointer**: Reference to actual row location
- **Metadata**: Index statistics and maintenance info
- **Structure**: Underlying data organization

## Types of Database Indexes

### B-Tree Indexes (Most Common)

#### Structure
- **Balanced Tree**: Self-balancing tree structure
- **Node-Based**: Internal nodes contain keys and pointers
- **Leaf Nodes**: Contain actual data or row pointers
- **Sorted Order**: Keys maintained in sorted order

#### Characteristics
- **Range Queries**: Excellent for range scans (BETWEEN, <, >)
- **Prefix Matching**: Good for LIKE queries with leading wildcards
- **Insertion/Deletion**: O(log n) performance
- **Storage**: Moderate space overhead

#### Use Cases
```sql
-- Excellent for B-tree indexes
SELECT * FROM users WHERE age BETWEEN 18 AND 65;
SELECT * FROM orders WHERE created_at > '2023-01-01';
SELECT * FROM products WHERE name LIKE 'Apple%';
```

### Hash Indexes

#### Structure
- **Hash Function**: Maps keys to hash values
- **Buckets**: Data organized into hash buckets
- **Direct Access**: O(1) lookup for exact matches
- **Fixed Size**: Pre-allocated bucket structure

#### Characteristics
- **Exact Match**: Perfect for equality queries (=)
- **Constant Time**: O(1) lookup performance
- **No Range Queries**: Cannot handle range operations
- **Memory Intensive**: Higher memory usage

#### Use Cases
```sql
-- Perfect for hash indexes
SELECT * FROM users WHERE user_id = 12345;
SELECT * FROM cache WHERE key = 'session_abc';
```

### Bitmap Indexes

#### Structure
- **Bit Vectors**: One bit per row for each distinct value
- **Compressed**: Highly compressed bitmaps
- **Multiple Columns**: Can combine multiple columns
- **Space Efficient**: Very low storage for low-cardinality columns

#### Characteristics
- **Low Cardinality**: Best for columns with few distinct values
- **Multiple Conditions**: Excellent for AND/OR operations
- **Compression**: Minimal storage space
- **Read-Only**: Not suitable for frequent updates

#### Use Cases
```sql
-- Excellent for bitmap indexes
SELECT * FROM users WHERE gender = 'M' AND status = 'active';
SELECT * FROM products WHERE category IN ('electronics', 'books');
```

### Full-Text Indexes

#### Structure
- **Inverted Index**: Maps words to document locations
- **Tokenization**: Text broken into searchable tokens
- **Stemming**: Word roots for broader matching
- **Ranking**: Relevance scoring for results

#### Characteristics
- **Text Search**: Optimized for text content
- **Relevance Ranking**: Built-in scoring algorithms
- **Complex Queries**: Support for phrases, proximity, fuzzy matching
- **Storage Heavy**: Significant space requirements

#### Use Cases
```sql
-- Full-text search queries
SELECT * FROM articles WHERE MATCH(title, content) AGAINST('machine learning');
SELECT * FROM products WHERE MATCH(description) AGAINST('"wireless headphones"' IN BOOLEAN MODE);
```

## Index Implementation in Different Databases

### PostgreSQL Indexes
```sql
-- B-tree index (default)
CREATE INDEX idx_users_email ON users(email);

-- Hash index
CREATE INDEX idx_cache_key ON cache USING hash(key);

-- GIN index for arrays/JSON
CREATE INDEX idx_products_tags ON products USING gin(tags);

-- Partial index
CREATE INDEX idx_active_orders ON orders(order_date) 
WHERE status = 'active';

-- Expression index
CREATE INDEX idx_users_lower_email ON users(lower(email));
```

### MySQL Indexes
```sql
-- B-tree index
CREATE INDEX idx_users_email ON users(email);

-- Full-text index
CREATE FULLTEXT INDEX idx_articles_content ON articles(title, content);

-- Spatial index
CREATE SPATIAL INDEX idx_locations_coords ON locations(coords);

-- Composite index
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);
```

### MongoDB Indexes
```javascript
// Single field index
db.users.createIndex({ email: 1 });

// Compound index
db.users.createIndex({ lastName: 1, firstName: 1 });

// Text index
db.articles.createIndex({ title: "text", content: "text" });

// Geospatial index
db.places.createIndex({ location: "2dsphere" });

// Hashed index
db.distributed.createIndex({ _id: "hashed" });
```

## Index Strategy and Best Practices

### Choosing Columns to Index

#### Primary Candidates
- **Primary Keys**: Automatically indexed
- **Foreign Keys**: Essential for joins
- **Frequently Queried Columns**: WHERE clause columns
- **JOIN Columns**: Columns used in join conditions
- **ORDER BY Columns**: Sorting columns
- **GROUP BY Columns**: Aggregation columns

#### Index Selectivity
- **High Selectivity**: Many distinct values (good candidates)
- **Low Selectivity**: Few distinct values (may not help)
- **Rule of Thumb**: Selectivity > 5-10% for effectiveness

### Composite Indexes

#### When to Use
- **Multiple WHERE Conditions**: Queries with AND conditions
- **Order Matters**: Leftmost columns used most frequently
- **Covering Indexes**: Include all queried columns

#### Best Practices
```sql
-- Good: Frequent query pattern
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Bad: Wrong column order
CREATE INDEX idx_orders_status_user ON orders(status, user_id); -- status has lower selectivity

-- Covering index
CREATE INDEX idx_users_name_email ON users(last_name, first_name, email);
-- Covers: SELECT email FROM users WHERE last_name = 'Smith' AND first_name = 'John'
```

### Index Maintenance

#### Rebuilding Indexes
```sql
-- PostgreSQL
REINDEX INDEX idx_users_email;

-- MySQL
ALTER TABLE users DROP INDEX idx_users_email;
ALTER TABLE users ADD INDEX idx_users_email(email);

-- MongoDB
db.users.reIndex();
```

#### Monitoring Index Usage
```sql
-- PostgreSQL: Check index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- MySQL: Index usage statistics
SHOW INDEX FROM users;
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
```

### Index Overhead

#### Storage Costs
- **Space Usage**: Indexes require additional disk space
- **Memory Usage**: Frequently used indexes cached in memory
- **Backup Size**: Indexes included in backups

#### Performance Costs
- **Write Performance**: INSERT/UPDATE/DELETE slower due to index maintenance
- **Maintenance**: Index rebuilding and reorganization
- **Locking**: Index operations may lock tables

## Common Indexing Problems and Solutions

### Problem 1: Missing Indexes
**Symptoms:**
- Slow queries with full table scans
- High CPU usage on database server
- Long response times for common queries

**Solution:**
```sql
-- Identify slow queries
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- Add appropriate index
CREATE INDEX idx_users_email ON users(email);
```

### Problem 2: Unused Indexes
**Symptoms:**
- High storage usage
- Slower write operations
- Maintenance overhead

**Solution:**
```sql
-- PostgreSQL: Find unused indexes
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Remove unused indexes
DROP INDEX idx_unused_index;
```

### Problem 3: Index Bloat
**Symptoms:**
- Indexes much larger than expected
- Poor query performance
- High maintenance costs

**Solution:**
```sql
-- PostgreSQL: Rebuild bloated indexes
REINDEX INDEX CONCURRENTLY idx_bloated_index;

-- MySQL: Optimize table
OPTIMIZE TABLE users;
```

### Problem 4: Wrong Index Type
**Symptoms:**
- Poor performance for specific query patterns
- Index not being used by optimizer

**Solution:**
- Analyze query patterns
- Choose appropriate index type (B-tree, hash, GIN, etc.)
- Consider partial or expression indexes

## Advanced Indexing Techniques

### Partial Indexes
Index only a subset of table rows.

```sql
-- PostgreSQL
CREATE INDEX idx_active_orders ON orders(order_date)
WHERE status = 'active';

-- Benefits: Smaller index, faster queries for common conditions
```

### Expression Indexes
Index results of expressions or functions.

```sql
-- PostgreSQL
CREATE INDEX idx_users_lower_email ON users(lower(email));

-- MySQL
CREATE INDEX idx_users_email_length ON users((length(email)));

-- Benefits: Optimize function-based queries
```

### Functional Indexes
Index computed values.

```sql
-- PostgreSQL
CREATE INDEX idx_products_price_category ON products((price * discount), category);

-- Benefits: Optimize computed column queries
```

### Covering Indexes
Index includes all columns needed by a query.

```sql
-- Include all SELECT and WHERE columns
CREATE INDEX idx_users_covering ON users(last_name, first_name, email, phone);

SELECT email, phone FROM users 
WHERE last_name = 'Smith' AND first_name = 'John';
-- Query satisfied entirely by index (no table access)
```

## Index in Java Applications

### JPA/Hibernate Indexing
```java
@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_users_email", columnList = "email"),
    @Index(name = "idx_users_name", columnList = "last_name, first_name")
})
public class User {
    
    @Id
    @GeneratedValue
    private Long id;
    
    @Column(nullable = false)
    private String email;
    
    @Column(name = "first_name")
    private String firstName;
    
    @Column(name = "last_name")
    private String lastName;
    
    // Other fields with appropriate indexes
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
}
```

### Spring Data JPA Query Methods
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Uses idx_users_email
    Optional<User> findByEmail(String email);
    
    // Uses idx_users_name
    List<User> findByLastNameAndFirstName(String lastName, String firstName);
    
    // Custom query with index hint
    @QueryHints(@QueryHint(name = "org.hibernate.comment", 
                          value = "USE INDEX (idx_users_email)"))
    @Query("SELECT u FROM User u WHERE u.email LIKE :emailPattern")
    List<User> findByEmailPattern(@Param("emailPattern") String emailPattern);
}
```

### Index Monitoring in Java
```java
@Service
public class DatabaseMetricsService {
    
    @Autowired
    private EntityManager entityManager;
    
    public List<IndexUsage> getIndexUsage() {
        // Native query to get index statistics
        Query query = entityManager.createNativeQuery(
            "SELECT indexname, idx_scan, idx_tup_read, idx_tup_fetch " +
            "FROM pg_stat_user_indexes WHERE schemaname = 'public'"
        );
        
        List<Object[]> results = query.getResultList();
        return results.stream()
            .map(row -> new IndexUsage(
                (String) row[0], 
                (Long) row[1], 
                (Long) row[2], 
                (Long) row[3]
            ))
            .collect(Collectors.toList());
    }
}
```

## Performance Impact Measurement

### Before and After Indexing
```sql
-- Measure query performance
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- After adding index
CREATE INDEX idx_users_email ON users(email);
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- Compare execution times and query plans
```

### Index Effectiveness Metrics
- **Index Hit Rate**: Percentage of queries using the index
- **Query Execution Time**: Reduction in response time
- **CPU Usage**: Decrease in database CPU utilization
- **I/O Operations**: Reduction in disk reads

## Index Design for Specific Use Cases

### E-commerce Platform
```sql
-- Product search and filtering
CREATE INDEX idx_products_category_price ON products(category, price);
CREATE INDEX idx_products_name_fts ON products USING gin(to_tsvector('english', name));

-- Order processing
CREATE INDEX idx_orders_user_status ON orders(user_id, status, created_at);
CREATE INDEX idx_orders_date_amount ON orders(created_at, total_amount);

-- User management
CREATE INDEX idx_users_email_active ON users(email) WHERE active = true;
```

### Social Media Platform
```sql
-- Timeline queries
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);
CREATE INDEX idx_follows_follower_following ON follows(follower_id, following_id);

-- Search functionality
CREATE INDEX idx_users_name_search ON users USING gin(to_tsvector('english', display_name));
CREATE INDEX idx_posts_content_fts ON posts USING gin(to_tsvector('english', content));
```

### Analytics System
```sql
-- Time-series data
CREATE INDEX idx_events_timestamp_type ON events(timestamp, event_type);
CREATE INDEX idx_metrics_date_value ON metrics(date, metric_value);

-- Aggregation queries
CREATE INDEX idx_sales_region_product ON sales(region, product_id, amount);
```

## Conclusion

Database indexing is a critical aspect of system performance optimization. Proper indexing can dramatically improve query performance, while poor indexing can severely impact application responsiveness.

**Key Takeaways:**
- **Choose Wisely**: Index frequently queried columns, not all columns
- **Monitor Usage**: Track which indexes are actually used
- **Balance Trade-offs**: Consider write performance vs read performance
- **Maintain Regularly**: Rebuild and optimize indexes as needed
- **Test Thoroughly**: Measure performance impact of index changes

Effective indexing requires understanding your query patterns, data access patterns, and performance requirements. Regular monitoring and optimization are essential for maintaining optimal database performance.

**Remember**: Indexes speed up reads but slow down writes. The key is finding the right balance for your specific use case.
