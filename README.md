# <h1 align="center"><font color="#FF9900">Production-Grade Scalable 3-Tier Web Application on AWS</font></h1>
<h3 align="center"><font color="#232F3E">AWS Solutions Architect – Associate Graduation Project</font></h3>

---

## <font color="#FF9900">Executive Summary & Project Overview</font>

This project presents a highly available, fault-tolerant, and scalable 3-tier web application architecture deployed on Amazon Web Services (AWS) in accordance with the **AWS Well-Architected Framework**. 

The architecture spans **three Availability Zones (AZ A, AZ B, and AZ C)** within a single AWS Region, leveraging strict network isolation across **Public**, **Web (Private)**, **Application (Private)**, and **Database (Private)** subnets. Edge services including **Amazon CloudFront**, **AWS WAF**, and **Amazon Route 53** safeguard and optimize user access, while **Application Load Balancers (ALB)** and **Auto Scaling Groups (ASG)** ensure dynamic scaling and high availability for compute resources.

---

## <font color="#FF9900">Solution Architecture Diagram</font>

Below is the visual representation of the production architecture designed in Lucidchart:

![AWS Solution Architecture Diagram](./AWSWebApplicationArchitectureDiagram.png)

---

## <font color="#FF9900">Comprehensive Architecture & Flow Breakdown</font>

### 1. Edge & DNS Layer (Global Services)
* **Amazon Route 53:** Serves as the authoritative Domain Name System (DNS) service. It routes end-user requests to the global CDN endpoint via an Alias record and monitors system availability using active health checks.
* **Amazon CloudFront:** Functions as the Global Content Delivery Network (CDN), caching static assets (e.g., images, CSS, JavaScript) at edge locations worldwide to reduce latency and minimize origin load.
* **AWS WAF (Web Application Firewall):** Integrated directly at the CloudFront/Edge perimeter to inspect incoming HTTP/HTTPS traffic. Custom WAF rule groups block common attack vectors, including the **OWASP Top 10** (SQL Injection, Cross-Site Scripting, Rate Limiting, etc.).

---

### 2. Network Infrastructure & Segmentation (VPC)
The core infrastructure is built inside a dedicated **Virtual Private Cloud (VPC)** with CIDR block `10.0.0.0/16`, attached to an **Internet Gateway (IGW)** for ingress and egress routing.

The VPC is logically partitioned across **3 Availability Zones (AZ A, AZ B, AZ C)** with a strict 4-tier subnet layout:

| Tier / Subnet Type | AZ A CIDR | AZ B CIDR | AZ C CIDR | Internet Accessibility |
| :--- | :--- | :--- | :--- | :--- |
| **Public Subnets** | `10.0.1.0/24` | `10.0.2.0/24` | `10.0.3.0/24` | Direct via IGW |
| **Web Tier (Private)** | `10.0.11.0/24` | `10.0.12.0/24` | `10.0.13.0/24` | Outbound via NAT GW |
| **App Tier (Private)** | `10.0.21.0/24` | `10.0.22.0/24` | `10.0.23.0/24` | Outbound via NAT GW |
| **Database Tier (Private)** | `10.0.31.0/24` | `10.0.32.0/24` | `10.0.33.0/24` | Isolated (No Internet) |

#### Cost-Optimized Outbound NAT Architecture Note
To balance production-grade resiliency with operational cost management:
* **NAT Gateways** are deployed in the **Public Subnets of AZ A and AZ B**.
* Private EC2 instances in AZ A and AZ B route outbound internet traffic (such as OS updates and package dependencies) through their local AZ NAT Gateway.
* Private EC2 instances in **AZ C** route outbound internet traffic through the **NAT Gateway in AZ A**. 
* *Rationale:* Deploying a 3rd NAT Gateway in AZ C incurs additional hourly provisioning costs (~$32+/month + data processing fees). Routing AZ C outbound traffic cross-AZ to AZ A achieves high availability and redundancy while reducing unnecessary infrastructure expenditure.

---

### 3. Compute & Load Balancing Tiers

#### A. Application Load Balancer (ALB)
* Deployed across the **Public Subnets** in AZ A, AZ B, and AZ C.
* Accepts traffic from the Internet Gateway, performs SSL/TLS termination, and evaluates Layer 7 routing rules.
* Health checks actively monitor Web Tier instances; traffic is automatically routed away from unhealthy nodes.

#### B. Web Tier (Presentation Layer)
* Contains EC2 instances residing in dedicated **Private Web Subnets**.
* Managed by an **Auto Scaling Group (ASG)** using Launch Templates.
* Receives HTTP/HTTPS traffic exclusively from the ALB.
* Performs web server functions and forwards business logic requests down to the Application Tier.

#### C. Application Tier (Business Logic Layer)
* Contains EC2 instances residing in dedicated **Private App Subnets**.
* Managed by a secondary **Auto Scaling Group (ASG)** with Target Tracking scaling policies (scaling on CPU utilization / Request Count).
* Receives API and backend execution requests exclusively from the Web Tier.
* Initiates secure SQL data queries down to the Primary Database Instance.

---

### 4. Database Tier (Persistence Layer)

* **Amazon RDS Multi-AZ DB Cluster (MySQL / PostgreSQL):**
  * **Primary DB Instance:** Located in Private DB Subnet - AZ A (handles all read/write traffic).
  * **Standby DB Instances:** Located in Private DB Subnets across AZ B and AZ C.
* **Synchronous Replication:** Storage-level synchronous replication ensures zero-data-loss failover between the Primary instance and Standby nodes. In the event of an AZ failure in AZ A, AWS RDS automatically triggers seamless failover to AZ B or AZ C without manual intervention.

---

### 5. Security & Instance Access Control

#### A. Security Group (SG) Chain (Least Privilege Enforced)
1. **ALB Security Group:** Inbound `80/443` allowed from `0.0.0.0/0` (or CloudFront IP ranges).
2. **Web Tier SG:** Inbound HTTP/HTTPS allowed **ONLY** from the **ALB Security Group**.
3. **App Tier SG:** Inbound Application Port (e.g., `8080`) allowed **ONLY** from the **Web Tier Security Group**.
4. **Database Tier SG:** Inbound DB Port (`3306` MySQL / `5432` PostgreSQL) allowed **ONLY** from the **App Tier Security Group**.

#### B. Bastion-Free Management via AWS Systems Manager (SSM)
* **AWS Systems Manager Session Manager** is enabled for all Web and App EC2 instances.
* Administrators gain secure shell access via AWS Console or AWS CLI without opening inbound SSH port `22` or exposing public IP addresses/bastion hosts.

---

### 6. Monitoring, Observability & Alerts

* **Amazon CloudWatch:** Collects system performance metrics (CPU Utilization, Memory, Network I/O, ALB HTTP 5xx errors, RDS Latency). CloudWatch Dashboards display real-time system health.
* **Amazon SNS (Simple Notification Service):** Connected to CloudWatch Alarms. Triggers immediate email / SMS notifications to system administrators whenever auto-scaling thresholds or failure conditions are triggered.

---

## <font color="#FF9900">Key AWS Services Summary</font>

| Category | AWS Service | Purpose in Architecture |
| :--- | :--- | :--- |
| **Networking & Content Delivery** | **Amazon VPC** | Isolated cloud network with 3-AZ subnet hierarchy |
| | **Amazon Route 53** | Global DNS routing and domain health monitoring |
| | **Amazon CloudFront** | Edge caching for static assets and global latency reduction |
| | **Application Load Balancer** | Layer 7 load distribution across Web instances |
| | **NAT Gateways** | Secure outbound internet access for private subnets (Cost-optimized in AZ A & B) |
| **Compute & Scaling** | **Amazon EC2** | Virtual compute instances for Web and App tiers |
| | **Auto Scaling Groups (ASG)** | Automated horizontal scaling based on metric policies |
| **Security & Identity** | **AWS WAF** | OWASP Top 10 web attack mitigation at the edge |
| | **Security Groups & NACLs** | Statefully and statelessly enforcing network isolation |
| | **AWS Systems Manager (SSM)** | Secure, bastion-free SSH/Session management |
| **Database** | **Amazon RDS Multi-AZ** | Managed relational DB with automatic synchronous failover |
| **Monitoring & Operations** | **Amazon CloudWatch** | Metrics collection, dashboards, and automated alarms |
| | **Amazon SNS** | Notification delivery for system alarms and auto-scaling events |

---

## <font color="#FF9900">Learning Outcomes Achieved</font>

* Designed a production-grade VPC with complete network segmentation across 3 Availability Zones.
* Configured edge security and caching mechanisms using AWS CloudFront, WAF, and Route 53.
* Implemented multi-tier Security Group chaining enforcing the Principle of Least Privilege.
* Implemented high-availability compute layers utilizing ALB listener rules and ASG Target Tracking policies.
* Designed an automated Multi-AZ database topology with synchronous failover capabilities.
* Applied bastion-free operational management using Systems Manager Session Manager.
* Executed architectural cost-optimization strategies regarding NAT Gateway placement without compromising system availability.
