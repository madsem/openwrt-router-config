# OpenWRT Router Config

I run this on a Linksys WRT3200ACM, and it works beautifully.

### What will be created:
- 2 unencrypted wireless ssids for direct ISP internet (5GHz + 2.4GHz)
- 1 encrypted VPN wireless ssid 5GHz (Windscribe with Wireguard or other VPN provider)
- WireGuard VPN for Road Warrior access to home network (Internet sharing from abroad)
- SSH remote login (Keybased only)
- AWS Route53 DynDns.


# Prerequisites:
- Fresh OpenWRT Router Install, tested with OpenWrt 19.07.4.
- VPN subscription with provider that has WireGuard support.
- AWS Account and domain zone managed in Route53.
- SSH key, doh
 

# SSH Tunnel To Use LuCi Web Interface Securely From Remote Location:
```bash
ssh root@<routerdns> -p <port> -i <key file> -L 8080:localhost:80
```

Open [http://localhost:8080](http://localhost:8080) in browser and LuCi UI will appear.


# Installation on Fresh Router:

Log into Luci at [http://192.168.1.1](http://192.168.1.1), or use the command line to install the following packages:
```bash
opkg update
opkg install luci-app-ddns /
ddns-scripts /
ddns-scripts_route53-v1 /
wireguard /
luci-app-wireguard /
ip-tiny /
kmod-ipt-nat6 /
ipset /
luci-app-advanced-reboot 
```

Now create a local project from this repo.
```bash
composer create-project my/openwrt path --repository-url=https://github.com/madsem/openwrt-router-config.git
```

Now all secrets have to be filled in:


Once all of the above is completed, run the following commands:
```bash
# Create tar archive from etc dir.
tar -czf backup.tar.gz etc

# Upload backup to router, adjust port as necessary
scp -P 22222 backup.tar.gz root@192.168.1.1:/tmp
 
# Restore backup
ls /tmp/backup.tar.gz
sysupgrade -r /tmp/backup.tar.gz
```
