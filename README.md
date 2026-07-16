# Home Lab: pfSense Router + Active Directory Domain

I built this lab in VirtualBox with tasks including — setting up a real firewall, segmenting a network properly, and standing up an Active Directory domain from nothing. Everything below was built solo bar some minor assistance from AI to create the architecture image and to help me present the layout

# What's in here

- A pfSense router/firewall, built from scratch
- A proper WAN/LAN split using NAT and an internal network
- A DNS outage I caused myself and had to actually diagnose
- A Windows Server 2022 box running in Server Core — no GUI, PowerShell only
- That server promoted to a Domain Controller, with users and OUs created by hand in PowerShell
- A Windows 10 machine joined to the domain, logging in as a real AD user

# Architecture

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

Something to note: pfSense doesn't call interfaces "Ethernet" or "WiFi" like Windows does — it names them after the driver. Both my virtual NICs use Intel's emulated driver, so they showed up as `em0` and `em1`. I assigned `em0` to WAN and `em1` to LAN.

From there I gave the LAN a static IP (`192.168.1.1/24`) and turned on its DHCP server so anything plugged into the LAN would get an address automatically. I also set the domain to `homelab.arpa` instead of the more common `.local`, since pfSense actually warns against `.local` as it clashes with mDNS/Bonjour discovery on some devices.

### 2. Windows 10 — the client
just a VM sitting on the LAN network, picking up an IP from pfSense's DHCP server. I used this machine as my test bench for reaching the pfSense dashboard and, later, for joining the domain.

### 3. Windows Server 2022 — the Domain Controller
I deliberately installed this in Server Core, which means no desktop, no Start menu, just PowerShell. I was adviced that this is a mor eproffesional route to take

Set a static IP (`192.168.1.10`) through the `SConfig` menu, then installed the AD DS role:

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

Promoted the box to a Domain Controller for `homelab.arpa`:

```powershell
Install-ADDSForest -DomainName "homelab.arpa"
```

And once that finished rebooting, I built out a small AD structure as an Organizational Unit and a test user:

```powershell
New-ADOrganizationalUnit -Name "Employees" -Path "DC=homelab,DC=arpa"

New-ADUser -Name "John Smith" -GivenName "John" -Surname "Smith" `
  -SamAccountName "jsmith" -UserPrincipalName "jsmith@homelab.arpa" `
  -Path "OU=Employees,DC=homelab,DC=arpa" `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
  -Enabled $true
```

### 4. Joining the client to the domain
pointed the Windows 10 client's DNS at the Domain Controller (`192.168.1.10`), joined it to `homelab.arpa` through System Properties, and logged in as `homelab\jsmith`. That login working was the moment the whole chain — router, DC, client — proved itself end to end.

## The DNS problem I ran into 

Once the client was on the network, I tried loading a website and got nothing. First move was `ping 8.8.8.8`, which worked fine — so the connection was working as intended. Then I tried `ping google.com` and that failed outright. meaning that it wasn't a connectivity problem, it was DNS not resolving names to addresses.

From there, the steps i took to resolve the issue were

1. Checked whether pfSense itself could reach the internet — pinged `8.8.8.8` from pfSense's own diagnostics page, and it worked, so pfSense had a connection.
2. Checked the LAN firewall rules to rule out traffic being blocked — the default "allow LAN to any" rule was there and showing traffic passing through it, so that wasn't that either.
3. Checked the client was actually pointed at pfSense for DNS and it was `192.168.1.1` which was set correctly.
4. Ran `nslookup google.com 192.168.1.1` directly, which cut Windows out of the equation and asked pfSense the question straight and got back "Server failed." That showed that the issue was at pfSense's own DNS service.
5. Went to pfSense's Services status page and found the issue. Neither the DNS Resolver nor the DNS Forwarder was actually running, even though the settings looked configured correctly on screen.

The fix was to tick and enable the DNS Forwarder and confirm it showed up as a running service, after that, lookups worked immediately.

## Screenshots

| Screenshot | Description |
|---|---|
| `assigning_em0_to_WAN_and_em1_to_LANm_-_pfsense_names_NICs_after_the_drivers.png` | Assigning WAN/LAN to their respective network interfaces |<img width="717" height="397" alt="assigning em0 to WAN and em1 to LANm - pfsense names NICs after the drivers" src="https://github.com/user-attachments/assets/cbaeff8a-9546-4269-af5c-fb49bebce92f" />

| `assigning_LAN_wip_an_IP_and_subnet.png` Configuring the LAN interface with a static IP and subnet |<img width="717" height="367" alt="assigning LAN wip an IP and subnet" src="https://github.com/user-attachments/assets/8c184e91-eac9-4f5b-a9dc-63544fcec45d" />

| `WAN___LAN_with_successfully_assigned_IPs.png` Confirmation screen — both WAN and LAN interfaces live with IPs assigned |<img width="715" height="277" alt="WAN   LAN with successfully assigned IPs" src="https://github.com/user-attachments/assets/d1dc3bd9-59bc-4a70-9044-e721ca114cbf" />

| `pfsense_dashboard_after_initial_setup.png` | pfSense web dashboard after completing the setup wizard |<img width="1022" height="867" alt="pfsense dashboard after initial setup" src="https://github.com/user-attachments/assets/34a89e6e-e2b9-4834-89a3-386cce1d4839" />

| `login_to_pfsense_from_client01.png` | Logging into the pfSense web GUI from the Windows 10 client |<img width="1017" height="862" alt="login to pfsense from client01" src="https://github.com/user-attachments/assets/e3aafad4-105c-41ac-9f83-a06bdb0073bb" />

| `client01_reaching_google_com.png`, Confirming internet access from the client after resolving the DNS issue | <img width="1022" height="867" alt="client01 reaching google com" src="https://github.com/user-attachments/assets/8447f28e-f8d0-4515-ad65-555c36120e95" />

| `AD.png` | Active Directory Organizational Units listed via PowerShell (`Get-ADOrganizationalUnit`) |<img width="1017" height="847" alt="AD" src="https://github.com/user-attachments/assets/edd375ea-927d-4659-a86c-d630c3544aef" />

| `AD_jsmith.png` | Active Directory user object for `jsmith` (`Get-ADUser`) |<img width="1022" height="847" alt="AD jsmith" src="https://github.com/user-attachments/assets/57dc9eee-8888-4c2b-9bdf-37e4540df950" />

| `signed_in_on_client_as_jsmith.png` System info confirming the Windows 10 client is joined to `homelab.arpa` |<img width="1017" height="862" alt="image" src="https://github.com/user-attachments/assets/a95f585d-f175-4d9c-a128-24ba2784075a" />


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
