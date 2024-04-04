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
```



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

We need to introduce the added counters to freeradius, also currently freeradius ignores almost all the attributes that are applied to the profiles.
That's why we add some custom code to check them, To do this, open the following file:
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

#Additional code is needed to check profile attributes
#Check Max-Sessions-Limit
update control {
	&Tmp-Integer-1 := "%{sql:SELECT rc.value FROM radgroupcheck rc INNER JOIN radusergroup rug ON rc.groupname = rug.groupname WHERE rug.username = '%{User-Name}' and rc.attribute = 'Max-Sessions-Limit'}"
}

if(&control:Tmp-Integer-1 > 0) {
	update control {
		&Tmp-Integer-2 := "%{sql:SELECT COUNT(*) FROM radacct WHERE username = '%{User-Name}' AND acctstoptime IS NULL}"
	}
	
	if(&control:Tmp-Integer-1 <= &control:Tmp-Integer-2) {
		update reply {
			Reply-Message = "Your simultaneous connection is complete"
		}
		reject
	}

} 

#Check Expire-After-Login
update control {
	&Tmp-Integer-3 := "%{sql:SELECT rc.value FROM radgroupcheck rc INNER JOIN radusergroup rug ON rc.groupname = rug.groupname WHERE rug.username = '%{User-Name}' and rc.attribute = 'Expire-After-Login'}"
}

if(&control:Tmp-Integer-3 > 0) {
	update control {
		&Tmp-Integer-4 := "%{sql:SELECT IFNULL( MAX(TIME_TO_SEC(TIMEDIFF(NOW(), acctstarttime))),0) FROM radacct WHERE username = '%{User-Name}' ORDER BY acctstarttime LIMIT 1}"
	}
	
	if(&control:Tmp-Integer-3 <= &control:Tmp-Integer-4) {
		update reply {
			Reply-Message = "Your account has expired"
		}
		reject
	}
}

#Check Daily-Traffic-Limit
update control {
	&Tmp-Integer-5 := "%{sql:SELECT rc.value FROM radgroupcheck rc INNER JOIN radusergroup rug ON rc.groupname = rug.groupname WHERE rug.username = '%{User-Name}' and rc.attribute = 'Daily-Traffic-Limit'}"
}

if(&control:Tmp-Integer-5 > 0) {
	update control {
		&Tmp-Integer-6 := "%{sql:SELECT SUM(acctinputoctets) + SUM(acctoutputoctets) FROM radacct WHERE username = '%{User-Name}'}"
	}
	
	if(&control:Tmp-Integer-5 <= &control:Tmp-Integer-6) {
		update reply {
			Reply-Message = "Your daily traffic capacity has been exhausted"
		}
		reject
	}
}

#Check Weekly-Traffic-Limit
update control {
	&Tmp-Integer-5 := "%{sql:SELECT rc.value FROM radgroupcheck rc INNER JOIN radusergroup rug ON rc.groupname = rug.groupname WHERE rug.username = '%{User-Name}' and rc.attribute = 'Weekly-Traffic-Limit'}"
}

if(&control:Tmp-Integer-5 > 0) {
	update control {
		&Tmp-Integer-6 := "%{sql:SELECT SUM(acctinputoctets) + SUM(acctoutputoctets) FROM radacct WHERE username = '%{User-Name}'}"
	}
	
	if(&control:Tmp-Integer-5 <= &control:Tmp-Integer-6) {
		update reply {
			Reply-Message = "Your weekly traffic capacity has been exhausted"
		}
		reject
	}
}

#Check Monthly-Traffic-Limit
update control {
	&Tmp-Integer-5 := "%{sql:SELECT rc.value FROM radgroupcheck rc INNER JOIN radusergroup rug ON rc.groupname = rug.groupname WHERE rug.username = '%{User-Name}' and rc.attribute = 'Monthly-Traffic-Limit'}"
}

if(&control:Tmp-Integer-5 > 0) {
	update control {
		&Tmp-Integer-6 := "%{sql:SELECT SUM(acctinputoctets) + SUM(acctoutputoctets) FROM radacct WHERE username = '%{User-Name}'}"
	}
	
	if(&control:Tmp-Integer-5 <= &control:Tmp-Integer-6) {
		update reply {
			Reply-Message = "Your monthly traffic capacity has been exhausted"
		}
		reject
	}
}

#Accounting data updates every 15 minutes
update control {
	&Tmp-Integer-7 := "%{sql:SELECT value FROM radreply WHERE username = '%{User-Name}' and attribute = 'Acct-Interim-Interval'}"
}

if(&control:Tmp-Integer-7 == 0) {
	%{sql:INSERT IGNORE INTO radreply (username, attribute, op, value) VALUES ('%{User-Name}', 'Acct-Interim-Interval', ':=', 1200)}
}
```

Now we need to restart the freeradius service
```bash
systemctl restart freeradius
```




**#4 Configure dallo radius and create user**

Below I show how to create a account and set the counters we created for it,

-In the picture below, we have created an account with a test username with a maximum connection limit of **2 simultaneous sessions** and **100 gigs** of monthly traffic.
We have also specified that his account expires **30 days** after the first connection
And specified that every **600 seconds** of his accounting data is updated and saved.-

Also, another method is that you can create one or more profiles and add attributes to them and then set profiles for your users.

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

Now we write a cron job to restart the freeradius and openvpn services once a day at **04:00 am**.

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

Now the work is done.



