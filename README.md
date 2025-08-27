# OpenVPN Server Setup on Ubuntu (with Password-Protected CA)

A secure, step-by-step guide to install OpenVPN on Ubuntu 20.04/22.04 and generate certificates with a password-protected Certificate Authority (CA) using Easy-RSA.

## 📖 What is OpenVPN <br>
is an open-source VPN (Virtual Private Network) solution that uses SSL/TLS for secure communication. It's highly flexible and can be configured for a wide range of network topologies, including point-to-point and site-to-site connections. Its use of the OpenSSL library for encryption and authentication makes it a popular choice for those seeking a secure and customizable VPN option.

## 📖 Key components <br>
Server: A machine running the OpenVPN server software that accepts incoming connections.
Client: A device (e.g., a computer, phone) running the OpenVPN client software to connect to the server.
Certificates and Keys: OpenVPN relies on a public key infrastructure (PKI) to authenticate clients and servers. This involves a Certificate Authority (CA) that signs and verifies digital certificates.

Configuration Files: These files, typically with a .ovpn extension, contain all the necessary settings for the client to connect to the server, including server address, port, and security protocols.

## 📖 How it Works <br>
The process of establishing an OpenVPN connection involves a handshake between the client and server. The client initiates the connection, and the server responds. During this exchange, they authenticate each other using their certificates and establish a secure, encrypted tunnel using the SSL/TLS protocol. Once the tunnel is established, all network traffic from the client is routed through it, appearing to originate from the server's network. This masks the client's actual IP address and location, providing anonymity and security.

OpenVPN can operate over both TCP (Transmission Control Protocol) and UDP (User Datagram Protocol). UDP is generally preferred for its speed, as it has less overhead, while TCP can be more reliable on unstable networks.

# 🧱 Requirements
Ubuntu 20.04 or 22.04 Server with root access <br>
Public IP address or domain <br>
Port 1194 open on firewall (UDP) - Forwarding <br>

## 📁 openvpn-setup/
├── README.md <br>
├── server.conf <br>
├── client1.ovpn <br>

🛠️ Step 1: Install OpenVPN and Easy-RSA
```
sudo apt update
sudo apt install openvpn easy-rsa -y
```
📁 Step 2: Set Up the PKI (Public Key Infrastructure)
```
make-cadir ~/openvpn-ca
cd ~/openvpn-ca

Initialize Easy-RSA:
./easyrsa init-pki
```
🔐 Step 3: Build the CA (Password-Protected)
```
./easyrsa build-ca
```
Enter a strong passphrase for the CA key <br>
Enter the Common Name (MyOpenVPN CA) <br>

🔑 Step 4: Generate the Server Certificate & Key
```
./easyrsa gen-req server
```
Enter a name (openserver) <br>
Enter a passphrase for the server key <br>

Now sign the server certificate:
```
./easyrsa sign-req server server
```
Confirm when asked and enter the CA passphrase <br>

🔑 Step 5: Generate Diffie-Hellman Parameters
```
./easyrsa gen-dh
```
🧾 Step 6: Generate TLS Authentication Key (recommaned)
```
openvpn --genkey --secret ta.key
```
🧳 Step 7: Move Server Files into Place
```
sudo cp pki/ca.crt pki/issued/server.crt pki/private/server.key pki/dh.pem ta.key /etc/openvpn/
```
⚙️ 8. Create OpenVPN Server Config
```
sudo nano /etc/openvpn/server.conf

Paste in to file and save

port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
tls-auth ta.key 0
topology subnet
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
keepalive 10 120
cipher AES-256-CBC
persist-key
persist-tun
status openvpn-status.log
verb 3
explicit-exit-notify 1
```

🛡️ 9. Enable IP Forwarding
```
sudo nano /etc/sysctl.conf

Uncomment net.ipv4.ip_forward=1

and apply via commnand sudo sysctl -p
```

🔥 10. Configure Firewall (UFW)
```
sudo nano /etc/ufw/before.rules

Before *filter, add:

*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT

sudo ufw allow 1194/udp
sudo ufw allow OpenSSH
sudo ufw enable
```
▶️ 11. Start and Enable OpenVPN
```
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
```
👤 12. Create a VPN Client Certificate (with password)
```
cd ~/openvpn-ca
./easyrsa gen-req client1
./easyrsa sign-req client client1
```
📦 13. Create All-in-One .ovpn Client File
```
nano client1.ovpn

client
dev tun
proto udp
remote your.server.com 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-CBC
verb 3
key-direction 1

<ca>
# Paste contents of ca.crt
</ca>

<cert>
# Paste contents of client1.crt
</cert>

<key>
# Paste contents of client1.key
</key>

<tls-auth>
# Paste contents of ta.key
</tls-auth>
```

