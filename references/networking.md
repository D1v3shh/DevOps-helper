# Networking & Security Reference
 
## Table of Contents
1. Core Concepts
2. DNS
3. TLS / SSL & Certificates
4. Load Balancers & Reverse Proxies
5. Firewalls & Security Groups
6. Kubernetes Networking (Ingress, NetworkPolicy)
7. VPNs & Private Connectivity
8. Diagnostic Commands
9. File Structure
---
 
## 1. Core Concepts
 
```
Client ──DNS lookup──> IP Address ──TCP/TLS handshake──> Server
                                          ↓
                              Load Balancer / Reverse Proxy
                                          ↓
                                  App servers (private subnet)
```
 
**OSI layers DevOps engineers touch most:**
| Layer | Examples |
|---|---|
| L7 (Application) | HTTP, HTTPS, gRPC — ingress, API gateways |
| L4 (Transport) | TCP, UDP — load balancers, security groups |
| L3 (Network) | IP routing — VPCs, subnets, route tables |
 
---
 
## 2. DNS
 
```
Browser → DNS Resolver → Root servers → TLD servers (.com) → Authoritative server → IP
```
 
**Record types:**
| Type | Purpose | Example |
|---|---|---|
| A | Hostname → IPv4 | `app.example.com → 203.0.113.10` |
| AAAA | Hostname → IPv6 | `app.example.com → 2001:db8::1` |
| CNAME | Alias to another hostname | `www → app.example.com` |
| MX | Mail server | `example.com → mail.example.com` |
| TXT | Verification / SPF / DKIM | `"v=spf1 include:_spf.google.com ~all"` |
| NS | Delegates a zone to nameservers | `example.com → ns1.cloudflare.com` |
 
```bash
# Diagnostics
dig example.com A              # Query A record
dig example.com +trace         # Full resolution path from root
nslookup example.com
host example.com
 
# TTL matters: lower TTL = faster propagation but more DNS queries
# Typical TTL: 300s (testing) to 86400s (stable production)
```
 
```hcl
# Terraform — Route 53 example
resource "aws_route53_record" "app" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"
 
  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}
```
 
---
 
## 3. TLS / SSL & Certificates
 
```
Client                          Server
  |── ClientHello ──────────────>|
  |<───── ServerHello + Cert ────|
  |── Verify cert, generate key ─>|
  |<──── Encrypted session ──────>|
```
 
**Getting certificates:**
 
```bash
# Let's Encrypt via certbot (manual / VM-based)
sudo certbot --nginx -d app.example.com -d www.app.example.com
sudo certbot renew --dry-run     # Test auto-renewal
```
 
```yaml
# cert-manager (Kubernetes) — auto-issues + renews certs
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: devops@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
```
 
```yaml
# Ingress requesting a cert automatically
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: ["app.example.com"]
      secretName: app-tls     # cert-manager creates this Secret
```
 
⚠️ **Common mistakes:**
- Letting certs expire (always automate renewal — certbot/cert-manager handle this)
- Using self-signed certs in production (browsers will warn users)
- Mixing HTTP and HTTPS content ("mixed content" errors)
```bash
# Check a cert's expiry from the command line
echo | openssl s_client -connect app.example.com:443 2>/dev/null | openssl x509 -noout -dates
```
 
---
 
## 4. Load Balancers & Reverse Proxies
 
```
                    ┌─> App server 1
Internet → LB ──────┼─> App server 2
                    └─> App server 3
        (health checks remove unhealthy targets automatically)
```
 
**Types:**
| Type | Layer | Use case |
|---|---|---|
| ALB (AWS) | L7 | HTTP routing by path/host, WebSocket support |
| NLB (AWS) | L4 | Raw TCP/UDP, extreme throughput, static IP |
| Nginx | L7 | Reverse proxy, caching, rate limiting |
| HAProxy | L4/L7 | High-performance TCP/HTTP load balancing |
 
```nginx
# nginx.conf — reverse proxy + load balancing
upstream app_servers {
    least_conn;                       # Load balancing strategy
    server 10.0.1.10:3000;
    server 10.0.1.11:3000;
    server 10.0.1.12:3000 backup;      # Only used if others are down
}
 
server {
    listen 443 ssl;
    server_name app.example.com;
 
    ssl_certificate     /etc/nginx/ssl/app.crt;
    ssl_certificate_key /etc/nginx/ssl/app.key;
 
    location / {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
 
    location /health {
        access_log off;
        return 200 "OK";
    }
}
```
 
---
 
## 5. Firewalls & Security Groups
 
```
Internet
   │
   ▼
[Security Group: web-sg]   allow 80, 443 from 0.0.0.0/0
   │
   ▼
[App Server]
   │
   ▼
[Security Group: db-sg]    allow 5432 ONLY from web-sg
   │
   ▼
[Database]   ── never exposed to the internet directly
```
 
```hcl
# Terraform — AWS Security Group
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
 
  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
 
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"             # All protocols
    cidr_blocks = ["0.0.0.0/0"]
  }
}
 
resource "aws_security_group" "db" {
  name   = "db-sg"
  vpc_id = aws_vpc.main.id
 
  ingress {
    description     = "Postgres from web tier only"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]   # Not a CIDR — references the SG
  }
}
```
 
⚠️ **Principle of least privilege:** never open `0.0.0.0/0` on database or SSH ports.
For SSH, prefer a bastion host or AWS Systems Manager Session Manager (no open port 22).
 
---
 
## 6. Kubernetes Networking
 
```yaml
# NetworkPolicy — deny-by-default, allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-from-frontend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```
 
```
Pod-to-pod traffic flow inside a cluster:
Frontend Pod ──> kube-proxy / CNI ──> Service (ClusterIP) ──> Backend Pod
                                              ↑
                                  Load-balances across all matching pod replicas
```
 
See `references/kubernetes.md` for Ingress and Service examples in full.
 
---
 
## 7. VPNs & Private Connectivity
 
| Option | Use case |
|---|---|
| Site-to-Site VPN | Connect on-prem network to cloud VPC |
| Client VPN | Individual developer access to private resources |
| VPC Peering | Connect two VPCs directly (same cloud) |
| Transit Gateway | Hub-and-spoke connectivity for many VPCs |
| PrivateLink / Private Service Connect | Access a service without traversing the public internet |
 
```
On-prem network (10.1.0.0/16) <══VPN tunnel══> AWS VPC (10.0.0.0/16)
                                                      │
                                            Private subnets only
```
 
---
 
## 8. Diagnostic Commands
 
```bash
# Connectivity
ping app.example.com
curl -v https://app.example.com/health
telnet db.example.com 5432              # Test raw TCP connectivity
 
# Trace the path
traceroute app.example.com
mtr app.example.com                     # Continuous traceroute + ping stats
 
# Ports
ss -tlnp                                # What's listening locally
nc -zv app.example.com 443              # Is the port reachable?
 
# TLS debugging
openssl s_client -connect app.example.com:443 -servername app.example.com
 
# Kubernetes networking debug
kubectl run debug --rm -it --image=nicolaka/netshoot -- /bin/bash
# Inside: curl, dig, nslookup, traceroute all available
```
 
---
 
## 9. File Structure
 
```
networking/
├── nginx/
│   ├── nginx.conf              # Main reverse proxy config
│   └── conf.d/
│       └── app.conf            # Per-site config
├── certs/
│   └── .gitkeep                # Never commit real certs/keys
└── terraform/
    ├── vpc.tf                  # VPC, subnets, route tables
    ├── security-groups.tf      # Security group rules
    └── route53.tf              # DNS records
 
k8s/
├── base/
│   ├── ingress.yaml
│   └── network-policy.yaml     # Pod-to-pod traffic rules
```
