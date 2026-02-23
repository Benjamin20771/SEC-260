
# SEC-260 Midterm Assessment Playbook

## Part 1: HTTPS Build (CA + Web Server)

### Phase 1 — Web Server Setup
```
sudo useradd -m YourFirstName && sudo passwd YourFirstName
sudo usermod -aG wheel YourFirstName
sudo passwd root
sudo hostnamectl set-hostname YourLastNameVM
dnf install -y httpd
sudo systemctl enable httpd && sudo systemctl start httpd && sudo systemctl status httpd | grep -i active
sudo firewall-cmd --permanent --add-port=80/tcp && sudo firewall-cmd --reload
ip addr  # Write down your IP!
```

### Phase 2 — CA VM Setup
```
sudo hostnamectl set-hostname ca-YourFirstName
sudo systemctl enable sshd && sudo systemctl start sshd
sudo firewall-cmd --permanent --add-port=22/tcp && sudo firewall-cmd --reload
ip addr  # Write down CA's IP!
openssl genrsa -aes256 -out cakey.pem 2048
openssl req -new -x509 -days 365 -key cakey.pem -out cacert.pem
```
> ⚠ Use **Joyce310** for Organization Name, Org Unit Name, and Common Name!

### Phase 3 — Web Server: Generate Key & CSR
```
openssl genrsa -out websrv.key 2048
openssl req -new -key websrv.key -out websrv.csr
scp websrv.csr root@CA_IP:/root/
```
> ⚠ Use **Joyce310** here too — must match the CA entries!

### Phase 4 — CA: Sign the Certificate
```
openssl x509 -req -days 365 -in websrv.csr -CA cacert.pem -CAkey cakey.pem -CAcreateserial -out websrv.crt
scp websrv.crt root@WEBSERVER_IP:/root/
```

### Phase 5 — Web Server: Enable HTTPS
```
cp /root/websrv.crt /etc/pki/tls/certs/websrv.crt
cp /root/websrv.key /etc/pki/tls/private/websrv.key
dnf install -y mod_ssl
vi /etc/httpd/conf.d/ssl.conf  # Update the two lines below
sudo firewall-cmd --permanent --add-port=443/tcp && sudo firewall-cmd --reload
sudo systemctl restart httpd
```

In `ssl.conf`, find and update:
```
SSLCertificateFile /etc/pki/tls/certs/websrv.crt
SSLCertificateKeyFile /etc/pki/tls/private/websrv.key
```

Test: `curl -k https://YOUR_IP`

### Verification (cat commands for submission)
```
# On CA VM:
cat /root/cacert.pem

# On Web Server:
cat /root/websrv.key
cat /root/websrv.crt
```

---

## Part 2: PCAP Analysis (Wireshark)

### Key Display Filters
| Filter | What It Shows |
|--------|---------------|
| `dns` | All DNS queries and responses |
| `tcp.flags.syn==1 && tcp.flags.ack==0` | SYN packets (start of TCP handshake) |
| `tcp.flags == 0x012` | SYN-ACK packets |
| `http` | All unencrypted HTTP traffic |
| `http.request.method == "GET"` | GET requests |
| `http.request.method == "POST"` | POST requests |
| `tls` | TLS/HTTPS handshake traffic |
| `tls.handshake.type == 11` | Server Certificate packet |
| `ip.addr == 192.168.x.x` | Filter by IP |

### TCP 3-Way Handshake
1. **SYN** — Client → Server (initiates)
2. **SYN-ACK** — Server → Client (agrees)
3. **ACK** — Client → Server (confirmed, connection open)

### GET vs POST
- **GET** — data is in the URL after `?` (visible in Start Line)
- **POST** — data is in the message body (after the blank line separating headers)

### TLS Handshake Order
1. Client Hello
2. Server Hello
3. **Certificate** ← cert details here (`tls.handshake.type == 11`)
4. Server Hello Done
5. Client Key Exchange
6. Change Cipher Spec
7. Finished (Encrypted)

To find cert details, expand: `TLS Handshake → Certificate → signedCertificate → subject/issuer/validity`

### HTTP Status Codes
| Code | Meaning | How to Trigger |
|------|---------|----------------|
| 200 | OK | Normal valid request |
| 304 | Not Modified | Reload a cached page |
| 400 | Bad Request | Send malformed start line via telnet/nc |
| 401 | Unauthorized | Access protected resource without credentials |
| 403 | Forbidden | Request file with no read permission |
| 404 | Not Found | Request a file that doesn't exist |

---

## Quick Tips
- Write down both VM IPs first — you need them for SCP
- `Joyce310` for Org Name, Org Unit, AND Common Name on both VMs
- Apache won't start? Check `/var/log/httpd/error_log`
- Check open ports: `sudo firewall-cmd --list-all`
- Start PCAP analysis with `dns` filter to orient yourself
