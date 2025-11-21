# Deploy 3 tier application (Proxy-server "Nginx", Back-End "go", DataBase "mysql")

```
# File Tree
📁 go-app-docker-k8s/
├── 📁 backend/
│   ├── 📄 db-password
│   ├── 📄 Dockerfile
│   ├── 📄 go.mod
│   ├── 📄 go.sum
│   └── 📄 main.go
├── 📄 compose.sh
├── 📄 compose.yml
├── 📄 docker-compose.yml
├── 📄 init.sql
├── 📁 k8s/
│   ├── 📄 configmap.yml
│   ├── 📄 deployment-db.yml
│   ├── 📄 deployment-go.yml
│   ├── 📄 deployment-nginx.yml
│   ├── 📄 namespace.yml
│   ├── 📄 pv-db.yml
│   ├── 📄 pvc-ns.yml
│   ├── 📄 secret-nginx.yaml
│   ├── 📄 secret.yml
│   └── 📄 start.sh
├── 📄 kindcluster.yml
├── 📁 nginx/
│   ├── 📄 Dockerfile
│   ├── 📄 generate-ssl.sh
│   ├── 📄 nginx.conf
│   └── 📁 ssl/
│       ├── 📄 localhost.crt
│       └── 📄 localhost.key
└── 📄 README.md
```

# Three-Tier Docker App with Unique L3 Ipvlan Networks

## Project Overview

This project deploys a three-tier application (Load Balancer, Go App, and Database) using Docker Compose. Unlike standard deployments, each container attaches to a unique Layer 3 ipvlan network. The setup explores advanced Docker networking possibilities—granting fine-grained control, ensuring host reachability, and aiming for inter-container communication via service names.

Reference repo for the base app:
[Deploy_3_Tier_Application](https://github.com/kareemloulah/Deploy_3_Tier_Application)

## Goals

- Each container is connected only to a **unique network**.
- Containers remain reachable from the host.
- Containers can communicate with each other via their Docker Compose service names (e.g., `goapp:8000`, `db:3306`).


## Docker Network Drivers Considered

- ipvlan (chosen for its granular L3 control)
- macvlan
- host
- overlay
- none


## Approach 1: Separate Ipvlan Networks

Three separate networks are defined, each with its own interface and subnet.

```yaml
networks:
  lb-network:
    driver: ipvlan
    driver_opts:
      parent: ens33.130
      ipvlan_mode: l3
      ipvlan_flag: bridge
    ipam:
      config:
        - subnet: 192.168.130.0/24

  go-network: 
    driver: ipvlan
    driver_opts:
      parent: ens33.110
      ipvlan_mode: l3
      ipvlan_flag: bridge
    ipam:
      config:
        - subnet: 192.168.110.0/24

  db-network:
    driver: ipvlan
    driver_opts:
      parent: ens33.120
      ipvlan_mode: l3
      ipvlan_flag: bridge
    ipam:
      config:
        - subnet: 192.168.120.0/24
```

>

**Result:**

- Host can ping containers.
- Containers CANNOT resolve each other’s service names, so the app fails (`nginx` expects `goapp:8000` and `goapp` expects `db:3306`).


**Note:** 
- could be solved my adding extra hosts to each container which i didnt like to do.
```
services:
  nginx:
    image: nginx
    networks:
      lb-network:
        ipv4_address: 192.168.130.10
    extra_hosts:
      - "goapp:192.168.110.10"
      - "db:192.168.120.10"
  goapp:
    image: goapp
    networks:
      go-network:
        ipv4_address: 192.168.110.10
    extra_hosts:
      - "nginx:192.168.130.10"
      - "db:192.168.120.10"
  db:
    image: mysql
    networks:
      db-network:
        ipv4_address: 192.168.120.10
    extra_hosts:
      - "nginx:192.168.130.10"
      - "goapp:192.168.110.10"
```

## Approach 2: One Ipvlan Network, Multiple Subnets

Use a single ipvlan network with three subnets for all containers. Each container is assigned a static IP.

```yaml
go-network:
  driver: ipvlan
  driver_opts:
    parent: ens33
    ipvlan_mode: l3
    ipvlan_flag: bridge
  ipam:
    config:
      - subnet: 192.168.150.0/24
      - subnet: 192.168.151.0/24
      - subnet: 192.168.152.0/24
```

**Result:**

- The application does NOT fail; containers reach one another.
- However, containers are NOT pingable from the host.

The parent interface must reference the host’s network interface, and manual creation of the virtual subnet on the host solves the connectivity issue.

## Host Network Setup (Manual Solution)

Create the network interface and route from the host:

```sh
sudo ip link add ipvlan0 link ens33 type ipvlan mode l3
sudo ip addr add 192.168.150.0/24 dev ipvlan0 # Example subnet (for NGINX)
sudo ip link set ipvlan0 up

sudo ip route add 192.168.150.0/24 dev ipvlan0 
sudo ip route add 192.168.150.0/24 via 192.168.100.26 # via host IP
```
