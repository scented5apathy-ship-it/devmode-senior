# Task 02 — Java Memory Model: Heap, Stack, Metaspace, GC Algorithms

> Bài nghiên cứu chi tiết về Java Memory Model và các thuật toán Garbage Collection hiện đại.
> Dành cho developers xây dựng hệ thống ngân hàng quy mô lớn.

---

## 1. Giới thiệu — Tại sao Memory Model quan trọng?

Trong Java, Memory Model là nền tảng quyết định cách JVM quản lý bộ nhớ trong suốt vòng đời ứng dụng. Không như C/C++ phải tự quản lý bộ nhớ thủ công (malloc/free), Java tự động cấp phát và thu hồi bộ nhớ thông qua **Garbage Collection (GC)**. Điều này giúp developers tránh được các lỗi nghiêm trọng như memory leak hay dangling pointers, nhưng đồng thời đặt ra thách thức riêng: **GC pauses** có thể gây ra latency spikes đáng kể, đặc biệt trong các hệ thống fintech yêu cầu thời gian phản hồi cực thấp (ultra-low latency).

Bài viết này khám phá sâu về:

- **Heap Memory** — nơi lưu trữ objects và arrays
- **Stack Memory** — nơi lưu trữ method frames và local variables
- **Metaspace** — vùng lưu trữ class metadata (thay thế PermGen từ Java 8)
- **Các GC algorithms hiện đại**: G1, ZGC, Shenandoah

---

## 2. Heap Memory — Trái tim của JVM

### 2.1. Cấu trúc Heap

Heap là vùng nhớ chính cho việc cấp phát động. Tất cả `new Object()`, `new int[]`, `new String()`... đều được lưu trữ tại đây. Heap là **shared memory** — tất cả threads đều có thể truy cập.

Heap được chia thành 2 thế hệ (generations) để tối ưu hóa việc thu gom rác:

| Vùng | Mô tả | Chi tiết |
|------|--------|---------|
| **Young Generation (Eden + Survivor Spaces)** | Nơi objects mới được tạo | Objects sống ngắn, thường bị thu hồi nhanh |
| **Old Generation** | Nơi objects "sống lâu" được chuyển đến | Objects tồn tại qua nhiều GC cycles |

```
┌─────────────────────────────────────────┐
│              HEAP MEMORY                 │
│  ┌─────────────────────────────────┐   │
│  │     YOUNG GENERATION             │   │
│  │  ┌─────────┐  ┌─────────────┐  │   │
│  │  │  Eden   │  │ Survivor S0 │  │   │
│  │  │         │  │             │  │   │
│  │  └─────────┘  └─────────────┘  │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │     OLD GENERATION               │   │
│  │     (Tenured/Old Gen)          │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### 2.2. Minor GC vs Major/Full GC

- **Minor GC**: Chỉ quét Young Generation. Vì Young gen nhỏ nên pause time rất ngắn (thường <50ms). Sau mỗi Minor GC, các objects còn sống trong Eden được chuyển sang Survivor space.

- **Major GC / Full GC**: Quét cảHeap. Major GC (Full GC) có thể mất hàng trăm ms甚至 hàng giây đối với heap lớn. Đây là nguyên nhân chính của latency spikes trong production.

---

## 3. Stack Memory — Memory riêng cho mỗi Thread

### 3.1. Cấu trúc Stack

Khác với Heap (chia sẻ), **Stack là private cho mỗi thread**. Mỗi thread khi được tạo sẽ có một Stack riêng.

Stack lưu trữ:

- **Method frames**: thông tin về method hiện tại đang chạy
- **Local variables**: biến primitives (`int`, `double`,...) và object references
- **Operand stack**: tạm thời tính toán
- **Frame data**: constant pool reference, return address

```java
public void processTransaction(Transaction tx) {
    // tx là reference trỏ đến object trong Heap
    // tx được lưu trong Stack như local variable
    BigDecimal amount = tx.getAmount();  // amount là primitive, lưu trực tiếp trong Stack
    
    BigDecimal fee = calculateFee(amount);  // fee cũng lưu trong Stack
    // ...
}
```

### 3.2. StackOverflowError

Mỗi thread có kích thước Stack giới hạn (mặc định ~1MB). Khi gọi đệ quy quá sâu, Stack sẽ đầy và ném `StackOverflowError`:

```java
// Ví dụ: đệ quy vô hạn -> StackOverflowError
public int recursiveSum(int n) {
    return n + recursiveSum(n - 1);  // KHÔNG có điều kiện dừng!
}
```

**Giải pháp**: Thêm điều kiện dừng hoặc dùng vòng lặp (loop) thay vì đệ quy.

---

## 4. Metaspace — Thay thế PermGen từ Java 8

### 4.1. PermGen vs Metaspace

Trước Java 8, JVM dùng **Permanent Generation (PermGen)** để lưu:

- Class metadata (class name, methods, fields...)
- Interned strings
- Bytecode của methods
- JIT compiled code

**Vấn đề của PermGen**:

- Kích thước cố định, đặt bằng `-XX:PermSize` và `-XX:MaxPermSize`
- Không thể mở rộng động → `OutOfMemoryError: PermGen space` khi load quá nhiều classes
- Gây khó khăn trong các ứng dụng Spring Boot hiện đại (load hàng ngàn classes)

**Giải pháp**. Từ Java 8, PermGen được **thay thế bằng Metaspace**:

| Đặc điểm | PermGen (≤Java 7) | Metaspace (Java 8+) |
|----------|-----------------|-------------------|
| Vị trí | Heap (JVM-managed) | Native memory (OS-managed) |
| Kích thước | Cố định (max ~85MB mặc định) | Động, có thể grow unlimited |
| Tuning | -XX:PermSize, -XX:MaxPermSize | -XX:MetaspaceSize, -XX:MaxMetaspaceSize |
| OOM | OutOfMemoryError: PermGen | OutOfMemoryError: Metaspace |

### 4.2. Cơ chế hoạt động của Metaspace

Metaspace sử dụng **native memory** của hệ điều hành thay vì Heap JVM. Điều này cho phép:

- Tự động mở rộng khi cần (miễn là còn RAM vật lý)
- Thu hồi bộ nhớ khi classes bị unload
- Giảm nguy cơ OOM do class loading quá mức

**Tuning Metaspace** (thường không cần thiết, nhưng có thể dùng trong production):

```bash
# Giới hạn max Metaspace <= 512MB
-XX:MaxMetaspaceSize=512m

# Kích thước ban đầu khi khởi động
-XX:MetaspaceSize=128m
```

---

## 5. Garbage Collection — Thu gom rác tự động

### 5.1. Nguyên lý hoạt động

GC hoạt động theo chu kỳ:

1. **Mark**: Đánh dấu tất cả objects còn sống (đang được tham chiếu)
2. **Sweep**: Thu hồi bộ nhớ của objects không còn tham chiếu
3. **Compact**: Gộp bộ nhớ rời rạc (giảm fragmentation)

```
Phase 1: MARK
┌───┬───┬───┬───┬───┐
│ X │ O │ O │ X │ O │  → X = dead, O = alive
├───┼───┼───┼───┼───┤
│Mark all reachable objects
└───┴───┴───┴───┴───┘

Phase 2: SWEEP
┌───┬───┬───┬───┬───┐
│   │███│███│   │███│  → ███ = freed memory
├───┼─���─��───┼───┼───┤
│Free unreachable objects
└───┴───┴───┴───┴───┘

Phase 3: COMPACT
┌───────────────────────┐
│████████████████       │  → Continuous memory block
└───────────────────────┘
```

### 5.2. Các loại GC trong Java

| GC Algorithm | Mô tả | Java Version | Đặc điểm |
|--------------|-------|-------------|-----------|
| **Serial GC** | Single-thread, dùng Stop-The-World | Tất cả | Đơn giản, phù hợp heap nhỏ |
| **Parallel GC (Throughput)** | Multi-thread, tối đa throughput | Tất cả | Cao throughput, pause lớn |
| **CMS (Concurrent Mark Sweep)** | Concurrent, giảm STW | 8 (deprecated) | Giảm pause nhưng có fragmentation |
| **G1 (Garbage First)** | Region-based, dùng cho large heaps | 9+ (default từ 11) | Predictable pause times |
| **ZGC** | Ultra-low latency, terabyte-scale | 15+ (production) | Pause <10ms, concurrent compaction |
| **Shenandoah** | Low pause, OpenJDK | 8+ (qua backport) | Pause ~few ms |

---

## 6. So sánh G1, ZGC, Shenandoah

### 6.1. G1 (Garbage-First) — Default của JVM hiện đại

**Thiết kế**: G1 chia Heap thành các vùng nhỏ (regions) cùng kích thước (~1-32MB). Thay vì quét toàn bộ Heap, G1 tập trung vào các vùng có nhiều "rác" nhất trước → **Garbage-First**.

**Ưu điểm**:

- Predictable pause times (có thể set `-XX:MaxGCPauseMillis`)
- Phù hợp heap từ vài GB đến ~4TB
- Cân bằng giữa throughput và latency
- Default từ Java 11

**Nhược điểm**:

- Vẫn có Full GC (Stop-The-World) nếu concurrent fails
- Không phù hợp latency cực thấp (<10ms)

**Use cases**: Microservices, web apps, enterprise apps

```bash
# Kích hoạt G1 (default từ Java 11)
-XX:+UseG1GC

# Set pause time goal = 200ms
-XX:MaxGCPauseMillis=200
```

### 6.2. ZGC — Ultra-low latency cho terabyte heaps

**Thiết kế**: ZGC sử dụng **colored pointers** và **load barriers** để theo dõi trạng thái objects trong quá trình di chuyển. Tất cả hoạt động marking và compaction đều **concurrent** với application threads.

**Ưu điểm**:

- Pause time cố định <10ms bất kể heap size
- Hỗ trợ heap lên đến multi-TB (8TB+)
- Không có Full GC events
- Highly scalable

**Nhược điểm**:

- Yêu cầu Java 15+ (production-ready)
- Throughput thấp hơn G1 ~10-15%
- Cần JVM tuning cẩn thận

**Use cases**: In-memory databases, real-time analytics, AI workloads, large-scale microservices

```bash
# Kích hoạt ZGC
-XX:+UseZGC

# Set heap size (ZGC hoạt động tốt với large heaps)
-Xmx32g

# Disable explicit GC (tránh System.gc() gây pause)
-XX:+DisableExplicitGC
```

### 6.3. Shenandoah — Low pause từ Red Hat

**Thiết kế**: Shenandoah tương tự ZGC về mục tiêu (minimize pause), nhưng dùng kỹ thuật ** Brooks pointer** (indirect pointer) để cho phép di chuyển objects concurrently mà không cần colored pointers.

**Ưu điểm**:

- Pause time ~few ms với heap lên đến ~2TB
- Compatible với nhiều JDK distributions (OpenJDK, Red Hat, Azul)
- Hỗ trợ Java 8+ (qua backport)

**Nhược điểm**:

- Throughput thấp hơn G1 ~10-20%
- Không hỗ trợ class data sharing
-ít tài liệu hơn ZGC

**Use cases**: Containerized apps, OpenJDK environments, latency-sensitive apps

```bash
# Kích hoạt Shenandoah
-XX:+UseShenandoahGC

# Set pause goal
-XX:MaxGCPauseMillis=10
```

### 6.4. So sánh tổng quan

| Tiêu chí | G1 | ZGC | Shenandoah |
|----------|-----|-----|-----------|
| **Pause Time** | Medium (~100-500ms) | Ultra-low (<10ms) | Low (~1-5ms) |
| **Throughput** | Cao | Trung bình | Trung bình |
| **Heap Size** | đến ~4TB | Multi-TB | ~2TB |
| **Concurrent Compaction** | Partial | Full | Full |
| **Java Version** | 9+ (default 11+) | 15+ | 8+ (backport) |
| **Complexity** | Trung bình | Cao | Trung bình |

---

## 7. Trade-offs — Đánh đổi trong GC

### 7.1. Throughput vs Latency

- **Throughput-oriented**: Parallel GC → Tối đa số lượng request xử lý, nhưng pause có thể lớn
- **Latency-oriented**: ZGC, Shenandoah → Pause nhỏ nhưng throughput thấp hơn

**Trong hệ thống ngân hàng**:

- **Core banking (batch processing)**: Ưu tiên throughput → Dùng G1 hoặc Parallel
- **Real-time payments (FPS/transfer)**: Ưu tiên latency → Dùng ZGC

### 7.2. Heap Size vs Pause Time

Heap lớn → nhiều objects → GC chậm hơn và pause dài hơn (đặc biệt với G1/Parallel).

**Giải pháp**:

- Dùng ZGC/Shenandoah cho large heaps
- Hoặc chia nhỏ thành nhiều microservices với heap riêng

### 7.3. CPU Usage

GC algorithms concurrent (ZGC, Shenandoah) tiêu tốn nhiều CPU hơn để chạy song song với application. Đây là trade-off chấp nhận được trong cloud environments với nhiều cores.

---

## 8. Failure Modes — Khi nào GC fail?

### 8.1. OutOfMemoryError: Java heap space

**Nguyên nhân**: Heap đầy, không thể cấp phát thêm object mới.

**Giải pháp**:

- Tăng heap size (`-Xmx`)
- Tối ưu object creation (tránh tạo object thừa)
- Điều chỉnh Old gen size (`-XX:OldSize`)
- Dùng profiler để tìm memory leaks

### 8.2. OutOfMemoryError: Metaspace

**Nguyên nhân**: Quá nhiều classes được load, vượt quá Metaspace limit.

**Giải pháp**:

- Tăng `-XX:MaxMetaspaceSize`
- Kiểm tra classloader leaks (thường trong app servers)

### 8.3. GC Overhead Limit Exceeded

**Nguyên nhân**: JVM spend >98% thời gian cho GC mà không tiến triển đáng kể.

**Giải pháp**:

- Giảm object allocation rate
- Tăng heap size
- Kiểm tra memory leaks

### 8.4. Long GC Pauses gâm timeout

**Trong hệ thống ngân hàng**: Transaction timeout (ví dụ: 30s) có thể bị breach nếu GC pause >30s.

**Giải pháp**:

- Dùng ZGC hoặc Shenandoah
- Giảm heap size (nếu đang quá lớn)
- Tăng `-XX:+AlwaysPreTouch` (pre-allocate heap)

---

## 9. Ứng dụng trong Hệ thống Ngân hàng

### 9.1. Yêu cầu đặc thù

Hệ thống fintech/banking có các yêu cầu khắc khe:

| Yêu cầu | Mục tiêu GC |
|---------|-------------|
| **Real-time payments** | Latency <10ms per transaction |
| **Batch processing** | Throughput cao, overnight processing |
| **High availability** | Không có downtime do GC |
| **Audit compliance** | Không được miss transactions |

### 9.2. Recommendations theo use case

| Use Case | GC khuyến nghị | Lý do |
|---------|----------------|-------|
| **Core Banking (batch)** | G1 hoặc Parallel | Throughput quan trọng hơn latency |
| **Payment Gateway** | ZGC | Ultra-low latency required |
| **Mobile Banking App** | G1 hoặc Shenandoah | Cân bằng latency/throughput |
| **Analytics/Reporting** | Parallel | Tối đa throughput |

### 9.3. Production Tuning Example

```bash
# Banking Payment Service - ZGC Configuration
# Heap 16GB (đủ lớn nhưng không quá lớn để avoid long GC)
-Xmx16g -Xms16g

# ZGC cho ultra-low latency
-XX:+UseZGC

# Disable explicit GC (tránh System.gc() gây pause)
-XX:+DisableExplicitGC

# Use G1 for fallback (nếu ZGC fails)
# -XX:+UseG1GC

# GC Logging để debug
-Xlog:gc*:file=/var/log/gc.log:time,uptime,level,tags
```

```bash
# Batch Processing - Parallel GC
# Tối đa throughput
-XX:+UseParallelGC

# Heap 32GB cho batch
-Xmx32g -Xms32g

# Full GC trước khi start để warm up
-XX:+AlwaysPreTouch
```

---

## 10. Ví dụ Code Java

### 10.1. Kiểm tra Memory Usage trong code

```java
import java.lang.management.ManagementFactory;
import java.lang.management.MemoryMXBean;
import java.lang.management.MemoryUsage;

public class MemoryMonitor {
    public static void main(String[] args) {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        
        // Lấy heap memory usage
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        System.out.println("Heap Memory:");
        System.out.println("  Used: " + bytesToMB(heapUsage.getUsed()) + " MB");
        System.out.println("  Max:  " + bytesToMB(heapUsage.getMax()) + " MB");
        System.out.println("  Committed: " + bytesToMB(heapUsage.getCommitted()) + " MB");
        
        // Lấy non-heap (Metaspace, etc.)
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
        System.out.println("\nNon-Heap Memory:");
        System.out.println("  Used: " + bytesToMB(nonHeapUsage.getUsed()) + " MB");
        System.out.println("  Max:  " + bytesToMB(nonHeapUsage.getMax()) + " MB");
    }
    
    private static long bytesToMB(long bytes) {
        return bytes / (1024 * 1024);
    }
}
```

### 10.2. Tạo Object và quan sát GC

```java
import java.util.ArrayList;
import java.util.List;

/**
 * Ví dụ tạo objects để quan sát GC behavior
 * Chạy với: -Xmx256m -XX:+UseG1GC -Xlog:gc*:gc.log
 */
public class ObjectCreator {
    // Static list giữ references -> objects không bị GC thu hồi
    static List<byte[]> leakList = new ArrayList<>();
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("Starting ObjectCreator...");
        System.out.println("Max Memory: " + Runtime.getRuntime().maxMemory() / (1024*1024) + " MB");
        
        int iteration = 0;
        while (true) {
            // Tạo objects để test memory allocation
            // Mỗi object 10MB
            byte[] data = new byte[10 * 1024 * 1024];
            leakList.add(data);
            
            iteration++;
            System.out.println("Iteration " + iteration + 
                ", List size: " + leakList.size() + 
                ", Free: " + Runtime.getRuntime().freeMemory() / (1024*1024) + " MB");
            
            Thread.sleep(1000);
        }
    }
}
```

### 10.3. JMX Monitoring

```java
import java.lang.management.ManagementFactory;
import java.lang.management.GarbageCollectorMXBean;
import java.util.List;

/**
 * Giám sát GC qua JMX
 */
public class GCMonitor {
    public static void main(String[] args) {
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        
        for (GarbageCollectorMXBean gc : gcBeans) {
            System.out.println("GC Name: " + gc.getName());
            System.out.println("  Collections: " + gc.getCollectionCount());
            System.out.println("  Collection Time: " + gc.getCollectionTime() + " ms");
            System.out.println("  Memory Pools: " + java.util.Arrays.toString(gc.getMemoryPoolNames()));
        }
    }
}
```

### 10.4. Tuning Example cho Banking

```java
import java.util.HashMap;
import java.util.Map;

/**
 * Demo: Object pooling để giảm GC pressure
 * Trong banking, tránh tạo quá nhiều objects ngắn sống
 */
public class TransactionPool {
    // Pool để reuse Transaction objects
    private static final int MAX_POOL_SIZE = 1000;
    private final Map<String, Transaction> pool = new HashMap<>();
    
    // Lấy transaction từ pool hoặc tạo mới nếu pool rỗng
    public synchronized Transaction borrow(String id) {
        Transaction tx = pool.remove(id);
        if (tx == null) {
            tx = new Transaction(id);
        }
        return tx;
    }
    
    // Trả transaction về pool
    public synchronized void release(Transaction tx) {
        if (pool.size() < MAX_POOL_SIZE) {
            // Reset trước khi reuse
            tx.reset();
            pool.put(tx.getId(), tx);
        }
        // Nếu pool đầy, cho GC thu hồi
    }
}

class Transaction {
    private String id;
    private double amount;
    
    public Transaction(String id) {
        this.id = id;
    }
    
    public String getId() { return id; }
    public void reset() { amount = 0; }
    public double getAmount() { return amount; }
    public void setAmount(double amount) { this.amount = amount; }
}
```

---

## 11. Kết luận

Java Memory Model và GC là nền tảng quan trọng cho việc xây dựng hệ thống hiệu năng cao. Các key takeaways:

1. **Heap vs Stack**: Heap chia sẻ, Stack private cho mỗi thread
2. **Metaspace**: Thay thế PermGen từ Java 8, dùng native memory
3. **G1**: Default hiện đại, cân bằng throughput/latency
4. **ZGC**: Ultra-low latency cho terabyte heaps
5. **Shenandoah**: Low pause, compatible với nhiều JDK
6. **Production**: Chọn GC dựa trên use case và latency requirements

Trong hệ thống ngân hàng:

- **Payment Gateway**: ZGC (ưu tiên latency)
- **Core Banking**: G1 hoặc Parallel (ưu tiên throughput)
- **Monitoring**: Dùng JFR, JMX để theo dõi GC behavior

---

## Tài liệu tham khảo

- Oracle Docs – G1 Garbage Collector: https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-g1-garbage-collector1.html
- Oracle Docs – ZGC: https://docs.oracle.com/en/java/javase/21/gctuning/z-garbage-collector.html
- OpenJDK Wiki – Shenandoah: https://wiki.openjdk.org/display/shenandoah/Main
- Java Code Geeks – G1 vs ZGC vs Shenandoah: https://www.javacodegeeks.com/2025/08/java-gc-performance-g1-vs-zgc-vs-shenandoah.html
- DigitalOcean – Java Memory Management: https://www.digitalocean.com/community/tutorials/java-jvm-memory-model-memory-management-in-java

---

> Bài viết thuộc series Java Developer Roadmap: Core → Senior (Banking-scale systems)
> Created: 2026-04-05