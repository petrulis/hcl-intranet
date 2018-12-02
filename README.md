# Intranet infrastructure

## Architecture

![Diagram](/docs/architecture.png)

## Setup

By following this setup guide you will create intranet website which will be
accesible only over ipsec.1 VPN connection over hclintranet.com domain.

### Import master.yml into CloudFormation

Before importing cloudoformation template ask yourself whether you already own intranet domain or
you want to create private hosted zone instead. If you own the domain, keep in mind that CloudFormation will
try to provision HostedZone for it and that you will have to provision ACM certificate for it and set
its ARN as CertificateARN


If you don't have domain and prefer private HostedZone and you don't own Private Certificate Authority then
self signed sertificate must be created and imported into AWS before referencing it with CertificateARN
paramaeter.

	openssl req -x509 -nodes -days 730 -newkey rsa:2048 -keyout cert.key -out cert.pem -config req.cnf -sha256

	aws acm import-certificate --certificate file://cert.pem --private-key file://cert.key --region ${REGION}

Keep in mind that REGION is the region where master.yml is going to be provisioned.

### Configure IPSEC VPN customer gateway

This guide is only relevant for Linux environments with strongswan.

### Enable Packet Forwarding and Configure the Tunnel

1. Install strongswan
	```console
	sudo apt-get install strongswan
	```
2. Open /etc/sysctl.conf and uncomment the following line to enable IP packet forwarding:
	```console
   net.ipv4.ip_forward = 1
   ```

3. Create ipsec conf file at /etc/ipsec.conf

	```console
	conn Tunnel1
		auto=start
		left=%defaultroute
		leftid=<PUBLIC_IP>
		right=<TUNNEL1_OUTSIDE_IP>
		type=tunnel
		leftauth=psk
		rightauth=psk
		keyexchange=ikev1
		ike=aes128-sha1-modp1024
		ikelifetime=8h
		esp=aes128-sha1-modp1024
		lifetime=1h
		keyingtries=%forever
		leftsubnet=0.0.0.0/0
		rightsubnet=0.0.0.0/0
		dpddelay=10s
		dpdtimeout=30s
		dpdaction=restart
		mark=100
		leftupdown="/etc/ipsec.d/aws-updown.sh -ln Tunnel1 -ll 169.254.6.2/30 -lr 169.254.6.1/30 -m 100 -r <VPC CIDR>"

	conn Tunnel2
		auto=start
		left=%defaultroute
		leftid=<PUBLIC_IP>
		right=<TUNNEL2_OUTSIDE_IP>
		type=tunnel
		leftauth=psk
		rightauth=psk
		keyexchange=ikev1
		ike=aes128-sha1-modp1024
		ikelifetime=8h
		esp=aes128-sha1-modp1024
		lifetime=1h
		keyingtries=%forever
		leftsubnet=0.0.0.0/0
		rightsubnet=0.0.0.0/0
		dpddelay=10s
		dpdtimeout=30s
		dpdaction=restart
		mark=200
		leftupdown="/etc/ipsec.d/aws-updown.sh -ln Tunnel1 -ll 169.254.7.2/30 -lr 169.254.7.1/30 -m 200 -r <VPC CIDR>"
		```
4) Create a new file at /etc/ipsec.secrets if it doesn't already exist, and append this line to the file. This value authenticates the tunnel endpoints:
	```console
	<PUBLIC_IP> <TUNNEL1_OUTSIDE_IP> : PSK "<SECRET1>"
	<PUBLIC_IP> <TUNNEL1_OUTSIDE_IP> : PSK "<SECRET2>"
	```

5. Create a file at /etc/ipsec.d/aws-updown.sh and append file contents from aws-updown.sh.

6. Restart ipsec
	```console
	sudo sysctl -p && ipsec restart
	```
7. As you establish private connectivity between your on-premises networks and your AWS Virtual Private Cloud (VPC) environments, the need for Domain Name System (DNS) resolution across these environments grows in importance.

7. To establish private connectivity between your on-premises network and VPC the need for DNS resolution accross these
environments were solved by creating SimpleAD directory service. When connected to VPC SimpleAD provides two
DNS addresses which have to be added to you DNS forwarder (if you control your on-promise DNS service) or /etc/resolve.conf at your customer gateway part.
	```console
	nameserver <IP_1>
	nameserver <IP_2>
	```
