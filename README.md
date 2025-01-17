#!/bin/bash
#
# Script for automatic setup of an IPsec VPN server on CentOS 9.
# Works on any dedicated server or virtual private server (VPS) with CentOS 9.
#
# DO NOT RUN THIS SCRIPT ON YOUR PC OR MAC!
#
# The latest version of this script is available at:
# https://github.com/hwdsl2/setup-ipsec-vpn
#

# =====================================================

# Define your own values for these variables
YOUR_USERNAME='vpnuser'
YOUR_PASSWORD='vpnpassword'
YOUR_PSK='your_pre_shared_key'

# =====================================================

export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
SYS_DT=$(date +%F-%T | tr ':' '_')

exiterr()  { echo "Error: $1" >&2; exit 1; }
conf_bk() { /bin/cp -f "$1" "$1.old-$SYS_DT" 2>/dev/null; }
bigecho() { echo; echo "## $1"; echo; }

# Check for root
if [ "$(id -u)" != 0 ]; then
  exiterr "Script must be run as root. Try 'sudo sh $0'"
fi

# Stop and disable firewalld
bigecho "Stopping and disabling firewalld..."
systemctl stop firewalld
systemctl disable firewalld

# Install required packages
bigecho "Installing required packages..."
dnf install -y epel-release

dnf install -y strongswan xl2tpd ppp lsof bind-utils net-tools iptables-services || exiterr "Failed to install packages."

# Enable IP forwarding
bigecho "Enabling IP forwarding..."
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p

# Set up IPsec configuration
bigecho "Setting up IPsec configuration..."
cat > /etc/ipsec.conf <<EOF
config setup
    uniqueids=no
    charondebug="ike 2, knl 2, cfg 2"

conn %default
    keyexchange=ikev1
    authby=secret
    ike=aes256-sha1-modp1024
    esp=aes256-sha1
    dpdaction=clear
    dpddelay=300s
    rekey=no
    left=%defaultroute
    leftsubnet=0.0.0.0/0
    leftfirewall=yes
    right=%any
    rightsourceip=10.0.0.0/24
    rightauth=psk

conn L2TP-PSK
    auto=add
    type=transport
    leftprotoport=17/1701
    rightprotoport=17/1701
EOF

# Set up IPsec secrets
cat > /etc/ipsec.secrets <<EOF
: PSK "$YOUR_PSK"
EOF

# Configure xl2tpd
bigecho "Configuring xl2tpd..."
cat > /etc/xl2tpd/xl2tpd.conf <<EOF
[global]
port = 1701

[lns default]
ip range = 10.0.0.10-10.0.0.100
local ip = 10.0.0.1
require chap = yes
refuse pap = yes
require authentication = yes
name = L2TP-Server
pppoptfile = /etc/ppp/options.xl2tpd
length bit = yes
EOF

# Set up PPP options for xl2tpd
cat > /etc/ppp/options.xl2tpd <<EOF
require-mschap-v2
refuse-mschap
auth
mtu 1280
mru 1280
proxyarp
lock
nobsdcomp
nodeflate
ms-dns 8.8.8.8
ms-dns 8.8.4.4
EOF

# Add VPN credentials
cat > /etc/ppp/chap-secrets <<EOF
"$YOUR_USERNAME" * "$YOUR_PASSWORD" *
EOF

# Update iptables rules
bigecho "Updating iptables rules..."
iptables -I INPUT -p udp --dport 500 -j ACCEPT
iptables -I INPUT -p udp --dport 4500 -j ACCEPT
iptables -I INPUT -p udp --dport 1701 -j ACCEPT
iptables -I FORWARD -s 10.0.0.0/24 -j ACCEPT
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

# Save iptables rules
service iptables save
systemctl restart iptables

# Restart and enable services
bigecho "Restarting and enabling services..."
systemctl enable strongswan
systemctl enable xl2tpd
systemctl restart strongswan
systemctl restart xl2tpd

bigecho "VPN setup completed."

cat <<EOF
================================================

IPsec VPN server is now ready for use!

Server IP: $(hostname -I | awk '{print $1}')
Username: $YOUR_USERNAME
Password: $YOUR_PASSWORD
Pre-shared key (PSK): $YOUR_PSK

Write these down. You'll need them to connect!

================================================
EOF
