# Splunk SOC Home Lab

A home lab built to practice the daily work of a SOC analyst: stand up a Windows domain, ship its logs to a SIEM, run a brute-force attack against it, and write a detection and alert for it.

**Status:** Part 1 (Active Directory domain) is complete. Parts 2 to 4 (Splunk, log forwarding, attack/detect/alert) are in progress. This README is updated as each part is finished.

| Part | What | Status |
|------|------|--------|
| 1 | Windows Server 2022 domain controller + Windows 11 domain client on an isolated network | Done |
| 2 | Splunk Enterprise on Ubuntu Server 24.04 | In progress |
| 3 | Audit policy + Splunk Universal Forwarders on both Windows machines | Planned |
| 4 | Brute-force attack, detection search (Event IDs 4625 / 4740), scheduled alert | Planned |

## Lab design

Everything runs in VirtualBox on a single host, on an isolated NAT Network so the lab machines can talk to each other and reach the internet, but nothing outside can reach them.

```
                 NAT Network "SOCLab"  192.168.10.0/24  (VirtualBox DHCP off)

   +--------------------+      +--------------------+      +--------------------+
   | DC01               |      | CLIENT01           |      | Splunk (Part 2)    |
   | Windows Server 2022|      | Windows 11         |      | Ubuntu Server 24.04|
   | AD DS, DNS, DHCP   |      | domain-joined      |      | Splunk Enterprise  |
   | 192.168.10.7       |      | 192.168.10.100     |      | 192.168.10.10      |
   +--------------------+      +--------------------+      +--------------------+
```

| Machine | OS | Resources | Address | Role |
|---------|----|-----------|---------|------|
| DC01 | Windows Server 2022 Standard Evaluation (Desktop Experience) | 4 GB RAM, 2 CPU, 50 GB | 192.168.10.7 (static) | Domain controller, DNS, DHCP |
| CLIENT01 | Windows 11 | 4 GB RAM, 2 CPU, 64 GB | DHCP from DC01 (192.168.10.100) | Domain workstation, attack source |
| Splunk | Ubuntu Server 24.04 | 4 GB RAM, 2 CPU, 50 GB | 192.168.10.10 (static, planned) | SIEM |

- Domain: `soclab.local` (NetBIOS `SOCLAB`)
- DHCP scope: 192.168.10.100 to 192.168.10.200, router 192.168.10.1, DNS 192.168.10.7, domain `soclab.local`
- Active Directory: OU `Lab Users` containing `labuser` (interactive login on the client) and `jsmith` (the planned attack target)

## Part 1: building the domain

1. **Isolated network.** Created a NAT Network named `SOCLab` (192.168.10.0/24) in VirtualBox with its DHCP turned off, so the domain controller hands out addresses instead. ([screenshot 1](screenshots/01-nat-networks-before-soclab.png))
2. **DC01 VM.** Windows Server 2022 evaluation ISO, unattended install skipped, adapter attached to `SOCLab`. ([2](screenshots/02-dc01-select-server-iso.png), [5](screenshots/05-dc01-adapter-soclab.png))
3. **Install Windows Server** with the Desktop Experience edition. ([4](screenshots/04-server-2022-edition-select.png))
4. **Static IP and name.** 192.168.10.7 / 255.255.255.0, gateway 192.168.10.1, DNS pointing at itself (127.0.0.1). Renamed to DC01. Verified with `ipconfig`. ([7](screenshots/07-dc01-server-manager.png), [8](screenshots/08-dc01-static-ip-settings.png), [9](screenshots/09-dc01-ipconfig-verified.png))
5. **Roles.** Installed AD DS, DNS Server and DHCP Server. ([10](screenshots/10-dc01-roles-installed.png))
6. **Promoted to domain controller** with a new forest, `soclab.local`. The two prerequisite warnings (weaker cryptography default, DNS delegation) are expected with no parent DNS zone. ([11](screenshots/11-dc01-promotion-prereq-check.png))
7. **DHCP scope** 192.168.10.100 to .200 with router, DNS server and DNS domain options. ([12](screenshots/12-dhcp-scope-options.png))
8. **Users.** Created the `Lab Users` OU with `labuser` and `jsmith`. ([13](screenshots/13-ad-lab-users-ou.png))
9. **CLIENT01.** Windows 11 VM on `SOCLab`. It got 192.168.10.100 from the DC's DHCP, confirming DHCP and DNS work end to end. ([6](screenshots/06-client01-adapter-soclab.png), [14](screenshots/14-client01-ipconfig-dhcp.png))
10. **Joined the domain.** CLIENT01 appears under Computers in Active Directory Users and Computers, and `whoami` after logging in returns `soclab\labuser`. ([15](screenshots/15-ad-computers-client01-joined.png), [16](screenshots/16-client01-whoami-labuser.png))

## What broke and how I fixed it

- **The SOCLab network showed the default 10.0.2.0/24 prefix and DHCP on.** The edit had not been applied. Retyped the prefix, unchecked DHCP, clicked Apply, then re-selected the row to confirm it stuck. ([screenshot 1](screenshots/01-nat-networks-before-soclab.png))
- **DC01's adapter was set to plain NAT, so the network name field was greyed out.** Changing "Attached to" from NAT to NAT Network unlocks it. ([3](screenshots/03-dc01-adapter-plain-nat-wrong.png))
- **The Windows Server installer restarted from the beginning after its first reboot.** I pressed a key at "Press any key to boot from CD". Fix: powered off, moved Hard Disk to the top of the boot order, restarted. The long black screen on first boot is normal.
- **The DHCP post-install wizard appeared before the server was promoted.** Cancelled it, promoted to a domain controller first, then completed DHCP configuration.
- **A typo in the DNS option (`192.68.10.7`).** Caught it when checking the scope options and corrected it to `192.168.10.7`. DHCP clients would otherwise have received a dead DNS server.
- **Active Directory rejected a password.** The default complexity policy refuses passwords that contain three or more consecutive characters of the account's own name. Used a password that doesn't.
- **CLIENT01 stayed on a black screen at first boot.** Boot order was greyed out because the VM uses UEFI (normal), and the ISO and settings were correct. Rebuilding the VM did not help. Fix: Settings > Display > Graphics Controller from VBoxSVGA to VMSVGA.
- **Windows 11 setup demanded a network and an account.** Used the local-account route (Shift+F10 and `start ms-cxh:localonly`) to create a local account, then joined the domain afterwards.
- **The first domain join failed with "user name or password is incorrect".** Reset the AD account passwords in Active Directory Users and Computers and retried; the join succeeded.

## What's next

- Take a clean-state snapshot of both VMs (`Day0-domain-working`)
- Splunk Enterprise on Ubuntu Server 24.04, static IP 192.168.10.10, receiving on port 9997, index `endpoint`
- Advanced audit policy via Group Policy (logon, credential validation, account management) and an account lockout threshold of 5
- Splunk Universal Forwarder on DC01 and CLIENT01, shipping Windows Security and System logs
- Run a PowerShell brute-force against `jsmith`, find it with Event ID 4625 (failed logon) and 4740 (account lockout), and write an SPL detection and a scheduled alert

## Notes

Evaluation ISOs and throwaway credentials are used throughout. No credentials are stored in this repository.
