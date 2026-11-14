# light-wifi-ap

## Description
this project is a docker container that allows you to create a WiFi HotSpot with hostapd and dnsmasq(optional).

## Installation
you can choose between 2 versions of the container, one with dhcp and one without.

start by cloning the repository and moving to the folder:
```sh
cd /opt
sudo git clone https://github.com/vampi62/light-wifi-ap.git
cd light-wifi-ap
```
then you can choose between the 2 versions of the container.

### Fixed name for the hotspot interface (all versions)

the kernel names the wifi interfaces `wlan0`, `wlan1`... in the order they are detected, so with a USB dongle the numbers can change from one boot to another.
to avoid this, the hotspot interface is renamed to a fixed name: `wlan_light_wifi`. This name is used in all the config files of this repository (`hostapd.conf`, `dnsmasq.conf`, ...).

get the MAC address of the wifi dongle/interface used by the hotspot (replace `wlan0` by the current name of the interface)
```sh
cat /sys/class/net/wlan0/address
```

create the `.link` file and set the MAC address of the dongle in it (`<wlan-mac-address>` is the address found above, NOT the one of `eth0`)
```sh
sudo cp /opt/light-wifi-ap/withDHCP/systemd/network/10-light-wifi-interface.link /etc/systemd/network/
# or withoutDHCP/systemd/network/ if you use the version without DHCP, the file is the same
sudo sed -i 's/AA:BB:CC:DD:EE:FF/<wlan-mac-address>/' /etc/systemd/network/10-light-wifi-interface.link
```

the file `/etc/systemd/network/10-light-wifi-interface.link` must look like this (with the MAC of your dongle)

```ini
[Match]
MACAddress=AA:BB:CC:DD:EE:FF

[Link]
Name=wlan_light_wifi
```
this file is handled by udev, so it works on Debian 12 and below and on Debian 13 and above. The interface is renamed at the next reboot (the reboot at the end of each section), after that `ip link show wlan_light_wifi` must show the interface.

in the rest of this document `wlan_light_wifi` is the interface used by the hotspot.

### with DHCP

in this section `wlan_light_wifi` is the interface used by the hotspot (`192.168.5.1/24`) and `eth0` is the interface that provides internet access.
if your uplink interface has another name, adapt the commands and the files in the config folder.
the subnet `192.168.5.0/24` must not be used by the upstream network.

#### Debian 12 and below

set a static ip on the wlan interface used by the hotspot and disable `wpa_supplicant` on it (dhcpcd)
```sh
sudo nano /etc/dhcpcd.conf
# add the following lines in the file
interface wlan_light_wifi # interface used by the hotspot
  static ip_address=192.168.5.1/24
  nohook wpa_supplicant
```

#### Debian 13 and above

Debian 13 uses NetworkManager instead of dhcpcd, so `/etc/dhcpcd.conf` is ignored: without the following steps `wlan_light_wifi` has no ip address, dnsmasq cannot answer the DHCP requests and the clients disconnect after a few seconds.

tell NetworkManager to ignore the wlan interface used by the hotspot
```sh
sudo tee /etc/NetworkManager/conf.d/99-unmanaged-wlan_light_wifi.conf <<'EOF'
[keyfile]
unmanaged-devices=interface-name:wlan_light_wifi
EOF
```

set the static ip with systemd-networkd
```sh
sudo tee /etc/systemd/network/20-wlan_light_wifi.network <<'EOF'
[Match]
Name=wlan_light_wifi

[Network]
Address=192.168.5.1/24
ConfigureWithoutCarrier=yes
EOF
sudo systemctl enable systemd-networkd
sudo systemctl restart systemd-networkd
sudo systemctl restart NetworkManager
```

check that the interface has its ip and is not managed by NetworkManager
```sh
ip -4 addr show wlan_light_wifi # must show 192.168.5.1/24
nmcli dev status                # wlan_light_wifi must be "unmanaged"
```

#### Common steps for both Debian 12 and below and Debian 13 and above

enable the ip forwarding
```sh
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-ip-forward.conf
sudo sysctl --system
```

set cron iptable rules to be applied at reboot
```sh
sudo crontab -e
# add the following lines
@reboot sleep 60 && sudo iptables -A FORWARD -m iprange --src-range 192.168.5.0-192.168.5.255 -j ACCEPT
@reboot sleep 60 && sudo iptables -A FORWARD -m iprange --dst-range 192.168.5.0-192.168.5.255 -j ACCEPT
```

reboot the system before running the container
```sh
sudo reboot
```

don't forget to edit the hostapd.conf and dnsmasq.conf files in the config folder to match your network configuration.
build and run the container with DHCP support
```sh
cd light-wifi-ap/withDHCP
sudo chmod 755 -R config
sudo chown root:root -R config
docker build -t light-wifi-ap:latest .
sudo docker run -d \
	--name light-wifi-ap \
	--net host -it \
	--privileged \
	-v /opt/light-wifi-ap/withDHCP/config/hostapd.conf:/etc/hostapd/hostapd.conf:ro \
	-v /opt/light-wifi-ap/withDHCP/config/dnsmasq.conf:/etc/dnsmasq.conf:ro \
	--cap-add=NET_ADMIN \
	--restart=always \
	light-wifi-ap:latest
```

### without DHCP

#### Debian 12 and below

install the bridge-utils package and configure the network interfaces
```sh
sudo apt-get install bridge-utils
sudo brctl addbr br0
sudo brctl addif br0 eth0
```

make persistent the bridge configuration
```sh
sudo nano /etc/network/interfaces.d/light-wifi-ap
# add the following lines in the file
auto br0
iface br0 inet dhcp
    bridge_ports eth0
```

disable the dhcp client on the wlan interface used by the hotspot
```sh
sudo nano /etc/dhcpcd.conf
# add the following lines in the file
denyinterfaces wlan_light_wifi # interface used by the hotspot

interface wlan_light_wifi # interface used by the hotspot
  nohook wpa_supplicant
```

#### Debian 13 and above

get the MAC address of the eth0 interface
```sh
cat /sys/class/net/eth0/address
```

copy the systemd-networkd configuration files and set the MAC address of eth0 in br0.netdev
(if you already copied `10-light-wifi-interface.link` and set the MAC of the dongle in it, see "Fixed name for the hotspot interface", do not overwrite it: copy only the `br0.*` and `eth0.network` files)
```sh
sudo cp /opt/light-wifi-ap/withoutDHCP/systemd/network/br0.* /opt/light-wifi-ap/withoutDHCP/systemd/network/eth0.network /etc/systemd/network/
sudo sed -i 's/xx:xx:xx:xx:xx:xx/<eth0-mac-address>/' /etc/systemd/network/br0.netdev
sudo systemctl enable systemd-networkd
sudo systemctl restart systemd-networkd
```

disable the DHCP client on the wlan interface used by the hotspot

tell NetworkManager to ignore the wlan interface used by the hotspot
```sh
sudo tee /etc/NetworkManager/conf.d/99-unmanaged-wlan_light_wifi.conf <<'EOF'
[keyfile]
unmanaged-devices=interface-name:wlan_light_wifi
EOF
```

#### Common steps for both Debian 12 and below and Debian 13 and above

in iptables add the following lines to allow the traffic between the wifi network and the ethernet network
```sh
# step 1
# add rule to iptables and make a backup file
sudo iptables -I DOCKER-USER -i br0 -j ACCEPT
sudo iptables-save > /etc/iptables/rules
# step 2
# add the iptables-restore command to the rc.local file
sudo nano /etc/rc.local
iptables-restore < /etc/iptables/rules
```
if you cannot use this method you can use the crontab for the same result
```sh
sudo crontab -e
# add the following lines in the file
@reboot sleep 60 && sudo iptables -I DOCKER-USER -i br0 -j ACCEPT
```

reboot the system before running the container
```sh
sudo reboot
```
don't forget to edit the hostapd.conf file in the config folder to match your network configuration.
build and run the container without DHCP support
```sh
cd /opt/light-wifi-ap/withoutDHCP
sudo chmod 755 -R config
sudo chown root:root -R config
docker build -t light-wifi-ap:latest .
sudo docker run -d \
	--name light-wifi-ap \
	--net host -it \
	--privileged \
	-v /opt/light-wifi-ap/withoutDHCP/config/hostapd.conf:/etc/hostapd/hostapd.conf:ro \
	--cap-add=NET_ADMIN \
	--restart=unless-stopped \
	light-wifi-ap:latest
```
