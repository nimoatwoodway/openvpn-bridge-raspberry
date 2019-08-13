# Create OpenVPN Server on RaspberryPI for remote Access of a Network

## Install Raspberry
Download Light Version of Raspain from here: https://www.raspberrypi.org/downloads/raspbian/

Follow installation guide here: https://www.raspberrypi.org/documentation/installation/installing-images/README.md

**ATTENTION**
When writing this tutorial I could not get work PiVPN with Raspian **Stretch** since installation failed without warning. Files where missing e.g. etc/openvpn/server.conf
Here is a fix for this: https://github.com/pivpn/pivpn/issues/713#issuecomment-481744434

To access the PI without monitor create ssh file on boot partition described here: https://hackernoon.com/raspberry-pi-headless-install-462ccabd75d0

On macOS remount the sd card and

```
touch /Volumes/boot/ssh
```

Boot up the PI and use mDNS for first acces (or a lan scanning tool)

```
ssh pi@raspberrypi.local
```
Default password: raspberry.
Set another password for safety.

```
passwd
```

Standard stuff I install on every new Installation + bridge-utils (needed later)

```
sudo apt-get install vim git screen bridge-utils
```

## Install PiVPN (OpenVPN)
Checkout Repo: https://github.com/pivpn/pivpn

```
sudo curl -L https://install.pivpn.io | bash
```
Follow instructions and reboot the PI.

## Config OpenVPN Server
Many thanks to the author of this article: https://www.toysdesk.com/2017/11/openvpn-bridge-mode-tap-with-raspberry-pi-for-chromecast/

I've mainly copy/pasted his stuff.

Create the brdige config file

```
sudo vim /etc/openvpn/openvpn-bridge
```
Paste this and adjust network settings:

```
#!/bin/sh

# Define Bridge Interface
br="br0"

# Define list of TAP interfaces to be bridged,
# for example tap="tap0 tap1 tap2".
tap="tap0"

# Define physical ethernet interface to be bridged
# with TAP interface(s) above.
eth="eth0"
eth_ip="192.168.33.36"
eth_netmask="255.255.255.0"
eth_broadcast="192.168.33.255"
eth_gateway="192.168.33.1"

case "$1" in
start)
for t in $tap; do
openvpn --mktun --dev $t
done

brctl addbr $br
brctl addif $br $eth

for t in $tap; do
brctl addif $br $t
done

for t in $tap; do
ifconfig $t 0.0.0.0 promisc up
done

sleep 10

ifconfig $eth 0.0.0.0 promisc up

sleep 5

ifconfig $br $eth_ip netmask $eth_netmask broadcast $eth_broadcast

sleep 2

route add default gw $eth_gateway
;;
stop)
ifconfig $br down
brctl delbr $br

for t in $tap; do
openvpn --rmtun --dev $t
done

ifconfig $eth $eth_ip netmask $eth_netmask broadcast $eth_broadcast

route add default gw $eth_gateway
;;
*)
echo "Usage: openvpn-bridge {start|stop}"
exit 1
;;
esac
exit 0
```
Make it executeable

```
sudo chmod 744 /etc/openvpn/openvpn-bridge
```
Then edit this following file, to add the script just created.

```
sudo vim /lib/systemd/system/openvpn@.service
```

Insert the following two lines after the line **WorkingDirectory=/etc/openvpn**

```
ExecStartPre=/etc/openvpn/openvpn-bridge start
ExecStopPost=/etc/openvpn/openvpn-bridge stop
```

Finally modify the file /etc/openvpn/server.conf with TAP instead TUN

```
sudo vim /etc/openvpn/server.conf
```

```
dev tap0
proto udp
port 1194
ca /etc/openvpn/easy-rsa/pki/ca.crt
cert /etc/openvpn/easy-rsa/pki/issued/server_*********.crt
key /etc/openvpn/easy-rsa/pki/private/server_*********.key
dh none
client-to-client
keepalive 1800 3600
remote-cert-tls client
tls-version-min 1.2
tls-crypt /etc/openvpn/easy-rsa/pki/ta.key
cipher AES-256-CBC
auth SHA256
user nobody
group nogroup
persist-key
#persist-tun
crl-verify /etc/openvpn/crl.pem
status /var/log/openvpn-status.log 20
status-version 3
syslog
verb 3
server-bridge 192.168.1.91 255.255.255.0 192.168.1.95 192.168.1.200
```
Adjust last line with your settings but leaf the cert and key param as is. First IP is the raspi and the last to the range of ips for the openvpn clients.

## Create OpenVPN config file for client

```
pivpn -a nopass
```

Change tun to tap in file

```
vim ~/ovpns/NAME.ovpn
```

Copy to your client and connect.
