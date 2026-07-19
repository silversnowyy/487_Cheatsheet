# CSE487 Cybersecurity Project — Full Step-by-Step Implementation Guide

This guide turns the official instructions into concrete, ordered Linux steps and commands.
It follows the project's own **Recommended Work Sequence (§32)**. Do it in this order — each
phase depends on the one before it. Everything below assumes:

- **Server VM** hostname `server`, isolated Host-only interface (e.g. `enp0s8`)
- **Client VM** hostname `client`
- Domain: `acmeca.com` (+ `www`, `apply` subdomains)
- Admin user: `super` (has `sudo`, never work as `root` directly)

> ⚠️ Safety reminder baked into every phase: all scans/attacks/floods target **only your own
> Server VM on your own Host-only network**. Never bridge these tests to the real network.

---

## Phase 0 — Prepare the VMs (§6.3)

**Server VM**
```bash
# After OS install, as the super user:
sudo hostnamectl set-hostname server
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```
Check adapters (VirtualBox: Adapter 1 = NAT, Adapter 2 = Host-only):
```bash
ip address
hostnamectl
hostname -I
groups          # confirm 'sudo' group membership
sudo whoami     # should print root only when using sudo
```
Take a **snapshot** now (VirtualBox: VM → Snapshots → Take).

**Client VM**: clone the Server VM in VirtualBox ("Full Clone"), boot it, then:
```bash
sudo hostnamectl set-hostname client
sudo reboot
```
Verify both VMs boot, then take a second snapshot on each.

---

## Phase 1 — Network Configuration (§7)

Find each VM's Host-only IP:
```bash
ip -4 addr show enp0s8      # adjust interface name as needed
```
Record `<server-ip>` and `<client-ip>`. Test connectivity:
```bash
# On Server:
ping -c 4 <client-ip>
# On Client:
ping -c 4 <server-ip>
```
Draw your network diagram now (NAT/Internet leg + Host-only leg, both VM addresses, and
list of services on the Server) — you'll need it for the report.

---

## Phase 2 — Install Apache2 (§9)

```bash
sudo apt install apache2 -y
sudo systemctl enable apache2
sudo systemctl start apache2
sudo systemctl status apache2
sudo apache2ctl configtest
```
Create document roots:
```bash
sudo mkdir -p /var/www/acmeca.com/public_html
sudo mkdir -p /var/www/apply.acmeca.com/public_html
sudo chown -R www-data:www-data /var/www/acmeca.com /var/www/apply.acmeca.com
```
Basic (HTTP, temporary) vhosts — you will convert these to HTTPS in Phase 12:
```bash
sudo tee /etc/apache2/sites-available/acmeca.com.conf >/dev/null <<'EOF'
<VirtualHost *:80>
    ServerName acmeca.com
    ServerAlias www.acmeca.com
    DocumentRoot /var/www/acmeca.com/public_html
    ErrorLog ${APACHE_LOG_DIR}/acmeca_error.log
    CustomLog ${APACHE_LOG_DIR}/acmeca_access.log combined
</VirtualHost>
EOF

sudo tee /etc/apache2/sites-available/apply.acmeca.com.conf >/dev/null <<'EOF'
<VirtualHost *:80>
    ServerName apply.acmeca.com
    DocumentRoot /var/www/apply.acmeca.com/public_html
    ErrorLog ${APACHE_LOG_DIR}/apply_error.log
    CustomLog ${APACHE_LOG_DIR}/apply_access.log combined
</VirtualHost>
EOF

sudo a2ensite acmeca.com apply.acmeca.com
sudo a2dissite 000-default
sudo apache2ctl configtest
sudo systemctl reload apache2
```
Confirm Apache runs as an unprivileged account with no shell:
```bash
ps -eo user,cmd | grep apache2
getent passwd www-data     # shell field should be /usr/sbin/nologin
```

---

## Phase 3 — Configure BIND9 Authoritative DNS (§10)

```bash
sudo apt install bind9 bind9utils dnsutils -y
```
Define the zone in `/etc/bind/named.conf.local`:
```bash
sudo tee -a /etc/bind/named.conf.local >/dev/null <<EOF

zone "acmeca.com" {
    type master;
    file "/etc/bind/zones/db.acmeca.com";
};

zone "$(echo <server-ip> | awk -F. '{print $3"."$2"."$1}').in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.<reverse-octets>";
};
EOF
```
(Replace `<server-ip>` / `<reverse-octets>` manually — reverse zone name uses the /24's
octets reversed, e.g. for `192.168.56.10` the zone is `56.168.192.in-addr.arpa`.)

Forward zone file:
```bash
sudo mkdir -p /etc/bind/zones
sudo tee /etc/bind/zones/db.acmeca.com >/dev/null <<'EOF'
$TTL 86400
@   IN  SOA ns1.acmeca.com. admin.acmeca.com. (
        2026071901 ; serial (increment every edit)
        3600       ; refresh
        1800       ; retry
        604800     ; expire
        86400 )    ; minimum

    IN  NS  ns1.acmeca.com.
ns1 IN  A   <server-ip>
@   IN  A   <server-ip>
www IN  CNAME acmeca.com.
apply IN CNAME acmeca.com.
EOF
```
Reverse zone file (`db.<reverse-octets>`):
```bash
sudo tee /etc/bind/zones/db.<reverse-octets> >/dev/null <<'EOF'
$TTL 86400
@   IN  SOA ns1.acmeca.com. admin.acmeca.com. (
        2026071901 ; serial
        3600 1800 604800 86400 )
    IN  NS  ns1.acmeca.com.
<last-octet> IN PTR acmeca.com.
EOF
```
Validate and restart:
```bash
sudo named-checkconf
sudo named-checkzone acmeca.com /etc/bind/zones/db.acmeca.com
sudo named-checkzone <reverse-zone-name> /etc/bind/zones/db.<reverse-octets>
sudo systemctl enable bind9
sudo systemctl restart bind9
```
Verify (run from Server first, then Client once split DNS below is set):
```bash
dig @<server-ip> acmeca.com
dig @<server-ip> www.acmeca.com
dig @<server-ip> apply.acmeca.com
dig @<server-ip> -x <server-ip>
```

**Split DNS on the Client (systemd-resolved):**
```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/acmeca.conf >/dev/null <<EOF
[Resolve]
DNS=<server-ip>
Domains=~acmeca.com
EOF
sudo systemctl restart systemd-resolved
resolvectl status
resolvectl query acmeca.com
resolvectl query www.google.com   # should still use normal external resolver
```
Confirm you are **not** using `/etc/hosts`:
```bash
cat /etc/hosts   # must contain no acmeca.com entries
```

---

## Phase 4 — Configure OpenSSH on Port 3000 (§11)

```bash
sudo apt install openssh-server -y
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo sed -i 's/^#\?Port .*/Port 3000/' /etc/ssh/sshd_config
sudo systemctl restart ssh
sudo systemctl enable ssh
```
From the Client:
```bash
ssh super@<server-ip>            # should time out / refuse (nothing on 22 yet, or blocked after firewall phase)
ssh -p 3000 super@<server-ip>    # should succeed
```
Generate one failed login on purpose (wrong password), then inspect logs on the Server:
```bash
journalctl -u ssh --since "5 min ago"
sudo grep -i "failed" /var/log/auth.log   # Debian/Ubuntu path
```
Optional hardening (bonus/recommended):
```bash
sudo sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo tee -a /etc/ssh/sshd_config >/dev/null <<'EOF'
AllowUsers super
EOF
sudo systemctl restart ssh
```
Bonus — SSH key auth:
```bash
# On Client:
ssh-keygen -t ed25519 -f ~/.ssh/acmeca_key
ssh-copy-id -p 3000 -i ~/.ssh/acmeca_key.pub super@<server-ip>
ssh -p 3000 -i ~/.ssh/acmeca_key super@<server-ip>
```

---

## Phase 5 — Configure the Firewall (§12)

Using UFW (recommended). **Do this in the exact order below** so you don't lock yourself out:
```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 3000/tcp        # SSH — allow THIS before enabling ufw
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow from <client-subnet e.g. 192.168.56.0/24> proto icmp
sudo ufw enable
sudo ufw status verbose
```
Evidence to save: `sudo ufw status numbered`, then from Client confirm:
```bash
curl -I http://acmeca.com
ssh -p 3000 super@<server-ip>
ssh super@<server-ip>       # port 22 should now fail/refuse
```

---

## Phase 6 — Install Snort or Suricata as an IDS (§13)

Suricata example (Debian/Ubuntu):
```bash
sudo apt install suricata -y
sudo suricata-update              # pull current ET-Open rules
```
Point Suricata at your Host-only interface in `/etc/suricata/suricata.yaml`:
```bash
sudo sed -n '/af-packet:/,/interface:/p' /etc/suricata/suricata.yaml
sudo sed -i 's/interface: eth0/interface: enp0s8/' /etc/suricata/suricata.yaml
```
Enable a rule capable of catching a SYN flood (ET SCAN rules cover high-rate SYN behavior),
or write a simple custom rule in `/etc/suricata/rules/local.rules`:
```bash
sudo tee /etc/suricata/rules/local.rules >/dev/null <<'EOF'
alert tcp any any -> $HOME_NET 443 (msg:"Possible SYN flood to HTTPS"; flags:S; threshold:type both, track by_dst, count 50, seconds 10; sid:1000001; rev:1;)
EOF
sudo sed -i '/rule-files:/a\  - local.rules' /etc/suricata/suricata.yaml
```
Run as a service and confirm:
```bash
sudo systemctl enable suricata
sudo systemctl restart suricata
sudo systemctl status suricata
sudo tail -f /var/log/suricata/fast.log
```
You will generate real alerts in Phase 16. Preserve `/var/log/suricata/eve.json` and
`fast.log` for the report.

---

## Phase 7 — Build the PKI Hierarchy (§14)

Create isolated OpenSSL CA directories for Root + 3 SubCAs:
```bash
sudo mkdir -p /etc/pki/{root-ca,dv-ca,ov-ca,ev-ca}/{certs,crl,newcerts,private,csr}
sudo chmod 700 /etc/pki/*/private
for ca in root-ca dv-ca ov-ca ev-ca; do
  sudo touch /etc/pki/$ca/index.txt
  sudo sh -c "echo 1000 > /etc/pki/$ca/serial"
  sudo sh -c "echo 1000 > /etc/pki/$ca/crlnumber"
done
```
### 7.1 Root CA (10-year validity)
`openssl.cnf` snippets go in each CA directory; minimally you need `[ ca ]`, `[ policy ]`,
`[ req ]`, `[ v3_ca ]` sections pointing `dir = /etc/pki/root-ca`, etc. Then:
```bash
# Key (encrypted, RSA 4096 recommended)
sudo openssl genrsa -aes256 -out /etc/pki/root-ca/private/root-ca.key.pem 4096
sudo chmod 400 /etc/pki/root-ca/private/root-ca.key.pem

# Self-signed Root cert, 10 years, CA:TRUE
sudo openssl req -config /etc/pki/root-ca/openssl.cnf -key /etc/pki/root-ca/private/root-ca.key.pem \
  -new -x509 -days 3650 -sha256 -extensions v3_ca \
  -out /etc/pki/root-ca/certs/root-ca.cert.pem \
  -subj "/C=US/O=ACME CA/CN=ACME Root CA"

openssl x509 -noout -text -in /etc/pki/root-ca/certs/root-ca.cert.pem
```
`v3_ca` extension block must contain:
```
basicConstraints = critical, CA:TRUE
keyUsage = critical, keyCertSign, cRLSign
```

### 7.2 Each SubCA (DV / OV / EV, 5-year validity, pathlen:0)
Repeat for `dv-ca`, `ov-ca`, `ev-ca` (example shown for `dv-ca`):
```bash
sudo openssl genrsa -aes256 -out /etc/pki/dv-ca/private/dv-ca.key.pem 4096
sudo chmod 400 /etc/pki/dv-ca/private/dv-ca.key.pem

sudo openssl req -config /etc/pki/dv-ca/openssl.cnf -new -sha256 \
  -key /etc/pki/dv-ca/private/dv-ca.key.pem \
  -out /etc/pki/dv-ca/csr/dv-ca.csr.pem \
  -subj "/C=US/O=ACME CA/CN=ACME DV SubCA"

sudo openssl ca -config /etc/pki/root-ca/openssl.cnf -extensions v3_intermediate_ca \
  -days 1825 -notext -md sha256 \
  -in /etc/pki/dv-ca/csr/dv-ca.csr.pem \
  -out /etc/pki/dv-ca/certs/dv-ca.cert.pem

openssl verify -CAfile /etc/pki/root-ca/certs/root-ca.cert.pem /etc/pki/dv-ca/certs/dv-ca.cert.pem
```
`v3_intermediate_ca` extension block:
```
basicConstraints = critical, CA:TRUE, pathlen:0
keyUsage = critical, keyCertSign, cRLSign, digitalSignature
```
Do the identical two steps for `ov-ca` and `ev-ca` (own key, own CSR, signed by Root).

---

## Phase 8 — Manual OpenSSL Competency (§15)

Practice each of these until you can do them without notes:
```bash
# RSA / ECC key
openssl genrsa -out test.key.pem 2048
openssl ecparam -name prime256v1 -genkey -noout -out test-ec.key.pem

# Derive public key
openssl rsa -in test.key.pem -pubout -out test.pub.pem

# Encrypt an existing private key
openssl rsa -aes256 -in test.key.pem -out test.enc.key.pem

# CSR with SAN
openssl req -new -key test.key.pem -out test.csr.pem \
  -subj "/CN=demo.acmeca.com" \
  -addext "subjectAltName=DNS:demo.acmeca.com"
openssl req -in test.csr.pem -noout -text

# Self-signed cert
openssl x509 -in test.csr.pem -signkey test.key.pem -req -days 365 -out test-selfsigned.pem

# Issue a leaf via the DV SubCA (see leaf openssl.cnf with SAN + CRL DP)
sudo openssl ca -config /etc/pki/dv-ca/openssl.cnf -extensions server_cert \
  -days 365 -notext -md sha256 -in test.csr.pem -out test-leaf.cert.pem

# Inspect / verify chain
openssl x509 -in test-leaf.cert.pem -noout -text
cat test-leaf.cert.pem /etc/pki/dv-ca/certs/dv-ca.cert.pem > chain.pem
openssl verify -CAfile /etc/pki/root-ca/certs/root-ca.cert.pem \
  -untrusted /etc/pki/dv-ca/certs/dv-ca.cert.pem test-leaf.cert.pem

# Revoke + CRL
sudo openssl ca -config /etc/pki/dv-ca/openssl.cnf -revoke test-leaf.cert.pem -crl_reason keyCompromise
sudo openssl ca -config /etc/pki/dv-ca/openssl.cnf -gencrl -out /etc/pki/dv-ca/crl/dvsubca.crl
openssl crl -in /etc/pki/dv-ca/crl/dvsubca.crl -noout -text

# TLS check
openssl s_client -connect acmeca.com:443 -servername acmeca.com -tls1_2 </dev/null
openssl s_client -connect acmeca.com:443 -servername acmeca.com -tls1_3 </dev/null
```

---

## Phase 9 — Build the CA Web Application (§16)

Choose any stack (Flask/Django/Node/PHP — pick what you know). Minimum architecture:

**Database** (SQLite is fine) — core tables:
```sql
CREATE TABLE applicants (
  id INTEGER PRIMARY KEY, username TEXT UNIQUE NOT NULL,
  full_name TEXT, email TEXT UNIQUE, password_hash TEXT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE password_reset_tokens (
  id INTEGER PRIMARY KEY, applicant_id INTEGER REFERENCES applicants(id),
  token_hash TEXT NOT NULL, expires_at DATETIME NOT NULL, used INTEGER DEFAULT 0
);
CREATE TABLE requests (
  id INTEGER PRIMARY KEY, applicant_id INTEGER REFERENCES applicants(id),
  cert_type TEXT CHECK(cert_type IN ('DV','OV','EV')),
  key_algo TEXT, domain TEXT, csr TEXT, status TEXT DEFAULT 'Pending',
  serial TEXT, subca TEXT, created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE admins (
  id INTEGER PRIMARY KEY, username TEXT UNIQUE, password_hash TEXT NOT NULL
);
CREATE TABLE audit_log (
  id INTEGER PRIMARY KEY, actor TEXT, action TEXT, target TEXT,
  result TEXT, created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```
Password hashing (example, Python + Flask/argon2):
```bash
pip install argon2-cffi
```
```python
from argon2 import PasswordHasher
ph = PasswordHasher()
hash = ph.hash(plaintext_password)          # store this, never the plaintext
ph.verify(hash, submitted_password)         # raises on mismatch
```
**Ownership enforcement (server-side, mandatory)** — pseudocode pattern for every route:
```python
@app.route("/applications/<int:req_id>")
@login_required
def view_request(req_id):
    row = db.get_request(req_id)
    if row is None or row.applicant_id != current_user.id:
        abort(403)     # never trust the request ID alone
    return render(row)
```
**Certificate/key generation calls out to OpenSSL** — call it safely, never via unsanitized
shell string concatenation:
```python
import subprocess
subprocess.run(
    ["openssl", "req", "-new", "-key", keyfile, "-out", csrfile,
     "-subj", f"/CN={validated_domain}"],
    check=True
)
```
Run the app under its own restricted OS account, not root:
```bash
sudo useradd -r -s /usr/sbin/nologin caapp
sudo chown -R caapp:caapp /opt/acmeca-app
```
Reverse-proxy it behind Apache with `mod_proxy` / `mod_wsgi`, or serve directly on an
internal port and have Apache `ProxyPass` to it — either is acceptable as long as TLS
termination happens at Apache (Phase 12).

---

## Phase 10 — Certificate Issuance & Download (§17)

Admin approval flow (backend triggers the real OpenSSL issuance):
```bash
sudo openssl ca -config /etc/pki/<subca>/openssl.cnf -extensions server_cert \
  -days 365 -notext -md sha256 \
  -in /path/to/applicant.csr.pem \
  -out /path/to/applicant-leaf.cert.pem
```
`server_cert` extension section (leaf):
```
basicConstraints = critical, CA:FALSE
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
crlDistributionPoints = URI:http://acmeca.com/crl/<subca>.crl
```
Bundle for download (leaf + SubCA + Root, for this educational project only):
```bash
cat applicant-leaf.cert.pem /etc/pki/dv-ca/certs/dv-ca.cert.pem \
    /etc/pki/root-ca/certs/root-ca.cert.pem > applicant-bundle.pem
```
Apache itself must send only **leaf + SubCA** (not Root) — configured in Phase 12.

---

## Phase 11 — CRL Distribution (§18)

```bash
sudo mkdir -p /var/www/acmeca.com/public_html/crl
for ca in dv ov ev; do
  sudo openssl ca -config /etc/pki/${ca}-ca/openssl.cnf -gencrl \
    -out /var/www/acmeca.com/public_html/crl/${ca}subca.crl
done
```
Serve over plain HTTP (CRLs are the one exception to the HTTP→HTTPS redirect — add an
exclusion in your Apache rewrite rules for the `/crl/` path). Full revoke → republish → verify
cycle:
```bash
sudo openssl ca -config /etc/pki/ov-ca/openssl.cnf -revoke issued-leaf.pem -crl_reason keyCompromise
sudo openssl ca -config /etc/pki/ov-ca/openssl.cnf -gencrl -out /var/www/acmeca.com/public_html/crl/ovsubca.crl
openssl crl -in /var/www/acmeca.com/public_html/crl/ovsubca.crl -noout -text | grep -A2 "Serial Number"
openssl verify -crl_check -CAfile <(cat /etc/pki/root-ca/certs/root-ca.cert.pem \
  /etc/pki/ov-ca/certs/ov-ca.cert.pem /var/www/acmeca.com/public_html/crl/ovsubca.crl) issued-leaf.pem
```

---

## Phase 12 — Configure HTTPS and TLS (§19)

Install the leaf+chain for each vhost and enable TLS 1.2/1.3 only:
```bash
sudo a2enmod ssl rewrite headers
sudo mkdir -p /etc/apache2/ssl
sudo cp server-leaf.cert.pem server-chain.pem server.key.pem /etc/apache2/ssl/
sudo chmod 400 /etc/apache2/ssl/server.key.pem
```
`/etc/apache2/sites-available/acmeca.com-ssl.conf`:
```apache
<VirtualHost *:443>
    ServerName acmeca.com
    ServerAlias www.acmeca.com
    DocumentRoot /var/www/acmeca.com/public_html
    SSLEngine on
    SSLCertificateFile      /etc/apache2/ssl/server-leaf.cert.pem
    SSLCertificateChainFile /etc/apache2/ssl/server-chain.pem
    SSLCertificateKeyFile   /etc/apache2/ssl/server.key.pem
    SSLProtocol -all +TLSv1.2 +TLSv1.3
</VirtualHost>
```
HTTP vhost — redirect all except `/crl/`:
```apache
<VirtualHost *:80>
    ServerName acmeca.com
    ServerAlias www.acmeca.com
    RewriteEngine On
    RewriteCond %{REQUEST_URI} !^/crl/
    RewriteRule ^(.*)$ https://%{HTTP_HOST}$1 [R=301,L]
    DocumentRoot /var/www/acmeca.com/public_html
</VirtualHost>
```
Repeat the same pattern for `apply.acmeca.com` with its own certificate. Then:
```bash
sudo a2ensite acmeca.com-ssl apply.acmeca.com-ssl
sudo apache2ctl configtest
sudo systemctl reload apache2
sudo ufw allow 443/tcp     # if not already done
```
Verify:
```bash
openssl s_client -connect acmeca.com:443 -servername acmeca.com -tls1_2 </dev/null | grep -i "protocol\|cipher"
openssl s_client -connect acmeca.com:443 -servername acmeca.com -tls1_3 </dev/null | grep -i "protocol\|cipher"
```

---

## Phase 13 — Configure Client Trust (§20)

Only the **Root CA** cert is imported. Copy it to the Client, then:
```bash
# Debian/Ubuntu system trust store
sudo cp root-ca.cert.pem /usr/local/share/ca-certificates/acmeca-root-ca.crt
sudo update-ca-certificates
```
For a browser (e.g. Firefox), import via Settings → Privacy & Security → Certificates →
View Certificates → Authorities → Import, selecting only the Root CA file. Then verify:
```bash
curl -v https://acmeca.com 2>&1 | grep -i "SSL certificate\|subject\|issuer"
curl -v https://apply.acmeca.com 2>&1 | grep -i "subject\|issuer"
curl -I http://acmeca.com   # confirm redirect to https
```

---

## Phase 14 — Mandatory Nmap Scanning (§21)

Run **from the Client**, against your own Server:
```bash
sudo apt install nmap -y
sudo nmap -sS -sV <server-ip>            | tee nmap-syn-version.txt
sudo nmap -sS -p- <server-ip>            | tee nmap-allports.txt
sudo nmap -sU -p 53 <server-ip>          | tee nmap-udp53.txt
sudo nmap -O <server-ip>                 | tee nmap-osdetect.txt
```
Expected: 80/443/3000 open, 53 open (tcp+udp), 22 closed/filtered. Save all four outputs
and write an explanation of each result for the report.

---

## Phase 15 — Mandatory Wireshark Analysis (§22)

```bash
sudo apt install wireshark -y
sudo usermod -aG wireshark $USER   # log out/in after this
```
Capture on the Host-only interface, then in a second terminal generate traffic:
```bash
# start capture (GUI, or CLI):
sudo tshark -i enp0s8 -w https-capture.pcapng &

curl -o testfile.pdf https://acmeca.com/some-test-file.pdf
```
Stop the capture, then apply filters in Wireshark (or tshark):
```bash
tshark -r https-capture.pcapng -Y "tcp.flags.syn==1"
tshark -r https-capture.pcapng -Y "tls.handshake.type==1"   # ClientHello
tshark -r https-capture.pcapng -Y "tls.handshake.type==2"   # ServerHello
tshark -r https-capture.pcapng -Y "tls"
```
Document: SYN, SYN/ACK, ACK; ClientHello/ServerHello; offered vs. selected cipher suite;
negotiated TLS version; certificate message (capture a TLS 1.2 session separately if it's
not visible in your TLS 1.3 capture, and explain why); first encrypted record; src/dst
IP:port pairs.

---

## Phase 16 — Controlled SYN-Flood Demonstration (§23)

**Hard limits: no `--flood`, max 10 seconds per test, ~100 pps.** Run from the Client.
```bash
sudo apt install hping3 -y

# Non-spoofed test
sudo hping3 -S -p 443 --interval u10000 -c 1000 <server-ip>

# Spoofed-source test (use an UNUSED address in your own Host-only subnet)
sudo hping3 -S -p 443 --interval u10000 -c 1000 -a <unused-host-only-ip> <server-ip>
```
While these run, capture with Wireshark/tshark and watch:
```bash
sudo tail -f /var/log/suricata/fast.log
```
Save the IDS alerts (with timestamps), the pcap of both tests, and write a short
explanation of why a spoofed source address doesn't reliably identify the real attacker.

---

## Phase 17 — Rehearse the 20-Minute Random-Domain Task (§26)

Practice this full sequence end-to-end with a placeholder domain, e.g. `testdomain.com`:
```bash
# 1–2: key + CSR with both SANs (bare + wildcard)
openssl genrsa -out testdomain.key.pem 2048
openssl req -new -key testdomain.key.pem -out testdomain.csr.pem \
  -subj "/CN=testdomain.com" \
  -addext "subjectAltName=DNS:testdomain.com,DNS:*.testdomain.com"

# 4–5: DNS records (add A + PTR to your zone files, then)
sudo named-checkzone acmeca.com /etc/bind/zones/db.acmeca.com
sudo systemctl reload bind9
dig @<server-ip> testdomain.com

# 6–7: submit via the web app, admin issues via OV SubCA
sudo openssl ca -config /etc/pki/ov-ca/openssl.cnf -extensions server_cert \
  -days 365 -in testdomain.csr.pem -out testdomain-leaf.cert.pem

# 8–10: new vhost + install cert + enable HTTPS (same pattern as Phase 12)
# 11: verify from Client
curl -v https://testdomain.com
```
Time yourself — the real task is 20 minutes for the whole group.

---

## Phase 18 — Report, Evidence, and Viva Prep (§27–§28)

Before booking the viva, walk the **Full Verification Checklist (§25)** literally line by
line on a live system. Collect these artifacts for the report/appendices:
```bash
# Configs & evidence to archive (never include private keys!)
mkdir -p ~/submission/{configs,certs,logs,captures,scans}
sudo cp /etc/apache2/sites-available/*.conf ~/submission/configs/
sudo cp -r /etc/bind/zones ~/submission/configs/
sudo cp /etc/ssh/sshd_config ~/submission/configs/
sudo ufw status numbered > ~/submission/configs/ufw-status.txt
sudo cp /etc/suricata/rules/local.rules ~/submission/configs/
cp /etc/pki/*/certs/*.pem ~/submission/certs/       # public certs only
cp /var/www/acmeca.com/public_html/crl/*.crl ~/submission/certs/
cp nmap-*.txt ~/submission/scans/
cp *.pcapng ~/submission/captures/
sudo cp /var/log/suricata/fast.log ~/submission/logs/
journalctl -u ssh --since "1 hour ago" > ~/submission/logs/ssh-auth.txt
```
Double-check **none** of these are present anywhere in the submission:
```bash
grep -rl "BEGIN.*PRIVATE KEY" ~/submission || echo "clean: no private keys found"
```
Then rehearse explaining, unaided, for each phase: *what it does, why it's needed, where
the config lives, and what the log/output means* — that's what the individual viva tests.

---

### Quick command reference cheat-sheet

| Task | Command |
|---|---|
| Check a service | `sudo systemctl status <service>` |
| Reload Apache safely | `sudo apache2ctl configtest && sudo systemctl reload apache2` |
| Check BIND zone syntax | `sudo named-checkzone <zone> <file>` |
| Forward/reverse DNS | `dig <name>` / `dig -x <ip>` |
| SSH log | `journalctl -u ssh` |
| Firewall status | `sudo ufw status verbose` |
| IDS alerts | `sudo tail -f /var/log/suricata/fast.log` |
| Inspect a cert | `openssl x509 -in file.pem -noout -text` |
| Verify a chain | `openssl verify -CAfile root.pem -untrusted subca.pem leaf.pem` |
| TLS handshake test | `openssl s_client -connect host:443 -servername host -tls1_3` |
| Full port scan | `sudo nmap -sS -p- <ip>` |
