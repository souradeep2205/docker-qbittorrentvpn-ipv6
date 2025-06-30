# [qBittorrent](https://github.com/qbittorrent/qBittorrent), WireGuard and OpenVPN
[![Docker Pulls](https://img.shields.io/docker/pulls/dyonr/qbittorrentvpn)](https://hub.docker.com/r/dyonr/qbittorrentvpn)
[![Docker Image Size (tag)](https://img.shields.io/docker/image-size/dyonr/qbittorrentvpn/latest)](https://hub.docker.com/r/dyonr/qbittorrentvpn)

Docker container which runs the latest [qBittorrent](https://github.com/qbittorrent/qBittorrent)-nox client while connecting to WireGuard or OpenVPN with iptables killswitch to prevent IP leakage when the tunnel goes down. Now Supports IPv6!

## Enabling IPv6 support
The support is mainly serves the following:

 1. **Wireguard Endpoint over IPV6**: This means we will be tunneling both IPv4 & IPv6 traffic through an IPv6 network. 
 2. **Supporting both IPv4 & IPv6 peers**: Yes, this is possible, but only if the VPN server supports it. Port forwarding to be enabled for both on server side.
 
 For the VPN server setup, we would be using [wg-easy](https://github.com/wg-easy/wg-easy).
 In wg-easy we should create a seperate docker network before running our container.
 ```bash
 docker  network  create  \  
 -d bridge --ipv6  \ 
 --subnet 10.42.42.0/24  \  
 --subnet  fdcc:ad94:bacf:61a3::/64  wg
 ```

Now, this is the docker run configuration I used. 
In the WG_HOST variable put your server's IPv6 address!
Please note the WG_PRE_UP & WG_POST_DOWN variables, as they are the magic behind the port forwarding for qBittorrent. The IP addresses used are the wireguard internal network's address assigned to the different peers.
After you create the peers on the wg-easy UI, note down their IPv4 address (also IPv6 address if available) and replace in the place where 10.8.0.2 is mentioned below under iptables statement. If you have ipv6 set up please put the address under ip6tables statement.

If the primary network of your device is not 'eth0' then put the approprate name for the iptables & ip6tables lines.

We are using ports 8999 as the qBittorrent's connectable TCP/uTP port. If you wish to change, you can, then replace in the appropriate places in this command. Also make sure to change the port in qBittorrent UI.

Since before you run the command, you may not have a peer IP address beforehand, so its better you run the command first without the PRE_UP and POST_DOWN variables. Create the peers. Note down the peer's addresses. Stop wg-easy container. Re-edit the docker run statement with the variables and run it. You are good to go!

```bash
docker  run  -d  \
--network  wg  \
-e  INSECURE=true  \
-e  WG_HOST="[SERVER IPV6 ADDRESS]"  \
-e  WG_PRE_UP="iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8999 -j DNAT --to-destination 10.8.0.2; iptables -t nat -A PREROUTING -i eth0 -p udp --dport 8999 -j DNAT --to-destination 10.8.0.2; ip6tables -t nat -A PREROUTING -i eth0 -p tcp --dport 8999 -j DNAT --to-destination fdcc:ad94:bacf:61a4::cafe:2; ip6tables -t nat -A PREROUTING -i eth0 -p udp --dport 8999 -j DNAT --to-destination fdcc:ad94:bacf:61a4::cafe:2"  \
-e  WG_POST_DOWN="iptables -t nat -D PREROUTING -i eth0 -p tcp --dport 8999 -j DNAT --to-destination 10.8.0.2; iptables -t nat -D PREROUTING -i eth0 -p udp --dport 8999 -j DNAT --to-destination 10.8.0.2; ip6tables -t nat -D PREROUTING -i eth0 -p tcp --dport 8999 -j DNAT --to-destination fdcc:ad94:bacf:61a4::cafe:2; ip6tables -t nat -D PREROUTING -i eth0 -p udp --dport 8999 -j DNAT --to-destination fdcc:ad94:bacf:61a4::cafe:2"  \
--name  wg-easy  \
--ip6  fdcc:ad94:bacf:61a3::2a  \
--ip  10.42.42.42  \
-v  ~/.wg-easy:/etc/wireguard  \
-v  /lib/modules:/lib/modules:ro  \
-p  51820:51820/udp  \
-p  51821:51821/tcp  \
-p  8999:8999/tcp  \
-p  8999:8999/udp  \
--cap-add  NET_ADMIN  \
--cap-add  SYS_MODULE  \
--sysctl  net.ipv4.ip_forward=1  \
--sysctl  net.ipv4.conf.all.src_valid_mark=1  \
--sysctl  net.ipv6.conf.all.disable_ipv6=0  \
--sysctl  net.ipv6.conf.all.forwarding=1  \
--sysctl  net.ipv6.conf.default.forwarding=1  \
--restart  unless-stopped  \
ghcr.io/wg-easy/wg-easy:latest
```
Once this is done, get the wireguard config file, store it in the client.
On the client, make sure to [enable IPV6 for docker's default bridge network](https://docs.docker.com/engine/daemon/ipv6/#use-ipv6-for-the-default-bridge-network). Also, make sure you change the IPv6 subnet CIDR and not use the example one. Please usee this [tool](https://www.site24x7.com/tools/ipv6-subnetcalculator.html) to generate a locally routable /64 subnet.
Use the following docker run config for running this container.
```bash
sudo docker run --name qbv6 -d \
              -v /home/ubuntu/wireguardconfig:/config \
              -v /mnt/ubuntu/qbit_downloads:/downloads \
              -e "VPN_ENABLED=yes" \
              -e "VPN_TYPE=wireguard" \
              -e "LAN_NETWORK=192.168.0.0/16" \
              -e "RESTART_CONTAINER=no" \
              -p 8080:8080 \
              --restart unless-stopped \
              --cap-add NET_ADMIN \
              --sysctl net.ipv6.conf.all.disable_ipv6=0 \
              --sysctl net.ipv4.conf.all.src_valid_mark=1 \
              5727sde/docker-qbittorrentvpn-ipv6:latest
```
Please follow the rest of the guide for the other configurations.

-----------------------------------------------------------


[preview]: https://raw.githubusercontent.com/DyonR/docker-templates/master/Screenshots/qbittorrentvpn/qbittorrentvpn-webui.png "qBittorrent WebUI"
![alt text][preview]

# Docker Features
* Base: Debian bullseye-slim
* [qBittorrent](https://github.com/qbittorrent/qBittorrent) compiled from source
* [libtorrent](https://github.com/arvidn/libtorrent) compiled from source
* Compiled with the latest version of [Boost](https://www.boost.org/)
* Compiled with the latest versions of [CMake](https://cmake.org/)
* Selectively enable or disable WireGuard or OpenVPN support
* IP tables killswitch to prevent IP leaking when VPN connection fails
* Configurable UID and GID for config files and /downloads for qBittorrent
* Created with [Unraid](https://unraid.net/) in mind
* BitTorrent port 8999 exposed by default

## Run container from Docker registry
The container is available from the Docker registry and this is the simplest way to get it  
To run the container use this command, with additional parameters, please refer to the Variables, Volumes, and Ports section:

```
$ docker run  -d \
              -v /your/config/path/:/config \
              -v /your/downloads/path/:/downloads \
              -e "VPN_ENABLED=yes" \
              -e "VPN_TYPE=wireguard" \
              -e "LAN_NETWORK=192.168.0.0/24" \
              -p 8080:8080 \
              --cap-add NET_ADMIN \
              --sysctl "net.ipv4.conf.all.src_valid_mark=1" \
              --restart unless-stopped \
              dyonr/qbittorrentvpn
```

## Docker Tags
| Tag | Description |
|----------|----------|
| `dyonr/qbittorrentvpn:latest` | The latest version of qBittorrent with libtorrent 1_x_x |
| `dyonr/qbittorrentvpn:rc_2_0` | The latest version of qBittorrent with libtorrent 2_x_x |
| `dyonr/qbittorrentvpn:legacy_iptables` | The latest version of qBittorrent, libtorrent 1_x_x and an experimental feature to fix problems with QNAP NAS systems, [Issue #25](https://github.com/DyonR/docker-qbittorrentvpn/issues/25) |
| `dyonr/qbittorrentvpn:alpha` | The latest alpha version of qBittorrent with libtorrent 2_0, incase you feel like testing new features |
| `dyonr/qbittorrentvpn:dev` | This branch is used for testing new Docker features or improvements before merging it to the main branch |
| `dyonr/qbittorrentvpn:v4_2_x` | (Legacy) qBittorrent version 4.2.x with libtorrent 1_x_x |

# Variables, Volumes, and Ports
## Environment Variables
| Variable | Required | Function | Example | Default |
|----------|----------|----------|----------|----------|
|`VPN_ENABLED`| Yes | Enable VPN (yes/no)?|`VPN_ENABLED=yes`|`yes`|
|`VPN_TYPE`| Yes | WireGuard or OpenVPN (wireguard/openvpn)?|`VPN_TYPE=wireguard`|`openvpn`|
|`VPN_USERNAME`| No | If username and password provided, configures ovpn file automatically |`VPN_USERNAME=ad8f64c02a2de`||
|`VPN_PASSWORD`| No | If username and password provided, configures ovpn file automatically |`VPN_PASSWORD=ac98df79ed7fb`||
|`LAN_NETWORK`| Yes (atleast one) | Comma delimited local Network's with CIDR notation |`LAN_NETWORK=192.168.0.0/24,10.10.0.0/24`||
|`LEGACY_IPTABLES`| No | Use `iptables (legacy)` instead of `iptables (nf_tables)` |`LEGACY_IPTABLES=yes`||
|`ENABLE_SSL`| No | Let the container handle SSL (yes/no)? |`ENABLE_SSL=yes`|`yes`|
|`NAME_SERVERS`| No | Comma delimited name servers |`NAME_SERVERS=1.1.1.1,1.0.0.1`|`1.1.1.1,1.0.0.1`|
|`PUID`| No | UID applied to /config files and /downloads |`PUID=99`|`99`|
|`PGID`| No | GID applied to /config files and /downloads  |`PGID=100`|`100`|
|`UMASK`| No | |`UMASK=002`|`002`|
|`HEALTH_CHECK_HOST`| No |This is the host or IP that the healthcheck script will use to check an active connection|`HEALTH_CHECK_HOST=one.one.one.one`|`one.one.one.one`|
|`HEALTH_CHECK_INTERVAL`| No |This is the time in seconds that the container waits to see if the internet connection still works (check if VPN died)|`HEALTH_CHECK_INTERVAL=300`|`300`|
|`HEALTH_CHECK_SILENT`| No |Set to `1` to supress the 'Network is up' message. Defaults to `1` if unset.|`HEALTH_CHECK_SILENT=1`|`1`|
|`HEALTH_CHECK_AMOUNT`| No |The amount of pings that get send when checking for connection.|`HEALTH_CHECK_AMOUNT=10`|`1`|
|`RESTART_CONTAINER`| No |Set to `no` to **disable** the automatic restart when the network is possibly down.|`RESTART_CONTAINER=yes`|`yes`|
|`INSTALL_PYTHON3`| No |Set this to `yes` to let the container install Python3.|`INSTALL_PYTHON3=yes`|`no`|
|`ADDITIONAL_PORTS`| No |Adding a comma delimited list of ports will allow these ports via the iptables script.|`ADDITIONAL_PORTS=1234,8112`||

## Volumes
| Volume | Required | Function | Example |
|----------|----------|----------|----------|
| `config` | Yes | qBittorrent, WireGuard and OpenVPN config files | `/your/config/path/:/config`|
| `downloads` | No | Default downloads path for saving downloads | `/your/downloads/path/:/downloads`|

## Ports
| Port | Proto | Required | Function | Example |
|----------|----------|----------|----------|----------|
| `8080` | TCP | Yes | qBittorrent WebUI | `8080:8080`|
| `8999` | TCP | Yes | qBittorrent TCP Listening Port | `8999:8999`|
| `8999` | UDP | Yes | qBittorrent UDP Listening Port | `8999:8999/udp`|

# Access the WebUI
Access https://IPADDRESS:PORT from a browser on the same network. (for example: https://192.168.0.90:8080)

## Default Credentials

| Credential | Default Value |
|----------|----------|
|`username`| `admin` |
|`password`| `adminadmin` |

# How to use WireGuard 
The container will fail to boot if `VPN_ENABLED` is set and there is no valid .conf file present in the /config/wireguard directory. Drop a .conf file from your VPN provider into /config/wireguard and start the container again. The file must have the name `wg0.conf`, or it will fail to start.

## WireGuard IPv6 issues
If you use WireGuard and also have IPv6 enabled, it is necessary to add the IPv6 range to the `LAN_NETWORK` environment variable.  
Additionally the parameter `--sysctl net.ipv6.conf.all.disable_ipv6=0` also must be added to the `docker run` command, or to the "Extra Parameters" in Unraid.  
The full Unraid `Extra Parameters` would be: `--restart unless-stopped --sysctl net.ipv6.conf.all.disable_ipv6=0"`  
If you do not do this, the container will keep on stopping with the error `RTNETLINK answers permission denied`.
Since I do not have IPv6, I am did not test.
Thanks to [mchangrh](https://github.com/mchangrh) / [Issue #49](https://github.com/DyonR/docker-qbittorrentvpn/issues/49)  

# How to use OpenVPN
The container will fail to boot if `VPN_ENABLED` is set and there is no valid .ovpn file present in the /config/openvpn directory. Drop a .ovpn file from your VPN provider into /config/openvpn (if necessary with additional files like certificates) and start the container again. You may need to edit the ovpn configuration file to load your VPN credentials from a file by setting `auth-user-pass`.

**Note:** The script will use the first ovpn file it finds in the /config/openvpn directory. Adding multiple ovpn files will not start multiple VPN connections.

## Example auth-user-pass option for .ovpn files
`auth-user-pass credentials.conf`

## Example credentials.conf
```
username
password
```

## PUID/PGID
User ID (PUID) and Group ID (PGID) can be found by issuing the following command for the user you want to run the container as:

```
id <username>
```

# Issues
If you are having issues with this container please submit an issue on GitHub.  
Please provide logs, Docker version and other information that can simplify reproducing the issue.  
If possible, always use the most up to date version of Docker, you operating system, kernel and the container itself. Support is always a best-effort basis.

### Credits:
[MarkusMcNugen/docker-qBittorrentvpn](https://github.com/MarkusMcNugen/docker-qBittorrentvpn)  
[DyonR/jackettvpn](https://github.com/DyonR/jackettvpn)  
This projects originates from MarkusMcNugen/docker-qBittorrentvpn, but forking was not possible since DyonR/jackettvpn uses the fork already.
