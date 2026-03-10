Architecture Overview

Key Components:

Layer 1 (CloudFront + WAF) – Public Edge

CloudFront distribution redirects HTTP → HTTPS
AWS WAFv2 Web ACL filters traffic with managed rules, rate limiting, and custom WebSocket rules
Injects custom header X-Origin-Verify: <secret> before forwarding to origin
Layer 2 (Security Group) – Network Isolation

EC2 security group allows only port 443 from CloudFront managed prefix list
SSH optionally allowed from specified CIDRs
Port 18789 (OpenClaw) never exposed externally
Layer 3 (Nginx) – TLS & Header Validation

Terminates TLS with Let's Encrypt certificates
Validates X-Origin-Verify header; returns 403 if missing or incorrect
Proxies valid WebSocket traffic to internal gateway on :18789
Sets security headers (HSTS, X-Frame-Options, etc.)
Layer 4 (OpenClaw Gateway) – Application Auth

Runs on internal Docker bridge, never publicly accessible
Token-based authentication from env variable GATEWAY_TOKEN
Exposes health check endpoint at /healthz
Support Services

Certbot Container – Automates Let's Encrypt certificate renewal via DNS-01 against SaaS Route53
SaaS Control Plane – Manages infrastructure via cross-account STS roles, stores secrets, manages DNS records
Security Flow:

User → HTTPS to CloudFront → WAF checks & rate limits → Injects secret header → EC2 SG allows only from CF IPs → Nginx validates header → Gateway validates token → User authenticated
This creates a defense-in-depth approach with four independent security layers.

