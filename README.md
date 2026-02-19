# Multi-Cloud HA Failover (AWS → Azure DR)

## Overview

This project implements a **DNS-based active-passive multi-cloud failover architecture** using:

- **AWS** as Primary
- **Azure** as Disaster Recovery (DR)
- **Route 53 failover routing policy**
- **Dynamic instance metadata injection**

The application dynamically displays:

```
Instance: <Instance-ID> | Region: <Region>
```

This confirms runtime execution in different cloud environments.

---

#  Architecture

## Normal Operation

Client  
↓  
Route 53 (Primary Record)  
↓  
AWS Application Load Balancer  
↓  
EC2 Instance  

## Failover Flow

If AWS health check fails:

Client  
↓  
Route 53 detects unhealthy endpoint  
↓  
DNS switches to Azure Public IP  
↓  
Azure VM serves traffic  

Failover occurs automatically without manual intervention.

---

# Design Decisions

### Why DNS-Level Failover?

- Cloud-agnostic
- Simple to implement
- No cross-cloud networking required
- Cost-effective

### Tradeoffs

- DNS propagation delay
- Not instant failover
- Active-passive only (not active-active)

---

# AWS Primary Implementation

## Components

- VPC
- Internet Gateway
- Security Group
- EC2 Instance
- Application Load Balancer
- Route 53 Health Check

---

## EC2 User Data Script

```bash
#!/bin/bash
set -e

yum update -y
yum install -y nginx git

systemctl enable nginx
systemctl start nginx

rm -rf /usr/share/nginx/html/*

git clone https://github.com/<your-username>/multi-cloud-ha-failover-observability.git /tmp/app
cp -r /tmp/app/site/* /usr/share/nginx/html/

INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)

sed -i "s/CLOUD_NAME/AWS PRIMARY/" /usr/share/nginx/html/index.html
echo "<p style='font-size:22px;'>Instance: $INSTANCE_ID | Region: $REGION</p>" >> /usr/share/nginx/html/index.html

systemctl restart nginx
```

---

## AWS Metadata Endpoint

```bash
curl http://169.254.169.254/latest/meta-data/instance-id
curl http://169.254.169.254/latest/meta-data/placement/region
```

---

# Azure DR Implementation

## Components

- Azure Virtual Network
- Network Security Group
- Azure Virtual Machine
- Public IP
- Custom Data (cloud-init)

---

## Azure Cloud-Init Script

```bash
#!/bin/bash
set -e

apt-get update -y
apt-get install -y nginx git jq

systemctl enable nginx
systemctl start nginx

rm -rf /var/www/html/*

git clone https://github.com/<your-username>/multi-cloud-ha-failover-observability.git /tmp/app
cp -r /tmp/app/site/* /var/www/html/

AZ_JSON=$(curl -s -H "Metadata:true" \
"http://169.254.169.254/metadata/instance?api-version=2021-02-01")

VM_ID=$(echo $AZ_JSON | jq -r '.compute.vmId')
REGION=$(echo $AZ_JSON | jq -r '.compute.location')

sed -i "s/CLOUD_NAME/AZURE DR/" /var/www/html/index.html
echo "<p style='font-size:22px;'>Instance: $VM_ID | Region: $REGION</p>" >> /var/www/html/index.html

systemctl restart nginx
```

---

## Azure Metadata Endpoint

```bash
curl -H "Metadata:true" \
"http://169.254.169.254/metadata/instance?api-version=2021-02-01"
```

---

# Failover Validation Steps

1. Deploy AWS EC2 + ALB
2. Deploy Azure VM
3. Configure Route 53 failover routing
4. Access primary domain
5. Stop AWS EC2 instance
6. Wait for health check failure
7. Refresh domain → Azure DR should serve traffic

---

# 📊 Runtime Output

### AWS

```
AWS PRIMARY
Instance: i-xxxxxxxxxxxxxxxxx | Region: us-east-1
```

### Azure

```
AZURE DR
Instance: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx | Region: eastus
```

Metadata is dynamically injected at runtime via cloud metadata services.

---

# Key Learnings

- Cloud metadata services differ (AWS vs Azure)
- Bootstrapping via User Data vs Cloud-Init
- DNS-based failover patterns
- Active-passive disaster recovery model
- Infrastructure automation fundamentals

---

# Future Improvements

- Terraform (Infrastructure as Code)
- Docker containerization
- CI/CD pipeline
- Auto Scaling Group
- Centralized monitoring (CloudWatch / Azure Monitor / Grafana)

---

# Author

Ajith Manoharan  
Cloud Engineer | AWS | Azure | DevOps | SRE
