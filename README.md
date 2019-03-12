# NetLab VPN Server
Firstly we should install openvpn and easy-rsa for internal certificate authority
```sudo apt-get install openvpn easy-rsa```

Then set up te CA directory, we can copy the easy-rsa template into home directory using make-cadir command
```$ make-cadir ~/openvpn-ca
cd ~/openvpn-ca```

Next we should configure variables in certificate authority using command
```$nano vars```
Then, adjust these variables to determine how the certificate will be created
```export KEY_COUNTRY="ID"
export KEY_PROVINCE="JT"
export KEY_CITY="Semarang"
export KEY_ORG="NetLab"
export KEY_EMAIL="admin@netlab.undip.ac.id"
export KEY_OU="NetLab"```

