# openvpn-with-freeradius
Openvpn + Freeradius configuration on Ubuntu server


**#0 Build openvpn radiusplugin**
We can 
```bash
apt install libgcrypt20-dev
```
```bash
wget https://github.com/ValdikSS/openvpn-radiusplugin/archive/refs/tags/v2.2.tar.gz
```
```bash
tar xvf v2.2.tar.gz
```
```bash
cd openvpn-radiusplugin-2.2
```
```bash
make
```



**#1 Configure openvpn**
First, we create a folder for the plugin
```bash
cd /etc/openvpn
```
```bash
mkdir /etc/openvpn/radius
```

We move the plugin we made in the previous step to the new folder
```bash
sudo mv /root/openvpn-radiusplugin-2.2/radiusplugin.so /etc/openvpn/radius/
```

Run the following command to create a config file in the same folder for the plugin
```bash
nano /etc/openvpn/radius/radiusplugin.cnf
```
Enter the following settings in the file and save the file: (If necessary, edit the values according to your NAS settings)
```bash
NAS-Identifier=OpenVpn
Service-Type=5
Framed-Protocol=1
NAS-Port-Type=5
NAS-IP-Address=127.0.0.1
OpenVPNConfig=/etc/openvpn/server.conf
subnet=255.255.255.0
overwriteccfiles=true
nonfatalaccounting=false
server
{
    acctport=1813
    authport=1812
    name=127.0.0.1
    retry=1
    wait=1
    sharedsecret=testing123
}
```

Open the openvpn server configuration file
```bash
nano /etc/openvpn/server.conf
```
Enter the settings as below and save the file
```bash

```



**#3 **
```bash

```
```bash

```



**#4 **
```bash

```


**#5 **
```bash
cd /etc/freeradius/3.0/mods-enabled
```
```bash
sudo ln -s ../mods-available/sqlcounter sqlcounter
```
```bash
cd
```



**#6 **
```bash

```



**#7 **
```bash

```



**#8 **
```bash

```
