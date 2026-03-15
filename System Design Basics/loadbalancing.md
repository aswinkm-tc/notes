# Load balancer
Load balancers distribute incoming client requests to computing resources such as application servers and databases. In each case, the load balancer returns the response from the computing resource to the appropriate client. Load balancers are effective at:
* Preventing requests from going to unhealthy servers
* Preventing overloading resources
* Helping to eliminate a single point of failure

**SSL Termination**: Decrypt incoming requests and encrypt server responses
**Session Persistence**: Issue cookies and route client requests to same instances, this helps in removing need for centralised session stores

> Note: To protect against failures, it's common to set up multiple load balancers, either in active-passive or active-active mode.

## Types
* Random
* Least loaded
* Session/cookies
* Round robin / weighted round robin
* L4
* L7

