# Rate limiting

Rate limiting helps protect services from being overwhelmed by too many requests from a single user or client.

## Most common rate limiting algorithms

---

### 1. Token Bucket

![Token Bucket](https://substackcdn.com/image/fetch/w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2955cacb-e76f-4606-b257-84718268524d_1132x956.png)

The **Token Bucket** algorithm is one of the most popular and widely used rate limiting approaches due to its simplicity and effectiveness.

### How It Works:

-   Imagine a bucket that holds tokens.
-   The bucket has a maximum capacity of tokens.
-   Tokens are added to the bucket at a fixed rate (e.g., **10 tokens per second**).
-   When a request arrives, it must obtain a token from the bucket to proceed.
-   If there are enough tokens, the request is allowed and tokens are removed.
-   If there aren't enough tokens, the request is dropped.

```javascript
class TokenBucket {
    constructor(capacity, refillRatePerSecond) {
        this.capacity = capacity;
        this.tokens = capacity;
        this.refillRatePerSecond = refillRatePerSecond;
        this.lastRefillTimestamp = Date.now();
    }

    // Refill the tokens based on elapsed time
    refillTokens() {
        const now = Date.now();
        const timeElapsed = (now - this.lastRefillTimestamp) / 1000; // Convert ms to seconds
        const tokensToAdd = timeElapsed * this.refillRatePerSecond;

        // Add tokens, but do not exceed capacity
        this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
        this.lastRefillTimestamp = now;
    }

    // Try to consume tokens for a request
    allowRequest(tokensNeeded = 1) {
        this.refillTokens();

        if (this.tokens >= tokensNeeded) {
            this.tokens -= tokensNeeded;
            console.log(`Request Allowed! Tokens left: ${this.tokens.toFixed(2)}`);
            return true;
        }

        console.log("Request Denied! Not enough tokens.");
        return false;
    }
}

// Usage Example
const bucket = new TokenBucket(10, 2); // Capacity: 10 tokens, Refill Rate: 2 tokens/second

// Simulate requests
setInterval(() => {
    bucket.allowRequest(); // Request consumes 1 token
}, 500);

// You can simulate bursts
setTimeout(() => {
    for (let i = 0; i < 12; i++) {
        bucket.allowRequest();
    }
}, 3000);
```

### ✅ Pros:

-   **Relatively straightforward** to implement and understand.
-   Allows **bursts of requests** up to the bucket's capacity, accommodating short-term spikes.

### ❌ Cons:

-   The **memory usage scales** with the number of users if implemented per-user.
-   It doesn’t guarantee a **perfectly smooth rate** of requests.

---

---

### 2. Leaky Bucket

![Leaky Bucket](https://substackcdn.com/image/fetch/w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F0815d30e-dc9c-4ff4-9eb8-ac76d21ba52d_1048x684.png)

The **Leaky Bucket** algorithm is similar to the **Token Bucket** but focuses on **smoothing out bursty traffic**.

### How It Works:

-   Imagine a **bucket with a small hole** in the bottom.
-   **Requests enter the bucket** from the top.
-   The bucket **processes ("leaks") requests at a constant rate** through the hole.
-   If the **bucket is full**, new requests are **discarded**.

```javascript
class LeakyBucket {
    constructor(capacity, leakRatePerSecond) {
        this.capacity = capacity; // Maximum number of requests in the bucket
        this.leakRatePerSecond = leakRatePerSecond; // Leak rate (requests/second)
        this.bucket = []; // Array to hold request timestamps
        this.lastLeakTime = Date.now(); // Last time the bucket leaked
    }

    // Simulate leaking from the bucket
    leak() {
        const now = Date.now();
        const timeElapsed = (now - this.lastLeakTime) / 1000; // Convert ms to seconds
        const leakedRequests = Math.floor(timeElapsed * this.leakRatePerSecond);

        if (leakedRequests > 0) {
            // Remove the leaked requests from the bucket
            this.bucket.splice(0, leakedRequests);
            this.lastLeakTime = now;
        }
    }

    // Check if a new request can be allowed
    allowRequest() {
        this.leak();

        if (this.bucket.length < this.capacity) {
            this.bucket.push(Date.now());
            console.log("✅ Request Allowed");
            return true;
        }

        console.log("❌ Request Denied - Bucket Full");
        return false;
    }
}

// Usage example
const limiter = new LeakyBucket(5, 1); // Capacity: 5 requests, Leak Rate: 1 request/second

// Simulate 10 requests
let requestCount = 0;
const interval = setInterval(() => {
    limiter.allowRequest();
    requestCount++;

    if (requestCount === 10) clearInterval(interval);
}, 100);

// Wait for the bucket to leak
setTimeout(() => {
    console.log("After 1 second...");
    limiter.allowRequest();
}, 1000);
```

### ✅ Pros:

-   **Processes requests at a steady rate**, preventing sudden bursts from overwhelming the system.
-   Provides a **consistent and predictable rate** of processing requests.

### ❌ Cons:

-   **Does not handle sudden bursts** of requests well; excess requests are immediately dropped.
-   Slightly **more complex to implement** compared to the Token Bucket.

---

---

# 3. Fixed Window Counter

## ![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ff6683da4-8b67-4bbf-bf95-8289d640e1b3_1236x832.png)

The **Fixed Window Counter** algorithm divides time into fixed windows and counts requests in each window.

## How It Works

-   Time is divided into **fixed windows** (e.g., 1-minute intervals).
-   Each **window** has a **counter** that starts at **zero**.
-   New requests **increment the counter** for the current window.
-   If the **counter exceeds the limit**, requests are **denied** until the **next window**.

## **Code**

```javascript
class FixedWindowCounter {
    constructor(windowSize, maxRequests) {
        this.windowSize = windowSize * 1000; // Convert window size to milliseconds
        this.maxRequests = maxRequests; // Maximum number of requests per window
        this.currentWindow = Math.floor(Date.now() / this.windowSize);
        this.requestCount = 0;
    }

    allowRequest() {
        const currentTime = Date.now();
        const window = Math.floor(currentTime / this.windowSize);

        // If a new window has started, reset the counter
        if (window !== this.currentWindow) {
            this.currentWindow = window;
            this.requestCount = 0;
        }

        // Check if the request is within the limit
        if (this.requestCount < this.maxRequests) {
            this.requestCount++;
            return true;
        }
        return false;
    }
}

// Usage example
const limiter = new FixedWindowCounter(60, 5); // 5 requests per minute

for (let i = 0; i < 10; i++) {
    console.log(limiter.allowRequest()); // Will print true for the first 5 requests, then false
    await new Promise((resolve) => setTimeout(resolve, 100)); // Wait a bit between requests
}

setTimeout(() => {
    console.log(limiter.allowRequest()); // After 60 seconds, should be true again
}, 60000);
```

---

## **Pros**

-   ✅ **Easy to implement and understand**.
-   ✅ Provides **clear and easy-to-understand rate limits** for each time window.

---

## **Cons**

-   ❌ **Does not handle bursts of requests** at the boundary of windows well.
-   ❌ Can allow **twice the rate of requests** at the edges of windows.

---

---

# 4. Sliding Window Log

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fb4db0d4a-77b4-44b1-b5c9-50c02d4764ec_950x622.png)

The **Sliding Window Log** algorithm keeps a log of timestamps for each request and uses this log to determine whether a new request should be allowed.

## **How It Works**

-   ✅ **Keep a log of request timestamps**.
-   ✅ When a new request comes in, **remove all entries** older than the window size.
-   ✅ **Count the remaining entries**.
-   ✅ If the count is **less than the limit**, allow the request and **add its timestamp** to the log.
-   ✅ If the count **exceeds the limit**, the request is **denied**.

---

## **Code**

```javascript
class SlidingWindowLog {
    constructor(windowSize, maxRequests) {
        this.windowSize = windowSize * 1000; // Convert window size to milliseconds
        this.maxRequests = maxRequests;
        this.requestLog = []; // Array to keep track of request timestamps
    }

    allowRequest() {
        const now = Date.now();

        // Remove timestamps that are outside the current window
        while (this.requestLog.length > 0 && now - this.requestLog[0] >= this.windowSize) {
            this.requestLog.shift();
        }

        // Check if we're still within the limit
        if (this.requestLog.length < this.maxRequests) {
            this.requestLog.push(now);
            return true;
        }

        return false;
    }
}

// Usage example
const limiter = new SlidingWindowLog(60, 5); // 5 requests per minute

for (let i = 0; i < 10; i++) {
    console.log(limiter.allowRequest()); // Will print true for the first 5 requests, then false
    await new Promise((resolve) => setTimeout(resolve, 100)); // Wait a bit between requests
}

setTimeout(() => {
    console.log(limiter.allowRequest()); // After 60 seconds, should be true again
}, 60000);
```

---

## **Pros**

-   ✅ **Very accurate**, no rough edges between windows.
-   ✅ **Works well for low-volume APIs**.

---

## **Cons**

-   ❌ Can be **memory-intensive** for high-volume APIs.
-   ❌ Requires **storing and searching through timestamps**, which can slow down processing.

---

## **Problem Example**

The Sliding Window Log algorithm helps avoid **double requests** at the edge of time windows, ensuring more **precise rate limiting**.
However, the downside is that it consumes **more memory** since it has to **store timestamps**.

---

---

# 5. Sliding Window Counter

![](https://substackcdn.com/image/fetch/w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F80181659-e4fa-4398-bd12-bef659174e81_972x686.png)

The **Sliding Window Counter** algorithm combines the **Fixed Window Counter** and **Sliding Window Log** approaches to offer a **more accurate and memory-efficient solution** for rate limiting.

## **How It Works**

-   ✅ Instead of tracking **every single request's timestamp** like the **Sliding Window Log**, this algorithm **tracks the number of requests** from the **current window** and **previous window**.
-   ✅ The algorithm **calculates a weighted sum** of requests based on how much time has passed in the **current window** and the remaining time from the **previous window**.
-   ✅ If the weighted sum **plus the new request** crosses the **rate limit**, the request is **denied**.

---

## **Weight Calculation**

The key concept in this algorithm is the **weight of requests** between two windows.

Suppose you are **75%** into the **current window**, then:

-   **25% of the weight** will come from the **previous window**.
-   **75% of the weight** will come from the **current window**.

The formula looks like:

```plaintext
weight = (100 - 75)% * lastWindowRequests + currentWindowRequests
```

---

## **Code**

```javascript
class SlidingWindowCounter {
    constructor(windowSize, maxRequests) {
        this.windowSize = windowSize * 1000; // Convert window size to milliseconds
        this.maxRequests = maxRequests;
        this.currentWindow = Math.floor(Date.now() / this.windowSize);
        this.requestCount = 0;
        this.previousCount = 0;
    }

    allowRequest() {
        const now = Date.now();
        const window = Math.floor(now / this.windowSize);

        // If moved to a new window, shift the counts
        if (window !== this.currentWindow) {
            this.previousCount = this.requestCount;
            this.requestCount = 0;
            this.currentWindow = window;
        }

        // Calculate the weighted request count
        const windowElapsed = (now % this.windowSize) / this.windowSize;
        const threshold = this.previousCount * (1 - windowElapsed) + this.requestCount;

        // Check if the request is within the rate limit
        if (threshold < this.maxRequests) {
            this.requestCount++;
            return true;
        }

        return false;
    }
}

// Usage example
const limiter = new SlidingWindowCounter(60, 5); // 5 requests per minute

for (let i = 0; i < 10; i++) {
    console.log(limiter.allowRequest()); // Will print true for the first 5 requests, then false
    await new Promise((resolve) => setTimeout(resolve, 100)); // Wait a bit between requests
}

setTimeout(() => {
    console.log(limiter.allowRequest()); // Might be true or false depending on the exact timing
}, 30000); // After half window time
```

---

## **Pros and Cons of Sliding Window Counter**

### ✅ **Pros**

-   **More accurate** than the **Fixed Window Counter**.
-   **More memory-efficient** than the **Sliding Window Log**.
-   **Smooths out request spikes** by reducing the impact of window boundaries.

---

### ❌ **Cons**

-   **Slightly more complex to implement** compared to other algorithms.

---

---

-   When implementing rate limiting, consider factors such as the scale of your system, the nature of your traffic patterns, and the granularity of control you need.

-   Lastly, always communicate your rate limits clearly to your API users, preferably through response headers, so they can implement appropriate retry and backoff strategies in their clients.
