# SIEM_Wazuh_based_Laboratory
Implementation of an Open-Source Security Information and Event Management (SIEM) System for the Identification of High-Risk Vulnerabilities
# SIEM Implementation Lab Guide
## Identifying High-Risk Vulnerabilities with Wazuh (Open-Source SIEM)

This guide walks you through building a small home-lab SIEM using your existing VMware setup, and using it to detect, prioritize, and alert on high-risk vulnerabilities.

---

## 0. Architecture Overview

| VM | Role |
|---|---|
| **Ubuntu 64-bit** | SIEM server — runs Wazuh Manager, Indexer, and Dashboard (all-in-one) |
| **Windows 11** | Monitored endpoint (Wazuh agent) — will host a deliberately outdated app to trigger detections |
| **Windows Server 2019 or 2022** | Second monitored endpoint (Wazuh agent) — represents a "server" asset |
| **Parrot Security** | Scanner/attacker box — used to run active scans (Nmap/OpenVAS) and generate test traffic/log events |
| Android | Optional — skip unless your assignment specifically wants a mobile asset |

**Why Wazuh?** It's a single free/open-source platform that does three things your task needs at once: it's a SIEM (log collection, correlation, alerting, dashboards), it has a built-in **Vulnerability Detector** module (matches installed software against CVE feeds — NVD, vendor advisories, etc.), and it supports custom rules and multi-channel alerting. Using one integrated tool instead of stitching together a separate SIEM + scanner will save you a lot of pain in a lab environment.

All commands below assume you're comfortable copy-pasting into a terminal — I'll explain what each one does.

---

## Part 1 — Tool Selection & Setup

### 1.1 Prepare the Ubuntu VM (SIEM server)

Power on your **Ubuntu 64-bit** VM. Give it at least 4GB RAM / 2 vCPUs in VMware settings (Wazuh's indexer is Java-based and needs headroom) — shut the VM down first, then edit via VM → Settings → Memory/Processors.

Open a terminal and update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 Install Wazuh (all-in-one)

Wazuh publishes a single installer script that sets up the Manager, Indexer, and Dashboard together — ideal for a lab.

```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

- `-a` means "all-in-one" install.
- This takes 10–20 minutes. **When it finishes, it prints an admin username (usually `admin`) and a generated password — copy this somewhere safe, you'll need it to log in.**

### 1.3 Access the dashboard

Find the Ubuntu VM's IP address:

```bash
ip a
```

Look for something like `192.168.x.x` under your main network adapter (not `lo`). From any browser (on the host machine, or another VM on the same network), go to:

```
https://<ubuntu-vm-ip>
```

You'll get a certificate warning (self-signed cert, safe to ignore in a lab) — accept it, then log in with the admin credentials from step 1.2. You should see the Wazuh Dashboard home screen.

### 1.4 Install agents on Windows 11 and Windows Server

On the **Ubuntu SIEM VM**, note its IP again (`ip a`) — you'll need it below.

On each **Windows** VM (Windows 11, and Windows Server 2019/2022), open **PowerShell as Administrator** and run (replace `<SIEM-IP>` with your Ubuntu VM's IP):

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.0-1.msi -OutFile wazuh-agent.msi
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="<SIEM-IP>"
NET START WazuhSvc
```

Repeat on both Windows machines, each pointing at the same Ubuntu SIEM IP.

**Verify agents connected:** back on the Wazuh Dashboard → left menu → **Agents**. You should see both Windows machines listed with status **Active**. If a machine shows "Disconnected," double check that both VMs are on the same VMware network (NAT or Host-only both work, as long as they can ping each other) and that Windows Firewall isn't blocking outbound traffic on port 1514/1515.

---

## Part 2 — Data Collection & Integration

This is about making sure the SIEM is pulling in enough data — internal (your own logs/scans) and external (public vulnerability databases) — to actually spot problems.

### 2.1 Turn on the Vulnerability Detector (external CVE data)

This module downloads and refreshes feeds from public vulnerability databases (NVD, and OS-vendor feeds) and cross-checks them against every package/app installed on your agents.

On the **Ubuntu SIEM VM**, edit the manager config:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Find the `<vulnerability-detector>` block (Wazuh 4.8+ ships this enabled by default, but confirm it looks like this):

```xml
<vulnerability-detector>
  <enabled>yes</enabled>
  <index-status>yes</index-status>
  <feed-update-interval>60m</feed-update-interval>
</vulnerability-detector>
```

Save (Ctrl+O, Enter, Ctrl+X) and restart the manager:

```bash
sudo systemctl restart wazuh-manager
```

Give it 10–15 minutes on first run to download the CVE feed database.

### 2.2 Confirm internal log sources are flowing

Wazuh agents automatically forward:
- **Windows Event Logs** (Security, System, Application) — on by default once the agent is installed.
- **File integrity monitoring (FIM)** and process/software inventory — used by the Vulnerability Detector to know what's installed.

You can sanity-check this: Dashboard → **Agents** → click a Windows agent → **Inventory data**. You should see a list of installed software with versions. That inventory is exactly what gets matched against the CVE feed.

### 2.3 Add an active scanner (Parrot Security)

Power on **Parrot Security**. This VM plays the "external scanning" role — simulating how an internal security team would actively probe assets rather than just waiting on installed-software inventory.

Run a basic service/version scan against your Windows targets (replace with their actual IPs):

```bash
sudo nmap -sV -O <windows11-ip> <windows-server-ip>
```

This isn't fed automatically into Wazuh, but it gives you a second, independent data source to cross-reference — useful for your report ("internal Vulnerability Detector findings corroborated by external Nmap scan results"). If you want deeper CVE-mapped scanning from Parrot, you can install **OpenVAS/Greenbone Community Edition** there too:

```bash
sudo apt install gvm -y
sudo gvm-setup
```

(This is a heavier install — optional if Nmap output is enough for your assignment. GVM's PDF/CSV reports can simply be described in your write-up as an external validation source.)

---

## Part 3 — Prioritization

Wazuh's Vulnerability Detector tags every finding with a **severity** pulled straight from the CVE's CVSS score: `Critical`, `High`, `Medium`, `Low`. That's your ready-made prioritization axis — no custom scoring needed, though you should explain the logic in your write-up:

- **Critical/High** = remotely exploitable, no auth required, high impact (data loss, full compromise) → immediate action
- **Medium** = exploitable but with mitigating factors (local access needed, partial impact)
- **Low** = minimal impact or very hard to exploit

To view prioritized results: Dashboard → **Threat Intelligence → Vulnerabilities**. Filter by `Severity: Critical` or `High` — this is your "high-risk" list.

---

## Part 4 — Design Custom Rules

Default rules already fire on vulnerability-detector events, but your task wants **tailored** rules — so let's add one that specifically flags Critical/High findings and tags them clearly.

On the Ubuntu SIEM VM:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Add (inside a `<group>` block — create the file content below if empty):

```xml
<group name="local,vulnerability-detector,">

  <rule id="100010" level="12">
    <if_group>vulnerability-detector</if_group>
    <field name="vulnerability.severity">Critical</field>
    <description>High-risk vulnerability detected (CRITICAL severity) on $(agent.name)</description>
    <options>no_full_log</options>
    <group>high_risk_vuln,</group>
  </rule>

  <rule id="100011" level="10">
    <if_group>vulnerability-detector</if_group>
    <field name="vulnerability.severity">High</field>
    <description>High-risk vulnerability detected (HIGH severity) on $(agent.name)</description>
    <options>no_full_log</options>
    <group>high_risk_vuln,</group>
  </rule>

</group>
```

What this does: any vulnerability event tagged `Critical` gets rule level 12 (very high), `High` gets level 10, and both get grouped under `high_risk_vuln` — which makes the next step (alerting) simple to target.

Save, then restart:

```bash
sudo systemctl restart wazuh-manager
```

---

## Part 5 — Configure Alerts

Now make the SIEM actually *notify someone* when a high-risk vulnerability shows up — the brief calls this out as "relevant stakeholders," so email is the simplest to demonstrate.
visit the wazuh documentation page for setting up email smtp/ftp
https://documentation.wazuh.com/current/user-manual/manager/alert-management.html#smtp-server-with-authentication

Edit the config again:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add (or edit) an email block near the top:

```xml
<global>
  <email_notification>yes</email_notification>
  <smtp_server>smtp.gmail.com</smtp_server>
  <email_from>your-lab-alert@gmail.com</email_from>
  <email_to>your-real-email@example.com</email_to>
  <email_maxperhour>12</email_maxperhour>
</global>
```

(For Gmail SMTP you'll need an "app password," not your normal password — Google account settings → Security → App Passwords. For a lab, it's fine to use a throwaway account, or you can just screenshot the Dashboard alert instead of configuring real SMTP if your assignment doesn't require live email.)

Then tell Wazuh which severity level triggers an email — inside the same `<global>` block:

```xml
<email_alert_level>10</email_alert_level>
```

This means any alert at level 10+ (i.e., your custom High/Critical vulnerability rules from Part 4) sends an email automatically.

Restart once more:

```bash
sudo systemctl restart wazuh-manager
```

*Optional stretch goal:* Wazuh also supports Slack/Teams webhook integrations if you'd rather demo a chat alert than email — worth mentioning as an alternative "stakeholder notification channel" in your report even if you only implement email.

---

## Part 6 — Validate Detection Capability

This is where you prove the whole pipeline actually works, using a known-vulnerable component.

### 6.1 Install a deliberately outdated app on Windows 11

Pick something with well-documented CVEs and an old installer still easily downloadable, e.g.:
- **7-Zip 18.05** (has known CVEs, e.g. CVE-2018-10115)
- **VLC 3.0.0** (older CVEs exist)
- **PuTTY 0.70**

Download and install the old version on the **Windows 11** VM as you normally would.

### 6.2 Trigger a fresh scan

Wazuh's inventory sync runs periodically, but you can force it. On the Windows agent, restart the service to push an immediate inventory update:

```powershell
Restart-Service WazuhSvc
```

Give it 5–10 minutes (inventory collection + vulnerability correlation isn't instant).

### 6.3 Confirm detection

- Dashboard → **Threat Intelligence → Vulnerabilities** → filter by that agent. You should see the CVE for your old app listed, with its severity.
- Dashboard → **Security Events** (or **Alerts**) → search for `rule.id: 100010` or `100011` (your custom rules) → you should see the alert with description "High-risk vulnerability detected…"
- Check your inbox (if you configured email) for the alert notification.

### 6.4 Visual reporting

Wazuh Dashboard has a built-in **Vulnerabilities** module with pie charts (by severity) and bar charts (by agent) out of the box — screenshot these for your report. You can also build a custom dashboard: Dashboard → **Dashboards Management → Create Dashboard**, add visualizations filtered on `rule.groups: high_risk_vuln`, and export as PDF (top-right menu → **Reporting**) for a clean deliverable.

---

## Wrap-Up Checklist for Your Report

- [ ] Screenshot: Wazuh Dashboard home + Agents page showing 2+ connected endpoints
- [ ] Screenshot: Vulnerability Detector config (`ossec.conf` snippet)
- [ ] Screenshot: Vulnerabilities list filtered by Critical/High severity
- [ ] Copy of your `local_rules.xml` custom rule
- [ ] Screenshot: alert triggered (dashboard + email if configured)
- [ ] Screenshot: known-vulnerable app (e.g., old 7-Zip) detected with its CVE
- [ ] Screenshot/export: severity dashboard/pie chart or PDF report
- [ ] Optional: Nmap/OpenVAS output from Parrot Security as external corroboration

## Troubleshooting Notes

- **Agent shows "Never Connected":** check VMware network mode — all VMs should be on the same virtual network (Host-only or NAT) and able to ping the Ubuntu VM's IP.
- **Vulnerability Detector shows no results:** the CVE feed can take up to 30 min to fully download on first run; check progress with `sudo tail -f /var/ossec/logs/ossec.log | grep vulnerability`.
- **Dashboard won't load:** confirm all three Wazuh services are running: `sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard`.
- **Low VM resources:** if Ubuntu feels sluggish, the indexer (OpenSearch-based) is the heaviest component — bump RAM to 6–8GB if your host machine can spare it.
