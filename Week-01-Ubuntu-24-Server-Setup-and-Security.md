# Week 1: Ubuntu 24 Server Setup & Security

**Challenge:** 6-Week Linux System Administration Challenge   
**Date:** 28/06/2026

## Goal

To successfully provision and update an Ubuntu 24.04 VPS, navigate the filesystem, monitor system resources, and implement the Principle of Least Privilege using user groups and strict directory permissions.

## Tasks

- Provisioned and updated the base Ubuntu 24.04 environment.
- Created a dedicated administrative user to bypass default root access.
- Navigated, created, and manipulated nested filesystem structures.
- Monitored active system processes and safely terminated targeted scripts.
- Architected a secure organisational directory structure utilising user groups.
- Applied strict octal permissions to enforce data isolation between departments.

## Commands Used

### 1. System Setup & Initial Connectivity

```bash
# Establishing Secure Shell (SSH) Connectivity
ssh ubuntu@<ip_address>

# Update package lists and upgrade system (using -y to auto-confirm)
sudo apt update && sudo apt upgrade -y
```

#### Key Findings: The && Trap

- && successfully chains terminal commands, sudo privileges do not bridge across the operator.
- The second command executes as a standard user, demonstrating strict privilege escalation

### 2. User & Identity Management

```bash
# Create a new standard user and assign a password
sudo adduser <username>

# Append the new user to the sudo group for administrative privileges
sudo usermod -aG sudo <username>

# Switch to the new user account (The '-' loads a fresh login shell)
sudo su - <username>

# Identity Verification Commands
whoami														# Confirms current effective user
groups <username>								# Lists all groups associated with the user
id <username>										# Displays UID, GID, and group memberships
sudo -l -U <username>						# Audits privileges; shows which binaries the user can run
```

#### Key Findings: The -aG Flag

- \-a (append) is strictly required when modifying groups.
- Omitting -a wipes all of the user's existing group memberships.
- Forgetting -a on an administrator causes an instant sudo lockout.

### 3. Filesystem Architecture and Data Manipulation

```bash
# Navigation and Orientation
pwd                            # Print working directory to confirm current location
ls                             # List contents (use 'ls -la' for hidden files/details)
ls -ld /path/to/dir            # Long-list directory details to check permissions
cd /home/<username>            # Change to specific directory ('cd' alone returns home)
cd ..                          # Move up one level in the hierarchy

# Directory and File Management
mkdir projects                 # Create a single directory
mkdir -p website/public/css    # -p creates parent directories and prevents errors
cp -r projects projects2       # Recursively copy a directory and its entire contents
mv oldname.txt newname.md      # Rename or move files
rm -r projects                 # Recursively remove a directory (use with extreme caution)
rm todo.txt                    # Remove a single file

# Visualisation and Editing
sudo apt install tree -y       # Install the tree utility
tree website                   # Display a visual map of the directory structure
nano todo.txt                  # Quick edits for simple configuration files
vim main.py                    # Advanced editor for programming and complex configs
```

#### Key Findings: CLI Text Editors

- nano: Ideal for rapid, simple text edits.
- vim: Advanced editor for complex, multi-line configurations.

### 4. Resource Monitoring & Process Management

```bash
# Launch interactive process monitors
top                    					# Standard summary of system health and load averages
htop                    				# Interactive monitor (use F3 - Search, F4 - Filter, F9 - Kill)

# Find the Process ID (PID) and name of a specific application
pgrep -l python3        

# Terminate a process via its unique ID
kill <PID>
```

#### Key Findings: Signals & Priorities

- 15 SIGTERM: Triggers a safe termination.
- 9 SIGKILL: Forces an immediate kill.
- Niceness (PRI/NI): Standard users can only lower CPU priority. Only root can raise it.

### 5. Directory Architecture & Enforcing the Principle of Least Privilege (PoLP)

```bash
# 1. Create the organisational groups
sudo groupadd developers
sudo groupadd accounting
sudo groupadd hr
sudo groupadd management

# 2. Create the test users
sudo adduser andy
sudo adduser bob
sudo adduser colin
sudo adduser emma

# 3. Assign users to supplementary groups
sudo usermod -aG developers andy
sudo usermod -aG accounting bob
sudo usermod -aG hr colin
sudo usermod -aG management emma

# 4. Build the directory structure
sudo mkdir -p /data/developers /data/accounting /data/hr /data/management

# 5. Set Ownership (Root User : Department Group)
sudo chown root:developers /data/developers
sudo chown root:accounting /data/accounting
sudo chown root:hr /data/hr
sudo chown root:management /data/management

# 6. Set Strict Permissions (770 - Owner/Group = Full Access, Others = None)
sudo chmod 770 /data/developers
sudo chmod 770 /data/hr
sudo chmod 770 /data/management

# 7. Set Read-Only Group Permissions (750 - Owner = Full, Group = Read-Only, Others = None)
sudo chmod 750 /data/accounting
```

#### Key Findings: Octal Logic

- Permissions sum octal values: Read (4), Write (2), Execute (1).
- 770 (rwxrwx---): Full access for owner and group. Ideal for team collaboration.
- 750 (rwxr-x---): Read-only access for the group. Strictly protects sensitive data integrity.

### Verification

- **Permission Test 1 (Success):** Switched to user andy (sudo su - andy) and successfully created a file inside /data/developers using touch test.txt.  
- **Permission Test 2 (Blocked as Expected):** Attempted to list contents of /data/accounting while logged in as andy; received Permission denied.  
- **Permission Test 3 (Read-Only Group):** Switched to user bob (Accounting). Successfully viewed the directory (ls /data/accounting) but received Permission denied when attempting to write a new file (touch /data/accounting/new-file.txt).  
- **Permission Test 4 (Unaffiliated User):** Switched to user emma (Management). Attempted to access developers and accounting directories; received Permission denied for both. Checked directory permissions globally using ls -ld and ls -l /data to confirm the modes were successfully applied.