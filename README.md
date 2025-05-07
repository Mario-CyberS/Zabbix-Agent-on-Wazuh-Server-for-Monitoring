# Zabbix Agent on Wazuh Server for Monitoring  
This project documents how to install and configure the Zabbix Agent on the Wazuh SIEM server to enable detailed monitoring using the Zabbix frontend. It includes service health checks, `/var` disk usage alerts, Rancid backup verification, and SELinux policy adjustments for secure command execution.

---

## 🎯 Objective  
To deploy and configure the Zabbix Agent on the Wazuh SITE server to monitor:
- Wazuh services (Manager, Indexer, Dashboard)
- Disk usage alerts for `/var` at 80%, 90%, and 95%
- ASA config backup status via a Rancid + Bash script
- SELinux integration for safe command execution with `sudo`

---

## 📚 Skills Learned  
- Installing and configuring Zabbix Agent  
- Creating and assigning host templates and items  
- Writing bash scripts and setting exit codes  
- Using macros for dynamic disk thresholds  
- Creating and applying SELinux policies  
- Working with the Zabbix frontend for monitoring and trigger testing

---

## 🛠️ Tools Used  
<div>
  <img src="https://img.shields.io/badge/-Zabbix-DC382D?&style=for-the-badge&logo=Zabbix&logoColor=white" />
  <img src="https://img.shields.io/badge/-Wazuh-0078D4?&style=for-the-badge&logo=Wazuh&logoColor=white" />
  <img src="https://img.shields.io/badge/-SELinux-000000?&style=for-the-badge" />
</div>

---

## 📝 Deployment Steps

### 1. Install Zabbix Agent on Wazuh Server
install the Zabbix agent on the Wazuh SITE server, but make sure it is the same version as Zabbix server or earlier:
<a href="https://www.zabbix.com/download?zabbix=6.0&os_distribution=alma_linux&os_version=8&components=agent&db=&ws=" target="_blank"><img src="https://img.shields.io/badge/-Zabbix-DC143C?&style=for-the-badge&logo=Zabbix&logoColor=white" />
Run the following commands for installation and deployment:
```bash
sudo rpm -Uvh https://repo.zabbix.com/zabbix/6.0/rhel/8/x86_64/zabbix-release-latest-6.0.el8.noarch.rpm
sudo dnf clean all
sudo dnf install zabbix-agent
sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent
```
### 2. Now create the host in Zabbix WebUI
Passive checks - Zabbix frontend
- Log into Zabbix frontend.
- Create a host in Zabbix web interface.
- This host will represent your Linux machine.
- Navigate to “Monitoring” → “Host” and click on “Create host”
- Give it a name
- In the Templates parameter, type or select Linux by Zabbix agent. 
- In Host groups, add “Linux servers” and “Virtual Machines”
- In the Interfaces parameter, add Agent interface and specify the IP address or DNS name of the Linux machine where the agent is installed.
### 3. Configure the Zabbix Agent config file to connect the agent and server
Edit:
```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```
Uncomment or add these comments so that they're enabled:
```
AllowKey=system.run[*]
AllowRoot=1
UnsafeUserParameters=1
Include=/etc/zabbix/zabbix_agentd.d/*.conf
Server=<Zabbix-Server-IP>
ServerActive=<Zabbix-Server-IP>
Hostname=Zabbix server
```
Restart:
```bash
sudo systemctl restart zabbix-agent
```
### 4. Open Zabbix Ports on Wazuh Server Firewall If not Already
```bash
sudo firewall-cmd --zone=public --add-port=10050/udp --permanent
sudo firewall-cmd --zone=public --add-port=10050/tcp --permanent
sudo firewall-cmd --zone=public --add-port=10051/tcp --permanent
sudo firewall-cmd --reload
```
Now go to your Zabbix WebUI and you should see your Wazuh-SITE host with a green ZBX which means its properly connected.
For Active Checks:
- Log in to Zabbix frontend.
- Create a host in Zabbix web interface.
- This host will represent your Linux machine.
- In the Templates parameter, type or select Linux by Zabbix agent active.
Then in the Zabbix agent config change the ServerActive IP to the Zabbix server IP
```bash
ServerActive=<Zabbix-Server-IP>
```
Restart Zabbix agent:
```bash
sudo systemctl restart zabbix-agent
```
---

## Wazuh SITE Disk Space Triggers on Zabbix

### 1. Create Triggers for `/var` Disk Space
Now we will create 3 triggers set to go off when our /var disk space hits 80%, 90%, & 95%. :
First navigate to “Data collection” → “Hosts” → Then scroll down to Wazuh-SITE host and click on “Triggers” next to its name. Then click on “Create trigger” and add the following info:
- Name: /var Disk space is Warningly low (used > {$VFS.FS.PUSED.MAX.WARN:"/var"}%) ({ITEM.VALUE} used)
- Severity: Average
- Expression: last(/Wazuh-SITE/vfs.fs.dependent.size[/var,pused])>{$VFS.FS.PUSED.MAX.WARN:"/var"}
- Description: This trigger activates when: The total disk space used in /var exceeds 80% usage.Take action to free up space or increase the partition size.

Next we can clone the last trigger and edit it to match our 90% trigger:
- Navigate to “Triggers” again for our Wazuh-SITE host, then you will see the new trigger at the top of the list, next we can click on the trigger then click “clone”. Then make sure to make these changes:
- Name: /var Disk space is Critically low (used > {$VFS.FS.PUSED.MAX.CRIT:"/var"}%) ({ITEM.VALUE} used)
- Severity: Average
- Expression: last(/Wazuh-SITE/vfs.fs.dependent.size[/var,pused])>{$VFS.FS.PUSED.MAX.CRIT:"/var"}
- Description: This trigger activates when: The total disk space used in /var exceeds 90% usage. Take action to free up space or increase the partition size.
- Clone 1: /var Disk space is Critically low, triggers when /var disk space is > 90%

Next we can clone the last trigger and edit it to match our 90% trigger:
- Navigate to “Triggers” again for our Wazuh-SITE host, then you will see the new trigger at the top of the list, next we can click on the trigger then click “clone”. Then make sure to make these changes:
- Name: /var Disk space is Disastrously low (used > {$VFS.FS.PUSED.MAX.DISAS:"/var"}%) ({ITEM.VALUE} used)
- Severity: Average
- Expression: last(/Wazuh-SITE/vfs.fs.dependent.size[/var,pused])>{$VFS.FS.PUSED.MAX.DISAS:"/var"}
- Description: This trigger activates when: The total disk space used in /var exceeds 95% usage. Take action to free up space or increase the partition size.
- Clone 2: /var Disk space is Disastrously low, triggers when /var disk space is > 95%

Lastly, Navigate to “Data collection” → “Hosts” → Then scroll down to Wazuh-SITE host and click on its name, you will get a popup, click on “Macros” then click on “Inherited and host macros”, then scroll to bottom:
- Here you will see “{$VFS.FS.PUSED.MAX.CRIT}” and “$VFS.FS.PUSED.MAX.WARN}” set to 90 and 80, which are built in macros for our triggers that sets the value in which the macro is triggered. So we see that we already have these 2 macros set to 90 and 80 which is correct, but we are missing “{$VFS.FS.PUSED.MAX.DISAS}” our third macro which doesn’t exist yet.
- Go ahead and click “Add” at the bottom and add the {$VFS.FS.PUSED.MAX.DISAS} macro, set it to 95, and give it the same description as the others, then click “Updated”
- You should then see all 3 now.

---

## Zabbix Monitoring Wazuh Components, Rancid, & Disk Space

### 1. First we want to make items and triggers for our Wazuh components
Navigate to the Item menu with “Monitoring” → “Hosts” → “Wazuh-SITE” → “Items” → “Create item”
The Item for Wazuh manager, in Wazuh-SITE host:
- We’ll name this Item “Wazuh-Manager Service Status”
- The type will be “Zabbix agent”
- Key will include a system.run command: “system.run[/usr/bin/sudo /usr/bin/systemctl is-active wazuh-manager]”
- Description: Checks the Wazuh Manager status.
### 2. Next we’ll make the Trigger for this Item
Same navigation as Item but choose “Triggers” instead of “Items”
The Trigger for “Wazuh-Manager Service Status” Item:
- Name: Wazuh Manager Service Down ({ITEM.VALUE})
- Severity: High
- Expression: last(/Wazuh-SITE/system.run[/usr/bin/sudo /usr/bin/systemctl is-active wazuh-manager])<>"active"
- note: if you’re not sure what to put in the expression due to unknown syntax, use the “Add” button on the right of expression box.
- Description: The Wazuh Manager in SITE is down.

Now make similar items and triggers for Wazuh indexer and dashboard, just clone the manager item and trigger and change the name/key
### 3. Begin Monitoring the Rancid Back-up Directories
Next we will begin monitoring the Rancid config_backup directories that hold our ASA config backups
First ssh to rancid user in SITE Wazuh box:
```bash
ssh rancid@<Wazuh-Server-IP>
```
Next create this script that will roll through our directories to ensure there was a file made within the last 24hrs:
```bash
cd /home/rancid
mkdir scripts
```
```bash
nano /home/rancid/scripts/check_rancid_backups.sh
```
```bash
#!/bin/bash
# Define directories to check
monthly_dir="/var/config_backup/SITE-monthly"
yearly_dir="/var/config_backup/SITE-yearly"
# Function to check if a backup exists in the last 24 hours
check_backup_age() {
    local dir="$1"
    if find "$dir" -maxdepth 1 -type f -mtime -1 -print -quit | grep -q .; then
        return 0 # Backup found
    else
        return 1 # No backup found
    fi
}
monthly_status=0
yearly_status=0
is_first_of_month=$(date +%d)
# Check monthly directory
if check_backup_age "$monthly_dir"; then
    echo "Monthly backups OK"
else
    echo "ERROR: No new monthly backup found in the last 24 hours."
    monthly_status=1
fi
# Check yearly directory only if it's the first of the month
if [ "$is_first_of_month" -eq "01" ]; then
    if check_backup_age "$yearly_dir"; then
        echo "Yearly backup OK"
    else
        echo "ERROR: No new yearly backup found in the last 24 hours (first of month)."
        yearly_status=2
    fi
else
    echo "Yearly backup check skipped (not the first of the month)."
fi
# Return an overall status code
if [ "$monthly_status" -eq 1 ]; then
    exit 1 # Monthly backup issue
elif [ "$yearly_status" -eq 2 ]; then
    exit 2 # Yearly backup issue (first of month)
else
    exit 0 # All OK
fi
```
Make the script executable:
```bash
chmod +x /home/rancid/scripts/check_rancid_backups.sh
```
Next we will create the Item for rancid in zabbix:
- Name: Rancid Config Back-up Status
- Type: Zabbix agent
- Key: system.run[sudo /home/rancid/scripts/check_rancid_backups.sh > /dev/null; echo $?]
- Type of info: Numeric (unsigned)
- Description: Runs the check_rancid_backups.sh script, then echos exit code for trigger confirmation.
Understanding the New Exit Codes:
- 0: All backups are up-to-date (or the yearly check was skipped and monthly was OK). No Trigger
- 1: No new monthly backup found. Trigger
- 2: No new yearly backup found (on the first of the month). Trigger
Next we will create the first Trigger for rancid monthly dir:
- Name: Rancid: No new monthly backup in the last 24 hours
- Severity: High
- last(/Wazuh-SITE/system.run[sudo /home/rancid/scripts/check_rancid_backups.sh > /dev/null; echo $?])=1
- Description: No new ASA configuration backup found in the SITE-monthly directory in the last 24 hours. Check the Rancid cron job on the Wazuh server.
Next we will create the second Trigger for rancid yearly dir:
- Name: Rancid: No new yearly backup on the first of the month
- Severity: High
- last(/Wazuh-SITE/system.run[sudo /home/rancid/scripts/check_rancid_backups.sh > /dev/null; echo $?])=2
- Description: No new ASA configuration backup found in the SITE-yearly directory on the first of the month. Check the Rancid cron job on the Wazuh server.
Now you can test all items to make sure they run by going to “Monitoring” → “Hosts” → “Wazuh-SITE” → “Items” → Choose your item → “Test” → “Get value and test”
- This will run the command in the key and give back the value that it outputted, if something like “Permission denied” comes back then there is an issue (most likely a SELinux denial), if the intended out put like “active” comes back then its all good.
You can also test the triggers, for example to test the Wazuh-Manager Service Status trigger, stop wazuh manager on the box, then you should receive an alert in slack, make sure to start it back up again.
### 4. Ensure to Make These Fixes
Next we will fix a couple of already found permission issues in Wazuh SITE box
ssh into the box:
```bash
ssh mtagaras@<Wazuh-Server-IP>
```
First issue is changing Zabbix user’s shell to /bin/bash, enabling login:
```bash
sudo usermod -s /bin/bash zabbix
```
Next add Zabbix to the Wheel group:
```bash
sudo usermod -aG wheel zabbix
```
Then we will add a Zabbix file to the suoders.d directory, to allow command permissions without the need for sudo for zabbix user:
```bash
sudo nano /etc/sudoers.d/zabbix
```
- add this to file then save: zabbix ALL=(ALL) NOPASSWD: ALL
Test that we are good by switching users to zabbix bash and run the system.run command and you should be able to thanks to our changes:
```bash
sudo su zabbix
```
```bash
/usr/bin/sudo /usr/bin/systemctl is-active wazuh-manager
```

Next issue was a lot of SELinux command denials, we need to create a policy to allow zabbix permission to run these commands through SELinux:
We must first search for the SELinux denials we’re getting from testing the zabbix item
- To test the item go to “Monitoring” → “Hosts” → “Wazuh-SITE” → “Items” → Choose your item → “Test” → “Get value and test”
- Next we will run a couple commands in the Wazuh SITE server CLI to search for denials received from testing:
```bash
sudo cat /var/log/audit/audit.log | grep AVC | tail -n 20
```
This^ reads the audit log, filters for lines containing "AVC" (SELinux denials), and displays the last 20 entries, showing recent permission failures.
```bash
sudo ausearch -m avc | grep zabbix_agent_t | audit2allow -M zabbix_agent_policy
```
This^ Extracts denials from ausearch. Converts them into an SELinux policy module. Saves the module as zabbix_agent_policy.te (source) and zabbix_agent_policy.pp (compiled).
- Run this to apply the policy and make it active:
```bash
sudo semodule -i zabbix_agent_policy.pp
```
Once applied we can test the item by going to “Monitoring” → “Hosts” → “Wazuh-SITE” → “Items” → Choose your item → “Test” → “Get value and test” , you will see permission denied or your expected output, if denied then you still got denials coming.
- Continue using this command to find new denials as you go:
```bash
sudo cat /var/log/audit/audit.log | grep AVC | tail -n 20
```
You will fix these denials by adding new permissions to the policy by editing and applying the updated policy below

Use this to Edit the policy when new denials come in:
```bash
nano zabbix_agent_policy.te
```
Add all allow policu updates to this file (there’s a lot):
```bash
module zabbix_agent_policy 1.0;
require {
    type zabbix_agent_t;
    type sudo_exec_t;
    type chkpwd_exec_t;
    type devlog_t;
    type var_run_t;
    type auditd_var_run_t;
    type kernel_t;
    type etc_t;
    type passwd_file_t;
    type shadow_t;
    type systemd_systemctl_exec_t;
    type systemd_unit_file_t;
    type init_t;
    type user_home_t;
    class file { execute execute_no_trans open read getattr write append map status };
    class dir { search open read getattr };
    class process { transition signal execmem execstack };
    class unix_stream_socket { connectto };
    class unix_dgram_socket { create connect write sendto read getattr };
    class netlink_audit_socket { create connect write read nlmsg_relay };
    class sock_file { write getattr open read };
    class capability { audit_write audit_control setuid dac_override fowner sys_admin dac_read_search };
    class service { status };
}
# Allow zabbix_agent_t to execute the sudo binary
allow zabbix_agent_t sudo_exec_t:file { execute execute_no_trans open read getattr map };
# Allow zabbix_agent_t to execute unix_chkpwd for PAM authentication
allow zabbix_agent_t chkpwd_exec_t:file { execute execute_no_trans open read getattr map };
# Allow zabbix_agent_t to execute systemctl with required permissions
allow zabbix_agent_t systemd_systemctl_exec_t:file { execute execute_no_trans open read getattr map };
# Allow zabbix_agent_t to connect to PAM sockets
allow zabbix_agent_t var_run_t:unix_stream_socket connectto;
# Allow zabbix_agent_t to write to /run/systemd/journal/dev-log
allow zabbix_agent_t devlog_t:sock_file { write getattr open read };
allow zabbix_agent_t kernel_t:unix_dgram_socket { sendto connect write read };
# Allow zabbix_agent_t to access auditd files and directories
allow zabbix_agent_t auditd_var_run_t:dir { search open read getattr };
allow zabbix_agent_t auditd_var_run_t:file { write open read getattr };
# Allow zabbix_agent_t to create and connect to netlink audit sockets
allow zabbix_agent_t self:netlink_audit_socket { create connect write read nlmsg_relay };
# Allow zabbix_agent_t to create and connect to unix dgram sockets
allow zabbix_agent_t self:unix_dgram_socket { create connect write sendto read getattr };
# Allow zabbix_agent_t to use audit_write, audit_control, and administrative capabilities
allow zabbix_agent_t self:capability { audit_write audit_control setuid dac_override fowner sys_admin dac_read_search };
# Allow process transition and signaling for sudo execution
allow zabbix_agent_t self:process { transition signal execmem execstack };
# Allow zabbix_agent_t to access /etc/sudoers.d/zabbix and related files
allow zabbix_agent_t etc_t:dir { search open read getattr };
allow zabbix_agent_t etc_t:file { open read getattr };
# Allow zabbix_agent_t to read passwd and shadow files if needed
allow zabbix_agent_t passwd_file_t:file { open read getattr };
allow zabbix_agent_t shadow_t:file { open read getattr };
# Allow zabbix_agent_t to interact with system journal
allow zabbix_agent_t devlog_t:sock_file { write getattr open read };
allow zabbix_agent_t kernel_t:unix_dgram_socket { sendto connect write read };
# Handle interactions with PAM and logging
allow zabbix_agent_t self:unix_dgram_socket { write sendto };
# Grant extended permissions for process management and transitions
allow zabbix_agent_t self:process { transition signal };
# Allow zabbix_agent_t to send to systemd journal log sockets
allow zabbix_agent_t devlog_t:unix_dgram_socket { write sendto };
# Allow zabbix_agent_t to interact with devlog and journald
allow zabbix_agent_t kernel_t:unix_dgram_socket { sendto write connect read };
# Allow zabbix_agent_t to interact with /usr/sbin/unix_chkpwd
allow zabbix_agent_t chkpwd_exec_t:file { execute_no_trans map };
# Allow zabbix_agent_t to check the status of systemd unit files
allow zabbix_agent_t systemd_unit_file_t:service status;
# Allow zabbix_agent_t to connect to systemd's private socket
allow zabbix_agent_t init_t:unix_stream_socket connectto;
# Allow zabbix_agent_t to execute scripts in user_home_t
allow zabbix_agent_t user_home_t:file { execute execute_no_trans open read getattr };
```
Next use these to compile and apply the policy:
```bash
checkmodule -M -m -o zabbix_agent_policy.mod zabbix_agent_policy.te
semodule_package -o zabbix_agent_policy.pp -m zabbix_agent_policy.mod
sudo semodule -i zabbix_agent_policy.pp
```
If more denials come up use this or the other command to find out what they are and add to the SELinux policy then recompile and reapply it:
```bash
sudo tail -f /var/log/audit/audit.log
```

---

## 👨‍💻 Author  
Mario Tagaras | Cybersecurity Engineer | Florida State University Alum  
