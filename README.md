# SOC-Exercise-In-order-to-Detect-Stop-Brute-Force-like-Activity

# Lab Setup

## Tools I used for this:
    - Splunk Enterprise (your SIEM); Splunk Universal Forwarder
    - Windows Server (AD DC) – logs forwarded to Splunk
    - Windows 10 client – logs forwarded to Splunk
    - Kali Linux – used to simulate the attack (safe mode)
    -Test user account (e.g., labtest) with a weak password for the simulation
    - VirtualBox internal network:
        * Network Map
          For my homelab attack path:
              [Kali Linux Attacker] --(Internal Network)--> [Windows Server DC] <--> [Windows 10 Client]
    - All machines share the same subnet (192.168.56.x) 
    - Splunk sits on the same network or gets logs via forwarders.
         - I used an Internal Network or Host-only Adapter in VirtualBox, so it’s isolated
         - All VMs on the same subnet for communication (192.168.56.0/24)

# Step-by-Step: Simulated Brute Force Attack Detection
## Splunk for detecting & alerting
      a. Download Splunk Universal Forwarder https://www.splunk.com/en_us/download/universal-forwarder.html
          - Select Windows 64-bit .msi package (for Windows machines).
      b. Install on Windows Server (AD DC) & Windows 10 Client
          - Double-click the .msi file.
          - In the Setup Wizard: Accept license.
          - Choose "Forward data to another Splunk instance".
          - Receiver Host: enter your Splunk server IP (e.g., 192.168.56.***).
          - Receiver Port: 9997 (default Splunk receiving port).
          - Set admin credentials for the forwarder.
          - Finish installation.
     c. Enable Splunk Receiving on Your Splunk Server
          - On the Splunk Enterprise server: Settings → Forwarding and Receiving → Configure Receiving.
          - Click Add new → enter port 9997 → Save.
     d. Configure Forwarder to Send Security Logs to forward Windows Security, System, and Application logs.
     e.On the Windows Forwarder: 
         - Open Command Prompt as Administrator.
         - Navigate to the forwarder’s bin folder:
                cd "C:\Program Files\SplunkUniversalForwarder\bin"
     f. Add your Splunk indexer as a target:
                splunk add forward-server 192.168.10.50:9997 -auth admin:yourforwarderpassword
     g. Enable Windows Event Logs:
                splunk add monitor "WinEventLog:Security"
                splunk add monitor "WinEventLog:System"
                splunk add monitor "WinEventLog:Application"
     h. Restart the forwarder: splunk restart
     i. Verify Data in Splunk
          - In Splunk Search:
               index=main host="WIN-SERVER01" or
               index=main sourcetype="WinEventLog:Security"
          - The Logs have started showing
    j. Installed the forwarder on the Windows server client and any other log source in my lab.
          - Used the same Splunk server IP and port 9997.
##  1) Prepared my AD & Logging
    a) On Windows Server: I created a test user: labtest;
    - Password: something easy like Password1! (only for the lab)
## * Enabled auditing:
     - Open Group Policy Management → Default Domain Policy →
       Computer Configuration → Windows Settings → Security Settings → Advanced Audit Policy Configuration
     - Enable: Audit Logon → Success & Failure
       Audit Account Logon → Success & Failure
       Audit Account Lockout → Success & Failure
## * Set Account Lockout Policy:
    - Lockout threshold: 10
    - Lockout duration: 15 minutes
    - Reset counter: 15 minutes
## b) I made sure Windows Event Logs are being forwarded to Splunk.

## 2) Configure Splunk Inputs
    a) Installed Splunk Universal Forwarder on:
          - Windows Server (AD DC)
          - Windows 10 client
    b) Forward Security Logs to Splunk.
    c) In Splunk, index these logs under winsec.
## 3) Simulate the Brute Force
    a) I used Hydra on Kali, but in a safe way.
       - On Kali:
           sudo apt install hydra -y
      - I created a simple password list (passlist.txt) containing only wrong passwords:
           echo -e "wrongpass1\nwrongpass2\nwrongpass3\nwrongpass4\nwrongpass5" > passlist.txt
      - Ran Hydra against the Windows Server (RDP), targeting the test account:
           hydra -l labtest -P passlist.txt rdp://192.168.56.10
      - You can replace the 192.168.56.10 with your Windows Server IP.
      - This failed every time but generated real failed logins.
## 4) I detected the failed login in Splunk
    - I used the easy Splunk searches for brute-force detection:
    a. Multiple failed logins
         - index=winsec EventCode=4625
         - stats count as failures values 192.168.56.*** as src_ips by TargetUserName
         - where failures >= 5
         - sort -failures
    b. Account lockouts
         - index=winsec EventCode=4740
         - stats count by TargetUser, Windows 10
    C. RDP-specific failures
         - index=winsec EventCode=4625 LogonType=10
         - stats count by TargetUserName, IpAddress
         - where count >= 5
## 5. I built an alert
    In Splunk (Optional):
       - I saved the “Multiple failed logins” search.
       - I set it to run every 5 minutes; I triggered when results > 0.
## 6. Harden After the Test
    Disable password auth for RDP/SSH (use keys or NLA).
    Kept the lockout policies enabled.
