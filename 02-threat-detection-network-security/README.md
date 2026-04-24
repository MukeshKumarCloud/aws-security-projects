# AWS: Threat Detection & Network Security with AWS Network Firewall + WAF

## Overview

Built a layered threat detection and web application defense stack: configured Suricata-compatible stateful firewall rules and DNS blocklists to detect and isolate rogue EC2 instances, and deployed WAF web ACLs with IP filtering, rate limiting, and managed rule sets to block SQLi and unauthorized API access — verified end-to-end via CloudWatch and Artillery load testing.

---

## Table of Contents

- [Part 1 — AWS Network Firewall & Threat Detection](#part-1--aws-network-firewall--threat-detection)
  - [Architecture](#architecture)
  - [Services Used](#services-used)
  - [Task 1: Explore the Network Architecture](#task-1-explore-the-network-architecture)
  - [Task 2: Stateful Firewall Rules](#task-2-stateful-firewall-rules)
  - [Task 3: Route 53 Resolver DNS Firewall](#task-3-route-53-resolver-dns-firewall)
  - [Task 4: Threat Hunting with CloudWatch](#task-4-threat-hunting-with-cloudwatch)
  - [Task 5: DNS Exfiltration Detection](#task-5-dns-exfiltration-detection)
  - [Task 6: Quarantine Rogue Instances](#task-6-quarantine-rogue-instances)
- [Part 2 — AWS WAF Web Application Protection](#part-2--aws-waf-web-application-protection)
  - [Architecture](#architecture-1)
  - [Services Used](#services-used-1)
  - [Task 1: Block API Requests with a WAF Web ACL](#task-1-block-api-requests-with-a-waf-web-acl)
  - [Task 2: Create and Test WAF Web ACL Rules](#task-2-create-and-test-waf-web-acl-rules)
  - [Task 3: Block SQL Injection Attacks](#task-3-block-sql-injection-attacks)
  - [Task 4: Monitor API Traffic with CloudWatch](#task-4-monitor-api-traffic-with-cloudwatch)
- [Conclusions](#conclusions)

---

## Part 1 — AWS Network Firewall & Threat Detection

### Objectives

- Configure stateful rule groups in AWS Network Firewall using Suricata-compatible IPS rule specifications.
- Use a combination of managed and custom DNS domain lists to create a DNS Firewall that alerts administrators to suspicious queries.
- Use Log Insights and Contributor Insights in Amazon CloudWatch to identify rogue EC2 instances.

### Architecture

![Architecture Diagram](./architecture-diagram.png)

> **Diagram description:** Data flows from EC2 instances through Route 53 Resolver DNS Firewalls to an AWS Transit Gateway, then to a separate VPC containing AWS Network Firewall endpoints, and ultimately out to the internet via an Internet Gateway. The major resources include two Spoke VPCs containing EC2 workloads, an Inspection-Egress-VPC containing AWS Network Firewall endpoints and a NAT Gateway, an AWS Transit Gateway connecting the three VPCs, and an Internet Gateway providing egress to the internet.

### Services Used

- Amazon Virtual Private Cloud (Amazon VPC)
- AWS Network Firewall
- Route 53 Resolver DNS Firewall
- Amazon CloudWatch
- Amazon EC2

---

### Task 1: Explore the Network Architecture

Verified network reachability from both Spoke VPCs to the internet using VPC Reachability Analyzer, confirming traffic correctly traverses the Transit Gateway, Firewall Endpoints, and NAT Gateway before reaching the Internet Gateway.

**Steps:**

1. Navigate to **VPC → Network Manager → Reachability Analyzer**.
2. Create and analyze a path:
   - **Name:** `Spoke A to Internet`
   - **Source type:** Instances → `Spoke-VPC-TestInstance1`
   - **Destination type:** Internet Gateways → `Inspection-Egress-VPC-IGW`
3. Repeat for:
   - **Name:** `Spoke B to Internet`
   - **Source:** `Spoke-VPC-TestInstance2`
   - **Destination:** `Inspection-Egress-VPC-IGW`

**Expected output:**

![Reachability Analyzer result](./snap-1.PNG)

> **Traffic path summary:** `Spoke-VPC-TestInstance1` → Security Group & NACL → Transit Gateway → Firewall Endpoint (vpce-) → InspectionFirewall → NAT Gateway → Internet Gateway.

---

### Task 2: Stateful Firewall Rules

#### Task 2.1: Domain Lists

Created a custom domain blocklist in AWS Network Firewall targeting low-reputation TLDs, restricting HTTP/HTTPS traffic from the `10.0.0.0/8` CIDR range.

**Steps:**

1. Navigate to **VPC → Network Firewall → Firewalls → InspectionFirewall → Firewall policy settings**.
2. Create a stateful rule group:
   - **Format:** Domain list
   - **Name:** `RestrictedDomains`
   - **Description:** `A domain list of low reputation domains`
   - **Capacity:** `100`
3. Configure domain rules:
   - **Domains:**
     ```
     .example.xyz
     .example.stream
     .example.party
     .example.click
     .example.win
     .example.download
     .example.bid
     .example.vip
     .example.net
     ```
   - **CIDR ranges:** Custom → `10.0.0.0/8`
   - **Protocols:** HTTP, HTTPS
   - **Action:** Deny

**Validation:**

```bash
curl https://www.example.net --max-time 5
```

Expected output:
```
curl: (28) Connection timed out after 5000 milliseconds
```

#### Task 2.2: Intrusion Prevention with Suricata Rules

Integrated a custom Suricata rule to detect non-TLS traffic on port 443 (a common indicator of protocol tunneling or evasion).

**Steps:**

1. Navigate to **VPC → Network Firewall → Network Firewall rule groups → Create rule group**.
2. Configure:
   - **Type:** Stateful rule group
   - **Format:** Suricata compatible rule string
   - **Evaluation order:** Action order
   - **Name:** `NonTlsTrafficOn443`
   - **Description:** `A rule that detects non-TLS traffic on port 443`
   - **Capacity:** `10`
3. Set IP variable:
   - **Key:** `HOME_NET` | **Value:** `10.0.0.0/8`
4. Enter Suricata rule:
   ```
   alert tcp any any <> any 443 (msg:"SURICATA Port 443 but not TLS"; flow:to_server,established; app-layer-protocol:!tls; sid:2271003; rev:1;)
   ```
   > **Rule breakdown:** Alerts on TCP port 443 traffic where the application-layer protocol is not TLS — targeting established server-bound connections. SID `2271003`, rev `1`.
5. Manually add `NonTlsTrafficOn443` to `InspectionFirewall-Policy` under **Stateful rule groups**.

#### Task 2.3: Managed Rule Groups

Added all 4 AWS managed Domain and IP rule groups plus the `ThreatSignaturesMalwareWebActionOrder` managed rule group to the `InspectionFirewall-Policy` for broad threat coverage at no additional cost.

---

### Task 3: Route 53 Resolver DNS Firewall

Extended domain blocking to all protocols (not just HTTP/HTTPS) by deploying a Route 53 Resolver DNS Firewall, blocking DNS resolution for the same low-reputation domains and an AWS-managed aggregate threat list.

**Steps:**

1. Navigate to **VPC → DNS Firewall → Domain lists → Add domain list**.
   - **Name:** `RestrictedDomains`
   - **Domains:**
     ```
     *.example.xyz
     *.example.stream
     *.example.party
     *.example.click
     *.example.win
     *.example.download
     *.example.bid
     *.example.vip
     *.example.net
     ```
2. Create a rule group:
   - **Name:** `Spoke-VPC-DNS-Firewall`
3. Add rule — **Custom blocklist:**
   - **Name:** `Blocklist`
   - **Domain list:** `RestrictedDomains`
   - **Action:** BLOCK → NXDOMAIN
4. Add rule — **Managed blocklist:**
   - **Name:** `ManagedBlocklist`
   - **Domain list:** `AWSManagedDomainsAggregateThreatList`
   - **Action:** BLOCK → NXDOMAIN
5. Associate rule group with **Spoke-VPC-A** and **Spoke-VPC-B**.
6. Configure Route 53 Resolver query logging:
   - **Name:** `Spoke-VPC-Query-Logs`
   - **Destination:** CloudWatch Logs → `/Lab/Route53/QueryLogs`
   - **VPCs:** Spoke-VPC-A, Spoke-VPC-B

---

### Task 4: Threat Hunting with CloudWatch

Used CloudWatch Log Insights and Contributor Insights to surface rogue EC2 instances generating anomalous traffic.

**Steps:**

1. Navigate to **CloudWatch → Logs → Contributor Insights → Create rule**.
   - **Log group:** `/Lab/Route53/QueryLogs`
   - **Rule type:** Custom rule (JSON)
   - **Contribution keys:** `srcids.instance`, `query_name`
   - **Filter:** `firewall_rule_action` In `BLOCK`
   - **Aggregation:** Count
   - **Rule name:** `Blocked-DNS-Queries`

2. Navigate to **CloudWatch → Logs → Log Insights**, select `/Lab/NetworkFirewall/Alert`.

**Query 1 — Check domain blocklist alerts:**

```
fields @timestamp, @message
| filter event.alert.signature like /denylisted FQDNs/
| display event.tls.sni, event.src_ip
```

![Log Insights - denylisted FQDNs](./snap-2.PNG)

**Query 2 — Detect ICMP anomalies (possible port scan):**

```
fields @timestamp, @message
| filter event.proto like /ICMP/
| display event.src_ip
| sort event.src_ip desc
```

![Log Insights - ICMP anomaly](./snap-3.PNG)

> **Finding:** An EC2 instance was sending high-frequency ICMP requests at regular intervals — consistent with a port scan from a compromised instance.

**Query 3 — Detect non-TLS / Suricata alerts:**

```
fields @timestamp, @message
| filter event.alert.signature like /SURICATA/
| display event.src_ip
| sort event.src_ip desc
```

![Log Insights - Suricata FTP on 443](./snap-4.png)

> **Finding:** An instance was using the FTP protocol on port 443 — a strong indicator of an attempt to bypass security controls using a non-standard port.

**Contributor Insights — Blocked DNS query visualization:**

- **Contributors:** Top 50
- **Period:** 1 Minute
- **Order by:** Max
- **Widget type:** Stacked area
- **Time range:** 30m
- **Refresh:** 10s

![Contributor Insights - Blocked DNS Queries](./snap-5.png)

---

### Task 5: DNS Exfiltration Detection

Analyzed DNS query logs from a suspected compromised instance to identify data exfiltration via DNS tunneling — where attackers encode stolen data in DNS subdomains.

**Steps:**

1. Connect to `Spoke-VPC-Command-Host` via **EC2 → Session Manager**.
2. Write the DNS query log to file:
   ```bash
   cd ~
   cat << 'EOF' > dnsqueries.txt
   IyBTYW1wbGUgUmVwb3J0IC0gTm8gaWRlbnRpZmljYXRpb24gb2YgYWN0dWFsIHB.example.xyz
   # ... (truncated — full payload in lab environment)
   EOF
   ```
3. Decode the exfiltrated data from subdomain labels:
   ```bash
   awk -F "." '{print $1}' dnsqueries.txt | base64 --decode
   ```

**Expected output (decoded):**

```
# Sample Report - No identification of actual persons
# or security credentials is intended or should be inferred.
Name,Access Key,Secret Access Key
Jorge Souza,AKIAIOSFODNN7EXAMPLE,wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Arnav Desai,AKIAI44QH8DHBEXAMPLE,je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY
...
```

> **Finding:** A compromised EC2 instance was exfiltrating a list of IAM Access Keys and Secret Access Keys encoded as base64 subdomains in DNS queries — a classic DNS data exfiltration technique.

---

### Task 6: Quarantine Rogue Instances

Isolated the three identified compromised EC2 instances by replacing their security groups with a deny-all `QuarantineSG`, containing the threat without destroying forensic evidence.

**Steps:**

For each of `Spoke-VPC-TestInstance1`, `Spoke-VPC-TestInstance2`, and `Spoke-VPC-TestInstance3`:

1. Navigate to **EC2 → Instances (running)**.
2. Select the instance → **Actions → Security → Change security groups**.
3. Remove the existing security group.
4. Search and select `QuarantineSG` → **Add security group → Save**.

---

## Part 2 — AWS WAF Web Application Protection

### Objectives

- Set up AWS WAF to protect API Gateway endpoints using Web ACLs and rules.
- Control API access using IP address filtering and rate limiting.
- Implement protection against common web attacks like SQL injection.
- Monitor and analyze web traffic using AWS WAF metrics and CloudWatch.

### Architecture

![WAF Architecture](./Architecture.png)

> **Diagram description:** Data flows from an external client's API request to a REST API, checked against the rules of a Web ACL. If the ACL allows the request, the API triggers a proxy Lambda function to a MySQL RDS table, parses the response, and sends the API response back to the client.

### Services Used

- AWS WAF
- Amazon API Gateway
- AWS Lambda
- Amazon RDS

---

### Task 1: Block API Requests with a WAF Web ACL

#### Task 1.1: Baseline API Test

Verified the API was accessible without any WAF protection in place.

```bash
curl https://$API_GATEWAY.execute-api.$AWS_REGION.amazonaws.com/Dev?id=1
```

Expected output:
```json
["Tennis Racket Pro Series","145 units","$45.99"]
```

#### Task 1.2: Create a Default-Block Web ACL

Created a WAF Web ACL with a **default Block** action and associated it with the API Gateway to deny all traffic by default before adding allow rules.

**Steps:**

1. Navigate to **WAF & Shield → Switch to old WAF console → Create web ACL**.
2. Configure:
   - **Resource type:** Regional
   - **Name:** `api-acl`
   - **Associated resource:** `democompany-api - Dev` (Amazon API Gateway REST API)
3. Set **Default action** to **Block**.
4. Enable **sampled requests** under metrics.
5. Create the Web ACL.

#### Task 1.3: Validate Default Block

```bash
curl https://$API_GATEWAY.execute-api.$AWS_REGION.amazonaws.com/Dev?id=1
```

Expected output:
```json
{"message":"Forbidden"}
```

![Default Block ACL](./snap11.png)

---

### Task 2: Create and Test WAF Web ACL Rules

#### Task 2.1: IP Allowlist — Partner IP Filtering

Created an IP set for authorized partner IPs, a rule group, and a Count rule that labels matching requests with `ipallowed`.

**Steps:**

1. Navigate to **WAF & Shield → IP sets → Create IP set**.
   - **Name:** `democompany_partners_ip_set`
   - **IP address:** `52.35.99.108/32`
2. Navigate to **Rule groups → Create rule group**.
   - **Name:** `core-rule-group`
3. Add rule:
   - **Name:** `count_partner_request`
   - **Type:** Regular rule
   - **Inspect:** Originates from an IP in `democompany_partners_ip_set`
   - **Action:** Count
   - **Label:** `ipallowed`
   - **Capacity:** `10`

#### Task 2.2: Rate Limiting — Block Over-Requesters

Added a rate-based rule to `core-rule-group` to block IPs exceeding 50 requests per minute, labeled `ratelimit`.

**Steps:**

1. Open `core-rule-group → Add rule`.
   - **Name:** `block_overrequested`
   - **Type:** Rate-based rule
   - **Rate limit:** `50`
   - **Evaluation window:** 1 minute
   - **Action:** Block
   - **Label:** `ratelimit`
2. Attach `core-rule-group` to `api-acl` under **Rules → Add rules → Add my own rules and rule groups**.

#### Task 2.3: Allow Rule for Valid Partner Requests

Added a compound allow rule evaluating both labels: allows requests tagged `ipallowed` AND NOT tagged `ratelimit`.

**Steps:**

1. On `api-acl → Add rules → Rule builder`.
   - **Name:** `allow_valid`
   - **Type:** Regular rule
   - **Condition:** matches ALL statements (AND)
   - **Statement 1:** Has label → `ipallowed`
   - **Statement 2:** NOT has label → `ratelimit`
   - **Action:** Allow
2. Ensure rule priority order: `core_rule_group` → `AWS-AWSManagedRulesSQLiRuleSet` → `allow_valid`.

#### Task 2.4: End-to-End Validation

**Test from non-partner IP (blocked):**

```bash
curl https://$API_GATEWAY.execute-api.$AWS_REGION.amazonaws.com/Dev?id=1
```

Expected output:
```json
{"message":"Forbidden"}
```

![Non-partner IP blocked](./snap-12.png)

**Test from partner IP (allowed):**

```bash
sudo su - ec2-user
curl https://$API_GATEWAY.execute-api.$AWS_REGION.amazonaws.com/Dev?id=1
```

Expected output:
```json
["Tennis Racket Pro Series","145 units","$45.99"]
```

![Partner IP allowed](./snap-13.png)

**Rate limit stress test using Artillery:**

```yaml
# test.yml
config:
  target: "https://$API_GATEWAY.execute-api.$AWS_REGION.amazonaws.com"
  phases:
    - duration: 60
      arrivalRate: 3
scenarios:
  - flow:
      - get:
          url: "/Dev?id=1"
```

```bash
artillery run test.yml
```

Expected output (abbreviated):

```
Metrics for period to: 17:22:40 (width: 9.957s)
http.codes.200: ........ 34

Metrics for period to: 17:23:00 (width: 9.803s)
http.codes.200: ........ 30
http.codes.403: ........ 4

Metrics for period to: 17:23:20 (width: 9.686s)
http.codes.403: ........ 30
```

![Rate limit enforcement](./snap-14.png)

> Initial requests succeed (200), then once the 50 req/min threshold is crossed, subsequent requests receive 403 Forbidden.

---

### Task 3: Block SQL Injection Attacks

#### Baseline SQLi Test (Pre-Rule)

Simulated a UNION-based SQL injection attack against the inventory API:

```bash
curl "https://$API_GATEWAY.execute-api.$AWS_REGION.amazonaws.com/Dev?id=1%20UNION%20SELECT%20profit%2Cwarehouse%2Csupplier%20FROM%20inventory%20WHERE%20id%3D1"
```

Expected output (pre-rule — data leak):
```json
[["Tennis Racket Pro Series",145,"45.99"],["34.01","East Coast","Example Supplier Inc."]]
```

#### Add AWS Managed SQLi Rule Group

1. Navigate to `api-acl → Rules → Add rules → Add managed rule groups`.
2. Under **AWS managed rule groups**, enable `SQL database` → **Add to web ACL**.
3. Set rule priority: `core-rule-group` → `AWS-AWSManagedRulesSQLiRuleSet` → `allow_valid`.

#### Post-Rule SQLi Validation

```bash
curl "https://$API_GATEWAY.execute-api.$AWS_REGION.amazonaws.com/Dev?id=1%20UNION%20SELECT%20profit%2CNULL%2Cwarehouse%20FROM%20inventory%20WHERE%20id%3D1"
```

Expected output:
```json
{"message":"Forbidden"}
```

![SQLi blocked](./snap-15.png)

---

### Task 4: Monitor API Traffic with CloudWatch

#### Task 4.1: WAF Traffic Overview

1. Navigate to `api-acl → Traffic overview`.
2. In **Data filters**, deselect Challenge and Captcha.
3. Review:
   - **Action totals graph** — spikes in allows vs. blocks across test phases.
   - **Top 10 rules** — rules matched during testing.
   - **Attack types bar graph** — single detected SQLi request and volumetric (bot-suspected) requests from the Artillery load test.

#### Task 4.2: CloudWatch Metrics

1. Navigate to **CloudWatch → Metrics → All metrics → WAFV2**.
2. Select dimension: **LabelName, LabelNamespace, Region, RuleGroup**.
3. Graph the following metrics:
   - `LabelName: ipallowed` | `MetricName: CountedRequests` — valid partner IP requests.
   - `LabelName: ratelimit` | `MetricName: BlockedRequests` — rate-limited blocks.
4. Navigate back to **WAFV2 → Country, Region, WebACL**.
5. Graph `BlockedRequests` and `AllowedRequests` with **Statistic: Sum** to visualize total allow/deny distribution across test runs.

---

## Conclusions

### Part 1 — Network Firewall & Threat Detection

- Configured stateful AWS Network Firewall rule groups using Suricata-compatible IPS specifications to detect protocol anomalies (FTP over port 443) and ICMP-based port scans.
- Built a multi-layer DNS Firewall using custom domain blocklists and the AWS-managed `AWSManagedDomainsAggregateThreatList` to block DNS resolution of low-reputation and known-threat domains across all protocols.
- Leveraged CloudWatch Log Insights and Contributor Insights to surface three rogue EC2 instances via network traffic pattern analysis.
- Detected and decoded a live DNS exfiltration attack in which base64-encoded IAM credentials were being tunneled out via DNS subdomain queries.
- Quarantined all identified compromised instances by applying a deny-all security group, preserving forensic state for further investigation.

### Part 2 — WAF Web Application Protection

- Deployed a WAF Web ACL with default-deny posture and layered allow rules based on IP allowlisting and rate limiting using compound label-based logic.
- Validated IP filtering (partner vs. non-partner) and rate limiting (50 req/min threshold) via real curl tests and Artillery load testing.
- Blocked SQL injection attacks using the AWS-managed `AWSManagedRulesSQLiRuleSet`, preventing UNION-based data exfiltration from the inventory API.
- Monitored traffic patterns end-to-end using WAF Traffic Overview dashboards and CloudWatch WAFV2 metrics, confirming accurate classification of SQLi, volumetric, and legitimate traffic.
