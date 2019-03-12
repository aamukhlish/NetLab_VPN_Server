# NetLab VPN Server
Firstly we should install openvpn and easy-rsa for internal certificate authority
```
$ sudo apt-get install openvpn easy-rsa
```

Then set up te CA directory, we can copy the easy-rsa template into home directory using make-cadir command
```
$ make-cadir ~/openvpn-ca
$ cd ~/openvpn-ca
```

Next we should configure variables in certificate authority using command
```
$ nano vars
```
Then, adjust these variables to determine how the certificate will be created
```
export KEY_COUNTRY="ID"
export KEY_PROVINCE="JT"
export KEY_CITY="Semarang"
export KEY_ORG="NetLab"
export KEY_EMAIL="admin@netlab.undip.ac.id"
export KEY_OU="NetLab"
```

In the same file, edit the KEY_NAME value to netlab
```
export KEY_NAME="server"
```

Next step is using variables and set easy-rsa to create certificate authority
```
$ cd ~/openvpn-ca
$ source vars
$ ./clean-all
$ ./build-ca
```

Then, generate server certificate and pair using command:
```
$ ./build-key-server server
```
After that, generate strong Diffie-Hellman keys by typing:
```
$ ./build-dh
```
Next step, strengthen server's TLS integrity by generating HMAC signature
```
$ openvpn --genkey --secret keys/ta.key
```

Afterwards, generate client certificate and key pair to authenticate access to vpn server. In this case, we will create client named netlab
```
$ cd ~/openvpn-ca
$ source vars
$ ./build-key-pass client1
```

In this step, we will configure OpenVPN Service. firstly we should copy files to the OpenVPN Directory using this command
```
$ cd ~/openvpn-ca/keys
$ sudo cp ca.crt server.crt server.key ta.key dh2048.pem /etc/openvpn
```

Next, we need to unzip a sample OpenVPN configuration file into configuration directory to use it as a basis for our setup:
```
$ gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz | sudo tee /etc/openvpn/server.conf
```

Now, we will modify the server configuration files in the server.conf files
```
sudo nano /etc/openvpn/netlab.conf
```

Change some configuration in the files. Uncomment the tls-auth and add key-direction to 0. Next uncomment the AES-128-CBC line to add encryption and add auth SHA256 line to enable authentication. Also uncomment the user and group lines, so the configuration will be like this
```
tls-auth ta.key 0 # This file is secret
key-direction 0

cipher AES-128-CBC

auth SHA256

user nobody
group nogroup
```

Next step is configure routing and NAT in the VPN configuration. To do this we can edit the rc.local files in /etc directory
```
$ nano /etc/rc.local
```

We need to enable forwarding and adjust the ip tables to allow forwarding and masquerading. in this scenario, we will translate the 10.8.0.0/24 (the tunnel network) to 10.42.12.0/24 (the local network)
add this configuration in the rc.local file
```
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -j SNAT --to 182.255.0.1$
echo 1 > /proc/sys/net/ipv4/ip_forward

iptables -X
iptables -F
iptables -t nat -F
iptables -t nat -X

iptables -A INPUT -m state --state ESTABLISH,RELATED -j ACCEPT
iptables -A FORWARD -m state --state ESTABLISH,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISH,RELATED -j ACCEPT
iptables -t nat -I POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
iptables -t nat -I POSTROUTING -s 10.9.0.0/24 -o eth0 -j MASQUERADE
iptables -t nat -I POSTROUTING -s 10.8.0.0/24 -o eth1 -j MASQUERADE

if ! [ -c /dev/net/tun ]; then
 mkdir -p /dev/net
 mknod -m 666 /dev/net/tun c 10 200
fi

systemctl restart openvpn@netlab.service
```

Afterwards, we will create the client configuration infrastructure, we need to create directory in home to store and adjust the privileges files using this command:
```
$ mkdir -p ~/client-configs/files
$ chmod 700 ~/client-configs/files
```

Next we will create the base configuration by copying the client.conf file from /usr/share/doc/openvpn/examples/sample-config-files/ directory using this command:
```
$ cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf
$ nano ~/client-configs/base.conf
```
Then, make some adjustments for remote address, protocol, user, group, encryption, and authentication
```
remote server_ip 1194
proto udp
user nobody
group nogroup
cipher AES-128-CBC
auth SHA256
key-direction 1
```

Finally, we will create the configuration generation script
```
nano ~/client-configs/make_config.sh
```
Inside the file, copy the following script
```
### #!/bin/bash#

### # First argument: Client identifier#

KEY_DIR=~/openvpn-ca/keys
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn
```
Next, mark the file as executable file using this command:
```
$ chmod +x ~/client-config/make_config.sh
```
The final step is run the file using this command , and copy the .ovpn file to the client
```
cd ~/client-configs
./make_config.sh netlab
```
