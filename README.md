# Setting up a Docker container as a DHCP server

In this guide I’ve tested a number of different commands and configurations using Docker to run a container with dhcpd (+macvlan driver) to serve my clients in my home network. In the end i’ll migrate from my Windows 2012 R2 Server running DHCP to a much more lightweight Docker container (7.42 **MB** in total). Wow.

## My home environment:

* Firewall (Juniper)
  * I’m running IP helper for bootp which in this case means that i relay DHCP requests from various VLANs into one where i've placed my Windows 2012 R2 server. This is also where my container will live. See the FW configuration below:
  
```
forwarding-options {
	helpers {
		bootp {
			description "Global DHCP relay service";
			server 10.0.99.6;
			maximum-hop-count 4;
			interface {
				ge-0/0/x.XX;
				ge-0/0/x.XX;
			}
		}
	}
}
```
 
* ESXi v6.5
  * Running a VM (Debian Jessie) where I’ve installed Docker. The VM have two network interfaces assigned to it, one in a “DMZ” zone and one which I will trunk all VLANs to via ESXi and a port group with VLAN id 4095 (trunk all VLANs).
  * On the port group that will be used for trunking VLANs i had to enable _Promiscuous mode_ via the Security-settings.

![dhcpd-as-container](https://user-images.githubusercontent.com/899665/33234993-35370a16-d230-11e7-8df7-36e774aa64fb.png)

## Configuring Docker and running container

1. Create a VLAN interface on the Docker host and give it an address in the subnet
```bash
iface ens192.99 inet static
  address 10.0.99.5
  netmask 255.255.255.0
```
2. Create a network using the macvlan driver.
```bash
docker network create -d macvlan --subnet=10.0.99.0/24 --gateway=10.0.99.1 -o parent=ens192.30 macvlan0
```
3. Here's the Docker container repo: [docker-dhcpd](https://github.com/mikejoh/docker-dhcpd/)
4. My dhcpd.conf that have worked for me, remember that i'm using a DHCP relay between the client and server
```bash
authoritative;

default-lease-time 86400;
max-lease-time 86400;

# This is a workaround to let this dhcpd server serve requests to other subnets
# then it's own.
# If this is not present then the dhcpd daemon will throw an error and exit.
subnet 10.0.99.0 netmask 255.255.255.0 {
}

# This is my WLAN subnet
subnet 10.0.100.0 netmask 255.255.255.0 {
	option routers 10.0.100.1;
	option subnet-mask 255.255.255.0;
	range 10.0.100.150 10.0.100.200;
	option broadcast-address 10.0.100.255;
	option domain-name-servers 8.8.8.8, 8.8.4.4;
}
```
5. Build the Docker image (from within project directory)
```bash
docker build . -t dhcpd
```
6. Run the container
```bash
docker run -d --restart unless-stopped --ip 10.0.99.6 --net=macvlan0 dhcpd
```
or simply
```bash
docker-compose up -d
```
7. Copy the dhcpd.lease file from the container to your local filesystem to check if you have any active leases
```bash
docker cp <Container ID>:/var/lib/dhcp/dhcpd.leases .
```
