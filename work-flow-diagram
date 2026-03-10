┌───────────────────────────┐
│        Viewer/User        │
│        Browser UI         │
└─────────────┬─────────────┘
              │ HTTPS
              ▼
┌─────────────────────────────────────────┐
│        Amazon CloudFront (CDN)          │
│  - Public entry point                   │
│  - WAF protection (Layer 1)             │
│  - Rate limiting                        │
│  - Custom header injection              │
│  - WebSocket support                    │
│                                         │
│  Viewer URL                             │
│  d1234abcd.cloudfront.net               │
└─────────────┬───────────────────────────┘
              │ HTTPS (Origin request)
              │ Header: X-Origin-Verify
              ▼
┌───────────────────────────────────────────────────────┐
│                 Customer AWS Account                  │
│                                                       │
│   ┌───────────────────────────────────────────────┐   │
│   │                 EC2 Instance                  │   │
│   │                                               │   │
│   │  Security Group                               │   │
│   │  Allow: 443 from CloudFront only              │   │
│   │                                               │   │
│   │   ┌───────────────────────────────────────┐   │   │
│   │   │        Nginx Container                │   │   │
│   │   │                                       │   │   │
│   │   │ TLS termination (443)                 │   │   │
│   │   │ Verify header: X-Origin-Verify        │   │   │
│   │   │ WebSocket reverse proxy               │   │   │
│   │   └───────────────┬───────────────────────┘   │   │
│   │                   │ HTTP                      │   │
│   │                   ▼                           │   │
│   │   ┌───────────────────────────────────────┐   │   │
│   │   │      OpenClaw Gateway Container       │   │   │
│   │   │                                       │   │   │
│   │   │ Local API server                      │   │   │
│   │   │ Token authentication                  │   │   │
│   │   │ Port 18789 (internal only)            │   │   │
│   │   └───────────────────────────────────────┘   │   │
│   │                                               │   │
│   │   ┌───────────────────────────────────────┐   │   │
│   │   │       Certbot Container               │   │   │
│   │   │                                       │   │   │
│   │   │ Let's Encrypt certificate automation  │   │   │
│   │   │ DNS-01 challenge via Route53          │   │   │
│   │   └───────────────────────────────────────┘   │   │
│   │                                               │   │
│   └───────────────────────────────────────────────┘   │
│                                                       │
└───────────────────────────────────────────────────────┘

        ▲
        │
        │ Cross-Account AssumeRole
        │
┌─────────────────────────────────────────────┐
│           OpenClaw SaaS Backend             │
│                                             │
│ Control Plane                               │
│                                             │
│ Responsibilities:                           │
│ - Provision customer infrastructure         │
│ - Manage EC2 / SG / CloudFront / WAF        │
│ - Store secrets                             │
│ - Rotate tokens                             │
│ - Trigger upgrades                          │
│ - Manage Route53 origin domain              │
│                                             │
└─────────────────────────────────────────────┘



 ## Deployment Workflow (End-to-End)

This section describes the full lifecycle of deploying an OpenClaw tenant environment from onboarding to a running system.

---

### 1. Customer Onboarding

The customer provides the following information:

- AWS Account ID
- Cross-account IAM Role ARN
- Deployment region
- Tenant slug (unique identifier)

Example:

tenantSlug=acme

The tenant slug is used to generate infrastructure resource names and the origin domain:

acme-origin.openclaw-saas.io

---

### 2. SaaS Control Plane Assumes Customer Role

The OpenClaw SaaS backend assumes the customer IAM role using AWS STS with an ExternalId.

This allows the SaaS platform to provision infrastructure in the customer account without storing credentials.

Flow:

SaaS Backend → STS AssumeRole → Customer AWS Account

---

### 3. Infrastructure Provisioning

The SaaS backend provisions required infrastructure in the customer AWS account.

Resources created:

- VPC
- Security Group
- EC2 instance
- EC2 IAM role and instance profile
- Secrets Manager entries

Naming convention:

openclaw-<stage>-<resource>

Example:

openclaw-prod-ec2  
openclaw-prod-gateway-token

---

### 4. EC2 Instance Launch

An Ubuntu EC2 instance is launched with the following configuration:

- Instance type: t3.small (minimum)
- Root volume encryption enabled
- IMDSv2 required
- Security group allowing port 443 only from CloudFront

Port 18789 (OpenClaw gateway) is not exposed publicly.

---

### 5. EC2 Bootstrap

When the EC2 instance starts, a bootstrap script runs via user-data.

The script performs the following:

- Installs Docker and Docker Compose
- Pulls secrets from AWS Secrets Manager
- Creates the OpenClaw runtime directories
- Writes environment configuration
- Generates docker-compose configuration

Directory layout:

/opt/openclaw  
/opt/openclaw/nginx  
/opt/openclaw/certbot  
/home/ubuntu/openclaw-data

---

### 6. Container Startup

Docker Compose launches three services:

openclaw-gateway  
nginx  
certbot

OpenClaw Gateway runs internally on:

127.0.0.1:18789

Nginx exposes HTTPS on:

443

Nginx proxies traffic to the OpenClaw gateway.

---

### 7. TLS Certificate Issuance

Certbot obtains a TLS certificate from Let's Encrypt using DNS-01 validation.

Domain issued:

<tenantSlug>-origin.openclaw-saas.io

Certificates are stored locally and mounted into the Nginx container.

Certificates automatically renew twice per day via cron.

---

### 8. CloudFront Distribution Setup

A CloudFront distribution is created to serve as the public entry point.

Viewer URL:

https://d1234abcd.cloudfront.net

Origin:

<tenantSlug>-origin.openclaw-saas.io

Configuration:

- HTTPS only
- HTTP → HTTPS redirect
- WebSocket support
- Caching disabled

CloudFront injects a custom header to protect the origin:

X-Origin-Verify: <origin-secret>

---

### 9. AWS WAF Configuration

A WAFv2 Web ACL is created and attached to the CloudFront distribution.

Enabled rules include:

- AWS Managed Common Rule Set
- Known Bad Inputs
- IP Reputation List
- Anonymous IP protection
- Rate limiting

---

### 10. Deployment Validation

After provisioning, the deployment is validated.

Checks performed:

- CloudFront status is "Deployed"
- WAF is attached to the distribution
- EC2 instance is running
- Containers are healthy

Gateway health check:

curl http://127.0.0.1:18789/healthz

Expected response:

HTTP 200

External verification:

https://d1234abcd.cloudfront.net

WebSocket connectivity is also tested.

---

### 11. Runtime Request Flow

Once deployed, requests follow this path:

User Browser  
→ CloudFront + WAF  
→ Nginx (origin verification)  
→ OpenClaw Gateway  
→ Application response

The gateway authenticates requests using the configured token.

---

### 12. Deployment Complete

The application is now available via CloudFront:

https://d1234abcd.cloudfront.net

Customers can optionally map their own domain:

app.customer-domain.com → CloudFront distribution
