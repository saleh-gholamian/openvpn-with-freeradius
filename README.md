# openvpn-with-freeradius
Openvpn + Freeradius configuration on Ubuntu server


**#1 Build openvpn radiusplugin**
```bash
apt install libgcrypt20-dev
```
```bash
wget https://github.com/saleh-gholamian/openvpn-freeradius-plugin/archive/refs/tags/v2.2.tar.gz
tar xvf v2.2.tar.gz
cd openvpn-freeradius-plugin-2.2
make



**#2 Configure openvpn**

First, we create a folder for the plugin
```bash
cd /etc/openvpn
```
```bash
mkdir /etc/openvpn/radius
```

We move the plugin we made in the previous step to the new folder
```bash
sudo mv /root/openvpn-freeradius-plugin-2.2/radiusplugin.so /etc/openvpn/radius/
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
Add the following lines in the file and save the file
```bash
plugin /etc/openvpn/radius/radiusplugin.so /etc/openvpn/radius/radiusplugin.cnf
username-as-common-name
verify-client-cert none
```

Add the following line in your openvpn client config file
```bash
auth-user-pass
```

Now you need to restart the openvpn server
```bash
systemctl restart openvpn@server
```




**#3 Configure Freeradius**

First, you need to add the sqlcounter module
```bash
cd /etc/freeradius/3.0/mods-enabled
```
```bash
sudo ln -s ../mods-available/sqlcounter sqlcounter
```
```bash
cd
```

We need to add some counters, open the sqlcounter file with an editor like nano
```bash
nano /etc/freeradius/3.0/mods-enabled/sqlcounter
```
Replace the following content in this file and save the file
```bash
sqlcounter max_sessions {
    counter_name = Max-Sessions-limit
    check_name = Max-Sessions-limit
    sql_module_instance = sql
    key = User-Name
    reset = never
	
    query = "\
	SELECT COUNT(*) \
	FROM radacct \
	WHERE username='%{${key}}' AND acctstoptime IS NULL"
}

sqlcounter expire_after {
    counter_name = Expire-After-Login
    check_name = Expire-After-Login
    sql_module_instance = sql
    key = User-Name
    reset = never
    
    query = "\
	SELECT IFNULL( MAX(TIME_TO_SEC(TIMEDIFF(NOW(), acctstarttime))),0) \
	FROM radacct \
	WHERE username='%{${key}}' \
	ORDER BY acctstarttime \
	LIMIT 1;"
}

sqlcounter daily_traffic {
    counter_name = Daily-Traffic-Limit
    check_name = Daily-Traffic-Limit
    sql_module_instance = sql
    key = User-Name
    reset = 1
    
    query = "SELECT SUM(acctinputoctets) + SUM(acctoutputoctets) FROM radacct WHERE username='%{${key}}'"
}

sqlcounter weekly_traffic {
    counter_name = Weekly-Traffic-Limit
    check_name = Weekly-Traffic-Limit
    sql_module_instance = sql
    key = User-Name
    reset = 7
    
    query = "SELECT SUM(acctinputoctets) + SUM(acctoutputoctets) FROM radacct WHERE username='%{${key}}'"
}

sqlcounter monthly_traffic {
    counter_name = Monthly-Traffic-Limit
    check_name = Monthly-Traffic-Limit
    sql_module_instance = sql
    key = User-Name
    reset = 30
    
    query = "SELECT SUM(acctinputoctets) + SUM(acctoutputoctets) FROM radacct WHERE username='%{${key}}'"
}
```

We need to introduce the added counters to freeradius, for this, open the following file
```bash
nano /etc/freeradius/3.0/sites-enabled/default
```
Find the **authorize** block and add the following lines inside it and save the file
```bash
max_sessions
expire_after
daily_traffic
weekly_traffic
monthly_traffic
```

Now we need to restart the freeradius service
```bash
systemctl restart freeradius
```




**#4 Configure dallo radius and create user**

Unfortunately, at the moment, the attributes only work if they are applied to the user himself,
and if you want to create different profiles with different attributes and then set these profiles for users,
don't worry, this will not work! I tried a lot to understand why it doesn't work or is this a problem at all or is it normal but there is no useful information available but I think this is a freeradius bug,
anyway we can still set attributes on users and This is also useful for me.
Below I show how to create a user and set the counters we created for him, I also add an attribute that updates the accounting data every 10 minutes.

We follow the following path in the Daloradius management panel
Manament -> Users -> New User
First, in the Account Info tab, we define a username and password for the new user
Then we enter the Attributes tab and select the "Quickly Locate attribute with autocomplete input" section and write the names of the attributes
and click on add, then enter the desired value for each one and finally click on Apply.
In the image below, we have created a user named test with a maximum connection limit of 2 simultaneous sessions and 100 gigs of monthly traffic.
 We have also specified that his account will expire 30 days after the first connection.
 We have also specified that every 600 seconds of data His accounting was updated and saved.

![alt text](https://github.com/saleh-gholamian/openvpn-with-freeradius/blob/main/create_user.png)




**#5 Solve some problems:**

There is one scenario, when freeradius is unable to receive the disconnect event from the NAS for any reason, 
the sessions that were open will remain open forever and this will cause the user to fail to login again, for example when the server to In the case of a sudden restart,
the sessions that were open at the time will remain open forever.
To solve this problem, when starting the freeradius service, we use a query to close all the open sessions in the database, 
also to make this query work better. We repeat using cron job once a day

First, we create a script:
```bash
nano /etc/freeradius/3.0/cleanup_sessions.sh
```
Enter the following codes and save the file:
**#6 **
```bash
#!/bin/bash

DB_USER="radius"
DB_PASS="PASSWORD"
DB_NAME="radius"

mysql -u"$DB_USER" -p"$DB_PASS" -D "$DB_NAME" <<EOF
UPDATE radacct
SET acctstoptime = NOW()
WHERE acctstoptime IS NULL;
EOF
```

Give execute permission to the script created
```bash
sudo chmod +x /etc/freeradius/3.0/cleanup_sessions.sh
```

Now you need to open the freeradius service:
```bash
nano /etc/systemd/system/multi-user.target.wants/freeradius.service
```
You need to add the following line after **[Service]**.

_ExecStartPost=/etc/freeradius/3.0/cleanup_sessions.sh_

The content of this file after adding the above line should be as follows:
```bash
[Unit]
Description=FreeRADIUS multi-protocol policy server
After=network.target
Documentation=man:radiusd(8) man:radiusd.conf(5) http://wiki.freeradius.org/ ht>

[Service]
Type=notify
PIDFile=/run/freeradius/freeradius.pid
EnvironmentFile=-/etc/default/freeradius
ExecStartPre=/etc/freeradius/3.0/cleanup_sessions.sh
ExecStartPre=/usr/sbin/freeradius $FREERADIUS_OPTIONS -Cxm -lstdout
ExecStart=/usr/sbin/freeradius -f $FREERADIUS_OPTIONS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Now we need to restart the freeradius service
```bash
systemctl daemon-reload
systemctl restart freeradius
```
First, we change the time zone of our server to our region, for me it is Iran:
```bash
timedatectl set-timezone Asia/Tehran
```

Now we write a cron job to restart the freeradius and openvpn services once a day at 04:00 am.

You may be asked for the type of editor after this command, type **1** and enter.
```bash
crontab -e
```
Add the following line at the end of the file and save the file
```bash
0 4 * * * systemctl restart freeradius && sleep 5 && systemctl restart openvpn@server
```

It is better to restart cron at the end
```bash
sudo systemctl restart cron
```

Now the work is done



