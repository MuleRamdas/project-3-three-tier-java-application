# Three-Tier Java Application Deployment on AWS

A hands-on DevOps project that deploys a **three-tier Java web application** on AWS using a custom VPC with public and private subnets, an Nginx reverse proxy, an Apache Tomcat application server, and a managed MariaDB database on Amazon RDS.

![AWS](https://img.shields.io/badge/AWS-VPC%20%7C%20EC2%20%7C%20RDS-orange)
![Nginx](https://img.shields.io/badge/Nginx-Reverse%20Proxy-green)
![Tomcat](https://img.shields.io/badge/Apache%20Tomcat-9-yellow)
![MariaDB](https://img.shields.io/badge/MariaDB-RDS-blue)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Network Design](#network-design)
- [Implementation Steps](#implementation-steps)
- [Verification](#verification)
- [Screenshots](#screenshots)
- [Key Learnings](#key-learnings)
- [Future Improvements](#future-improvements)
  
---

## Overview

The goal of this project is to design and deploy a secure, layered architecture on AWS where each tier is isolated in its own subnet:

| Tier | Component | Subnet |
|------|-----------|--------|
| Presentation | Nginx Reverse Proxy (EC2) | Public |
| Application | Java + Apache Tomcat (EC2) | Private Subnet 1 |
| Data | Database EC2 server + Amazon RDS (MariaDB) | Private Subnet 2 |

Only the proxy server is exposed to the internet. The application and database tiers are private and reachable only from within the VPC.

**Region:** Asia Pacific (Mumbai), `ap-south-1`

---

## Architecture

```
                Internet
                   |
                   v
           Internet Gateway
                   |
    ┌──────────────┴───────────────┐
    │  Public Subnet (ap-south-1a) │
    │      Nginx Proxy Server      │
    └──────────────┬───────────────┘
                   |
    ┌──────────────┴───────────────┐
    │ Private Subnet 1 (1b)        │
    │  Tomcat Application Server   │
    └──────────────┬───────────────┘
                   |
    ┌──────────────┴───────────────┐
    │ Private Subnet 2 (1c)        │
    │  DB Server + RDS (MariaDB)   │
    └──────────────────────────────┘

  NAT Gateway (in Public Subnet) gives private
  subnets outbound-only internet access.
```

---

## Tech Stack

- **Cloud:** AWS (VPC, EC2, RDS, Internet Gateway, NAT Gateway, Route Tables, Security Groups)
- **Web / Reverse Proxy:** Nginx
- **Application Server:** Apache Tomcat 9
- **Language / Runtime:** Java
- **Database:** MariaDB on Amazon RDS
- **OS:** Amazon Linux

---

## Network Design

### VPC

| Setting | Value |
|---------|-------|
| VPC Name | `three-tire-architecture` |
| CIDR | `10.0.0.0/16` |

### Subnets

| Subnet | Availability Zone | CIDR | Purpose |
|--------|-------------------|------|---------|
| public-subnet | ap-south-1a | 10.0.0.0/20 | Nginx Proxy |
| private-subnet-1 | ap-south-1b | 10.0.16.0/20 | Tomcat Application |
| private-subnet-2 | ap-south-1c | 10.0.32.0/20 | Database Layer |

### Gateways and Routing

| Resource | Details |
|----------|---------|
| Internet Gateway | Provides internet connectivity to the public subnet |
| NAT Gateway | `three-tire-architecture-nat-g`, placed in the public subnet, gives private subnets outbound-only internet access |
| Public Route Table | `0.0.0.0/0 → Internet Gateway` (associated with public subnet) |
| Private Route Table | `0.0.0.0/0 → NAT Gateway` (associated with both private subnets) |

### Security Group

Ports opened for the infrastructure:

| Type | Port | Purpose |
|------|------|---------|
| SSH | 22 | Administrative access |
| HTTP | 80 | Web traffic to Nginx |
| Custom TCP | 8080 | Tomcat |
| MySQL/Aurora | 3306 | Database access |

> In production, restrict these rules by source (for example, allow 3306 only from the application server's security group and SSH only from your own IP).

---

## Implementation Steps

### Part 1: Network Infrastructure (AWS Console)

1. **VPC:** Create a VPC (`VPC only`) with CIDR `10.0.0.0/16`.
2. **Subnets:** Create `public-subnet` (1a), `private-subnet-1` (1b) and `private-subnet-2` (1c) using the CIDRs above. Enable *auto-assign public IPv4 address* on the public subnet only.
3. **Internet Gateway:** Create an Internet Gateway and attach it to the VPC.
4. **Public Route Table:** Add route `0.0.0.0/0 → Internet Gateway` and associate the public subnet.
5. **NAT Gateway:** Create a Zonal NAT Gateway in the public subnet with a new Elastic IP.
6. **Private Route Table:** Create it in the same VPC, add route `0.0.0.0/0 → NAT Gateway`, and associate both private subnets.
7. **Verify:** Open the VPC **Resource map** and confirm all subnets, route tables and gateways are connected correctly.

### Part 2: Security Group and Servers

1. Create one Security Group shared by all instances, with inbound rules for **SSH, HTTP, MySQL/Aurora and Custom TCP (Tomcat)**. Outbound rules are left at default because Security Groups are stateful.
2. Launch three EC2 instances with the same key pair and Security Group:

| Instance | Subnet | Role |
|----------|--------|------|
| `proxy` | public-subnet | Nginx reverse proxy and jump server |
| `app` | private-subnet-1 | Java + Tomcat application server |
| `database` | private-subnet-2 | Database server |

### Part 3: Amazon RDS (MariaDB)

1. RDS → Create database → **Full configuration** → Engine: **MariaDB**.
2. DB instance identifier: `Three-Tier-RDS`, master username: `admin`, auto-generated password (stored securely, never committed to GitHub).
3. Connectivity: select the project VPC, attach the project Security Group (remove the default one), Availability Zone `ap-south-1c` (same AZ as private-subnet-2).
4. Wait until the status becomes **Available**.

### Part 4: Server Access via Jump Server

The private servers have no public IP, so the proxy server is used as a jump host.

```bash
# Copy the key to the proxy server
scp -i key.pem key.pem ec2-user@<PROXY_PUBLIC_IP>:/home/ec2-user/

# Log in to the proxy server
ssh -i key.pem ec2-user@<PROXY_PUBLIC_IP>

# From the proxy, reach the private servers
ssh -i key.pem ec2-user@<APP_PRIVATE_IP>
ssh -i key.pem ec2-user@<DB_PRIVATE_IP>
```

Hostnames were set on each server for easy identification:

```bash
sudo hostnamectl hostname proxy   # on the proxy server
sudo hostnamectl hostname app     # on the app server
sudo hostnamectl hostname db      # on the database server
```

### Part 5: Proxy Server (Nginx Reverse Proxy)

```bash
sudo yum update -y
sudo yum install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

Edit `/etc/nginx/nginx.conf` and add a `location` block inside the `server` block so requests are forwarded to Tomcat:

```nginx
location / {
    proxy_pass http://<APP_PRIVATE_IP>:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

```bash
sudo systemctl restart nginx
```

### Part 6: Application Server (Java + Tomcat)

```bash
sudo yum update -y
sudo yum install java -y
java --version

# Download Tomcat 9 (tar.gz) and extract to /opt
sudo wget <TOMCAT_9_DOWNLOAD_URL>
sudo tar -xvzf apache-tomcat-9.*.tar.gz -C /opt
sudo mv /opt/apache-tomcat-9.* /opt/apache-tomcat

# Start Tomcat
sudo -i
cd /opt/apache-tomcat/bin
./catalina.sh start
```

### Part 7: Test

Open `http://<PROXY_PUBLIC_IP>` in a browser. The Tomcat page is served through Nginx from the private application server, which confirms the full request path works.

---

## Verification

The following components were created and verified:

- [x] Custom VPC
- [x] Public Subnet and two Private Subnets
- [x] Internet Gateway
- [x] NAT Gateway
- [x] Public and Private Route Tables
- [x] Security Group
- [x] Proxy EC2 (Nginx running)
- [x] Application EC2 (Java and Tomcat running)
- [x] Database EC2 in Private Subnet 2
- [x] RDS MariaDB instance
- [x] Tomcat page reachable through the Nginx proxy public IP
- [x] SSH access to private servers via the proxy server

---

## Screenshots

### VPC Resource Map

The resource map shows the VPC, the three subnets across `ap-south-1a`, `ap-south-1b` and `ap-south-1c`, the public and private route tables, the Internet Gateway and the NAT Gateway.

![VPC Resource Map](screenshots/vpc-subnets.png)

---

## Key Learnings

- Designing a custom VPC with public and private subnets
- Configuring Internet Gateway, NAT Gateway and Route Tables
- Isolating application and database tiers from the internet
- Writing Security Group rules for each tier
- Setting up Nginx as a reverse proxy
- Installing and running Java and Apache Tomcat on Linux
- Accessing private EC2 instances through a jump server
- Provisioning a managed MariaDB database on Amazon RDS

---

## Future Improvements

- Use SSH agent forwarding (`ssh -A`) or AWS Systems Manager Session Manager instead of copying the key to the proxy server
- Provision the whole infrastructure with **Terraform** or **CloudFormation**
- Add an **Application Load Balancer** and Auto Scaling Group
- Enable HTTPS with ACM and a custom domain (Route 53)
- Enable RDS Multi-AZ for high availability
- Add CI/CD with GitHub Actions or Jenkins
- Add monitoring with CloudWatch

---


