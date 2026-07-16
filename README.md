# Home Lab: pfSense Router + Active Directory Domain

I built this lab in VirtualBox to get hands-on with the stuff that's hard to fake on a CV — setting up a real firewall, segmenting a network properly, and standing up an Active Directory domain from nothing. Everything below was built, broken, and fixed on my own laptop.

## What's in here

- A pfSense router/firewall, built from scratch
- A proper WAN/LAN split using NAT and an internal network
- A DNS outage I caused myself and had to actually diagnose
- A Windows Server 2022 box running in **Server Core** — no GUI, PowerShell only
- That server promoted to a Domain Controller, with users and OUs created by hand in PowerShell
- A Windows 10 machine joined to the domain, logging in as a real AD user

## Architecture

```
                    Internet
                       |
                   [ WAN - NAT ]
                       |
                  ┌─────────────┐
                  │   pfSense   │  192.168.1.1
                  │  (Router/   │
                  │  Firewall)  │
                  └─────────────┘
                       |
                  [ LAN - Internal Network ]
                       |
          ┌────────────┴────────────┐
          |                         |
   ┌─────────────┐          ┌─────────────┐
   │    DC01     │          │  Client01   │
   │ Windows     │          │ Windows 10  │
   │ Server 2022 │          │             │
   │ (Core)      │◄─domain──│             │
   │ 192.168.1.10│   join   │192.168.1.100│
   └─────────────┘          └─────────────┘
   Domain Controller          Domain Client
   homelab.arpa                (jsmith)
```

## How I built it

### 1. pfSense — the router/firewall
I installed pfSense Community Edition (2.8.1-RELEASE) as a VM and gave it two virtual network cards: one on NAT for internet (WAN), one on an Internal Network for the private side (LAN).

One small detail I found interesting: pfSense doesn't call interfaces "Ethernet" or "WiFi" like Windows does — it names them after the driver. Both my virtual NICs use Intel's emulated driver, so they showed up as `em0` and `em1`. I assigned `em0` to WAN and `em1` to LAN.

From there I gave LAN a static IP (`192.168.1.1/24`) and turned on its DHCP server so anything plugged into the LAN would get an address automatically. I also set the domain to `homelab.arpa` instead of the more common `.local`, since pfSense actually warns against `.local` — it clashes with mDNS/Bonjour discovery on some devices.

### 2. Windows 10 — the client
Nothing fancy here — just a VM sitting on the LAN network, picking up an IP from pfSense's DHCP server. I used this machine as my test bench for reaching the pfSense dashboard and, later, for joining the domain.

### 3. Windows Server 2022 — the Domain Controller
I deliberately installed this in **Server Core**, which means no desktop, no Start menu — just PowerShell. It's more work up front but it's genuinely how a lot of production servers are run, so I wanted the practice.

Set a static IP (`192.168.1.10`) through the `SConfig` menu, then installed the AD DS role:

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

Promoted the box to a Domain Controller for `homelab.arpa`:

```powershell
Install-ADDSForest -DomainName "homelab.arpa"
```

And once that finished rebooting, built out a small AD structure by hand — an Organizational Unit and a test user:

```powershell
New-ADOrganizationalUnit -Name "Employees" -Path "DC=homelab,DC=arpa"

New-ADUser -Name "John Smith" -GivenName "John" -Surname "Smith" `
  -SamAccountName "jsmith" -UserPrincipalName "jsmith@homelab.arpa" `
  -Path "OU=Employees,DC=homelab,DC=arpa" `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
  -Enabled $true
```

### 4. Joining the client to the domain
Last step: pointed the Windows 10 client's DNS at the Domain Controller (`192.168.1.10`), joined it to `homelab.arpa` through System Properties, and logged in as `homelab\jsmith`. That login working was the moment the whole chain — router, DC, client — proved itself end to end.

## The DNS problem I ran into (and how I chased it down)

Once the client was on the network, I tried loading a website and got nothing. First move was `ping 8.8.8.8`, which worked fine — so the connection itself was okay. Then I tried `ping google.com` and that failed outright. That gap between the two told me straight away it wasn't a connectivity problem, it was specifically DNS not resolving names to addresses.

From there I worked through it one layer at a time instead of guessing:

1. Checked whether pfSense itself could reach the internet — pinged `8.8.8.8` from pfSense's own diagnostics page, and it worked, so pfSense had a connection.
2. Checked the LAN firewall rules to rule out traffic being blocked — the default "allow LAN to any" rule was there and showing traffic passing through it, so that wasn't it either.
3. Checked the client was actually pointed at pfSense for DNS — it was, `192.168.1.1` was set correctly.
4. Ran `nslookup google.com 192.168.1.1` directly, which cut Windows out of the equation and asked pfSense the question straight — and got back **"Server failed."** That pointed the finger squarely at pfSense's own DNS service.
5. Went to pfSense's Services status page and found the real problem: neither the DNS Resolver nor the DNS Forwarder was actually running, even though the settings looked configured correctly on screen.

The fix was to properly enable the DNS Forwarder and confirm it showed up as a running service — after that, lookups worked immediately. It was a good reminder that a settings page looking right and a service actually running are two different things worth checking separately.

## Screenshots

| Screenshot | Description |
|---|---|
| `Vbox_set_up.png` | Initial pfSense VM configuration in VirtualBox — NAT (WAN) and Internal Network (LAN) adapters |
| `assigning_em0_to_WAN_and_em1_to_LANm_-_pfsense_names_NICs_after_the_drivers.png` | Assigning WAN/LAN to their respective network interfaces |
| `assigning_LAN_wip_an_IP_and_subnet.png` | Configuring the LAN interface with a static IP and subnet |
| `WAN___LAN_with_successfully_assigned_IPs.png` | Confirmation screen — both WAN and LAN interfaces live with IPs assigned |
| `pfsense_dashboard_after_initial_setup.png` | pfSense web dashboard after completing the setup wizard |
| `login_to_pfsense_from_client01.png` | Logging into the pfSense web GUI from the Windows 10 client |
| `client01_reaching_google_com.png` | Confirming internet access from the client after resolving the DNS issue |
| `AD.png` | Active Directory Organizational Units listed via PowerShell (`Get-ADOrganizationalUnit`) |
| `AD_jsmith.png` | Active Directory user object for `jsmith` (`Get-ADUser`) |
| `signed_in_on_client_as_jsmith.png` | System info confirming the Windows 10 client is joined to `homelab.arpa` |

## Tools Used

- VirtualBox
- pfSense Community Edition 2.8.1
- Windows Server 2022 (Server Core)
- Windows 10
- PowerShell / Active Directory Domain Services

## Skills Demonstrated

- Network segmentation and routing (WAN/LAN, NAT)
- Firewall rule management
- DNS configuration and layered troubleshooting
- Windows Server administration via command line (Server Core)
- Active Directory deployment and object management via PowerShell
- Domain join and authentication verification
