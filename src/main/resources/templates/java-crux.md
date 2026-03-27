Below is the enhanced Markdown file. For each Q&A, I've added a concise **“Main Crux / Why‑When‑How”** section that distills the essential reasoning, use cases, and application steps. The original content remains untouched—the additions are inserted right after the code examples, before any existing “Best Practice” notes.

```markdown
# Java Interview Preparation Guide

## Part 1: Object-Oriented Design & Language Fundamentals

### 1. How do you design immutable classes correctly in production?

**Practical Example:**
```java
public final class User {
    private final String name;
    private final List<String> roles;
    private final LocalDate createdAt;

    public User(String name, List<String> roles, LocalDate createdAt) {
        this.name = name;
        // Defensive copy for mutable collections
        this.roles = List.copyOf(roles);  // Java 10+ immutable copy
        this.createdAt = createdAt;        // LocalDate is immutable
    }

    public String getName() { return name; }
    public List<String> getRoles() { return roles; }  // Already immutable
    public LocalDate getCreatedAt() { return createdAt; }
}
```

**Best Practice:** Use `record` (Java 14+) for simple data carriers — they're immutable by design:
```java
public record UserRecord(String name, List<String> roles, LocalDate createdAt) {
    public UserRecord {
        roles = List.copyOf(roles);  // Normalizing constructor
    }
}
```

**Why:** Immutable objects are thread‑safe, cacheable, and prevent defensive copying bugs.  
**When to use:** Use for value objects, DTOs, configuration data, or any object that should not change after creation.  
**How to use:** Mark the class `final`, make all fields `final`, don’t provide setters, and defensively copy mutable parameters (collections, arrays) in the constructor. Use `record` for simple carriers.

---

### 2. Checked vs unchecked exceptions — when to use which with practical examples?

**Practical Example:**
```java
// Checked exception — caller must handle or declare
public class InsufficientFundsException extends Exception {
    public InsufficientFundsException(String message) {
        super(message);
    }
}

// Unchecked exception — programming error, not recoverable
public class InvalidTransferAmountException extends RuntimeException {
    public InvalidTransferAmountException(String message) {
        super(message);
    }
}

// Usage in service layer
public void transfer(Account from, Account to, BigDecimal amount) 
        throws InsufficientFundsException {
    if (amount.compareTo(BigDecimal.ZERO) <= 0) {
        throw new InvalidTransferAmountException("Amount must be positive");
    }
    if (from.getBalance().compareTo(amount) < 0) {
        throw new InsufficientFundsException("Balance insufficient");
    }
    // proceed with transfer
}

// In controller layer — convert to appropriate HTTP response
@ExceptionHandler(InsufficientFundsException.class)
public ResponseEntity<?> handleInsufficientFunds(InsufficientFundsException e) {
    return ResponseEntity.badRequest().body(e.getMessage());
}
```

**Best Practice:** In layered architectures, wrap low-level checked exceptions in domain-specific unchecked exceptions at service boundaries:
```java
try {
    jdbcTemplate.queryForObject(sql, ...);
} catch (SQLException e) {
    throw new DataAccessException("Failed to fetch user", e);  // Unchecked
}
```

**Why:** Checked exceptions force callers to handle recoverable conditions; unchecked exceptions represent programming errors or unrecoverable situations.  
**When to use:** Use checked exceptions for conditions the caller might reasonably recover from (e.g., “balance insufficient”). Use unchecked for invalid arguments, state violations, or infrastructure failures that should propagate up.  
**How to use:** Extend `Exception` for checked, `RuntimeException` for unchecked. At API boundaries, convert to appropriate HTTP status codes or domain‑specific responses.

---

### 3. How to choose the right collection — practical decision guide?

**Decision Flow:**
```java
// 1. Need ordering? → List or LinkedHashSet
List<String> logs = new ArrayList<>();        // Random access
Queue<Task> pendingTasks = new ArrayDeque<>(); // FIFO processing

// 2. Need uniqueness? → Set
Set<String> uniqueEmails = new HashSet<>();      // Fast O(1) lookup
Set<String> sortedUsers = new TreeSet<>();       // Sorted order

// 3. Need key-value mapping? → Map
Map<String, User> userCache = new HashMap<>();               // General purpose
Map<String, User> sortedCache = new TreeMap<>();             // Sorted keys
Map<Status, List<Task>> groupedTasks = new EnumMap<>(Status.class); // Enum keys

// 4. Concurrent access? → ConcurrentHashMap
Map<String, Session> sessions = new ConcurrentHashMap<>();
sessions.computeIfAbsent(sessionId, k -> new Session());  // Atomic operation

// 5. Performance: pre-size to avoid rehashing
Map<String, User> users = new HashMap<>(expectedSize * 4/3 + 1);
```

**Best Practice:** Always program to the interface (`List`, `Map`, `Set`), not the concrete class. Use `Collections.emptyList()`, `List.of()` for immutability.

**Why:** Each collection has different performance characteristics and capabilities.  
**When to use:**  
- **ArrayList**: 95% of cases – fast random access, compact.  
- **LinkedList**: when you need frequent insertion/removal at ends (as Queue/Deque).  
- **HashSet**: for unique elements, fast lookup.  
- **TreeSet**: when you need sorted order.  
- **HashMap**: general purpose key‑value.  
- **ConcurrentHashMap**: when multiple threads access the map.  
**How to use:** Analyse your access patterns (indexed vs sequential, unique vs duplicate, concurrency) and pick the appropriate implementation. Pre‑size when you know the expected size.

---

### 4. String concatenation — when to use StringBuilder vs + operator?

**Practical Example:**
```java
// ❌ BAD: In loops — creates new StringBuilder each iteration
String result = "";
for (User user : users) {
    result += user.getName();  // Compiles to: result = new StringBuilder(result).append(...)
}

// ✅ GOOD: Single StringBuilder for loop
StringBuilder sb = new StringBuilder(users.size() * 20); // Pre-size
for (User user : users) {
    sb.append(user.getName()).append(", ");
}
String result = sb.toString();

// ✅ OK: Single statement with + (compiler optimizes)
String message = "User " + username + " logged in at " + LocalDateTime.now();

// ✅ BEST: Logging with placeholders (lazy evaluation)
logger.debug("User {} logged in with {} attempts", username, attempts);
// No string concatenation if debug level is disabled
```

**Performance Note:** The `+` operator in a single statement compiles to a single `StringBuilder`. Only avoid it in loops.

**Why:** Using `+` in a loop repeatedly creates intermediate `StringBuilder` objects, causing memory churn.  
**When to use:**  
- Use `+` for simple, one‑off concatenations (the compiler optimises it).  
- Use `StringBuilder` explicitly when concatenating inside a loop or when building large strings dynamically.  
**How to use:** Pre‑size `StringBuilder` if you know the final length to avoid internal resizing. For logging, use parameterised messages to avoid string construction when the log level is disabled.

---

### 5. How to safely use date/time APIs and avoid SimpleDateFormat pitfalls?

**Practical Example:**
```java
// ❌ DANGEROUS: SimpleDateFormat is NOT thread-safe
private static final SimpleDateFormat DATE_FORMAT = new SimpleDateFormat("yyyy-MM-dd");
// Multiple threads using DATE_FORMAT will corrupt each other's state

// ✅ SOLUTION 1: ThreadLocal wrapper
private static final ThreadLocal<SimpleDateFormat> DATE_FORMAT = 
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

// ✅ SOLUTION 2: Use java.time (Java 8+) — immutable and thread-safe
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
LocalDate date = LocalDate.parse("2024-01-15", formatter);
String formatted = LocalDate.now().format(formatter);

// ✅ Parse with fallback
public static LocalDate parseDateSafely(String dateStr) {
    try {
        return LocalDate.parse(dateStr, DateTimeFormatter.ISO_LOCAL_DATE);
    } catch (DateTimeParseException e) {
        logger.warn("Invalid date format: {}", dateStr);
        return null;
    }
}
```

**Best Practice:** Use `java.time` exclusively for all new code. It's immutable, thread-safe, and more expressive.

**Why:** `SimpleDateFormat` is not thread‑safe and leads to concurrency bugs. The `java.time` API is immutable, thread‑safe, and provides a rich set of operations.  
**When to use:** Always prefer `java.time` (LocalDate, LocalDateTime, ZonedDateTime, etc.) over legacy Date/Calendar.  
**How to use:** Create formatters as static final constants (they are thread‑safe). Use `LocalDate.parse()` with a `DateTimeFormatter`. For legacy code, wrap `SimpleDateFormat` in `ThreadLocal`.

---

### 6. equals() and hashCode() — practical implementation with JPA entities

**Practical Example:**
```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    private String name;
    
    // For JPA: Use business key, not database ID
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User user = (User) o;
        return Objects.equals(email, user.email);  // Business key
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(email);  // Consistent with equals
    }
    
    // toString for logging — never include sensitive fields
    @Override
    public String toString() {
        return "User{id=" + id + ", email=" + email + "}";
    }
}
```

**Best Practice:** For JPA entities, use business key (unique, immutable) in equals/hashCode — never use database ID as it's null before persist.

**Why:** `equals`/`hashCode` are used in collections (e.g., `Set`, `Map`). Using the database ID can cause inconsistencies because it’s null before the entity is persisted. A business key (like email) is stable.  
**When to use:** Every class that might be used as a key in a `Map` or stored in a `Set` must override them. For JPA entities, always implement them.  
**How to use:** Use a unique, immutable business key. If none exists, use a synthetic key like a UUID that is set before persistence. Never include the database ID in the calculation.

---

### 7. Singleton pattern — which implementation is safe and testable?

**Practical Example:**
```java
// ✅ BEST: Enum singleton (Josh Bloch)
public enum DataSourceSingleton {
    INSTANCE;
    
    private final HikariDataSource dataSource;
    
    DataSourceSingleton() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(System.getenv("DB_URL"));
        this.dataSource = new HikariDataSource(config);
    }
    
    public DataSource getDataSource() {
        return dataSource;
    }
}

// ✅ GOOD: Initialization-on-demand holder (lazy, thread-safe)
public class ConfigHolder {
    private ConfigHolder() {}
    
    private static class Holder {
        static final Config INSTANCE = new Config();
    }
    
    public static Config getInstance() {
        return Holder.INSTANCE;
    }
}

// ✅ TESTABLE: Use DI instead — Spring manages singletons
@Component
public class UserService {
    private final UserRepository repository;
    // Spring injects singleton — no manual pattern needed
}
```

**Best Practice:** Use dependency injection (Spring) instead of implementing singletons manually. It simplifies testing and reduces coupling.

**Why:** Manual singletons can hinder testability (global state). Enum singleton is safe and concise; holder class provides lazy loading.  
**When to use:** Only when you absolutely need a single instance and DI is not an option (e.g., utility classes, legacy code).  
**How to use:** Prefer enum singleton for simple cases. For lazy initialization, use the holder pattern. In modern applications, let Spring (or another DI container) manage singletons.

---

### 8. Method overriding and covariant return types — practical use

**Practical Example:**
```java
// Base class with factory method
public abstract class BaseParser {
    public abstract BaseParser clone();  // Returns BaseParser
}

// Subclass with covariant return type (Java 5+)
public class JsonParser extends BaseParser {
    @Override
    public JsonParser clone() {  // Returns JsonParser, not BaseParser
        return new JsonParser();
    }
    
    // Override with @Override annotation — catches typos
    @Override
    public String toString() {
        return "JsonParser{}";
    }
}

// For interfaces with default methods (Java 8+)
public interface Loggable {
    default Logger getLogger() {
        return LoggerFactory.getLogger(this.getClass());  // Uses concrete class
    }
}
```

**Best Practice:** Always use `@Override` annotation — it prevents subtle bugs when you think you're overriding but actually overloading.

**Why:** Covariant return types allow more specific return types in overridden methods, reducing casting. `@Override` ensures you’re actually overriding, not overloading.  
**When to use:** Use covariant return types when a subclass can provide a more specific return type (e.g., factory methods). Always annotate overridden methods.  
**How to use:** In the subclass, specify a return type that is a subtype of the parent’s return type. Use `@Override` for every overridden method.

---

### 9. Abstract class vs interface — when to use which in modern Java?

**Practical Example:**
```java
// Interface with default methods (Java 8+)
public interface PaymentProcessor {
    // Abstract method — must implement
    PaymentResult process(PaymentRequest request);
    
    // Default method — common behavior
    default boolean validate(PaymentRequest request) {
        return request.getAmount().compareTo(BigDecimal.ZERO) > 0;
    }
    
    // Static factory method
    static PaymentProcessor createStripeProcessor() {
        return new StripePaymentProcessor();
    }
}

// Abstract class — for state and template methods
public abstract class BasePaymentProcessor implements PaymentProcessor {
    protected final Logger logger = LoggerFactory.getLogger(this.getClass());
    protected final MetricsCollector metrics;
    
    protected BasePaymentProcessor(MetricsCollector metrics) {
        this.metrics = metrics;
    }
    
    // Template method pattern
    @Override
    public final PaymentResult process(PaymentRequest request) {
        long start = System.nanoTime();
        try {
            logger.debug("Processing payment: {}", request.getId());
            PaymentResult result = doProcess(request);
            metrics.recordLatency(System.nanoTime() - start);
            return result;
        } catch (Exception e) {
            metrics.recordError();
            throw e;
        }
    }
    
    protected abstract PaymentResult doProcess(PaymentRequest request);
}
```

**Best Practice:** Prefer interfaces for polymorphism, abstract classes for code reuse with state. Use default methods to evolve interfaces without breaking existing implementations.

**Why:** Interfaces define contracts; abstract classes can hold state and provide common code. Since Java 8, interfaces can have default/static methods, blurring the line.  
**When to use:**  
- Use interface when you need to define a contract that multiple unrelated classes can implement.  
- Use abstract class when you need to share code and state among closely related classes.  
**How to use:** Use default methods in interfaces for backward‑compatible evolution. Use abstract classes for template method pattern or shared fields.

---

### 10. What are the best practices for constructor design?

**Practical Example:**
```java
public class Order {
    private final String id;
    private final List<OrderItem> items;
    private final LocalDateTime createdAt;
    private final User user;
    
    // Primary constructor with all fields
    public Order(String id, List<OrderItem> items, LocalDateTime createdAt, User user) {
        this.id = Objects.requireNonNull(id, "Order ID cannot be null");
        this.items = List.copyOf(items);  // Defensive copy
        this.createdAt = Objects.requireNonNull(createdAt, "Created date required");
        this.user = Objects.requireNonNull(user, "User required");
    }
    
    // Convenience constructor with defaults
    public Order(User user, List<OrderItem> items) {
        this(UUID.randomUUID().toString(), items, LocalDateTime.now(), user);
    }
    
    // Static factory method with meaningful name
    public static Order createEmptyCart(User user) {
        return new Order(user, List.of());
    }
    
    // Builder pattern for complex objects
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        private String id;
        private List<OrderItem> items = new ArrayList<>();
        private LocalDateTime createdAt;
        private User user;
        
        public Builder id(String id) { this.id = id; return this; }
        public Builder addItem(OrderItem item) { this.items.add(item); return this; }
        public Builder user(User user) { this.user = user; return this; }
        
        public Order build() {
            return new Order(
                id != null ? id : UUID.randomUUID().toString(),
                items,
                createdAt != null ? createdAt : LocalDateTime.now(),
                user
            );
        }
    }
}
```

**Best Practice:** Validate and copy parameters early (fail-fast). Prefer static factory methods over constructors for descriptive naming.

**Why:** Constructors with many parameters are error‑prone. Defensive copying protects against external mutations.  
**When to use:** Use a primary constructor for all fields, convenience constructors for common defaults, static factory methods for descriptive names, and the builder pattern for complex objects with many optional parameters.  
**How to use:** Always validate non‑null parameters with `Objects.requireNonNull`. Defensively copy mutable collections. For immutable objects, store only immutable copies.

---

## Part 2: Collections & Data Structures

### 11. HashMap internal working and performance tuning — practical insights

**Practical Example:**
```java
// Pre-size HashMap to avoid rehashing
int expectedEntries = 1000;
// Formula: initialCapacity = expectedEntries / loadFactor + 1
Map<String, User> users = new HashMap<>(expectedEntries * 4/3 + 1);

// For enum keys — use EnumMap (much faster)
Map<OrderStatus, List<Order>> ordersByStatus = new EnumMap<>(OrderStatus.class);

// For concurrent access — use ConcurrentHashMap
ConcurrentHashMap<String, Session> sessions = new ConcurrentHashMap<>();
sessions.computeIfAbsent(sessionId, k -> new Session());  // Atomic

// Never use HashMap in multi-threaded environment without synchronization
// ❌ Will cause infinite loops and data corruption
Map<String, String> unsafe = new HashMap<>();
unsafe.put("key", "value");  // Not thread-safe

// ✅ Use ConcurrentHashMap
ConcurrentHashMap<String, String> safe = new ConcurrentHashMap<>();
safe.put("key", "value");
```

**Key Insight:** HashMap uses `hashCode()` and `equals()`. Poorly implemented `hashCode()` degrades performance to O(n). Use IDE generators for these methods.

**Why:** HashMap is a core data structure; understanding its internals helps tune performance.  
**When to use:** Use HashMap for most key‑value needs, except when you need thread safety (use ConcurrentHashMap) or sorted keys (TreeMap).  
**How to use:** Pre‑size when you know the number of entries. Use `computeIfAbsent` for atomic get‑or‑create. For enum keys, use EnumMap. In concurrent scenarios, use ConcurrentHashMap.

---

### 12. ConcurrentModificationException — how to avoid it in practice?

**Practical Example:**
```java
List<String> items = new ArrayList<>(List.of("A", "B", "C"));

// ❌ Throws ConcurrentModificationException
for (String item : items) {
    if (item.equals("B")) {
        items.remove(item);
    }
}

// ✅ SOLUTION 1: Use Iterator's remove()
Iterator<String> iterator = items.iterator();
while (iterator.hasNext()) {
    if (iterator.next().equals("B")) {
        iterator.remove();
    }
}

// ✅ SOLUTION 2: Use removeIf (Java 8+)
items.removeIf(item -> item.equals("B"));

// ✅ SOLUTION 3: Use CopyOnWriteArrayList for concurrent read/write
List<String> safeList = new CopyOnWriteArrayList<>(items);
for (String item : safeList) {
    safeList.remove(item);  // Works, but copies array each write
}

// ✅ SOLUTION 4: Collect to new list
List<String> filtered = items.stream()
    .filter(item -> !item.equals("B"))
    .collect(Collectors.toList());
```

**Best Practice:** For multi-threaded scenarios, use `ConcurrentHashMap`, `CopyOnWriteArrayList`, or synchronize externally.

**Why:** ConcurrentModificationException occurs when a collection is structurally modified while iterating.  
**When to use:** When you need to modify a collection during iteration, use iterator’s `remove`, `removeIf`, or collect to a new list. For concurrent access, use thread‑safe collections.  
**How to use:** Use `Iterator.remove()` for safe removal. For bulk removal, use `removeIf`. For high‑concurrency reads, use `CopyOnWriteArrayList`.

---

### 13. How to implement a thread-safe cache with expiration?

**Practical Example:**
```java
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;

@Service
public class UserCacheService {
    private final Cache<String, User> cache;
    
    public UserCacheService() {
        this.cache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(30))
            .recordStats()  // Monitor hit/miss rates
            .removalListener((key, value, cause) -> 
                logger.debug("Removed {} due to {}", key, cause))
            .build();
    }
    
    public User getUser(String userId) {
        return cache.get(userId, this::loadFromDatabase);
    }
    
    private User loadFromDatabase(String userId) {
        return userRepository.findById(userId).orElse(null);
    }
    
    // For monitoring
    public CacheStats getStats() {
        return cache.stats();
    }
}
```

**Best Practice:** Use production-ready libraries (Caffeine, Guava) instead of building your own cache. They handle concurrency, expiration, and eviction correctly.

**Why:** Caching improves performance but requires correct concurrency and expiration handling.  
**When to use:** When you have expensive or frequently accessed data that can tolerate some staleness.  
**How to use:** Use Caffeine (or Guava) with appropriate size, expiration, and eviction policies. Monitor hit/miss rates.

---

### 14. When to use LinkedList vs ArrayList — practical decision guide?

**Practical Example:**
```java
// ✅ ArrayList — 95% of use cases
List<String> users = new ArrayList<>(1000);  // Fast random access, compact memory

// ✅ LinkedList — specific scenarios
Queue<Task> taskQueue = new LinkedList<>();   // As Queue
Deque<Event> history = new LinkedList<>();    // As Stack/Deque

// Performance comparison:
public class ListPerformanceDemo {
    public static void main(String[] args) {
        List<Integer> arrayList = new ArrayList<>();
        List<Integer> linkedList = new LinkedList<>();
        
        // Insert at beginning: LinkedList O(1), ArrayList O(n)
        arrayList.add(0, 1);    // Shifts all elements
        linkedList.add(0, 1);   // Just updates pointers
        
        // Random access: ArrayList O(1), LinkedList O(n)
        arrayList.get(500);      // Direct index
        linkedList.get(500);     // Traverses from start/end
    }
}
```

**Rule of Thumb:** Use `ArrayList` unless you frequently insert/delete at beginning or middle of large lists. For queues/deques, use `ArrayDeque` over `LinkedList`.

**Why:** ArrayList provides O(1) random access but O(n) insertion in the middle; LinkedList provides O(1) insertion/deletion at ends but O(n) random access.  
**When to use:** ArrayList for most scenarios (fast iteration, random access). LinkedList for queues, deques, or when you need efficient removal from ends.  
**How to use:** For queues, use ArrayDeque (faster and less memory than LinkedList). For stacks, use ArrayDeque.

---

### 15. How to properly override hashCode() for performance?

**Practical Example:**
```java
public class Product {
    private final String sku;
    private final String name;
    private final BigDecimal price;
    
    // ✅ GOOD: Using Objects.hash (concise, but creates array each call)
    @Override
    public int hashCode() {
        return Objects.hash(sku, name, price);
    }
    
    // ✅ BETTER: Manual calculation for performance-critical code
    @Override
    public int hashCode() {
        int result = 31 + (sku != null ? sku.hashCode() : 0);
        result = 31 * result + (name != null ? name.hashCode() : 0);
        result = 31 * result + (price != null ? price.hashCode() : 0);
        return result;
    }
    
    // ✅ BEST: Cached hashCode for immutable objects
    private volatile int cachedHashCode;
    
    @Override
    public int hashCode() {
        int result = cachedHashCode;
        if (result == 0) {
            result = Objects.hash(sku, name, price);
            cachedHashCode = result;
        }
        return result;
    }
}
```

**Key Insight:** If objects are frequently used as map keys, ensure `hashCode()` distributes values evenly to avoid collisions.

**Why:** hashCode is called frequently in hash‑based collections. Poor implementation leads to collisions and O(n) performance.  
**When to use:** Always override hashCode when you override equals. For immutable objects, consider caching the hash.  
**How to use:** Use the same fields as in equals. For performance‑critical code, manually compute using a prime (like 31). Cache the result if the object is immutable.

---

## Part 3: Concurrency & Multithreading

### 16. Thread vs Runnable — which to use and when?

**Practical Example:**
```java
// Option 1: Extend Thread (only if you need to override thread behavior)
class UploadThread extends Thread {
    private final File file;
    
    UploadThread(File file) {
        super("upload-" + file.getName());  // Custom thread name
        this.file = file;
    }
    
    @Override
    public void run() {
        // Upload logic
    }
}
// Use: new UploadThread(file).start();

// Option 2: Implement Runnable (preferred — more flexible)
class UploadTask implements Runnable {
    private final File file;
    
    UploadTask(File file) { this.file = file; }
    
    @Override
    public void run() {
        // Upload logic
    }
}
// Use: new Thread(new UploadTask(file)).start();

// ✅ BEST: Use ExecutorService (manages thread lifecycle)
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(() -> uploadFile(file));
executor.shutdown();

// For one-off async operations
CompletableFuture.runAsync(() -> uploadFile(file))
    .exceptionally(ex -> { logger.error("Upload failed", ex); return null; });
```

**Best Practice:** Never create threads directly. Use `ExecutorService` or `CompletableFuture` for better resource management and error handling.

**Why:** Implementing Runnable decouples task from thread, allowing reuse. Extending Thread is less flexible and forces inheritance.  
**When to use:** Use Runnable when you only need to define a task. Use ExecutorService to manage thread pools and avoid manual thread creation.  
**How to use:** Create an ExecutorService with appropriate pool size and submit Runnable or Callable tasks. For one‑off async, use CompletableFuture.

---

### 17. synchronized vs Lock — practical concurrency control

**Practical Example:**
```java
public class BankAccount {
    private double balance;
    private final Lock lock = new ReentrantLock();
    private final Condition sufficientFunds = lock.newCondition();
    
    // ✅ Simple synchronization — use synchronized
    public synchronized void deposit(double amount) {
        balance += amount;
    }
    
    // ✅ Complex coordination — use ReentrantLock
    public void withdraw(double amount) throws InterruptedException {
        lock.lock();
        try {
            while (balance < amount) {
                sufficientFunds.await();  // Wait with timeout
            }
            balance -= amount;
        } finally {
            lock.unlock();  // Always unlock in finally
        }
    }
    
    // ✅ Try-lock with timeout (not possible with synchronized)
    public boolean tryTransfer(BankAccount target, double amount, long timeout, TimeUnit unit) 
            throws InterruptedException {
        if (this.lock.tryLock(timeout, unit)) {
            try {
                if (target.lock.tryLock(timeout, unit)) {
                    try {
                        if (this.balance >= amount) {
                            this.balance -= amount;
                            target.balance += amount;
                            return true;
                        }
                    } finally {
                        target.lock.unlock();
                    }
                }
            } finally {
                this.lock.unlock();
            }
        }
        return false;
    }
}
```

**Best Practice:** Use `synchronized` for simple mutual exclusion, `ReentrantLock` for advanced features (tryLock, timed waits, multiple conditions).

**Why:** synchronized is simpler but lacks advanced features like tryLock, interruptible locks, and multiple conditions.  
**When to use:** Use synchronized for basic mutual exclusion. Use ReentrantLock when you need timeout, fairness, or multiple condition variables.  
**How to use:** Always unlock in a finally block. Use tryLock with timeout to avoid deadlocks.

---

### 18. volatile vs AtomicInteger — practical visibility vs atomicity

**Practical Example:**
```java
public class Counter {
    // volatile: ensures visibility, but NOT atomicity
    private volatile int count1 = 0;
    public void increment1() {
        count1++;  // ❌ NOT thread-safe! read-modify-write race condition
    }
    
    // AtomicInteger: both visibility and atomicity
    private AtomicInteger count2 = new AtomicInteger(0);
    public void increment2() {
        count2.incrementAndGet();  // ✅ Thread-safe
    }
    
    // Use volatile for simple flags
    private volatile boolean running = true;
    public void stop() {
        running = false;  // ✅ Atomic write, visibility guaranteed
    }
    
    // LongAdder for high-contention counters
    private final LongAdder requestCount = new LongAdder();
    public void recordRequest() {
        requestCount.increment();  // Better than AtomicLong under contention
    }
    
    // Complex CAS operations
    private final AtomicReference<State> state = new AtomicReference<>(State.INIT);
    public boolean transitionTo(State newState) {
        return state.compareAndSet(State.INIT, newState);  // Atomic update
    }
}
```

**Rule of Thumb:** Use `volatile` only for simple flags. For counters and accumulators, use `Atomic*` classes. For high contention, use `LongAdder`.

**Why:** volatile guarantees visibility but not atomicity. Atomic* classes provide both.  
**When to use:**  
- volatile: simple boolean flags or single‑variable reads/writes that don’t depend on current value.  
- AtomicInteger: counters, accumulators where you need atomic increment.  
- LongAdder: when many threads update a counter (high contention).  
**How to use:** Declare variables volatile if they are only set by one thread and read by others. Use AtomicInteger for thread‑safe counters. Use LongAdder for high‑traffic counters.

---

### 19. ThreadLocal — practical use cases and memory leak prevention

**Practical Example:**
```java
public class RequestContext {
    // ThreadLocal per request — must clean up!
    private static final ThreadLocal<User> currentUser = new ThreadLocal<>();
    private static final ThreadLocal<Transaction> currentTransaction = new ThreadLocal<>();
    
    public static void setCurrentUser(User user) {
        currentUser.set(user);
    }
    
    public static User getCurrentUser() {
        return currentUser.get();
    }
    
    // ✅ CRITICAL: Clean up after request
    public static void clear() {
        currentUser.remove();  // Prevents memory leak
        currentTransaction.remove();
    }
}

// In Spring MVC interceptor
@Component
public class UserContextInterceptor implements HandlerInterceptor {
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, 
                                Object handler, Exception ex) {
        RequestContext.clear();  // Always clean up thread-local
    }
}

// Alternative: Use try-finally pattern
public void processWithContext(User user) {
    RequestContext.setCurrentUser(user);
    try {
        // Business logic
    } finally {
        RequestContext.clear();  // Guaranteed cleanup
    }
}
```

**Critical Warning:** In web applications, threads are reused. Always call `remove()` after request completion to prevent memory leaks.

**Why:** ThreadLocal associates data with the current thread. In thread‑pooled environments, forgetting to remove leads to memory leaks.  
**When to use:** Use ThreadLocal to carry per‑thread context (e.g., user, transaction) across layers without passing it explicitly.  
**How to use:** Always call `remove()` in a finally block or after request processing. Use in frameworks like Spring interceptors to clean up.

---

### 20. CompletableFuture — practical async orchestration

**Practical Example:**
```java
@Service
public class OrderService {
    private final InventoryService inventory;
    private final PaymentService payment;
    private final NotificationService notification;
    
    public CompletableFuture<OrderResult> processOrderAsync(Order order) {
        return CompletableFuture
            // Async check inventory
            .supplyAsync(() -> inventory.checkStock(order))
            .thenCompose(availability -> {
                if (!availability.isAvailable()) {
                    return CompletableFuture.failedFuture(
                        new OutOfStockException("Item unavailable"));
                }
                return CompletableFuture.supplyAsync(() -> payment.charge(order));
            })
            .thenApply(chargeResult -> {
                order.setPaymentId(chargeResult.getPaymentId());
                return order;
            })
            .thenAcceptAsync(processedOrder -> 
                notification.sendConfirmation(processedOrder))
            .thenApply(v -> new OrderResult(OrderStatus.COMPLETED, order.getId()))
            .exceptionally(throwable -> {
                logger.error("Order processing failed", throwable);
                return new OrderResult(OrderStatus.FAILED, order.getId());
            });
    }
    
    // Combine multiple async operations
    public CompletableFuture<UserDashboard> getUserDashboard(String userId) {
        CompletableFuture<User> userFuture = userService.findByIdAsync(userId);
        CompletableFuture<List<Order>> ordersFuture = orderService.findByUserAsync(userId);
        CompletableFuture<List<Notification>> notificationsFuture = 
            notificationService.getUnreadAsync(userId);
        
        return CompletableFuture.allOf(userFuture, ordersFuture, notificationsFuture)
            .thenApply(v -> new UserDashboard(
                userFuture.join(),
                ordersFuture.join(),
                notificationsFuture.join()
            ));
    }
    
    // Timeout handling
    public CompletableFuture<Payment> processWithTimeout(Payment payment) {
        return CompletableFuture.supplyAsync(() -> processPayment(payment))
            .orTimeout(5, TimeUnit.SECONDS)
            .exceptionally(ex -> {
                if (ex instanceof TimeoutException) {
                    return handleTimeout(payment);
                }
                throw new CompletionException(ex);
            });
    }
}
```

**Best Practice:** Use `CompletableFuture` for non-blocking async operations. Always handle exceptions with `exceptionally()` or `whenComplete()`.

**Why:** CompletableFuture provides a fluent API for composing asynchronous operations, handling failures, and combining multiple futures.  
**When to use:** Use when you need to run multiple async tasks and coordinate their results (e.g., microservice calls, parallel processing).  
**How to use:** Chain stages with thenApply, thenCompose, thenAccept. Use `allOf` to wait for multiple futures. Always handle exceptions.

---

### 21. ExecutorService — practical thread pool configurations

**Practical Example:**
```java
// Different thread pool types for different workloads
public class ThreadPoolConfig {
    
    // CPU-bound tasks (e.g., calculations, image processing)
    public ExecutorService cpuBoundPool() {
        int coreCount = Runtime.getRuntime().availableProcessors();
        return Executors.newFixedThreadPool(coreCount);
    }
    
    // I/O-bound tasks (e.g., database calls, HTTP requests)
    public ExecutorService ioBoundPool() {
        int targetConcurrency = 100;  // Higher than core count
        return Executors.newFixedThreadPool(targetConcurrency);
    }
    
    // Work-stealing pool (Java 8+) — adapts to available processors
    public ExecutorService workStealingPool() {
        return Executors.newWorkStealingPool();
    }
    
    // Virtual threads (Java 21+) — massive concurrency
    public ExecutorService virtualThreadPool() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
    
    // Custom thread pool with monitoring
    public ExecutorService monitoredPool() {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            10, 50, 60, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000),
            new CustomThreadFactory(),
            new ThreadPoolExecutor.CallerRunsPolicy()  // Backpressure
        );
        
        // Monitor pool metrics
        ScheduledExecutorService monitor = Executors.newSingleThreadScheduledExecutor();
        monitor.scheduleAtFixedRate(() -> {
            logger.info("Pool stats - Active: {}, Queue: {}, Completed: {}",
                executor.getActiveCount(),
                executor.getQueue().size(),
                executor.getCompletedTaskCount());
        }, 1, 1, TimeUnit.MINUTES);
        
        return executor;
    }
    
    // Always shutdown properly
    @PreDestroy
    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

**Best Practice:** Choose pool size based on workload type. CPU-bound: cores + 1; I/O-bound: higher numbers (often 2-4x cores).

**Why:** Thread pools manage resources and prevent thread explosion. Proper sizing ensures optimal throughput.  
**When to use:** Use ExecutorService whenever you need to run multiple tasks concurrently.  
**How to use:** For CPU‑intensive tasks, use a fixed pool equal to cores. For I/O‑bound tasks, use a larger pool. For massive concurrency, consider virtual threads (Java 21+). Always shut down gracefully.

---

### 22. Virtual Threads (Java 21+) — practical migration from platform threads

**Practical Example:**
```java
// Traditional platform threads (heavy)
ExecutorService platformExecutor = Executors.newFixedThreadPool(100);
for (int i = 0; i < 10000; i++) {
    platformExecutor.submit(() -> {
        Thread.sleep(1000);  // Blocks OS thread
    });
}
// This requires 100 OS threads — resource intensive

// Virtual threads (Java 21+)
ExecutorService virtualExecutor = Executors.newVirtualThreadPerTaskExecutor();
for (int i = 0; i < 10000; i++) {
    virtualExecutor.submit(() -> {
        Thread.sleep(1000);  // Yields, doesn't block OS thread
    });
}
// Handles 10,000 concurrent tasks with minimal OS threads

// Spring Boot with virtual threads
@Bean
public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadCustomizer() {
    return protocolHandler -> {
        protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
    };
}

// Structured concurrency (Java 21+)
public Response handleRequest(Request request) throws InterruptedException {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Subtask<Data> dataTask = scope.fork(() -> fetchData(request));
        Subtask<Metadata> metadataTask = scope.fork(() -> fetchMetadata(request));
        
        scope.join();  // Wait for both
        scope.throwIfFailed();  // Propagate any failure
        
        return new Response(dataTask.get(), metadataTask.get());
    }
}
```

**Performance Impact:** Virtual threads reduce thread creation overhead by ~90% and enable massive concurrency without degrading performance.

**Why:** Virtual threads are lightweight, allowing millions of concurrent tasks with minimal memory. They make blocking operations cheap.  
**When to use:** Use for high‑concurrency I/O‑bound tasks (e.g., web servers, microservices). Replace existing thread pools where tasks often block.  
**How to use:** Use `Executors.newVirtualThreadPerTaskExecutor()`. In Spring Boot, customise the Tomcat executor. Use structured concurrency for scoped subtasks.

---

### 23. ConcurrentHashMap — advanced operations beyond put/get

**Practical Example:**
```java
public class UserSessionManager {
    private final ConcurrentHashMap<String, UserSession> sessions = new ConcurrentHashMap<>();
    
    // Atomic get-or-create
    public UserSession getOrCreateSession(String sessionId) {
        return sessions.computeIfAbsent(sessionId, id -> {
            logger.info("Creating new session: {}", id);
            return new UserSession(id);
        });
    }
    
    // Atomic update
    public void updateLastAccess(String sessionId) {
        sessions.computeIfPresent(sessionId, (id, session) -> {
            session.updateLastAccess();
            return session;  // Return updated value
        });
    }
    
    // Batch operations safely
    public void evictExpiredSessions(Duration maxIdleTime) {
        LocalDateTime cutoff = LocalDateTime.now().minus(maxIdleTime);
        
        // Use keySet() with removeIf — not entrySet() for modification
        sessions.keySet().removeIf(sessionId -> {
            UserSession session = sessions.get(sessionId);
            return session != null && session.getLastAccess().isBefore(cutoff);
        });
    }
    
    // Searching with concurrency
    public List<UserSession> findSessionsByUser(String userId) {
        return sessions.reduceValues(2, (session, result) -> {
            if (session.getUserId().equals(userId)) {
                // Accumulate matches
            }
            return result;
        }, (r1, r2) -> combineResults(r1, r2));
    }
    
    // Bulk operations
    public void notifyAllSessions(String message) {
        sessions.forEachValue(1, session -> session.sendNotification(message));
    }
}
```

**Key Insight:** `ConcurrentHashMap` allows concurrent reads and segmented writes. Use `computeIfAbsent` for atomic get-or-create operations.

**Why:** It provides fine‑grained concurrency with non‑blocking reads and per‑segment locking for writes.  
**When to use:** Whenever a map is accessed by multiple threads. Avoid manual synchronization.  
**How to use:** Use `computeIfAbsent` for thread‑safe get‑or‑put. Use `forEach`, `reduce`, `search` for bulk operations. Never use `entrySet()` for structural modifications.

---

## Part 4: Memory Management & Performance

### 24. Memory leaks in Java — how to detect and fix with practical examples

**Practical Example:**
```java
// Common memory leak patterns and fixes

// 1. Unbounded cache
class BadCache {
    private static final Map<String, byte[]> cache = new HashMap<>();
    public void put(String key, byte[] data) {
        cache.put(key, data);  // Never removed — OOM
    }
}

// Fix: Use bounded cache with eviction
class GoodCache {
    private final Cache<String, byte[]> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(30))
        .build();
}

// 2. Listener registry without removal
class BadEventBus {
    private final List<Listener> listeners = new ArrayList<>();
    public void register(Listener listener) {
        listeners.add(listener);  // Never removed
    }
}

// Fix: Use WeakReference or ensure removal
class GoodEventBus {
    private final Set<WeakReference<Listener>> listeners = 
        Collections.newSetFromMap(new WeakHashMap<>());
}

// 3. ThreadLocal in thread-pooled environments
class BadRequestContext {
    private static final ThreadLocal<User> currentUser = new ThreadLocal<>();
    // Never cleared — threads are reused
}

// Fix: Always clear ThreadLocal
class GoodRequestContext {
    public static void executeWithUser(User user, Runnable task) {
        currentUser.set(user);
        try {
            task.run();
        } finally {
            currentUser.remove();  // CRITICAL
        }
    }
}
```

**Detection with JFR:** Use Java Flight Recorder to monitor heap usage and identify objects that grow over time.

```bash
# Generate heap dump for analysis
jmap -dump:live,format=b,file=heap.hprof <pid>

# Analyze with Eclipse MAT or JProfiler
# Look for: large retained sizes, excessive instances, GC root paths 
```

**Why:** Memory leaks cause application crashes (OOM) and degrade performance.  
**When to use:** Monitor heap usage in production; if it increases over time, investigate.  
**How to use:** Use tools like JFR, heap dumps, and profilers. Look for objects that are referenced from static fields, thread‑locals, or caches without eviction. Fix by using bounded caches, weak references, or explicit cleanup.

---

### 25. Garbage Collection tuning — practical JVM flags

**Practical Example:**
```bash
# G1GC (default in Java 17) — balanced throughput and latency
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:G1HeapRegionSize=16m \
     -Xms4g -Xmx4g \
     -XX:+PrintGCDetails \
     -Xlog:gc*:file=gc.log:time,uptime \
     -jar app.jar

# ZGC (for sub-millisecond pauses, Java 11+)
java -XX:+UseZGC \
     -Xms8g -Xmx8g \
     -XX:ZCollectionInterval=120 \
     -Xlog:gc*:file=gc.log \
     -jar app.jar

# Heap size recommendations
# - Set -Xms equal to -Xmx to avoid resizing overhead
# - For containers, use -XX:MaxRAMPercentage=75.0
java -XX:MaxRAMPercentage=75.0 \
     -XX:InitialRAMPercentage=75.0 \
     -XX:+UseContainerSupport \
     -jar app.jar

# Enable GC logging for production troubleshooting
java -Xlog:gc*:file=/var/log/app-gc.log:time,uptime,level,tags:filecount=5,filesize=100m
```

**Monitoring:** Use `jstat -gc <pid> 1000` to watch GC activity in real-time.

**Why:** GC tuning reduces pause times and improves throughput.  
**When to use:** When you experience high GC overhead, long pauses, or memory pressure.  
**How to use:** Start with defaults (G1GC). Monitor GC logs. Adjust heap size (`-Xms`, `-Xmx`) and pause time target (`-XX:MaxGCPauseMillis`). For containers, use `-XX:MaxRAMPercentage`. Use ZGC for very low latency.

---

### 26. Profiling CPU and memory — practical tools and workflow

**Practical Workflow:**
```bash
# 1. Profile during load testing
java -XX:+FlightRecorder -XX:StartFlightRecording=filename=profile.jfr,duration=60s -jar app.jar

# 2. Analyze with Java Mission Control
jmc profile.jfr
# Look at: CPU hotspots, allocation profiles, lock contention

# 3. For CPU profiling during active application
jcmd <pid> JFR.start name=CPUProfile duration=60s filename=cpu-profile.jfr

# 4. Heap dump analysis
jmap -dump:live,format=b,file=heap.hprof <pid>
# Open in Eclipse MAT or JProfiler
```

**Key Metrics to Monitor:**
- GC frequency and duration
- Heap usage after full GC (live set)
- Object allocation rates
- Thread counts and contention
- CPU usage per thread

**Why:** Profiling identifies performance bottlenecks and memory leaks.  
**When to use:** During load testing or when investigating production issues.  
**How to use:** Use JFR + JMC for low‑overhead continuous monitoring. Use heap dumps to find memory leaks. Use async‑profiler or JMC for CPU profiling.

---

### 27. Stream API performance — practical optimization techniques

**Practical Example:**
```java
// ❌ AVOID: Multiple passes over data
List<String> names = users.stream()
    .filter(u -> u.getAge() > 18)
    .map(User::getName)
    .collect(Collectors.toList());

long count = users.stream()
    .filter(u -> u.getAge() > 18)
    .count();  // Second pass

// ✅ GOOD: Single pass with partitioning
Map<Boolean, List<User>> partitioned = users.stream()
    .collect(Collectors.partitioningBy(u -> u.getAge() > 18));
List<String> names = partitioned.get(true).stream().map(User::getName).collect(Collectors.toList());
long count = partitioned.get(true).size();

// ❌ AVOID: Auto-boxing with streams
List<Integer> numbers = IntStream.range(0, 1_000_000)
    .boxed()  // Creates Integer objects
    .collect(Collectors.toList());

// ✅ GOOD: Use primitive streams
int sum = IntStream.range(0, 1_000_000).sum();

// ❌ AVOID: Parallel streams for small or I/O-bound operations
list.parallelStream()
    .forEach(item -> callExternalApi(item));  // Worse than sequential

// ✅ GOOD: Parallel streams for CPU-bound operations on large datasets
double avg = largeList.parallelStream()
    .mapToDouble(this::expensiveComputation)
    .average()
    .orElse(0.0);

// ❌ AVOID: Unnecessary intermediate collections
list.stream()
    .filter(x -> x > 0)
    .map(x -> x * 2)
    .collect(Collectors.toList())
    .stream()  // Extra collection!
    .filter(x -> x < 100)
    .collect(Collectors.toList());

// ✅ GOOD: Chain operations
list.stream()
    .filter(x -> x > 0)
    .map(x -> x * 2)
    .filter(x -> x < 100)
    .collect(Collectors.toList());

// ✅ BEST: Short-circuit when possible
list.stream()
    .filter(this::expensiveCheck)
    .findFirst()  // Stops early
    .orElse(null);
```

**Performance Guidelines:**
- Use primitive streams (`IntStream`, `LongStream`) to avoid boxing
- Short-circuit with `findFirst()`, `limit()` when possible
- Pre-size collections when collecting
- Parallel streams benefit only CPU-bound, stateless operations on large datasets

**Why:** Streams can be inefficient if used incorrectly.  
**When to use:** Use streams for readable declarative code, but be aware of performance trade‑offs.  
**How to use:** Avoid multiple passes; use primitive streams; prefer sequential over parallel unless data is large and CPU‑bound; short‑circuit early.

---

### 28. Optional — practical patterns and anti-patterns

**Practical Example:**
```java
// ❌ BAD: Using Optional as field or parameter
public class User {
    private Optional<String> middleName;  // Don't do this
}
public void process(Optional<String> param) {}  // Don't do this

// ✅ GOOD: Optional only for return values
public Optional<User> findUser(String id) {
    return Optional.ofNullable(userRepository.get(id));
}

// ❌ BAD: Using get() without check
Optional<User> user = findUser("123");
User u = user.get();  // NoSuchElementException if empty

// ✅ GOOD: Safe extraction patterns
User user = findUser("123")
    .orElse(defaultUser);

User user = findUser("123")
    .orElseGet(() -> loadFallbackUser());  // Lazy

User user = findUser("123")
    .orElseThrow(() -> new UserNotFoundException("User not found"));

// ❌ BAD: Complex if-else with Optional
if (userOpt.isPresent()) {
    processUser(userOpt.get());
} else {
    logWarning();
}

// ✅ GOOD: Use ifPresent or ifPresentOrElse
userOpt.ifPresentOrElse(
    this::processUser,
    () -> logWarning()
);

// ✅ GOOD: Chaining with map/filter
String email = findUser("123")
    .filter(u -> u.isActive())
    .map(User::getEmail)
    .orElse("default@example.com");
```

**Best Practice:** Use `Optional` only for return types where absence is valid. Never use as fields, parameters, or in collections.

**Why:** Optional forces explicit handling of absent values, reducing NPEs.  
**When to use:** As a return type for methods that may not return a value.  
**How to use:** Use `orElse`, `orElseGet`, `orElseThrow` to extract values. Use `ifPresentOrElse` for side effects. Avoid `get()` without checking.

---

### 29. JVM memory model — practical configuration

**Practical Example:**
```yaml
# Docker container JVM configuration
JAVA_OPTS: >-
  -XX:+UseContainerSupport
  -XX:MaxRAMPercentage=75.0
  -XX:InitialRAMPercentage=75.0
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:+PrintGCDetails
  -Xlog:gc*:file=/var/log/app/gc.log:time,uptime:filecount=5,filesize=100m
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/var/log/app/heapdump.hprof
  -XX:ErrorFile=/var/log/app/hs_err_pid%p.log

# Memory regions (for 4GB heap)
# Young Gen: ~1.5GB (Eden: 1.2GB, Survivor: 300MB)
# Old Gen: ~2.5GB
# Metaspace: default unlimited, use -XX:MaxMetaspaceSize=256m
```

**Monitoring Commands:**
```bash
# Heap summary
jmap -heap <pid>

# Live object histogram
jmap -histo:live <pid> | head -20

# GC stats
jstat -gcutil <pid> 1000
```

**Why:** Understanding the JVM memory model helps diagnose memory issues and tune performance.  
**When to use:** When configuring heap size, GC, or troubleshooting OOM.  
**How to use:** Set `-Xms` = `-Xmx` for fixed heap. Use `-XX:MaxRAMPercentage` for containers. Enable OOM heap dump. Monitor young/old generation sizes.

---

### 30. Class loading and memory — avoiding ClassLoader leaks

**Practical Example:**
```java
// ClassLoader leak in application servers
// ❌ Pattern that causes leaks:
public class LeakyServlet extends HttpServlet {
    private static final Map<String, Object> cache = new HashMap<>();
    // When webapp undeploys, static fields keep classloader alive
}

// ✅ Fix: Use weak references or clean up on destroy
public class SafeServlet extends HttpServlet {
    private Map<String, Object> cache;  // Instance, not static
    
    @Override
    public void destroy() {
        cache.clear();  // Clean up on undeploy
        cache = null;
    }
}

// Thread context classloader pattern
public <T> T executeWithClassLoader(ClassLoader cl, Supplier<T> task) {
    ClassLoader original = Thread.currentThread().getContextClassLoader();
    try {
        Thread.currentThread().setContextClassLoader(cl);
        return task.get();
    } finally {
        Thread.currentThread().setContextClassLoader(original);  // Always restore
    }
}
```

**Warning:** In application servers, static fields referencing classes loaded by webapp ClassLoader prevent undeployment and cause memory leaks.

**Why:** ClassLoader leaks occur when references to classes loaded by a ClassLoader survive after the application is undeployed, preventing GC.  
**When to use:** In application servers (e.g., Tomcat, WebLogic) or any environment where applications are dynamically loaded/unloaded.  
**How to use:** Avoid static fields referencing application‑specific classes. Use instance fields and clean up in `destroy()`. Always restore thread context classloader.

---

## Part 5: Java 17+ Modern Features

### 31. Records — practical usage beyond simple DTOs

**Practical Example:**
```java
// Basic record
public record User(String id, String email, UserStatus status) {}

// Record with validation
public record Order(String id, BigDecimal amount, LocalDateTime createdAt) {
    public Order {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        if (id == null || id.isBlank()) {
            throw new IllegalArgumentException("ID required");
        }
        // Normalization
        createdAt = createdAt != null ? createdAt : LocalDateTime.now();
    }
    
    // Additional methods
    public boolean isExpired() {
        return createdAt.isBefore(LocalDateTime.now().minusDays(30));
    }
    
    // Static factory
    public static Order create(BigDecimal amount) {
        return new Order(UUID.randomUUID().toString(), amount, LocalDateTime.now());
    }
}

// Record as map key — automatic equals/hashCode
Map<User, List<Order>> ordersByUser = new HashMap<>();

// Record with custom constructor
public record UserProfile(String username, String email, List<String> roles) {
    public UserProfile(String username, String email, List<String> roles) {
        this.username = username.toLowerCase();
        this.email = email.toLowerCase();
        this.roles = List.copyOf(roles);  // Defensive copy
    }
}
```

**Best Practice:** Use records for immutable data carriers. They're perfect for DTOs, API responses, and map keys.

**Why:** Records reduce boilerplate and provide concise, immutable data objects with automatic equals/hashCode/toString.  
**When to use:** For simple data carriers, DTOs, value objects, and map keys.  
**How to use:** Declare with `record` and a compact constructor for validation/normalization. Add methods as needed.

---

### 32. Sealed classes — modeling restricted hierarchies

**Practical Example:**
```java
// Sealed interface with permitted subtypes
public sealed interface PaymentMethod 
    permits CreditCard, PayPal, BankTransfer {
    
    String getPaymentId();
    BigDecimal getAmount();
}

// Permitted implementations
public final class CreditCard implements PaymentMethod {
    private final String cardNumber;
    private final BigDecimal amount;
    
    public CreditCard(String cardNumber, BigDecimal amount) {
        this.cardNumber = maskCardNumber(cardNumber);
        this.amount = amount;
    }
    
    private String maskCardNumber(String number) {
        return "****-****-****-" + number.substring(12);
    }
    
    @Override
    public String getPaymentId() { return "CC-" + cardNumber; }
    @Override
    public BigDecimal getAmount() { return amount; }
}

public final class PayPal implements PaymentMethod {
    private final String email;
    private final BigDecimal amount;
    
    public PayPal(String email, BigDecimal amount) {
        this.email = email;
        this.amount = amount;
    }
    
    @Override
    public String getPaymentId() { return "PP-" + email; }
    @Override
    public BigDecimal getAmount() { return amount; }
}

public non-sealed class BankTransfer implements PaymentMethod {
    // non-sealed allows further extension
}

// Pattern matching with sealed classes (exhaustive)
public String formatPayment(PaymentMethod payment) {
    return switch (payment) {
        case CreditCard cc -> "Card: " + cc.getPaymentId();
        case PayPal pp -> "PayPal: " + pp.getPaymentId();
        case BankTransfer bt -> "Bank: " + bt.getPaymentId();
        // No default needed — exhaustive!
    };
}
```

**Best Practice:** Use sealed classes for domain modeling where you control the entire hierarchy. They enable exhaustive pattern matching.

**Why:** Sealed classes give you control over which classes can extend/implement a type, enabling exhaustive checks in pattern matching.  
**When to use:** When you have a fixed set of subtypes (e.g., domain events, payment methods).  
**How to use:** Use `sealed` with `permits` list. Subtypes must be `final`, `sealed`, or `non-sealed`.

---

### 33. Pattern matching for switch — practical examples

**Practical Example:**
```java
// Pattern matching with type checking
public String handlePayment(Object obj) {
    return switch (obj) {
        case null -> "Null payment";  // Null handling!
        case CreditCard cc -> "Processing credit card " + cc.getLastFourDigits();
        case PayPal pp -> "Processing PayPal " + pp.getEmail();
        case String s when s.startsWith("REF_") -> "Refund: " + s;
        case String s -> "Unknown string: " + s;
        case int[] arr -> "Array of length " + arr.length;
        default -> "Unknown payment type";
    };
}

// Record pattern matching (Java 21+)
public record Point(int x, int y) {}
public record Rect(Point topLeft, Point bottomRight) {}

public String describeShape(Object shape) {
    return switch (shape) {
        case Rect(Point(var x1, var y1), Point(var x2, var y2)) 
            when x1 < x2 && y1 < y2 -> 
            String.format("Rectangle from (%d,%d) to (%d,%d)", x1, y1, x2, y2);
        case Point(var x, var y) -> "Point at (" + x + "," + y + ")";
        default -> "Unknown";
    };
}

// Deconstruction in for loops
List<Rect> rectangles = List.of(new Rect(new Point(0,0), new Point(10,10)));
for (Rect(Point(var x1, var y1), Point(var x2, var y2)) : rectangles) {
    System.out.println("Width: " + (x2 - x1));
}
```

**Best Practice:** Pattern matching eliminates casts and makes code safer. Use guarded patterns for additional conditions.

**Why:** It simplifies type checking and extraction, reducing boilerplate.  
**When to use:** In switch expressions where you need to check types or deconstruct records.  
**How to use:** Use `case` with type patterns, null handling, and guarded patterns. For records, use deconstruction patterns.

---

### 34. Text blocks — clean multi-line strings

**Practical Example:**
```java
// Traditional string concatenation — hard to read
String json = "{\n" +
              "  \"name\": \"John\",\n" +
              "  \"age\": 30,\n" +
              "  \"address\": {\n" +
              "    \"street\": \"123 Main St\",\n" +
              "    \"city\": \"Boston\"\n" +
              "  }\n" +
              "}";

// Text blocks — preserve formatting
String json = """
    {
      "name": "John",
      "age": 30,
      "address": {
        "street": "123 Main St",
        "city": "Boston"
      }
    }
    """;

// SQL queries with placeholders
String sql = """
    SELECT u.id, u.name, o.total
    FROM users u
    INNER JOIN orders o ON u.id = o.user_id
    WHERE u.status = 'ACTIVE'
      AND o.created_at >= ?
    ORDER BY o.total DESC
    """;

// HTML templates
String html = """
    <!DOCTYPE html>
    <html>
      <head>
        <title>%s</title>
      </head>
      <body>
        <h1>Welcome, %s!</h1>
      </body>
    </html>
    """.formatted(title, username);  // Java 15+ formatted

// YAML configuration
String yaml = """
    server:
      port: 8080
      context-path: /api
    logging:
      level:
        root: INFO
        com.example: DEBUG
    """;
```

**Best Practice:** Use text blocks for JSON, SQL, HTML, XML, and YAML to improve readability and reduce escaping.

**Why:** Text blocks preserve formatting and reduce the need for escape sequences.  
**When to use:** For any multi‑line string literal (JSON, SQL, HTML, XML).  
**How to use:** Enclose in triple double‑quotes. Indentation is based on the closing delimiter. Use `formatted()` for interpolation.

---

### 35. Switch expressions — concise and safe

**Practical Example:**
```java
// Traditional switch (statement)
String status;
switch (orderStatus) {
    case PENDING:
        status = "Processing";
        break;
    case SHIPPED:
        status = "On the way";
        break;
    case DELIVERED:
        status = "Completed";
        break;
    default:
        status = "Unknown";
}

// Switch expression — no fall-through, returns value
String status = switch (orderStatus) {
    case PENDING -> "Processing";
    case SHIPPED -> "On the way";
    case DELIVERED -> "Completed";
    default -> "Unknown";
};

// Multiple labels and yield for block
String message = switch (errorCode) {
    case 400, 401, 403 -> "Client error";
    case 500, 502, 503 -> {
        logServerError(errorCode);
        yield "Server error";  // yield returns value from block
    }
    default -> "Unknown error";
};

// Enum switch — exhaustive without default
public enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }

public String getDayType(Day day) {
    return switch (day) {
        case MON, TUE, WED, THU, FRI -> "Weekday";
        case SAT, SUN -> "Weekend";
        // No default needed — all cases covered
    };
}
```

**Best Practice:** Use switch expressions over traditional switch statements — they're more concise and prevent fall-through bugs.

**Why:** Switch expressions are more compact, return a value, and eliminate accidental fall‑through.  
**When to use:** Use whenever you have a switch that assigns a value or performs a mapping.  
**How to use:** Use arrow syntax (`->`) for single expressions; use `yield` in blocks. For enums, you can be exhaustive and omit default.

---

## Part 6: Security & Safe Coding

### 36. SQL injection — practical prevention

**Practical Example:**
```java
// ❌ DANGEROUS: String concatenation
String query = "SELECT * FROM users WHERE email = '" + userEmail + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(query);  // SQL injection!

// ✅ SAFE: PreparedStatement
String query = "SELECT * FROM users WHERE email = ?";
PreparedStatement stmt = conn.prepareStatement(query);
stmt.setString(1, userEmail);
ResultSet rs = stmt.executeQuery();

// ✅ SAFE: JPA with parameter binding
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// ❌ DANGEROUS: Dynamic table/column names
String query = "SELECT * FROM " + tableName + " WHERE status = ?";
// PreparedStatement can't protect table names

// ✅ SAFE: Whitelist validation for dynamic names
private static final Set<String> ALLOWED_TABLES = Set.of("users", "orders", "products");
public List<?> queryTable(String tableName, String condition) {
    if (!ALLOWED_TABLES.contains(tableName)) {
        throw new IllegalArgumentException("Invalid table name");
    }
    String query = "SELECT * FROM " + tableName + " WHERE " + condition;
    // Still use PreparedStatement for condition parameters
}

// JPA with named parameters
@Query("SELECT u FROM User u WHERE u.email = :email AND u.status = :status")
User findByEmailAndStatus(@Param("email") String email, @Param("status") String status);
```

**Key Principle:** Never concatenate user input into SQL. Use parameterized queries or JPA.

**Why:** SQL injection is a critical vulnerability that can lead to data loss or compromise.  
**When to use:** Always use prepared statements or JPA with parameter binding for any query with user input.  
**How to use:** For JDBC, use `PreparedStatement`. For JPA, use named parameters. For dynamic table/column names, whitelist allowed values.

---

### 37. XML External Entity (XXE) — safe XML parsing

**Practical Example:**
```java
// ❌ VULNERABLE: Default XML parser
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
DocumentBuilder builder = factory.newDocumentBuilder();
Document doc = builder.parse(xmlInputStream);  // XXE vulnerability

// ✅ SAFE: Disable DOCTYPE and external entities
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
factory.setXIncludeAware(false);
factory.setExpandEntityReferences(false);
DocumentBuilder builder = factory.newDocumentBuilder();
Document doc = builder.parse(xmlInputStream);

// ✅ SAFE: Use Jackson with safe configuration
XmlMapper xmlMapper = new XmlMapper();
xmlMapper.disable(DeserializationFeature.UNWRAP_ROOT_VALUE);
xmlMapper.enable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
xmlMapper.configure(JsonParser.Feature.ALLOW_COMMENTS, false);

// ✅ SAFE: For modern APIs, prefer JSON over XML
ObjectMapper jsonMapper = new ObjectMapper();  // No XXE risk
```

**Best Practice:** When using XML, always configure parsers to disable external entities. Prefer JSON when possible.

**Why:** XXE allows attackers to read local files, perform SSRF, or cause denial of service.  
**When to use:** Whenever parsing untrusted XML (e.g., from user input).  
**How to use:** Disable DOCTYPE and external entities in XML parsers. Use Jackson’s safe defaults for XML. Consider using JSON instead.

---

### 38. Deserialization attacks — safe object deserialization

**Practical Example:**
```java
// ❌ DANGEROUS: Java native deserialization
ObjectInputStream ois = new ObjectInputStream(inputStream);
Object obj = ois.readObject();  // Arbitrary code execution risk!

// ✅ SAFE: Use JSON with strict schema
ObjectMapper mapper = new ObjectMapper();
mapper.enableDefaultTyping();  // Don't do this!
// Instead, configure for type safety
mapper.activateDefaultTyping(
    LaissezFaireSubTypeValidator.instance,
    ObjectMapper.DefaultTyping.NON_FINAL
);

// ✅ BETTER: Use whitelisted classes
@Configuration
public class SafeObjectMapper {
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.addMixIn(Object.class, WhitelistMixin.class);
        return mapper;
    }
    
    // Whitelist approach
    private static final Set<String> ALLOWED_CLASSES = Set.of(
        "com.example.User",
        "com.example.Order",
        "java.util.ArrayList",
        "java.util.HashMap"
    );
    
    static class WhitelistMixin {
        @JsonTypeInfo(use = JsonTypeInfo.Id.CLASS, 
                      include = JsonTypeInfo.As.PROPERTY, 
                      property = "@class")
        @JsonTypeIdResolver(WhitelistTypeIdResolver.class)
        private Object value;
    }
}

// ✅ SAFE: Use Protocol Buffers or Avro for serialization
message User {
    string id = 1;
    string email = 2;
    int32 age = 3;
}
```

**Warning:** Java native deserialization is inherently unsafe. Use structured data formats with explicit schemas.

**Why:** Deserialization of untrusted data can lead to remote code execution.  
**When to use:** Avoid native Java deserialization. Prefer JSON/XML with safe libraries.  
**How to use:** Use Jackson with whitelisted classes. Use Protocol Buffers or Avro for safe, schema‑based serialization.

---

### 39. Secure password storage — practical hashing

**Practical Example:**
```java
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Component
public class PasswordService {
    private final PasswordEncoder passwordEncoder;
    
    public PasswordService() {
        // BCrypt with strength factor 12 (takes ~0.1s)
        this.passwordEncoder = new BCryptPasswordEncoder(12);
    }
    
    public String hashPassword(String plainPassword) {
        return passwordEncoder.encode(plainPassword);
    }
    
    public boolean verifyPassword(String plainPassword, String hashedPassword) {
        return passwordEncoder.matches(plainPassword, hashedPassword);
    }
}

// NEVER do this:
// - MD5: broken, too fast
// - SHA-1: broken
// - SHA-256 without salt: vulnerable to rainbow tables
// - Custom algorithms: never invent crypto

// ❌ BAD:
String hashed = DigestUtils.md5Hex(password);  // Never!

// ✅ For legacy compatibility with argon2id
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    // Uses bcrypt by default, supports multiple encodings
}

// For high-security applications, use Argon2id
@Bean
public PasswordEncoder argon2PasswordEncoder() {
    return new Argon2PasswordEncoder(16, 32, 1, 65536, 10);
}
```

**Best Practice:** Use BCrypt, Argon2id, or PBKDF2. Never roll your own crypto. Never log passwords.

**Why:** Passwords must be stored with a strong, slow hash function to resist brute‑force attacks.  
**When to use:** Any time you store user passwords.  
**How to use:** Use Spring Security’s `BCryptPasswordEncoder` or Argon2. Use a high cost factor (work factor) that provides acceptable performance.

---

### 40. JWT security — safe token handling

**Practical Example:**
```java
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import java.security.Key;

@Service
public class JwtService {
    private final Key signingKey;
    
    public JwtService() {
        // Use strong key (minimum 256 bits for HS256)
        String secret = System.getenv("JWT_SECRET");
        if (secret == null || secret.length() < 32) {
            throw new IllegalStateException("JWT secret must be at least 32 characters");
        }
        this.signingKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }
    
    public String generateToken(User user) {
        return Jwts.builder()
            .setSubject(user.getId())
            .claim("email", user.getEmail())
            .claim("role", user.getRole())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 3600000))  // 1 hour
            .signWith(signingKey, SignatureAlgorithm.HS256)
            .compact();
    }
    
    public Claims validateToken(String token) {
        try {
            return Jwts.parserBuilder()
                .setSigningKey(signingKey)
                .build()
                .parseClaimsJws(token)
                .getBody();
        } catch (ExpiredJwtException e) {
            throw new AuthException("Token expired");
        } catch (JwtException e) {
            throw new AuthException("Invalid token");
        }
    }
    
    // For refresh tokens: store in database, not just JWT
    // Include JTI claim for revocation
    public String generateRefreshToken(User user) {
        String jti = UUID.randomUUID().toString();
        // Store jti in database with user mapping
        
        return Jwts.builder()
            .setSubject(user.getId())
            .setId(jti)
            .setExpiration(new Date(System.currentTimeMillis() + 604800000))  // 7 days
            .signWith(signingKey, SignatureAlgorithm.HS256)
            .compact();
    }
}
```

**Security Tips:**
- Use strong secrets (minimum 32 characters for HS256)
- Set appropriate expiration times
- Store refresh tokens in database with revocation capability
- Never log tokens
- Use HTTPS exclusively

**Why:** JWTs are used for stateless authentication, but must be properly secured.  
**When to use:** For stateless APIs, microservices, and single sign‑on.  
**How to use:** Use a strong secret; set short expiration; include JTI for revocation; use refresh tokens stored in a database. Validate tokens on each request.

---

### 41. Logging sensitive data — masking and safe logging

**Practical Example:**
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class UserService {
    private static final Logger logger = LoggerFactory.getLogger(UserService.class);
    
    // ✅ GOOD: Parameterized logging, no concatenation
    public void login(String username, String password) {
        logger.info("User {} attempted login", username);  // No password logged
        
        try {
            authenticate(username, password);
            logger.info("User {} successfully logged in", username);
        } catch (AuthException e) {
            logger.warn("Failed login attempt for user {}", username);
        }
    }
    
    // ❌ BAD: Logging sensitive data
    public void badLogging(User user) {
        logger.info("User created: {}", user);  // Might contain PII
    }
    
    // ✅ GOOD: Mask sensitive fields in toString
    public class User {
        private String email;
        private String ssn;
        private String password;
        
        @Override
        public String toString() {
            return String.format("User{email=%s, ssn=***-**-****, password=***}",
                maskEmail(email));
        }
        
        private String maskEmail(String email) {
            if (email == null) return null;
            int atIndex = email.indexOf('@');
            if (atIndex <= 1) return email;
            return email.charAt(0) + "***" + email.substring(atIndex);
        }
    }
    
    // ✅ GOOD: Masking converter for logback
    @Component
    public class MaskingConverter extends MessageConverter {
        private static final Pattern SSN_PATTERN = Pattern.compile("\\d{3}-\\d{2}-\\d{4}");
        private static final Pattern EMAIL_PATTERN = Pattern.compile("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}");
        
        @Override
        public String convert(ILoggingEvent event) {
            String message = event.getFormattedMessage();
            message = SSN_PATTERN.matcher(message).replaceAll("***-**-****");
            message = EMAIL_PATTERN.matcher(message).replaceAll(m -> maskEmail(m.group()));
            return message;
        }
    }
}

// logback-spring.xml configuration
<conversionRule conversionWord="mask" converterClass="com.example.MaskingConverter" />
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
        <pattern>%d{HH:mm:ss} [%thread] %-5level %logger{36} - %mask%n</pattern>
    </encoder>
</appender>
```

**Critical Rule:** Never log passwords, tokens, credit cards, SSNs, or other PII. Use parameterized logging for performance and security.

**Why:** Logging sensitive data exposes it in logs, violating compliance (GDPR, PCI) and security.  
**When to use:** Always when logging any data that might contain personal or sensitive information.  
**How to use:** Use parameterised logging to avoid string concatenation. Override `toString` to mask sensitive fields. Use Logback converters to redact patterns.

---

## Part 7: Testing & Debugging

### 42. Unit testing best practices — practical examples

**Practical Example:**
```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

class UserServiceTest {
    
    @Mock
    private UserRepository repository;
    @Mock
    private EmailService emailService;
    @InjectMocks
    private UserService service;
    
    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
    }
    
    @Test
    @DisplayName("Should create user successfully when email is unique")
    void shouldCreateUser_whenEmailIsUnique() {
        // Given
        UserDto dto = new UserDto("john@example.com", "John");
        when(repository.existsByEmail("john@example.com")).thenReturn(false);
        when(repository.save(any(User.class))).thenAnswer(inv -> inv.getArgument(0));
        
        // When
        User result = service.createUser(dto);
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.getEmail()).isEqualTo("john@example.com");
        verify(repository).save(any(User.class));
        verify(emailService).sendWelcomeEmail("john@example.com");
    }
    
    @Test
    void shouldThrowException_whenEmailAlreadyExists() {
        // Given
        UserDto dto = new UserDto("existing@example.com", "John");
        when(repository.existsByEmail("existing@example.com")).thenReturn(true);
        
        // When/Then
        assertThatThrownBy(() -> service.createUser(dto))
            .isInstanceOf(DuplicateEmailException.class)
            .hasMessage("Email already exists: existing@example.com");
        
        verify(repository, never()).save(any());
    }
    
    @ParameterizedTest
    @CsvSource({
        "john@example.com, true",
        "invalid-email, false",
        "user@, false"
    })
    void shouldValidateEmailCorrectly(String email, boolean expected) {
        assertThat(service.isValidEmail(email)).isEqualTo(expected);
    }
    
    @Test
    @Timeout(1)
    void shouldProcessWithinTimeout() {
        service.processLargeDataset();  // Fails if >1 second
    }
    
    @Test
    void testWithAssertAll() {
        User user = service.findById(1L);
        assertAll("user",
            () -> assertThat(user.getId()).isNotNull(),
            () -> assertThat(user.getEmail()).contains("@"),
            () -> assertThat(user.getCreatedAt()).isBefore(LocalDateTime.now())
        );
    }
}
```

**Best Practice:** Name tests clearly (`should...when...`), test one behavior per test, use parameterized tests for edge cases, and verify interactions with mocks.

**Why:** Good unit tests catch bugs early and serve as documentation.  
**When to use:** Write unit tests for all public methods and business logic.  
**How to use:** Follow Arrange‑Act‑Assert pattern. Use mocking to isolate dependencies. Use parameterized tests for multiple inputs. Assert both state and interactions.

---

### 43. Integration testing with Testcontainers — practical setup

**Practical Example:**
```java
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

@Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class OrderIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Container
    static RedisContainer redis = new RedisContainer("redis:7");
    
    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void shouldCreateOrder() throws Exception {
        String request = """
            {
                "userId": "user-123",
                "items": [
                    {"productId": "prod-1", "quantity": 2}
                ]
            }
            """;
        
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(request))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists())
            .andExpect(jsonPath("$.total").value(49.98));
        
        // Verify database state
        Order order = orderRepository.findByExternalId(extractedId);
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CREATED);
    }
}
```

**Best Practice:** Use Testcontainers for real database/redis testing. Avoid H2 for testing if production uses PostgreSQL.

**Why:** Real databases ensure compatibility and avoid H2‑specific quirks.  
**When to use:** For integration tests that require a real database, message broker, or external service.  
**How to use:** Use Testcontainers to spin up containers. Override properties with `@DynamicPropertySource`. Use `@Container` to manage lifecycle.

---

### 44. Heap dump analysis — practical memory leak debugging

**Practical Example:**
```bash
# Generate heap dump
jmap -dump:live,format=b,file=/tmp/heap.hprof <pid>

# Or with JCMD
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# Analyze with Eclipse MAT
# 1. Open heap.hprof
# 2. Run "Leak Suspects Report"
# 3. Look for:
#    - Classes with largest retained size
#    - Dominator tree for reference chains
#    - GC root paths

# Key metrics to check
jstat -gcutil <pid> 1000

#  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
#  0.00  42.86  23.45  78.32  94.56  89.23    125    2.345     5    1.234    3.579
# If O (Old Gen) > 80% and increasing → memory leak

# JFR for continuous monitoring
jcmd <pid> JFR.start name=memoryleak duration=5m filename=leak.jfr
```

**Common Leak Patterns:**
- **Large retained size:** Objects holding references to many others
- **Excessive instances:** Class with high instance count
- **Reference chains:** Objects held by GC roots (static fields, thread locals)

**Why:** Heap dumps help identify what is consuming memory and why objects are not being garbage collected.  
**When to use:** When you suspect a memory leak (e.g., OOM, increasing heap usage).  
**How to use:** Take a heap dump. Use Eclipse MAT to find leak suspects. Look at largest objects and their GC roots.

---

### 45. JMH benchmarking — practical performance testing

**Practical Example:**
```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.stream.IntStream;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(1)
public class StringConcatBenchmark {
    
    private static final int SIZE = 1000;
    private List<String> strings;
    
    @Setup
    public void setup() {
        strings = IntStream.range(0, SIZE)
            .mapToObj(i -> "string-" + i)
            .collect(Collectors.toList());
    }
    
    @Benchmark
    public String stringBuilderConcat() {
        StringBuilder sb = new StringBuilder(SIZE * 10);
        for (String s : strings) {
            sb.append(s).append(",");
        }
        return sb.toString();
    }
    
    @Benchmark
    public String stringPlusConcat() {
        String result = "";
        for (String s : strings) {
            result += s + ",";  // Creates new StringBuilder each iteration
        }
        return result;
    }
    
    @Benchmark
    public String streamCollectConcat() {
        return strings.stream().collect(Collectors.joining(","));
    }
    
    @Benchmark
    public String stringJoinConcat() {
        return String.join(",", strings);
    }
}

// Run: mvn clean install && java -jar target/benchmarks.jar
```

**Key Metrics:** Throughput, average time, percentile latency, allocations.

**Why:** JMH provides accurate microbenchmarking by controlling JVM warm‑up, forking, and measurement.  
**When to use:** When you need to compare performance of two implementations or optimise a hot path.  
**How to use:** Annotate with `@Benchmark`. Use `@Warmup`, `@Measurement`. Avoid common pitfalls like dead‑code elimination. Run with Maven plugin.

---

## Part 8: Design Patterns & Architecture

### 46. Builder pattern — practical implementation with validation

**Practical Example:**
```java
public class Order {
    private final String id;
    private final List<OrderItem> items;
    private final Address shippingAddress;
    private final Address billingAddress;
    private final PaymentMethod paymentMethod;
    private final LocalDateTime createdAt;
    
    private Order(Builder builder) {
        this.id = builder.id;
        this.items = List.copyOf(builder.items);
        this.shippingAddress = builder.shippingAddress;
        this.billingAddress = builder.billingAddress != null ? 
            builder.billingAddress : builder.shippingAddress;
        this.paymentMethod = builder.paymentMethod;
        this.createdAt = builder.createdAt != null ? 
            builder.createdAt : LocalDateTime.now();
        validate();
    }
    
    private void validate() {
        if (id == null || id.isBlank()) {
            throw new IllegalArgumentException("Order ID required");
        }
        if (items.isEmpty()) {
            throw new IllegalArgumentException("Order must have items");
        }
        if (shippingAddress == null) {
            throw new IllegalArgumentException("Shipping address required");
        }
    }
    
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        private String id;
        private List<OrderItem> items = new ArrayList<>();
        private Address shippingAddress;
        private Address billingAddress;
        private PaymentMethod paymentMethod;
        private LocalDateTime createdAt;
        
        public Builder id(String id) { this.id = id; return this; }
        public Builder addItem(OrderItem item) { this.items.add(item); return this; }
        public Builder shippingAddress(Address address) { this.shippingAddress = address; return this; }
        public Builder billingAddress(Address address) { this.billingAddress = address; return this; }
        public Builder paymentMethod(PaymentMethod method) { this.paymentMethod = method; return this; }
        
        public Order build() {
            return new Order(this);
        }
    }
}

// Usage
Order order = Order.builder()
    .id("ORD-123")
    .addItem(new OrderItem("PROD-1", 2))
    .shippingAddress(address)
    .paymentMethod(PaymentMethod.CREDIT_CARD)
    .build();
```

**Best Practice:** Use builder for objects with many parameters, especially optional ones. Validate in constructor, not builder.

**Why:** Builder pattern improves readability and prevents telescoping constructors.  
**When to use:** When a class has many parameters (especially optional ones), or when you need to enforce invariants after construction.  
**How to use:** Create a static inner `Builder` class with methods for each parameter. Validate in the private constructor. Use `build()` to create the instance.

---

### 47. Factory pattern — practical with Spring

**Practical Example:**
```java
// Strategy pattern with factory
public interface PaymentStrategy {
    PaymentResult process(PaymentRequest request);
    boolean supports(PaymentMethod method);
}

@Component
public class CreditCardStrategy implements PaymentStrategy {
    @Override
    public PaymentResult process(PaymentRequest request) { /* ... */ }
    @Override
    public boolean supports(PaymentMethod method) { 
        return method == PaymentMethod.CREDIT_CARD; 
    }
}

@Component
public class PayPalStrategy implements PaymentStrategy {
    @Override
    public PaymentResult process(PaymentRequest request) { /* ... */ }
    @Override
    public boolean supports(PaymentMethod method) { 
        return method == PaymentMethod.PAYPAL; 
    }
}

@Component
public class PaymentStrategyFactory {
    private final List<PaymentStrategy> strategies;
    
    public PaymentStrategyFactory(List<PaymentStrategy> strategies) {
        this.strategies = strategies;
    }
    
    public PaymentStrategy getStrategy(PaymentMethod method) {
        return strategies.stream()
            .filter(s -> s.supports(method))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("No strategy for " + method));
    }
}

@Service
public class PaymentService {
    private final PaymentStrategyFactory factory;
    
    public PaymentService(PaymentStrategyFactory factory) {
        this.factory = factory;
    }
    
    public PaymentResult process(PaymentRequest request) {
        PaymentStrategy strategy = factory.getStrategy(request.getMethod());
        return strategy.process(request);
    }
}
```

**Best Practice:** Use Spring's DI to auto-discover strategies. This makes adding new strategies easy without modifying existing code.

**Why:** Factory pattern encapsulates object creation logic, making code more maintainable.  
**When to use:** When creation logic is complex, or when you need to return different implementations based on input.  
**How to use:** Use Spring’s `@Component` and autowire a list of strategies. Implement a `supports()` method in each strategy. Factory selects the appropriate one.

---

### 48. Dependency injection — constructor injection best practices

**Practical Example:**
```java
// ✅ BEST: Constructor injection (immutable, testable)
@Service
public class UserService {
    private final UserRepository repository;
    private final EmailService emailService;
    private final PasswordEncoder passwordEncoder;
    
    public UserService(UserRepository repository, 
                       EmailService emailService,
                       PasswordEncoder passwordEncoder) {
        this.repository = repository;
        this.emailService = emailService;
        this.passwordEncoder = passwordEncoder;
    }
}

// ❌ AVOID: Field injection (hard to test, null safety issues)
@Service
public class BadUserService {
    @Autowired
    private UserRepository repository;  // Can't be final
    
    // Test requires reflection to set
}

// ❌ AVOID: Setter injection (mutable, less clear)
@Service
public class SetterUserService {
    private UserRepository repository;
    
    @Autowired
    public void setRepository(UserRepository repository) {
        this.repository = repository;
    }
}

// For optional dependencies, use Optional or @Autowired(required=false)
@Service
public class OptionalDependencyService {
    private final Logger logger;
    
    public OptionalDependencyService(Optional<MetricsCollector> metrics) {
        this.metrics = metrics.orElse(null);
    }
}
```

**Best Practice:** Use constructor injection exclusively. It makes dependencies explicit, enables immutability, and simplifies unit testing.

**Why:** Constructor injection ensures dependencies are provided at creation, making the object immutable and testable.  
**When to use:** Always use constructor injection in Spring (and in general) for mandatory dependencies.  
**How to use:** Declare fields as `final`. Use constructor parameters for all dependencies. For optional dependencies, use `Optional` parameters.

---

### 49. Repository pattern — practical implementation with Spring Data

**Practical Example:**
```java
// Base entity
@MappedSuperclass
public abstract class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;
    
    @Version
    private Long version;
    
    @CreatedDate
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
}

// Domain entity
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_user_status", columnList = "userId, status")
})
public class Order extends BaseEntity {
    private String userId;
    private BigDecimal total;
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    private LocalDateTime completedAt;
}

// Repository with custom queries
@Repository
public interface OrderRepository extends JpaRepository<Order, String> {
    
    // Derived query
    List<Order> findByUserIdAndStatus(String userId, OrderStatus status);
    
    // JPQL with pagination
    @Query("SELECT o FROM Order o WHERE o.userId = :userId AND o.createdAt >= :since")
    Page<Order> findByUserSince(@Param("userId") String userId, 
                                 @Param("since") LocalDateTime since,
                                 Pageable pageable);
    
    // Projection for performance
    @Query("SELECT o.id as id, o.total as total, o.status as status FROM Order o WHERE o.userId = :userId")
    List<OrderSummary> findOrderSummariesByUserId(@Param("userId") String userId);
    
    // Batch update
    @Modifying
    @Query("UPDATE Order o SET o.status = :status WHERE o.id IN :ids")
    int updateStatus(@Param("ids") List<String> ids, @Param("status") OrderStatus status);
}

// DTO projection
public interface OrderSummary {
    String getId();
    BigDecimal getTotal();
    OrderStatus getStatus();
}

// Specification for dynamic queries
public class OrderSpecifications {
    public static Specification<Order> hasUserId(String userId) {
        return (root, query, cb) -> cb.equal(root.get("userId"), userId);
    }
    
    public static Specification<Order> statusIn(List<OrderStatus> statuses) {
        return (root, query, cb) -> root.get("status").in(statuses);
    }
    
    public static Specification<Order> createdAfter(LocalDateTime date) {
        return (root, query, cb) -> cb.greaterThan(root.get("createdAt"), date);
    }
}
```

**Best Practice:** Use projections for read-only queries to avoid loading entire entities. Index columns used in WHERE clauses.

**Why:** Repository pattern abstracts data access, making code testable and maintainable.  
**When to use:** In any application with a database.  
**How to use:** Extend Spring Data JPA’s `JpaRepository`. Use derived queries for simple ones, `@Query` for complex ones. Use projections to reduce data transfer. Use `@Modifying` for updates.

---

### 50. Clean architecture — package structure for large projects

**Practical Structure:**
```
com.example.app/
├── domain/                     # Core business logic
│   ├── model/
│   │   ├── Order.java
│   │   ├── OrderItem.java
│   │   └── OrderStatus.java
│   ├── service/
│   │   ├── OrderService.java
│   │   └── interfaces/         # Ports
│   │       ├── OrderRepositoryPort.java
│   │       └── PaymentPort.java
│   └── exception/
│       └── DomainException.java
│
├── application/                # Use cases
│   ├── CreateOrderUseCase.java
│   ├── CancelOrderUseCase.java
│   └── dto/
│       ├── CreateOrderRequest.java
│       └── OrderResponse.java
│
├── infrastructure/             # External concerns
│   ├── persistence/
│   │   ├── JpaOrderRepository.java    # Implements port
│   │   ├── entity/
│   │   │   └── OrderEntity.java
│   │   └── mapper/
│   │       └── OrderMapper.java
│   ├── payment/
│   │   └── StripePaymentAdapter.java  # Implements port
│   ├── config/
│   │   └── SecurityConfig.java
│   └── messaging/
│       └── KafkaOrderProducer.java
│
└── interfaces/                 # API layer
    ├── rest/
    │   ├── OrderController.java
    │   └── dto/
    │       ├── ApiOrderRequest.java
    │       └── ApiOrderResponse.java
    └── listener/
        └── OrderEventListener.java
```

**Key Principles:**
- **Domain** has no external dependencies
- **Application** depends only on domain
- **Infrastructure** implements ports defined in domain
- **Interfaces** call application use cases

**Best Practice:** Keep domain pure. Use dependency inversion to decouple infrastructure from business logic.

**Why:** Clean architecture separates concerns, making the code maintainable, testable, and independent of frameworks.  
**When to use:** For large, complex applications expected to evolve over time.  
**How to use:** Place core business logic in `domain`. Define ports (interfaces) in domain. Implement them in `infrastructure`. Use `application` for use cases that orchestrate domain services.

---
```

Each Q&A now includes a short **“Why / When to use / How to use”** block (or a “Main crux” where appropriate) that summarises the essential reasoning and practical application. This addition makes the guide even more actionable and easier to review.