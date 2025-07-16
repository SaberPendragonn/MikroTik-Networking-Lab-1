# 📘 MikroTik Networking Labs – Introduction & Setup

This lab section walks through the initial configuration and setup process for MikroTik RouterOS. Topics include Winbox and PuTTY access, identity customization, upgrades, and backup management — essential for network technicians working with MikroTik-based infrastructures.

---

## 🛠 Topics Covered

- Initial configuration of MikroTik RouterOS (CHR or physical)
- Winbox GUI and PuTTY terminal access
- Router identity and admin password setup
- RouterOS upgrades via Winbox and CLI
- Backup and restore using `.backup` and `.rsc` files

---

## 🎯 Lab Objectives

- Access the router using both **Winbox** and **PuTTY**
- Change default system identity and secure the device
- Upgrade RouterOS to the latest stable version
- Create and restore system backups
- Prepare the device for further networking configurations

---

## 🖼️ Screenshots & Walkthroughs

### 🔹 1. Accessing MikroTik via Winbox and PuTTY

> **Winbox GUI access using MAC & IP**  
> ![Winbox Access Screenshot](https://i.imgur.com/yourimage_winbox.png)

> **PuTTY CLI login using SSH**  
> ![PuTTY SSH Screenshot](https://i.imgur.com/yourimage_putty.png)

- Winbox provides a quick and visual way to configure your router.  
- PuTTY allows remote CLI access via SSH for terminal-level control.

---

### 🔹 2. Identity and Password Configuration

> **Customize system identity and set a secure password**

![System Identity Screenshot](https://i.imgur.com/yourimage_identity.png)

- Set hostname to `Lab-Router01`  
- Changed default admin password

---

### 🔹 3. RouterOS Upgrade Process

> **Using Winbox: System → Packages → Check for Updates**

![RouterOS Upgrade Screenshot](https://i.imgur.com/yourimage_upgrade.png)

- Checked for the latest stable version and applied update

---

### 🔹 4. Backup & Restore Configuration

> **Creating `.backup` and exporting `.rsc` script file**

![Backup Screenshot](https://i.imgur.com/yourimage_backup.png)

- `.backup` preserves current state (binary format)  
- `.rsc` is a human-readable export (for reviewing or cloning configs)

---

### 🔹 5. Restoring Settings

> **Restore configurations via Files panel or CLI**

![Restore Screenshot](https://i.imgur.com/yourimage_restore.png)

- Demonstrated backup recovery using both Winbox and terminal

---

## 🔗 Return to Lab Directory

⬅️ [Back to MikroTik Networking Labs Main Page](https://github.com/yourusername/MikroTik-Networking-Labs)

---

## 🚀 Coming Up Next

➡️ [Routing Labs](https://github.com/yourusername/mikrotik-routing-labs)  
➡️ [Firewall & NAT Labs](https://github.com/yourusername/mikrotik-firewall-nat-labs)

---
