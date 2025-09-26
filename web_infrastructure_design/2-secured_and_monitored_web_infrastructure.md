# Secured and Monitored Web Infrastructure

![Secured and Monitored Web Infrastructure Diagram](./2-secured_and_monitored_web_infrastructure_v2.svg)

## User story

1. A user wants to access `www.foobar.com` via their browser.
2. The browser verifies if the domain is known by the local device (in cache or otherwise).
3. If not found, the browser sends a DNS query to resolve `www.foobar.com` to an IP address.
4. The DNS responds with the A record (8.8.8.8), the IP address corresponding to the requested domain.
5. The browser connects to the load-balancer (HAproxy) at IP 8.8.8.8 on port 443 (HTTPS).
	- The connection between client and HAproxy is encrypted via SSL/TLS using the SSL certificate for `www.foobar.com`.
6. HAproxy evaluates the request and forwards it to one of the web servers (always https port 443) using the configured load distribution algorithm.
	- If HAproxy receives a request in port 80 it redirects it to port 443.
	- HAproxy allows the connection of the monitoring service/agent.
	- If HAproxy receives a request in a different port it is rejected by the firewall.
7. The web server (Nginx) receives the request (if not rejected by the firewall) and serves static files and/or forwards the request to the application server (process running at a certain port) to produce dynamic content.
	- The firewall protects the server from connections coming from external IPs other that of the load-balancer (HAproxy) and monitoring service (Sumologic) at port 443, or MySQL replication connection at port 3306.
	- The firewall also blocks connections coming from local IPs (127.0.0.x) to ports other than those specifically open for server operation (e.g., application server at port 9000, database at port 5432).
8. The application server uses the application files to satisfy the application's logic.
9. If necessary, the application server may read/write persistent data from/to the database cluster (MySQL), where writes go to the primary database and reads to the replicas.
	- If a write operation is performed it is recorded in a log that is accessed, read, and replayed by the replicas.  
10. The application server sends back the necessary data to the web server (Nginx).
11. The web server returns the encrypted response to the load-balancer and this returns the final encrypted response to the user's browser (e.g., in the form of an html document).
- All servers, including the load-balancer, run monitoring agents that collect data on server health (logs, events) and performance (metrics).

## Architecture (secured and monitored three-server + HAproxy infrastructure)

- **DNS**
  - Domain: foobar.com
  - DNS A record: `www.foobar.com`, resolving to 8.8.8.8 (the public IP of the load-balancer)

<div style="background: #582f2fff; border: 2px solid #3c0202ff; padding: 1em; text-align: center; font-size: 1.5em;">
<strong>⚠️ REVIEW NEEDED BELOW THIS POINT ⚠️</strong><br>
Everything below this line requires review and may be incomplete or unverified.
</div>

- **Load-balancer (HAProxy, IPv4 8.8.8.8)**
  - Public IP: 8.8.8.8
  - Listens on ports 80 (HTTP) and 443 (HTTPS)
  - Terminates SSL using an SSL certificate for `www.foobar.com`
  - Distributes requests to backend servers
  - Monitored by a monitoring client
  - Protected by a firewall

- **Servers (3 instances)**
  - Each server contains:
    - Web server (Nginx)
    - Application server (Python)
    - Application files
    - Database (MySQL) cluster member (one primary, two replicas)
    - Monitored by a monitoring client
    - Protected by a firewall

- **Firewalls (3 total)**
  - One firewall per server (including the load balancer)
  - Restricts access to only necessary ports and IPs

- **SSL Certificate**
  - Used by HAProxy to serve `www.foobar.com` over HTTPS
  - Ensures encrypted traffic between clients and the load balancer

- **Monitoring Clients (3 total)**
  - Installed on each server (including the load balancer)
  - Collects metrics, logs, and events for a monitoring service (e.g., Sumologic, Prometheus, Datadog)

## Why Each Additional Element is Added

- **Firewalls:**
  - Protect servers from unauthorized access
  - Restrict traffic to only required ports (e.g., 80/443 for HTTP/HTTPS, 3306 for MySQL)
  - Limit exposure to attacks and reduce risk of compromise

- **SSL Certificate (HTTPS):**
  - Encrypts traffic between users and the website
  - Protects sensitive data (login credentials, personal info)
  - Prevents eavesdropping and man-in-the-middle attacks
  - Builds user trust (browser shows secure padlock)

- **Monitoring Clients:**
  - Provide visibility into server health, performance, and security
  - Alert administrators to issues (downtime, high load, suspicious activity)
  - Collect metrics (CPU, memory, disk, network, QPS, error rates)
  - Enable proactive maintenance and troubleshooting

## What are Firewalls For?

Firewalls are network security devices or software that control incoming and outgoing traffic based on predefined rules. They:
- Block unauthorized access to servers
- Allow only trusted IPs and ports
- Prevent attacks such as port scanning, brute force, and exploitation of open services
- Can be host-based (e.g., `ufw`, `iptables`) or network-based (cloud security groups)

## Why is Traffic Served Over HTTPS?

- Encrypts all data exchanged between the user and the website
- Protects against interception and tampering
- Required for secure login, payments, and sensitive transactions
- Improves SEO ranking and user trust

## What is Monitoring Used For?

- Detects server failures, performance bottlenecks, and security incidents
- Provides real-time and historical metrics for analysis
- Enables alerting and automated responses to issues
- Helps with capacity planning and scaling decisions

## How the Monitoring Tool is Collecting Data

- Monitoring clients (agents) run on each server
- Collect system metrics (CPU, memory, disk, network)
- Collect application metrics (web server QPS, error rates, response times)
- Collect logs (access logs, error logs)
- Send data to a central monitoring service (e.g., Sumologic, Prometheus, Datadog)
- Data can be visualized in dashboards and used for alerting

## How to Monitor Web Server QPS (Queries Per Second)

- Configure the monitoring client to collect web server metrics (Nginx, Apache, etc.)
- Enable access log collection and parsing for request rates
- Use built-in plugins or exporters (e.g., Prometheus Nginx exporter)
- Visualize QPS in the monitoring dashboard
- Set up alerts for abnormal spikes or drops in QPS

## Issues with This Infrastructure

- **SSL Termination at Load Balancer Level:**
  - Issue: Traffic between the load balancer and backend servers is unencrypted (plain HTTP)
  - Risk: Internal traffic could be intercepted if the network is compromised
  <!-- - Mitigation: Use HTTPS between load balancer and backend servers, or isolate the internal network -->

- **Only One MySQL Server Accepts Writes:**
  - Issue: The Primary is a single point of failure for write operations
  - Risk: If the Primary fails, no writes are possible until failover
  <!-- - Mitigation: Use automated failover tools, or consider multi-primary (clustered) databases -->

- **Servers with All Components (DB, Web, App):**
  - Issue: Increases attack surface and resource contention
  - Risk: Compromise of one service can affect others; harder to scale and maintain
  <!-- - Mitigation: Separate roles (dedicated DB servers, web servers, app servers) for better security and scalability -->
