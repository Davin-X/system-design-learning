# Database Schema Design Examples

Good database schema design is fundamental to building scalable, maintainable, and performant applications. This guide provides practical examples of schema design patterns for common application scenarios, with detailed explanations of design decisions and trade-offs.

## Schema Design Principles

### 1. Normalization vs Denormalization
- **Normalization**: Reduce redundancy, improve data integrity
- **Denormalization**: Improve read performance, accept redundancy
- **Hybrid Approach**: Normalize for writes, denormalize for reads

### 2. Data Types and Constraints
- **Appropriate Types**: Choose smallest sufficient data type
- **Constraints**: Primary keys, foreign keys, check constraints
- **Defaults**: Sensible default values
- **Validation**: Domain constraints at database level

### 3. Indexing Strategy
- **Primary Keys**: Always indexed
- **Foreign Keys**: Index for join performance
- **Query Patterns**: Index based on access patterns
- **Composite Indexes**: Multiple columns for complex queries

### 4. Naming Conventions
- **Consistency**: Use consistent naming across tables
- **Clarity**: Names should be self-explanatory
- **Standards**: Follow database naming conventions
- **Prefixes/Suffixes**: Use for related tables (user_, order_)

## E-commerce Platform Schema

### Core Entities
```sql
-- Users table
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_users_email (email),
    INDEX idx_users_created_at (created_at)
);

-- Products table
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    category_id BIGINT,
    brand VARCHAR(100),
    stock_quantity INT DEFAULT 0 CHECK (stock_quantity >= 0),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (category_id) REFERENCES categories(id),
    INDEX idx_products_category (category_id),
    INDEX idx_products_brand (brand),
    INDEX idx_products_price (price),
    INDEX idx_products_active (is_active)
);

-- Categories table (hierarchical)
CREATE TABLE categories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    parent_id BIGINT,
    level INT DEFAULT 1,
    is_active BOOLEAN DEFAULT TRUE,
    
    FOREIGN KEY (parent_id) REFERENCES categories(id),
    INDEX idx_categories_parent (parent_id),
    INDEX idx_categories_level (level)
);
```

### Order Management
```sql
-- Orders table
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    status ENUM('pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    total_amount DECIMAL(12,2) NOT NULL CHECK (total_amount >= 0),
    tax_amount DECIMAL(10,2) DEFAULT 0,
    shipping_amount DECIMAL(10,2) DEFAULT 0,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    shipping_address_id BIGINT,
    billing_address_id BIGINT,
    payment_method VARCHAR(50),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    shipped_at TIMESTAMP NULL,
    delivered_at TIMESTAMP NULL,
    
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (shipping_address_id) REFERENCES addresses(id),
    FOREIGN KEY (billing_address_id) REFERENCES addresses(id),
    INDEX idx_orders_user (user_id),
    INDEX idx_orders_status (status),
    INDEX idx_orders_date (order_date),
    INDEX idx_orders_number (order_number)
);

-- Order items table
CREATE TABLE order_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL CHECK (unit_price >= 0),
    total_price DECIMAL(10,2) NOT NULL CHECK (total_price >= 0),
    product_name VARCHAR(255), -- Denormalized for history
    product_sku VARCHAR(50),   -- Denormalized for history
    
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id),
    INDEX idx_order_items_order (order_id),
    INDEX idx_order_items_product (product_id)
);
```

### Addresses and Payments
```sql
-- Addresses table (reusable)
CREATE TABLE addresses (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    type ENUM('billing', 'shipping') NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    company VARCHAR(100),
    street_address VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100),
    postal_code VARCHAR(20) NOT NULL,
    country VARCHAR(100) NOT NULL DEFAULT 'US',
    phone VARCHAR(20),
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_addresses_user (user_id),
    INDEX idx_addresses_type (type),
    INDEX idx_addresses_country (country)
);

-- Payments table
CREATE TABLE payments (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    payment_method ENUM('credit_card', 'debit_card', 'paypal', 'bank_transfer') NOT NULL,
    payment_status ENUM('pending', 'processing', 'completed', 'failed', 'refunded') DEFAULT 'pending',
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    transaction_id VARCHAR(255) UNIQUE,
    payment_gateway VARCHAR(50),
    payment_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (order_id) REFERENCES orders(id),
    INDEX idx_payments_order (order_id),
    INDEX idx_payments_status (payment_status),
    INDEX idx_payments_date (payment_date)
);
```

### Design Decisions
1. **Normalized Addresses**: Reusable across orders
2. **Denormalized Order Items**: Historical data preservation
3. **Status Enums**: Controlled vocabulary for data integrity
4. **Comprehensive Indexing**: Support common query patterns
5. **Audit Fields**: created_at, updated_at for tracking

## Social Media Platform Schema

### User Management
```sql
-- Users with profiles
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    display_name VARCHAR(100),
    bio TEXT,
    avatar_url VARCHAR(500),
    website VARCHAR(255),
    location VARCHAR(100),
    is_verified BOOLEAN DEFAULT FALSE,
    is_private BOOLEAN DEFAULT FALSE,
    follower_count INT DEFAULT 0,
    following_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_users_username (username),
    INDEX idx_users_email (email),
    INDEX idx_users_created_at (created_at),
    FULLTEXT idx_users_search (display_name, bio, username)
);

-- User relationships
CREATE TABLE user_relationships (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    follower_id BIGINT NOT NULL,
    following_id BIGINT NOT NULL,
    status ENUM('pending', 'accepted', 'blocked') DEFAULT 'accepted',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (follower_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (following_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY unique_relationship (follower_id, following_id),
    INDEX idx_relationships_follower (follower_id),
    INDEX idx_relationships_following (following_id),
    INDEX idx_relationships_status (status)
);
```

### Content Management
```sql
-- Posts table
CREATE TABLE posts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    media_urls JSON, -- Array of media URLs
    post_type ENUM('text', 'image', 'video', 'link') DEFAULT 'text',
    visibility ENUM('public', 'followers', 'private') DEFAULT 'public',
    like_count INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    share_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_posts_user (user_id),
    INDEX idx_posts_type (post_type),
    INDEX idx_posts_visibility (visibility),
    INDEX idx_posts_created_at (created_at),
    FULLTEXT idx_posts_content (content)
);

-- Comments with threading
CREATE TABLE comments (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    post_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    parent_comment_id BIGINT, -- For nested comments
    content TEXT NOT NULL,
    like_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (post_id) REFERENCES posts(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (parent_comment_id) REFERENCES comments(id) ON DELETE CASCADE,
    INDEX idx_comments_post (post_id),
    INDEX idx_comments_user (user_id),
    INDEX idx_comments_parent (parent_comment_id),
    INDEX idx_comments_created_at (created_at)
);

-- Likes (polymorphic)
CREATE TABLE likes (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    entity_type ENUM('post', 'comment') NOT NULL,
    entity_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_likes_user (user_id),
    INDEX idx_likes_entity (entity_type, entity_id),
    UNIQUE KEY unique_like (user_id, entity_type, entity_id)
);
```

### Activity and Notifications
```sql
-- Activities for feed generation
CREATE TABLE activities (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    activity_type ENUM('post', 'like', 'comment', 'follow', 'share') NOT NULL,
    entity_type ENUM('post', 'comment', 'user') NOT NULL,
    entity_id BIGINT NOT NULL,
    metadata JSON, -- Additional activity data
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_activities_user (user_id),
    INDEX idx_activities_type (activity_type),
    INDEX idx_activities_entity (entity_type, entity_id),
    INDEX idx_activities_created_at (created_at)
);

-- Notifications
CREATE TABLE notifications (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    notification_type ENUM('like', 'comment', 'follow', 'mention', 'system') NOT NULL,
    title VARCHAR(255) NOT NULL,
    message TEXT,
    entity_type ENUM('post', 'comment', 'user'),
    entity_id BIGINT,
    is_read BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_notifications_user (user_id),
    INDEX idx_notifications_read (is_read),
    INDEX idx_notifications_created_at (created_at)
);
```

### Design Decisions
1. **JSON Fields**: Flexible metadata storage
2. **Polymorphic Likes**: Single table for different entity types
3. **Threaded Comments**: Hierarchical comment structure
4. **Counters**: Denormalized counts for performance
5. **Full-Text Search**: Content search capabilities

## Multi-Tenant SaaS Schema

### Tenant Management
```sql
-- Tenants (organizations)
CREATE TABLE tenants (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    subdomain VARCHAR(100) UNIQUE NOT NULL,
    status ENUM('active', 'suspended', 'inactive') DEFAULT 'active',
    plan ENUM('free', 'basic', 'premium', 'enterprise') DEFAULT 'free',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_tenants_subdomain (subdomain),
    INDEX idx_tenants_status (status),
    INDEX idx_tenants_plan (plan)
);

-- Tenant configurations
CREATE TABLE tenant_configs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id BIGINT NOT NULL,
    config_key VARCHAR(100) NOT NULL,
    config_value TEXT,
    
    FOREIGN KEY (tenant_id) REFERENCES tenants(id) ON DELETE CASCADE,
    UNIQUE KEY unique_config (tenant_id, config_key),
    INDEX idx_configs_tenant (tenant_id)
);
```

### Multi-Tenant Data Isolation
```sql
-- Users belong to tenants
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id BIGINT NOT NULL,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role ENUM('admin', 'user', 'viewer') DEFAULT 'user',
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (tenant_id) REFERENCES tenants(id) ON DELETE CASCADE,
    UNIQUE KEY unique_user_email (tenant_id, email),
    INDEX idx_users_tenant (tenant_id),
    INDEX idx_users_role (role),
    INDEX idx_users_active (is_active)
);

-- Projects (tenant-scoped)
CREATE TABLE projects (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id BIGINT NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    status ENUM('active', 'archived', 'deleted') DEFAULT 'active',
    created_by BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (tenant_id) REFERENCES tenants(id) ON DELETE CASCADE,
    FOREIGN KEY (created_by) REFERENCES users(id),
    INDEX idx_projects_tenant (tenant_id),
    INDEX idx_projects_status (status),
    INDEX idx_projects_created_by (created_by)
);

-- Tasks (belong to projects)
CREATE TABLE tasks (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id BIGINT NOT NULL, -- Denormalized for performance
    project_id BIGINT NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    assigned_to BIGINT,
    status ENUM('todo', 'in_progress', 'review', 'done') DEFAULT 'todo',
    priority ENUM('low', 'medium', 'high', 'urgent') DEFAULT 'medium',
    due_date DATE,
    created_by BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (tenant_id) REFERENCES tenants(id) ON DELETE CASCADE,
    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE,
    FOREIGN KEY (assigned_to) REFERENCES users(id),
    FOREIGN KEY (created_by) REFERENCES users(id),
    INDEX idx_tasks_tenant (tenant_id),
    INDEX idx_tasks_project (project_id),
    INDEX idx_tasks_assigned_to (assigned_to),
    INDEX idx_tasks_status (status),
    INDEX idx_tasks_priority (priority),
    INDEX idx_tasks_due_date (due_date)
);
```

### Row-Level Security Implementation
```sql
-- Create a function to check tenant access
DELIMITER //
CREATE FUNCTION check_tenant_access(user_tenant_id BIGINT, resource_tenant_id BIGINT)
RETURNS BOOLEAN
DETERMINISTIC
BEGIN
    RETURN user_tenant_id = resource_tenant_id;
END //
DELIMITER ;

-- Add tenant context to queries
SELECT * FROM projects p
WHERE check_tenant_access(@current_tenant_id, p.tenant_id);
```

### Design Decisions
1. **Tenant ID in Every Table**: Enforce data isolation
2. **Row-Level Security**: Application-level tenant isolation
3. **Configurable Features**: Tenant-specific configurations
4. **Soft Deletes**: Preserve data for compliance
5. **Audit Trails**: Track changes across tenants

## Time-Series Data Schema

### IoT Sensor Data
```sql
-- Sensors table
CREATE TABLE sensors (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    device_id VARCHAR(100) UNIQUE NOT NULL,
    sensor_type ENUM('temperature', 'humidity', 'pressure', 'motion') NOT NULL,
    location VARCHAR(255),
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_sensors_type (sensor_type),
    INDEX idx_sensors_location (location),
    INDEX idx_sensors_active (is_active),
    SPATIAL INDEX idx_sensors_location_coords (location_coords)
);

-- Time-series measurements
CREATE TABLE sensor_measurements (
    sensor_id BIGINT NOT NULL,
    timestamp TIMESTAMP(3) NOT NULL, -- Microsecond precision
    value DECIMAL(10,4) NOT NULL,
    unit VARCHAR(20),
    quality_score TINYINT CHECK (quality_score BETWEEN 0 AND 100),
    
    PRIMARY KEY (sensor_id, timestamp),
    FOREIGN KEY (sensor_id) REFERENCES sensors(id),
    
    -- Partition by month for time-series optimization
) PARTITION BY RANGE (YEAR(timestamp)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);

-- Aggregated data for fast queries
CREATE TABLE sensor_aggregates_hourly (
    sensor_id BIGINT NOT NULL,
    date_hour DATETIME NOT NULL,
    avg_value DECIMAL(10,4),
    min_value DECIMAL(10,4),
    max_value DECIMAL(10,4),
    count INT,
    
    PRIMARY KEY (sensor_id, date_hour),
    FOREIGN KEY (sensor_id) REFERENCES sensors(id),
    
    INDEX idx_aggregates_sensor (sensor_id),
    INDEX idx_aggregates_hour (date_hour)
);

CREATE TABLE sensor_aggregates_daily (
    sensor_id BIGINT NOT NULL,
    date DATE NOT NULL,
    avg_value DECIMAL(10,4),
    min_value DECIMAL(10,4),
    max_value DECIMAL(10,4),
    count INT,
    
    PRIMARY KEY (sensor_id, date),
    FOREIGN KEY (sensor_id) REFERENCES sensors(id),
    
    INDEX idx_aggregates_daily_sensor (sensor_id),
    INDEX idx_aggregates_daily_date (date)
);
```

### Design Decisions
1. **Composite Primary Key**: sensor_id + timestamp for time-series
2. **Table Partitioning**: Monthly partitions for performance
3. **Pre-aggregated Data**: Hourly/daily aggregates for fast queries
4. **Precision Timestamps**: Microsecond precision for IoT data
5. **Quality Scores**: Data validation and reliability tracking

## Content Management System Schema

### Content Structure
```sql
-- Content types
CREATE TABLE content_types (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    fields JSON, -- Dynamic field definitions
    is_active BOOLEAN DEFAULT TRUE,
    
    INDEX idx_content_types_name (name)
);

-- Content items
CREATE TABLE content_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    content_type_id BIGINT NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    excerpt TEXT,
    status ENUM('draft', 'published', 'archived') DEFAULT 'draft',
    author_id BIGINT NOT NULL,
    published_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (content_type_id) REFERENCES content_types(id),
    FOREIGN KEY (author_id) REFERENCES users(id),
    INDEX idx_content_type (content_type_id),
    INDEX idx_content_author (author_id),
    INDEX idx_content_status (status),
    INDEX idx_content_published (published_at),
    INDEX idx_content_slug (slug),
    FULLTEXT idx_content_search (title, content, excerpt)
);

-- Dynamic fields (EAV pattern for flexibility)
CREATE TABLE content_fields (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    content_item_id BIGINT NOT NULL,
    field_name VARCHAR(100) NOT NULL,
    field_value TEXT,
    field_type ENUM('text', 'number', 'boolean', 'date', 'json') DEFAULT 'text',
    
    FOREIGN KEY (content_item_id) REFERENCES content_items(id) ON DELETE CASCADE,
    INDEX idx_fields_item (content_item_id),
    INDEX idx_fields_name (field_name)
);

-- Categories and tags
CREATE TABLE categories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    parent_id BIGINT,
    
    FOREIGN KEY (parent_id) REFERENCES categories(id),
    INDEX idx_categories_parent (parent_id),
    INDEX idx_categories_slug (slug)
);

CREATE TABLE content_categories (
    content_item_id BIGINT NOT NULL,
    category_id BIGINT NOT NULL,
    
    PRIMARY KEY (content_item_id, category_id),
    FOREIGN KEY (content_item_id) REFERENCES content_items(id) ON DELETE CASCADE,
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE CASCADE,
    INDEX idx_content_categories_category (category_id)
);

CREATE TABLE tags (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) UNIQUE NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL,
    
    INDEX idx_tags_name (name),
    INDEX idx_tags_slug (slug)
);

CREATE TABLE content_tags (
    content_item_id BIGINT NOT NULL,
    tag_id BIGINT NOT NULL,
    
    PRIMARY KEY (content_item_id, tag_id),
    FOREIGN KEY (content_item_id) REFERENCES content_items(id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE,
    INDEX idx_content_tags_tag (tag_id)
);
```

### Design Decisions
1. **Dynamic Content Types**: Flexible schema with JSON fields
2. **EAV Pattern**: Entity-Attribute-Value for custom fields
3. **Hierarchical Categories**: Tree structure for organization
4. **Many-to-Many Relationships**: Tags and categories
5. **Full-Text Search**: Content search capabilities

## Schema Migration Strategy

### Version Control for Schema
```sql
-- Schema versions table
CREATE TABLE schema_versions (
    version VARCHAR(20) PRIMARY KEY,
    description TEXT,
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    applied_by VARCHAR(100)
);

-- Migration: Add user preferences
INSERT INTO schema_versions (version, description) VALUES ('1.1.0', 'Add user preferences');

ALTER TABLE users ADD COLUMN preferences JSON DEFAULT '{}';
CREATE INDEX idx_users_preferences ON users((preferences->>'theme'));

-- Record migration completion
UPDATE schema_versions SET applied_at = NOW() WHERE version = '1.1.0';
```

### Safe Migration Practices
1. **Backward Compatibility**: New features don't break existing code
2. **Zero-Downtime Deployments**: Schema changes during maintenance windows
3. **Rollback Plans**: Ability to revert changes if needed
4. **Data Validation**: Verify data integrity after migrations
5. **Gradual Rollouts**: Test migrations on subsets of data first

## Performance Optimization Patterns

### Indexing Strategy
```sql
-- Composite indexes for common queries
CREATE INDEX idx_orders_user_status_date ON orders(user_id, status, order_date);

-- Partial indexes for active records
CREATE INDEX idx_products_active_name ON products(name) WHERE is_active = true;

-- Functional indexes for computed values
CREATE INDEX idx_users_email_domain ON users(SUBSTRING_INDEX(email, '@', -1));

-- Covering indexes for SELECT queries
CREATE INDEX idx_users_covering ON users(id, name, email, created_at);
```

### Query Optimization
```sql
-- Use UNION ALL for mutually exclusive conditions
SELECT * FROM products WHERE category = 'electronics' AND price < 100
UNION ALL
SELECT * FROM products WHERE category = 'books' AND price < 50;

-- Avoid correlated subqueries
SELECT u.name, (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) as order_count
FROM users u;

-- Prefer JOINs for better performance
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
```

## Schema Design Best Practices

### 1. Normalization Principles
- **1NF**: Atomic values, no repeating groups
- **2NF**: No partial dependencies on composite keys
- **3NF**: No transitive dependencies
- **BCNF**: Every determinant is a candidate key

### 2. Denormalization Decisions
- **Read-Heavy Workloads**: Consider denormalization
- **Write-Heavy Workloads**: Maintain normalization
- **Real-Time Requirements**: Balance consistency vs performance
- **Storage Costs**: Consider denormalization impact

### 3. Constraint Usage
- **Primary Keys**: Always use surrogate keys (auto-increment)
- **Foreign Keys**: Maintain referential integrity
- **Check Constraints**: Enforce business rules
- **Unique Constraints**: Prevent duplicate data

### 4. Data Types Selection
- **Storage Efficiency**: Use smallest sufficient types
- **Precision**: Choose appropriate numeric precision
- **Character Sets**: Consider internationalization
- **Temporal Types**: Use appropriate date/time types

### 5. Naming Conventions
- **Tables**: Plural nouns (users, orders, products)
- **Columns**: snake_case (user_id, created_at, is_active)
- **Indexes**: idx_table_columns (idx_users_email)
- **Constraints**: Descriptive names (fk_orders_user_id)

## Conclusion

Database schema design requires balancing multiple competing concerns: performance, maintainability, scalability, and data integrity. The examples above demonstrate practical approaches for common application patterns.

**Key Takeaways:**
- **Understand Access Patterns**: Design schema based on query requirements
- **Balance Normalization**: Normalize for integrity, denormalize for performance
- **Proper Indexing**: Index for query patterns, not blindly
- **Consider Growth**: Design for future scaling needs
- **Document Decisions**: Explain design choices for maintenance

Good schema design evolves with application needs. Regular review and optimization based on actual usage patterns is essential for long-term success.
