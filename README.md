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

in iptables add the following lines to allow the traffic between the wifi network and the ethernet network
```sh
# step 1
# add rule to iptables and make a backup file
sudo iptables -A FORWARD -m iprange --src-range 192.168.5.0-192.168.5.255 -j ACCEPT
sudo iptables -A FORWARD -m iprange --dst-range 192.168.5.0-192.168.5.255 -j ACCEPT
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
@reboot sleep 60 && sudo iptables -A FORWARD -m iprange --src-range 192.168.5.0-192.168.5.255 -j ACCEPT
@reboot sleep 60 && sudo iptables -A FORWARD -m iprange --dst-range 192.168.5.0-192.168.5.255 -j ACCEPT
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

don't forget to edit the hostapd.conf and dnsmasq.conf files in the config folder to match your network configuration.
build and run the container without DHCP support
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
