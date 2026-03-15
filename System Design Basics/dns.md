# Domain Name System(DNS)
A DNS traslates domain names into IP address
Hierarchical
Cache is applied on different levels
Cache duration is computed by the time to live (TTL)

## Records
* NS record (name server) - Specifies the DNS servers for your domain/subdomain.
* MX record (mail exchange) - Specifies the mail servers for accepting messages.
* A record (address) - Points a name to an IP address.
* CNAME (canonical) - Points a name to another name or CNAME (example.com to www.example.com) or to an A record.

### Routing logics
**Weighted Round Robin**
* Prevents traffic from goin to servers under maintenance
* Balance between different cluster sizes
* A/B testing
**Latency based**
* Send user to the region / server with lowest latency
**Geolocation Based**
* Send user to the closes region to the user

### Disadvantage(s): DNS
* Accessing a DNS server introduces a slight delay, although mitigated by caching described above.
* DNS server management could be complex and is generally managed by governments, ISPs, and large companies.
* DNS services have recently come under DDoS attack, preventing users from accessing websites such as Twitter without knowing Twitter's IP address(es).
