# URL Shortner
Goal: Create a bit.ly like URL shortner service.

## The Shortening Logic
* To convert a long URL, we use Base62 encoding ([0-9, a-z, A-Z]).
* A 7-character string gives us 62^7  approx 3.5 Trillion unique IDs.

## The API Layer
We need two main REST endpoints:
* POST /api/v1/shorten: Takes long_url, returns short_url.
* GET /{short_url}: Returns a 302 Redirect (Temporary) or 301 (Permanent).
> Pro-tip: Use 302 if you want to collect analytics on every click; use 301 if you want to reduce server load by letting the browser cache the redirect.

## The Distributed Solution: Range Handler (KGS)
* Instead of a single counter, we use a Key Generation Service (KGS) with a coordination layer like **Zookeeper**
* Zookeeper maintains ranges (e.g., Node A gets 1–1M, Node B gets 1M–2M).
* Application servers pull a "chunk" of IDs into memory.
* Even if an app server crashes and loses its remaining chunk, we have billions of IDs to spare; losing 1,000 is negligible.

## Storage & Caching
### Database Choice
NoSQL (e.g., Cassandra or DynamoDB): Ideal here. We need a simple Key-Value lookup, high write throughput, and easy horizontal scaling.
Schema:
``` 
{
    short_hash: Primary Key, 
    original_url: string,
    expiration_ts: timestamp
}
```

## Caching Strategy
Since 20% of the URLs will likely generate 80% of the traffic (Pareto Principle), we implement a Redis cache layer.
> The Pareto principle states that, for many outcomes, roughly 80% of consequences come from 20% of causes. In 1941, management consultant Joseph M. Juran developed the concept in the context of quality control and improvement after reading the works of Italian sociologist and economist Vilfredo Pareto, who wrote in 1906 about the 80:20 connection while teaching at the University of Lausanne.

Eviction Policy: LRU (Least Recently Used).
```mermaid
Flow: Check Cache → If Miss, Check DB → Update Cache → Redirect.
```

| Challenge | Strategy |
|---|---|
| Global Latency | Deploy Edge redirection servers in multiple AWS regions using GeoDNS. |
| Analytics | Don't update "click counts" synchronously in the main database. Stream events to **kafka** for asynchronous processing into OLAP DB(like ClickHouse) |
| Abuse/Security | Implement **Rate limiting** per API key/IP and integrate a maleware scanning service to the taget long_url |
| Data Cleanup | Use **TTL** features in Cassandra to automatically expire old links without manual "delete" scripts |

## KGS
A KGS (Key Generation Service) is a dedicated microservice that pre-generates unique IDs (keys) to ensure high performance and prevent collisions.

Instead of calculating a hash or checking a database for uniqueness every time a user submits a URL, the application simply "grabs" an existing key from the KGS.

### How the KGS Works
The primary goal of a KGS is to move the heavy lifting of "uniqueness" to an offline process.

Offline Generation: The KGS runs a background process that generates random, unique strings (e.g., 6-character strings like 5aZ8pQ) and stores them in a dedicated "Key DB."

Key Storage: These keys are stored in a simple table with two columns: Key and Status (e.g., Available or Used).

In-Memory Buffering: To avoid hitting the Key DB for every single request, the KGS keeps a small portion of "Available" keys in its local RAM.

Concurrency Management: The KGS uses a locking mechanism (or moves keys to a "Used" table) to ensure that the same key is never given to two different application servers.

## What the Application Server Should Do
When a user wants to shorten a URL, the application server acts as the orchestrator. Its responsibilities are streamlined because the KGS handles the ID logic:
1. Request the Key
The application server sends a request to the KGS for a fresh key. It doesn't need to worry about Base62 encoding or hashing algorithms at this stage; it just expects a string back.
2. Map and Store
Once it receives the key (e.g., 8jM2x), the application server creates a record in the main database that maps that key to the original long URL:
Key: 8jM2x
Original URL: https://www.example.com/very/long/path/to/content
User ID / Timestamp: For analytics and expiration.
3. Handle Caching
To make the redirection process lightning-fast for end-users, the application server should store these mappings in a cache (like Redis). When a user visits tiny.url/8jM2x, the server checks the cache first before querying the main database.
4. Direct the User
Finally, the application server returns a 301 (Permanent Redirect) or 302 (Temporary Redirect) response to the user's browser, pointing them to the original long URL.
