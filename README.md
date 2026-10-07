# Homelab: Windows Active Directory Build

Documented build of a two-machine Active Directory lab in VirtualBox: one Windows Server domain controller and one Windows 10/11 client joined to the domain. This is the lab I used to learn the identity and access management skills behind most tier-1 helpdesk tickets — password resets, account unlocks, group membership, and Group Policy.

## Skills demonstrated

- Windows Server administration (AD DS, DNS, DHCP roles)
- Domain controller promotion and FSMO basics
- OU design, user/group provisioning, Group Policy Objects (GPOs)
- VirtualBox networking (NAT vs. host-only vs. internal)
- DNS/DHCP troubleshooting in a domain environment

## What I learned

- Why DNS is the backbone of AD — if the client can't resolve the DC, nothing works.
- How Group Policy actually applies (LSDOU order) and how to verify with `gpresult`.
- The difference between a workgroup and a domain, felt hands-on instead of memorized.

---

## Lab topology

```
                    ┌──────────────────────────────┐
                    │  Host-only network           │
                    │  192.168.56.0/24            │
                    │                              │
   ┌────────────────┴──────────┐    ┌──────────────┴─────────────┐
   │ DC01 (Windows Server)     │    │ CLIENT01 (Windows 10/11)   │
   │ 192.168.56.10             │    │ 192.168.56.20 (DHCP)       │
   │ AD DS + DNS + DHCP        │    │ Domain: lab.local          │
   │ Domain: lab.local         │    │                            │
   └───────────────────────────┘    └────────────────────────────┘
```

Both VMs use a **host-only adapter** so lab traffic stays off my real LAN. The DC also keeps a NAT adapter for internet access during setup (Windows updates, downloading tools) — I disable it afterward to simulate an isolated corp network.

## Build steps

### 1. VirtualBox prep

1. Install VirtualBox and the Extension Pack.
2. Create the host-only network: **File → Host Network Manager** → create `vboxnet0` → set IPv4 to `192.168.56.1/24`, **disable its DHCP server** (the DC will handle DHCP).
3. Download ISOs: Windows Server 2022/2025 evaluation and Windows 10/11.

> ✅ **Checkpoint:** `vboxnet0` exists at 192.168.56.1 with DHCP off.

### 2. Build DC01 (Windows Server)

1. New VM: 4 GB RAM, 60 GB disk, 2 vCPUs. Attach the Server ISO.
2. Install with **Desktop Experience**.
3. Add two adapters: Adapter 1 = NAT, Adapter 2 = Host-only (`vboxnet0`).
4. Set a static IP on the host-only NIC:
   - IP: `192.168.56.10`, mask `255.255.255.0`, gateway: *(none — internal net)*
   - DNS: `127.0.0.1` (it will be its own DNS server)
   - Rename the NICs to `NAT` and `LAB` so you always know which is which:
     ```powershell
     Rename-NetAdapter -Name "Ethernet" -NewName "LAB"
     New-NetIPAddress -InterfaceAlias "LAB" -IPAddress 192.168.56.10 -PrefixLength 24
     Set-DnsClientServerAddress -InterfaceAlias "LAB" -ServerAddresses 127.0.0.1
     ```
5. Rename the computer to `DC01` and reboot:
   ```powershell
   Rename-Computer -NewName "DC01" -Restart
   ```

> ✅ **Checkpoint:** `ipconfig` shows 192.168.56.10 on the LAB adapter.

### 3. Install AD DS + DNS and promote

```powershell
# Install the roles (DNS comes along with AD DS)
Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools

# Promote to a domain controller, creating the new forest lab.local
Install-ADDSForest `
    -DomainName "lab.local" `
    -DomainNetbiosName "LAB" `
    -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password") `
    -InstallDns -Force
```

The server reboots. Log back in as `LAB\Administrator`.

> ✅ **Checkpoint:** `Get-ADDomain` returns `lab.local`; `nslookup lab.local` resolves to 192.168.56.10.

**What went wrong the first time:** I left the NAT adapter as the primary DNS and the promotion wizard complained it couldn't find itself. Lesson: the DC must point at *itself* for DNS before promoting.

### 4. Configure DHCP on the DC

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools

# Authorize in AD, then create the scope
Add-DhcpServerInDC -DnsName "dc01.lab.local" -IPAddress 192.168.56.10
Add-DhcpServerv4Scope -Name "LAB-LAN" -StartRange 192.168.56.20 -EndRange 192.168.56.100 `
    -SubnetMask 255.255.255.0 -State Active
Set-DhcpServerv4OptionValue -ScopeId 192.168.56.0 -DnsServer 192.168.56.10 `
    -Router 192.168.56.10 -DnsDomain "lab.local"
```

> ✅ **Checkpoint:** In DHCP Manager the scope is active with a green arrow.

### 5. Build CLIENT01 and join the domain

1. New VM: 4 GB RAM, 50 GB disk. Install Windows 10/11.
2. One adapter: Host-only (`vboxnet0`). Rename computer to `CLIENT01`.
3. Confirm it gets DHCP: `ipconfig` should show `192.168.56.20` with DNS `192.168.56.10`.
4. Join the domain:
   ```powershell
   Add-Computer -DomainName "lab.local" -Credential LAB\Administrator -Restart
   ```
5. Log in as a domain user to confirm.

> ✅ **Checkpoint:** `systeminfo | findstr Domain` shows `lab.local`; you can log in with a domain account.

**Gotcha:** the join fails if the client's DNS isn't 192.168.56.10 — the client must resolve `lab.local` through the DC. This is the #1 real-world domain-join failure and now I know exactly what to check first.

### 6. OU design + users/groups

I built an OU structure the way a small company would:

```
lab.local
├── IT
│   ├── Admins
│   └── Helpdesk
├── HR
└── Workstations
```

```powershell
# OUs
New-ADOrganizationalUnit -Name "IT" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Helpdesk" -Path "OU=IT,DC=lab,DC=local"

# Groups
New-ADGroup -Name "Helpdesk-Techs" -GroupScope Global -GroupCategory Security `
    -Path "OU=Helpdesk,OU=IT,DC=lab,DC=local"

# A test user (same pattern as my New-HDUser.ps1 toolkit script)
New-ADUser -Name "Test Tech" -SamAccountName "ttech" `
    -UserPrincipalName "ttech@lab.local" `
    -Path "OU=Helpdesk,OU=IT,DC=lab,DC=local" `
    -AccountPassword (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force) `
    -Enabled $true -ChangePasswordAtLogon $true
Add-ADGroupMember -Identity "Helpdesk-Techs" -Members "ttech"
```

> ✅ **Checkpoint:** `ttech` can log into CLIENT01 and is forced to change the password.

### 7. Starter Group Policies

**GPO 1 — Mapped drive for IT:**
`gpmc.msc` → right-click `OU=IT` → *Create a GPO and Link it here* → name `IT - Mapped Drive` → Edit → User Configuration → Preferences → Windows Settings → Drive Maps → New Mapped Drive → `S:` → `\\dc01\shared`. Run `gpupdate /force` on the client and the drive appears.

**GPO 2 — Lock the screen after 10 idle minutes (security baseline):**
Same OU → Computer Configuration → Policies → Administrative Templates → Control Panel → Personalization → *Enable screen saver*, *Screen saver timeout = 600*, *Password protect the screen saver = Enabled*.

**GPO 3 — Block Control Panel for non-admins (least privilege demo):**
User Configuration → Policies → Administrative Templates → Control Panel → *Prohibit access to Control Panel and PC settings = Enabled*. Log in as `ttech` — Control Panel is blocked; log in as domain admin — it's fine. That contrast is the whole point of the demo.

Verify with:
```powershell
gpresult /r        # which GPOs applied
gpresult /h report.html  # pretty version
```

> ✅ **Checkpoint:** `gpresult /r` on CLIENT01 lists all three GPOs as applied.

## Troubleshooting log (things that actually broke)

| Symptom | Cause | Fix |
|---|---|---|
| DC promotion wizard error about DNS | DC pointed at router for DNS | Set DNS to 127.0.0.1 *before* promoting |
| Client couldn't join domain | Client got DNS from host-only adapter default | Set DHCP option 006 to 192.168.56.10 |
| GPO didn't apply | Linked at domain level, blocked inheritance on OU | Moved link to the OU, ran `gpupdate /force` |
| Couldn't ping between VMs | Windows Firewall on the DC | Enabled File and Printer Sharing rule group for the lab profile |

## How to reproduce

1. Install VirtualBox + Extension Pack on any 16 GB RAM host.
2. Download the Windows Server and Windows 10/11 evaluation ISOs (Microsoft Evaluation Center).
3. Follow steps 1–7 above; each ✅ checkpoint tells you you're good to continue.
4. Tear down by deleting the VMs and the `vboxnet0` network — nothing touches your real network.
