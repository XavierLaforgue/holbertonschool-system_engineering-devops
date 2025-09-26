# Distributed web infrastructure

![Distributed Web Infrastructure Diagram](./1-distributed_web_infrastructure_v1.svg)

## User story

1. A user wants to access `www.foobar.com` via their browser.
2. The browser verifies if the domain is known by the local device (in cache or otherwise). 
3. If not found, the browser sends a DNS query to resolve `www.foobar.com` to an IP address. 
4. The DNS responds with the A record (8.8.8.8), the IP address corresponding to the requested domain.
5. The browser connects to the load-balancer (HAproxy) at IP 8.8.8.8 on port 80 (HTTP) or 443 (HTTPS) depending on the protocol.
6. HAproxy evaluates the request and forwards it to one of the web servers using the configured load distribution algorithm.
7. The web server (Nginx) receives the request and serves static files and/or forwards the request to the application server (process running at a certain port) to produce dynamic content.
8. The application server uses the application files to satisfy the application's logic.
9. If necessary, the application server may read/write persistent data from/to the database cluster (MySQL), where writes go to the primary database and reads to the replicas.
	- If a write operation is performed it is recorded in a log that is accessed, read, and replicated by the replicas.  
10. The application server sends back the necessary data to the web server (Nginx).
11. The web server returns the response to the load-balancer and this returns a response to the user's browser (e.g., in the form of an html document).

## Architecture (three-server + HAProxy infrastructure)

High-level layout of a three-server (physical or virtual) architecture using a load-balancer with public IP address 8.8.8.8:

- **DNS**
  - Domain: foobar.com
  - DNS A record: `www.foobar.com`, resolving to 8.8.8.8 (the public IP of the load-balancer)

- **Load-balancer (HAProxy, IPv4 8.8.8.8)**
  Distributes requests among the servers using traffic distribution algorithm (e.g., round-robin)

- **Servers (3 instances)**
	Each server is reachable only from the load-balancer and contains a copy of:
	- Web server (Nginx)
	- Application server (e.g., Python)
	- Application files (codebase: static assets, backend code)
	- Database (MySQL) cluster member (one primary, two replicas)

## Specifics about the infrastructure

### Why each additional element is added

- **Additional servers:**
	- For redundancy: even if one server crashes, the site remains available via the other servers
	- To facilitate scalability: requests/tasks can be dsitributed among servers and thus higher traffic may be handled
	- For maintenance: servers can be down in turns to perform maintenance on them, thus always keeping the site up served by the other servers

- **Load balancer (HAproxy):**
	- To hide internal infrastructure topology
	- To continue providing a single public endpoint while adding servers and thus higher capacity
	- To distribute requests to working servers and improve reliability (thus taking advantage of the additional servers)

- **Database Replicas:**
	- To distribute the load of read operations onto the replicas and thus improve availability and reactivity of the primary
	- To provide fail-safes for database failures

### Load balancer distribution algorithm

There are several distribution algorithm we could use, the most common one is: Round Robin.
Using Round Robin: HAproxy keeps track of the server it has last sent a request to and forwards the next connection to the next server in the list of available ones. 
This distributes the number of connections equally among the servers, leading to even loads over time if the servers are identical and the requests all require about the same computational ressources.

Noteworthy alternatives are:
- Least Connections algorithm: it tracks the number of active connections and forwards new requests to the server with the fewest.
- Cookie-based sticky sessions algorithm: using cookies, the load-balancer can guarantee that each subsequent request from a client will be forwarded to the same server.

### Active-Active vs Active-Passive load-balancer setups

The current architecture is not enabling either setup since it only contains a single load-balancer.

| Feature / Setup             | Active-Active | Active-Passive |
|-----------------------------|---------------|----------------|
| Nº of active load-balancers | Multiple      | One            |
| Traffic distribution        | Shared among all load-balancers | Only active load-balancer serves traffic (others remain on standby) |
| Failover                    | Healthy load-balancer(s) continue handling traffic, load is distributed among them | Passive load-balancer takes charge on failure (of the previously active load-balancer) |
| Complexity                  | Higher (synchronization of information among load-balancers) | Lower (simpler to setup) |
| Resource usage              | Efficient (all load-balancers' resources are available and ready to be used) | Less resource-efficient (standby load-balancers = idle = wasted resources) |
| Availability                | Higher (multiple load-abalancers guarantee requests) | Good (the active load-balancer needs to fail before one in standby can become available) |
| Implementation              | DNS round-robin (DNS returns multiple IPs for the same domain, each referring to a different load-balancer) or VIP (multiple load-balancers, same IP, using VRRP/keepalived) | VRRP/keepalived to assign the VIP to a standby server after failure |
| Session Handling            | Sticky sessions (in-memory: cookies) or shared session storage (redis) | Perhaps also sticky sessions |
------------------------------------------------
### How a database (MySQL) Primary-Replica (Master-Slave) cluster works

The purpose of the cluster is to increase availability of the database and reduce wait time of database queries as they may be better distributed among the members of the cluster.

The primary database instance, or master, is the sole responsible for (and allowed to perform) user requested write operations (insertions, modificatuons, deletions).
Once write operations are performed, the primary database records it in a binary log (binlog) that is shared with the replicas.

The database replicas, or slaves, access the binlog in the primary database and reproduce the operations locally to synchronize their states.
There are a few ways replication can be setup, the main ones are: 
- Asynchronously: primary performs operation and returns success, replicas perform it later, replication lag is possible.
- Semi-synchronously: primary performs operation and awaits confirmation of success from at least one replica before returning success, increased database consistency but higher latency.
- Synchronously: primary performs operation and awaits confirmation from all replicas before returning success, maximum consistency but highest latency.

In case of failure on the part of the primary, a replica may be promoted to primary.
If a replica fails the site remains operational via the other cluster members.

### Primary node vs Replica node in regards to the application

User-requested write operations are always sent to the primary node.
User-requested read operations are distributed among the replicas as to increase availability of the primary.

|Feature / Node type | Primary         | Replicas                     |
|--------------------|-----------------|------------------------------|
| Operations         | reads/writes    | reads only                   |
| Usage              | write operations and necessarily up-to-date reads | relieve read load off the primary |
| Purpose            | source of truth | scale read queries           |
| Consistency        | up to date      | may lag behind the primary   |
----------------------------------------------------------------------

The application must, therefore, route queries approppriately to the primary or replica nodes:
- All writes to the primary.
- Reads distributed among primary and replica depending on tolerance to possible consistency lag.

## Issues with this infrastructure

- **Single Points Of Failure (SPOF):**
  The main single point of failure is the reliance on a single load-balancer through which all traffic is directed (if the load-balancer crashes the site is unreachable).
  Another one could be the primary node of the database cluster (if the promotion of replicas to primary is not properly automated, a crashed primary will cause the site to fail all writes).

- **Security issues:**
  The absence of firewalls may leave ports exposed to the public, which may risk the privacy and integrity of database.
  As a secured communication method (HTTPS/TLS) has not been enforced, traffic is not encrypted and may be read (spied on) by outsiders with possible malicious intent.

- **Monitoring:**
  The absence of monitoring tools means that crashes (downtime), performance issues, or security violations may occur unnoticed.
