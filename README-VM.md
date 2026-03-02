# Signal Replacer using a Virtual Machine on Mac OS X (Apple Silicon)

A documentation on how to create a special tool for the purpose of developing HbbTV applications.
The Virtual Machine we are creating is essentialy a small router with ability to capture network traffic on port 80 and redirect to another port, host or domain.

Prerequisites:
1. An Apple Mac with some kind of virtualization software. In this tutorial I will be using UTM.
2. Some time :)
3. A TV set with HbbTV Support :)

This documentation will be based on a setting up this solution on a Virtual Machine.

Let's start:

**INITIAL SETUP**

1. Prepare the hardware, setup the Virtual Machine, use 2 or 4 cores and 64-128GB of disk space, 4 - 16GB of RAM.
2. Install the operating system. For the purpose of this manual I will be using an Debian arm64 image 13.3.0 version.
3. Go throught the setup of operating using default options. For the purpose of this manual I will use user named **oper**.
4. Setup the UTM to use Bridge Network.
5. From the interface adapter list use: e1000.
6. Attach the USB WiFi adapter of your choosing to your computer and attach it to the Virtual Machine. I will be using for the purpose of this tutorial TP-Link T4U.
7. After completing the install let's get to setting up the development environemnt.

This completes the INTIAL SETUP.

Now let's prepare Virtual Image for our specific use case :)

**SOFTWARE PREPARATION**

1. Switch to the root user (su -)
2. Install net-tools and check the Virtual Machine IP Address. The command is: apt install net-tools
3. Enable the ssh service: systemctl enable ssh
4. Start openssh server: systemctl start ssh.
5. Connect to the machine using ssh.
6. Run: apt update
7. Run: apt upgrade
8. Wait for it to complete
9. Reboot the device if necessary.
10. Switch to root again.
11. Let's install required software:
12. apt install pipx dnsmasq bridge-utils hostapd ufw netplan.io
13. Exit root: exit
14. Let's install mitmproxy: pipx install mitmproxy
15. Let's add pyyaml to mitmproxy: pipx inject mitmproxy pyyaml
16. Create the following file: touch redirects.yaml
17. Create the following file: touch redirect-request.py
18. Edit the redirect-request.py and paster the following code:

```
from mitmproxy import http
import yaml
import io

REDIRECTS_FILE = '/home/oper/redirects.yaml'

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
21. Let's setup the Wireless USB adapter.
22. Let's install the required packages: apt install build-essential dkms git iw bc linux-headers-arm64
23. Let's clone the driver: git clone https://github.com/morrownr/88x2bu-20210702.git
24. Let's enter the driver directory: cd 88x2bu-20210702/
25. Let's build the driver: ./install-driver.sh
26. After install is complete you will be prompted with two questions: say "n" to both.
27. Load the driver: modprobe 88x2bu
28. Now after issuing the command: "ifconfig -a" you should see the the interface starting with name like: wlxe4fac4fb81d8 (where the e4fac4fb81d8 is mac address of the card).
30. Clear the /etc/dnsmasq.conf file: truncate -s 0 /etc/dnsmasq.conf
31. Edit the dnsmasq.conf: nano /etc/dnsmasq.conf
32. Paste the following initial settings (adjust the network settings to your preference):

```
interface=br0
domain-needed
bogus-priv
no-resolv
server=8.8.8.8
server=8.8.4.4
cache-size=1000
dhcp-range=192.168.65.2,192.168.65.240,255.255.255.0,72h
#address=/hbbtv-dev-app.localdomain/192.168.65.2/
local=/localdomain/
domain=localdomain
```

22. Enable the dnsmasq as a service: systemctl enable dnsmasq
23. Let's setup the network bridge.
24. Create the following file: nano /etc/netplan/10-br0-network.yaml
25. Paste the contents:

```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s1:
      dhcp4: yes
      dhcp6: yes
    wlxe4fac4fb81d8:
      dhcp4: no
      dhcp6: no
  bridges:
    br0:
      dhcp4: no
      dhcp6: no
      addresses: [ 192.168.65.1/24 ]
      parameters:
        stp: false
        forward-delay: 0
```

27. Save the file.
28. Execut the following command: sudo netplan apply
29. Edit the /etc/hostapd/hostapd.conf: nano /etc/hostapd/hostapd.conf
30. Paste the following contents to setup an Access Point (the network will be named: SignalReplacer with password: MyCustomPassword123, but you can change to anything you want) and remember to also change the country code to your country equivalent:

```
logger_syslog=-1
logger_syslog_level=2
logger_stdout=-1
logger_stdout_level=2

interface=wlxe4fac4fb81d8
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

26. Save the file.
27. Setup hostapd to auto start with the system: systemctl unmask hostapd && systemctl enable hostapd
28. Let's setup UFW now.
29. Let's edit the defaults file for UFW: nano /etc/default/ufw
30. Now paste the following settings:

```
# /etc/default/ufw
#

# Set to yes to apply rules to support IPv6 (no means only IPv6 on loopback
# accepted). You will need to 'disable' and then 'enable' the firewall for
# the changes to take affect.
IPV6=yes

# Set the default input policy to ACCEPT, DROP, or REJECT. Please note that if
# you change this you will most likely want to adjust your rules.
**DEFAULT_INPUT_POLICY="ACCEPT"**

# Set the default output policy to ACCEPT, DROP, or REJECT. Please note that if
# you change this you will most likely want to adjust your rules.
DEFAULT_OUTPUT_POLICY="ACCEPT"

# Set the default forward policy to ACCEPT, DROP or REJECT.  Please note that
# if you change this you will most likely want to adjust your rules
**DEFAULT_FORWARD_POLICY="ACCEPT"**

# Set the default application policy to ACCEPT, DROP, REJECT or SKIP. Please
# note that setting this to ACCEPT may be a security risk. See 'man ufw' for
# details
DEFAULT_APPLICATION_POLICY="SKIP"

# By default, ufw only touches its own chains. Set this to 'yes' to have ufw
# manage the built-in chains too. Warning: setting this to 'yes' will break
# non-ufw managed firewall rules
**MANAGE_BUILTINS=yes**

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

31. Let's edit the ufw sysctl.conf: nano /etc/ufw/sysctl.conf
32. Let's enable ipv4 and ipv6 forwarding by uncommenting the following lines:

```
net/ipv4/ip_forward=1
net/ipv6/conf/default/forwarding=1
net/ipv6/conf/all/forwarding=1
```

33. Let's setup the rules for our firewall to capture HTTP traffic.
34. Edit the before.rules file: nano /etc/ufw/before.rules
35. And paste after last COMMIT the following code:

```
# NAT
*nat

:PREROUTING ACCEPT [0:0]
-A PREROUTING -i br0 -p tcp --dport 80 -j REDIRECT --to-port 8080

:POSTROUTING ACCEPT [0:0]

-A POSTROUTING -o enp0s1 -j MASQUERADE

COMMIT
```

36. Apply the changes by issuing the following command: ufw enable && ufw reload
37. We are ready.
38. Reboot and let's start developing applications.

**USAGE**

1. Connect to the SignalReplacer WiFi Network your laptop and TV set.
2. Enter in the web browser on the computer: http://192.168.65.1:8081 (the password is mitmproxy if you haven't changed it earlier).
3. Switch channels on the TV set with HbbTV enabled and save first links with http:// for the channels you want to replace with a development application.
4. Edit the redirects.yaml in the /home/oper/redirects.yaml on the Raspberry PI 5 and add entries in the following format:

```
http://[some link you acquired]: http://your_computers_local_network_ip_address_with_the_application_using_http/
http://[some link you acquired]: https://your_computers_local_network_ip_address_with_the_application_using_https/
```

6. Enter the channel again.
7. Now you should see your HbbTV application.

Big thanks to the authors of mitmproxy, the UTM authors and maintaners of the Debian distrubution for making this project possible! :) 
