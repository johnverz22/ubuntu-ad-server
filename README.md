## Project: The "Zero-License" Unified Lab

### 1. The Core Objective
The goal is to provide **400 students** with a **Single Sign-On (SSO)** experience across **4 physical rooms** (Windows and macOS). A student should be able to sit at any workstation, log in with their unique school ID, and find their environment, SSH keys, and local files exactly as they left them—without the school paying for Windows Server or MDM licenses.

### 2. High-Level Architecture
We are utilizing a **Hybrid-Local Model**:
* **Identity (The Brain):** An Ubuntu Server running **Samba 4** acts as a Domain Controller. It handles passwords and security "tickets" (Kerberos).
* **Storage (The Muscle):** We leverage the **local SSDs** of the lab computers for performance. Files are not "streamed" from the server; they are **cached locally** and **synchronized** to the server upon logout.
* **Data Strategy:** Since 90% of work is stored on **GitHub**, the server primarily manages "identity" and "configuration" (SSH keys, Git configs, browser bookmarks) rather than massive project files.

### 3. Key Design Principles
* **Data Mobility:** Profiles "follow" the student from Room 1 to Room 4.
* **Privacy by Isolation:** Each student’s local profile is isolated. Student A cannot browse Student B’s local home directory.
* **Resilience:** If the network or server experiences a momentary blip, the student can continue working on their local files because of the **Offline Files (Windows)** and **Mobile Account (Mac)** architecture.
* **Scalability:** Bulk registration scripts allow for onboarding entire semesters of students in seconds.

---

## I. Pre-Deployment: The Virtual Sandbox
Before hardware deployment, build a **Proof of Concept (PoC)** to validate the logic.
* **Hypervisor:** Use VirtualBox, VMware, or Proxmox.
* **Virtual Network:** Set to "Internal Network" to isolate DNS/DHCP from the school's main grid.
* **VM Specs:** * **Server:** 4 vCPUs, 8GB RAM, 100GB SSD (Ubuntu Server).
    * **Clients:** 1 Windows 11 Pro VM, 1 macOS VM.

---

## II. Server Setup: Ubuntu Samba 4 AD
This turns Ubuntu into a Domain Controller (DC) that Windows and Macs treat as a genuine Microsoft Server.

### 1. Networking & Hostname
```bash
# Set Static IP
sudo nano /etc/netplan/00-installer-config.yaml
# Ensure hostname is FQDN
sudo hostnamectl set-hostname dc1.lab.school.edu
```

### 2. Installation
```bash
sudo apt update
# When Kerberos prompt appears, enter REALM in ALL CAPS: LAB.SCHOOL.EDU
sudo apt install -y samba krb5-config krb5-user winbind libpam-winbind libnss-winbind
```

### 3. Provisioning the Domain
```bash
sudo rm /etc/samba/smb.conf
sudo samba-tool domain provision --use-rfc2307 --interactive
# Settings: Realm=LAB.SCHOOL.EDU, Domain=LAB, Role=dc, DNS=SAMBA_INTERNAL
```

### 4. Create the Profile Sync Share
```bash
sudo mkdir -p /srv/samba/profiles
sudo chmod 1777 /srv/samba/profiles
# Add to /etc/samba/smb.conf:
# [profiles]
#    path = /srv/samba/profiles
#    read only = no
#    vfs objects = acl_xattr
```

---

## III. Client Configuration (4 Rooms)

### 1. Windows Clients (Rooms 1-3)
* **DNS:** Set Primary DNS to the **Ubuntu Server IP**.
* **Join Domain:** System > About > Domain or Workgroup > Change > Domain: `LAB.SCHOOL.EDU`.
* **Installer Deployment:** Create a share `\\dc1\installers`. Use a **Startup Script** (GPO) to run silent installers:
    `msiexec /i "\\dc1\installers\vscode.msi" /quiet`

### 2. macOS Clients (Room 4)
* **Binding:** System Settings > Users & Groups > Network Account Server > Join.
* **Persistence:** **Crucial:** Check **"Create mobile account at login."**
    * This ensures the student's home folder is physically stored on the Mac's local disk.
* **Login Experience:** Users see a standard Mac login but enter their AD credentials.

---

## IV. The "Local-First" Sync Strategy
You want files to persist on the last client used but follow the student. 

### 1. Windows: Folder Redirection + Offline Files
Use **RSAT** on a Windows machine to manage these GPOs:
* **Folder Redirection:** Redirect `Desktop` and `Documents` to `\\dc1\profiles\%username%`.
* **Offline Files:** Enable "Allow use of Offline Files."
    * *How it works:* Windows keeps a permanent cache in `C:\Windows\CSC`. The student works at SSD speeds. On logout, changes sync to Ubuntu. On a new machine, files download once and stay there.

### 2. Git/GitHub Optimization
Since 90% use GitHub, add a login script to the GPO:
```batch
:: Auto-configure Git on login
git config --global user.name "%username%"
git config --global user.email "%username%@school.edu"
:: Ensure .ssh folder is part of the redirected profile
```

---

## V. Restrictions & Lockdown (GPO & Profiles)

### 1. Windows Lockdown (via GPMC)
* **Prevent Installs:** `Computer Configuration > Windows Settings > Security Settings > Software Restriction Policies`.
* **Disable CMD/Registry:** `User Configuration > Admin Templates > System > Prevent access to command prompt`.
* **Web Restrictions:** Use Chrome/Edge ADMX templates to set `URLBlocklist`.

### 2. macOS Lockdown
Since there is no native GPO for Mac, use **iMazing Profile Editor** to create a `.mobileconfig` file and install it on the Mac fleet:
* **Payloads:** Restrict System Settings, App Store, and set Safari Content Filters.

---

## VI. User Management CLI Reference

| Action | Command |
| :--- | :--- |
| **Add Student** | `samba-tool user create student.name P@ssword123` |
| **Reset Password** | `samba-tool user setpassword student.name` |
| **Create Lab Group** | `samba-tool group add "Room1_Students"` |
| **Add to Group** | `samba-tool group addmembers "Room1_Students" student.name` |

---

## VII. Validation Checklist for Evaluators
1.  **DNS Check:** Can the client ping `dc1.lab.school.edu`?
2.  **Auth Check:** Can a student log into a Windows PC and a Mac using the same credentials?
3.  **Sync Check:** If a student creates a file on the Desktop in Room 1, does it appear when they log into Room 2? (Note: First sync will take time; subsequent logins should be instant).
4.  **GitHub Check:** Does the `.ssh` key persist across different machines?
5.  **Persistence Check:** After logging out, does the user folder still exist on the local disk (C:\Users or /Users)?

---

To make this guide production-ready for 400+ students, you need an automated way to handle onboarding. Manually typing `samba-tool` for every student is not feasible.

Here is the addition to your guide for **Bulk User Provisioning** and the final **Validated Command List**.

---

## VIII. Bulk Student Registration (Automation)

The most efficient way to register 400 students is using a Bash script on the Ubuntu server that reads a `.csv` file.

### 1. The CSV Format
Create a file named `students.csv`. Ensure it has no header row and follows this format:
`username,password,firstname,lastname`

**Example:**
```csv
j.doe,StudentPass123,John,Doe
s.smith,SecurePass456,Sarah,Smith
```

### 2. The Bulk Import Script
Create a script named `bulk_users.sh`:
```bash
#!/bin/bash
# Usage: sudo ./bulk_users.sh students.csv

if [ "$#" -ne 1 ]; then
    echo "Usage: $0 <csv_file>"
    exit 1
fi

CSV_FILE=$1

while IFS=, read -r username password fname lname
do
    echo "Registering $username..."
    # Create user with defined attributes
    samba-tool user create "$username" "$password" \
        --given-name="$fname" \
        --surname="$lname" \
        --use-username-as-cn
        
    # Optional: Force password change on first login
    samba-tool user setexpiry "$username" --days=0
    
    # Add to a global 'Students' group
    samba-tool group addmembers "Domain Users" "$username"
    
done < "$CSV_FILE"

echo "Bulk registration complete."
```
**To Run:** `sudo chmod +x bulk_users.sh && sudo ./bulk_users.sh students.csv`

---

## IX. Managing the "Last Client" Persistence (Logon Scripts)

To ensure the environment is ready for coding and GitHub, you can use a **Logon Script** pushed via GPO (Windows) or a **LaunchAgent** (Mac).

### 1. Windows Git Auto-Config (Batch Script)
Save this as `git-setup.bat` in your `SYSVOL` folder:
```batch
@echo off
:: Set Git to use the redirected profile for config
setx HOME %HOMEDRIVE%%HOMEPATH%

:: Configure Git identity using AD Environment Variables
git config --global user.name "%username%"
git config --global user.email "%username%@school.edu"

:: Ensure SSH directory exists in the redirected folder
if not exist "%HOMEPATH%\.ssh" mkdir "%HOMEPATH%\.ssh"
```

### 2. macOS Persistence Check (Shell Script)
Since you are using **Mobile Accounts**, the files persist locally by default. However, you can run this script to ensure the GitHub SSH keys are symlinked correctly to the local drive:
```bash
#!/bin/zsh
# Ensures SSH keys are stored on the local persistent disk
USER_HOME="/Users/$USER"
if [ ! -d "$USER_HOME/.ssh" ]; then
    mkdir -p "$USER_HOME/.ssh"
    chmod 700 "$USER_HOME/.ssh"
fi
```

---

## X. Summary of All Up-to-Date CLI Commands

| Category | Command | Description |
| :--- | :--- | :--- |
| **Service** | `sudo systemctl stop smbd nmbd winbind` | Stop services before provisioning |
| **Service** | `sudo systemctl disable smbd nmbd winbind` | Required for AD DC mode |
| **Service** | `sudo systemctl unmask samba-ad-dc` | Enable the AD specific service |
| **Service** | `sudo systemctl start samba-ad-dc` | Start the Domain Controller |
| **User** | `samba-tool user list` | List all registered students |
| **User** | `samba-tool user delete <username>` | Remove a student account |
| **Group** | `samba-tool group list` | View all room/class groups |
| **DNS** | `samba-tool dns query 127.0.0.1 lab.school.edu @ ALL` | Verify DNS is working |

---

## XI. Troubleshooting for Validators

If a student cannot log in, have the validators check these three things:
1.  **Time Sync:** Run `ntpdate -q <server_ip>`. If the client and server clocks differ by more than 5 minutes, Kerberos will reject the login.
2.  **DNS Reachability:** On the client, run `nslookup lab.school.edu`. It **must** return the Ubuntu Server's IP.
3.  **Local Profile Conflict:** If a student previously logged in as a "Local User" with the same name, the AD login may fail. Delete the local profile first.

