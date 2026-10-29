# Splunk SOC Home Lab

A home lab built to practice the daily work of a SOC analyst: stand up a Windows domain, ship its logs to a SIEM, run a brute-force attack against it, and write a detection and alert for it.

**Status:** All four parts are complete: Active Directory domain, Splunk server, log forwarding, and a brute-force attack with a working detection and scheduled alert.

| Part | What | Status |
|------|------|--------|
| 1 | Windows Server 2022 domain controller + Windows 11 domain client on an isolated network | Done |
| 2 | Splunk Enterprise on Ubuntu Server 24.04 | Done |
| 3 | Audit policy + Splunk Universal Forwarders on both Windows machines | Done |
| 4 | Brute-force attack, detection search (Event IDs 4625 / 4740), scheduled alert | Done |

## Lab design

Everything runs in VirtualBox on a single host, on an isolated NAT Network so the lab machines can talk to each other and reach the internet, but nothing outside can reach them.

```
                 NAT Network "SOCLab"  192.168.10.0/24  (VirtualBox DHCP off)

   +--------------------+      +--------------------+      +--------------------+
   | DC01               |      | CLIENT01           |      | SPLUNK             |
   | Windows Server 2022|      | Windows 11         |      | Ubuntu Server 24.04|
   | AD DS, DNS, DHCP   |      | domain-joined      |      | Splunk Enterprise  |
   | 192.168.10.7       |      | 192.168.10.100     |      | 192.168.10.10      |
   +--------------------+      +--------------------+      +--------------------+
```

| Machine | OS | Resources | Address | Role |
|---------|----|-----------|---------|------|
| DC01 | Windows Server 2022 Standard Evaluation (Desktop Experience) | 4 GB RAM, 2 CPU, 50 GB | 192.168.10.7 (static) | Domain controller, DNS, DHCP |
| CLIENT01 | Windows 11 | 4 GB RAM, 2 CPU, 64 GB | DHCP from DC01 (192.168.10.100) | Domain workstation, attack source |
| SPLUNK | Ubuntu Server 24.04 | 4 GB RAM, 2 CPU, 50 GB | 192.168.10.10 (static) | SIEM |

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

## Part 2: the Splunk server

1. **SPLUNK VM.** Ubuntu Server 24.04, 4 GB RAM, 2 CPUs, 50 GB, attached to `SOCLab`. ([17](screenshots/17-splunk-vm-settings-soclab.png))
2. **Static IP set in the installer** (192.168.10.10/24, gateway 192.168.10.1, DNS 192.168.10.7 and 8.8.8.8), with OpenSSH server selected. Confirmed after install on the login banner. ([18](screenshots/18-splunk-ubuntu-login-static-ip.png))
3. **Confirmed the network** settings of `SOCLab` (192.168.10.0/24, DHCP off). ([19](screenshots/19-nat-networks-soclab-final.png))
4. **Port forwarding on SOCLab** so the host PC can reach the VM: host 2222 to guest 22 (SSH) and host 8000 to guest 8000 (Splunk Web), both bound to 127.0.0.1 so only the host can use them. ([20](screenshots/20-soclab-port-forwarding-rules.png))
5. **Installed Splunk Enterprise 10.6.1.1** from the .deb using `wget` and `dpkg -i`, started it as the `splunk` user, and created the admin account. ([21](screenshots/21-splunk-download-and-install.png))
6. **Reached Splunk Web from the host browser** at `http://localhost:8000`. ([22](screenshots/22-splunk-web-home.png))
7. **Enabled boot-start** (systemd) so Splunk survives VM reboots. ([23](screenshots/23-splunk-boot-start-error-and-fix.png))
8. **Receiving on port 9997** for the forwarders. ([24](screenshots/24-splunk-receiving-9997.png))
9. **Created the `endpoint` index** for the Windows logs. ([25](screenshots/25-splunk-endpoint-index.png))

## Part 3: audit policy and log forwarding

1. **Audit policy (Default Domain Policy).** Turned on Success and Failure auditing for Audit Logon, Audit Credential Validation and Audit User Account Management, so Windows writes the failed-logon (4625) and lockout (4740) events the detection will depend on. ([26](screenshots/26-gpo-audit-logon.png), [27](screenshots/27-gpo-audit-credential-validation.png), [28](screenshots/28-gpo-audit-user-account-management.png))
2. **Account lockout threshold of 5**, so the Part 4 attack locks the target account. ([29](screenshots/29-gpo-account-lockout-threshold.png))
3. **Splunk Universal Forwarder on DC01**, pointed at 192.168.10.10:9997, with an `inputs.conf` that collects the Windows Security and System logs into the `endpoint` index. ([30](screenshots/30-dc01-universal-forwarder-installed.png), [31](screenshots/31-dc01-inputs-conf-save-as.png))
4. **Same on CLIENT01**, installed as the local administrator, then `gpupdate /force` so the audit policy applies there too. ([35](screenshots/35-client01-universal-forwarder-installed.png), [36](screenshots/36-client01-forwarder-restart-gpupdate.png))
5. **Verified in Splunk** with `index=endpoint | stats count by host, source`: Security and System events from both DC01 and CLIENT01. ([34](screenshots/34-splunk-dc01-logs-arriving.png), [37](screenshots/37-splunk-both-hosts-logging.png))

## Part 4: attack, detection, alert

1. **Attack.** From CLIENT01, a PowerShell loop made 10 failed logon attempts against `jsmith` with wrong passwords. ([38](screenshots/38-client01-brute-force-script-run.png))
2. **Found the failed logons (Event ID 4625)** in Splunk with `index=endpoint EventCode=4625`. ([39](screenshots/39-splunk-4625-failed-logons-raw.png))
3. **Found a mistake in my own lab.** Every 4625 said "Account currently disabled" and there was no lockout: `jsmith` had been created disabled, so the attack never tested the lockout policy. Enabled the account, reset the password, re-ran the attack. ([40](screenshots/40-splunk-4625-account-disabled.png), [41](screenshots/41-splunk-4625-bad-password-after-fix.png))
4. **Found the lockout (Event ID 4740)** on DC01 after the threshold of 5 was hit. ([42](screenshots/42-splunk-4740-jsmith-lockout.png))
5. **Detection search.** Failed logons grouped by host and target account, flagging 5 or more:
   ```
   index=endpoint EventCode=4625
   | eval target=mvindex(Account_Name, 1)
   | stats count AS failures BY host, target
   | where failures >= 5
   ```
   `Account_Name` is multivalue in 4625 events: index 0 is the account that made the request, index 1 is the account being logged on to, which is why the search uses `mvindex(..., 1)`. Result: CLIENT01 / jsmith / 15. ([43](screenshots/43-splunk-detection-search-jsmith-15.png))
6. **Scheduled alert** "Brute force: 5+ failed logons": cron `*/5 * * * *`, last 15 minutes, trigger when results are greater than 0, action Add to Triggered Alerts. Alerting needs the Splunk trial license, since the Free license disables it. ([44](screenshots/44-splunk-save-as-alert-settings.png))
7. **Verified it fired.** The scheduler log shows the run at 16:35:01 with result_count 1, and the alert appears under Activity > Triggered Alerts. ([45](screenshots/45-splunk-scheduler-alert-ran.png), [46](screenshots/46-splunk-fired-alerts-brute-force.png), [47](screenshots/47-splunk-triggered-alerts-page.png))

## What broke and how I fixed it

- **The SOCLab network showed the default 10.0.2.0/24 prefix and DHCP on.** The edit had not been applied. Retyped the prefix, unchecked DHCP, clicked Apply, then re-selected the row to confirm it stuck. ([screenshot 1](screenshots/01-nat-networks-before-soclab.png))
- **DC01's adapter was set to plain NAT, so the network name field was greyed out.** Changing "Attached to" from NAT to NAT Network unlocks it. ([3](screenshots/03-dc01-adapter-plain-nat-wrong.png))
- **The Windows Server installer restarted from the beginning after its first reboot.** I pressed a key at "Press any key to boot from CD". Fix: powered off, moved Hard Disk to the top of the boot order, restarted. The long black screen on first boot is normal.
- **The DHCP post-install wizard appeared before the server was promoted.** Cancelled it, promoted to a domain controller first, then completed DHCP configuration.
- **A typo in the DNS option (`192.68.10.7`).** Caught it when checking the scope options and corrected it to `192.168.10.7`. DHCP clients would otherwise have received a dead DNS server.
- **Active Directory rejected a password.** The default complexity policy refuses passwords that contain three or more consecutive characters of the account's own name. Used a password that doesn't.
- **CLIENT01 stayed on a black screen at first boot.** Boot order was greyed out because the VM uses UEFI (normal), and the ISO and settings were correct. Rebuilding the VM did not help. Fix: Settings > Display > Graphics Controller from VBoxSVGA to VMSVGA.
- **`splunk stop` did nothing and `enable boot-start` refused to run.** I had started Splunk as the `splunk` user but ran stop as root, and this Splunk version refuses to manage the process as root, so boot-start still saw it running and `Splunkd.service` did not exist. Fix: stop it as the `splunk` user (`sudo -u splunk /opt/splunk/bin/splunk stop`), then run `enable boot-start -user splunk -systemd-managed 1`, then `sudo systemctl start Splunkd`. ([23](screenshots/23-splunk-boot-start-error-and-fix.png))
- **The DC01 forwarder showed the Splunk server as "configured but inactive".** `Test-NetConnection 192.168.10.10 -Port 9997` failed with DestinationHostUnreachable. Cause: the SPLUNK VM was powered off. Fix: start SPLUNK first and give Splunk a couple of minutes to come up (boot-start handles it); the forwarder then connected and the logs flowed. Lesson: the SIEM has to be up before the machines that send to it. ([32](screenshots/32-dc01-forward-server-inactive.png), [33](screenshots/33-dc01-test-netconnection-failed.png))
- **The attack produced 4625 events but no lockout and every event said "Account currently disabled".** `jsmith` had been created in a disabled state. Enabled it and reset the password in Active Directory Users and Computers; the re-run produced "Unknown user name or bad password" and a 4740 lockout. ([39](screenshots/39-splunk-4625-failed-logons-raw.png), [41](screenshots/41-splunk-4625-bad-password-after-fix.png))
- **Windows 11 setup demanded a network and an account.** Used the local-account route (Shift+F10 and `start ms-cxh:localonly`) to create a local account, then joined the domain afterwards.
- **The first domain join failed with "user name or password is incorrect".** Reset the AD account passwords in Active Directory Users and Computers and retried; the join succeeded.

## What's next

- Add detections for other common events (new account created 4720, logon from an unusual host) and write them up as Sigma rules in a separate repository
- Work through the Splunk Boss of the SOC (BOTS) dataset and TryHackMe SOC Level 1

## Notes

Evaluation ISOs and throwaway credentials are used throughout. No credentials are stored in this repository.
