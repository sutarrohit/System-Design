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
