# Application Layer
[Application Layer](https://github.com/donnemartin/system-design-primer/blob/master/images/yB5SYwm.png)
* Seperating application layer from web servers allows you to scale and configure both layers independently
* Adding a new API is additng another application server without adding more web servers

# Microservices
* Suite of independently deployable, small modular services
*  Each service runs a unique process and communicates through a well-defined, lightweight mechanism to serve a business goal.

# Service Discovery
* Systems such as consul, etcd and zookeeper can help services discover each other
* They keep track of registered names, addresses, and ports
* Healthchecks verify the service integrity and are often done via HTTP/Grpc endpoints
* Consul and Etcd have built-in kv stores which can be useful for storing config values and other shared data

