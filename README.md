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

### with DHCP

in this section `wlan0` is the interface used by the hotspot (`192.168.5.1/24`) and `eth0` is the interface that provides internet access.
if your interfaces have other names (for example a USB wifi dongle named `wlan1`, or an uplink on `wlan0`), adapt the commands and the files in the config folder.
the subnet `192.168.5.0/24` must not be used by the upstream network.

#### Debian 12 and below

set a static ip on the wlan interface used by the hotspot and disable `wpa_supplicant` on it (dhcpcd)
```sh
sudo nano /etc/dhcpcd.conf
# add the following lines in the file
interface wlan0 # interface used by the hotspot
  static ip_address=192.168.5.1/24
  nohook wpa_supplicant
```

#### Debian 13 and above

Debian 13 uses NetworkManager instead of dhcpcd, so `/etc/dhcpcd.conf` is ignored: without the following steps `wlan0` has no ip address, dnsmasq cannot answer the DHCP requests and the clients disconnect after a few seconds.

tell NetworkManager to ignore the wlan interface used by the hotspot
```sh
sudo tee /etc/NetworkManager/conf.d/99-unmanaged-wlan0.conf <<'EOF'
[keyfile]
unmanaged-devices=interface-name:wlan0
EOF
```

set the static ip with systemd-networkd
```sh
sudo tee /etc/systemd/network/20-wlan0.network <<'EOF'
[Match]
Name=wlan0

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
ip -4 addr show wlan0 # must show 192.168.5.1/24
nmcli dev status      # wlan0 must be "unmanaged"
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

#### Debian 13 and above

get the MAC address of the eth0 interface
```sh
cat /sys/class/net/eth0/address
```

copy the systemd-networkd configuration files and set the MAC address of eth0 in br0.netdev
```sh
sudo cp /opt/light-wifi-ap/withoutDHCP/systemd/network/* /etc/systemd/network/
sudo sed -i 's/xx:xx:xx:xx:xx:xx/<eth0-mac-address>/' /etc/systemd/network/br0.netdev
sudo systemctl enable systemd-networkd
sudo systemctl restart systemd-networkd
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

disable the dhcp client on the wlan interface used by the hotspot
```sh
sudo nano /etc/dhcpcd.conf
# add the following lines in the file
denyinterfaces wlan0 # interface used by the hotspot

interface wlan0 # interface used by the hotspot
  nohook wpa_supplicant
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
