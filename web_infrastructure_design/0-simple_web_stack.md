# Simple web infrastructure

![Simple Web Infrastructure Diagram](./0-simple_web_stack.drawio_v4.svg)

## User story

A user wants to access www.foobar.com via their browser.
The browser verifies if the domain is known by the local device (in cache or otherwise). 
If not found, the browser sends a DNS query to resolve www.foobar.com to an IP address. 
The DNS responds with the A record (8.8.8.8), the IP address corresponding to the requested domain.
The browser connects to 8.8.8.8 on port 80 (HTTP) or 443 (HTTPS).
The web server (Nginx) receives the request and serves static files and/or forwards the request to the application server (process running at a certain port) to produce dynamic content.
The application server uses the application files to satisfy the application's logic.
If necessary, the application server may read/write persistent data from/to the local database (MySQL).
The application server sends back the necessary data to the web server (Nginx) and this returns a response to the user's browser (e.g., in the form of an html document).

## Architecture (single-server infrastructure)

High-level layout of a single-server (physical or virtual) architecture with public IP address 8.8.8.8:

- **DNS**
	- Domain: foobar.com
	- DNS A record: `www.foobar.com`, resolving to 8.8.8.8
- **Server (IPv4 8.8.8.8)**
	- Web server (Nginx)
	- Application server (e.g., Python)
	- Application files (codebase: static assets, backend code)
	- MySQL database

## Infrastructure Q&A 

- **What is a server?**
	A server is a computer (physical or virtual) that provides services over a network, i.e., a computer not meant to be interacted with directly.

- **What is the role of the domain name?**
	A domain name (example.com) is a human-readable name for a server located an IP address resolved using the Domain Name System (DNS).

- **What type of DNS record is `www` in `www.foobar.com`?**
	In this case, `www` corresponds to an A record (Address record) resolving directly to an IP address. 
	However, in general it could also refer to a CNAME record pointing to another hostname like the domain name `foobar.com`.

- **What is the role of the web server?**
	Its role is to accept HTTP(S) connections, serve static files (e.g., HTML, images), and transmit requests to the application server for dynamic content.

- **What is the role of the application server?**
	The application server is the process charged with the application logic (including interacting with the database), i.e., running the backend code.

- **What is the role of the database (MySQL)?**
	Stores all data that needs to persist between connections.

- **How does the server communicate with the user's computer?**
	Using a TCP/IP connection to the server's IP on the approppriate ports, typically 80 for the HTTP protocol and 443 for HTTPS.

## Issues with the single-server infrastructure

- **Single Point Of Failure (SPOF):**
	As a the infrastructure relies on a single server containing all services; if the server fails, the whole system fails.

- **Downtime during maintenance:**
	To maintain (updates, fixes) the server all services need to be restarted, which causes the website to be down (inaccessible).

- **Cannot scale for high traffic:**
	If the site becomes popular and many users attempt to use it simultaneously, the server may reach its maximum capacity (in terms of ressources) and slow down, become unavailable, or crash.
	The architecture could be scaled up by adding more ressources to the server, but this method is not tenable for long.
