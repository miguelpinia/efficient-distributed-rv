# User Manual - Efficient Distributed Runtime Verification

**Version:** 1.0  
**Date:** December 2025  
**For:** RV 2025 Artifact Submission

---

## Table of Contents

1. [Introduction](#introduction)
2. [Installation](#installation)
3. [Quick Start](#quick-start)
4. [Core Concepts](#core-concepts)
5. [API Reference](#api-reference)
6. [Examples](#examples)
7. [Configuration](#configuration)
8. [Troubleshooting](#troubleshooting)
9. [Limitations](#limitations)
10. [References](#references)

---

## 1. Introduction

### What is This Tool?

This tool verifies **linearizability** of concurrent data structures. Linearizability is a correctness condition that ensures concurrent operations appear to execute atomically at some point between their invocation and response.

### Key Features

- **Two snapshot strategies**: GAI (Fetch-And-Increment) and RAW (Read-After-Write)
- **JIT-based verification** using undo operations for efficient backtracking
- **Support for Java concurrent collections** (ConcurrentLinkedQueue, ConcurrentHashMap, etc.)
- **Clean fluent API** for easy integration
- **Extensible architecture** for adding new algorithms

### What Makes This Tool Different?

- **Dual snapshot strategies**: Compare different event collection approaches
- **Undo operations**: Efficient backtracking without full state copying
- **Practical focus**: Works with real Java concurrent collections

---

## 2. Installation

### System Requirements

- **Java:** 21 or higher
- **Maven:** 3.x
- **Memory:** 2GB minimum, 4GB recommended
- **OS:** Linux, macOS, or Windows

### Build from Source

```bash
# Clone repository
git clone <repository-url>
cd efficient-distributed-rv

# Build
mvn clean install

# Run tests
mvn test
```

### Verify Installation

```bash
# Should show "BUILD SUCCESS"
mvn compile

# Should show "Tests run: 78, Failures: 0, Errors: 0"
mvn test
```

---

## 3. Quick Start

### Example 1: Basic Verification

```java
import phd.distributed.api.*;
import phd.distributed.core.*;

public class BasicExample {
    public static void main(String[] args) {
        // 1. Create algorithm wrapper
        DistAlgorithm algorithm = new A("java.util.concurrent.ConcurrentLinkedQueue");
        
        // 2. Create executioner (4 threads, 100 operations, queue type)
        Executioner exec = new Executioner(4, 100, algorithm, "queue");
        
        // 3. Run concurrent operations
        exec.taskProducers();
        
        // 4. Verify linearizability
        boolean isLinearizable = exec.taskVerifiers();
        
        System.out.println("Linearizable: " + isLinearizable);
    }
}
```

### Example 2: Using High-Level API

```java
import phd.distributed.api.*;

public class HighLevelExample {
    public static void main(String[] args) {
        VerificationResult result = VerificationFramework
            .verify(java.util.concurrent.ConcurrentLinkedQueue.class)
            .withThreads(4)
            .withOperations(100)
            .withObjectType("queue")
            .run();
        
        System.out.println("Linearizable: " + result.isCorrect());
        System.out.println("Time: " + result.getExecutionTime().toMillis() + " ms");
    }
}
```

### Example 3: Different Snapshot Strategy

```java
// Use RAW snapshot instead of default GAI
Executioner exec = new Executioner(4, 100, algorithm, "queue", "rawsnap");
exec.taskProducers();
boolean result = exec.taskVerifiers();
```

---

## 4. Core Concepts

A distributed monitoring algorithm consists of three layers: 
instrumentation, monitoring and specification. 
In the instrumenation layer are developed the mechanisms to obtain the current execution to verify, the monitoring layer consists on decide giving an specification, if the execution is correct or not. 

### 4.2 Instrumentation: Wrappers (Happens-before detection objects)

A wrapper is a distributed algorithm that allows that every process that execute this to construct a happens-before (partial) order of the operations invoked so far. 
In comparison to other instrumentation algorithms (instrumentation in AspectJ analyzed by El-Hoyakem and Falcone (2018)), if the execution produced by the wrapper is not linearizable, then the concurrent algorithm under inspection it is always linearizable.

Developed for first time in Castañeda and Rodriguez (2023), the main stucture is the following:

1. Write invocation of the method call in a shared memory (*non-linearizable snapshot*)
2. Apply the method call to the concurrent algorithm under inspection to obtain a response
3. Share the response and all the precedent invocations written so far in the shared memory (*non-linearizable snapshot*)


### 4.1 Specification: Linearizability

A concurrent execution is linearizable if:

1. Each operation appears to take effect instantaneously at some point between its invocation and response
2. The order of non-overlapping operations is preserved
3. The sequential execution respects the sequential specification



### 4.3 Snapshot Strategies

We use non-linearizable snapshot strategies based on a simple,
decoupled collect mechanism.
This approach was introduced by Rodríguez and Castañeda (2024),
where it was shown that a collect object, even though it is
not linearizable, is sufficient to runtime verify linearizability.

We consider two snapshot strategies:

**GAI (Get-And-Increment):**

- Uses an atomic fetch-and-increment operation to order events.
- Provides a total order over collected events.
- Has low overhead, making it suitable for lightweight instrumentation.


**RAW (Read-After-Write):**

- Uses a read-after-write mechanism to order events.
- The read operation returns a pointer to an immutable vector
implemented using Clojure’s persistent data structures.
- Captures happens-before relationships between events.
- Provides more precise causality information than GAI.

### 4.4 JIT-based Checking

The verifier checks whether a concurrent history of operations over a
shared object is **linearizable** with respect to a given sequential
specification.
It is based on the *Just-In-Time* tree search algorithm for
linearizability testing proposed by Lowe (2017), adapted to a functional
setting in Clojure.

- Works incrementally and supports partial histories, including pending operations.
- Uses a backtrackable sequential specification: states are immutable, so backtracking is achieved by reusing previous states rather than explicitly undoing operations.

Traditional linearizability checkers typically explore the state space by:

- Creating full copies of the sequential state,
- Restoring saved states during backtracking,
- Which can be expensive for large state spaces.



### 4.4 Object Types

The tool supports different data structure types:
- `"queue"` - FIFO queues (enqueue/dequeue)
- `"stack"` - LIFO stacks (push/pop)
- `"set"` - Sets (add/remove/contains)
- `"map"` - Maps (put/get/remove)
- `"deque"` - Double-ended queues (offerFirst/offerLast/pollFirst/pollLast)

---

## 5. API Reference

### 5.1 Executioner Class

Main class for verification workflow.

**Constructor:**

```java
Executioner(int threads, int operations, DistAlgorithm algorithm, String objectType)
Executioner(int threads, int operations, DistAlgorithm algorithm, String objectType, String snapType)
```

**Parameters:**

- `threads` - Number of concurrent threads
- `operations` - Total operations to execute
- `algorithm` - Algorithm wrapper (use class `A`)
- `objectType` - Type of data structure ("queue", "map", "set", etc.)
- `snapType` - Snapshot strategy ("gaisnap" or "rawsnap", default: "gaisnap")

**Methods:**

- `taskProducers()` - Execute concurrent operations without a workload
- `taskSeedProducers()` - Execute concurrent operations with a workload
- `taskVerifiers()` - Verify linearizability (returns boolean)

### 5.2 VerificationFramework Class

High-level fluent API.

**Static Methods:**

```java
VerificationBuilder verify(Class<?> algorithmClass)
VerificationBuilder verify(String className)
VerificationBuilder verify(Object instance)
```

**Builder Methods:**

```java
withThreads(int threads)
withOperations(int operations)
withObjectType(String type)
withSnapshot(String snapType)
withTimeout(Duration timeout)
run()  // Returns VerificationResult
runAsync()  // Returns CompletableFuture<VerificationResult>
```

### 5.3 VerificationResult Class

Result object with verification details.

**Methods:**

- `isCorrect()` / `isLinearizable()` - Verification result
- `getExecutionTime()` - Duration of verification
- `getStatistics()` - Execution statistics

### 5.4 AlgorithmLibrary Class

Registry of built-in algorithms.

**Methods:**

```java
static List<AlgorithmInfo> listAll()
static List<AlgorithmInfo> byCategory(Category category)
static AlgorithmInfo getInfo(String name)
```

---

## 6. Examples

### Example 1: Verify ConcurrentHashMap

```java
DistAlgorithm algorithm = new A("java.util.concurrent.ConcurrentHashMap", 
                                 "put", "get", "remove");
Executioner exec = new Executioner(4, 100, algorithm, "map");
exec.taskProducers();
boolean result = exec.taskVerifiers();
```

### Example 2: Verify ConcurrentSkipListSet

```java
DistAlgorithm algorithm = new A("java.util.concurrent.ConcurrentSkipListSet",
                                 "add", "remove", "contains");
Executioner exec = new Executioner(4, 100, algorithm, "set");
exec.taskProducers();
boolean result = exec.taskVerifiers();
```

### Example 3: Compare Snapshot Strategies

```java
DistAlgorithm algorithm = new A("java.util.concurrent.ConcurrentLinkedQueue");

// Test with GAI
Executioner execGAI = new Executioner(4, 100, algorithm, "queue", "gaisnap");
execGAI.taskProducers();
boolean resultGAI = execGAI.taskVerifiers();

// Test with RAW
Executioner execRAW = new Executioner(4, 100, algorithm, "queue", "rawsnap");
execRAW.taskProducers();
boolean resultRAW = execRAW.taskVerifiers();

System.out.println("GAI: " + resultGAI + ", RAW: " + resultRAW);
```

### Example 4: Detect Non-Linearizable Implementation

```java
// BrokenQueue is intentionally non-linearizable
DistAlgorithm algorithm = new A("phd.distributed.verifier.BrokenQueue");
Executioner exec = new Executioner(4, 100, algorithm, "queue");
exec.taskProducers();
boolean result = exec.taskVerifiers();
// Should print: false
```

---

## 7. Configuration

### 7.1 Feature Flags

Edit `src/main/resources/system.properties`:

```properties
# Enable/disable features
feature.parallel.verification=false
feature.smart.pruning=false
feature.result.caching=false

# System settings
system.thread.pool.size=8
system.batch.size=100
```

### 7.2 Logging Configuration

Edit `src/main/resources/log4j2.xml` for logging levels.

---

## 8. Troubleshooting

### Problem: OutOfMemoryError

**Solution:** Reduce operations or increase heap size:

```bash
java -Xmx4g -cp ... YourClass
```

### Problem: Tests fail to compile

**Solution:** Ensure Java 21 is installed:

```bash
java -version  # Should show 21 or higher
```

### Problem: Verification takes too long

**Solution:** Reduce operations or threads:

```java
Executioner exec = new Executioner(2, 50, algorithm, "queue");
```

### Problem: "Class not found" error

**Solution:** Use fully qualified class name:

```java
DistAlgorithm algorithm = new A("java.util.concurrent.ConcurrentLinkedQueue");
```

---

## 9. Limitations

### Current Limitations

1. **Sequential verification only**
   '- Parallel verification infrastructure exists but not integrated
   - Single-threaded verification of event history

2. **In-memory datasets only**
   - Streaming infrastructure exists but not functional
   - Limited by available memory

3. **No performance benchmarks**
   '- No comparison with other tools (AspectJ, Kater, Line-Up, etc.)
   - No comparison with other tools (AspectJ, etc.)
   - Performance characteristics not formally measured

4. **Limited test coverage**
   - 78 tests cover core functionality
   - Optimization features not tested

### Known Issues

1. **Clojure initialization**
   - Some tests require Clojure runtime initialization
   - May see warnings in logs (can be ignored)

2. **Large state spaces**
   - Very large operations counts (>10,000) may be slow
   - Consider reducing operations for testing

### Future Work

1. **Parallel verification integration**
   - Connect ParallelVerifier to JITLinUndoTester
   - Achieve actual speedup

2. **Reactive streaming integration**
   - Enable large-scale verification
   - Reduce memory footprint

3. **Performance benchmarking**
   - Compare with existing tools
   - Measure and optimize

4. **Extended test suite**
   - Test all features
   - Increase coverage



## 10. References

- **Castañeda, Armando and Rodríguez, Gilde Valeria.** (2023). *Asynchronous Wait-Free Runtime Verification and Enforcement of Linearizability.* In Proceedings of the 2023 ACM Symposium on Principles of Distributed Computing (PODC '23). Association for Computing Machinery, New York, NY, USA, 90–101. [https://doi.org/10.1145/3583668.3594563]

- **Lowe, Gavin.** *Testing for Linearizability.*
  *Concurrency and Computation: Practice and Experience,* Vol. 29, No. 4, Article e3928, 2017.
  ISSN: 1532-0626.
  [https://doi.org/10.1002/cpe.3928](https://doi.org/10.1002/cpe.3928)

- **Rodríguez, Gilde Valeria and Castañeda, Armando** (2024). *Towards Efficient Runtime Verified Linearizable Algorithms.* In: Ábrahám, E., Abbas, H. (eds) Runtime Verification. RV 2024. Lecture Notes in Computer Science, vol 15191. Springer, Cham. [https://doi.org/10.1007/978-3-031-74234-7_17]


## Appendix A: Supported Algorithms

### Java Concurrent Collections

- ConcurrentLinkedQueue
- ConcurrentHashMap
- ConcurrentLinkedDeque
- LinkedBlockingQueue
- ConcurrentSkipListSet
- LinkedTransferQueue
- ConcurrentSkipListMap
- LinkedBlockingDeque
- ArrayBlockingQueue
- PriorityBlockingQueue
- CopyOnWriteArraySet


## Appendix B: Method Names by Object Type

### Queue Methods
- `offer`, `poll`, `peek`
- `add`, `remove`, `element`

### Map Methods
- `put`, `get`, `remove`
- `containsKey`, `containsValue`

### Set Methods
- `add`, `remove`, `contains`

### Deque Methods
- `offerFirst`, `offerLast`
- `pollFirst`, `pollLast`
- `peekFirst`, `peekLast`

---

**End of User Manual**
