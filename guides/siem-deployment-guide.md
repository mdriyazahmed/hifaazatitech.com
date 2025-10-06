# Complete SIEM Deployment Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Planning Phase](#planning)
3. [Architecture Design](#architecture)
4. [Implementation](#implementation)
5. [Tuning & Optimization](#tuning)
6. [Use Cases](#use-cases)

---

## Introduction

This guide provides step-by-step instructions for deploying a Security Information and Event Management (SIEM) solution in an enterprise environment.

**Target Audience:** Security Operations, Security Engineers, SOC Analysts
**Difficulty:** Advanced
**Estimated Time:** 8-12 weeks for full deployment

---

## Planning Phase

### 1. Define Requirements

**Business Objectives:**
- Compliance requirements (PCI-DSS, HIPAA, SOC 2)
- Threat detection and response
- Forensics and investigation
- Real-time monitoring

**Technical Requirements:**
- Events per second (EPS): Calculate based on environment size
- Storage requirements: Plan for 30-90 days hot storage
- Retention policy: 1-3 years typically
- Search performance needs

**Staffing:**
- SIEM Administrator (1-2 FTE)
- SOC Analysts (3-5 FTE for 24/7 coverage)
- Detection Engineers (1-2 FTE)

### 2. Data Source Inventory

Identify all log sources:

| Source Type | Examples | Priority |
|-------------|----------|----------|
| Authentication | Active Directory, LDAP, SSO | Critical |
| Network | Firewalls, Proxies, IDS/IPS | Critical |
| Endpoints | EDR, Antivirus, Windows Events | High |
| Cloud | AWS CloudTrail, Azure AD, O365 | High |
| Applications | Web servers, Databases | Medium |
| Security Tools | Vulnerability scanners, DLP | Medium |

---

## Architecture Design

### Deployment Model

**Option 1: On-Premises**
```
[Data Sources] → [Forwarders] → [Indexers] → [Search Heads] → [Analysts]
                                      ↓
                                 [Storage]
```

**Option 2: Hybrid (Recommended)**
```
[On-Prem Sources] → [Forwarders] ──┐
                                    ├─→ [Cloud SIEM] → [Analysts]
[Cloud Sources] ────────────────────┘
```

### Capacity Planning

**EPS Calculation Example:**
- 5,000 endpoints × 50 events/day = 250K events
- 50 servers × 1,000 events/day = 50K events
- Network devices: 100K events/day
- **Total:** ~400K events/day = 4.6 EPS average
- **Peak:** Multiply by 3-5x = 14-23 EPS

**Storage Formula:**
```
Daily Storage = (Events/Day × Avg Event Size) ÷ Compression Ratio
```

Example:
```
400,000 events × 500 bytes ÷ 5 (compression) = 40 MB/day
40 MB × 90 days = 3.6 GB hot storage
```

### High Availability

- Minimum 3 indexers for search head cluster
- Replication factor: 2-3
- Site-to-site replication for disaster recovery
- Load balancer for forwarder traffic

---

## Implementation

### Phase 1: Core Infrastructure (Week 1-2)

**1. Deploy SIEM Components**

```bash
# Example: Splunk installation
sudo dpkg -i splunk-enterprise_9.0.deb
sudo /opt/splunk/bin/splunk start --accept-license
sudo /opt/splunk/bin/splunk enable boot-start
```

**2. Configure Indexers**

```conf
# indexes.conf
[security]
homePath = $SPLUNK_DB/security/db
coldPath = $SPLUNK_DB/security/colddb
thawedPath = $SPLUNK_DB/security/thaweddb
maxDataSize = auto_high_volume
frozenTimePeriodInSecs = 7776000  # 90 days
```

**3. Set Up Forwarders**

```conf
# outputs.conf
[tcpout]
defaultGroup = indexer_pool

[tcpout:indexer_pool]
server = indexer1.example.com:9997, indexer2.example.com:9997
autoLBFrequency = 30
useACK = true
```

### Phase 2: Data Onboarding (Week 3-6)

**Priority 1 Sources: Authentication**

```xml
<!-- Windows Event Forwarding -->
<Subscription>
  <SubscriptionId>Security-Events</SubscriptionId>
  <Query>
    <QueryList>
      <Query Id="0">
        <Select Path="Security">*[System[(EventID=4624 or EventID=4625 or EventID=4648)]]</Select>
      </Query>
    </QueryList>
  </Query>
</Subscription>
```

**Priority 2 Sources: Network**

```conf
# Palo Alto Networks syslog
inputs.conf:
[udp://514]
connection_host = ip
sourcetype = pan:traffic
index = firewall
```

**Priority 3 Sources: Cloud**

```python
# AWS CloudTrail ingestion
import boto3

s3 = boto3.client('s3')
bucket = 'cloudtrail-logs'
prefix = 'AWSLogs/'

# Download and forward to SIEM
```

### Phase 3: Parsing & Enrichment (Week 7-8)

**Field Extractions**

```conf
# props.conf
[authentication_logs]
REPORT-fields = extract_user, extract_src, extract_dest
LOOKUP-geo = geo_lookup src_ip OUTPUT city, country
EVAL-risk_score = case(action="failure" AND count>5, 75, action="success", 0)
```

**CIM Compliance**

```conf
# Map to Common Information Model
[source::firewall]
FIELDLIB-cim = authentication
EVAL-action = case(fw_action="allow", "success", fw_action="deny", "failure")
EVAL-user = coalesce(src_user, username)
```

---

## Tuning & Optimization

### 1. Search Optimization

**Inefficient Search:**
```spl
index=* "failed login" | stats count
```

**Optimized Search:**
```spl
index=auth sourcetype=wineventlog EventCode=4625
| stats count by user, src
| where count > 5
```

### 2. Alert Tuning

**Reduce False Positives:**
1. Baseline normal activity (2-4 weeks)
2. Implement thresholds
3. Add whitelists for known-good
4. Use risk-based alerting

**Example Alert:**
```spl
index=auth action=failure
| stats count by user, src
| where count > 10
| lookup whitelist_users user OUTPUT is_service
| where is_service != 1
```

### 3. Performance Tuning

**Acceleration:**
- Data models for common searches
- Summary indexing for long-running reports
- Accelerated lookups for enrichment

**Resource Optimization:**
- Adjust bucket size based on search patterns
- Use  bloom filters effectively
- Archive cold data to S3/object storage

---

## Use Cases

### UC-001: Brute Force Detection

**Objective:** Detect multiple failed login attempts

**Search:**
```spl
index=auth action=failure
| bin span=5m _time
| stats count by _time, user, src
| where count > 5
| table _time, user, src, count
```

**Actions:**
- Alert SOC analyst
- Trigger automated response (block IP)
- Create investigation case

### UC-002: Lateral Movement Detection

**Objective:** Identify abnormal authentication patterns

**Search:**
```spl
index=auth EventCode=4624 Logon_Type=3
| stats dc(dest) as unique_systems by user
| where unique_systems > 10
```

**Baseline:** Users typically access 2-5 systems

### UC-003: Data Exfiltration Detection

**Objective:** Detect large data transfers to external IPs

**Search:**
```spl
index=proxy action=allowed
| lookup internal_ips src OUTPUT is_internal
| where is_internal=1
| stats sum(bytes_out) as total_out by src, dest_domain
| where total_out > 100000000
```

**Threshold:** Alert on transfers > 100 MB

---

## Maintenance & Operations

### Daily Tasks
- Review overnight alerts
- Verify forwarder health
- Check license usage

### Weekly Tasks
- Review new detection rules
- Tune noisy alerts
- Update threat intelligence feeds

### Monthly Tasks
- Capacity planning review
- Performance optimization
- Detection coverage gap analysis

---

## Additional Resources

- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [Splunk Common Information Model](https://docs.splunk.com/Documentation/CIM)
- [ELK Stack Documentation](https://www.elastic.co/guide/)
- [Microsoft Sentinel Docs](https://docs.microsoft.com/azure/sentinel/)

---

**Related Guides:**
- [SOC Operations Guide](guides/soc-operations-guide.md)
- [Detection Engineering Guide](guides/detection-engineering.md)
- [Incident Response Playbooks](guides/incident-response.md)
