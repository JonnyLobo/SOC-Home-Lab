# Lab setup

How I built the lab, with the commands that actually worked. Every step ends with a check, because a step that "finished" is not the same as a step that worked.

## 1. Network

- VMware **host-only** network `VMnet1`: `192.168.73.0/24`, no internet access.
- The Wazuh VM has a second adapter (NAT) for updates only. It stays **disconnected** ("Connect at power on" unchecked).

**Check:** from the endpoint, confirm the route and the ports the agent needs.

```powershell
Find-NetRoute -RemoteIPAddress 192.168.73.128
Test-NetConnection 192.168.73.128 -Port 1514   # agent events
Test-NetConnection 192.168.73.128 -Port 1515   # agent enrollment
Test-NetConnection 192.168.73.128 -Port 443    # dashboard
```

> Troubleshooting: a VPN client on the host blocked all LAN traffic to the lab (LAN access is a paid feature in that VPN). Disconnect the VPN while using the dashboard.

## 2. Wazuh server

- Official **Wazuh 4.14.5 OVA** (Amazon Linux 2023): manager, indexer and dashboard in one VM.
- Resources: 8 GB RAM, 4 vCPU.
- Address on the host-only network: `192.168.73.128`, reserved for the Wazuh MAC address in VMware's DHCP (`C:\ProgramData\VMware\vmnetdhcp.conf`), so it never changes.

**Check:** the dashboard opens at `https://192.168.73.128` from the host.

## 3. Windows endpoint (WIN-LAB1)

- Windows 10 22H2 x64, installed from a clean official ISO.
- Resources: 4 GB RAM, 2 vCPU. Address: `192.168.73.133`.
- Snapshot taken **with the VM powered off** after each milestone.

## 4. Wazuh agent

Download the official installer from `packages.wazuh.com` and verify its signature before running it:

```powershell
Get-AuthenticodeSignature "C:\Lab\wazuh-agent-4.14.5-1.msi"
```

Expected: `Status: Valid`, signed by Wazuh, Inc.

Install it. `msiexec` runs **asynchronously**, so without `-Wait` the command returns before the install ends and errors go unnoticed. That is exactly what happened on my first attempt.

```powershell
$p = Start-Process msiexec.exe -Wait -PassThru -ArgumentList '/i "C:\Lab\wazuh-agent-4.14.5-1.msi" /q WAZUH_MANAGER="192.168.73.128" WAZUH_AGENT_NAME="WIN-LAB1" /l*v "C:\Lab\wazuh-install.log"'
$p.ExitCode
```

Expected: `0`. Any other code means the install failed; read `C:\Lab\wazuh-install.log`.

Start the service (its name is `WazuhSvc`):

```powershell
Start-Service WazuhSvc
Get-Service WazuhSvc
```

**Check:** the agent appears as **active** in the dashboard.

## 5. Sysmon

Sysmon (Microsoft Sysinternals) with the community configuration from [SwiftOnSecurity/sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config):

```powershell
C:\Lab\Sysmon\Sysmon64.exe -accepteula -i C:\Lab\sysmonconfig-export.xml
```

**Check:** Sysmon is writing events locally.

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
```

## 6. Send Sysmon events to Wazuh

Add this block to `C:\Program Files (x86)\ossec-agent\ossec.conf`, just before `</ossec_config>`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Restart the agent so it reads the new configuration:

```powershell
Restart-Service WazuhSvc
```

**Check:** Sysmon alerts (rule groups `sysmon`) appear in the dashboard for WIN-LAB1.

## Known limitation

The `wazuh-alerts-*` index only stores events that match a rule. Sysmon events that do not trigger an alert (for example, many process creations, Event ID 1) are **not** searchable in the dashboard. In [TRIAGE-002](../triage-notes/2026-10-15-WIN-LAB1-rule-92213.md) I had to query the local Sysmon log on the endpoint to find the parent process:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=1} |
  Where-Object { $_.Message -match '<ProcessGuid>' } |
  Format-List TimeCreated, Message
```

Planned improvement: enable Wazuh **archives** to keep all events, not only alerts.
