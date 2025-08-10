# Configuring Nginx as a Load-Balancer and Reverse-Proxy


## Environment Setup

We have three Ubuntu nodes in our test project:

- **nginx**: Nginx load balancer/Reverse Proxy
- **node01** & **node02**: Apache backend servers

On each backend, Apache is running and UFW is active but only port 22 is open by default:

```bash
# On node01 (repeat on node02)
sudo apt update
sudo apt install apache2
sudo systemctl start apache2
sudo systemctl enable apache2
sudo ufw enable
sudo ufw allow 22
```
## Find IP of Nginx (Load-Balancer/Reverse-Proxy)

<img width="1028" height="534" alt="image" src="https://github.com/user-attachments/assets/a3a2c6b0-575d-4877-885c-0b2eae72052d" />


## Allow Only the Load-Balancer/Reverse-Proxy on node01 and node02

For security, restrict HTTP access so only the load-balancer/Reverse-Proxy (IP 192.168.142.225) can reach port 80 on your backends:

```bash
# On node01 
sudo ufw enable # to enable uncomplicated-firewall
sudo ufw allow from 192.168.142.225 to any port 80
sudo ufw reload
sudo ufw status # verify Rules
```

<img width="948" height="411" alt="image" src="https://github.com/user-attachments/assets/15fbfddd-835e-4c12-b310-a3d28ca16f3f" />



Same steps on noed02
```bash
# On node02 
sudo ufw enable # to enable uncomplicated-firewall
sudo ufw allow from 192.168.142.225 to any port 80
sudo ufw reload
sudo ufw status # verify Rules
```
<img width="948" height="360" alt="image" src="https://github.com/user-attachments/assets/f63adfc1-2b03-43e7-84cf-7b47b41577a4" />

# Frist configure Nginx as a load-balancer using three common algorithms:
1. **Round Robin**: Distributes requests evenly across all backend servers
2. **Weighted Round Robin**: Assigns traffic proportionally based on server weights
3. **IP Hash**: Routes each client IP to the same backend to maintain session persistence
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/663da7c4-ff55-42ef-baac-8ba77c17a5b1" />


On the **nginx** node, create or edit your site configuration:

```bash
sudo nano /etc/nginx/sites-available/load-balancer
```
Start with a basic server block:
<img width="708" height="350" alt="image" src="https://github.com/user-attachments/assets/a66ce654-708f-4109-9163-ff87c8604341" />


### 1. Round Robin

Add an `upstream` block for your backend pool:
<img width="857" height="462" alt="image" src="https://github.com/user-attachments/assets/b4d628e1-7221-403a-b544-a0e358b95f4b" />


### 2. Weighted Round Robin

Assign different weights to servers based on their capacity:
<img width="857" height="462" alt="image" src="https://github.com/user-attachments/assets/7c00646e-db19-4fc2-849d-bedf25b6e4fc" />

### 3. IP Hash

Enable `ip_hash` for session stickiness by client IP:
<img width="873" height="479" alt="image" src="https://github.com/user-attachments/assets/69e07a9c-4d6c-4771-b3f0-ff0870cd3c8b" />


## Activating the Configuration

Enable the site and restart Nginx:

```bash
# Enable the configuration
sudo ln -s /etc/nginx/sites-available/load-balancer /etc/nginx/sites-enabled/

# Test the configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
sudo systemctl enable nginx
```

## Testing the Load Balancer

Create different content on each backend server to verify load balancing:

```bash
# On node01
echo "<h1>Backend Server 1 - Node01</h1>" | sudo tee /var/www/html/index.html

# On node02  
echo "<h1>Backend Server 2 - Node02</h1>" | sudo tee /var/www/html/index.html
```
---
# Second configure Nginx as a Reverse-Proxy

#### What Is a Reverse Proxy?
A reverse proxy sits between clients and one or more backend servers. It receives incoming requests, routes them to the appropriate server pool, and returns the server’s response to the client. Common use cases include:

Hiding backend server identities
SSL/TLS offloading
Caching static assets
Distributing traffic across multiple application servers
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/40ddad88-c7bc-4b3d-a8b2-d71dafa2d7d1" />
#### SSL/TLS Termination (Offloading)
Offloading SSL/TLS decryption to the reverse proxy reduces CPU load on your application servers. Clients connect over HTTPS to the proxy, which decrypts the traffic, 
forwards plain HTTP to backends, then re-encrypts responses.
#### NGINX as a reverse proxy and simple load balancer for two Flask apps running on separate hosts (node01 and node02) at port 5000. Incoming HTTP requests hit NGINX, which forwards them to healthy backends—improving scalability and fault tolerance.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/23eae0fe-e194-426c-83c1-f762db14fd94" />

## Enviroment Setup
Below is the topology for this Configuration:
<table><thead><tr><th scope="col">Host</th><th scope="col">Role</th><th scope="col">IP Address</th></tr></thead><tbody><tr><td><code>nginx</code></td><td>Reverse proxy server (NGINX)</td><td>192.230.206.12</td></tr><tr><td><code>node01</code></td><td>Flask application (port 5000)</td><td>192.230.206.3</td></tr><tr><td><code>node02</code></td><td>Flask application (port 5000)</td><td>192.230.206.6</td></tr></tbody></table>

### Step 1: Verify Backend Servers
On node01:

```bash
root@node01 ~ ➜ curl localhost:5000
# <h1>Hello, Human!</h1>[Not Authenticated]
```
On node02:
```bash
root@node02 ~ ➜ curl localhost:5000
# <h1>Hello, Human!</h1>[Not Authenticated]
```
### Step 2: Configure Firewall Rules
If UFW is already enabled, skip sudo ufw enable and proceed with rule creation.
```bash
# 1. Enable UFW (if not already running)
sudo ufw enable


# 2. Allow SSH management
sudo ufw allow 22/tcp


# 3. Permit Flask traffic from the proxy only
sudo ufw allow from 192.230.206.12 proto tcp to any port 5000


# 4. Verify the active rules
sudo ufw status
# Status: active
# To                         Action      From
# --                         ------      ----
# 22/tcp                     ALLOW       Anywhere
# 5000/tcp                   ALLOW       192.230.206.12
# 22/tcp (v6)                ALLOW       Anywhere (v6)
```
### Step 3: Configure NGINX as a Reverse Proxy
1. Remove the default site and switch to the config directory:

```bash
cd /etc/nginx/sites-enabled
sudo rm default
cd /etc/nginx/sites-available
```
2. Create /etc/nginx/sites-available/helloworld with upstream and server blocks:
```nginx
# Upstream definition for Flask backends
upstream hello_world {
    server 192.230.206.3:5000;
    server 192.230.206.6:5000;
}


server {
    listen 80;
    server_name helloworld.com;


    root /var/www/html;
    index index.html index.htm;


    location / {
        proxy_pass http://hello_world;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
3. Enable the site and reload NGINX:
```bash
sudo ln -s /etc/nginx/sites-available/helloworld /etc/nginx/sites-enabled/
sudo nginx -t
sudo nginx -s reload
```

### Step 4: Test the Reverse Proxy

Use curl with a custom Host header to emulate requests to helloworld.com:
```bash
# Root path
curl -H "Host: helloworld.com" http://localhost
# /foo endpoint
curl -H "Host: helloworld.com" http://localhost/foo
# /bar endpoint
curl -H "Host: helloworld.com" http://localhost/bar
# <h1>Bar page</h1>...
```
### Step 5: Simulate a Backend Failure
Comment out one server in the upstream block to test NGINX’s resilience:

```bash
 /etc/nginx/sites-available/helloworld
```
```nginx
upstream hello_world {
    server 192.230.206.3:5000;
    # server 192.230.206.6:5000;  # node02 offline
}
```
Reload NGINX and re-run the curl tests; traffic should automatically route to the healthy backend.

```bash
sudo nginx -s reload
curl -H "Host: helloworld.com" http://localhost/foo
curl -H "Host: helloworld.com" http://localhost/bar
```

## Links and References

- [Nginx Load Balancing Documentation](http://nginx.org/en/docs/http/load_balancing.html)
- [UFW Documentation](https://help.ubuntu.com/community/UFW)
- [Apache HTTP Server Documentation](https://httpd.apache.org/docs/)
- [Nginx Upstream Module](http://nginx.org/en/docs/http/ngx_http_upstream_module.html)

---

