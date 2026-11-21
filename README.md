# Project 10: Internal DNS Setup using AWS EC2 and BIND9

## Table of Contents

- [Objective](#objective)
- [Technologies Used](#technologies-used)
- [Infrastructure Configuration](#infrastructure-configuration)
- [Implementation Steps](#implementation-steps)
- [DNS Configuration Result](#dns-configuration-result)
- [Verification and Testing](#verification-and-testing)
- [Deliverables](#deliverables)
- [Conclusion](#conclusion)

---

## Objective

Deploy an internal DNS server on AWS EC2 using BIND9 and configure an A record to enable hostname-based access to a web server within a VPC using a fully qualified domain name instead of its private IP address.

---

## Technologies Used

| Technology | Purpose |
|-----------|---------|
| **AWS EC2** | Cloud compute infrastructure for hosting the DNS server |
| **Ubuntu Server 22.04** | Operating system for the DNS server |
| **BIND9** | DNS server software |
| **Linux** | Operating system kernel and utilities |
| **DNS A Records** | DNS record type for hostname-to-IPv4 mapping |
| **nslookup** | DNS query utility for verification |
| **EC2 User Data** | Automated installation and configuration of BIND9 |

---

## Infrastructure Configuration

| Component | Configuration |
|-----------|---|
| **DNS Server Instance Name** | Internal-DNS-Server-v2 |
| **DNS Server Private IP** | 172.31.6.138 |
| **Web Server Private IP** | 172.31.10.249 |
| **DNS Hostname** | app.internal |
| **DNS Record Type** | A Record |
| **DNS Port** | 53 (TCP/UDP) |
| **DNS Software** | BIND9 |
| **Security Group** | internal-dns-sg |
| **VPC** | Default VPC |

---

## Implementation Steps

### Step 1: Create Security Group
- **Name:** `internal-dns-sg`
- **Inbound Rules:**
  - TCP Port 53 from VPC CIDR (for internal DNS queries)
  - UDP Port 53 from VPC CIDR (for DNS lookups)
- **Purpose:** Control traffic to the DNS server

### Step 2: Launch EC2 Instance for DNS Server
- **AMI:** Ubuntu Server 22.04 LTS
- **Instance Type:** t2.micro (or appropriate sizing)
- **Private IP:** 172.31.6.138
- **Security Group:** internal-dns-sg
- **Storage:** 20 GB EBS volume (default)

### Step 3: Configure BIND9 Using EC2 User Data
The following commands are executed via User Data to automate installation and configuration:

```bash
#!/bin/bash
# Update system packages
sudo apt-get update
sudo apt-get upgrade -y

# Install BIND9
sudo apt-get install -y bind9 bind9-utils

# Create DNS zone configuration file
sudo tee /etc/bind/zones/db.internal > /dev/null <<EOF
\$TTL 300
@   IN  SOA ns1.internal. admin.internal. (
        2024010101  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400       ; Minimum TTL
        )
@   IN  NS  ns1.internal.
ns1 IN  A   172.31.6.138
app IN  A   172.31.10.249
EOF

# Update BIND9 options configuration
sudo tee /etc/bind/named.conf.options > /dev/null <<EOF
options {
    directory "/var/cache/bind";
    listen-on { any; };
    allow-query { any; };
    allow-recursion { 172.31.0.0/16; };
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };
    forward only;
};
EOF

# Update BIND9 zone configuration
sudo tee /etc/bind/named.conf.local > /dev/null <<EOF
zone "internal" {
    type master;
    file "/etc/bind/zones/db.internal";
};
EOF

# Create zones directory if it doesn't exist
sudo mkdir -p /etc/bind/zones

# Set proper permissions
sudo chown -R bind:bind /etc/bind/zones
sudo chmod -R 755 /etc/bind/zones

# Enable and start BIND9 service
sudo systemctl enable bind9
sudo systemctl start bind9

# Verify BIND9 service status
sudo systemctl status bind9
```

### Step 4: Configure BIND9 Options
- **Listen Address:** 0.0.0.0 (Listen on all interfaces)
- **Allow Query:** Internal VPC traffic (172.31.0.0/16)
- **Recursion:** Enabled for internal VPC
- **Forwarders:** Google DNS (8.8.8.8, 8.8.4.4)

### Step 5: Create Internal DNS Zone
- **Zone Name:** `internal`
- **Zone File:** `/etc/bind/zones/db.internal`
- **SOA Record:** Configured with appropriate serial, refresh, and retry settings
- **NS Record:** Points to the nameserver (ns1.internal)

### Step 6: Configure A Records
Added A record mapping:
- **Hostname:** `app.internal`
- **IP Address:** `172.31.10.249` (Web server private IP)
- **TTL:** 300 seconds

### Step 7: Enable and Start BIND9 Service
```bash
sudo systemctl enable bind9    # Enable on system startup
sudo systemctl start bind9     # Start the service immediately
```

---

## DNS Configuration Result

### Zone Configuration Summary

**Zone File: `/etc/bind/zones/db.internal`**

```
$TTL 300
@   IN  SOA ns1.internal. admin.internal. (
        2024010101  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400       ; Minimum TTL
        )
@   IN  NS  ns1.internal.
ns1 IN  A   172.31.6.138
app IN  A   172.31.10.249
```

### DNS Resolution Mapping

| Hostname | IP Address | Record Type |
|----------|-----------|-------------|
| app.internal | 172.31.10.249 | A Record |
| ns1.internal | 172.31.6.138 | A Record |

---

## Verification and Testing

### Method 1: Using nslookup

Execute the following command on any EC2 instance within the VPC:

```bash
nslookup app.internal 172.31.6.138
```

**Expected Output:**
```
Server:    172.31.6.138
Address:   172.31.6.138#53

Name:      app.internal
Address:   172.31.10.249
```

### Method 2: Using dig

```bash
dig @172.31.6.138 app.internal
```

**Expected Output:**
```
; <<>> DiG 9.16.1-3ubuntu2 <<>> @172.31.6.138 app.internal
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 1

;; QUESTION SECTION:
;app.internal.                  IN      A

;; ANSWER SECTION:
app.internal.           300     IN      A       172.31.10.249

;; AUTHORITY SECTION:
internal.               300     IN      NS      ns1.internal.
internal.               300     IN      NS      ns1.internal.

;; ADDITIONAL SECTION:
ns1.internal.           300     IN      A       172.31.6.138

;; Query time: 1 msec
;; SERVER: 172.31.6.138#53(172.31.6.138)
;; WHEN: Fri Jan 01 00:00:00 UTC 2024
;; MSG SIZE  rcvd: 100
```

### Method 3: Using host Command

```bash
host app.internal 172.31.6.138
```

**Expected Output:**
```
Using domain server:
Name: 172.31.6.138
Address: 172.31.6.138#53
Aliases:

app.internal has address 172.31.10.249
```

---

## Deliverables

The following deliverables document the successful implementation of the internal DNS infrastructure:

### 1. DNS EC2 Instance Screenshot
- Shows the DNS server instance (`Internal-DNS-Server-v2`) running in AWS EC2 console
- Displays instance details including:
  - Instance ID
  - Private IP: 172.31.6.138
  - Instance state: Running
  - Availability Zone
  - Security Group: internal-dns-sg

### 2. DNS Security Group Configuration
- Security group name: `internal-dns-sg`
- Inbound Rules:
  - **Type:** DNS (TCP)
  - **Protocol:** TCP
  - **Port Range:** 53
  - **Source:** 172.31.0.0/16 (VPC CIDR)
  
  - **Type:** DNS (UDP)
  - **Protocol:** UDP
  - **Port Range:** 53
  - **Source:** 172.31.0.0/16 (VPC CIDR)

### 3. BIND9 Configuration through User Data
- Complete EC2 User Data script including:
  - BIND9 package installation
  - DNS zone configuration (`db.internal`)
  - BIND9 options configuration
  - Zone file definitions
  - Service enablement and startup commands

### 4. DNS Resolution Verification Screenshot
- Shows successful DNS query results
- Displays resolution of `app.internal` to `172.31.10.249`
- Includes query response time and server details

### 5. nslookup Output Log
```
Server:    172.31.6.138
Address:   172.31.6.138#53

Name:      app.internal
Address:   172.31.10.249
```

---

## Troubleshooting Guide

### Issue: DNS Service Not Starting

**Solution:**
```bash
# Check BIND9 service status
sudo systemctl status bind9

# Check BIND9 logs
sudo journalctl -u bind9 -n 50

# Check named configuration syntax
sudo named-checkconf

# Restart BIND9 service
sudo systemctl restart bind9
```

### Issue: DNS Queries Timing Out

**Solution:**
1. Verify DNS instance security group allows ports 53 (TCP/UDP)
2. Ensure BIND9 is listening on correct interface:
   ```bash
   sudo netstat -tuln | grep 53
   ```
3. Check firewall rules within instance:
   ```bash
   sudo ufw status
   ```

### Issue: Zone File Syntax Errors

**Solution:**
```bash
# Check zone file syntax
sudo named-checkzone internal /etc/bind/zones/db.internal

# Fix any reported errors and reload BIND9
sudo systemctl reload bind9
```

### Issue: Hostname Not Resolving

**Solution:**
1. Verify DNS record exists:
   ```bash
   sudo dig @localhost internal A
   ```
2. Check zone file contains correct A record:
   ```bash
   sudo cat /etc/bind/zones/db.internal
   ```
3. Increment SOA serial number and reload:
   ```bash
   sudo systemctl reload bind9
   ```

---

## Security Considerations

1. **VPC Isolation:** DNS server is restricted to internal VPC communication (172.31.0.0/16)
2. **Port Restrictions:** Only ports 53 (TCP/UDP) are open to VPC traffic
3. **Access Control:** DNS queries limited to VPC CIDR block
4. **Recursive Queries:** Enabled only for internal VPC addresses
5. **External Forwarders:** Requests for external domains forwarded to Google DNS

---

## Conclusion

The internal DNS infrastructure has been successfully deployed on AWS EC2 using BIND9. The configuration enables:

- **Hostname-based access:** Web servers and applications can be accessed using `app.internal` instead of private IP addresses
- **Centralized DNS management:** All DNS records are managed through a single BIND9 server
- **Internal VPC isolation:** DNS traffic is restricted to internal VPC communication
- **Automatic configuration:** EC2 User Data automates the entire setup process
- **Scalability:** Additional A records can be easily added to support more services

This setup provides a foundation for a larger internal DNS infrastructure within the AWS environment, reducing the complexity of managing IP addresses and enabling easier service discovery within the VPC.

---

## Additional Resources

- [BIND9 Documentation](https://www.isc.org/bind/)
- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)
- [DNS A Records](https://www.cloudflare.com/learning/dns/dns-records/dns-a-record/)
- [Ubuntu BIND9 Package](https://packages.ubuntu.com/jammy/bind9)
- [RFC 1035 - Domain Names - Implementation and Specification](https://tools.ietf.org/html/rfc1035)

---

**Project Date:** January 1, 2024  
**Status:** Complete  
**Last Updated:** November 21, 2026
