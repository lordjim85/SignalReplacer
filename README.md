# Signal Replacer
A documentation on how to create a special tool for the purpose of developing HbbTV applications

Prerequisites:
1. Some kind of computer with two ethernet ports or one ethernet port and WiFi capable of working as an Access-Point
2. Some time :)

This documentation will be based on Raspberry PI 5 but you can use any small form factor computer or any PC you have laying around.

Let's start:

INITIAL SETUP

1. Prepare the hardware, install the operating system.
2. In this case use an SD card with minimal size of 32-64GB.
3. Download the Raspberry PI Imager: https://www.raspberrypi.com/software/ for your preffered platform.
4. We will be using the "Lite" variant of Raspberry PI OS. You can grab it from here: https://www.raspberrypi.com/software/operating-systems/
5. Prepare the SD Card using Raspberry PI Imager.
6. Insert the card to Raspberry PI 5.
7. Boot the Raspberry PI 5.
8. Setup an user name and a password for the user.
9. We will user username "oper" for the purpose of this documentation.
10. Login using credentials from above.
11. Connect the ethernet in the Raspberry PI 5.
12. Attach the second USB based ethernet card.

This completes the INTIAL SETUP.

Now let's prepare the Raspberry PI 5 for our specific use case :)

SOFTWARE PREPARATION

1. Switch to the root user (sudo -s -H)
2. Start openssh server: systemctl ssh start
3. Enable the ssh service: systemctl enable ssh
4. Run: apt update
5. Run: apt upgrade
6. Wait for it to complete
7. Reboot the device if necessary.
8. Switch to root again.
9. Let's install required software:
10. apt install pipx dnsmasq bridge-utils hostapd ufw
11. Exit root: exit
12. Let's install mitmproxy: pipx install mitmproxy
13. Create the following file: touch redirects.yaml
14. Create the following file: touch redirect-request.py
15. Edit the redirect-request.py and paster the following code:

```
from mitmproxy import http
import yaml
import io

REDIRECTS_FILE = './redirects.yaml'

def request(flow: http.HTTPFlow) -> None:
    with open(REDIRECTS_FILE) as config_file:
        redirects = yaml.safe_load(config_file)

    if redirects is not None:
        for patternURL, redirectURL in redirects.items():
            if flow.request.pretty_url == patternURL:
                flow.response = http.Response.make(302, b'', http.Headers(Location=str(redirectURL), Content_Length='0'))
                if flow.request.method == "GET":
                   flow.response.headers['Content-Type'] = "application/vnd.hbbtv.xhtml+xml"
```

15. We will edit the redirects.yaml after setting up mitmproxy.
16. Switch back to root.
17. Create the following file for the purpose of running the mimtproxy as a service: nano /etc/systemd/system/mitmproxy.service
18. Paste the following contents:

```
[Unit]
Description=mitmweb service
After=network-online.target

[Service]
Type=simple
User=oper
Group=oper
ExecStart=/home/oper/.local/bin/mitmweb --mode transparent --showhost --web-port 8081 --web-host 0.0.0.0 --no-web-open-browser --listen-host 0.0.0.0 --http2 --listen-port 8080 -s /home/oper/redirect-request.py --set ignore_hosts='google.com' --set web_password='mitmproxy'
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```

18. Save the file.
19. The password to the mitmproxy web interface will be: mitmproxy
20. Enable the service: systemctl enable mitmproxy
21. Edit the dnsmasq.conf: nano /etc/dnsmasq.conf
22. Clear the /etc/dnsmasq.conf file (truncate -s 0 /etc/dnsmasq.conf) and paste the following initial settings (adjust the network settings to your preference):

```
interface=br0
domain-needed
bogus-priv
no-resolv
server=8.8.8.8
server=8.8.4.4
cache-size=1000
dhcp-range=192.168.65.2,192.168.65.240,255.255.255.0,72h
address=/hbbtv-dev-app.localdomain/
local=/localdomain/
domain=localdomain
```

22. Enable the dnsmasq as a service: systemctl enable dnsmasq
23. Let's setup the network bridge.
24. We will use the USB Ethernet adapter with the WiFi built-in to the Raspberry PI 5 to create the bridge.
25. Create the following file: nano /etc/netplan/10-br0-network.yaml
26. Paste the contents:

```
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      dhcp6: no
    eth1:
      dhcp4: yes
      dhcp6: yes
    wlan0:
      dhcp4: no
      dhcp6: no
  bridges:
    br0:
      dhcp4: no
      dhcp6: no
      addresses: [ 192.168.65.1/24 ]
      interfaces: [ eth0 ]
      parameters:
        stp: false
        forward-delay: 0
```

27. Save the file.
28. Edit the /etc/hostapd/hostapd.conf: nano /etc/hostapd/hostapd.conf
29. Paste the following contents to setup an Access Point (the network will be named: SignalReplacer with password: MyCustomPassword123, but you can change to anything you want) and remember to also change the country code to your country equivalent:

```
logger_syslog=-1
logger_syslog_level=2
logger_stdout=-1
logger_stdout_level=2

interface=wlan0
bridge=br0

driver=nl80211

hw_mode=g
country_code=PL
channel=11

ssid=SignalReplacer
ignore_broadcast_ssid=0

ieee80211n=1
ieee80211w=1

auth_algs=1
wpa=2
wpa_passphrase=MyCustomPassword123
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP
rsn_pairwise=CCMP

macaddr_acl=0
wmm_enabled=1
```

27. Save the file.
28. Setup hostapd to auto start with the system: systemctl unmask hostapd && systemctl enable hostapd
29. Run: raspi-config
30. Under the: 5. Localisation Options -> L4 Wlan Country -> Set the country Code
31. Let's setup UFW now.
32. Let's edit the defaults file for UFW: nano /etc/default/ufw
33. Now paste the following settings:

```
# /etc/default/ufw
#

# Set to yes to apply rules to support IPv6 (no means only IPv6 on loopback
# accepted). You will need to 'disable' and then 'enable' the firewall for
# the changes to take affect.
IPV6=yes

# Set the default input policy to ACCEPT, DROP, or REJECT. Please note that if
# you change this you will most likely want to adjust your rules.
DEFAULT_INPUT_POLICY="ACCEPT"

# Set the default output policy to ACCEPT, DROP, or REJECT. Please note that if
# you change this you will most likely want to adjust your rules.
DEFAULT_OUTPUT_POLICY="ACCEPT"

# Set the default forward policy to ACCEPT, DROP or REJECT.  Please note that
# if you change this you will most likely want to adjust your rules
DEFAULT_FORWARD_POLICY="ACCEPT"

# Set the default application policy to ACCEPT, DROP, REJECT or SKIP. Please
# note that setting this to ACCEPT may be a security risk. See 'man ufw' for
# details
DEFAULT_APPLICATION_POLICY="SKIP"

# By default, ufw only touches its own chains. Set this to 'yes' to have ufw
# manage the built-in chains too. Warning: setting this to 'yes' will break
# non-ufw managed firewall rules
MANAGE_BUILTINS=yes

#
# IPT backend
#
# only enable if using iptables backend
IPT_SYSCTL=/etc/ufw/sysctl.conf

# Extra connection tracking modules to load. IPT_MODULES should typically be
# empty for new installations and modules added only as needed. See
# 'CONNECTION HELPERS' from 'man ufw-framework' for details. Complete list can
# be found in net/netfilter/Kconfig of your kernel source. Some common modules:
# nf_conntrack_irc, nf_nat_irc: DCC (Direct Client to Client) support
# nf_conntrack_netbios_ns: NetBIOS (samba) client support
# nf_conntrack_pptp, nf_nat_pptp: PPTP over stateful firewall/NAT
# nf_conntrack_ftp, nf_nat_ftp: active FTP support
# nf_conntrack_tftp, nf_nat_tftp: TFTP support (server side)
# nf_conntrack_sane: sane support
IPT_MODULES=
```

30. Let's edit the ufw sysctl.conf: nano /etc/ufw/sysctl.conf
31. Let's enable ipv4 and ipv6 forwarding by uncommenting the following lines:

```
net/ipv4/ip_forward=1
net/ipv6/conf/default/forwarding=1
net/ipv6/conf/all/forwarding=1
```

32. Let's setup the rules for our firewall to capture HTTP traffic.
33. Edit the before.rules file: nano /etc/ufw/before.rules
34. And paste after last COMMIT the following code:

```
# NAT
*nat

:PREROUTING ACCEPT [0:0]
-A PREROUTING -i br0 -p tcp --dport 80 -j REDIRECT --to-port 8080

:POSTROUTING ACCEPT [0:0]

-A POSTROUTING -o eth1 -j MASQUERADE

COMMIT
```

32. Apply the changes by issuing the following command: ufw enable && ufw reload
33. We are ready.
34. Reboot and let's start developing applications.
