# End-to-End AWS 3-Tier Architecture with Docker & Real-Time Monitoring

![AWS Architecture Diagram](./images/architecture-diagram.png) 
*(Note: Upload your architecture diagram image to an `images` folder and link it here)*

## Project Overview
This project demonstrates a production-ready, highly available 3-tier architecture deployed on AWS. [cite_start]The foundational infrastructure includes a custom VPC, five carefully configured Security Groups, and customized route tables for precise traffic routing[cite: 1]. 

[cite_start]A key highlight of this project is the custom-built NAT Instance which acts as a router for private subnets[cite: 1], offering a cost-optimized alternative to managed NAT Gateways. [cite_start]The application layer features a containerized Java Spring Boot application (VProfile) that was cloned, built locally, and pushed to Amazon ECR[cite: 1]. [cite_start]This application is deployed via Launch Templates, managed by an Auto Scaling Group (ASG), and exposed through an Application Load Balancer (ALB)[cite: 2]. [cite_start]The backend relies on a secure, Multi-AZ RDS MySQL database configured with its own subnet group and dedicated security group[cite: 1].

[cite_start]Furthermore, an Event-Driven Serverless Monitoring Pipeline was implemented using CloudWatch alarms attached to the ASG, triggering an SNS topic and a Lambda function to send real-time alerts to a Discord webhook[cite: 28].

**Author:** Wajahat Rasool

---

## Prerequisites & Tools Required
To deploy, test, and modify this architecture, the following tools are utilized:
* **Git:** To clone the application source code.
* **AWS CLI:** To interact with AWS services and authenticate with the ECR repository.
* **Docker:** To build the application container image locally before pushing to AWS ECR.
* **MySQL Client (`mysql-client`):** To connect to the RDS endpoint from a bastion host and initialize the database schema.
* **SSH Client:** To connect securely to the NAT/Bastion instance and subsequent private EC2 instances for debugging.

---

## Architecture Component Flow (Integration & Automation)
This architecture relies on the seamless integration of compute, networking, and monitoring components:

1. **Traffic Ingress (ALB & TG):** The Application Load Balancer (ALB) sits in the public subnets and receives internet traffic. [cite_start]It continuously monitors the Target Group (TG), which contains the dynamic IPs of healthy EC2 instances[cite: 7]. [cite_start]The TG's health check status codes were explicitly modified to accommodate application redirects[cite: 2].
2. **Compute Scaling (ASG & Launch Template):** The Auto Scaling Group (ASG) manages instance count. [cite_start]When a new server is required, the ASG utilizes the assigned Launch Template[cite: 2].
3. **Application Deployment (Docker, IAM & ECR):** The Launch Template contains a `User Data` script. [cite_start]Upon boot, this script uses an attached IAM Role (`EC2-ECR-Push-Role`) [cite: 25] to securely authenticate with Amazon ECR, pull the custom Docker image, and run it.
4. **Outbound Internet via Custom Routing (NAT):** Because the ASG launches instances in private subnets, they cannot reach ECR directly. [cite_start]The custom NAT Instance intercepts their outbound requests[cite: 21], masquerades the IPs, fetches the Docker image from ECR via the IGW, and returns it to the private instances.
5. [cite_start]**Real-Time Alerting (CloudWatch, SNS, Lambda, Discord):** A CloudWatch alarm monitors the ASG's CPU metrics[cite: 28]. [cite_start]Upon breaching defined thresholds, it triggers an SNS topic[cite: 28]. [cite_start]An AWS Lambda function is subscribed to this topic; it processes the SNS payload using configured environment variables and pushes a formatted alert to a Discord channel via a Webhook URL[cite: 28].

---

## End-to-End Traffic Flow (Internet to Database)
[cite_start]When a user accesses the website via the ALB DNS, the traffic follows this path[cite: 3]:
1. [cite_start]**Internet Gateway (IGW):** The user's request first hits the VPC's IGW[cite: 4].
2. [cite_start]**Application Load Balancer (ALB):** The IGW forwards the request directly to the ALB residing in the Public Subnets[cite: 5]. [cite_start]The ALB evaluates incoming traffic on Port 80 to determine the routing destination[cite: 6].
3. [cite_start]**Private EC2 / Docker App:** The request is routed to the Docker container running on Port 8080 within the private subnet[cite: 8].
4. [cite_start]**RDS Database:** For login and data retrieval, the Java Tomcat App uses its connection string to request data from the RDS MySQL instance (Port 3306) located in the isolated private subnet[cite: 9].

---

## Security Groups: The Virtual Firewalls
[cite_start]Security was designed using the "Principle of Least Privilege"[cite: 11]. [cite_start]Each architectural layer only accepts communication from strictly required sources[cite: 12].

| Security Group Name | Assigned To | Inbound Rules | Purpose |
| :--- | :--- | :--- | :--- |
| **ALB-SG** | Load Balancer | `HTTP (80)` & `HTTPS (443)` from `0.0.0.0/0` | [cite_start]Allow global internet access to the website[cite: 13]. |
| **NAT-SG** | NAT Instance | `All Traffic` from `10.0.0.0/16` (VPC CIDR)<br>`SSH (22)` from `0.0.0.0/0` | [cite_start]Route internet traffic for private subnets and act as a bastion/jump server for SSH access[cite: 13]. |
| **App-SG** | ASG (EC2 Machines) | `Custom TCP (8080)` from **ALB-SG**<br>`SSH (22)` from **NAT-SG** | [cite_start]Accept traffic ONLY from the Load Balancer, and allow SSH debugging via the NAT instance[cite: 13]. |
| **RDS-SG** | RDS Database | `MySQL (3306)` from **App-SG** | Ensure the database strictly listens to Application Servers. [cite_start]This is the most secure layer[cite: 13]. |

---

## Route Tables: Network Routing Logic
[cite_start]While Security Groups act as guards, Route Tables provide the map for data packet routing[cite: 15].

**A. [cite_start]Public Route Table (Assigned to Public Subnets - ALB & NAT)** [cite: 16]
* **Target: `local` | [cite_start]Destination: `10.0.0.0/16`** -> Enables internal VPC machine-to-machine communication[cite: 17].
* **Target: `igw-xxxx` | [cite_start]Destination: `0.0.0.0/0`** -> Routes all internet-bound traffic out through the Internet Gateway[cite: 18].

**B. [cite_start]Private Route Table (Assigned to Private Subnets - ASG & RDS)** [cite: 19]
* **Target: `local` | [cite_start]Destination: `10.0.0.0/16`** -> Connects App servers and the Database directly[cite: 20].
* **Target: `eni-xxxx` (NAT Instance) | [cite_start]Destination: `0.0.0.0/0`** -> Routes private internet traffic through the custom NAT instance[cite: 21].

---

## Custom NAT Routing Configuration
To make the custom NAT instance function as a router, IP forwarding and masquerading were explicitly configured at the OS level:
```bash
# Verify network interfaces
ip a  

# Enable IP Forwarding in the kernel
sudo sysctl -w net.ipv4.ip_forward=1  

# Verify IP Forwarding is active
cat /proc/sys/net/ipv4/ip_forward  

# Configure IP Tables for Masquerading and Forwarding
sudo iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE  
sudo iptables -I FORWARD 1 -j ACCEPT  

# Make IPTables persistent across reboots
sudo apt update  
sudo apt install iptables-persistent -y  

# Monitor ICMP traffic for testing connectivity
sudo tcpdump -n -i any icmp