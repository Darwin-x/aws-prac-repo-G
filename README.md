# Project-01: AWS 3-Tier Architecture — Progress Report

**Environment:** `stage-env-1-vpc` | Region: `ap-south-1` (Mumbai)

---

## Task 1 — Networking Infrastructure ✅ Complete

### VPC
| Resource | ID | CIDR |
|---|---|---|
| VPC | `vpc-0e025e43ada8d80e6` (`stage-env-1-vpc`) | 10.0.0.0/16 |

DNS hostnames and DNS resolution both enabled.

### Subnets (6, across 2 AZs)

| Subnet | AZ | CIDR | Tier |
|---|---|---|---|
| `stage-env-1-subnet-public1-ap-south-1a` | ap-south-1a | 10.0.0.0/20 | Web (public) |
| `stage-env-1-subnet-public2-ap-south-1b` | ap-south-1b | 10.0.16.0/20 | Web (public) |
| `stage-env-1-subnet-private1-ap-south-1a` | ap-south-1a | 10.0.128.0/20 | Application |
| `stage-env-1-subnet-private2-ap-south-1b` | ap-south-1b | 10.0.144.0/20 | Application |
| `stage-env-1-subnet-private3-ap-south-1a` | ap-south-1a | 10.0.160.0/20 | Database |
| `stage-env-1-subnet-private4-ap-south-1b` | ap-south-1b | 10.0.176.0/20 | Database |

### Internet Gateway
| Resource | ID |
|---|---|
| Internet Gateway | `igw-090320e8c2d2c4485` |

Attached to `stage-env-1-vpc`.

### NAT Gateway
| Resource | ID |
|---|---|
| Elastic IP | `eipalloc-017a74c2fea0457f5` |
| NAT Gateway | `nat-0c3473f188e2b32d0` |

Deployed in a public subnet; provides outbound internet for private-subnet instances.

### Route Tables

| Route Table | Associated Subnet(s) | Default Route |
|---|---|---|
| `stage-env-1-rtb-public` | Both public subnets | `0.0.0.0/0` → IGW |
| `stage-env-1-rtb-private1-ap-south-1a` | private1 (App, AZ-a) | `0.0.0.0/0` → NAT Gateway |
| `stage-env-1-rtb-private2-ap-south-1b` | private2 (App, AZ-b) | `0.0.0.0/0` → NAT Gateway |
| `stage-env-1-rtb-private3-ap-south-1a` | private3 (DB, AZ-a) | `0.0.0.0/0` → NAT Gateway |
| `stage-env-1-rtb-private4-ap-south-1b` | private4 (DB, AZ-b) | `0.0.0.0/0` → NAT Gateway |

> Note: created as 5 route tables (1 per public group + 1 per private subnet) via the VPC console wizard, rather than 3 tier-level tables. Functionally equivalent — each table carries the correct route for its subnet's tier — with the added benefit of per-subnet routing isolation.

### Bonus: S3 Gateway Endpoint
| Resource | ID |
|---|---|
| VPC Endpoint (S3, Gateway type) | `vpce-0095ce72a91b049db` |

Associated with private-subnet route tables — lets private instances reach S3 without traversing the NAT Gateway.

---

## Task 2 — Web Tier (Presentation Layer) ✅ Complete

- [x] Security group `web-sg` created — inbound 80 (from `alb-sg`, tightened in Task 5), 22 (restricted to admin IP); outbound all
- [x] 2 EC2 instances launched, one per public subnet (AZ-a and AZ-b), auto-assign public IP enabled
- [x] Nginx installed and running on **both** instances (`apt install nginx`, Ubuntu-based AMI)
- [x] Project HTML files cloned from repo and deployed to Nginx web root on both instances
- [x] Verified locally via `curl -I http://localhost` on both instances → `200 OK`

**Resolved issue:** `ec2-2` (`i-0eaa9ba31a55a4ecc`) initially failed both browser access and ALB health checks. Root cause: Nginx was never installed on that instance — an earlier "it works" curl test had actually been run against `ec2-1` by mistake (similar SSH sessions, easy mix-up). Confirmed via mismatched private IP (`ip-10-0-12-231` vs `ip-10-0-28-232`). Fixed by installing Nginx + redeploying HTML on `ec2-2`; SGs and NACLs were checked and confirmed correctly configured throughout — not the cause.

---

## Task 3 — Application Tier (Backend API) ✅ Complete

- [x] Subnets confirmed: `private1-ap-south-1a`, `private2-ap-south-1b`
- [x] Security group `app-sg` created — inbound 5000 from `web-sg` only; no SSH rule
- [x] IAM role with `AmazonSSMManagedInstanceCore` attached for Session Manager access
- [x] 2 EC2 instances launched in the App subnets, no public IP
- [x] Connected via Session Manager (no direct SSH)
- [x] Python venv created per instance; Flask + PyMySQL installed (`pip install flask pymysql`)
- [x] Repo (`Three-Tier-Architecture`) cloned on both instances
- [x] `app.py` running on both instances, bound and reachable on port 5000
- [x] Verified via `curl http://localhost:5000/` on both instances → `{"status": "Flask is running"}`

**Note:** `DB_CONFIG` in `app.py` still holds placeholder values (`RDS-ENDPOINT`, dummy password) — intentionally left untouched until Task 9.

---

## Task 4 — Database Tier (RDS MySQL) 🟡 Recreating

RDS instance deleted earlier for cost control, now being recreated with identical settings:
- [x] DB Subnet Group reused: `stage-env-db-subnet-1` (private3-ap-south-1a, private4-ap-south-1b)
- [x] Security group reused: `db-sg` (inbound 3306 from `app-sg`) — chosen via "Choose existing," not recreated
- [x] Public access: **No**
- [ ] Instance provisioning (check console for "Available" status)
- [ ] `feedback` table not yet recreated — run after instance is Available:
  ```sql
  CREATE TABLE feedback (
      id INT AUTO_INCREMENT PRIMARY KEY,
      name VARCHAR(100),
      email VARCHAR(100),
      feedback TEXT
  );
  ```
- [ ] Note the new endpoint (will differ from the old `database-1.c1ic6ymsg8md...` one even if instance identifier is reused)

---

## Security Groups Summary (built incrementally across Tasks 2–4)

| SG | Inbound Allowed | Source |
|---|---|---|
| `web-sg` | 22, 80 | Admin IP (22), 0.0.0.0/0 (80) |
| `app-sg` | 5000 | `web-sg` |
| `db-sg` | 3306 | `app-sg` |

---

## Task 5 — Security Groups ✅ Complete
- [x] `alb-sg` created — inbound 80 from `0.0.0.0/0`
- [x] `web-sg` tightened — inbound 80 now sourced from `alb-sg` (no longer open to raw internet); SSH still restricted to admin IP
- [x] `app-sg`, `db-sg` unchanged from Tasks 3–4, confirmed correct

## Task 6 — Internet-Facing Application Load Balancer ✅ Complete
- [x] Target group `stage-env-TG-name` created, port 80, health check path `/`
- [x] Both Web-tier instances registered
- [x] ALB `Stage-LB` created — internet-facing, attached to both public subnets
- [x] Both targets showing **Healthy** (`stage-env-ec2-1` in ap-south-1a, `stage-env-ec2-2` in ap-south-1b)
- [x] Verified via `curl -I http://<alb-dns-name>` → `200 OK`

## Task 7 — Internal Application Load Balancer ✅ Complete
- [x] Target group `stage-env-TG-app` created, port 5000, health check path `/`, both App instances registered
- [x] `internal-alb-sg` created — inbound 80/5000 from `web-sg`; outbound all
- [x] `app-sg` updated — inbound 5000 now sourced from `internal-alb-sg` (was `web-sg`)
- [x] Internal ALB `stage-env-alb-internal` created — scheme Internal, attached to both private App subnets
- [x] Listener HTTP:80 → forwards to `stage-env-TG-app` (confirmed 100% weight)

**Resolved issue:** both App instances showed unhealthy despite correct SG/listener/health-check config. Root cause: the Flask process (`python3 app.py`, run in foreground) had died when the originating Session Manager session ended — not a networking or SG issue at all. Compounded by instance-specific root-owned files and a stale venv living in an ephemeral `/var/snap/amazon-ssm-agent/<session-id>/` path on one node. Fixed by: moving the repo to `/opt/Three-Tier-Architecture` on both instances (fixing ownership to `ssm-user`), recreating a clean venv there, and backgrounding Flask with `nohup python3 app.py > flask.log 2>&1 & disown` so it survives SSM session disconnects. Verified via a fresh reconnect + `curl localhost:5000` on both nodes post-disconnect.

**Note for Task 9/10 polish:** `nohup` is sufficient for this project but not production-grade — a `systemd` service would auto-restart Flask on crash. Worth adding as a final touch once validation (Task 10) passes.

## Task 8 — Nginx Reverse Proxy (API Routing) ✅ Complete
- [x] Repo's `nginx.conf` copied over the default `/etc/nginx/nginx.conf` on both Web instances (default config was never replaced during Task 2 — only HTML was deployed then)
- [x] `proxy_pass` line updated with real Internal ALB DNS (`internal-stage-env-alb-internal-505168705.ap-south-1.elb.amazonaws.com`) on both nodes
- [x] `nginx -t` passes, `systemctl restart nginx` successful on both
- [x] `curl http://localhost/api/` returns a live Flask response on both Web instances

**Resolved issue:** first edit attempt dropped the required `http://` scheme and trailing `/` from the `proxy_pass` value (manual `vim` edit), causing `nginx -t` to fail with "invalid URL prefix." Fixed by correcting the line to the full form: `proxy_pass http://<internal-alb-dns>/;`

## Not Yet Started

### Task 9 — Integrate Flask with RDS
Remaining steps:
1. On **both** App instances (`/opt/Three-Tier-Architecture/app.py`), edit `DB_CONFIG` with the real RDS endpoint + password set during creation
2. Kill the existing backgrounded Flask process and restart it the same way as before:
   ```bash
   pkill -f app.py
   cd /opt/Three-Tier-Architecture
   . venv/bin/activate
   bash
   nohup python3 app.py > flask.log 2>&1 &
   disown
   curl http://localhost:5000/
   ```
3. Test via **Public ALB DNS** in browser → submit feedback form → expect success, not "Error submitting"
4. Confirm row landed: `mysql -h <rds-endpoint> -u admin -p -e "SELECT * FROM feedback;" feedbackdb`

### Task 10 — Validate the Complete Architecture
- [ ] Public ALB DNS loads the page
- [ ] Feedback form submits successfully end-to-end
- [ ] Row confirmed in RDS via SQL query
- [ ] Direct internet access to App/DB tier still fails (confirms isolation held throughout)

### Bonus — Auto Scaling (optional)
- [ ] Launch template from Web-tier AMI (Nginx + HTML + proxy config pre-baked, or via user_data)
- [ ] Auto Scaling Group attached to `Stage-LB`'s target group (`stage-env-TG-name`)
- [ ] Target tracking on CPU, min 2 / desired 2 / max 4
- [ ] Verify: terminate one instance, confirm ASG replaces it and it re-registers healthy

---

## Key Resource Reference (for quick resume)

| Resource | Value |
|---|---|
| VPC | `vpc-0e025e43ada8d80e6` |
| Public ALB | `Stage-LB` |
| Internal ALB | `stage-env-alb-internal` — DNS: `internal-stage-env-alb-internal-505168705.ap-south-1.elb.amazonaws.com` |
| Web instances | public subnets; Nginx + reverse proxy config in place |
| App instances | `/opt/Three-Tier-Architecture`; Flask backgrounded via `nohup`, port 5000 |
| DB subnet group | `stage-env-db-subnet-1` |
| DB name / user | `feedbackdb` / `admin` |
| Security groups | `web-sg`, `app-sg`, `db-sg`, `alb-sg`, `internal-alb-sg` — all correctly chained |

---

*Report last updated during Task 4 (RDS recreation) — Tasks 1–3, 5–8 fully complete; Task 4 in progress; Tasks 9–10 and bonus remaining.*
