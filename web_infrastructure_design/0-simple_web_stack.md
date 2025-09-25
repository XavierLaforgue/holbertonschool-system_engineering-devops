![Simple Web Infrastructure Diagram](./0-simple_web_stack.drawio_v3.svg)

# Simple web infrastructure

## User story

A user wants to access www.foobar.com via their browser.
The browser verifies if the domain is known by the local device (in cache or otherwise). 
If not found, the browser sends a DNS query to resolve www.foobar.com to an IP address. 
The DNS responds with the A record (8.8.8.8), the IP address corresponding to the requested domain.
The browser connects to 8.8.8.8 on port 80 (HTTP) or 443 (HTTPS).
The web server (Nginx) receives the request and serves static files or communicates with the application server to produce dynamic content.
The application server uses the application files to execute the code required by the application's logic.
If necessary, the application server may read/write persistent data from/to the local database (MySQL).
The application server sends back the necessary data to the web server (Nginx) and this returns a response to the user's browser (e.g., in the form of an html document).

## Architecture (single-server infrastructure)

High-level layout of a single-server (physical or virtual) architecture with public IP address 8.8.8.8:

- DNS
	- Domain: foobar.com
	- DNS A record: www.foobar.com, resolving to 8.8.8.8
- Server (IPv4 8.8.8.8)
	- Web server (Nginx)
	- Application server (e.g., Python)
	- Application files (codebase: static assets, backend code)
	- MySQL database

## Infrastructure Q&A 

- What is a server?
	A server is a computer (physical or virtual) that provides services over a network, i.e., a computer not meant to be interacted with directly.

- What is the role of the domain name?
	A domain name (example.com) is a human-readable name for a server located an IP address resolved using the Domain Name System (DNS).

- What type of DNS record is `www` in `www.foobar.com`?
	In this case `www` corresponds to an A record (Address record) resolving directly to an IP address. 
	However, in general it could also refer to a CNAME record pointing to another hostname like the domain name `foobar.com`.

- What is the role of the web server?
	- Nginx accepts HTTP(S) connections from clients, serves static assets (HTML, CSS, JS, images) efficiently, handles TLS termination (SSL certificates), and acts as a reverse proxy to the application server for dynamic requests. It can also do gzip compression, caching, load balancing, request buffering, and connection management.

- What is the role of the application server?
	- The application server runs the website's backend code and application logic. It handles route processing, templating, business logic, authentication, sessions, and interacts with the database. Nginx proxies dynamic requests to the app server (often on localhost and a higher port or a UNIX socket). Examples: Node.js, Gunicorn (Python WSGI), Puma (Ruby), etc.

- What is the role of the database (MySQL)?
	- The database stores persistent data: users, posts, configurations, transactions, etc. The application server sends SQL queries to MySQL to read and write data. MySQL runs as a service on the same server in this architecture.

- How does the server communicate with the user's computer?
	- Communication happens over TCP/IP. For the web, the browser opens a TCP connection to the server's IP (8.8.8.8) on port 80 (HTTP) or 443 (HTTPS), using the HTTP protocol. If HTTPS is used, TLS encrypts the connection. The OS networking stack and the server software (Nginx) handle the transport and application layers. DNS resolution happens first so the browser knows which IP to connect to.

## Issues with this one-server design

- **Single Point of Failure (SPOF):**
	- All services (web server, application server, database) run on one machine. If the server fails, the entire website goes down. There is no redundancy.

- **Downtime during maintenance:**
	- Any maintenance (like OS updates, server reboots, or deploying new code) requires stopping services, causing downtime. Users cannot access the site during these periods.

- **Cannot scale for high traffic:**
	- One server has limited resources. If traffic increases beyond its capacity, the site will slow down or become unavailable. Scaling up (adding more CPU/RAM) has limits and can be costly. Horizontal scaling (adding more servers) is not possible with this setup.

## Mitigation suggestions

- Add more servers and use a load balancer to distribute traffic and reduce SPOF.
- Use database replication for high availability.
- Implement rolling deployments or blue-green deployments to minimize downtime.
- Use caching and a CDN to reduce load on the server and improve scalability.

