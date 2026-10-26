# Windows LAN Shared Folder (Read-Only)

A reusable guide for creating a Windows shared folder that other devices on the same trusted local network can access to read and copy files.

> **Security notes**
> - Do not expose SMB (TCP port 445) directly to the internet.
> - Use this on a trusted local network only.
> - Use a strong, unique password for the dedicated sharing account.
> - Replace all example values such as `HOSTNAME`, `HOST_IP`, and `Wi-Fi` with values from your own computer.
> - Commands below assume Windows 10/11 and PowerShell. Run administrator commands in **PowerShell as Administrator**.

## 1. Create the folder and a dedicated account

On the computer that will host the files, open PowerShell as Administrator.

Create the folder:

```powershell
New-Item -Path "C:\LabShare" -ItemType Directory -Force
```

Create a dedicated local account. Enter a strong password when prompted:

```powershell
$Password = Read-Host "Enter a strong password for labreader" -AsSecureString

New-LocalUser -Name "labreader" `
  -Password $Password `
  -Description "Read-only LAN file sharing account"
```

If the `labreader` account already exists, do not create it again. Make sure you know its password and review its permissions.

## 2. Set NTFS permissions to read-only for the sharing account

The following commands remove inherited permissions from this folder and grant access to SYSTEM, local Administrators, and the dedicated account. Review the impact before using them on a folder that already contains important files.

```powershell
icacls "C:\LabShare" /inheritance:r

icacls "C:\LabShare" /grant:r `
  "SYSTEM:(OI)(CI)F" `
  "BUILTIN\Administrators:(OI)(CI)F" `
  "$env:COMPUTERNAME\labreader:(OI)(CI)RX"
```

- `F` means Full Control.
- `RX` means Read and Execute.
- `(OI)(CI)` applies permissions to files and subfolders.

If the folder already contains files, verify their effective permissions as well.

## 3. Share the folder with read-only share permissions

```powershell
New-SmbShare `
  -Name "LabShare" `
  -Path "C:\LabShare" `
  -ReadAccess "$env:COMPUTERNAME\labreader" `
  -CachingMode None
```

Check the share permissions:

```powershell
Get-SmbShareAccess -Name "LabShare"
```

The `labreader` account should have **Read** access, not Change or Full.

If the share already exists, inspect it rather than creating it again. You can review shares with:

```powershell
Get-SmbShare
```

## 4. Check the network profile and firewall

View network profiles:

```powershell
Get-NetConnectionProfile
```

Only mark a network as `Private` if it is a trusted network you control. Replace `Wi-Fi` with the correct interface name shown on your computer:

```powershell
Set-NetConnectionProfile `
  -InterfaceAlias "Wi-Fi" `
  -NetworkCategory Private
```

Enable the built-in SMB inbound firewall rule for the Private profile and restrict remote sources to the local subnet:

```powershell
Set-NetFirewallRule `
  -Name "FPS-SMB-In-TCP" `
  -Enabled True `
  -Profile Private `
  -RemoteAddress LocalSubnet
```

If the rule name is not available on your Windows version, inspect the available rules:

```powershell
Get-NetFirewallRule -DisplayGroup "File and Printer Sharing" |
  Select-Object Name, DisplayName, Enabled, Profile
```

Do **not** disable Windows Firewall. Do **not** create router port-forwarding rules for TCP 445. Keep SMB accessible only on trusted local networks.

## 5. Find the host computer name and IP address

On the host computer:

```powershell
hostname
ipconfig
```

Record:
- The computer name (used below as `HOSTNAME`).
- The IPv4 address of the network interface shared with the other device (used below as `HOST_IP`).

Do not use an address from a VPN adapter, VMware/Hyper-V virtual adapter, or disconnected adapter. Usually, both computers should be connected to the same local network and have addresses in the same subnet.

## 6. Connect from the other Windows computer

On the client computer, first confirm that it is connected to the same trusted Wi-Fi or LAN.

Test connectivity (replace `HOST_IP` with the host's actual local IPv4 address):

```powershell
ping HOST_IP
Test-NetConnection HOST_IP -Port 445
```

A successful port test should show:

```text
TcpTestSucceeded : True
```

Note: a failed `ping` alone does not prove SMB is unavailable; ICMP may be blocked. The TCP 445 test is more directly relevant.

Open the share using **Win + R**, replacing `HOST_IP`:

```text
\\HOST_IP\LabShare
```

If prompted for credentials, use:

```text
HOSTNAME\labreader
```

Enter the password configured for `labreader` on the host computer. Do not put passwords in this guide, scripts, or GitHub.

If Windows keeps using old credentials, open **Credential Manager → Windows Credentials** on the client and remove only the stale entry for this host, then try again.

## 7. Optional: map the share as a drive

On the client computer, run this in PowerShell or Command Prompt. Replace `HOST_IP` and `HOSTNAME` with the correct values:

```powershell
net use Z: \\HOST_IP\LabShare /user:HOSTNAME\labreader * /persistent:yes
```

The `*` makes Windows prompt for the password instead of putting it in the command. The share should then appear as drive `Z:` in File Explorer.

To disconnect it later:

```powershell
net use Z: /delete
```

## 8. Add or update files on the host

On the host computer, copy files into:

```text
C:\LabShare
```

For example, to copy a file from the current user's Downloads folder:

```powershell
Copy-Item "$env:USERPROFILE\Downloads\sample.pcap" "C:\LabShare\"
```

Replace `sample.pcap` with the actual filename. You can also use File Explorer and normal Copy/Paste.

The host's authorized account can add or update files. The remote `labreader` account should only be able to read and copy files, not upload, edit, or delete them.

## 9. Verify read-only access

On the host, inspect share permissions and NTFS permissions:

```powershell
Get-SmbShareAccess -Name "LabShare"
icacls "C:\LabShare"
Get-SmbServerConfiguration |
  Select-Object EnableSMB1Protocol, EnableSMB2Protocol
```

Prefer SMB2/SMB3; do not enable SMB1 to fix connection issues.

From the client, confirm that files can be opened and copied to the client computer. Attempting to create, modify, or delete files in the share should fail for `labreader`.

## 10. Troubleshooting checklist

If the client cannot connect:

1. Verify both devices are on the same trusted LAN.
2. Verify the host IP address with `ipconfig`.
3. Run `Test-NetConnection HOST_IP -Port 445` from the client.
4. Confirm the SMB share exists: `Get-SmbShare`.
5. Confirm the account has Read access: `Get-SmbShareAccess -Name "LabShare"`.
6. Check Windows Firewall and whether a VPN or endpoint security tool blocks local network traffic.
7. Check for stale credentials in Credential Manager.
8. Do not solve the issue by disabling the firewall or exposing TCP 445 to the internet.

## Quick reference

| Item | Value |
|---|---|
| Host folder | `C:\LabShare` |
| Share name | `LabShare` |
| Share account | `labreader` |
| Share permissions | Read |
| Local network path | `\\HOST_IP\LabShare` |
| Optional mapped drive | `Z:` |
| SMB port | TCP `445` (trusted LAN only) |

## Security reminder

This setup is intended for a trusted local network. For access from outside the LAN, use a properly configured VPN or a secure file-sharing service instead of exposing SMB directly to the internet. Use only sanitized or authorized lab artifacts, and do not place passwords, secrets, confidential logs, or sensitive data in a public repository.
