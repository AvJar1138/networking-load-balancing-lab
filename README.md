# Application-Layer Round-Robin Load Balancing with Nginx

## 1. Overview

This lab demonstrates application-layer load balancing using Nginx and Containerlab.

An Nginx load balancer receives HTTP requests from a client and distributes them between two backend web servers using the Round-Robin scheduling algorithm.

Each backend serves a different HTML page, making it possible to identify which server handled each request.

### Objectives

- Build a four-node network topology using Containerlab.
- Configure IP addresses and network interfaces using node-level `exec` commands.
- Configure Nginx as a reverse proxy and load balancer.
- Use file bind mounts to provide custom Nginx configuration and HTML pages.
- Verify Round-Robin request distribution using HTTP requests from the client container.

## 2. Network Topology

The lab consists of four containers:

| Node | Role | Image |
|---|---|---|
| `client` | Generates HTTP requests | `alpine:latest` |
| `lb` | Nginx load balancer | `nginx:alpine` |
| `web1` | Backend web server 1 | `nginx:alpine` |
| `web2` | Backend web server 2 | `nginx:alpine` |

The load balancer connects three separate subnets.

### IP Addressing

| Node | Interface | IP Address | Subnet |
|---|---|---|---|
| `client` | eth1 | 10.0.1.10/24 | 10.0.1.0/24 |
| `lb` | eth1 | 10.0.1.100/24 | 10.0.1.0/24 |
| `lb` | eth2 | 10.0.2.100/24 | 10.0.2.0/24 |
| `web1` | eth1 | 10.0.2.10/24 | 10.0.2.0/24 |
| `lb` | eth3 | 10.0.3.100/24 | 10.0.3.0/24 |
| `web2` | eth1 | 10.0.3.10/24 | 10.0.3.0/24 |

### Traffic Flow

1. The client sends an HTTP request to `10.0.1.100`.
2. Nginx receives the request on port 80.
3. Nginx forwards the request to one of the backend web servers.
4. The selected backend returns its HTML page through the load balancer.
5. Subsequent requests are distributed using Round-Robin scheduling.

## 3. Project Structure

The project uses the following directory structure:

```text
lablb/
├── load-balancing.clab.yml
├── README.md
└── config/
    ├── nginx.conf
    ├── web1.html
    └── web2.html
```

### File Descriptions

- `load-balancing.clab.yml`: Defines the nodes, container images, IP configuration, interface setup, links, and bind mounts.
- `config/nginx.conf`: Defines the Nginx upstream servers and reverse proxy.
- `config/web1.html`: Custom HTML page served by the first backend.
- `config/web2.html`: Custom HTML page served by the second backend.
- `README.md`: Documents the topology, deployment, and verification procedure.

## 4. Nginx Configuration

The `nginx.conf` file defines an upstream group containing both backend servers:

```nginx
events {
    worker_connections 1024;
}

http {
    upstream backend_servers {
        server 10.0.2.10:80;
        server 10.0.3.10:80;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend_servers;
        }
    }
}
```

Nginx uses Round-Robin scheduling by default when multiple servers are defined in an upstream group without another balancing method.

The load balancer listens on port 80 and forwards incoming HTTP requests to the configured backend group.

## 5. File Bind Mounts

Containerlab bind mounts provide the configuration files from the project directory to the appropriate locations inside the containers.

The load balancer uses:

```yaml
binds:
  - config/nginx.conf:/etc/nginx/nginx.conf
```

The backend servers use:

```yaml
binds:
  - config/web1.html:/usr/share/nginx/html/index.html
```

```yaml
binds:
  - config/web2.html:/usr/share/nginx/html/index.html
```

These mounts allow the containers to use the custom Nginx configuration and backend HTML pages without manually copying the files into the containers.

## 6. Deployment

From the project root directory, deploy the lab using:

```bash
containerlab deploy --topo load-balancing.clab.yml
```


## 7. Verification and Testing

The main verification test sends four HTTP requests to the load balancer from inside the client container.

### Test 1: Round-Robin Distribution

**Command:**

```bash
docker exec -it clab-load-balancing-client sh -c 'for i in $(seq 1 4); do wget -qO- http://10.0.1.100; echo ""; done'
```

**Expected behavior:**

The responses should alternate between the two backend servers:

1. Backend server 1 (`web1`)
2. Backend server 2 (`web2`)
3. Backend server 1 (`web1`)
4. Backend server 2 (`web2`)

The custom HTML pages make it possible to identify which backend handled each request.

### Test Results

The lab was deployed successfully, and the four-request HTTP test completed successfully. The responses alternated between the two custom backend HTML pages, demonstrating that the Nginx Round-Robin load balancing configuration worked as intended.

## 8. Cleanup

After completing the tests, remove the lab containers and their Containerlab-managed connections:

```bash
containerlab destroy --topo load-balancing.clab.yml
```


The project files remain available for subsequent deployments and testing.

## 9. Conclusion

This lab demonstrates how Nginx can distribute HTTP requests across multiple backend web servers using Round-Robin scheduling.

It also provides practical experience with Containerlab topology definitions, Linux network interfaces, IP addressing, Docker containers, Nginx reverse proxy configuration, and bind mounts.
