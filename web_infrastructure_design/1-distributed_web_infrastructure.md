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



Recommended default: Round Robin
- How it works: HAProxy forwards each new connection to the next backend in the list (A -> B -> C -> A ...). This evens out traffic over time assuming equal server capacity.

When to choose alternatives:
- Least Connections: choose this when requests are long-lived or uneven in cost — it forwards to the backend with the fewest active connections.
- Source (hash): consistent routing by client IP. Useful for simple affinity (if no shared session store is available).
- Cookie-based sticky sessions: HAProxy can inject a cookie so subsequent requests from the same client return to the same backend.

Notes:
- For truly stateless apps, avoid sticky sessions and use a shared session store (Redis) or token-based auth.
- Consider health checks (HTTP checks against /health) so HAProxy can detect unhealthy backends and stop routing to them.

## Active-Active vs Active-Passive Load Balancer setups

- Active-Active:
  - Multiple load balancer instances are active at the same time and share traffic.
  - Requires a frontend mechanism (DNS round-robin, anycast, or a virtual IP shared via keepalived) to distribute incoming traffic among LBs.
  - Benefits: higher capacity, no single LB becomes a bottleneck, better availability.
  - More complex to manage (state synchronization, session affinity considerations).

- Active-Passive:
  - One LB is active and serves all traffic. A second LB is passive and takes over only if the active one fails (failover via VRRP/keepalived).
  - Simpler and easy to reason about; passive node is idle until failover.
  - Wasteful of resources but straightforward to implement.

Current doc: describes a single HAProxy (single active instance). To remove LB SPOF you should deploy either:
- Two HAProxy instances in Active-Active with a VIP or DNS strategy; or
- Two HAProxy instances in Active-Passive coordinated via keepalived (VRRP).

## How Primary-Replica (Master-Slave) MySQL works

- Primary (Master) takes all write operations (INSERT/UPDATE/DELETE) and records changes in a binary log (binlog).
- Replicas (Slaves) connect to the Primary and read the binlog events, replaying them locally to reproduce the Primary's state.
- Replication modes:
  - Asynchronous: Replicas apply changes after the Primary commits; this can cause replication lag.
  - Semi-synchronous: Primary waits for acknowledgement from at least one Replica before reporting commit success — reduces chance of data loss but increases write latency.
- Failover: if the Primary fails, one Replica can be promoted to Primary. Promotion can be manual or automated via tools (Orchestrator, MHA, or custom scripts). After promotion, former Primary must be re-added as a Replica.

## Primary vs Replica from the application's perspective

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

- Single Points of Failure (SPOF):
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

## Minimal next-step artifacts (I can add these if you want)

- Example `haproxy.cfg` implementing Round Robin with simple HTTP health checks.
- Example `keepalived` configuration for an Active-Passive HAProxy setup (VRRP).
- Small `ufw` ruleset to lock down SSH, allow LB->backends, and allow LB ports.
- Minimal Prometheus `node_exporter` config and Grafana dashboard stub.

---

If you'd like, I will now:
- Add the `haproxy.cfg` example and `keepalived` example into this repository; or
- Add a small `README.md` with run/verify steps; or
- Add firewall rules and a minimal Prometheus quickstart.

Tell me which artifact you'd like next and I'll create it (I'll update the todo list and start that work).