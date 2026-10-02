# Lab: Static IP and DNS Configuration on a Windows Server 2022 Domain Controller

**Author:** [Your Name] | **Date:** [Date] | **Platform:** VMware | **Status:** Completed and verified

## 1. Objective

Starting from an existing Windows Server VM already promoted to a domain controller (`DC01`, domain `domain.local`), convert its network settings from DHCP to a static IP address, configure DNS forwarders so the server can resolve external names, and verify that DNS and directory services are healthy.

A domain controller must have a static IP because clients and other servers locate it through DNS records that contain its address. If a DHCP lease changed the address, those records would go stale and domain logons and name resolution could fail.

## 2. Environment

| Item | Value |
|---|---|
| Server name | DC01 |
| Domain | domain.local |
| Roles | Active Directory Domain Services (AD DS), DNS Server |
| Hypervisor | VMware |
| Virtual network subnet | 192.168.202.0 /24 (255.255.255.0) |
| Default gateway | 192.168.202.2 |
| DHCP range on the virtual network | 192.168.202.126 to 192.168.202.254 |
| Static IP chosen | 192.168.202.20 |
| DNS forwarders | 8.8.8.8 (Google), 1.1.1.1 (Cloudflare) |

## 3. Procedure

### 3.1 Discover the current network settings
Ran `ipconfig /all` in PowerShell to identify the network adapter, the current IPv4 address, subnet mask and default gateway, and to confirm that **DHCP Enabled** was set to **Yes**.

### 3.2 Cross-check the VMware network
Compared the values against the virtual network settings in VMware (the subnet IP and the DHCP range). The subnet is `192.168.202.0`, the gateway is `192.168.202.2`, and VMware's DHCP hands out addresses from `192.168.202.126` to `192.168.202.254`.

### 3.3 Choose the static address
Selected **192.168.202.20**. It is in the same subnet as the gateway (required for communication) but **outside the DHCP range**, so VMware's DHCP can never assign it to another VM and cause an IP conflict.

### 3.4 Apply the static IP
1. Press `Win + R`, run `ncpa.cpl`.
2. Right-click the network adapter > **Properties**.
3. Double-click **Internet Protocol Version 4 (TCP/IPv4)**.
4. Select **Use the following IP address** and enter:
   - IP address: `192.168.202.20`
   - Subnet mask: `255.255.255.0`
   - Default gateway: `192.168.202.2`
5. Select **Use the following DNS server addresses** and set the **Preferred DNS server** to `192.168.202.20` (the DC's own address, because it hosts the domain's DNS zones).
6. Click **OK** to apply.

### 3.5 Refresh the DNS registration
Ran:
```powershell
ipconfig /registerdns
```
This makes the server re-register its host (A) record with the new address. Then confirmed in DNS Manager that the A record for `DC01` shows `192.168.202.20`.

### 3.6 Configure DNS forwarders
1. Open **DNS Manager**.
2. Right-click the **server name** (not a zone) > **Properties**.
3. Open the **Forwarders** tab and click **Edit**.
4. Enter `8.8.8.8`, press Enter, then enter `1.1.1.1` and press Enter. (There is no domain field for a normal forwarder. Only an IP address is needed.)
5. Click **OK**, then **Apply**.
6. Leave **Use root hints if no forwarders are available** ticked.

A validation warning that the server FQDN could not be resolved appears when entering plain IP addresses. It is harmless.

### 3.7 Verify functionality
```powershell
ipconfig /all           # DHCP Enabled: No, DNS server: 192.168.202.20
ping 192.168.202.2      # gateway reachable
ping 8.8.8.8            # internet reachable
ping 1.1.1.1
nslookup google.com     # external name resolution
dcdiag /test:dns        # DNS health test
```

### 3.8 DNS records: A and CNAME
Created and tested both record types in the `domain.local` forward zone (see Section 4.3).

## 4. Key concepts

### 4.1 Forwarders vs root hints

| | Forwarders | Root hints |
|---|---|---|
| What it is | Specific DNS servers (here 8.8.8.8 and 1.1.1.1) that DC01 hands external queries to | A built-in list of the internet's root DNS servers stored on the DC |
| How it resolves | The forwarder does the work and returns the answer (often already cached) | DC01 starts at a root server and follows referrals down (root, TLD, authoritative server) itself |
| Speed and reliability | Usually faster, and simple to control | Works without configuration but can be slower or blocked in NAT or lab networks |
| Used when | Configured and reachable | No forwarders configured, or forwarders unreachable (fallback) |

**Conditional forwarders** are different: they send queries for one named domain to a specific server. That is the only place a "DNS Domain" field appears. They were not needed in this lab.

A successful `nslookup` alone does not prove the forwarders are in use, since root hints could produce the same result. To test a forwarder path directly, use `nslookup google.com 8.8.8.8`, or run `Clear-DnsServerCache` and look up a name not yet cached.

### 4.2 Why a DC gets its own IP as DNS server
The domain's records (`_ldap`, `_kerberos` and others) live in the DC's own DNS zone. A DC pointing at an outside DNS server such as 8.8.8.8 would not find them. The DC then uses forwarders to resolve everything else.

### 4.3 A record vs CNAME record

| | A record | CNAME record |
|---|---|---|
| Points to | An IPv4 address | Another **name** |
| Example | `intranet` > `192.168.202.20` | `intranet` > `dc01.domain.local` |
| If the server's IP changes | Must be **updated manually** (unless it updates dynamically) | **Follows automatically**, because the IP is stored in the target name's A record |
| Extra lookups | None | One extra step to resolve the target |
| Limits | Can coexist with other record types | Cannot be used at the zone root or alongside other records of the same name |

Verification: `nslookup intranet.domain.local` shows the canonical name (`dc01.domain.local`) followed by the IP address when a CNAME is used.

### 4.4 DNS and the web server are separate roles
If DC01 also hosts an internal website, its DNS role only turns the site's name into an IP. The web server role (IIS) then delivers the content. The client must use DC01 as its DNS server, a DNS record must exist for the site name, and the firewall must allow ports 80/443. Running a web server on a DC is fine for a lab, but production environments keep DCs dedicated to AD and DNS.

## 5. Verification results

| Check | Command / tool | Result |
|---|---|---|
| Static IP applied, DHCP off | `ipconfig /all` | DHCP Enabled: No; IP 192.168.202.20; DNS 192.168.202.20 |
| Gateway reachable | `ping 192.168.202.2` | Successful (clean on retest) |
| Internet reachable | `ping 8.8.8.8` / `ping 1.1.1.1` | Successful, with intermittent packet loss at first (see 6.2) |
| External name resolution | `nslookup google.com` | Successful |
| DNS records updated | DNS Manager | DC01 A record shows the new IP |
| DNS health | `dcdiag /test:dns` | **Passed, no errors** |

## 6. Troubleshooting log

### 6.1 DHCP to static change
- **Problem:** The adapter was using a DHCP-assigned address, which is unsafe for a domain controller.
- **Cause:** Default VMware/Windows behaviour.
- **Fix:** Chose an address outside the DHCP range, set it manually through `ncpa.cpl`, and pointed DNS at the DC itself. Ran `ipconfig /registerdns` afterwards so DNS records matched.

### 6.2 Intermittent packet loss when pinging 8.8.8.8 and 1.1.1.1
- **Problem:** Pings to public DNS servers repeatedly lost one or two packets.
- **Investigation:** Pinged the default gateway and the public servers again and compared. The VM's internet traffic is routed through the host, so the connection of the host machine matters.
- **Finding:** The loss was traced to the **host machine's internet connectivity**, not the DC's configuration. A later retest to both the gateway and 8.8.8.8 showed no loss.
- **Note:** A single clean result does not fully rule out recurrence. If the loss returns, ping the gateway with `ping 192.168.202.2 -n 30` to see whether it starts at the gateway or beyond it.

### 6.3 The same DNS server appearing twice in DNS Manager
- **Problem:** DNS Manager listed two servers: `DC01` and `DC01.domain.local`.
- **Cause:** The server was already listed after adding it, and then **Connect to DNS Server** was used, which added a second entry under a different name.
- **Finding:** Both entries had identical zones and contents, so they were the same server under two names, not two servers.
- **Fix:** The extra entry can be removed from the console with no effect on the server or zones.

## 7. Observations and lessons learned

- **Order matters:** discover the network settings, choose an address outside the DHCP range, apply it, refresh DNS, then test.
- **Static addressing is required for servers** that other machines depend on. DHCP is for clients.
- **A domain controller uses itself for DNS** and uses forwarders for external names.
- **Forwarders and root hints both resolve external names**, but only forwarders are explicitly configured. Root hints are the built-in fallback.
- **Connectivity and name resolution are separate things.** A ping to an IP tests the network path; `nslookup` tests DNS. Running both tells you which layer is failing.
- **Test more than once before drawing conclusions.** Intermittent problems can come from the host or outside network rather than the VM configuration.
- **A CNAME follows the IP changes of its target; an A record does not.** This affects how I choose record types for services that may move.
- **DNS manager console entries are not always separate servers.** Compare zones and properties before assuming a problem.
- **`dcdiag /test:dns` is a quick way to confirm DNS health** after any network or DNS change.
- **Documenting errors and their causes** makes the work repeatable and shows the reasoning behind each decision.

## 8. Next steps

- Create a reverse lookup zone for `192.168.202.0/24` so PTR records resolve.
- Add a Windows client VM, point its DNS at `192.168.202.20`, and join it to `domain.local`.
- Create OUs, users and security groups, and link a Group Policy Object.
- Take a VM snapshot after each stable milestone (e.g. "Static IP and forwarders configured").
