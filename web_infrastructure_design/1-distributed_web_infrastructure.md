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

| Feature / Setup | Active-Active | Active-Passive |
|-----------------|---------------|----------------|
| Nº of active load-balancers | Multiple | One |
| Traffic distribution | Shared among all load-balancers | Only active load-balancer serves traffic (others remain on standby) |
| Failover | Healthy load-balancer(s) continue handling traffic, load is distributed among them | Passive load-balancer takes charge on failure (of the previously active load-balancer) |
| Complexity | Higher (synchronization of information among load-balancers) | Lower (simpler to setup) |
| Resource Utilization | Efficient (all load-balancers' resources are available and ready to be used) | Less resource-efficient (standby load-balancers = idle = wasted resources) |
| Availability | Higher (multiple load-abalancers guarantee requests) | Good (the active load-balancer needs to fail before one in standby can become available) |
| Implementation | DNS round-robin (DNS returns multiple IPs for the same domain, each referring to a different load-balancer) or VIP (multiple load-balancers, same IP, using VRRP/keepalived) | VRRP/keepalived to assign the VIP to a standby server after failure |
| Session Handling | Sticky sessions (in-memory: cookies) or shared session storage (redis) | Perhaps also sticky sessions |

### How Primary-Replica (Master-Slave) MySQL works

- Primary (Master) takes all write operations (INSERT/UPDATE/DELETE) and records changes in a binary log (binlog).
- Replicas (Slaves) connect to the Primary and read the binlog events, replaying them locally to reproduce the Primary's state.
- Replication modes:
  - Asynchronous: Replicas apply changes after the Primary commits; this can cause replication lag.
  - Semi-synchronous: Primary waits for acknowledgement from at least one Replica before reporting commit success — reduces chance of data loss but increases write latency.
- Failover: if the Primary fails, one Replica can be promoted to Primary. Promotion can be manual or automated via tools (Orchestrator, MHA, or custom scripts). After promotion, former Primary must be re-added as a Replica.

### Primary vs Replica from the application's perspective

- Primary:
  - Accepts writes and reads.
  - Provides the most up-to-date data.
  - Recommended for write operations and for reads that require the latest data.

- Replica:
  - Typically read-only (or treated as such by the application).
  - Serves read-only queries and reduces Primary load.
  - May lag behind the Primary (eventual consistency). Applications must tolerate stale reads if using replicas immediately after writes.

Application responsibilities:
- Route all write queries to the Primary.
- Optionally route read queries to Replicas (read-scaling) while handling possible replication lag (e.g., reading after writes should go to Primary or ensure replica has caught up).

## Issues with this infrastructure

- **Single Points of Failure (SPOF):**
  - Single HAProxy instance: if the load balancer fails, clients cannot reach the backend servers. Mitigation: add a second LB and configure Active-Active or Active-Passive topology.
  - Primary database: if the Primary fails and you lack automated failover, writes stop. Mitigation: use automated failover tools and keep regular backups.

- Security issues:
  - No firewall rules described: servers might expose management ports (SSH, DB) to the public. Mitigation: use security groups and host firewalls (ufw/iptables), restrict access to known admin IPs.
  - No HTTPS/TLS by default: traffic is unencrypted. Mitigation: terminate TLS on HAProxy (recommended) using certificates from a CA (Let's Encrypt) and automate renewal.
  - No rate-limiting or WAF: the stack is exposed to abuse and simple attacks. Mitigation: configure HAProxy limits and consider a WAF or CDN.

- Observability and monitoring:
  - No monitoring or alerting: you won't be notified of failures or performance problems. Mitigation: deploy Prometheus + node_exporter + MySQL exporter + Grafana + Alertmanager and log aggregation (ELK/EFK).

- Operational concerns:
  - Session state: if the app stores session state in-process, load balancing without sticky sessions will break sessions. Use Redis for sessions or enable sticky sessions.
  - Configuration drift: multiple backend servers must be kept in sync (use configuration management: Ansible, Chef, Puppet, or container images).
