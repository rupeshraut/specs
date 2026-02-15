# Effective Ways of Using CompletableFuture

A comprehensive guide to mastering asynchronous programming in Java with CompletableFuture.

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Creating CompletableFuture](#creating-completablefuture)
3. [Transforming Results](#transforming-results)
4. [Combining Multiple Futures](#combining-multiple-futures)
5. [Error Handling](#error-handling)
6. [Async vs Non-Async Methods](#async-vs-non-async-methods)
7. [Thread Pool Management](#thread-pool-management)
8. [Real-World Patterns](#real-world-patterns)
9. [Performance Optimization](#performance-optimization)
10. [Common Pitfalls](#common-pitfalls)
11. [Testing CompletableFuture](#testing-completablefuture)

---

## Core Concepts

### What is CompletableFuture?

`CompletableFuture<T>` represents a future result of an asynchronous computation that can be:
- **Completed manually**
- **Composed with other futures**
- **Handled with callbacks**
- **Combined in various ways**

### Key Characteristics

```java
// 1. Implements Future and CompletionStage
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
    // Manual completion
    public boolean complete(T value)
    public boolean completeExceptionally(Throwable ex)
    
    // Blocking retrieval (avoid in production!)
    public T get() throws InterruptedException, ExecutionException
    public T get(long timeout, TimeUnit unit)
    
    // Non-blocking retrieval
    public T getNow(T valueIfAbsent)
    public T join()  // Like get() but throws unchecked exception
}
```

### When to Use CompletableFuture

```
✅ Use When:
- Making parallel API calls
- I/O-bound operations (database, HTTP, file)
- Non-blocking event processing
- Composing multiple async operations
- Building reactive pipelines

❌ Avoid When:
- CPU-bound tasks in default ForkJoinPool (use custom executor)
- Simple sequential operations
- Operations that complete instantly
- When virtual threads (Project Loom) are available and simpler
```

---

## Creating CompletableFuture

### 1. **Completed Futures (Already Done)**

```java
public class CompletedFutures {
    
    // Immediately completed with value
    public CompletableFuture<String> getCompletedFuture() {
        return CompletableFuture.completedFuture("result");
    }
    
    // Immediately failed future
    public CompletableFuture<String> getFailedFuture() {
        return CompletableFuture.failedFuture(
            new RuntimeException("Simulated failure")
        );
    }
    
    // Use case: Caching
    private final Map<String, CompletableFuture<User>> cache = new ConcurrentHashMap<>();
    
    public CompletableFuture<User> getUser(String id) {
        User cached = localCache.get(id);
        if (cached != null) {
            return CompletableFuture.completedFuture(cached);  // No async work needed
        }
        return fetchUserFromDatabase(id);
    }
}
```

### 2. **Async Suppliers (Deferred Execution)**

```java
public class AsyncSuppliers {
    
    // runAsync - for Runnable (no return value)
    public CompletableFuture<Void> asyncTask() {
        return CompletableFuture.runAsync(() -> {
            System.out.println("Running on: " + Thread.currentThread().getName());
            // Do some work
        });
    }
    
    // supplyAsync - for Supplier (returns value)
    public CompletableFuture<String> fetchData() {
        return CompletableFuture.supplyAsync(() -> {
            // Simulate API call
            sleep(1000);
            return "data from API";
        });
    }
    
    // With custom executor
    private final ExecutorService executor = Executors.newFixedThreadPool(10);
    
    public CompletableFuture<UserProfile> loadProfile(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            return databaseService.fetchUserProfile(userId);
        }, executor);  // Use custom thread pool
    }
}
```

### 3. **Manual Completion**

```java
public class ManualCompletion {
    
    // Pattern: Wrapping callback-based APIs
    public CompletableFuture<String> callbackToFuture() {
        CompletableFuture<String> future = new CompletableFuture<>();
        
        legacyApi.fetchData(new Callback() {
            @Override
            public void onSuccess(String result) {
                future.complete(result);  // Complete with value
            }
            
            @Override
            public void onFailure(Exception e) {
                future.completeExceptionally(e);  // Complete with error
            }
        });
        
        return future;
    }
    
    // Pattern: Timeout with manual completion
    public CompletableFuture<String> fetchWithTimeout(long timeoutMs) {
        CompletableFuture<String> future = new CompletableFuture<>();
        
        // Start async operation
        CompletableFuture.runAsync(() -> {
            String result = slowOperation();
            future.complete(result);
        });
        
        // Timeout mechanism
        scheduler.schedule(() -> {
            future.completeExceptionally(new TimeoutException("Operation timed out"));
        }, timeoutMs, TimeUnit.MILLISECONDS);
        
        return future;
    }
    
    // Pattern: Circuit breaker
    public CompletableFuture<String> withCircuitBreaker(Supplier<String> operation) {
        CompletableFuture<String> future = new CompletableFuture<>();
        
        if (circuitBreaker.isOpen()) {
            future.completeExceptionally(new CircuitBreakerOpenException());
            return future;
        }
        
        try {
            String result = operation.get();
            future.complete(result);
            circuitBreaker.recordSuccess();
        } catch (Exception e) {
            future.completeExceptionally(e);
            circuitBreaker.recordFailure();
        }
        
        return future;
    }
}
```

---

## Transforming Results

### 1. **thenApply() - Transform Value**

```java
public class TransformationExamples {
    
    // Simple transformation
    public CompletableFuture<Integer> getStringLength() {
        return CompletableFuture.supplyAsync(() -> "hello")
            .thenApply(String::length);  // Transform String to Integer
    }
    
    // Chaining transformations
    public CompletableFuture<UserDTO> getUserDTO(String userId) {
        return userRepository.findById(userId)              // CF<User>
            .thenApply(this::enrichWithProfile)             // CF<User>
            .thenApply(this::convertToDTO);                 // CF<UserDTO>
    }
    
    // Real-world example: Payment processing
    public CompletableFuture<PaymentReceipt> processPayment(PaymentRequest request) {
        return validatePayment(request)                      // CF<ValidatedPayment>
            .thenApply(this::calculateFees)                  // CF<PaymentWithFees>
            .thenApply(this::chargeCustomer)                 // CF<ChargeResult>
            .thenApply(this::generateReceipt);               // CF<PaymentReceipt>
    }
}
```

### 2. **thenCompose() - Flatten Nested Futures**

```java
public class CompositionExamples {
    
    // WRONG - Creates nested CompletableFuture<CompletableFuture<User>>
    public CompletableFuture<CompletableFuture<User>> wrongWay(String orderId) {
        return orderService.getOrder(orderId)                // CF<Order>
            .thenApply(order -> userService.getUser(order.getUserId()));  // CF<CF<User>> ❌
    }
    
    // RIGHT - Use thenCompose to flatten
    public CompletableFuture<User> rightWay(String orderId) {
        return orderService.getOrder(orderId)                // CF<Order>
            .thenCompose(order -> userService.getUser(order.getUserId()));  // CF<User> ✅
    }
    
    // Real-world example: Dependent API calls
    public CompletableFuture<OrderSummary> getOrderSummary(String orderId) {
        return orderService.getOrder(orderId)                // CF<Order>
            .thenCompose(order -> 
                userService.getUser(order.getUserId())       // CF<User>
                    .thenCompose(user ->
                        paymentService.getPayment(order.getPaymentId())  // CF<Payment>
                            .thenApply(payment -> 
                                new OrderSummary(order, user, payment)
                            )
                    )
            );
    }
    
    // Better approach - combine multiple futures (see next section)
    public CompletableFuture<OrderSummary> getOrderSummaryOptimized(String orderId) {
        return orderService.getOrder(orderId)
            .thenCompose(order -> {
                CompletableFuture<User> userFuture = userService.getUser(order.getUserId());
                CompletableFuture<Payment> paymentFuture = paymentService.getPayment(order.getPaymentId());
                
                return userFuture.thenCombine(paymentFuture, 
                    (user, payment) -> new OrderSummary(order, user, payment)
                );
            });
    }
}
```

### 3. **thenAccept() and thenRun() - Side Effects**

```java
public class SideEffectExamples {
    
    // thenAccept - consume result (no return value)
    public CompletableFuture<Void> processAndLog(String userId) {
        return userService.getUser(userId)
            .thenAccept(user -> {
                logger.info("Processing user: {}", user.getName());
                metricsService.recordUserAccess(user.getId());
            });  // Returns CompletableFuture<Void>
    }
    
    // thenRun - execute action (doesn't receive result)
    public CompletableFuture<Void> cleanupAfterProcess() {
        return processData()
            .thenRun(() -> {
                logger.info("Processing complete");
                cleanupResources();
            });
    }
    
    // Combining side effects with transformations
    public CompletableFuture<Order> placeOrder(OrderRequest request) {
        return orderService.createOrder(request)
            .thenApply(order -> {
                // Side effect during transformation
                eventPublisher.publish(new OrderCreatedEvent(order.getId()));
                return order;
            })
            .thenApply(this::enrichWithDefaults)
            .thenApply(order -> {
                // Another side effect
                notificationService.notifyCustomer(order.getCustomerId());
                return order;
            });
    }
}
```

---

## Combining Multiple Futures

### 1. **thenCombine() - Combine Two Futures**

```java
public class CombiningTwoFutures {
    
    // Combine two independent async operations
    public CompletableFuture<OrderSummary> getOrderSummary(String orderId) {
        CompletableFuture<Order> orderFuture = orderService.getOrder(orderId);
        CompletableFuture<Customer> customerFuture = customerService.getCustomer(customerId);
        
        return orderFuture.thenCombine(customerFuture, 
            (order, customer) -> new OrderSummary(order, customer)
        );
    }
    
    // With transformation
    public CompletableFuture<BigDecimal> calculateTotalPrice(String orderId) {
        CompletableFuture<Order> orderFuture = orderService.getOrder(orderId);
        CompletableFuture<Discount> discountFuture = discountService.getDiscount(customerId);
        
        return orderFuture.thenCombine(discountFuture,
            (order, discount) -> order.getTotal().subtract(discount.getAmount())
        );
    }
    
    // Real-world: Parallel data enrichment
    public CompletableFuture<EnrichedUser> enrichUserData(String userId) {
        CompletableFuture<User> userFuture = userService.getUser(userId);
        CompletableFuture<List<Order>> ordersFuture = orderService.getUserOrders(userId);
        CompletableFuture<Preferences> prefsFuture = preferencesService.getPreferences(userId);
        
        return userFuture
            .thenCombine(ordersFuture, (user, orders) -> {
                user.setOrders(orders);
                return user;
            })
            .thenCombine(prefsFuture, (user, prefs) -> {
                user.setPreferences(prefs);
                return new EnrichedUser(user);
            });
    }
}
```

### 2. **allOf() - Wait for All Futures**

```java
public class AllOfExamples {
    
    // Basic allOf usage
    public CompletableFuture<Void> waitForAll() {
        CompletableFuture<String> future1 = fetchData1();
        CompletableFuture<String> future2 = fetchData2();
        CompletableFuture<String> future3 = fetchData3();
        
        return CompletableFuture.allOf(future1, future2, future3);
        // Returns when ALL complete (or any fails)
    }
    
    // Collect results from multiple futures
    public CompletableFuture<List<User>> getAllUsers(List<String> userIds) {
        List<CompletableFuture<User>> futures = userIds.stream()
            .map(userService::getUser)
            .collect(Collectors.toList());
        
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)  // Safe because allOf completed
                .collect(Collectors.toList())
            );
    }
    
    // Real-world: Parallel API calls with aggregation
    public CompletableFuture<DashboardData> getDashboardData(String userId) {
        CompletableFuture<UserProfile> profileFuture = profileService.getProfile(userId);
        CompletableFuture<List<Order>> ordersFuture = orderService.getRecentOrders(userId);
        CompletableFuture<List<Notification>> notifsFuture = notificationService.getNotifications(userId);
        CompletableFuture<AccountBalance> balanceFuture = accountService.getBalance(userId);
        
        return CompletableFuture.allOf(profileFuture, ordersFuture, notifsFuture, balanceFuture)
            .thenApply(v -> new DashboardData(
                profileFuture.join(),
                ordersFuture.join(),
                notifsFuture.join(),
                balanceFuture.join()
            ));
    }
    
    // Helper method for better type safety
    public static <T> CompletableFuture<List<T>> allOf(List<CompletableFuture<T>> futures) {
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList())
            );
    }
}
```

### 3. **anyOf() - Wait for First Completion**

```java
public class AnyOfExamples {
    
    // Return first successful result
    public CompletableFuture<String> getFastestResponse() {
        CompletableFuture<String> api1 = callApi1();
        CompletableFuture<String> api2 = callApi2();
        CompletableFuture<String> api3 = callApi3();
        
        return CompletableFuture.anyOf(api1, api2, api3)
            .thenApply(result -> (String) result);  // Type unsafe!
    }
    
    // Type-safe version
    public CompletableFuture<String> getFastestResponseTypeSafe() {
        List<CompletableFuture<String>> futures = List.of(
            callApi1(),
            callApi2(),
            callApi3()
        );
        
        return anyOfTypeSafe(futures);
    }
    
    @SuppressWarnings("unchecked")
    private static <T> CompletableFuture<T> anyOfTypeSafe(List<CompletableFuture<T>> futures) {
        return (CompletableFuture<T>) CompletableFuture.anyOf(
            futures.toArray(new CompletableFuture[0])
        );
    }
    
    // Real-world: Multi-region fallback
    public CompletableFuture<Data> getDataWithFallback() {
        CompletableFuture<Data> primaryRegion = fetchFromRegion("us-east-1");
        CompletableFuture<Data> fallbackRegion = fetchFromRegion("us-west-2");
        CompletableFuture<Data> cache = fetchFromCache();
        
        return CompletableFuture.anyOf(primaryRegion, fallbackRegion, cache)
            .thenApply(result -> (Data) result);
    }
    
    // Pattern: Timeout with fallback
    public CompletableFuture<String> withTimeout(long timeoutMs) {
        CompletableFuture<String> operation = slowOperation();
        CompletableFuture<String> timeout = timeoutAfter(timeoutMs, "TIMEOUT");
        
        return CompletableFuture.anyOf(operation, timeout)
            .thenApply(result -> (String) result);
    }
    
    private <T> CompletableFuture<T> timeoutAfter(long millis, T value) {
        CompletableFuture<T> future = new CompletableFuture<>();
        scheduler.schedule(() -> future.complete(value), millis, TimeUnit.MILLISECONDS);
        return future;
    }
}
```

### 4. **Complex Combinations**

```java
public class ComplexCombinations {
    
    // Fan-out, then fan-in pattern
    public CompletableFuture<ReportData> generateReport(String customerId) {
        // Fan-out: Start multiple parallel operations
        CompletableFuture<CustomerProfile> profileFuture = 
            profileService.getProfile(customerId);
        
        CompletableFuture<List<Transaction>> transactionsFuture = 
            transactionService.getTransactions(customerId);
        
        CompletableFuture<CreditScore> creditFuture = 
            creditService.getCreditScore(customerId);
        
        CompletableFuture<List<Account>> accountsFuture = 
            accountService.getAccounts(customerId);
        
        // Fan-in: Combine all results
        return CompletableFuture.allOf(
                profileFuture, transactionsFuture, creditFuture, accountsFuture
            )
            .thenApply(v -> new ReportData(
                profileFuture.join(),
                transactionsFuture.join(),
                creditFuture.join(),
                accountsFuture.join()
            ))
            .thenCompose(reportData -> 
                // Additional async processing on combined data
                enrichmentService.enrichReport(reportData)
            );
    }
    
    // Conditional execution based on results
    public CompletableFuture<OrderResult> processOrder(OrderRequest request) {
        return inventoryService.checkAvailability(request.getProductId())
            .thenCompose(available -> {
                if (available) {
                    return paymentService.processPayment(request.getPaymentInfo())
                        .thenCompose(payment -> 
                            orderService.createOrder(request, payment)
                        );
                } else {
                    return CompletableFuture.completedFuture(
                        OrderResult.outOfStock()
                    );
                }
            });
    }
    
    // Parallel processing with different types
    public CompletableFuture<CheckoutSummary> prepareCheckout(String cartId) {
        CompletableFuture<Cart> cartFuture = cartService.getCart(cartId);
        
        return cartFuture.thenCompose(cart -> {
            // Start multiple dependent operations in parallel
            CompletableFuture<BigDecimal> taxFuture = 
                taxService.calculateTax(cart);
            
            CompletableFuture<BigDecimal> shippingFuture = 
                shippingService.calculateShipping(cart);
            
            CompletableFuture<List<Discount>> discountsFuture = 
                discountService.getApplicableDiscounts(cart);
            
            // Combine all
            return taxFuture.thenCombine(shippingFuture, (tax, shipping) -> 
                    Map.of("tax", tax, "shipping", shipping)
                )
                .thenCombine(discountsFuture, (charges, discounts) -> 
                    new CheckoutSummary(cart, charges, discounts)
                );
        });
    }
}
```

---

## Error Handling

### 1. **exceptionally() - Handle Errors**

```java
public class ExceptionallyExamples {
    
    // Basic error recovery
    public CompletableFuture<String> fetchDataWithFallback() {
        return apiClient.fetchData()
            .exceptionally(ex -> {
                logger.error("API call failed", ex);
                return "default-value";  // Fallback value
            });
    }
    
    // Different fallback based on exception type
    public CompletableFuture<User> getUserWithFallback(String userId) {
        return userRepository.findById(userId)
            .exceptionally(ex -> {
                if (ex instanceof NotFoundException) {
                    return User.guestUser();
                } else if (ex instanceof TimeoutException) {
                    return User.fromCache(userId);
                } else {
                    throw new RuntimeException("Unhandled error", ex);
                }
            });
    }
    
    // Chain recovery
    public CompletableFuture<Data> fetchWithMultipleFallbacks() {
        return primaryDataSource.fetch()
            .exceptionally(ex -> {
                logger.warn("Primary failed, trying secondary", ex);
                return secondaryDataSource.fetch().join();
            })
            .exceptionally(ex -> {
                logger.warn("Secondary failed, using cache", ex);
                return cache.get();
            })
            .exceptionally(ex -> {
                logger.error("All sources failed", ex);
                return Data.empty();
            });
    }
}
```

### 2. **handle() - Transform Both Success and Error**

```java
public class HandleExamples {
    
    // Transform success and error uniformly
    public CompletableFuture<Result<User>> getUser(String userId) {
        return userService.getUser(userId)
            .handle((user, ex) -> {
                if (ex != null) {
                    logger.error("Failed to get user", ex);
                    return Result.error(ex.getMessage());
                } else {
                    return Result.success(user);
                }
            });
    }
    
    // Convert exceptions to domain objects
    public CompletableFuture<PaymentResponse> processPayment(PaymentRequest request) {
        return paymentGateway.charge(request)
            .handle((result, ex) -> {
                if (ex != null) {
                    return PaymentResponse.failed(
                        extractErrorCode(ex),
                        ex.getMessage()
                    );
                } else {
                    return PaymentResponse.success(result);
                }
            });
    }
    
    // Logging wrapper
    public <T> CompletableFuture<T> withLogging(
            CompletableFuture<T> future, 
            String operation) {
        return future.handle((result, ex) -> {
            if (ex != null) {
                logger.error("Operation {} failed", operation, ex);
                throw new CompletionException(ex);
            } else {
                logger.info("Operation {} succeeded", operation);
                return result;
            }
        });
    }
}
```

### 3. **whenComplete() - Side Effect on Completion**

```java
public class WhenCompleteExamples {
    
    // Log success and failure without changing result
    public CompletableFuture<Order> processOrder(OrderRequest request) {
        return orderService.createOrder(request)
            .whenComplete((order, ex) -> {
                if (ex != null) {
                    metricsService.recordFailure("order_creation");
                    alertService.sendAlert("Order creation failed", ex);
                } else {
                    metricsService.recordSuccess("order_creation");
                    auditLog.log("Order created: " + order.getId());
                }
            });
        // Result is unchanged - exception is propagated
    }
    
    // Resource cleanup
    public CompletableFuture<Data> processWithCleanup() {
        Resource resource = acquireResource();
        
        return processData(resource)
            .whenComplete((result, ex) -> {
                resource.close();  // Always cleanup, regardless of success/failure
                if (ex != null) {
                    logger.error("Processing failed", ex);
                }
            });
    }
    
    // Metrics and monitoring
    public CompletableFuture<Response> timedOperation() {
        long startTime = System.currentTimeMillis();
        
        return performOperation()
            .whenComplete((result, ex) -> {
                long duration = System.currentTimeMillis() - startTime;
                metricsService.recordLatency("operation", duration);
                
                if (ex != null) {
                    metricsService.incrementCounter("operation.failures");
                } else {
                    metricsService.incrementCounter("operation.successes");
                }
            });
    }
}
```

### 4. **Retrying Failed Operations**

```java
public class RetryPatterns {
    
    // Simple retry with fixed attempts
    public CompletableFuture<String> retryOperation(Supplier<CompletableFuture<String>> operation, int maxRetries) {
        return operation.get()
            .exceptionallyCompose(ex -> {
                if (maxRetries > 0) {
                    logger.warn("Operation failed, retrying. Attempts left: {}", maxRetries);
                    return retryOperation(operation, maxRetries - 1);
                } else {
                    logger.error("Operation failed after all retries", ex);
                    return CompletableFuture.failedFuture(ex);
                }
            });
    }
    
    // Retry with exponential backoff
    public CompletableFuture<String> retryWithBackoff(
            Supplier<CompletableFuture<String>> operation,
            int maxRetries,
            long initialDelayMs) {
        
        return retryWithBackoffInternal(operation, maxRetries, initialDelayMs, 1);
    }
    
    private CompletableFuture<String> retryWithBackoffInternal(
            Supplier<CompletableFuture<String>> operation,
            int retriesLeft,
            long delayMs,
            int attempt) {
        
        return operation.get()
            .exceptionallyCompose(ex -> {
                if (retriesLeft > 0 && isRetryable(ex)) {
                    logger.warn("Attempt {} failed, retrying after {}ms", attempt, delayMs);
                    
                    return delay(delayMs)
                        .thenCompose(v -> retryWithBackoffInternal(
                            operation,
                            retriesLeft - 1,
                            delayMs * 2,  // Exponential backoff
                            attempt + 1
                        ));
                } else {
                    return CompletableFuture.failedFuture(ex);
                }
            });
    }
    
    private CompletableFuture<Void> delay(long millis) {
        CompletableFuture<Void> future = new CompletableFuture<>();
        scheduler.schedule(() -> future.complete(null), millis, TimeUnit.MILLISECONDS);
        return future;
    }
    
    private boolean isRetryable(Throwable ex) {
        return ex instanceof TimeoutException 
            || ex instanceof IOException
            || (ex instanceof ServiceException && ((ServiceException) ex).isRetryable());
    }
    
    // Retry with circuit breaker
    public CompletableFuture<String> retryWithCircuitBreaker(
            Supplier<CompletableFuture<String>> operation) {
        
        if (circuitBreaker.isOpen()) {
            return CompletableFuture.failedFuture(
                new CircuitBreakerOpenException("Circuit breaker is open")
            );
        }
        
        return operation.get()
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    circuitBreaker.recordFailure();
                } else {
                    circuitBreaker.recordSuccess();
                }
            })
            .exceptionallyCompose(ex -> {
                if (circuitBreaker.shouldRetry() && isRetryable(ex)) {
                    return delay(100).thenCompose(v -> retryWithCircuitBreaker(operation));
                }
                return CompletableFuture.failedFuture(ex);
            });
    }
}
```

---

## Async vs Non-Async Methods

### Understanding the Difference

```java
public class AsyncVsNonAsync {
    
    /**
     * Non-async methods run on the CALLING thread
     * Async methods run on the ForkJoinPool.commonPool() or custom executor
     */
    
    // thenApply - runs on same thread as previous stage
    public void demonstrateThenApply() {
        CompletableFuture.supplyAsync(() -> {
            System.out.println("Supply: " + Thread.currentThread().getName());
            return "result";
        })
        .thenApply(result -> {
            System.out.println("ThenApply: " + Thread.currentThread().getName());
            // Runs on SAME thread as supplyAsync (ForkJoinPool thread)
            return result.toUpperCase();
        });
    }
    
    // thenApplyAsync - explicitly runs on different thread
    public void demonstrateThenApplyAsync() {
        CompletableFuture.supplyAsync(() -> {
            System.out.println("Supply: " + Thread.currentThread().getName());
            return "result";
        })
        .thenApplyAsync(result -> {
            System.out.println("ThenApplyAsync: " + Thread.currentThread().getName());
            // Runs on DIFFERENT ForkJoinPool thread
            return result.toUpperCase();
        });
    }
    
    /**
     * When to use Async variants:
     * ✅ CPU-intensive transformations
     * ✅ Blocking operations (I/O, database)
     * ✅ When you want to control thread pool
     * 
     * When to use Non-async variants:
     * ✅ Simple, fast transformations
     * ✅ When you don't want thread context switches
     * ✅ For better performance in simple cases
     */
}
```

### Best Practices

```java
public class AsyncBestPractices {
    
    private final ExecutorService cpuBoundPool = 
        Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
    
    private final ExecutorService ioBoundPool = 
        Executors.newFixedThreadPool(100);  // Higher for I/O
    
    // Use non-async for simple transformations
    public CompletableFuture<String> simpleTransformation() {
        return fetchData()
            .thenApply(String::trim)           // Fast, non-blocking
            .thenApply(String::toLowerCase);   // Fast, non-blocking
    }
    
    // Use async for heavy computations
    public CompletableFuture<Image> processImage(String imageUrl) {
        return downloadImage(imageUrl)                    // I/O operation
            .thenApplyAsync(this::resizeImage, cpuBoundPool)      // CPU-intensive
            .thenApplyAsync(this::applyFilters, cpuBoundPool)     // CPU-intensive
            .thenApplyAsync(this::compress, cpuBoundPool);        // CPU-intensive
    }
    
    // Use async for blocking operations
    public CompletableFuture<User> saveUser(User user) {
        return CompletableFuture.supplyAsync(() -> {
            return userRepository.save(user);  // Blocking database call
        }, ioBoundPool);  // Use I/O pool, not ForkJoinPool
    }
    
    // Mix async and non-async appropriately
    public CompletableFuture<OrderResult> processOrder(OrderRequest request) {
        return validateOrder(request)                     // Fast validation
            .thenCompose(validOrder -> 
                // Blocking database call - use async
                CompletableFuture.supplyAsync(() -> 
                    orderRepository.save(validOrder), ioBoundPool
                )
            )
            .thenApply(this::generateOrderNumber)         // Fast generation
            .thenComposeAsync(order ->                    // Payment is I/O
                paymentService.processPayment(order), ioBoundPool
            )
            .thenApply(OrderResult::success);             // Fast wrapper
    }
}
```

---

## Thread Pool Management

### 1. **Custom Executors**

```java
public class ThreadPoolManagement {
    
    // CPU-bound tasks: Number of cores
    private final ExecutorService cpuBoundExecutor = Executors.newFixedThreadPool(
        Runtime.getRuntime().availableProcessors(),
        new ThreadFactoryBuilder()
            .setNameFormat("cpu-bound-%d")
            .setDaemon(false)
            .build()
    );
    
    // I/O-bound tasks: Higher thread count
    private final ExecutorService ioBoundExecutor = Executors.newFixedThreadPool(
        100,  // Can be much higher for I/O
        new ThreadFactoryBuilder()
            .setNameFormat("io-bound-%d")
            .setDaemon(false)
            .build()
    );
    
    // Virtual thread executor (Java 21+)
    private final ExecutorService virtualExecutor = 
        Executors.newVirtualThreadPerTaskExecutor();
    
    // Proper shutdown
    @PreDestroy
    public void shutdown() {
        shutdownExecutor(cpuBoundExecutor, "CPU-bound");
        shutdownExecutor(ioBoundExecutor, "I/O-bound");
    }
    
    private void shutdownExecutor(ExecutorService executor, String name) {
        logger.info("Shutting down {} executor", name);
        executor.shutdown();
        try {
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                executor.shutdownNow();
                if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                    logger.error("{} executor did not terminate", name);
                }
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

### 2. **ForkJoinPool Configuration**

```java
public class ForkJoinPoolConfig {
    
    // Default ForkJoinPool (used by *Async methods without executor)
    // Configured via system property:
    // -Djava.util.concurrent.ForkJoinPool.common.parallelism=20
    
    // Custom ForkJoinPool
    private final ForkJoinPool customForkJoinPool = new ForkJoinPool(
        20,  // Parallelism level
        ForkJoinPool.defaultForkJoinWorkerThreadFactory,
        (thread, throwable) -> logger.error("Uncaught exception", throwable),
        false  // Async mode
    );
    
    // Use custom ForkJoinPool
    public <T> CompletableFuture<T> runInCustomPool(Supplier<T> task) {
        return CompletableFuture.supplyAsync(task, customForkJoinPool);
    }
    
    // WARNING: Don't block ForkJoinPool threads!
    public CompletableFuture<String> badExample() {
        return CompletableFuture.supplyAsync(() -> {
            // ❌ BAD - Blocking ForkJoinPool thread
            return blockingDatabaseCall();
        });  // Uses common pool
    }
    
    public CompletableFuture<String> goodExample() {
        return CompletableFuture.supplyAsync(() -> {
            // ✅ GOOD - Using dedicated I/O pool
            return blockingDatabaseCall();
        }, ioBoundExecutor);
    }
}
```

---

## Real-World Patterns

### 1. **API Gateway Pattern**

```java
@Service
public class ApiGatewayService {
    
    private final ExecutorService executor = Executors.newFixedThreadPool(50);
    
    /**
     * Aggregate data from multiple microservices
     */
    public CompletableFuture<ProductDetails> getProductDetails(String productId) {
        // Start all requests in parallel
        CompletableFuture<Product> productFuture = 
            productService.getProduct(productId);
        
        CompletableFuture<List<Review>> reviewsFuture = 
            reviewService.getReviews(productId);
        
        CompletableFuture<Inventory> inventoryFuture = 
            inventoryService.getInventory(productId);
        
        CompletableFuture<List<Product>> relatedFuture = 
            recommendationService.getRelatedProducts(productId);
        
        CompletableFuture<PriceInfo> priceFuture = 
            pricingService.getPricing(productId);
        
        // Combine all results
        return CompletableFuture.allOf(
                productFuture, reviewsFuture, inventoryFuture, 
                relatedFuture, priceFuture
            )
            .thenApply(v -> new ProductDetails(
                productFuture.join(),
                reviewsFuture.join(),
                inventoryFuture.join(),
                relatedFuture.join(),
                priceFuture.join()
            ))
            .exceptionally(ex -> {
                logger.error("Failed to load product details", ex);
                // Return partial data or default
                return ProductDetails.builder()
                    .product(productFuture.getNow(null))
                    .reviews(reviewsFuture.getNow(Collections.emptyList()))
                    .build();
            });
    }
}
```

### 2. **Bulk Operations Pattern**

```java
@Service
public class BulkOperationService {
    
    private final ExecutorService executor = Executors.newFixedThreadPool(20);
    
    /**
     * Process items in batches asynchronously
     */
    public CompletableFuture<BulkResult> processBulkOrders(List<OrderRequest> orders) {
        int batchSize = 100;
        
        // Split into batches
        List<List<OrderRequest>> batches = partition(orders, batchSize);
        
        // Process each batch asynchronously
        List<CompletableFuture<List<OrderResult>>> batchFutures = batches.stream()
            .map(batch -> CompletableFuture.supplyAsync(
                () -> processBatch(batch), 
                executor
            ))
            .collect(Collectors.toList());
        
        // Wait for all batches
        return CompletableFuture.allOf(batchFutures.toArray(new CompletableFuture[0]))
            .thenApply(v -> {
                List<OrderResult> allResults = batchFutures.stream()
                    .map(CompletableFuture::join)
                    .flatMap(List::stream)
                    .collect(Collectors.toList());
                
                return new BulkResult(allResults);
            });
    }
    
    private List<OrderResult> processBatch(List<OrderRequest> batch) {
        return batch.stream()
            .map(this::processOrder)
            .collect(Collectors.toList());
    }
    
    private <T> List<List<T>> partition(List<T> list, int size) {
        return IntStream.range(0, (list.size() + size - 1) / size)
            .mapToObj(i -> list.subList(i * size, Math.min((i + 1) * size, list.size())))
            .collect(Collectors.toList());
    }
}
```

### 3. **Timeout Pattern**

```java
@Service
public class TimeoutService {
    
    private final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(1);
    
    /**
     * Add timeout to any CompletableFuture
     */
    public <T> CompletableFuture<T> withTimeout(
            CompletableFuture<T> future, 
            long timeout, 
            TimeUnit unit) {
        
        CompletableFuture<T> timeoutFuture = new CompletableFuture<>();
        
        // Schedule timeout
        ScheduledFuture<?> timeoutTask = scheduler.schedule(() -> {
            timeoutFuture.completeExceptionally(
                new TimeoutException("Operation timed out after " + timeout + " " + unit)
            );
        }, timeout, unit);
        
        // Complete with original future or timeout
        future.whenComplete((result, ex) -> {
            timeoutTask.cancel(false);  // Cancel timeout if completed
            if (ex != null) {
                timeoutFuture.completeExceptionally(ex);
            } else {
                timeoutFuture.complete(result);
            }
        });
        
        return timeoutFuture;
    }
    
    // Java 9+ has built-in timeout
    public <T> CompletableFuture<T> withTimeoutJava9(
            CompletableFuture<T> future,
            long timeout,
            TimeUnit unit) {
        
        return future.orTimeout(timeout, unit);  // Java 9+
    }
    
    // Timeout with fallback
    public <T> CompletableFuture<T> withTimeoutAndFallback(
            CompletableFuture<T> future,
            long timeout,
            TimeUnit unit,
            T fallbackValue) {
        
        return future
            .orTimeout(timeout, unit)
            .exceptionally(ex -> {
                if (ex instanceof TimeoutException) {
                    logger.warn("Operation timed out, using fallback");
                    return fallbackValue;
                }
                throw new CompletionException(ex);
            });
    }
}
```

### 4. **Cache Pattern**

```java
@Service
public class CacheService {
    
    private final Map<String, CompletableFuture<User>> cache = new ConcurrentHashMap<>();
    private final ExecutorService executor = Executors.newFixedThreadPool(10);
    
    /**
     * Prevent thundering herd - multiple requests for same key
     */
    public CompletableFuture<User> getUser(String userId) {
        return cache.computeIfAbsent(userId, id -> 
            CompletableFuture.supplyAsync(() -> {
                logger.info("Cache miss for user: {}", id);
                return userRepository.findById(id);
            }, executor)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    // Remove from cache on failure
                    cache.remove(userId);
                } else {
                    // Schedule cache eviction
                    scheduler.schedule(() -> cache.remove(userId), 5, TimeUnit.MINUTES);
                }
            })
        );
    }
    
    /**
     * Refresh cache proactively
     */
    public CompletableFuture<User> getUserWithRefresh(String userId) {
        CompletableFuture<User> cached = cache.get(userId);
        
        if (cached != null && !cached.isCompletedExceptionally()) {
            // Return cached value
            return cached;
        }
        
        // Fetch fresh data
        CompletableFuture<User> fresh = CompletableFuture.supplyAsync(
            () -> userRepository.findById(userId), 
            executor
        );
        
        cache.put(userId, fresh);
        return fresh;
    }
}
```

---

## Performance Optimization

### 1. **Avoid Blocking Operations**

```java
public class PerformanceOptimization {
    
    // ❌ BAD - Blocking in CompletableFuture
    public CompletableFuture<List<User>> badExample(List<String> userIds) {
        return CompletableFuture.supplyAsync(() -> {
            return userIds.stream()
                .map(id -> userRepository.findById(id).join())  // ❌ Blocking!
                .collect(Collectors.toList());
        });
    }
    
    // ✅ GOOD - Fully async
    public CompletableFuture<List<User>> goodExample(List<String> userIds) {
        List<CompletableFuture<User>> futures = userIds.stream()
            .map(userRepository::findById)  // Returns CF<User>
            .collect(Collectors.toList());
        
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)  // Safe - all completed
                .collect(Collectors.toList())
            );
    }
}
```

### 2. **Optimize Thread Pool Size**

```java
public class ThreadPoolOptimization {
    
    /**
     * CPU-bound: threads = cores
     * I/O-bound: threads = cores * (1 + wait_time / service_time)
     * 
     * Example: If operation spends 90% waiting for I/O:
     * threads = cores * (1 + 9) = cores * 10
     */
    
    private int calculateOptimalPoolSize() {
        int cores = Runtime.getRuntime().availableProcessors();
        double blockingCoefficient = 0.9;  // 90% I/O wait
        return (int) (cores * (1 + blockingCoefficient / (1 - blockingCoefficient)));
    }
    
    private final ExecutorService optimizedExecutor = 
        Executors.newFixedThreadPool(calculateOptimalPoolSize());
}
```

---

## Common Pitfalls

### 1. **Memory Leaks**

```java
public class MemoryLeakPitfalls {
    
    // ❌ BAD - Futures never complete, holding references
    private final Map<String, CompletableFuture<User>> cache = new ConcurrentHashMap<>();
    
    public CompletableFuture<User> badCaching(String userId) {
        return cache.computeIfAbsent(userId, id -> 
            userService.getUser(id)  // If this fails, future stays in cache forever
        );
    }
    
    // ✅ GOOD - Clean up on completion
    public CompletableFuture<User> goodCaching(String userId) {
        return cache.computeIfAbsent(userId, id -> 
            userService.getUser(id)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        cache.remove(userId);  // Remove failed futures
                    } else {
                        // Schedule eviction
                        scheduler.schedule(() -> cache.remove(userId), 5, TimeUnit.MINUTES);
                    }
                })
        );
    }
}
```

### 2. **Exception Swallowing**

```java
public class ExceptionSwallowing {
    
    // ❌ BAD - Exception lost
    public void badExample() {
        CompletableFuture.runAsync(() -> {
            throw new RuntimeException("Error!");
        });
        // Exception is swallowed - no one handles it
    }
    
    // ✅ GOOD - Handle exceptions
    public void goodExample() {
        CompletableFuture.runAsync(() -> {
            throw new RuntimeException("Error!");
        }).exceptionally(ex -> {
            logger.error("Async operation failed", ex);
            return null;
        });
    }
}
```

---

## Testing CompletableFuture

### 1. **Testing Async Code**

```java
@ExtendWith(MockitoExtension.class)
class AsyncServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void testAsyncOperation() {
        // Arrange
        User expectedUser = new User("1", "John");
        when(userRepository.findById("1"))
            .thenReturn(CompletableFuture.completedFuture(expectedUser));
        
        // Act
        CompletableFuture<User> future = userService.getUser("1");
        
        // Assert - Use join() in tests (it's ok to block in tests)
        User actualUser = future.join();
        assertEquals(expectedUser, actualUser);
    }
    
    @Test
    void testErrorHandling() {
        when(userRepository.findById("1"))
            .thenReturn(CompletableFuture.failedFuture(
                new NotFoundException("User not found")
            ));
        
        CompletableFuture<User> future = userService.getUser("1");
        
        CompletionException exception = assertThrows(CompletionException.class, 
            () -> future.join()
        );
        
        assertTrue(exception.getCause() instanceof NotFoundException);
    }
}
```

---

## Quick Reference

### Method Comparison

| Method | Returns | Use Case |
|--------|---------|----------|
| `thenApply(fn)` | `CF<U>` | Transform result (same thread) |
| `thenApplyAsync(fn)` | `CF<U>` | Transform result (async) |
| `thenCompose(fn)` | `CF<U>` | Flatten nested futures |
| `thenCombine(other, fn)` | `CF<V>` | Combine two futures |
| `thenAccept(consumer)` | `CF<Void>` | Consume result |
| `exceptionally(fn)` | `CF<T>` | Handle error, recover |
| `handle(fn)` | `CF<U>` | Handle success & error |
| `whenComplete(consumer)` | `CF<T>` | Side effect on completion |
| `allOf(futures...)` | `CF<Void>` | Wait for all |
| `anyOf(futures...)` | `CF<Object>` | Wait for first |

### Best Practices Checklist

```
✅ Use custom executor for I/O operations
✅ Never block CompletableFuture threads
✅ Always handle exceptions
✅ Use thenCompose for dependent async operations
✅ Use allOf for parallel independent operations
✅ Avoid get() - use join() or callbacks
✅ Clean up resources in whenComplete
✅ Use orTimeout for network calls (Java 9+)
✅ Test async code properly
✅ Monitor thread pool metrics

❌ Don't use ForkJoinPool for blocking operations
❌ Don't swallow exceptions
❌ Don't create nested futures (use thenCompose)
❌ Don't block in transformation functions
❌ Don't forget to shutdown custom executors
```

---

## Conclusion

CompletableFuture is powerful but requires discipline:

1. **Think async-first** - Avoid blocking
2. **Handle errors properly** - Use exceptionally/handle
3. **Use right thread pool** - I/O vs CPU-bound
4. **Compose, don't nest** - Use thenCompose
5. **Test thoroughly** - Async bugs are sneaky

With these patterns and practices, you can build robust, high-performance asynchronous Java applications!
