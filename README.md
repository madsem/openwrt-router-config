# OpenWRT Router Config

This configuration was tested on a Linksys WRT3200ACM.

This config was made with Apple devices in mind, it uses a single VLAN so that Apple's `Bonjour` works without having to use something like `Avahi`.

So `AirPlay`, `Airprint` etc are all working out of the box.

### What will be created:
- 2 wireless networks: 1 5Ghz, and 1 2.4Ghz
- Static IP route and rule(s) to redirect any device inside network through Windscribe (or other) VPN via WireGuard.
- Second WireGuard VPN for Road Warrior access to home network rom abroad (Remote Internet sharing)
- SSH remote login (Keybased only)
- AWS Route53 DynDns.


# What you need:
- Fresh OpenWRT Router Install, tested with OpenWrt 19.07.4.
  - [Download OpenWrt 19.0.7 Factory Image](http://downloads.openwrt.org/releases/19.07.4/targets/mvebu/cortexa9/openwrt-19.07.4-mvebu-cortexa9-linksys_wrt3200acm-squashfs-factory.img)
- VPN subscription with provider that has WireGuard support.
- AWS Account and domain zone managed in Route53 for DynDNS.
- SSH key


# Installation on Fresh Router:

## Log into LuCi:
- Change the default password

Then make sure `System -> Administration -> SSH Access` has `Password authentication` & `Allow root logins with password` disabled.

SSH port should be set to a high port, like `22222`.

Interface: `unspecified`.

`System -> Administration -> SSH Keys`:  
Paste the public ssh key of the device(s) you want to use for SSH seesions in the `Add Key` field.
 ```bash
 cat ~/.ssh/id_rsa.pub
 ```

## Create AWS Resources For DynDNS
- Create a hosted zone for your domain in Route53, write down the Hosted zone ID.
- Create a user for programmatic access, write down the access key & secret.
- Attach a managed policy to the user
    - Fill in your hosted zone id
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "route53:ChangeResourceRecordSets",
            "Resource": "arn:aws:route53:::hostedzone/AWS_ROUTE_53_HOSTED_ZONE_ID"
        }
    ]
}
```

## Install the following packages:
```bash
opkg update
```

```bash
opkg install luci-app-ddns \
ddns-scripts \
ddns-scripts_route53-v1 \
wireguard \
luci-app-wireguard \
luci-app-advanced-reboot \
nano \
ipset
```


## Adjust OpenWrt Config Files:

### /etc/config/ddns:

This file can be completely replaced with below contents:
```bash
config ddns 'global'
	option ddns_dateformat '%F %R'
	option ddns_loglines '250'
	option upd_privateip '0'

config service 'route53_ddns'
	option enabled '1'
	option service_name 'route53-v1'
	option use_https '1'
	option lookup_host '<sub.mydomain.com>'
	option domain '<aws_route53_hosted_zone_id>'
	option username '<aws_access_key>'
	option password '<aws_secret_access_key'
	option ip_source 'network'
	option interface 'wan'
	option check_interval '5'
	option use_logfile '1'
```


### /etc/config/dhcp:

Configure DHCP for `lan` & `wan` like this:
```bash
config dhcp 'lan'
	option interface 'lan'
	option start '100'
	option limit '150'
	option leasetime '12h'
	option dhcpv6 'server'
	option ra 'server'
	option ra_management '1'

config dhcp 'wan'
	option interface 'wan'
	option ignore '1'
```

Now append the following config, one `host` per device that is supposed to have a static leased IP.

If the device is also supposed to be tunneled through VPN,
add `networkid` and set it to `wan_vpn`, `dhcp_option` with the VPN's DNS server(s) and `dhcpv6` disabled. This is to give the device it's own DNS and to prevent DNS leaks.

```bash
config host
	option mac '<MAC:ADDRESS>'
	option leasetime '900'
	option dns '1'
	option name 'apple-tv'
	option ip '192.168.1.192'
	option networkid 'wan_vpn'
	list dhcp_option '6,10.255.255.3'
	option dhcpv6 'disabled'

config host
	option mac '<MAC:ADDRESS>'
	option leasetime '900'
	option dns '1'
	option name 'printer'
	option ip '192.168.1.193'

config host
	option mac '<MAC:ADDRESS>'
	option leasetime '900'
	option dns '1'
	option name 'playstation'
	option ip '192.168.1.194'

config host
	option mac '<MAC:ADDRESS>'
	option leasetime '900'
	option dns '1'
	option name 'macbook'
	option ip '192.168.1.195'

config host
	option mac '<MAC:ADDRESS>'
	option leasetime '900'
	option dns '1'
	option name 'ipad'
	option ip '192.168.1.196'

config host
	option mac '<MAC:ADDRESS>'
	option leasetime '900'
	option dns '1'
	option name 'iphone'
	option ip '192.168.1.197'
```


### /etc/config/firewall:
Make sure the firewall rules for `lan`, `wan` are configured like below, and that `lan` is allowed to forward to `wan`.

```bash
config zone
	option name 'lan'
	list network 'lan'
	list network 'wg_rd_warrior'
	option input 'ACCEPT'
	option output 'ACCEPT'
	option forward 'ACCEPT'
	option mtu_fix '1'

config zone
	option name 'wan'
	list network 'wan'
	list network 'wan6'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	option masq '1'
	option mtu_fix '1'

config forwarding
	option src 'lan'
	option dest 'wan'
```

Now append the following rules to the config file.
Replace the `<MAC:ADDRESS>` portion in the rule that prevents IPV6 leaks, with the MAC address of the device you configured in `/etc/config/dhcp` to be tunneled through VPN.

If it's multiple devices, seperate mac addresses by white-space.
```bash
config rule
	option dest_port '22222'
	option src 'wan'
	option name 'Remote-SSH-22222'
	option dest 'lan'
	option target 'ACCEPT'
	list proto 'tcp'

config rule
	option src '*'
	option target 'ACCEPT'
	option proto 'udp'
	option dest_port '55555'
	option name 'Allow-Wireguard-Clients-Inbound'

config zone
	option name 'wan_vpn'
	option network 'wan_vpn'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	option mtu_fix '1'
	option masq '1'

config forwarding
	option src 'lan'
	option dest 'wan_vpn'

config rule
	option name 'Prevent-IPV6-Leak-VPN'
	option family 'ipv6'
	list proto 'all'
	option src 'lan'
	option dest 'wan'
	option target 'DROP'
	option src_mac '<MAC:ADDRESS> <MAC:ADDRESS> <MAC:ADDRESS>'
```


### /etc/config/network:

Configure `lan` and `wan` like below.
If you aren't using PPPoE, look up the correct config for your interface type.

```bash
config interface 'lan'
	option type 'bridge'
	option ifname 'eth0.1'
	option proto 'static'
	option ipaddr '192.168.1.1'
	option netmask '255.255.255.0'
	option ip6assign '60'
	list dns '1.1.1.1'
	list dns '1.0.0.1'
	list dns '2606:4700:4700::1111'
	list dns '2606:4700:4700::1001'

config interface 'wan'
	option ifname 'eth1.2'
	option proto 'pppoe'
	option username '<username>'
	option ipv6 'auto'
	list dns '1.1.1.1'
	list dns '1.0.0.1'
	list dns '2606:4700:4700::1111'
	list dns '2606:4700:4700::1001'
	option peerdns '0'
	option password '<password>'

config interface 'wan6'
	option ifname 'eth1.2'
	option proto 'dhcpv6'
	option reqprefix 'auto'
	option reqaddress 'try'
```

Now configure the WireGuard interfaces & Peers we will use for VPN, and Road Warrior access.

```bash
# create wireguard keyfiles on router
wg genkey | tee wg.key | wg pubkey > wg.pub
```

```bash
# private key
cat wg.key

# public key
cat wg.pub
```

#### Configure Wireguard on your Road Warrior Devices:
- Install WireGuard app on every device you want to be able to connect back home as a `Road Warrior`. 

- Use the public key `(wg.pub)` from above, in the `PublicKey` option of the `[Peer]` section in the Wireguard app on each device.

- Under the `Addresses` option on each device, enter the corresponding value from `list addresses` below.

- Now, enter your DynDNS domain with the `listen_port` configured on the `wg_rd_warrior` interface, into the WireGuard app on each device in the `endpoint` option. Like for example `sub.domain.com:55555`.

##### Example WireGuard App config on the Macbook Peer:
```config
[Interface]
PrivateKey = some_auto_generated_key_we_dont_need_it
Address = 10.20.20.2/32
DNS = 1.1.1.1, 1.0.0.1

[Peer]
PublicKey = <public_key_from_router_wireguard_wb.pub_file>
AllowedIPs = 0.0.0.0/0, ::/128
Endpoint = sub.domain.com:55555
PersistentKeepalive = 25
```

Save config on each device, then you can see the `Public key` for that new Road Warrior device, which you need to fill into each of the peers below `option public_key`.

#### Continue with Router Network Config:
Use the private key from router `(wg.key)` in the `wg_rd_warrior` interface below.
Fill in this public key from each Road Warrior device, in the `public_key` option of each peer in the config below.

Then append the adjusted config below into the network config file:
```bash
config interface 'wg_rd_warrior'
	option proto 'wireguard'
	option listen_port '55555'
	list addresses '10.20.20.1/24'
	option delegate '0'
	option private_key '<private-key-wg.key-file>'

# Peers:
config wireguard_wg_rd_warrior
	option public_key '<device-public-key>'
	option description 'Macbook'
	option persistent_keepalive '25'
	list allowed_ips '10.20.20.2/32'
	option route_allowed_ips '1'

config wireguard_wg_rd_warrior
	option public_key '<device-public-key>'
	option description 'iPhone'
	option persistent_keepalive '25'
	list allowed_ips '10.20.20.3/32'
	option route_allowed_ips '1'

config wireguard_wg_rd_warrior
	option public_key '<device-public-key>'
	option description 'iPad'
	list allowed_ips '10.20.20.4/32'
	option route_allowed_ips '1'
	option persistent_keepalive '25'
```
  

#### Second WireGuard Interface For VPN Tunnels:

Now we configure the VPN used to tunnel devices in our network. (Here we come, Netflix US)  
Run this command to add a permanent entry to our main routing table:
```bash
cat << 'EOF' >> /etc/iproute2/rt_tables
10      vpn
EOF
```

Now get a WireGuard config file from your VPN provider, add the config from that file to the config below.

```bash
config interface 'wan_vpn'
	option proto 'wireguard'
	list addresses '<Address_from_conf_file>'
	option delegate '0'
	option private_key '<PrivateKey_from_conf_file>'
	option listen_port '55556'

config wireguard_wan_vpn
	option public_key '<PublicKey_from_conf_file>'
	option description 'Some Identifying Name'
	option persistent_keepalive '25'
	option endpoint_port '<Port_from_conf_file>'
	option preshared_key '<PresharedKey_from_conf_file_or_empty>'
	option endpoint_host '<Host_from_conf_file>'
	list allowed_ips '0.0.0.0/0'

config rule
	option in 'lan'
    # This must be the IP(s) of the device(s) tunneled through VPN, in CIDR notation
	option src '192.168.1.192/32'
	option lookup 'vpn'

config route
	option interface 'wan_vpn'
	option target '0.0.0.0'
	option netmask '0.0.0.0'
	option metric '10'
	option table 'vpn'
	option mtu '1420'
    # Here again, add the DNS of your VPN provider
	list dns '10.255.255.3'
	option dhcpv6 'disabled'
```


### /etc/config/wireless:

Now just replace the contents of the wireless config with the below, after you configured the name & password for your two wireless networks (2.4GHz and 5GHz):

(If you are not from Germany, also enter your country ISO code, and select a valid channel for your country).
```bash
config wifi-device 'radio0'
	option type 'mac80211'
	option hwmode '11a'
	option path 'soc/soc:pcie/pci0000:00/0000:00:01.0/0000:01:00.0'
	option htmode 'VHT80'
	option channel '161'
	option legacy_rates '0'
	option country 'DE'

config wifi-iface 'default_radio0'
	option device 'radio0'
	option network 'lan'
	option mode 'ap'
	option wpa_disable_eapol_key_retries '1'
	option key '<YOUR_PASS>'
	option ssid '<SOME_SSID_NAME>'
	option encryption 'psk2+ccmp'

config wifi-device 'radio1'
	option type 'mac80211'
	option hwmode '11g'
	option path 'soc/soc:pcie/pci0000:00/0000:00:02.0/0000:02:00.0'
	option channel '1'
	option legacy_rates '0'
	option htmode 'HT40'
	option country 'DE'

config wifi-iface 'default_radio1'
	option device 'radio1'
	option network 'lan'
	option mode 'ap'
	option wpa_disable_eapol_key_retries '1'
	option key '<YOUR_PASS>'
	option ssid '<SOME_SSID_NAME>'
	option encryption 'psk2+ccmp'

# radio2 on wrt3200acm should be disabled, it causes problems.
config wifi-device 'radio2'
	option type 'mac80211'
	option channel '36'
	option hwmode '11a'
	option path 'platform/soc/soc:internal-regs/f10d8000.sdhci/mmc_host/mmc0/mmc0:0001/mmc0:0001:1'
	option htmode 'VHT80'
	option disabled '1'
```

**Reboot the router**
```bash
$ reboot
```

Everything should be working now!

(If both router partitions should be flashed with the same OS and settings, export the config files now via `LuCi`. Then flash the router a second time and import the config files. Now both partitions should have the same OS and the same settings.)

# SSH Tunnel To Use LuCi Web Interface Securely From Remote Location:
```bash
ssh root@<router-dyndns.com> -p <port> -i <key file> -L 8080:localhost:80
```

Open [http://localhost:8080](http://localhost:8080) in browser and LuCi UI will appear.



