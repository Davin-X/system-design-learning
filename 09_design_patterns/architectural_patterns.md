# Architectural Patterns

Architectural patterns provide high-level structures for organizing software systems. This guide covers essential architectural patterns used in enterprise software development, with implementation examples and guidance on when to apply each pattern.

## What are Architectural Patterns?

Architectural patterns are fundamental structural organizations for software systems that provide solutions to recurring architectural problems. They define the overall shape and structure of software applications, establishing relationships between components and defining communication protocols.

### Pattern Categories
- **Layered Architecture**: Separation of concerns through horizontal layers
- **Hexagonal Architecture**: Dependency inversion and port-adapter pattern
- **CQRS/ES Architecture**: Command Query Responsibility Segregation and Event Sourcing
- **Microservices Architecture**: Distributed system decomposition
- **Event-Driven Architecture**: Loose coupling through events
- **Serverless Architecture**: Function-as-a-service computing model

## Layered Architecture Pattern

### Overview
Organize the application into horizontal layers, each with specific responsibilities and well-defined interfaces.

#### Traditional Layered Architecture
```java
// Presentation Layer (Controllers)
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderApplicationService orderService;
    
    @PostMapping
    public ResponseEntity<OrderDto> createOrder(@RequestBody CreateOrderRequest request) {
        Order order = orderService.createOrder(request);
        return ResponseEntity.created(URI.create("/api/orders/" + order.getId()))
                           .body(OrderDto.from(order));
    }
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderDto> getOrder(@PathVariable String orderId) {
        Order order = orderService.getOrder(orderId);
        return ResponseEntity.ok(OrderDto.from(order));
    }
}

// Application Layer (Application Services)
@Service
@Transactional
public class OrderApplicationService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    public Order createOrder(CreateOrderRequest request) {
        // Application logic orchestration
        validateOrderRequest(request);
        
        User user = userService.getUser(request.getUserId());
        checkInventoryAvailability(request.getItems());
        
        Order order = new Order(request.getUserId(), request.getItems());
        order = orderRepository.save(order);
        
        // Process payment
        PaymentResult payment = paymentService.processPayment(order.getId(), order.getTotalAmount());
        if (!payment.isSuccessful()) {
            throw new PaymentFailedException("Payment processing failed");
        }
        
        order.setStatus(OrderStatus.CONFIRMED);
        return orderRepository.save(order);
    }
    
    public Order getOrder(String orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
    
    private void validateOrderRequest(CreateOrderRequest request) {
        if (request.getItems() == null || request.getItems().isEmpty()) {
            throw new InvalidOrderException("Order must contain at least one item");
        }
        // Additional validation logic
    }
    
    private void checkInventoryAvailability(List<OrderItem> items) {
        for (OrderItem item : items) {
            if (!inventoryService.isAvailable(item.getProductId(), item.getQuantity())) {
                throw new InsufficientInventoryException(item.getProductId());
            }
        }
    }
}

// Domain Layer (Domain Entities and Business Logic)
@Entity
@Table(name = "orders")
public class Order {
    
    @Id
    private String id;
    
    @Column(name = "user_id")
    private Long userId;
    
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    @JoinColumn(name = "order_id")
    private List<OrderItem> items;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @Column(name = "total_amount")
    private BigDecimal totalAmount;
    
    @Column(name = "created_at")
    private Instant createdAt;
    
    @Column(name = "updated_at")
    private Instant updatedAt;
    
    // Constructors, getters, setters
    
    public BigDecimal getTotalAmount() {
        if (totalAmount == null) {
            calculateTotal();
        }
        return totalAmount;
    }
    
    private void calculateTotal() {
        this.totalAmount = items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    // Domain business methods
    public void confirm() {
        if (status != OrderStatus.CREATED) {
            throw new InvalidOrderStateException("Can only confirm orders in CREATED state");
        }
        this.status = OrderStatus.CONFIRMED;
        this.updatedAt = Instant.now();
    }
    
    public void ship() {
        if (status != OrderStatus.CONFIRMED) {
            throw new InvalidOrderStateException("Can only ship confirmed orders");
        }
        this.status = OrderStatus.SHIPPED;
        this.updatedAt = Instant.now();
    }
}

@Entity
@Table(name = "order_items")
public class OrderItem {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "order_id")
    private String orderId;
    
    @Column(name = "product_id")
    private Long productId;
    
    @Column(name = "product_name")
    private String productName;
    
    @Column(name = "quantity")
    private int quantity;
    
    @Column(name = "price")
    private BigDecimal price;
    
    // Constructors, getters, setters
}

// Infrastructure Layer (Repositories, External Services)
@Repository
public interface OrderRepository extends JpaRepository<Order, String> {
    
    @Query("SELECT o FROM Order o WHERE o.userId = :userId ORDER BY o.createdAt DESC")
    List<Order> findByUserIdOrderByCreatedAtDesc(@Param("userId") Long userId);
    
    @Query("SELECT o FROM Order o WHERE o.status = :status AND o.createdAt >= :since")
    List<Order> findByStatusAndCreatedAfter(@Param("status") OrderStatus status, 
                                          @Param("since") Instant since);
}

@Service
public class JpaOrderRepository implements OrderRepository {
    
    @Autowired
    private OrderJpaRepository jpaRepository;
    
    @Autowired
    private OrderItemJpaRepository itemJpaRepository;
    
    @Override
    public Order save(Order order) {
        // Save order and items
        OrderEntity orderEntity = orderJpaRepository.save(OrderEntity.from(order));
        
        for (OrderItem item : order.getItems()) {
            OrderItemEntity itemEntity = OrderItemEntity.from(item, orderEntity.getId());
            itemJpaRepository.save(itemEntity);
        }
        
        return orderEntity.toOrder();
    }
    
    // Other repository methods...
}

@Service
public class ExternalUserService implements UserService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Value("${user.service.url}")
    private String userServiceUrl;
    
    @Override
    public User getUser(Long userId) {
        String url = userServiceUrl + "/users/" + userId;
        ResponseEntity<User> response = restTemplate.getForEntity(url, User.class);
        return response.getBody();
    }
    
    @Override
    public List<User> getUsers(List<Long> userIds) {
        // Batch user retrieval
        String ids = userIds.stream().map(String::valueOf).collect(Collectors.joining(","));
        String url = userServiceUrl + "/users/batch?ids=" + ids;
        ResponseEntity<List<User>> response = restTemplate.exchange(url, HttpMethod.GET, null, 
            new ParameterizedTypeReference<List<User>>() {});
        return response.getBody();
    }
}
```

### Clean Architecture Variation
```java
// Use Cases (Application Layer)
public interface CreateOrderUseCase {
    Order execute(CreateOrderRequest request);
}

public interface GetOrderUseCase {
    Order execute(String orderId);
}

@UseCase
public class CreateOrderUseCaseImpl implements CreateOrderUseCase {
    
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    private final InventoryRepository inventoryRepository;
    private final PaymentService paymentService;
    private final DomainEventPublisher eventPublisher;
    
    public CreateOrderUseCaseImpl(OrderRepository orderRepository, 
                                UserRepository userRepository,
                                InventoryRepository inventoryRepository,
                                PaymentService paymentService,
                                DomainEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.userRepository = userRepository;
        this.inventoryRepository = inventoryRepository;
        this.paymentService = paymentService;
        this.eventPublisher = eventPublisher;
    }
    
    @Override
    public Order execute(CreateOrderRequest request) {
        // Use case orchestration
        User user = userRepository.findById(request.getUserId())
            .orElseThrow(() -> new UserNotFoundException(request.getUserId()));
            
        // Check inventory
        for (OrderItemRequest itemRequest : request.getItems()) {
            Product product = inventoryRepository.findProductById(itemRequest.getProductId());
            if (product.getStockQuantity() < itemRequest.getQuantity()) {
                throw new InsufficientInventoryException(itemRequest.getProductId());
            }
        }
        
        // Create order
        Order order = Order.create(user.getId(), request.getItems());
        
        // Process payment
        PaymentResult payment = paymentService.processPayment(order.getTotalAmount());
        if (!payment.isSuccessful()) {
            throw new PaymentFailedException("Payment failed");
        }
        
        // Reserve inventory
        for (OrderItem item : order.getItems()) {
            inventoryRepository.reserveStock(item.getProductId(), item.getQuantity());
        }
        
        // Save order
        Order savedOrder = orderRepository.save(order);
        
        // Publish domain event
        eventPublisher.publish(new OrderCreatedEvent(savedOrder.getId(), user.getId()));
        
        return savedOrder;
    }
}

// Domain Entities
public class Order {
    
    private String id;
    private Long userId;
    private List<OrderItem> items;
    private OrderStatus status;
    private BigDecimal totalAmount;
    private Instant createdAt;
    
    private Order(String id, Long userId, List<OrderItem> items) {
        this.id = id;
        this.userId = userId;
        this.items = new ArrayList<>(items);
        this.status = OrderStatus.CREATED;
        this.createdAt = Instant.now();
        calculateTotal();
    }
    
    public static Order create(Long userId, List<OrderItemRequest> itemRequests) {
        String orderId = generateOrderId();
        List<OrderItem> items = itemRequests.stream()
            .map(request -> OrderItem.create(request.getProductId(), request.getQuantity()))
            .collect(Collectors.toList());
        
        return new Order(orderId, userId, items);
    }
    
    private void calculateTotal() {
        this.totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    // Business methods
    public void confirm() {
        if (status != OrderStatus.CREATED) {
            throw new InvalidOrderStateException();
        }
        this.status = OrderStatus.CONFIRMED;
    }
    
    // Getters
    public String getId() { return id; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public List<OrderItem> getItems() { return Collections.unmodifiableList(items); }
    
    private static String generateOrderId() {
        return "ORD-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
}

// Repository Interfaces (Domain Layer)
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(String orderId);
    List<Order> findByUserId(Long userId);
}

public interface UserRepository {
    Optional<User> findById(Long userId);
    List<User> findByIds(List<Long> userIds);
}

public interface InventoryRepository {
    Product findProductById(Long productId);
    void reserveStock(Long productId, int quantity);
    void releaseStock(Long productId, int quantity);
}

// Infrastructure Implementation
@Repository
public class JpaOrderRepositoryImpl implements OrderRepository {
    
    @Autowired
    private OrderJpaRepository jpaRepository;
    
    @Override
    public Order save(Order order) {
        OrderEntity entity = OrderEntity.from(order);
        entity = jpaRepository.save(entity);
        return entity.toOrder();
    }
    
    @Override
    public Optional<Order> findById(String orderId) {
        return jpaRepository.findById(orderId)
            .map(OrderEntity::toOrder);
    }
    
    @Override
    public List<Order> findByUserId(Long userId) {
        return jpaRepository.findByUserId(userId).stream()
            .map(OrderEntity::toOrder)
            .collect(Collectors.toList());
    }
}

@Repository
public class RestUserRepositoryImpl implements UserRepository {
    
    @Autowired
    private UserServiceClient userServiceClient;
    
    @Override
    public Optional<User> findById(Long userId) {
        try {
            return Optional.of(userServiceClient.getUser(userId));
        } catch (UserNotFoundException e) {
            return Optional.empty();
        }
    }
    
    @Override
    public List<User> findByIds(List<Long> userIds) {
        return userServiceClient.getUsers(userIds);
    }
}
```

## Hexagonal Architecture (Ports & Adapters)

### Overview
Isolate the core business logic from external concerns through ports and adapters, enabling technology-agnostic business rules.

#### Core Business Logic (Domain)
```java
// Domain Entities
public class Product {
    
    private Long id;
    private String name;
    private String description;
    private BigDecimal price;
    private int stockQuantity;
    private ProductStatus status;
    
    public Product(Long id, String name, BigDecimal price, int stockQuantity) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.stockQuantity = stockQuantity;
        this.status = ProductStatus.ACTIVE;
    }
    
    public boolean hasEnoughStock(int requestedQuantity) {
        return stockQuantity >= requestedQuantity;
    }
    
    public void reduceStock(int quantity) {
        if (!hasEnoughStock(quantity)) {
            throw new InsufficientStockException(id, quantity, stockQuantity);
        }
        this.stockQuantity -= quantity;
    }
    
    public void increaseStock(int quantity) {
        this.stockQuantity += quantity;
    }
    
    // Getters and business methods
}

public class ShoppingCart {
    
    private final Long userId;
    private final List<CartItem> items;
    private BigDecimal totalAmount;
    
    public ShoppingCart(Long userId) {
        this.userId = userId;
        this.items = new ArrayList<>();
        this.totalAmount = BigDecimal.ZERO;
    }
    
    public void addItem(Product product, int quantity) {
        CartItem existingItem = findItemByProductId(product.getId());
        
        if (existingItem != null) {
            existingItem.increaseQuantity(quantity);
        } else {
            CartItem newItem = new CartItem(product, quantity);
            items.add(newItem);
        }
        
        recalculateTotal();
    }
    
    public void removeItem(Long productId, int quantity) {
        CartItem item = findItemByProductId(productId);
        if (item != null) {
            item.decreaseQuantity(quantity);
            if (item.getQuantity() <= 0) {
                items.remove(item);
            }
            recalculateTotal();
        }
    }
    
    public Order checkout() {
        if (items.isEmpty()) {
            throw new EmptyCartException(userId);
        }
        
        List<OrderItem> orderItems = items.stream()
            .map(cartItem -> new OrderItem(cartItem.getProduct(), cartItem.getQuantity()))
            .collect(Collectors.toList());
        
        return new Order(userId, orderItems, totalAmount);
    }
    
    private CartItem findItemByProductId(Long productId) {
        return items.stream()
            .filter(item -> item.getProduct().getId().equals(productId))
            .findFirst()
            .orElse(null);
    }
    
    private void recalculateTotal() {
        this.totalAmount = items.stream()
            .map(CartItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// Domain Services
public interface ProductService {
    Product getProduct(Long productId);
    void updateStock(Long productId, int newStock);
    List<Product> searchProducts(String query, int page, int size);
}

public interface ShoppingCartService {
    ShoppingCart getCart(Long userId);
    void addToCart(Long userId, Long productId, int quantity);
    void removeFromCart(Long userId, Long productId, int quantity);
    Order checkoutCart(Long userId);
}

public interface OrderService {
    Order createOrder(Order order);
    Order getOrder(String orderId);
    void updateOrderStatus(String orderId, OrderStatus status);
}

@Service
public class ShoppingCartServiceImpl implements ShoppingCartService {
    
    private final ProductService productService;
    private final OrderService orderService;
    private final ShoppingCartRepository cartRepository;
    
    public ShoppingCartServiceImpl(ProductService productService, 
                                 OrderService orderService,
                                 ShoppingCartRepository cartRepository) {
        this.productService = productService;
        this.orderService = orderService;
        this.cartRepository = cartRepository;
    }
    
    @Override
    public ShoppingCart getCart(Long userId) {
        return cartRepository.findByUserId(userId)
            .orElse(new ShoppingCart(userId));
    }
    
    @Override
    public void addToCart(Long userId, Long productId, int quantity) {
        Product product = productService.getProduct(productId);
        ShoppingCart cart = getCart(userId);
        
        cart.addItem(product, quantity);
        cartRepository.save(cart);
    }
    
    @Override
    public void removeFromCart(Long userId, Long productId, int quantity) {
        ShoppingCart cart = getCart(userId);
        cart.removeItem(productId, quantity);
        cartRepository.save(cart);
    }
    
    @Override
    public Order checkoutCart(Long userId) {
        ShoppingCart cart = getCart(userId);
        Order order = cart.checkout();
        
        // Validate stock availability
        for (OrderItem item : order.getItems()) {
            Product product = productService.getProduct(item.getProductId());
            if (!product.hasEnoughStock(item.getQuantity())) {
                throw new InsufficientStockException(product.getId(), 
                                                   item.getQuantity(), 
                                                   product.getStockQuantity());
            }
        }
        
        // Reserve stock
        for (OrderItem item : order.getItems()) {
            productService.updateStock(item.getProductId(), 
                productService.getProduct(item.getProductId()).getStockQuantity() - item.getQuantity());
        }
        
        // Create order
        Order createdOrder = orderService.createOrder(order);
        
        // Clear cart
        cartRepository.deleteByUserId(userId);
        
        return createdOrder;
    }
}
```

#### Ports (Interfaces)
```java
// Driving Ports (use cases that drive the application)
public interface ProductSearchPort {
    List<Product> searchProducts(String query, int page, int size);
    Product getProductDetails(Long productId);
}

public interface CartManagementPort {
    void addProductToCart(Long userId, Long productId, int quantity);
    void removeProductFromCart(Long userId, Long productId, int quantity);
    ShoppingCart getUserCart(Long userId);
    Order checkoutCart(Long userId);
}

public interface OrderManagementPort {
    Order createOrder(CreateOrderCommand command);
    Order getOrder(String orderId);
    List<Order> getUserOrders(Long userId);
}

// Driven Ports (interfaces that the application drives)
public interface ProductRepositoryPort {
    Product findById(Long productId);
    List<Product> search(String query, int page, int size);
    void updateStock(Long productId, int newStock);
    void save(Product product);
}

public interface ShoppingCartRepositoryPort {
    Optional<ShoppingCart> findByUserId(Long userId);
    void save(ShoppingCart cart);
    void deleteByUserId(Long userId);
}

public interface OrderRepositoryPort {
    Order save(Order order);
    Optional<Order> findById(String orderId);
    List<Order> findByUserId(Long userId);
}

public interface PaymentServicePort {
    PaymentResult processPayment(BigDecimal amount, PaymentDetails paymentDetails);
    PaymentResult getPaymentStatus(String paymentId);
    void refundPayment(String paymentId);
}

public interface NotificationServicePort {
    void sendOrderConfirmation(String orderId, Long userId);
    void sendPaymentConfirmation(String orderId, Long userId);
    void sendShippingNotification(String orderId, Long userId);
}
```

#### Adapters (Infrastructure)
```java
// REST API Adapter (Driving Adapter)
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    private final ProductSearchPort productSearchPort;
    
    public ProductController(ProductSearchPort productSearchPort) {
        this.productSearchPort = productSearchPort;
    }
    
    @GetMapping("/search")
    public ResponseEntity<List<ProductDto>> searchProducts(
            @RequestParam String query,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        List<Product> products = productSearchPort.searchProducts(query, page, size);
        List<ProductDto> dtos = products.stream()
            .map(ProductDto::from)
            .collect(Collectors.toList());
        
        return ResponseEntity.ok(dtos);
    }
    
    @GetMapping("/{productId}")
    public ResponseEntity<ProductDto> getProduct(@PathVariable Long productId) {
        Product product = productSearchPort.getProductDetails(productId);
        return ResponseEntity.ok(ProductDto.from(product));
    }
}

@RestController
@RequestMapping("/api/cart")
public class CartController {
    
    private final CartManagementPort cartManagementPort;
    
    public CartController(CartManagementPort cartManagementPort) {
        this.cartManagementPort = cartManagementPort;
    }
    
    @PostMapping("/items")
    public ResponseEntity<Void> addToCart(@RequestBody AddToCartRequest request) {
        cartManagementPort.addProductToCart(request.getUserId(), 
                                          request.getProductId(), 
                                          request.getQuantity());
        return ResponseEntity.ok().build();
    }
    
    @DeleteMapping("/items")
    public ResponseEntity<Void> removeFromCart(@RequestBody RemoveFromCartRequest request) {
        cartManagementPort.removeProductFromCart(request.getUserId(), 
                                               request.getProductId(), 
                                               request.getQuantity());
        return ResponseEntity.ok().build();
    }
    
    @GetMapping
    public ResponseEntity<ShoppingCartDto> getCart(@RequestParam Long userId) {
        ShoppingCart cart = cartManagementPort.getUserCart(userId);
        return ResponseEntity.ok(ShoppingCartDto.from(cart));
    }
    
    @PostMapping("/checkout")
    public ResponseEntity<OrderDto> checkoutCart(@RequestParam Long userId) {
        Order order = cartManagementPort.checkoutCart(userId);
        return ResponseEntity.ok(OrderDto.from(order));
    }
}

// Repository Adapters (Driven Adapters)
@Repository
public class JpaProductRepositoryAdapter implements ProductRepositoryPort {
    
    @Autowired
    private ProductJpaRepository jpaRepository;
    
    @Override
    public Product findById(Long productId) {
        return jpaRepository.findById(productId)
            .map(ProductEntity::toProduct)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }
    
    @Override
    public List<Product> search(String query, int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return jpaRepository.searchProducts(query, pageable).stream()
            .map(ProductEntity::toProduct)
            .collect(Collectors.toList());
    }
    
    @Override
    public void updateStock(Long productId, int newStock) {
        ProductEntity entity = jpaRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        entity.setStockQuantity(newStock);
        jpaRepository.save(entity);
    }
    
    @Override
    public void save(Product product) {
        ProductEntity entity = ProductEntity.from(product);
        jpaRepository.save(entity);
    }
}

@Repository
public class JpaShoppingCartRepositoryAdapter implements ShoppingCartRepositoryPort {
    
    @Autowired
    private ShoppingCartJpaRepository jpaRepository;
    
    @Override
    public Optional<ShoppingCart> findByUserId(Long userId) {
        return jpaRepository.findByUserId(userId)
            .map(ShoppingCartEntity::toShoppingCart);
    }
    
    @Override
    public void save(ShoppingCart cart) {
        ShoppingCartEntity entity = ShoppingCartEntity.from(cart);
        jpaRepository.save(entity);
    }
    
    @Override
    public void deleteByUserId(Long userId) {
        jpaRepository.deleteByUserId(userId);
    }
}

// External Service Adapters
@Service
public class StripePaymentServiceAdapter implements PaymentServicePort {
    
    @Value("${stripe.api.key}")
    private String apiKey;
    
    @Autowired
    private StripeClient stripeClient;
    
    @Override
    public PaymentResult processPayment(BigDecimal amount, PaymentDetails paymentDetails) {
        try {
            // Convert amount to cents
            long amountInCents = amount.multiply(BigDecimal.valueOf(100)).longValue();
            
            // Create payment intent
            PaymentIntent intent = PaymentIntent.create(
                PaymentIntentCreateParams.builder()
                    .setAmount(amountInCents)
                    .setCurrency("usd")
                    .setPaymentMethod(paymentDetails.getPaymentMethodId())
                    .setConfirm(true)
                    .build()
            );
            
            return new PaymentResult(intent.getId(), 
                                   intent.getStatus().equals("succeeded"),
                                   intent.getStatus());
            
        } catch (StripeException e) {
            logger.error("Payment processing failed", e);
            return new PaymentResult(null, false, e.getMessage());
        }
    }
    
    @Override
    public PaymentResult getPaymentStatus(String paymentId) {
        try {
            PaymentIntent intent = PaymentIntent.retrieve(paymentId);
            return new PaymentResult(paymentId, 
                                   intent.getStatus().equals("succeeded"),
                                   intent.getStatus());
        } catch (StripeException e) {
            return new PaymentResult(paymentId, false, e.getMessage());
        }
    }
    
    @Override
    public void refundPayment(String paymentId) {
        try {
            Refund.create(
                RefundCreateParams.builder()
                    .setPaymentIntent(paymentId)
                    .build()
            );
        } catch (StripeException e) {
            logger.error("Refund failed for payment {}", paymentId, e);
            throw new PaymentRefundException(paymentId, e);
        }
    }
}

@Service
public class EmailNotificationServiceAdapter implements NotificationServicePort {
    
    @Autowired
    private JavaMailSender mailSender;
    
    @Autowired
    private TemplateEngine templateEngine;
    
    @Value("${app.email.from}")
    private String fromEmail;
    
    @Override
    public void sendOrderConfirmation(String orderId, Long userId) {
        // Get user email and order details
        User user = userRepository.findById(userId).orElseThrow();
        Order order = orderRepository.findById(orderId).orElseThrow();
        
        // Prepare email context
        Context context = new Context();
        context.setVariable("user", user);
        context.setVariable("order", order);
        
        // Render template
        String htmlContent = templateEngine.process("order-confirmation-email", context);
        
        // Send email
        sendEmail(user.getEmail(), "Order Confirmation - " + orderId, htmlContent);
    }
    
    @Override
    public void sendPaymentConfirmation(String orderId, Long userId) {
        // Similar implementation for payment confirmation
    }
    
    @Override
    public void sendShippingNotification(String orderId, Long userId) {
        // Similar implementation for shipping notification
    }
    
    private void sendEmail(String to, String subject, String htmlContent) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            
            helper.setFrom(fromEmail);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(htmlContent, true);
            
            mailSender.send(message);
            
        } catch (MessagingException e) {
            logger.error("Failed to send email to {}", to, e);
        }
    }
}
```

## CQRS and Event Sourcing Architecture

### Overview
CQRS separates read and write operations, while Event Sourcing stores state changes as events.

#### CQRS Implementation
```java
// Commands (Write Operations)
public interface Command {
    String getAggregateId();
}

public class CreateProductCommand implements Command {
    
    private final String productId;
    private final String name;
    private final BigDecimal price;
    private final String description;
    
    public CreateProductCommand(String productId, String name, BigDecimal price, String description) {
        this.productId = productId;
        this.name = name;
        this.price = price;
        this.description = description;
    }
    
    @Override
    public String getAggregateId() {
        return productId;
    }
    
    // Getters
}

public class UpdateProductPriceCommand implements Command {
    
    private final String productId;
    private final BigDecimal newPrice;
    
    public UpdateProductPriceCommand(String productId, BigDecimal newPrice) {
        this.productId = productId;
        this.newPrice = newPrice;
    }
    
    @Override
    public String getAggregateId() {
        return productId;
    }
    
    // Getters
}

// Command Handlers
@Component
public class CreateProductCommandHandler implements CommandHandler<CreateProductCommand> {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Override
    public void handle(CreateProductCommand command) {
        Product product = new Product(command.getProductId(), 
                                    command.getName(), 
                                    command.getPrice());
        product.setDescription(command.getDescription());
        
        productRepository.save(product);
    }
}

@Component
public class UpdateProductPriceCommandHandler implements CommandHandler<UpdateProductPriceCommand> {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Override
    public void handle(UpdateProductPriceCommand command) {
        Product product = productRepository.findById(command.getProductId())
            .orElseThrow(() -> new ProductNotFoundException(command.getProductId()));
        
        product.updatePrice(command.getNewPrice());
        productRepository.save(product);
    }
}

// Queries (Read Operations)
public interface Query<R> {
    // Marker interface for queries
}

public class GetProductByIdQuery implements Query<ProductDto> {
    
    private final String productId;
    
    public GetProductByIdQuery(String productId) {
        this.productId = productId;
    }
    
    // Getters
}

public class SearchProductsQuery implements Query<List<ProductDto>> {
    
    private final String searchTerm;
    private final int page;
    private final int size;
    private final String sortBy;
    private final String sortDirection;
    
    public SearchProductsQuery(String searchTerm, int page, int size, 
                             String sortBy, String sortDirection) {
        this.searchTerm = searchTerm;
        this.page = page;
        this.size = size;
        this.sortBy = sortBy;
        this.sortDirection = sortDirection;
    }
    
    // Getters
}

// Query Handlers
@Component
public class GetProductByIdQueryHandler implements QueryHandler<GetProductByIdQuery, ProductDto> {
    
    @Autowired
    private ProductReadRepository readRepository;
    
    @Override
    public ProductDto handle(GetProductByIdQuery query) {
        return readRepository.findProductDtoById(query.getProductId());
    }
}

@Component
public class SearchProductsQueryHandler implements QueryHandler<SearchProductsQuery, List<ProductDto>> {
    
    @Autowired
    private ProductReadRepository readRepository;
    
    @Override
    public List<ProductDto> handle(SearchProductsQuery query) {
        return readRepository.searchProducts(query.getSearchTerm(), 
                                           query.getPage(), 
                                           query.getSize(),
                                           query.getSortBy(),
                                           query.getSortDirection());
    }
}

// Command Bus
@Service
public class CommandBus {
    
    @Autowired
    private Map<String, CommandHandler> commandHandlers;
    
    public <C extends Command> void send(C command) {
        String commandType = command.getClass().getSimpleName();
        CommandHandler<C> handler = commandHandlers.get(commandType);
        
        if (handler == null) {
            throw new UnsupportedCommandException(commandType);
        }
        
        handler.handle(command);
    }
}

// Query Bus
@Service
public class QueryBus {
    
    @Autowired
    private Map<String, QueryHandler> queryHandlers;
    
    public <Q extends Query<R>, R> R send(Q query) {
        String queryType = query.getClass().getSimpleName();
        QueryHandler<Q, R> handler = (QueryHandler<Q, R>) queryHandlers.get(queryType);
        
        if (handler == null) {
            throw new UnsupportedQueryException(queryType);
        }
        
        return handler.handle(query);
    }
}

// REST API Layer
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private CommandBus commandBus;
    
    @Autowired
    private QueryBus queryBus;
    
    @PostMapping
    public ResponseEntity<Void> createProduct(@RequestBody CreateProductRequest request) {
        String productId = UUID.randomUUID().toString();
        
        CreateProductCommand command = new CreateProductCommand(
            productId,
            request.getName(),
            request.getPrice(),
            request.getDescription()
        );
        
        commandBus.send(command);
        
        return ResponseEntity.created(URI.create("/api/products/" + productId)).build();
    }
    
    @PutMapping("/{productId}/price")
    public ResponseEntity<Void> updatePrice(@PathVariable String productId, 
                                          @RequestBody UpdatePriceRequest request) {
        
        UpdateProductPriceCommand command = new UpdateProductPriceCommand(
            productId, request.getNewPrice());
        
        commandBus.send(command);
        
        return ResponseEntity.ok().build();
    }
    
    @GetMapping("/{productId}")
    public ResponseEntity<ProductDto> getProduct(@PathVariable String productId) {
        GetProductByIdQuery query = new GetProductByIdQuery(productId);
        ProductDto product = queryBus.send(query);
        
        return ResponseEntity.ok(product);
    }
    
    @GetMapping("/search")
    public ResponseEntity<List<ProductDto>> searchProducts(
            @RequestParam String q,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "name") String sortBy,
            @RequestParam(defaultValue = "asc") String sortDirection) {
        
        SearchProductsQuery query = new SearchProductsQuery(q, page, size, sortBy, sortDirection);
        List<ProductDto> products = queryBus.send(query);
        
        return ResponseEntity.ok(products);
    }
}
```

## Microservices Architecture

### Overview
Decompose applications into small, independent services that communicate through APIs.

#### Service Decomposition Patterns
```java
// API Gateway Service
@SpringBootApplication
@EnableEurekaClient
public class ApiGatewayApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}

@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // User service routes
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .rewritePath("/api/users/(?<segment>.*)", "/${segment}")
                    .circuitBreaker(c -> c
                        .setName("user-service-circuit")
                        .setFallbackUri("forward:/fallback/users"))
                    .requestRateLimiter(c -> c
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver())))
            // Product service routes
            .route("product-service", r -> r
                .path("/api/products/**")
                .filters(f -> f
                    .rewritePath("/api/products/(?<segment>.*)", "/${segment}")
                    .circuitBreaker(c -> c
                        .setName("product-service-circuit")
                        .setFallbackUri("forward:/fallback/products")))
            // Order service routes
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .rewritePath("/api/orders/(?<segment>.*)", "/${segment}")
                    .circuitBreaker(c -> c
                        .setName("order-service-circuit")
                        .setFallbackUri("forward:/fallback/orders")))
            .build();
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(10, 20, 1);
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getHeaders().getFirst("X-User-Id"));
    }
}

// User Service
@Service
@RestController
@RequestMapping("/users")
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody CreateUserRequest request) {
        User user = new User(request.getEmail(), request.getName());
        user = userRepository.save(user);
        
        // Publish event
        eventPublisher.publish("user-events", 
            new UserCreatedEvent(user.getId(), user.getEmail()));
        
        return ResponseEntity.created(URI.create("/users/" + user.getId()))
                           .body(user);
    }
    
    @GetMapping("/{userId}")
    public ResponseEntity<User> getUser(@PathVariable Long userId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        return ResponseEntity.ok(user);
    }
    
    @GetMapping
    public ResponseEntity<List<User>> getUsers(@RequestParam List<Long> ids) {
        List<User> users = userRepository.findAllById(ids);
        return ResponseEntity.ok(users);
    }
}

// Product Service
@Service
@RestController
@RequestMapping("/products")
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private SearchService searchService;
    
    @PostMapping
    public ResponseEntity<Product> createProduct(@RequestBody CreateProductRequest request) {
        Product product = new Product(request.getName(), request.getPrice());
        product.setDescription(request.getDescription());
        product.setStockQuantity(request.getStockQuantity());
        
        product = productRepository.save(product);
        
        // Index for search
        searchService.indexProduct(product);
        
        return ResponseEntity.created(URI.create("/products/" + product.getId()))
                           .body(product);
    }
    
    @GetMapping("/{productId}")
    public ResponseEntity<Product> getProduct(@PathVariable Long productId) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        return ResponseEntity.ok(product);
    }
    
    @GetMapping("/search")
    public ResponseEntity<List<Product>> searchProducts(@RequestParam String q) {
        List<Product> products = searchService.searchProducts(q);
        return ResponseEntity.ok(products);
    }
    
    @PutMapping("/{productId}/stock")
    public ResponseEntity<Void> updateStock(@PathVariable Long productId, 
                                          @RequestParam int quantity) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        product.setStockQuantity(quantity);
        productRepository.save(product);
        
        return ResponseEntity.ok().build();
    }
}

// Order Service
@Service
@RestController
@RequestMapping("/orders")
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private UserServiceClient userServiceClient;
    
    @Autowired
    private ProductServiceClient productServiceClient;
    
    @Autowired
    private PaymentServiceClient paymentServiceClient;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest request) {
        // Validate user exists
        User user = userServiceClient.getUser(request.getUserId());
        
        // Validate products and calculate total
        BigDecimal totalAmount = BigDecimal.ZERO;
        List<OrderItem> orderItems = new ArrayList<>();
        
        for (OrderItemRequest itemRequest : request.getItems()) {
            Product product = productServiceClient.getProduct(itemRequest.getProductId());
            
            if (product.getStockQuantity() < itemRequest.getQuantity()) {
                throw new InsufficientInventoryException(itemRequest.getProductId());
            }
            
            OrderItem orderItem = new OrderItem(product.getId(), product.getName(), 
                                              itemRequest.getQuantity(), product.getPrice());
            orderItems.add(orderItem);
            
            totalAmount = totalAmount.add(orderItem.getSubtotal());
        }
        
        // Create order
        Order order = new Order(request.getUserId(), orderItems, totalAmount);
        order = orderRepository.save(order);
        
        // Reserve inventory
        for (OrderItem item : orderItems) {
            productServiceClient.reserveStock(item.getProductId(), item.getQuantity());
        }
        
        // Process payment
        PaymentResult payment = paymentServiceClient.processPayment(order.getId(), totalAmount);
        if (!payment.isSuccessful()) {
            // Rollback inventory reservation
            for (OrderItem item : orderItems) {
                productServiceClient.releaseStock(item.getProductId(), item.getQuantity());
            }
            throw new PaymentFailedException("Payment processing failed");
        }
        
        order.setStatus(OrderStatus.CONFIRMED);
        order = orderRepository.save(order);
        
        // Publish order created event
        eventPublisher.publish("order-events", 
            new OrderCreatedEvent(order.getId(), request.getUserId(), totalAmount));
        
        return ResponseEntity.created(URI.create("/orders/" + order.getId()))
                           .body(order);
    }
    
    @GetMapping("/{orderId}")
    public ResponseEntity<Order> getOrder(@PathVariable String orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        return ResponseEntity.ok(order);
    }
    
    @GetMapping
    public ResponseEntity<List<Order>> getUserOrders(@RequestParam Long userId) {
        List<Order> orders = orderRepository.findByUserIdOrderByCreatedAtDesc(userId);
        return ResponseEntity.ok(orders);
    }
}

// Service Clients
@Service
public class UserServiceClient {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Value("${user.service.url}")
    private String userServiceUrl;
    
    public User getUser(Long userId) {
        String url = userServiceUrl + "/users/" + userId;
        return restTemplate.getForObject(url, User.class);
    }
}

@Service
public class ProductServiceClient {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Value("${product.service.url}")
    private String productServiceUrl;
    
    public Product getProduct(Long productId) {
        String url = productServiceUrl + "/products/" + productId;
        return restTemplate.getForObject(url, Product.class);
    }
    
    public void reserveStock(Long productId, int quantity) {
        String url = productServiceUrl + "/products/" + productId + "/stock/reserve";
        restTemplate.put(url, Map.of("quantity", quantity));
    }
    
    public void releaseStock(Long productId, int quantity) {
        String url = productServiceUrl + "/products/" + productId + "/stock/release";
        restTemplate.put(url, Map.of("quantity", quantity));
    }
}

@Service
public class PaymentServiceClient {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Value("${payment.service.url}")
    private String paymentServiceUrl;
    
    public PaymentResult processPayment(String orderId, BigDecimal amount) {
        String url = paymentServiceUrl + "/payments";
        PaymentRequest request = new PaymentRequest(orderId, amount);
        
        ResponseEntity<PaymentResult> response = restTemplate.postForEntity(url, request, PaymentResult.class);
        return response.getBody();
    }
}
```

## Event-Driven Architecture

### Overview
Systems communicate through events, enabling loose coupling and scalability.

#### Event-Driven Implementation
```java
// Domain Events
public interface DomainEvent {
    String getEventType();
    Instant getTimestamp();
    Map<String, Object> getData();
}

public class OrderCreatedEvent implements DomainEvent {
    
    private final String eventType = "OrderCreated";
    private final String orderId;
    private final Long userId;
    private final BigDecimal totalAmount;
    private final Instant timestamp;
    
    public OrderCreatedEvent(String orderId, Long userId, BigDecimal totalAmount) {
        this.orderId = orderId;
        this.userId = userId;
        this.totalAmount = totalAmount;
        this.timestamp = Instant.now();
    }
    
    @Override
    public String getEventType() { return eventType; }
    
    @Override
    public Instant getTimestamp() { return timestamp; }
    
    @Override
    public Map<String, Object> getData() {
        return Map.of(
            "orderId", orderId,
            "userId", userId,
            "totalAmount", totalAmount
        );
    }
}

// Event Publisher
@Service
public class EventPublisher {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    public void publish(String topic, DomainEvent event) {
        try {
            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send(topic, event.getEventType(), eventJson);
            
            logger.info("Published event {} to topic {}", event.getEventType(), topic);
            
        } catch (JsonProcessingException e) {
            logger.error("Failed to serialize event {}", event.getEventType(), e);
            throw new EventPublishingException(event, e);
        }
    }
    
    public void publishAsync(String topic, DomainEvent event) {
        CompletableFuture<SendResult<String, Object>> future = 
            kafkaTemplate.send(topic, event.getEventType(), event);
        
        future.whenComplete((result, exception) -> {
            if (exception != null) {
                logger.error("Failed to publish event {} asynchronously", 
                           event.getEventType(), exception);
            } else {
                logger.info("Successfully published event {} to topic {}", 
                          event.getEventType(), topic);
            }
        });
    }
}

// Event Consumers
@Service
public class OrderEventConsumer {
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private AnalyticsService analyticsService;
    
    @KafkaListener(topics = "order-events", groupId = "order-processing")
    public void handleOrderEvents(String eventJson) {
        try {
            // Deserialize event
            JsonNode eventNode = objectMapper.readTree(eventJson);
            String eventType = eventNode.get("eventType").asText();
            
            switch (eventType) {
                case "OrderCreated":
                    handleOrderCreated(objectMapper.treeToValue(eventNode, OrderCreatedEvent.class));
                    break;
                case "OrderConfirmed":
                    handleOrderConfirmed(objectMapper.treeToValue(eventNode, OrderConfirmedEvent.class));
                    break;
                case "OrderShipped":
                    handleOrderShipped(objectMapper.treeToValue(eventNode, OrderShippedEvent.class));
                    break;
                default:
                    logger.warn("Unknown event type: {}", eventType);
            }
            
        } catch (Exception e) {
            logger.error("Failed to process order event", e);
            // Could send to dead letter queue
        }
    }
    
    private void handleOrderCreated(OrderCreatedEvent event) {
        // Send order confirmation notification
        notificationService.sendOrderConfirmation(event.getOrderId(), event.getUserId());
        
        // Update analytics
        analyticsService.recordOrderCreated(event.getUserId(), event.getTotalAmount());
        
        logger.info("Processed OrderCreated event for order {}", event.getOrderId());
    }
    
    private void handleOrderConfirmed(OrderConfirmedEvent event) {
        // Reserve inventory (if not already done)
        // Update analytics
        analyticsService.recordOrderConfirmed(event.getOrderId());
        
        logger.info("Processed OrderConfirmed event for order {}", event.getOrderId());
    }
    
    private void handleOrderShipped(OrderShippedEvent event) {
        // Send shipping notification
        notificationService.sendShippingNotification(event.getOrderId(), event.getUserId());
        
        // Update inventory
        inventoryService.updateInventoryOnShipment(event.getOrderId());
        
        // Update analytics
        analyticsService.recordOrderShipped(event.getOrderId());
        
        logger.info("Processed OrderShipped event for order {}", event.getOrderId());
    }
}

@Service
public class UserEventConsumer {
    
    @Autowired
    private UserAnalyticsService analyticsService;
    
    @Autowired
    private RecommendationService recommendationService;
    
    @KafkaListener(topics = "user-events", groupId = "user-processing")
    public void handleUserEvents(String eventJson) {
        try {
            JsonNode eventNode = objectMapper.readTree(eventJson);
            String eventType = eventNode.get("eventType").asText();
            
            switch (eventType) {
                case "UserRegistered":
                    handleUserRegistered(objectMapper.treeToValue(eventNode, UserRegisteredEvent.class));
                    break;
                case "UserLoggedIn":
                    handleUserLoggedIn(objectMapper.treeToValue(eventNode, UserLoggedInEvent.class));
                    break;
                case "UserProfileUpdated":
                    handleUserProfileUpdated(objectMapper.treeToValue(eventNode, UserProfileUpdatedEvent.class));
                    break;
            }
            
        } catch (Exception e) {
            logger.error("Failed to process user event", e);
        }
    }
    
    private void handleUserRegistered(UserRegisteredEvent event) {
        // Initialize user analytics
        analyticsService.initializeUserAnalytics(event.getUserId());
        
        // Generate initial recommendations
        recommendationService.generateInitialRecommendations(event.getUserId());
        
        logger.info("Processed UserRegistered event for user {}", event.getUserId());
    }
    
    private void handleUserLoggedIn(UserLoggedInEvent event) {
        // Update user activity analytics
        analyticsService.recordUserLogin(event.getUserId(), event.getTimestamp());
        
        logger.info("Processed UserLoggedIn event for user {}", event.getUserId());
    }
    
    private void handleUserProfileUpdated(UserProfileUpdatedEvent event) {
        // Update recommendation preferences
        recommendationService.updateUserPreferences(event.getUserId(), event.getPreferences());
        
        logger.info("Processed UserProfileUpdated event for user {}", event.getUserId());
    }
}

// Event Sourcing with CQRS
@Service
public class EventSourcedAggregateRepository {
    
    @Autowired
    private EventStore eventStore;
    
    @Autowired
    private SnapshotStore snapshotStore;
    
    public <T extends AggregateRoot> T load(Class<T> aggregateClass, String aggregateId) {
        // Try to load from snapshot first
        Optional<Snapshot> snapshot = snapshotStore.findLatestSnapshot(aggregateId);
        
        T aggregate;
        int startVersion;
        
        if (snapshot.isPresent()) {
            // Reconstruct from snapshot
            aggregate = reconstructFromSnapshot(snapshot.get(), aggregateClass);
            startVersion = snapshot.get().getVersion() + 1;
        } else {
            // Create new aggregate
            aggregate = createNewAggregate(aggregateClass, aggregateId);
            startVersion = 1;
        }
        
        // Apply remaining events
        List<DomainEvent> events = eventStore.findByAggregateIdAndVersionGreaterThan(
            aggregateId, startVersion - 1);
        
        for (DomainEvent event : events) {
            aggregate.applyEvent(event);
        }
        
        return aggregate;
    }
    
    public void save(AggregateRoot aggregate) {
        // Save uncommitted events
        for (DomainEvent event : aggregate.getUncommittedEvents()) {
            eventStore.save(event);
        }
        
        // Create snapshot if needed
        if (shouldCreateSnapshot(aggregate)) {
            createSnapshot(aggregate);
        }
        
        aggregate.markEventsAsCommitted();
    }
    
    private boolean shouldCreateSnapshot(AggregateRoot aggregate) {
        // Create snapshot every 50 events
        return aggregate.getVersion() % 50 == 0;
    }
    
    private void createSnapshot(AggregateRoot aggregate) {
        Snapshot snapshot = new Snapshot(
            aggregate.getId(),
            aggregate.getVersion(),
            serializeAggregate(aggregate),
            Instant.now()
        );
        
        snapshotStore.save(snapshot);
    }
    
    private <T> T reconstructFromSnapshot(Snapshot snapshot, Class<T> aggregateClass) {
        return deserializeAggregate(snapshot.getData(), aggregateClass);
    }
    
    private <T> T createNewAggregate(Class<T> aggregateClass, String aggregateId) {
        // Use reflection or factory method to create aggregate
        return null; // Implementation would create aggregate
    }
    
    private String serializeAggregate(AggregateRoot aggregate) {
        // Serialize aggregate state
        return null; // Implementation would serialize
    }
    
    private <T> T deserializeAggregate(String data, Class<T> aggregateClass) {
        // Deserialize aggregate state
        return null; // Implementation would deserialize
    }
}
```

## Serverless Architecture

### Overview
Run code in response to events without managing servers.

#### AWS Lambda Implementation
```java
public class OrderProcessingLambda implements RequestHandler<OrderProcessingRequest, OrderProcessingResponse> {
    
    private final OrderService orderService;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final NotificationService notificationService;
    
    public OrderProcessingLambda() {
        // Initialize services (could use dependency injection framework)
        DynamoDBMapper mapper = new DynamoDBMapper(DynamoDbClient.create());
        this.orderService = new OrderService(mapper);
        this.paymentService = new PaymentService();
        this.inventoryService = new InventoryService(mapper);
        this.notificationService = new NotificationService();
    }
    
    @Override
    public OrderProcessingResponse handleRequest(OrderProcessingRequest request, Context context) {
        LambdaLogger logger = context.getLogger();
        
        try {
            logger.log("Processing order: " + request.getOrderId());
            
            // Validate order
            Order order = orderService.getOrder(request.getOrderId());
            if (order == null) {
                throw new OrderNotFoundException(request.getOrderId());
            }
            
            // Check inventory
            for (OrderItem item : order.getItems()) {
                if (!inventoryService.isAvailable(item.getProductId(), item.getQuantity())) {
                    throw new InsufficientInventoryException(item.getProductId());
                }
            }
            
            // Process payment
            PaymentResult payment = paymentService.processPayment(order.getTotalAmount(), 
                                                                request.getPaymentDetails());
            if (!payment.isSuccessful()) {
                throw new PaymentFailedException("Payment failed: " + payment.getErrorMessage());
            }
            
            // Reserve inventory
            for (OrderItem item : order.getItems()) {
                inventoryService.reserveStock(item.getProductId(), item.getQuantity());
            }
            
            // Update order status
            order.setStatus(OrderStatus.CONFIRMED);
            order.setPaymentId(payment.getTransactionId());
            orderService.updateOrder(order);
            
            // Send notifications
            notificationService.sendOrderConfirmation(order);
            
            logger.log("Successfully processed order: " + request.getOrderId());
            
            return new OrderProcessingResponse(order.getId(), OrderStatus.CONFIRMED, null);
            
        } catch (Exception e) {
            logger.log("Failed to process order: " + request.getOrderId() + ", error: " + e.getMessage());
            
            // Update order status to failed
            try {
                Order order = orderService.getOrder(request.getOrderId());
                if (order != null) {
                    order.setStatus(OrderStatus.FAILED);
                    order.setErrorMessage(e.getMessage());
                    orderService.updateOrder(order);
                }
            } catch (Exception updateException) {
                logger.log("Failed to update order status: " + updateException.getMessage());
            }
            
            return new OrderProcessingResponse(request.getOrderId(), OrderStatus.FAILED, e.getMessage());
        }
    }
}

// API Gateway Integration
@RestController
public class OrderController {
    
    @Autowired
    private LambdaInvoker lambdaInvoker;
    
    @PostMapping("/orders/{orderId}/process")
    public ResponseEntity<OrderProcessingResponse> processOrder(
            @PathVariable String orderId,
            @RequestBody PaymentDetails paymentDetails) {
        
        OrderProcessingRequest request = new OrderProcessingRequest(orderId, paymentDetails);
        
        // Invoke Lambda function
        OrderProcessingResponse response = lambdaInvoker.invoke("order-processing-function", 
                                                              OrderProcessingResponse.class, 
                                                              request);
        
        if (response.getStatus() == OrderStatus.CONFIRMED) {
            return ResponseEntity.ok(response);
        } else {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                               .body(response);
        }
    }
}

// Step Functions for Complex Workflows
public class OrderFulfillmentWorkflow {
    
    public static void main(String[] args) {
        // Define state machine
        StateMachine orderFulfillmentStateMachine = StateMachine.builder()
            .comment("Order fulfillment workflow")
            .startAt("ValidateOrder")
            .state("ValidateOrder", Task.builder()
                .resource("arn:aws:lambda:us-east-1:123456789012:function:validate-order")
                .next("ProcessPayment")
                .catchError("OrderFailed"))
            .state("ProcessPayment", Task.builder()
                .resource("arn:aws:lambda:us-east-1:123456789012:function:process-payment")
                .next("ReserveInventory")
                .catchError("PaymentFailed"))
            .state("ReserveInventory", Task.builder()
                .resource("arn:aws:lambda:us-east-1:123456789012:function:reserve-inventory")
                .next("ShipOrder")
                .catchError("InventoryFailed"))
            .state("ShipOrder", Task.builder()
                .resource("arn:aws:lambda:us-east-1:123456789012:function:ship-order")
                .next("OrderCompleted")
                .catchError("ShippingFailed"))
            .state("OrderCompleted", Succeed.builder().build())
            .state("OrderFailed", Fail.builder()
                .error("OrderValidationFailed")
                .cause("Order validation failed"))
            .state("PaymentFailed", Fail.builder()
                .error("PaymentProcessingFailed")
                .cause("Payment processing failed"))
            .state("InventoryFailed", Fail.builder()
                .error("InventoryReservationFailed")
                .cause("Inventory reservation failed"))
            .state("ShippingFailed", Fail.builder()
                .error("ShippingFailed")
                .cause("Shipping failed"))
            .build();
        
        // Deploy state machine
        StepFunctionsClient client = StepFunctionsClient.create();
        client.createStateMachine(CreateStateMachineRequest.builder()
            .name("OrderFulfillmentWorkflow")
            .definition(orderFulfillmentStateMachine.toJson())
            .roleArn("arn:aws:iam::123456789012:role/StepFunctionsRole")
            .build());
    }
}

// Event-Driven Serverless Functions
public class OrderEventHandler implements RequestHandler<SQSEvent, Void> {
    
    private final OrderService orderService;
    private final NotificationService notificationService;
    
    public OrderEventHandler() {
        // Initialize services
        this.orderService = new OrderService();
        this.notificationService = new NotificationService();
    }
    
    @Override
    public Void handleRequest(SQSEvent sqsEvent, Context context) {
        LambdaLogger logger = context.getLogger();
        
        for (SQSEvent.SQSMessage message : sqsEvent.getRecords()) {
            try {
                // Parse order event
                OrderEvent orderEvent = parseOrderEvent(message.getBody());
                
                // Process event based on type
                switch (orderEvent.getEventType()) {
                    case "ORDER_CREATED":
                        handleOrderCreated((OrderCreatedEvent) orderEvent);
                        break;
                    case "ORDER_CONFIRMED":
                        handleOrderConfirmed((OrderConfirmedEvent) orderEvent);
                        break;
                    case "ORDER_SHIPPED":
                        handleOrderShipped((OrderShippedEvent) orderEvent);
                        break;
                }
                
                logger.log("Successfully processed order event: " + orderEvent.getOrderId());
                
            } catch (Exception e) {
                logger.log("Failed to process order event: " + e.getMessage());
                // Send to dead letter queue or retry
                throw e; // Lambda will retry or send to DLQ
            }
        }
        
        return null;
    }
    
    private void handleOrderCreated(OrderCreatedEvent event) {
        // Send order confirmation
        notificationService.sendOrderConfirmation(event.getOrderId(), event.getUserId());
    }
    
    private void handleOrderConfirmed(OrderConfirmedEvent event) {
        // Update inventory
        orderService.updateInventoryForOrder(event.getOrderId());
    }
    
    private void handleOrderShipped(OrderShippedEvent event) {
        // Send shipping notification
        notificationService.sendShippingNotification(event.getOrderId(), event.getUserId());
        
        // Update analytics
        analyticsService.recordOrderShipped(event.getOrderId());
    }
    
    private OrderEvent parseOrderEvent(String messageBody) {
        // Parse JSON message to event object
        return objectMapper.readValue(messageBody, OrderEvent.class);
    }
}
```

## Best Practices

### Architectural Decision Records
```java
public class ArchitecturalDecisionRecord {
    
    private final String title;
    private final String context;
    private final String decision;
    private final String consequences;
    private final LocalDate date;
    private final String status; // PROPOSED, ACCEPTED, REJECTED, DEPRECATED
    
    public ArchitecturalDecisionRecord(String title, String context, 
                                     String decision, String consequences) {
        this.title = title;
        this.context = context;
        this.decision = decision;
        this.consequences = consequences;
        this.date = LocalDate.now();
        this.status = "PROPOSED";
    }
    
    // Example ADR for microservices decision
    public static ArchitecturalDecisionRecord createMicroservicesADR() {
        return new ArchitecturalDecisionRecord(
            "Adopt Microservices Architecture",
            "The monolithic application has grown too large and complex, " +
            "making it difficult to deploy changes and scale individual features.",
            "Decompose the monolithic application into microservices based on " +
            "business capabilities. Each service will have its own database " +
            "and communicate via REST APIs and message queues.",
            "Positive:\n" +
            "- Independent deployment and scaling\n" +
            "- Technology diversity\n" +
            "- Fault isolation\n" +
            "Negative:\n" +
            "- Increased complexity in distributed systems\n" +
            "- Eventual consistency challenges\n" +
            "- Operational overhead\n" +
            "Risks:\n" +
            "- Service discovery and communication complexity\n" +
            "- Distributed transaction management\n" +
            "- Monitoring and debugging challenges"
        );
    }
    
    public static ArchitecturalDecisionRecord createCQRSADR() {
        return new ArchitecturalDecisionRecord(
            "Implement CQRS Pattern",
            "The application has complex read models with different performance " +
            "requirements than write operations, leading to slow queries and " +
            "complex domain models.",
            "Separate read and write operations using the CQRS pattern. " +
            "Write operations will use domain-driven design with aggregates, " +
            "while read operations will use optimized query models and views.",
            "Positive:\n" +
            "- Optimized read and write performance\n" +
            "- Independent scaling of read and write workloads\n" +
            "- Simplified domain models\n" +
            "Negative:\n" +
            "- Increased complexity in maintaining two models\n" +
            "- Eventual consistency between read and write models\n" +
            "- Additional infrastructure for synchronization\n" +
            "Risks:\n" +
            "- Complexity in keeping models synchronized\n" +
            "- Learning curve for development team"
        );
    }
}
```

## Conclusion

Architectural patterns provide high-level structural guidance for building scalable, maintainable software systems. Each pattern has specific trade-offs and should be chosen based on project requirements, team capabilities, and organizational context.

**Key Takeaways:**
- **Layered Architecture**: Provides clear separation of concerns and is easy to understand
- **Hexagonal Architecture**: Enables technology-agnostic business logic through ports and adapters
- **CQRS/ES**: Optimizes performance through separate read/write models and comprehensive audit trails
- **Microservices**: Enables independent scaling and deployment but increases operational complexity
- **Event-Driven**: Promotes loose coupling and scalability through asynchronous communication
- **Serverless**: Reduces operational overhead but requires careful cost and performance monitoring

**Choosing the Right Pattern:**
- **Start Simple**: Begin with layered architecture and evolve as complexity grows
- **Consider Trade-offs**: Each pattern solves specific problems while introducing new challenges
- **Team Experience**: Choose patterns that match your team's skills and experience
- **Business Requirements**: Align architectural decisions with business goals and constraints
- **Evolution**: Be prepared to refactor or evolve the architecture as requirements change

Effective use of architectural patterns requires understanding both their benefits and limitations, as well as the ability to adapt them to specific project contexts.
