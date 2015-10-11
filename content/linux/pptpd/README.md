Install pptpd with ufw in Ubuntu 12.04
--------------------------------------


####Install pptpd

```
sudo apt-get install pptpd
```

#### Open the port 22, 1723 for pptpd vpn

```
sudo ufw allow 22
sudo ufw allow 1723
```

#### Edit /etc/ppp/pptpd-options

```
# Comment out these lines
#refuse-pap
#refuse-chap
#refuse-mschap

# Uncomment thest lines
ms-dns 8.8.8.8
ms-dns 8.8.4.4
```

#### Edit /etc/pptpd.conf

```
# Add these lines
localip 10.99.99.99
remoteip 10.99.99.100-199
```

#### Edit /etc/ppp/chap-secrets

```
#[Username] [Service] [Password] [Allowed IP Address]
sampleusername pptpd samplepassword *
```

#### Reboot pptpd

```
sudo /etc/init.d/pptpd restart
```

#### Edit /etc/sysctl.conf

```
#Uncomment this line
net.ipv4.ip_forward=1
```

#### Reload the configuration

```
sudo sysctl -p
```

#### Edit /etc/default/ufw

```
#DEFAULT_FORWARD_POLICY="DROP"
DEFAULT_FORWARD_POLICY="ACCEPT"
```

#### Edit /etc/ufw/before.rules

```
# Add these lines before filter rules
# NAT table rules
*nat

:POSTROUTING ACCEPT [0:0]
# Allow forward traffic to eth0
-A POSTROUTING -s 10.99.99.0/24 -o eth0 -j MASQUERADE

# Process the NAT table rules
COMMIT
```
