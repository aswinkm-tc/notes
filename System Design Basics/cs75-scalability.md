# Vertical Scaling (Scale Up)
Increasing the resources on a node or multiple nodes
There are real world concerns with this on the cap of machine type you can procure
Multi-threaded or parallel processing restrictions
Per machine Disc IOPS limitations
Per machine Bandwidth limitations
# Horizontal Scaling (Scaling Out)
Increasing the number of nodes
Distributed systems are required
Loadbalancing is required

# Loadbalancing
## DNS
Multiple A records pointing to the same domain
Does roundrobin by default





# Clones
Every server contains same codebase
None of the servers hosts stateful information like user data
Sessions needs to be stored centralized datastore (External DB or Persistent cache)

# Database Scaling
Sharding: Splitting data into addressible chunks using a unique key and hashing algorithms 
Denormalization: Remove unneccessary joins

# Caching
Simple key-value store
Between application and data storage
if Read from cache:
    hit: return result
    miss:   search datastore
            fill cache

## Cache database queries
key: Hashed version of query
value: result of query
### Cons
Hard to expire or delete cache

## Cache objects
key: object (example user)
Value: object properties (username, password, balance etc)

### Pros
Easy to expire and delete cache on changes

Some ideas of objects to cache:
    user sessions (never use the database!)
    fully rendered blog articles
    activity streams
    user<->friend relationships 

# Asynchronization
## Proactive asynchronization
Heavy tasks like rendering a website is done is advance and stored
When asked for the website the rendered pages are returned

# Reactive asynchronization
Using job queues and consumers
A response is returned to the user that the job has been added to the queue so the user is not blocked
The fronted can poll for the status of the job via a different endpoint
Or server can notify the frontend via websockets or using server side messages
