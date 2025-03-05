# IKEv2 VPN Server Setup on Ubuntu for macOS Clients

## Overview
This guide walks through setting up an **IKEv2 VPN server** on an **Ubuntu server** using **strongSwan** for macOS clients.

## Prerequisites
- **Ubuntu 20.04+** server with root access
- **Public IP address**
- **Domain name (optional)** for easier certificate management
- **macOS client** for testing

---

## Step 1: Install strongSwan
Update and install strongSwan:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y strongswan strongswan-pki libstrongswan-extra-plugins libcharon-extauth-plugins libstrongswan-extra-plugins
```

---

## Step 2: Generate Certificates
IKEv2 requires SSL certificates for authentication.

### Create Directories
```bash
mkdir -p ~/pki/{cacerts,certs,private}
chmod 700 ~/pki
```

### Generate a CA Certificate
```bash
ipsec pki --gen --outform pem > ~/pki/private/ca-key.pem
ipsec pki --self --ca --lifetime 3650 --in ~/pki/private/ca-key.pem --type rsa --dn "CN=IKEv2 VPN CA" --outform pem > ~/pki/cacerts/ca-cert.pem
```

### Generate a Server Certificate
Replace `vpn.example.com` with your actual domain or server IP.
```bash
ipsec pki --gen --outform pem > ~/pki/private/server-key.pem
ipsec pki --pub --in ~/pki/private/server-key.pem | ipsec pki --issue --lifetime 1825 --cacert ~/pki/cacerts/ca-cert.pem --cakey ~/pki/private/ca-key.pem --dn "CN=vpn.example.com" --san="vpn.example.com" --flag serverAuth --flag ikeIntermediate --outform pem > ~/pki/certs/server-cert.pem
```

---

## Step 3: Configure strongSwan
Edit the configuration file:
```bash
sudo nano /etc/ipsec.conf
```

Replace with:
```ini
config setup
    charondebug="ike 2, knl 2, cfg 2"

conn ikev2-vpn
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
    dpdaction=clear
    dpddelay=300s
    rekey=no
    left=%any
    leftid=vpn.example.com
    leftcert=server-cert.pem
    leftsendcert=always
    leftsubnet=0.0.0.0/0
    right=%any
    rightdns=8.8.8.8,8.8.4.4
    rightsourceip=10.10.10.0/24
    rightsendcert=never
    ike=aes256gcm16-sha256-ecp256,aes256-sha256-modp2048!
    esp=aes256gcm16-sha256,aes256-sha256!
    rightauth=eap-mschapv2
    eap_identity=%any
```

---

## Step 4: Configure Secrets
Edit:
```bash
sudo nano /etc/ipsec.secrets
```

Add:
```bash
: RSA "server-key.pem"
username : EAP "password"
```
Replace `username` and `password` with actual credentials.

---

## Step 5: Enable IP Forwarding
Edit:
```bash
sudo nano /etc/sysctl.conf
```
Uncomment:
```ini
net.ipv4.ip_forward=1
```
Apply changes:
```bash
sudo sysctl -p
```

---

## Step 6: Configure Firewall
If UFW is installed, run:
```bash
sudo ufw allow 500/udp
sudo ufw allow 4500/udp
sudo ufw enable
```

If using iptables:
```bash
sudo iptables -A INPUT -p udp --dport 500 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 4500 -j ACCEPT
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -s 10.10.10.0/24 -j ACCEPT
sudo iptables-save > /etc/iptables.rules
```
To apply rules on boot, add this to `/etc/rc.local` **before `exit 0`**:
```bash
iptables-restore < /etc/iptables.rules
```

---

## Step 7: Restart strongSwan
```bash
sudo systemctl restart strongswan
sudo systemctl enable strongswan
```

---

## Step 8: Configure macOS Client
1. Open **System Settings > VPN > Add VPN Configuration**.
2. Choose **IKEv2** as the type.
3. Enter:
   - **Server Address**: `vpn.example.com`
   - **Remote ID**: `vpn.example.com`
   - **Local ID**: *(leave empty)*
   - **Username & Password**: (from `/etc/ipsec.secrets`)
4. Click **Connect**.

---

## Step 9: Verify Connection
Run:
```bash
sudo ipsec statusall
sudo journalctl -u strongswan -f
```
If successful, your macOS client should be connected to the VPN.

---

## Troubleshooting
### Check Logs
```bash
sudo journalctl -u strongswan -f
```
Look for errors related to **encryption algorithms** or **authentication failures**.

### Verify Firewall Rules
```bash
sudo iptables -L -v -n
```
Ensure **UDP ports 500/4500 are open**.

### Restart VPN Server
```bash
sudo systemctl restart strongswan
```

---

## Conclusion
This guide sets up a **secure IKEv2 VPN server on Ubuntu** for **macOS clients**. If you encounter issues, check logs and verify your configuration.

Enjoy your **secure VPN connection**! 🚀

