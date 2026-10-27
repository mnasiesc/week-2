# AWS BUILDERS - CLOUD-DEVOPS Track - WEEK 2

## 1. Identify Network Information
- Operating system: Linux
- Command used: ip addr
- Network adapter: wlan0 | Wi-Fi
- Local IPv4: 10.2.59.183/19 (on wlan0)
- IPv4 classification: Private
- Default gateway: Not shown on the command

## 2. Practive Subnetting
- Starting Network: 10.0.0.0/24
#### Calculations

- Subnets needed: 4 = 2², so borrow 2 bits: /24 + 2 = /26
- Addresses per subnet: 2^(32−26) = 2⁶ = 64
- Block size: 64, so network addresses step by 64 in the last octet: 0, 64, 128, 192
- Conventional usable hosts: 64 − 2 (network + broadcast) = 62
- AWS reserves the first 4 addresses (network, VPC router, DNS, future use) and the last (broadcast). So a total of 5 addresses are reserved by AWS.
- AWS usable hosts: 64 − 5 = 59


| Subnet | CIDR | Network | Full range | Conventional usable | Broadcast | AWS usable (59) |
|---|---|---|---|---|---|---|
| 1 | 10.0.0.0/26 | 10.0.0.0 | 10.0.0.0 – 10.0.0.63 | 10.0.0.1 – 10.0.0.62 | 10.0.0.63 | 10.0.0.4 – 10.0.0.62 |
| 2 | 10.0.0.64/26 | 10.0.0.64 | 10.0.0.64 – 10.0.0.127 | 10.0.0.65 – 10.0.0.126 | 10.0.0.127 | 10.0.0.68 – 10.0.0.126 |
| 3 | 10.0.0.128/26 | 10.0.0.128 | 10.0.0.128 – 10.0.0.191 | 10.0.0.129 – 10.0.0.190 | 10.0.0.191 | 10.0.0.132 – 10.0.0.190 |
| 4 | 10.0.0.192/26 | 10.0.0.192 | 10.0.0.192 – 10.0.0.255 | 10.0.0.193 – 10.0.0.254 | 10.0.0.255 | 10.0.0.196 – 10.0.0.254 |


Check: 4 × 64 = 256 = the /24 total, with no overlap.


## 3. Test DNS Resolution

- Domain name tested: example.com
- Command Used: `nslookup`
- DNS Resolution have succeeded
- Ip Addresses returned: 
```
Name:  example.com
Address: 104.20.23.154
Name:   example.com
Address: 172.66.147.243
Name:   example.com
Address: 2606:4700:10::ac42:93f3
Name:   example.com
Address: 2606:4700:10::6814:179a
```
-  a DNS is a server that resolves domain names to their IP Addresses. DNS lookup is usually the first step in Network Requests.

## 4. Inspect an HTTPS Website

- Website: `https://example.com`
- Command: `curl -I https://example.com`

- The Website have loaded successfully
- The HTTPS Status Code returned: `HTTP/2 200`
- HTTPS adds `Secure` to HTTP. Which guarantees the security of a connection and also the server by using TLS certificates.
- The DNS only resolves the IP of the site. The question whether the website loads also depends on other factors like the status of the server, the current network bandwidth etc...


## 5. Interpret HTTP Status Codes
Complete the table in your README.md.

| Status | Code | Meaning | Example situation |
|---|---|---|---|
| 200 | OK | Request succeeded; response body contains the result | `GET /index.html` returns the page |
| 301 | Moved Permanently | Resource has a new permanent URL; clients/search engines should update | `http://example.com` redirects to `https://example.com` |
| 403 | Forbidden | Server understood the request but refuses it; authentication won't help | Accessing `/admin` without permission, or a directory with no read access |
| 404 | Not Found | Server can't find the requested resource | `GET /nonexistent-page` |
| 500 | Internal Server Error | Server hit an unexpected failure | Unhandled exception or crash in backend code |

#### Then answer: What is the difference between a DNS failure, a connection timeout, a TLS certificate error,
 - DNS failure: The DNS server failed to resolve a domain name.
 - Connection timeout: The TCP handshake (SYN) gets no reply within the time limit. Typical causes are host down, firewall silently dropping packets, wrong IP, or heavy packet loss. Contrast with connection refused, where the host replies with RST because nothing is listening.
 - TLS Certificate Error: The server provides an invalid TLS Certificate. This might because the certificate is expired, hostname mismatched (CN/SAN), self-signed or untrusted CA, or revoked.


 ## 6. Write a Least-Priviledge IAM Policy

### S3 Read-Only Policy Lab

`s3-read-policy.json` allows only `s3:GetObject` on objects in `example-bucket`.

### Q&A

**Which field specifies the permitted action?**
`Action`. Here: `"s3:GetObject"`.

**Which field specifies the resources?**
`Resource`. Here: `"arn:aws:s3:::example-bucket/*"` (`/*` = every object in the bucket).

**Does `s3:GetObject` allow listing all objects in a bucket?**
No. Listing requires `s3:ListBucket` on the bucket ARN (`arn:aws:s3:::example-bucket`, no `/*`). `GetObject` only reads an object whose key is already known.

**Why is a narrow permission safer than `s3:*`?**
Least privilege limits damage from leaked credentials, bugs, or mistakes. `s3:*` permits deleting, overwriting, and exfiltrating data and changing bucket policies. `GetObject` can only read.

**Why should AWS access keys never be committed to GitHub?**
Git history is permanent: deleting the file later doesn't remove the key from old commits. Bots will be able to scrape public repos for keys within minutes, and anyone holding a key gets the full permissions of its identity.


## 7. Troubleshooting Practice
| # | Scenario | First layer | Why |
|---|---|---|---|
| 1 | Domain doesn't resolve | DNS | Name→IP lookup failed before any connection. Check the record exists, nameserver, typos, propagation (`dig`, `nslookup`). |
| 2 | Resolves, but connection times out | Network / transport (IP + TCP) | Name resolved, so DNS is fine. SYN gets no reply: firewall, security group/NACL, routing, or host down. Check with `ping`, `traceroute`, `nc -vz host 443`. |
| 3 | TLS certificate warning | TLS / security | TCP connected; the handshake's certificate validation failed. Check expiry, hostname (SAN) mismatch, untrusted CA, incomplete chain (`openssl s_client`). |
| 4 | HTTP 404 | Application / HTTP | Every lower layer worked, since the server answered. The server doesn't have that path. Check URL, routing config, file/deploy location. |
| 5 | EC2 reaches S3 but gets `AccessDenied` | IAM / authorization | Network and S3 are reachable (it got a reply). Identity lacks permission. Check the EC2 role's policy (`s3:GetObject` and the correct resource ARN), bucket policy, and explicit denies. |
